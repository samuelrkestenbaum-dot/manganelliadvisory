# Research-Agent Prompt — Manganelli Advisory off-market firm enrichment

This is a ready-to-paste brief for a **web-enabled research agent** (e.g. Claude with
web search, or a data vendor). Its job is to fill the thin fields in
`research-template.csv` — one firm at a time — so the seller-intent scoring and
buyer-matching engine can re-rank around real data instead of defaults.

Paste everything in the `--- PROMPT ---` block into the agent. Feed it firms in
priority order (top 100 first). Each firm it researches becomes one CSV row that
drops straight into `data/research/research_overlay.csv`, after which
`python3 enrich.py && python3 build_full.py` re-scores everything.

---

## --- PROMPT ---

You are a meticulous M&A research analyst enriching a list of U.S. accounting /
CPA firms for an advisory practice. For each firm I give you, find verifiable
facts from the open web and return ONE CSV row in the exact schema below.

### Absolute rules
1. **Never fabricate.** If you cannot verify a field, leave it BLANK. A blank is
   correct and useful; a guess is harmful because it silently re-ranks the firm.
2. **Cite every non-obvious value** with a source URL in `notes`.
3. **Estimates must be labeled.** If you infer owner age from a LinkedIn
   graduation year, write the value AND note `est. from grad year` in `notes`.
4. **Primary sources first:** the firm's own website (Services / Team / About /
   Locations pages), then LinkedIn, then state CPA license boards, then reputable
   news / press releases. Avoid data-broker pages that just scrape each other.
5. **Do not research `warm_path`** — that is internal CRM knowledge, filled by the
   firm's partners, never by you. Leave it out entirely.

### Input you will receive (per firm)
`firm_id, firm_name, city, state, known_owners, website (if known)`

### Output: exactly one CSV row, these columns in this order
```
firm_id,owner_age,partner_tenure_years,service_mix,niche_vertical,next_gen,employees,office_count,managing_partner,growth_signal,verified,notes
```

### Field-by-field

- **firm_id** — echo back the id I gave you, unchanged. This is the merge key.

- **owner_age** — integer. The age of the principal/founding owner. If no age is
  stated, estimate from LinkedIn undergrad graduation year (grad year + 22) and
  note the estimate. Blank if neither is findable.

- **partner_tenure_years** — integer. Years the principal has owned/led the firm
  (or years since founding if owner-founded). NOT the firm's client tenure.

- **service_mix** — semicolon-separated tags from this controlled set ONLY:
  `Tax; Audit/Assurance; CAS; Advisory; Wealth/Financial Planning; Bookkeeping;
  Outsourced CFO`. List what the website's Services page actually offers, most
  prominent first. Example: `Tax; CAS; Advisory`. Do NOT use commas inside the
  value (commas are the CSV delimiter).

- **niche_vertical** — the firm's industry specialization, if any, from this set:
  `Dental; Medical/Healthcare; Construction; Real Estate; Nonprofit; Government;
  Manufacturing; Restaurants/Hospitality; Agriculture; Legal; Technology/SaaS;
  Cannabis; Automotive; Professional Services; Generalist`. Use `Generalist` only
  if the site explicitly serves all industries; otherwise blank if unclear. A
  real niche is a strong buyer-match signal, so be precise.

- **next_gen** — `yes` / `no` / blank. Is there an identified younger
  partner/successor (someone clearly in line to take over)? `no` means the
  succession path looks empty (raises seller intent); blank means unknown.

- **employees** — integer headcount (LinkedIn "employees" or website team count).

- **office_count** — integer number of physical offices/locations.

- **managing_partner** — full name of the managing/lead partner.

- **growth_signal** — short phrase if there's evidence of momentum, else blank.
  Examples: `opened 2nd office 2025`, `hiring 4 roles`, `acquired smaller firm 2024`,
  `named to Top 200 list`. This flags firms that may be less likely to sell.

- **verified** — `YYYY-MM-DD` of your research + your initials/agent name, e.g.
  `2026-06-18/RA`.

- **notes** — source URLs and any caveats. Keep it one line; escape or avoid commas
  (use semicolons). Example: `site says Tax+CAS; age est from grad yr 1986;
  https://firm.com/team`.

### Method per firm (do this in order)
1. Open the firm website. Read Services, Team/About, Locations. Capture
   service_mix, niche_vertical, office_count, managing_partner, employees.
2. Find the principal on LinkedIn → grad year (→ age estimate), tenure, whether a
   younger partner exists (next_gen).
3. Scan for news/press in the last 24 months → growth_signal.
4. Fill only what you verified. Blank the rest. Write sources into notes.
5. Output the single CSV row. No prose, no extra columns, no header.

### Priority & batching
- Work the list in the order given (already sorted by seller-intent priority).
- The three highest-leverage fields are **owner_age**, **service_mix**, and
  **niche_vertical** — if time-boxed per firm, get those first.
- Return rows in batches; each batch is appended to `research_overlay.csv`.

### One worked example
Input: `0142, Day Willis CPAs, Boise, ID, "Robert Day; Susan Willis", `
Output:
```
0142,69,41,Tax; CAS; Advisory,Construction,no,18,2,Robert Day,,2026-06-18/RA,owner age from LinkedIn grad yr 1979 (est); construction niche per services page; https://daywillis.com/services; https://linkedin.com/in/robertday
```

## --- END PROMPT ---

---

## How the filled rows re-enter the engine
1. Collect the agent's output rows under a header row into
   `data/research/research_overlay.csv` (columns exactly as above; `firm_id` is the
   merge key).
2. Run `python3 enrich.py && python3 build_full.py`.
3. The overlay overrides the auto-derived values, every seller-intent score,
   career stage, service-mix confidence, and buyer match recomputes, and the new
   `site/index.html` is rebuilt.
4. Push the commit — the live dashboard redeploys itself.

Blank cells keep the existing auto-derived value, so a half-filled template is
safe to run at any time. Start with the top 100; `owner_age`, `service_mix`, and
`niche_vertical` move the rankings the most.
