# Pipeline Stage Model & Forecasting Layer — build spec

**Status:** drop-in ready for the maintainer. This is the first and highest-leverage
slice of the larger pipeline upgrade. It is self-contained: it needs no external data
and no backend — it works inside the current single-file `netlify-deploy/index.html`
with `localStorage` edits, exactly like the warm-path toggles do today.

It turns the 442 firms from a *scored list* into *opportunities that move through
stages and carry a forecastable value*. Two later slices (activity log, real
seller↔buyer matching) build on the fields defined here; this spec calls out those
seams but does not require them.

---

## 0. What changes, in one paragraph

Replace the `outreach_status` dropdown (`New / Researching / Queued / Contacted /
In conversation / Meeting set / Passed / Engaged`) with a clean, ordered **stage**
model. Add an **owner** to every opportunity. Compute an **estimated deal value**
(revenue × sector multiple) and a **Manganelli fee estimate**, multiply by a
**stage-based win probability**, and roll the book up into a **weighted pipeline
forecast**. Stamp `stage_entered_at` so "days in stage" and a staleness flag fall
out for free. "Meeting set / call booked" stops being a stage and becomes an
attribute (a date field), because it can be true in several stages.

---

## 1. The stage model

Eleven stages in three categories: **Open** (live, counts toward the forecast),
**Won**, **Lost/Nurture** (out of the forecast, recyclable). Probabilities are
anchored to the values you specified (Sourced 2, Qualified 15, Mandated 40,
In Market 60, LOI 80) and interpolated between.

| # | `stage` key | Label | Category | Win prob | One-line meaning |
|---|-------------|-------|----------|---------:|------------------|
| 1 | `sourced` | Sourced | Open | 2% | In the book; not yet touched. |
| 2 | `researched` | Researched | Open | 4% | Enrichment pass done; ready to work. |
| 3 | `contacted` | Contacted | Open | 6% | First touch sent; no reply yet. |
| 4 | `engaged` | Engaged | Open | 10% | Owner replied with interest. |
| 5 | `qualified` | Qualified | Open | 15% | Real conversation; would consider a deal + rough economics confirmed. |
| 6 | `mandated` | Mandated | Open | 40% | **Signed engagement letter — the conversion event that matters most.** |
| 7 | `in_market` | In Market | Open | 60% | Actively being taken to buyers. |
| 8 | `loi` | Under LOI | Open | 80% | A buyer LOI is signed. |
| 9 | `diligence` | Diligence | Open | 90% | Confirmatory diligence underway. |
| 10 | `closed_won` | Closed-Won | Won | 100% | Deal closed; fee earned. |
| 11 | `closed_lost` | Closed-Lost / Nurture | Lost | 0% | Not now; recyclable back to `researched`. |

The deliberate jump from Qualified (15%) to Mandated (40%) encodes that the signed
engagement letter is the real conversion — everything before it is unpaid pipeline.

### Per-stage entry / exit criteria, owner action, and SLA

