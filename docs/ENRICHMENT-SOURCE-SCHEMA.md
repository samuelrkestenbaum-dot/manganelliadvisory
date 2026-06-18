# Enrichment Source & Confidence Schema — build spec

**Status:** drop-in ready for the maintainer. This is the **enrichment workflow** slice
referenced as slice 3 of the larger pipeline upgrade (see
`PIPELINE-STAGE-MODEL-SPEC.md` §7). It is self-contained: it needs no backend and no
external data feed — it works inside the current single-file `netlify-deploy/index.html`
with `localStorage` edits, exactly like the warm-path toggles and (soon) the stage model
do today.

It turns the 442 `SELLERS[]` and 60 `BUYERS[]` from a *seeded list* into *records whose
qualitative fields are filled from confirmed public sources, each value carrying a
source, a confidence, and a date* — and it recomputes the six-component seller-intent
score automatically as those facts land. The product discipline is unchanged: **never
guess.** A field is filled only from a confirmed public source; where the public record
is silent, the field stays blank and the confidence says so.

---

## 0. What changes, in one paragraph

Every qualitative field stops being a bare scalar and becomes a small **enriched value**:
`{ value, confidence, source_url, date_pulled, note? }`. A four-level confidence model
(**Confirmed / Inferred / Estimated / Unknown**) generalizes the existing
`service_mix_confidence` pattern (`default / text-inferred / researched`) across *all*
fields. Confidence **gates the score**: only `Confirmed` (and explicitly whitelisted
`Inferred`) values are allowed to move the six seller-intent components; `Estimated` and
`Unknown` display with a flag but do not move the score. A field-by-field **source map**
(primary → fallback → extraction method → confidence rule → cadence → which score
component it feeds) makes the research pass **repeatable, not ad hoc**. When a `Confirmed`
field lands, the relevant score component recomputes **live**, the same way toggling a
warm path recomputes today. Buyers get the parallel treatment: a written thesis plus an
explicit, sourced **target box** that powers real seller↔buyer matching.

---

## 1. Scope — what this fills and how it plugs into the score

In scope: the **qualitative** seller fields and the **buyer thesis** fields, filled from
the public internet. Out of scope: anything requiring a phone call or a signed NDA
(those are pipeline activity, not enrichment) and anything paid-broker-gated.

### How it plugs into the existing fields

Two integration surfaces already exist in the file and this slice reuses both:

1. **The `service_mix_confidence` field** (`default` / `text-inferred` / `researched`) is
   the prototype for the whole confidence model. We keep that exact field and its values
   for `service_mix`, and generalize the *idea* to a parallel `*_meta` envelope on every
   enriched field (§3). `service_mix_confidence` maps onto the new levels as:
   `default → Estimated`, `text-inferred → Inferred`, `researched → Confirmed`.
2. **The seller-intent score recompute in `view()`** — the same place that already sums
   the six weighted components and reacts to warm-path toggles — is where enriched,
   `Confirmed` fields are read. Nothing else recomputes the score; this slice just feeds
   it better inputs and gates them on confidence (§5).

### The six score components (unchanged weights — the recompute target)

| Component | Max | One-line meaning |
|---|---:|---|
| `succession` | 35 | Retirement / no-successor / listed-for-sale pressure to transact. |
| `age_tenure` | 25 | Owner age and partner tenure — proximity to a natural exit. |
| `ownership` | 15 | Concentration / founder-led / thin next-gen bench. |
| `revenue_fit` | 15 | Revenue lands in the sweet spot buyers want. |
| `contactability` | 10 | We can reach a decision-maker (email/phone confidence). |
| `warm_path` | 10 | A warm route in exists (already toggled live today). |
| **Total** | **100** | → `seller_intent_score` → `seller_intent_band` → `priority_rank`. |

Enrichment moves the **first five**; `warm_path` is already wired. The job of this spec is
to say, field by field, *which confirmed fact moves which component, in which direction.*

---

## 2. Confidence model

Four levels. They are deliberately coarse — the goal is an auditable gate, not false
precision. Each level has a **rule** (when you may assign it) and a **score posture**
(whether it may feed the score).

