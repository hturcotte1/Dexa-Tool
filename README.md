# DEXA Results Workflow — handoff notes

A one-question-at-a-time decision aid for clinicians reviewing DEXA results from the in-basket. It ends with a copyable problem-list line plus either a MyChart message to the patient or a nurse pool message.

## Technical summary

- **One file:** `index.html`. Plain HTML, CSS, and vanilla JavaScript. No build step, no frameworks, no server, no database.
- **External dependency:** Google Fonts only (Public Sans, IBM Plex Mono), with system-font fallbacks.
- **No patient data:** nothing is stored or transmitted. All state lives in page memory and clears on reload.
- **Hosting:** drop the file on any static host (intranet web server, GitHub Pages, Netlify, SharePoint page, etc.).
- **Code layout (inside the `<script>`):**
  - `nodes`: every question, keyed by id. `kind` is `yn`, `multi`, `report`, `frax`, `txend`, or `pdate`. Routes are node ids or `@outcome` ids, or functions returning one.
  - `outcome(id)`: builds each result (title, note, problem-list `plan`, optional `msg` for MyChart, optional `nurse` for the nurse pool).
  - `interval()`: repeat-DEXA interval for below-threshold patients.
  - `lineStart()`: the opening of the problem-list line.
  - `CAL`: shared calcium / vitamin D paragraph appended to MyChart messages.

## Scope

For patients 50 and older with eGFR above 35. Stated at the top of the page.

## Decision logic

1. **DEXA results:** scan date, lowest T-score, optional site (femoral neck, total hip, L1–L4, forearm).
2. **Fragility fracture in the last 6 months?** Yes → *Appointment required* (nurse pool).
3. **Lowest T-score ≤ −2.5?**
   - **Yes** → osteoporosis path (step 5).
   - **No** → step 4.
4. **Scan with TBS?** Collect TBS-adjusted FRAX if yes, standard FRAX if no.
   - FRAX ≥ 20% major osteoporotic or ≥ 3% hip → osteoporosis path (step 5).
   - Below threshold → **Diabetes?** → *Repeat DEXA* at an interval set by `interval()`:
     - Diabetes: 2–3 years, or 1–2 years if T ≤ −2.0 (ADA Standards of Care).
     - No diabetes, by lowest T-score: > −1.0 → 10 yr; −1.0 to −1.49 → 7–10 yr; −1.5 to −1.99 → 3–5 yr; −2.0 to −2.49 → 1–2 yr.
5. **Prior diagnosis of osteoporosis?** No → *Appointment required* (nurse pool).
6. **History of osteoporosis treatment?**
   - **No (untreated):**
     - Prior DEXA available (same facility/machine)? Yes → prior month/year → "Treatment discussed since prior DEXA?" No → "Discussed in the past 2 years?"
     - Not discussed → *Appointment recommended* (MyChart).
     - Discussed → reason not treated (patient preference / not a candidate due to comorbidity) → BMD change (if prior available; otherwise treated as stable):
       - No real change → repeat DEXA 2 yr
       - Mild drop (3–5%) → repeat DEXA 2 yr, "worsened" message
       - Moderate drop (5–10%) → repeat DEXA 2 yr, "worsened" message
       - Large drop (>10%) → *Appointment recommended* (MyChart)
       - "Not a candidate" messages omit the push toward medication.
   - **Yes (assumed drug holiday):** medication completed:
     - Anabolic agent → *Appointment required* (nurse pool) for follow-on antiresorptive.
     - Alendronate, risedronate/ibandronate, or zoledronic acid → last dose month/year → prior DEXA available? → BMD change (unavailable = stable):
       - No real change → continue holiday, repeat 2 yr
       - Mild drop → continue holiday, repeat 1 yr
       - Moderate or large drop → *Appointment recommended* (MyChart) to discuss restarting
       - Flags "longer than typical" when the holiday exceeds ~5 yr (alendronate), ~2 yr (risedronate/ibandronate), or ~3 yr (zoledronic acid).

**BMD change rule:** use "Change vs. Previous %" at the summary sites only (L1–L4, total hip, femoral neck). Count only changes the report marks significant (exceeds the facility's least significant change). If there's no marker, require ≥ 3% at the spine or ≥ 5% at the hip. If the spine and hip disagree, go with the hip.

## Outputs

- **Problem-list line:** `DEXA [date], lowest T-score [x] ([site]), [FRAX or FRAX-TBS] [x]% MOF / [y]% hip, vs prior DEXA [m/yyyy] — [plan]. Reviewed [today].` Fields that don't apply are omitted. Repeat recommendations include a due month/year.
- **Nurse pool message** (appointment-required outcomes): call patient, osteoporosis on DEXA, schedule appointment, labs beforehand. Order BMP, PTH, 25-OH vitamin D, and CBC.
- **MyChart message** (all other outcomes): first person, with calcium guidance (1,000–1,200 mg/day, calcium citrate if supplementing, ≤500–600 mg per dose) and a link to the NIH calcium fact sheet.

## Open items

- Gray-zone handling for near-threshold FRAX (not built; strict 20%/3% cutoffs).
- Possible rule: a moderate drop on two consecutive scans → appointment for untreated patients.
- Final review of all MyChart wording.
- Testing across every path with sample cases.