Each stage advances only when its **exit criteria** are met (which are the next
stage's **entry criteria**). "SLA" is the days-in-stage after which the opportunity
auto-flags as stale (see §4).

**1 · Sourced** — *Entry:* loaded into the book. *Exit:* enrichment pass complete
(service mix, niche, owner facts confirmed and dated). *Owner action:* none yet /
assign. *SLA:* n/a (idle is fine here).

**2 · Researched** — *Entry:* enriched + `service_mix_confidence = researched`.
*Exit:* a first outreach has been sent. *Owner action:* draft + send first touch.
*SLA:* 21 days (research that never gets called is waste).

**3 · Contacted** — *Entry:* first touch sent (logged). *Exit:* owner replies, or we
get a referral/intro that opens a thread. *Owner action:* 2–3 follow-ups on a cadence,
then decide nurture vs. drop. *SLA:* 14 days since last touch.

**4 · Engaged** — *Entry:* a two-way conversation exists (reply, call accepted).
*Exit:* a substantive call happens where they confirm openness to a transaction and
give rough size/economics. *Owner action:* book the qualifying call. *SLA:* 10 days.

**5 · Qualified** — *Entry:* confirmed they'd consider a deal + rough revenue/terms.
*Exit:* engagement letter signed. *Owner action:* present and close the mandate.
*SLA:* 30 days (the Qualified→Mandated gap is the classic bottleneck — watch it).

**6 · Mandated** — *Entry:* signed engagement letter. *Exit:* materials ready and we
begin approaching buyers. *Owner action:* build the one-pager / CIM, set the buyer list.
*SLA:* 21 days.

**7 · In Market** — *Entry:* at least one buyer formally approached. *Exit:* an LOI is
signed. *Owner action:* run the buyer process; track each seller↔buyer pair
(approached / passed / interested). *SLA:* 60 days.

**8 · Under LOI** — *Entry:* signed LOI. *Exit:* diligence opens / exclusivity starts.
*Owner action:* manage to diligence. *SLA:* 21 days.

**9 · Diligence** — *Entry:* confirmatory diligence active. *Exit:* close. *Owner
action:* shepherd to close. *SLA:* 45 days.

**10 · Closed-Won** — terminal. Record final EV and actual fee.

**11 · Closed-Lost / Nurture** — *Entry:* owner declines, deal dies, or timing is
"not now." Record a **loss reason** (see field below). Can be recycled to
`researched` with a `nurture_until` date.

### Movement rules

- **Forward by default, regression allowed.** Stages normally advance, but a deal can
  slip back (e.g., LOI falls out → back to In Market). Log the change; don't block it.
- **Probability follows stage automatically** but is **overridable per deal**
  (`win_probability_override`) for the rare case a partner knows better.
- **Lost is not deletion.** Closed-Lost stays in the book, drops out of the forecast,
  and is filterable so it can be re-worked later.

### Migration map (run once on existing `outreach_status`)

| Current `outreach_status` | New `stage` |
|---|---|
| New | `sourced` (if `service_mix_confidence ≠ researched`) else `researched` |
| Researching | `researched` |
| Queued | `researched` |
| Contacted | `contacted` |
| In conversation | `engaged` |
| Meeting set | `engaged` (+ set `meeting_at`) |
| Engaged | `engaged` |
| Passed | `closed_lost` (loss_reason = "passed at sourcing") |

Because the whole book was just enriched, the one-time backfill will land **all 441
researched firms in `researched`** unless they already carry a later status.

---

## 2. The forecasting layer

Three inputs per opportunity → one weighted number that rolls up.

### 2a. Estimated enterprise value (EV)

`EV = revenue × multiple`. Accounting-firm transaction multiples by size, expressed
on **revenue** (simpler than EBITDA given we already store a revenue band). These are
**assumptions to tune**, grounded in current small/mid-market CPA M&A norms (roughly
~1× revenue for small comp-tax shops, rising toward ~2×+ for platform-quality firms;
equivalently ~4–8× EBITDA).

| Revenue band | Low multiple | High multiple |
|---|---:|---:|
| `< $1.5M` | 0.90× | 1.10× |
| `$1.5–3M` | 1.00× | 1.30× |
| `$3–5M` | 1.10× | 1.50× |
| `$5–10M` | 1.30× | 1.80× |
| `$10–15M` | 1.50× | 2.00× |
| `> $15M` | 1.80× | 2.50× |

**Quality adjustment** (applied to both ends, capped at the next band's range):
`+0.20×` if the firm has a real niche, is CAS/advisory-heavy, or is growing;
`-0.20×` if it's declining or pure seasonal comp-tax. This is where the **enrichment
pass pays off** — researched service mix and niche feed the multiple, not just the
display. Store the EV as a **range** (`est_ev_low`, `est_ev_high`) and forecast off the
**midpoint**.

> Worked example — ADKF, P.C. (#345), revenue $10.0M, strong multi-industry niche.
> Bands are **lower-inclusive / upper-exclusive**, so $10.0M sits at the bottom of the
> `$10–15M` band = 1.50–2.00×; +0.20× quality → 1.70–2.20× → **EV ≈ $17.0M–$22.0M,
> midpoint $19.5M.**

### 2b. Manganelli fee estimate

Sell-side success fee as a function of EV (tunable; modeled here as a declining
schedule with a floor, the usual shape for sub-$15M deals):

| EV | Success-fee rate | Notes |
|---|---:|---|
| ≤ $2M | 10% | subject to **$50k minimum** |
| $2–5M | 8% | |
| $5–10M | 6% | |
| > $10M | 5% | "double-Lehman"-ish blended |

`fee_estimate = max(EV_midpoint × rate(EV_midpoint), 50_000)`. Readiness/valuation and
post-close-integration engagements (the other Manganelli service lines) can be added
later as separate fee rows; the forecast below uses the sell-side success fee.

> ADKF example: EV midpoint $19.5M → 5% → **fee ≈ $975k.**

### 2c. Win probability

`p = win_probability_override ?? STAGE_PROB[stage]` (from the §1 table).

### 2d. Weighted value and roll-ups

```
weighted_fee = fee_estimate × p
weighted_ev  = EV_midpoint  × p
```

Management roll-ups (all computed over **Open** opportunities only, unless noted):

- **Gross pipeline** — Σ `fee_estimate` (unweighted) and Σ `EV_midpoint`.
- **Weighted pipeline** — Σ `weighted_fee` (the headline "what's the book worth to us")
  and Σ `weighted_ev`.
- **By stage** — count, gross, and weighted, per stage. This exposes the bottleneck
  ("$X weighted stuck at Qualified").
- **By owner** — same three, per assignee (needs §3 ownership).
- **Expected to close this quarter** — Σ `weighted_fee` where `expected_close` ≤ quarter
  end (needs the optional `expected_close` field).
- **Won YTD** — Σ actual fee where `stage = closed_won`.

> ADKF example, at `qualified` (p = 15%): weighted_fee = $975k × 0.15 = **$146k**.
> If it converts to `mandated` (p = 40%) the same deal jumps to **$390k weighted** —
> which is exactly the Qualified→Mandated conversion leadership should be pushing.

---

## 3. Data model — new fields per opportunity

Add to each seller record (all editable + persisted to `localStorage` like today's
edits; computed fields are derived at render time, not stored):

| Field | Type | Source | Notes |
|---|---|---|---|
| `stage` | enum (§1 keys) | editable | replaces `outreach_status` |
| `stage_entered_at` | ISO date | set on stage change | drives days-in-stage |
| `owner` | enum (Joe, Sam, …) | editable | §3 ownership |
| `meeting_at` | ISO date | editable | was the "Meeting set" status |
| `win_probability_override` | int 0–100 / null | editable | optional |
| `expected_close` | ISO date / null | editable | enables quarter forecast |
| `loss_reason` | enum / text | editable | required when `closed_lost` |
| `nurture_until` | ISO date / null | editable | recycle date |
| `est_ev_low` / `est_ev_high` | computed $ | revenue × multiple table | display as range |
| `fee_estimate` | computed $ | §2b | |
| `win_probability` | computed % | override ?? stage prob | |
| `weighted_fee` | computed $ | fee × prob | |

`loss_reason` enum suggestion: `price`, `timing / not ready`, `chose another advisor`,
`going to known buyer directly`, `merged elsewhere`, `no successor issue resolved`,
`unresponsive`, `other`.

> **Seam for later slices:** the activity-log slice adds an `activity[]` array
> (`{ts, who, channel, note, stage_from, stage_to}`) and derives *days since last
> touch* from its last entry; until then, `stage_entered_at` gives *days in stage*,
> which already powers the staleness flag. The real-matching slice replaces the
> `cbuyers` array with scored, deduped pairs each carrying their own
> `pair_status` (approached / passed / interested / loi).

---

## 4. Operational signals (free from the fields above)

- `days_in_stage = today − stage_entered_at`.
- **Stale flag:** `days_in_stage > SLA[stage]` → render a ⚠ chip on the row and in the
  drawer. This is the "what's gone quiet" signal; it's the single most useful
  operational cue in a pipeline.
- **Call queue** stays, but its definition becomes stage-aware: open stages
  `researched`/`contacted`/`engaged`/`qualified`, intent High/Medium, sorted by stale
  first. Add an **owner filter** so each person works their own queue ("My day").

---

## 5. Drop-in implementation notes (single-file dashboard)

The current file already has the exact patterns needed: an editable field via
`saveEdit(firm, patch)` + `localStorage`, a `<select>` helper `sel(e,f,opts)`, the
score recompute in `view()`, and the drawer/table renderers. Wiring:

### 5a. Config (paste near the top of the `<script>`, beside `COMPDEF`)

```js
const STAGES=[
  {k:"sourced",    label:"Sourced",      cat:"open", p:2,  sla:null},
  {k:"researched", label:"Researched",   cat:"open", p:4,  sla:21},
  {k:"contacted",  label:"Contacted",    cat:"open", p:6,  sla:14},
  {k:"engaged",    label:"Engaged",      cat:"open", p:10, sla:10},
  {k:"qualified",  label:"Qualified",    cat:"open", p:15, sla:30},
  {k:"mandated",   label:"Mandated",     cat:"open", p:40, sla:21},
  {k:"in_market",  label:"In Market",    cat:"open", p:60, sla:60},
  {k:"loi",        label:"Under LOI",    cat:"open", p:80, sla:21},
  {k:"diligence",  label:"Diligence",    cat:"open", p:90, sla:45},
  {k:"closed_won", label:"Closed-Won",   cat:"won",  p:100,sla:null},
  {k:"closed_lost",label:"Closed-Lost / Nurture", cat:"lost", p:0, sla:null},
];
const STAGE=Object.fromEntries(STAGES.map(s=>[s.k,s]));
const OWNERS=["Joe","Sam","Unassigned"];

// revenue × multiple → [low,high]; quality = +0.2 / 0 / -0.2
const EV_BANDS=[ // [maxRevExclusive, lo, hi]
  [1.5e6,0.90,1.10],[3e6,1.00,1.30],[5e6,1.10,1.50],
  [10e6,1.30,1.80],[15e6,1.50,2.00],[Infinity,1.80,2.50]];
function evRange(e){
  const r=e.revenue; if(!r) return null;
  const b=EV_BANDS.find(x=>r<x[0]); const q=qualityAdj(e);
  return [r*(b[1]+q), r*(b[2]+q)];
}
function qualityAdj(e){
  const niche=(e.niche_vertical||"").trim() && e.niche_vertical!=="Generalist";
  const advisory=/CAS|Advisory|Outsourced CFO/.test(e.service_mix||"");
  const growing=/grow|opened|acquired|merger|hiring/i.test(e.growth_signal||"");
  const decline=/declin|wind-?down|closed|retir/i.test(e.growth_signal||"");
  if(decline) return -0.2;
  return (niche||advisory||growing)?0.2:0;
}
function feeRate(ev){ return ev<=2e6?0.10:ev<=5e6?0.08:ev<=10e6?0.06:0.05; }
function forecast(e){
  const ev=evRange(e); if(!ev) return null;
  const mid=(ev[0]+ev[1])/2;
  const fee=Math.max(mid*feeRate(mid),50000);
  const st=STAGE[e.stage]||STAGE.sourced;
  const p=(e.win_probability_override??st.p)/100;
  return {ev_lo:ev[0],ev_hi:ev[1],ev_mid:mid,fee,p,weighted_fee:fee*p,weighted_ev:mid*p};
}
function daysInStage(e){ if(!e.stage_entered_at) return null;
  return Math.floor((Date.now()-Date.parse(e.stage_entered_at))/864e5); }
function isStale(e){ const st=STAGE[e.stage]; const d=daysInStage(e);
  return st&&st.sla!=null&&d!=null&&d>st.sla; }
```

### 5b. Editable stage + owner in the drawer

In the "Outreach workflow" section, swap the `outreach_status` select for a stage
select and add owner + dates. Reuse `sel()`/`saveEdit()`; intercept the stage change
to stamp `stage_entered_at`:

```js
// replace the status <select> with:
`<label class="fld">Stage</label>
 <select class="full" data-f="stage">${STAGES.map(s=>`<option value="${s.k}" ${e.stage===s.k?"selected":""}>${s.label} · ${s.p}%</option>`).join("")}</select>
 <label class="fld">Owner</label>
 <select class="full" data-f="owner">${OWNERS.map(o=>`<option ${e.owner===o?"selected":""}>${o}</option>`).join("")}</select>
 <label class="fld">Expected close</label><input class="full" type="date" data-f="expected_close" value="${esc(e.expected_close)}"/>`
```

In the `data-f` change handler, when `f==="stage"` also
`saveEdit(firm,{stage_entered_at:new Date().toISOString().slice(0,10)})`. When the new
stage is `closed_lost`, reveal a `loss_reason` select.

### 5c. Forecast strip in the drawer

Add a section under "Seller-intent breakdown":

```js
const f=forecast(e);
`<div class="sec"><h3>Deal forecast<span class="cap">Revenue × sector multiple, × stage probability. Assumptions in /docs.</span></h3>
  ${f?`<div class="kv">
    <div>Est. enterprise value</div><div>${fmtMoney(f.ev_lo)}–${fmtMoney(f.ev_hi)}</div>
    <div>Manganelli fee est.</div><div>${fmtMoney(f.fee)}</div>
    <div>Stage / probability</div><div>${STAGE[e.stage].label} · ${Math.round(f.p*100)}%</div>
    <div>Weighted fee</div><div><b>${fmtMoney(f.weighted_fee)}</b></div>
  </div>`:'<div class="muted">Revenue unknown — value not estimated.</div>'}
  ${isStale(e)?`<div class="touch" style="border-color:#a55">⚠ Stale: ${daysInStage(e)} days in ${STAGE[e.stage].label} (SLA ${STAGE[e.stage].sla}d).</div>`:""}</div>`
```

### 5d. Table + new "Forecast" tab

- **Supply table:** replace the `Status` column with `Stage`; add a `⚠`/days-in-stage
  cell and (optionally) `Weighted fee`. Add an **Owner** filter to the toolbar next to
  the band/state filters, and an owner-aware Call queue.
- **New 5th tab "Forecast"** (mirror the Overview pattern): top cards = Gross pipeline,
  Weighted pipeline, Won YTD, #Mandated; then a **by-stage table** (stage · count ·
  gross fee · weighted fee) using the existing `barChart`/table helpers; then **by-owner**
  and a **"Stale & needs attention"** list (open opps where `isStale`).

### 5e. One-time backfill

On load, if a record has no `stage`, derive it from `outreach_status` via the §1.4 map
and set `stage_entered_at` to today (or to `last_touch` if present). Keep
`outreach_status` in the data for one release as a fallback, then drop it.

---

## 6. Assumptions to own (put these in front of Joe/Sam to tune)

1. **Multiples table (§2a)** — set to current small/mid CPA-firm norms; confirm against
   Manganelli's own recent comps and adjust per band.
2. **Quality adjustment** — currently ±0.20× from niche / advisory-mix / growth signals;
   tune the size and the triggers.
3. **Fee schedule (§2b)** — the 10/8/6/5% steps and $50k floor are placeholders for the
   real Manganelli engagement-letter economics.
4. **Stage probabilities (§1)** — anchored to your inputs (2/15/40/60/80); revisit once
   there's enough closed history to calibrate from actual conversion rates.
5. **SLAs (§1)** — staleness thresholds; tune to how the team actually works a deal.

Every number above is a single named constant in §5a so it's auditable and tunable in
one place — same discipline as the "never guessed" data fields.

---

## 7. Where this sits in the larger upgrade

This slice (stage + forecast + ownership + staleness) is the spine. The remaining
slices from the full spec snap onto the fields defined here, in this order:

1. **Activity log** — `activity[]` per firm → "days since last touch", momentum,
   collision-avoidance. (Adds history to the stages defined here.)
2. **Real seller↔buyer matching** — scored, deduped pairs replacing the flat "442",
   each a trackable mini-record (approached/passed/interested/LOI) → the buyer rolodex
   becomes a demand pipeline mirroring this supply pipeline.
3. **Enrichment workflow** — the field-by-field source/confidence/date schema (the other
   candidate for full write-up) that keeps the multiples and narratives honest and
   recomputes intent as facts land.
4. **Management views** — pipeline-by-stage, forecast, throughput-by-owner, stale list.

I can write slice 2 or 3 to the same drop-in depth next.
