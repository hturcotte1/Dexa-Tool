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

1. **DEXA results:** scan month/year (month buttons; year buttons for the current and prior year, dropdown for earlier), lowest T-score, optional site (femoral neck, total hip, L1–L4, forearm).
2. **Fragility fracture in the last 6 months?** Yes → *Appointment required* (nurse pool).
3. **Lowest T-score ≤ −2.5?**
   - **Yes** → osteoporosis path (step 5).
   - **No** → **Ever had a fragility fracture?** Yes → choose site(s): hip (right/left), spinal compression (lumbar/thoracic/both), distal radius (right/left), or other → month/year of each fracture → osteoporosis path (step 5); No → **History of osteoporosis treatment?** Yes → drug-holiday branch (step 6, Yes), skipping FRAX; No → step 4.
4. **Scan with TBS?** Collect TBS-adjusted FRAX if yes, standard FRAX if no.
   - FRAX ≥ 20% major osteoporotic or ≥ 3% hip → osteoporosis path (step 5).
   - Below threshold with osteopenia (T ≤ −1.0) → **Is there a prior DEXA available for comparison?** Yes → prior month/year → BMD change: improved / no real change / mild drop → usual interval; moderate drop → repeat in 1–2 yr; large drop (>10%) → *Appointment recommended* (MyChart). Then:
   - Below threshold → **Diabetes?** → *Repeat DEXA* at an interval set by `interval()`:
     - Diabetes: 2–3 years, or 1–2 years if T ≤ −2.0 (ADA Standards of Care).
     - No diabetes, by lowest T-score: > −1.0 → 10 yr; −1.0 to −1.49 → 7–10 yr; −1.5 to −1.99 → 3–5 yr; −2.0 to −2.49 → 1–2 yr.
5. **Prior diagnosis of osteoporosis?** No → *Appointment required* (nurse pool).
6. **History of osteoporosis treatment?**
   - **No (untreated):**
     - Prior DEXA available (same facility/machine)? Yes → prior month/year → "Treatment discussed since prior DEXA?" No → "Discussed in the past 2 years?"
     - Not discussed → *Appointment recommended* (MyChart).
     - Discussed → reason not treated (patient preference / not a candidate due to comorbidity) → BMD change (if prior available; otherwise treated as stable):
       - Improvement (mild 3–5% / moderate 5–10% / large >10%) or no real change → repeat DEXA 2 yr
       - Mild drop (3–5%) → repeat DEXA 2 yr, "worsened" message
       - Moderate drop (5–10%) → repeat DEXA 2 yr, "worsened" message
       - Large drop (>10%) → *Appointment recommended* (MyChart)
       - "Not a candidate" messages omit the push toward medication.
   - **Yes (assumed drug holiday):** medication completed:
     - Anabolic agent → which (teriparatide / abaloparatide / romosozumab) → last dose month/year → **Completed two years (romosozumab: full 12-month course)?** No → how long treated + why stopped → **Zoledronic acid (Reclast) received or scheduled?** (Received/Scheduled → exact date) → **prior DEXA available?** → comparison incl. improvement level (recorded, does not change the plan) → follow-on noted, repeat DEXA 2 yr (MyChart). Neither → repeat DEXA 2 yr (MyChart) with an on-screen NOTE that follow-on Reclast is very important; contact patient if necessary.
     - Alendronate (Fosamax), risedronate (Actonel), ibandronate (Boniva), or zoledronic acid (Reclast) → last dose month/year → prior DEXA available? → BMD change (unavailable = stable):
       - Improvement (mild 3–5% / moderate 5–10% / large >10%) or no real change → continue holiday, repeat 2 yr
       - Mild drop → continue holiday, repeat 1 yr
       - Moderate or large drop → *Appointment recommended* (MyChart) to discuss restarting
       - Flags "longer than typical" when the holiday exceeds ~5 yr (alendronate), ~2 yr (risedronate, ibandronate), or ~3 yr (zoledronic acid).

**BMD note:** the comparison screen has an optional free-text box for mixed results; its text is appended to the problem-list line as "BMD note: ...".

**BMD change rule:** use "Change vs. Previous %" at the summary sites only (L1–L4, total hip, femoral neck). Count only changes the report marks significant (exceeds the facility's least significant change). If there's no marker, require ≥ 3% at the spine or ≥ 5% at the hip. If the spine and hip disagree, go with the hip.

## Outputs

- **Problem-list line:** `DEXA [month/year], lowest T-score [x] ([site]), [FRAX or FRAX-TBS] [x]% MOF / [y]% hip, vs prior DEXA [m/yyyy] — [plan]. Reviewed [today].` Fields that don't apply are omitted. Repeat recommendations include a due month/year.
- **Nurse pool message** (appointment-required outcomes): call patient, osteoporosis on DEXA, schedule appointment, labs beforehand. Order BMP, PTH, 25-OH vitamin D, and CBC.
- **MyChart message** (all other outcomes): first person, with calcium guidance (1,000–1,200 mg/day, calcium citrate if supplementing, ≤500–600 mg per dose) and a link to the NIH calcium fact sheet.

## Sleep study workflow (`sleep.html`)

A second tool on the same site, built the same way (one file, no build step, no patient data). The two pages link to each other from their headers.

**Steps**

1. **Study type:** home sleep study, in-lab PSG, or split-night (diagnostic portion only).
2. **Sleep clinic referral placed with the order?** Decides who owns treatment.
3. **Results:** study month/year, AHI, supine AHI (optional), central AHI, SpO2 nadir, minutes <89%, BMI. Labels match the Nomad report.
4. **Outcomes checklist**, pre-checked by rule, every box editable.
5. **Output:** problem-list line plus a MyChart message, or a nurse pool message if *Visit with PCP* is checked.

**Calculated values:** severity (<5 normal, 5–14.9 mild, 15–29.9 moderate, ≥30 severe); central-predominant (central AHI > 50% of AHI); positional (supine AHI ≥ 2× AHI); hypoxemia (≥5 min <89%, shown as a flag and an on-screen note); obesity (BMI ≥30).

**Pre-check rules** (`OUT` array in the script; its order is also the paragraph order of the MyChart message):

| Outcome | Pre-checked when |
| --- | --- |
| Normal, no further testing | AHI <5 |
| Normal home study → in-lab PSG | Never (shown for home studies only) |
| Wait for sleep clinic | OSA and referred |
| Offer auto-CPAP (+ 1–3 month follow-up) | OSA, not referred, not central-predominant |
| Titration study | OSA, not referred, central-predominant |
| Offer sleep clinic referral | Not referred, central-predominant |
| Oral appliance | Mild OSA |
| Positional therapy | Positional |
| Weight management | BMI ≥30 (tirzepatide line added if moderate/severe and no central component: central AHI <5 and not central-predominant) |
| Visit with PCP | Never; switches output to a nurse pool message |

Every OSA message gets fixed drowsy-driving and alcohol/sedative lines. On-screen notes appear if auto-CPAP is checked for a central-predominant study, or when any hypoxemia is present (consider oximetry on CPAP).

**Open items:** confirm PSG report labels match the Nomad terms; nurse pool default visit length and timeframe.

## Open items (DEXA)

- Gray-zone handling for near-threshold FRAX (not built; strict 20%/3% cutoffs).
- Possible rule: a moderate drop on two consecutive scans → appointment for untreated patients.
- Final review of all MyChart wording.
- Testing across every path with sample cases.