| Level | `conf` key | Rule — assign only when… | Feeds score? | Visual |
|---|---|---|---|---|
| **Confirmed** | `confirmed` | A primary public source states the fact directly (firm's own site, a license-board record, a press release, a live broker listing). One unambiguous source is enough. | **Yes** | normal text, small source-link icon |
| **Inferred** | `inferred` | Not stated outright but a reasonable read of stated facts (e.g., headcount + a "tax & audit" services page ⇒ service_mix split; LinkedIn grad year ⇒ approximate age). Logic must be reconstructable from the cited source. | **Whitelist only** (§5b) | normal text, dotted underline |
| **Estimated** | `estimated` | A defensible placeholder from indirect signal or a default model (revenue band from "Book of Lists" tier; service mix from firm-name keywords). | **No** — display only | amber chip / italic |
| **Unknown** | `unknown` | Public record is silent. Field stays blank. | **No** | em-dash, muted |

**Gating rule (the core of the model):** a value moves a score component **iff** its
confidence is `confirmed`, **or** it is `inferred` **and** that field appears on the
score whitelist in §5b. Everything else displays (so the analyst sees it) but is treated
as *absent* by the scorer. This is what stops a keyword-guessed service mix or a
broker-tier revenue estimate from silently inflating intent.

**Mapping onto the existing `service_mix_confidence`:**

| Existing value | New level | Score posture for `service_mix` |
|---|---|---|
| `default` (untouched seed) | `Estimated` | display only |
| `text-inferred` (parsed from firm name/notes) | `Inferred` | feeds `revenue_fit`/quality only via §5b whitelist |
| `researched` (analyst confirmed on the site) | `Confirmed` | feeds normally |

**Aging.** A `confirmed` value older than its field's **refresh cadence** (§4 column) is
downgraded one notch at render time (`confirmed → inferred`, shown with a "stale, pulled
YYYY-MM-DD — re-verify" tooltip) until re-pulled. This keeps the book honest as firms
change without silently dropping data.

---

## 3. The stored shape of an enriched value

Every enriched field is stored as an envelope. Keep the human-readable scalar on the
record (so existing renderers and the `service_mix_confidence` field keep working
untouched) and attach a parallel `_meta` map keyed by field name. This is additive and
`localStorage`-friendly — nothing existing breaks.

```json
{
  "firm": "ADKF, P.C.",
  "service_mix": "Tax 45% · Audit 25% · CAS/Advisory 30%",
  "service_mix_confidence": "researched",
  "owner_count": 7,
  "next_gen_leadership": "No",
  "_meta": {
    "service_mix": {
      "value": "Tax 45% · Audit 25% · CAS/Advisory 30%",
      "confidence": "confirmed",
      "source_url": "https://www.adkf.com/services/",
      "date_pulled": "2026-06-18",
      "note": "Three service lines named on /services; split estimated from page emphasis."
    },
    "owner_count": {
      "value": 7,
      "confidence": "confirmed",
      "source_url": "https://www.adkf.com/our-team/",
      "date_pulled": "2026-06-18"
    },
    "owner_age_estimate": {
      "value": 63,
      "confidence": "inferred",
      "source_url": "https://www.linkedin.com/in/...",
      "date_pulled": "2026-06-18",
      "note": "Managing partner BBA 1985 → ~age 63; LinkedIn login-walled, hence Inferred."
    }
  }
}
```

**Field contract:** `value` (any) · `confidence` (one of the four keys) · `source_url`
(string; the page that proves it) · `date_pulled` (`YYYY-MM-DD`) · `note?` (optional free
text: the inference logic or a caveat). `source_url` and `date_pulled` are **mandatory**
for any value above `unknown` — that is what makes a value re-verifiable and age-able. A
field with no `_meta` entry is treated as `Estimated`/seed (legacy), never as `Confirmed`.

*Illustrative only: ADKF, P.C. (#345) is used as a running example because it appears in
the stage-model spec; figures here are placeholders to show the shape, not researched
facts.*

---

## 4. Seller enrichment table — field by field

The repeatable source map. **Primary** = hit first; **Fallback** = if primary is silent.
**Confidence rule** = what level a clean pull earns. **Cadence** = re-verify interval
(drives §2 aging). **Feeds** = which §1 score component the `Confirmed` value moves (or
"display only").

| Target field | What it captures | Primary source | Fallback source(s) | Extraction method | Confidence rule | Cadence | Feeds |
|---|---|---|---|---|---|---|---|
| `website` | Canonical firm URL | Google search "firm + city CPA" | State CPA society directory; firm Google Business panel | Resolve to the real owned domain (not a directory listing) | `Confirmed` if it's the firm's own domain | 12 mo | display (unlocks all others) |
| `service_mix` | Tax / Audit / CAS-Advisory / niche split | Firm site **/services** page | Home-page hero copy; Glassdoor "what we do" | Read the named service lines; estimate split from page emphasis | `Confirmed` (lines named) / `Inferred` (split) / `Estimated` (name-keyword only) → set `service_mix_confidence` accordingly | 12 mo | `revenue_fit` (quality lift; §5) |
| `niche_vertical` | Industry specialization | Firm site **/industries** page | News/award copy; LinkedIn "specialties" | Take explicitly listed verticals; "Generalist" if none claimed | `Confirmed` if a page claims it; else `Estimated`/`Unknown` | 12 mo | `revenue_fit` (quality lift) |
| `owners` / `partner_count` / `owner_count` | Named owners & how many | Firm **/team** or **/about** page | SoS officer list; state license board | Count partner/principal/owner titles; capture names | `Confirmed` (titles shown) / `Inferred` (counted from bios) | 6 mo | `ownership` |
| `managing_partner` | Lead decision-maker | Firm /team page (MP/founder title) | LinkedIn; news bylines | Name + title where stated | `Confirmed` if titled on site | 6 mo | `contactability` (route), display |
| `next_gen_leadership` | Is there a visible successor? | Firm /team page (younger partners/"future leaders") | LinkedIn tenure spread; recent partner-admit news | Yes if non-founder partners < ~50 / recently admitted; No if all principals are senior | `Confirmed` (named young partners) / `Inferred` (tenure read) | 6 mo | `ownership`, `succession` |
| `owner_age_estimate` | Approx. age of lead owner(s) | LinkedIn (grad year → age) | Bio "since 19xx"; alumni notes | Graduation/first-license year + ~22; round to nearest year | **`Inferred` max** (LinkedIn is login-walled — never `Confirmed` from it) | 12 mo | `age_tenure` |
| `max_tenure_years` / partner tenure | Longest-tenured owner's years | LinkedIn "Partner since"; firm bio | SoS formation date as a ceiling | Today − start year; cap at `years_in_business` | `Inferred` (LinkedIn) / `Confirmed` (firm bio states it) | 12 mo | `age_tenure` |
| `founded` / `years_in_business` | Firm age | State **Secretary of State** entity formation | Firm "since 19xx"; About page | Entity formation year; `years = currentYear − founded` | `Confirmed` (SoS/registry) | 36 mo (rarely changes) | `age_tenure` (supports tenure) |
| `employees` | Headcount | Firm /team page count | LinkedIn "company size" band; AICPA/best-firms tier | Count staff; or take LinkedIn band midpoint | `Confirmed` (counted on site) / `Estimated` (band) | 6 mo | `revenue_fit` (size sanity) |
| `office_count` | # of offices | Firm **/locations** page | Google Business listings; SoS branch filings | Count distinct office addresses | `Confirmed` (locations page) | 12 mo | display (multi-office = harder succession) |
| `growth_signal` | Hiring / opening / merger chatter | Indeed/Glassdoor open reqs; firm news | Local business journal; LinkedIn headcount trend | Open roles or "we're hiring"/new office ⇒ growing; layoffs/wind-down ⇒ declining | `Confirmed` (live posting/press) / `Inferred` (trend) | 3 mo (fast-moving) | display + EV quality (stage spec §2a) |
| `retirement_flag` | Owner signaling retirement | News / firm announcement | LinkedIn "retiring"/emeritus; obituary-adjacent succession notes | Explicit retirement language about an owner | `Confirmed` (stated) / `Inferred` (age+no-successor) | 6 mo | **`succession`** |
| `no_successor_flag` | No internal next gen | Firm /team (all senior, no junior partners) | LinkedIn tenure spread | True when `next_gen_leadership = No` and ownership is concentrated | `Inferred` (read) / `Confirmed` (stated in press) | 6 mo | **`succession`**, `ownership` |
| `active_listing_flag` | Firm literally for sale | **BizBuySell** / broker listing matching the profile | Other M&A marketplaces; broker teaser | Match location + revenue band + service mix to a live listing | `Confirmed` only (a real listing) / else `Unknown` | 1 mo (listings expire) | **`succession`** (strong) |
| `best_email` / `email_confidence` | Reachable email | Firm /contact page (named inbox) | Pattern from a confirmed name (`first.last@domain`); MX-valid | Prefer a named address over `info@`; record the existing `email_confidence` | `Confirmed` (published) / `Inferred` (pattern) | 12 mo | **`contactability`** |
| `office_phone` | Main line | Firm /contact page | Google Business; SoS registered agent | Take the published main number | `Confirmed` (published) | 12 mo | `contactability` (minor) |
| `revenue` / `revenue_display` | Annual revenue / band | "Book of Lists" / business-journal ranking | AICPA tier; headcount × per-head revenue proxy | Take ranked figure; else estimate band from headcount | `Confirmed` (ranking) / `Estimated` (proxy) | 12 mo | **`revenue_fit`** |

> **Note on `revenue`:** revenue is partly numeric, but its *confidence* matters for the
> score and the stage-model EV multiple. A `Confirmed` (ranked) revenue feeds `revenue_fit`
> and the EV calc at full weight; an `Estimated` (headcount-proxy) one displays and feeds
> EV but is flagged so a band boundary isn't trusted blindly.

---

## 5. Score-recompute rules

This is the contract between an enriched field and the six components. It runs in the
**same `view()` recompute** that already powers the warm-path toggle: when a `Confirmed`
(or whitelisted `Inferred`) field is written via `saveEdit`, the affected components
re-derive and `seller_intent_score` / `band` / `priority_rank` update **live**. No
separate batch step.

### 5a. Component-by-component triggers (direction explicit)

**`succession` (max 35) — the heaviest lever.**
- `active_listing_flag = true` (Confirmed live listing) → push toward the **35 max**
  (firm is self-identifying as a seller — the strongest possible signal).
- `retirement_flag = true` (Confirmed) → large positive push.
- `no_successor_flag = true` → positive push; **compounds** with `retirement_flag` and
  with `owner_age_estimate ≥ 62`.
- All three silent/`Unknown` → succession contributes near its floor.

**`age_tenure` (max 25).**
- `owner_age_estimate ≥ 62` → push toward the **25 max**; `55–61` → mid; `< 50` → low.
- `max_tenure_years ≥ 25` (long-tenured owner) → reinforces toward max.
- Because age is **`Inferred` at best** (LinkedIn gating, §8), it is whitelisted (§5b) but
  contributes at a slightly **discounted weight** vs. a `Confirmed` fact.

**`ownership` (max 15).**
- `founder_led = true` and `owner_count ≤ 2` and `next_gen_leadership = No` →
  push toward the **15 max** (concentrated, no bench → must transact to exit).
- A visible `next_gen_leadership = Yes` → **pull down** (internal succession is an
  alternative to selling).

**`revenue_fit` (max 15).**
- `revenue` Confirmed within the buyer sweet spot (e.g., `$1.5M–$10M`) → toward max.
- Service-mix quality adjustment: `service_mix` Confirmed as CAS/advisory-heavy or a real
  `niche_vertical` → small positive nudge (mirrors the stage-model EV quality bump).

**`contactability` (max 10).**
- `email_confidence = Confirmed` (named, published inbox) → toward the **10 max**.
- `Inferred` (pattern) email → partial. `office_phone` Confirmed → small additive.
- All Unknown → floor.

**`warm_path` (max 10).** Unchanged — already recomputes live on toggle; enrichment does
not touch it (it's relationship data, not public-source data).

### 5b. The `Inferred`-feeds-score whitelist

Most `Inferred` values display only. These specific ones **are allowed to move the
score**, because the inference is well-grounded and the field would otherwise be chronically
empty (the data is structurally login-walled or never stated outright):

- `owner_age_estimate` (grad-year inference) → `age_tenure`, at discounted weight.
- `max_tenure_years` (LinkedIn "since") → `age_tenure`.
- `next_gen_leadership` / `no_successor_flag` (tenure-spread read) → `ownership`,
  `succession`.
- `service_mix` = `text-inferred` → `revenue_fit` quality nudge only.

Everything not on this list: `Inferred` and `Estimated` **display only**.

### 5c. Worked example (illustrative)

> Take a firm where research confirms: founder-led, `owner_count = 2`,
> `next_gen_leadership = No` (Confirmed — all principals senior on /team), and infers
> `owner_age_estimate = 64` (LinkedIn grad year). When those three `saveEdit`s land:
> `succession` pushes up (no-successor + concentrated), `age_tenure` pushes toward its 25
> max (age ≥ 62, Inferred/whitelisted at discount), and `ownership` pushes toward its 15
> max (founder-led, ≤ 2 owners, no bench). The score recomputes in the drawer instantly —
> the same loop as flipping a warm-path chip — and the firm's `priority_rank` rises. If a
> later pass finds an `active_listing_flag = true` on BizBuySell, `succession` jumps toward
> 35 and the firm likely crosses into the top band.

**Rule, stated plainly:** *the recompute must fire automatically the moment a `Confirmed`
(or whitelisted `Inferred`) field is saved — never on a manual "rescore" button.*

---

## 6. Buyer enrichment table — building an acquisition thesis

Each of the 60 `BUYERS[]` gets a **written thesis** plus an explicit, sourced **target
box**. The target box + recent deals are what make seller↔buyer matching real (§6b).

| Target field | What it captures | Primary source | Fallback source(s) | Extraction method | Confidence rule | Cadence | Powers |
|---|---|---|---|---|---|---|---|
| `fit` / `rationale` | The written thesis: why this buyer, what they want | Buyer site **/acquisitions** or "grow with us" page | Press releases; CEO interviews | Summarize stated criteria into 1–2 sentences | `Confirmed` (stated) / `Inferred` (read from deal pattern) | 12 mo | thesis display, match narrative |
| `size_lo` / `size_hi` | Target **revenue band** they'll acquire | Buyer's stated acquisition criteria | Inferred from `recent_deals` sizes | Take stated $ range; else min/max of recent target revenues | `Confirmed` (stated) / `Inferred` (from deals) | 12 mo | **revenue-band match** |
| `geos` | Geographies they want | Acquisitions page; press | Footprint on /locations; deal cities | List states/regions stated or implied by deal map | `Confirmed` (stated) / `Inferred` (footprint) | 12 mo | **geography overlap** |
| `services` | Must-have / priority service lines | Strategy/press ("expanding CAS", "audit-led") | Recent-deal target profiles | List the service lines they're buying for | `Confirmed` (stated) / `Inferred` | 12 mo | **service-line fit** |
| Deal structures | What they'll do (full buyout / majority recap / equity rollover / tuck-in / merger-of-equals) | Sponsor/buyer press describing deal terms | Trade-press deal write-ups | Note structures seen in their deals | `Confirmed` (stated terms) / `Inferred` | 12 mo | **structure fit** |
| `recent_deals` | Named recent acquisitions | **Accounting Today** / **INSIDE Public Accounting** deal coverage | Buyer press releases; sponsor site | List target name + date + (if public) size | `Confirmed` (named in press) | 3 mo (consolidation wave is fast) | cadence, size/geo inference |
| Deal cadence | Deals per year | Derived from `recent_deals` dates | Sponsor "platform" commentary | Count deals ÷ years observed | `Inferred` (computed) | 3 mo | appetite / prioritization |
| `sponsor` + fund vintage | PE backer and fund age | **PE sponsor site** + press | Trade press ("backed by X, Fund N, 20xx") | Name sponsor; capture fund vintage/close year | `Confirmed` (sponsor site/press) | 12 mo | appetite, hold-period read |
| `contact` / `contact_title` | Named corp-dev / M&A contact | Buyer site "team"/"M&A" page | LinkedIn corp-dev titles; press contacts | Capture name + title of the deal-sourcing person | `Confirmed` (named on site) / `Inferred` (LinkedIn) | 6 mo | warm-path / outreach routing |
| `type` / `tier` | Buyer category & priority | Self-description + sponsor structure | Trade-press classification | Map to the 7 `type` values; set tier from fit | `Confirmed` / `Inferred` | 12 mo | match ranking |

Buyer `type` values (unchanged, for reference): **PE-Backed Consolidator, PE Platform,
Top-100 Strategic, Regional Consolidator, Regional / Boutique Firm, Search Fund,
Independent Sponsor.**

### 6b. How the target box powers seller↔buyer matching

A seller matches a buyer when the seller's **enriched, Confirmed** facts fall inside the
buyer's **target box**. Four hard filters + two bonuses:

1. **Revenue band** — `buyer.size_lo ≤ seller.revenue ≤ buyer.size_hi`. (Seller `revenue`
   must be at least `Estimated`; a `Confirmed` revenue makes the match high-confidence.)
2. **Geography overlap** — `seller.state`/`region` ∈ `buyer.geos` (or buyer geos = national).
3. **Service-line fit** — `buyer.services` intersects `seller.service_mix` /
   `niche_vertical` (e.g., a CAS-hungry buyer ↔ a CAS-heavy seller).
4. **Structure fit** — seller's situation (e.g., founder wanting a clean exit vs. a younger
   partner wanting rollover) is compatible with a structure the buyer will do.
5. **Cadence bonus** — a buyer doing many deals/year and actively in-market is prioritized
   over a dormant one (from `recent_deals`).
6. **Warm-path bonus** — if a known relationship connects the seller to this buyer
   (the existing warm-path data), the pair is boosted — mirroring how `warm_path`
   already adds to the seller score.

The output is a ranked buyer shortlist per seller (and, inverted, a `matched_count` per
buyer). This is exactly the "real seller↔buyer matching" slice the stage-model spec calls
out (§7 item 2); the **target box defined here is its required input**. Every filter runs
off sourced, dated fields, so a match is explainable ("matched on revenue band + Texas +
CAS, buyer did 6 deals in the last 12 months").

---

## 7. Repeatable research checklists (time-boxed)

The point of a fixed order is that two analysts produce the same record. Hit sources in
this sequence; stop early when the fields are filled.

### 7a. Per-seller firm pass — target ~12–15 min

1. **Firm website (5 min)** — the spine. `/services` → `service_mix`; `/industries` →
   `niche_vertical`; `/team` or `/about` → `owners`, `partner_count`, `owner_count`,
   `managing_partner`, `next_gen_leadership`, `employees`; `/locations` → `office_count`;
   `/contact` → `best_email`, `office_phone`. Mark all `Confirmed`.
2. **Secretary of State / state CPA license board (2 min)** — `founded` (entity formation),
   officers (cross-check `owners`), registered agent. `Confirmed`.
3. **LinkedIn (3 min, login-walled → cap at `Inferred`)** — managing-partner grad year →
   `owner_age_estimate`; "Partner since" → `max_tenure_years`; headcount band / recent
   hires & departures → `growth_signal`, sanity-check `employees`.
4. **Google Business / review sites (1 min)** — confirm `office_count`, read client volume.
5. **Glassdoor / Indeed (1 min)** — open reqs ⇒ `growth_signal = growing`; culture/ownership
   hints.
6. **"Book of Lists" / business journal / AICPA & state-society directory (1–2 min)** —
   `revenue` band, size-tier confirmation, any merger chatter → `growth_signal`.
7. **BizBuySell / broker marketplaces (1 min)** — a live listing matching the profile →
   `active_listing_flag = Confirmed` (high-value; check every refresh).
8. **Stamp & recompute** — each value saved with `source_url` + `date_pulled`; the
   `Confirmed`/whitelisted writes auto-recompute the score (§5).

### 7b. Per-buyer pass — target ~12–15 min

1. **Buyer's own site (5 min)** — `/acquisitions` / "partner with us" → `fit`, `rationale`,
   `size_lo`/`size_hi`, `geos`, `services`, deal structures; "team"/"M&A" → `contact`,
   `contact_title`. `Confirmed`.
2. **PE sponsor site + press (3 min)** — `sponsor`, fund vintage, platform strategy,
   hold-period posture. `Confirmed`.
3. **Accounting Today + INSIDE Public Accounting deal coverage (4 min)** — `recent_deals`
   (named targets + dates + sizes), from which derive **deal cadence** and back-fill
   `size_lo`/`size_hi`/`geos`/`services` where the buyer didn't state them (mark those
   `Inferred`).
4. **LinkedIn (1–2 min, → `Inferred`)** — confirm/locate the corp-dev contact title.
5. **Stamp** — every thesis and target-box field carries `source_url` + `date_pulled`;
   refresh `recent_deals` on the 3-month cadence (the wave moves fast).

---

## 8. Practical & ethical notes

- **Respect site ToS and login walls.** Most LinkedIn data is login-gated. Treat anything
  read there — age, tenure, headcount trend, contact titles — as **`Inferred`/`Estimated`
  at best, never `Confirmed`**, and don't scrape behind the wall. The §5b whitelist exists
  precisely so these structurally-walled facts can still inform the score without being
  overstated.
- **Primary sources over data-broker scrapers.** Prefer the firm's own site, a license
  board, a Secretary of State registry, or a named press release over an aggregated
  data-broker profile. Brokered/aggregated data, when used at all, is `Estimated`.
- **Always record `source_url` + `date_pulled`.** This is non-negotiable: it makes every
  value re-verifiable by a second analyst and lets the §2 aging logic age values out.
  A value without a source link is treated as seed/`Estimated`, never `Confirmed`.
- **Public + factual only.** Stick to business facts a firm publishes about itself or that
  appear in news/registries. No personal data beyond professional, published bio facts.
- **Blanks are a feature.** Leaving a field `Unknown` is correct and expected — it is the
  "never guess" discipline, and the score treats absence honestly rather than inventing a
  number.

---

## 9. Assumptions to tune (put these in front of Joe/Sam)

1. **Confidence gate (§2)** — that only `Confirmed` + whitelisted `Inferred` move the
   score. Tighten (drop the whitelist) or loosen per appetite for inferred signal.
2. **The `Inferred` whitelist (§5b)** — which inferred fields are "good enough" to score.
   Age and successor reads are the debatable ones.
3. **Discount weight on Inferred** (`age_tenure` from LinkedIn) — currently a soft
   discount vs. a `Confirmed` fact; set the exact factor.
4. **Score-trigger thresholds (§5a)** — age `≥ 62`, tenure `≥ 25`, `owner_count ≤ 2`,
   revenue sweet spot `$1.5M–$10M`. All placeholders; calibrate to the book.
5. **Refresh cadences (§4 / §6)** — listings monthly, growth signal quarterly, the rest
   6–12 months; tune to how fast each fact actually moves and to analyst capacity.
6. **Match filters & bonuses (§6b)** — which of the four filters are hard vs. soft, and
   the size of the cadence and warm-path bonuses.

Each of these is a single named constant when implemented — same discipline as the
stage-model spec: every judgment lives in one auditable place.

---

## 10. Where this sits in the larger upgrade

This is **slice 3** in `PIPELINE-STAGE-MODEL-SPEC.md` §7 ("Enrichment workflow — the
field-by-field source/confidence/date schema that keeps the multiples and narratives
honest and recomputes intent as facts land"). It is the data-quality foundation the other
slices lean on:

- **Stage model (slice 0, the spine)** — enrichment is the **exit criterion for the
  `Sourced → Researched` stage transition** (`service_mix_confidence = researched` ⇒
  Researched). The EV quality adjustment in that spec's §2a reads the `service_mix`,
  `niche_vertical`, and `growth_signal` that *this* spec confirms.
- **Real seller↔buyer matching (slice 2)** — consumes the buyer **target box** (§6) as its
  required input; matches are only as trustworthy as the `Confirmed` fields behind them.
- **Activity log (slice 1)** — orthogonal; enrichment is public-source data, activity is
  relationship history. They meet only at `warm_path`, which enrichment deliberately does
  not touch.

The through-line across all four specs is identical: **every number traces to one
auditable source — a constant, a stage rule, or now a `{source_url, date_pulled,
confidence}` envelope — and nothing is ever guessed.**
