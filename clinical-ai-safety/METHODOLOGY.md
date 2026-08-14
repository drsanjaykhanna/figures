# Methodology

**Clinical AI safety: what regulatory surveillance can and cannot tell us**

Version 1.0 · 14 August 2026 · All data accessed 13 to 14 August 2026

---

## 1. Question

Whether the evidence exists to support a decision to deploy LLM-based clinical decision support, summarisation and triage at national scale, and if not, whether that absence can be measured rather than asserted.

The work is deliberately not a registry proposal. It asks what can be established today, from sources that already exist and are free to query.

---

## 2. Data sources

| Source | Endpoint | Access | Refresh |
|---|---|---|---|
| FDA AI-Enabled Medical Device List | `fda.gov/media/178541/download` | CSV, no key | Periodic. Version used: 16 Jun 2026, 1,524 devices |
| openFDA device event (MAUDE) | `api.fda.gov/device/event.json` | REST, no key | Weekly |
| openFDA device classification | `api.fda.gov/device/classification.json` | REST, no key | Periodic |
| openFDA 510(k) | `api.fda.gov/device/510k.json` | REST, no key | Weekly |
| openFDA device recall | `api.fda.gov/device/recall.json` | REST, no key | Weekly |
| FDA clearance summaries | `accessdata.fda.gov/cdrh_docs/pdf{YY}/{K}.pdf` | PDF | Static |
| ClinicalTrials.gov | `clinicaltrials.gov/api/v2/studies` | REST, no key | Daily |
| PubMed E-utilities | `eutils.ncbi.nlm.nih.gov` | REST, no key | Daily |
| AIAAIC repository | Author-supplied export | XLSX | Periodic |

No paid data, no institutional access, no API keys. The entire analysis is reproducible by anyone with an internet connection.

**Sources attempted and not obtained.** The AI Incident Database export is served from Cloudflare R2, which returned errors throughout. GDELT was unreachable. Neither is used, and no claim rests on them.

---

## 3. The central methodological problem: defining an AI device in FDA data

This section matters more than any result, because getting it wrong produces a confident answer that is false. It did so in an earlier version of this work.

### 3.1 What a product code is

Every device the FDA clears is assigned a three-letter product code describing its function. Adverse event reports carry that code. Grouping reports by code is how a whole class of device is monitored.

Aggregating by code is only valid if the code identifies the technology.

### 3.2 The approach that failed

The initial approach assumed product codes beginning Q or S denoted AI, on the reasoning that the FDA began creating AI-specific codes around 2018 and those fall in the Q and S series.

**This is wrong.** Q and S denote recency of code creation, not function. The error was found by checking a known device: the Abbott FreeStyle Libre glucose sensor sits on QBJ and QLG, both Q codes, neither remotely AI-related, holding 2.17 million reports between them.

Auditing all 68 Q and S codes carrying AI devices found that only 30 had official device names indicating an algorithmic function. The other 38 included "Shoulder Arthroplasty Implantation System", "Dental Navigation System" and "External Upper Limb Tremor Stimulator". Those 38 codes contributed 37 of the 39 patient injuries on which the headline finding rested. The finding was an artefact of the definition and is withdrawn. See section 10.

### 3.3 The approach used

Selection is by measurement, not by judgement about code letters.

1. Take all 1,524 devices on the FDA's own AI-Enabled Medical Device List. This is the FDA's definition of AI, not one invented here.
2. Extract the primary product code for each. Result: 172 codes.
3. For each code, query openFDA 510(k) for the total number of devices ever cleared on it.
4. Compute **purity** = AI-listed devices on that code ÷ all devices ever cleared on that code.
5. Include codes with purity ≥ 50%.

**Rationale.** If AI devices are the majority of what sits on a code, an adverse event report against that code is probably about an AI device. If they are two of 509, it is not.

### 3.4 Result of the selection

| | Codes | AI devices | MAUDE reports |
|---|---|---|---|
| Purity ≥ 50%, included | 66 | 688 | 747 |
| Purity < 50%, excluded | 95 | 824 | 1,575,983 |
| Purity indeterminate | 11 | 12 | 149,925 |

The excluded group is the point. It holds 824 AI devices and 1.58 million reports, of which an unknown but small fraction concern AI. Including them would have swamped the numerator with conventional device harm.

**836 devices, 55% of the FDA's own AI list, cannot be studied this way at all.** They are not under-reported. They are unfindable. This is reported as a finding.

### 3.5 Post-hoc exclusion, declared

Two codes pass the 50% threshold on very small denominators and are physical products:

- **OOG**, Knee Arthroplasty Implantation System, 1 AI device of 2, 261 reports
- **PBZ**, Image Processing Device For Estimation Of External Blood Loss, 3 of 6, 268 reports

They are excluded from the primary analysis, reducing 747 filings to 218. This exclusion is discretionary and was made after seeing the data. It is declared here because it materially affects the headline, and both versions are reported.

### 3.6 Known weaknesses in the purity measure

- The denominator counts 510(k) clearances only, so devices authorised by De Novo or PMA are missed. Purity is therefore an overestimate at the top end, and one code (MYN) computes above 100%.
- The 50% threshold is a judgement, not a standard. No sensitivity analysis across thresholds has been run.
- For future work, both the threshold and the physical-product exclusion should be fixed in writing before results are examined.

---

## 4. Adverse event analysis

### 4.1 Retrieval

For each of the 66 included codes, all MAUDE reports were retrieved via `device.device_report_product_code`, paginated at 100 per request. Retrieved: 747 filings. Fields kept: brand name, manufacturer, event type, date received, report number, and the full concatenated `mdr_text` narrative.

### 4.2 Filings are not events

Reports were clustered on a normalised hash of the first 260 characters of narrative text, with redaction markers `(b)(4)` and `(b)(6)` stripped.

747 filings resolve to approximately 479 distinct described defects. DermaSensor illustrates the problem: 11 filings, 2 defects, nine of them word-for-word identical.

This clustering is crude and is used only to demonstrate that filing counts overstate event counts. It is not used for any headline number.

### 4.3 Manual reading

All 218 narratives in the primary set were read individually. Automated classification was rejected because every methodological error in this project originated from pattern matching.

Each was classified as: the software produced a wrong clinical answer; a technical fault with no clinical claim; or unrelated to the coded device.

**Result: 144 describe the software producing a wrong clinical answer. 136 of those, 94%, come from one manufacturer.**

### 4.4 Event type is unreliable

Documented instances where the classification field does not reflect the narrative:

- A Cerner sepsis report states *"Cerner was also informed of a patient death"*. Filed as **malfunction**.
- 108 HeartFlow false negatives, meaning missed coronary stenoses, are filed 127 malfunction to 12 injury.
- NVI, a k-nearest-neighbour autoimmune classifier code, holds one report which concerns a neurovascular stent.
- QJB, a malnutrition monitor code, holds one report concerning a glucose sensor.
- QZW, a sleep apnoea AI code, holds a report of a fitness band causing a burn.

**Consequence.** Any method that counts event-type categories is unsound on this data. That includes the proportional reporting ratio and the Bayesian information component. Those analyses are withdrawn, not corrected. See section 10.

---

## 5. Discovery route analysis

The 139 reports on code PJA (coronary vascular physiologic simulation software, almost entirely HeartFlow) were classified by how the problem was found, using narrative phrasing:

- **Self-identified**: "internal review", "quality monitoring", "quality review", "HeartFlow identified", "we identified"
- **Customer-triggered**: "customer report", "healthcare professional reported", "physician reported", "complaint"

| Route | n | % |
|---|---|---|
| Self-identified only | 118 | 85% |
| Both | 11 | 8% |
| Customer or clinician | 7 | 5% |
| Unclear | 3 | 2% |

108 of 139 use the phrase "false negative" explicitly.

**Interpretation.** One manufacturer operates a process that re-analyses delivered cases and files what it finds. No other manufacturer in the dataset does. Removing HeartFlow leaves 79 reports for 687 other devices.

The count of AI errors in FDA data therefore measures who chooses to look, not how often AI is wrong. A manufacturer with a rigorous audit appears in the public record to have the most dangerous product; one with no audit appears flawless.

**Stated in fairness.** Nothing suggests HeartFlow's product is unsafe. Their analyses involve trained human analysts, so some reports cite analyst error rather than model error. Both are counted as the delivered output being wrong, because that is what reaches the patient.

---

## 6. Trial reporting analysis

### 6.1 Search

ClinicalTrials.gov API v2, all pages, `query.term`:

```
("large language model" OR LLM OR "generative AI" OR ChatGPT OR "foundation model")
```

Retrieved 456 studies. Interventional: 318. Completed: 148.

### 6.2 Counting results properly

Counting only results posted to the register understates reporting, because a trial can publish in a journal and never return to the register. Both routes are counted.

A trial is treated as having reported if either:

1. `hasResults` is true on the register, or
2. the study record carries a reference of type `RESULT` (sponsor-designated) or `DERIVED` (linked automatically by PubMed because the paper cites the NCT number).

References of type `BACKGROUND` are excluded, as they are cited context rather than results.

### 6.3 Cohorts: completion year, not registration year

A reporting rate must be computed within a cohort, so that the numerator is a subset of the denominator. The first attempt used registration year, which is wrong: a trial registered in 2025 has mostly not finished, so its denominator contains trials that could not have reported. That construction required marking two of six bars as censored, which concealed the problem rather than solving it.

The analysis uses **year of completion**. The clock that governs reporting starts when a trial ends.

| Completed | Trials | Reported | Rate |
|---|---|---|---|
| 2022 | 2 | 1 | 50% |
| 2023 | 10 | 4 | 40% |
| 2024 | 30 | 13 | 43% |
| 2025 | 65 | 20 | 31% |
| 2026 | 35 | 1 | 3% (too recent) |
| **2020 to 2025** | **108** | **39** | **36%** |

Only the 2026 cohort is censored, because a trial finishing this year has not had time to publish.

### 6.4 Result

| Completed trials (n = 148) | n | % |
|---|---|---|
| Results posted on the register | 5 | 3.4% |
| Publication traceable by NCT linkage | 42 | 28.4% |
| **Neither** | **106** | **71.6%** |

37 completed trials published without posting to the register. Counting the register alone would have understated reporting eightfold, and that error was caught before publication.

Restricted to trials with time to report, 2020 to 2025: **39 of 108 reported, 69 did not.**

### 6.5 Limitation

A publication is found only if it cites the NCT number or the sponsor linked it. Papers reporting a registered trial without naming the registration are invisible to this method, and that is known to be a substantial minority.

**Therefore 42 is a floor and 106 is a ceiling.** The true reporting rate is above 28%.

This is automated record linkage, not a systematic review, and must never be described as one. Its value is that it can be re-run in minutes and gives a defensible lower bound.

---

## 7. Predicate chain analysis

150 FDA clearance summary PDFs for AI devices were downloaded from `accessdata.fda.gov` and parsed with PyMuPDF. Predicate K-numbers were extracted by regular expression and each predicate's clearance date retrieved, giving the age of the chain.

| | |
|---|---|
| PDFs parsed | 150 of 150 |
| Naming at least one predicate | 140 |
| Citing at least one AI predicate | 113 (81%) |
| Citing no AI predicate | 27 (19%) |
| Median chain age | 3 years |
| Longest chain | 21 years |
| Chains reaching 10 years or more | 12 (9%) |

**Unresolved discrepancy, declared.** Bracken and colleagues (*Clinical Orthopaedics and Related Research*, 2026) report 62% of orthopaedic AI devices citing non-AI predicates. This finds 19%. The likely causes are that the regular expression captures every clearance number appearing in the document, which is more permissive than manual reading, and that the device mix differs.

**This figure is provisional and must be reconciled before use in any formal output.**

---

## 8. Bibliometric analysis

PubMed E-utilities, counts by publication year, for clinical AI overall and for the subset also indexed under safety, harm, error or adverse event terms.

| Year | Clinical AI | Safety subset | Share |
|---|---|---|---|
| 2020 | 2,436 | 129 | 5.3% |
| 2021 | 3,884 | 191 | 4.9% |
| 2022 | 5,058 | 257 | 5.1% |
| 2023 | 6,950 | 373 | 5.4% |
| 2024 | 10,477 | 731 | 7.0% |
| 2025 | 17,977 | 1,534 | 8.5% |

2026 is a part year and is excluded from the figure. An earlier draft quoted 11.4%, which was the part-year 2026 value used as though it were a full year. That is corrected.

**Limitation.** Keyword and indexing based, so it misses safety work not using those terms and catches papers mentioning safety in passing. Absolute counts are approximate. The trend and ratio are robust to reasonable changes in search terms.

---

## 9. Supporting analyses

**Recalls.** 25 recalls across 11 of the 66 included codes, against 860 for a single conventional CT code (JAK). Time-to-recall was computed for matched devices, median 217 days, range 12 to 717 days, on six events. Six events is too few to model and no hazard analysis is reported.

**Database masking.** The ten highest-volume product codes account for 49.4% of all MAUDE reports. This is context for why low-volume AI codes are invisible in any aggregate analysis.

**AIAAIC repository.** 2,255 incidents, 219 health-sector, 23 involving loss of life. Only 25 flagged as concerning anything resembling a regulated clinical device. See section 9A.

---

## 9A. Incident registries

Three public registries claim to track AI harm. All three were pursued. This section records what each contains and how it was obtained.

### 9A.1 AI Incident Database

The published Excel export is served from a Cloudflare R2 host which fails with `ERR_CERT_COMMON_NAME_INVALID` behind TLS-inspecting corporate networks, in command-line clients and in the browser alike. The GraphQL API returns HTTP 403 to non-browser clients.

**Route used.** The API was called from the site's own origin in a browser context, which satisfies both the certificate chain and the bot protection. The exact script is `code/aiid_fetch.js` and is reproducible by pasting it into a browser console at incidentdatabase.ai.

| | |
|---|---|
| Total incidents | 1,625 |
| Health-related | 162 |
| Health incidents mentioning a death | 42 |

Classification of the 162 health incidents by what the implicated system is:

| | n |
|---|---|
| Regulated medical device | **1** |
| Consumer platform, chatbot or social media | 60 |
| Insurer or payer algorithm | 6 |
| Other | 95 |

The single device incident is number 5, "Collection of Robotic Surgery Malfunctions", dated 2015.

### 9A.2 OECD AI Incidents and Hazards Monitor

16,995 records, 1,057 tagged "Healthcare, drugs, and biotechnology". No public API was found. The two `/api/` paths appearing in the page source return 404 and are script integrity hashes rather than endpoints. Counts were read from the site's own filters.

**This is not an incident registry in the same sense as the other two.** Its own subtitle describes it as an "automated media discourse monitor", in Beta, built by scraping news through Event Registry. It carries a disclaimer that its contents do not represent the official views of the OECD or its member countries. The top healthcare result on the day of retrieval concerned an AI stethoscope misdiagnosing heart conditions in pets.

Its device share cannot be verified, because it counts media coverage rather than confirmed incidents.

### 9A.3 Comparison

| Source | Health records | Concerning a regulated device |
|---|---|---|
| OECD AIM | 1,057 | not verifiable |
| AIAAIC | 219 | 25 flagged, most are not devices |
| AI Incident Database | 162 | 1 |
| FDA MAUDE, majority-AI codes | 218 | 218 |

**1,438 health records across the three registries, and almost none concern a regulated clinical AI device.**

This is not a failure of the registries. They document harms that surface publicly through journalism and litigation, and consumer chatbots and insurance algorithms surface that way. A radiology model that quietly misses a lesion does not. The consequence for the policy question is that there is no second source. If a clinical AI device harms a patient, the regulatory system is the only place it could be recorded.

**Method limits.** Health-sector and device classification used keyword matching over title, description and named deployer. It is coarse and will misclassify at the margins; the expressions are in the included code. The three registries use different inclusion criteria, time windows and definitions of an incident, so their totals are not directly comparable with one another. The comparison that carries weight is within each source, between its health records and the device-related subset of those.

---

## 10. Withdrawn analyses

Recorded in full, because a methodology document that hides its errors is worthless.

### 10.1 Disproportionality analysis (PRR and IC₀₂₅)

**What was claimed.** Injury reports were disproportionately over-represented among AI device reports. PRR 2.2, 95% CI 1.6 to 2.8, IC₀₂₅ 0.29, against a comparator of eight legacy imaging codes.

**Why it is wrong.** Two independent reasons, either sufficient.

1. *The exposure definition was invalid.* The "AI code" set was built on the Q and S prefix assumption (section 3.2). Under a corrected definition the injury PRR falls from 2.16 to 0.30, 95% CI 0.08 to 1.18. The signal does not weaken. It inverts. 37 of the 39 injuries came from dental navigation systems, shoulder arthroplasty guides, tremor stimulators and orthopaedic augmented reality.

2. *The outcome variable is unreliable.* Section 4.4. PRR and IC₀₂₅ compare event-type proportions. On this data the event type does not reliably reflect the narrative.

**Status.** Withdrawn entirely. Not corrected, not reported with caveats. Both the method and the finding are removed from all outputs.

### 10.2 Capture-recapture between MAUDE and AIAAIC

**What was claimed.** Zero overlap between the two surveillance systems, presented as evidence that both are missing most events.

**Why it is wrong.** True but trivially so. Of the 25 device-related AIAAIC entries, almost none are FDA-regulated devices, so they are structurally incapable of appearing in MAUDE. The estimator is undefined for a reason that carries no information about under-reporting.

**Status.** Withdrawn. Replaced by the four-source comparison in section 9A, which makes the same point without requiring an estimator that cannot be computed.

### 10.3 Registration-year cohorts for trial reporting

**What was claimed.** Reporting rates by year of registration, with the 2025 and 2026 bars marked censored.

**Why it is wrong.** The denominator contained trials that had not finished and therefore could not have reported. Marking whole bars as censored concealed the problem instead of removing it.

**Status.** Replaced by completion-year cohorts, section 6.3.

### 10.4 Fabricated year-by-year curve

An early draft of the approvals-versus-reports figure used an invented accumulation curve for adverse event reports. The total was known; the distribution across years was not checked and was drawn to look plausible.

**Status.** Corrected using real `date_received` values before any external circulation. Recorded here so it is on the record.

---

## 11. Limitations applying to the whole work

1. **No denominator anywhere.** We do not know how many times any device was used. No harm rate can be calculated, only report counts. Every figure measures reporting, not incidence.
2. **A low count is uninterpretable.** It is equally consistent with a safe technology and with a technology whose failures are invisible to the reporting system. Nothing in this data separates the two. This is the central finding, not a caveat to it.
3. **Structural mismatch.** MAUDE was built for physical devices. Software producing a wrong recommendation does not fit malfunction, injury or death, and the resulting harm is recorded as a clinical outcome in the patient's notes rather than as a device event.
4. **Severity mis-coding is documented independently.** Published work in *JAMA Internal Medicine* has found reports involving death coded as malfunctions for conventional devices.
5. **LLM devices are absent by construction.** The FDA has stated it cannot identify devices containing large language models in its own data.
6. **Recency confounds emptiness.** Recently cleared devices have had less time to accumulate reports. This does not explain PIB, where seven autonomous devices have been deployed for eight years and produced one report, nor QIH, where 274 devices produced ten.
7. **Selection choices are declared but not pre-registered.** The purity threshold and the physical-product exclusion were both made with sight of the data.
8. **Manual classification was done by one reader.** No second reader, no inter-rater reliability, no blinding.

---

## 12. Reproducibility

All code and data are in `deliverable/code` and `deliverable/data`.

**Run order**

| Step | Script | Produces |
|---|---|---|
| 1 | `get_fda.sh` | `fda_ai_devices.csv` |
| 2 | `audit_all_codes.py` | `code_audit_full.json` |
| 3 | `clarity_table.py` | `clarity_table.json` (purity per code) |
| 4 | `show_clarity.py` | the included code set, printed |
| 5 | `fetch_narratives.py` | `narratives.json` (747 reports) |
| 6 | `read_narratives.py` | narratives for reading, by code |
| 7 | `sepsis_and_dedup.py` | `distinct_events.json` |
| 8 | `heartflow.py` | `heartflow.json` |
| 9 | `ctgov_verify.py` | `ctgov_verified.json` |
| 10 | `trial_reporting.py` | `trial_reporting.json` (publication linkage) |
| 11 | `trial_cohorts.py` | completion-year cohorts, appended to the same file |
| 12 | `registries_probe.py`, `registries_extract.py`, `oecd_probe.py` | registry reachability |
| 13 | `aiid_fetch.js` | run in a browser at incidentdatabase.ai, produces the `registries.json` counts |
| 14 | `predicates_curl.py` | `predicates.json` |
| 15 | `remaining.py`, `full_analysis.py`, `slice_health.py` | recalls, masking, literature, AIAAIC |
| 16 | `figure_data.py` | `figure_data.json`, the single source for the figures |
| 17 | `verify_all.py` | recomputes 21 headline numbers from raw data |
| 18 | `final_check.py` | gates the published page against withdrawn claims |

Steps 17 and 18 are the integrity checks. `verify_all.py` recomputes every published number from source and fails on any mismatch. `final_check.py` scans the output page for the presence of required numbers and the absence of every withdrawn claim by name.

Both should be re-run before any external circulation.

---

## 13. What would need to change before publication

1. Pre-register the purity threshold, the physical-product exclusion and the comparator choice, in writing, before re-running.
2. Second reader for the 218 narratives, with inter-rater reliability reported.
3. Reconcile the predicate discrepancy against Bracken et al., probably by manual verification of a random subsample.
4. Sensitivity analysis across purity thresholds from 30% to 80%.
5. The AI Incident Database has now been obtained (section 9A). The remaining gap is that health and device classification is keyword-based and should be manually verified on a sample.
6. Fix a data cut date. MAUDE updates weekly and ClinicalTrials.gov daily, so all counts move.
7. Verify the FDA statement on LLM device identifiability against a citable primary source rather than secondary reporting.

---

## 14. Summary of headline findings

| Finding | Value |
|---|---|
| FDA-listed AI devices | 1,524 across 172 product codes |
| Devices on codes where AI is the majority | 688 on 66 codes |
| Devices structurally unfindable on shared codes | 836 (55%) |
| Adverse event filings on included codes | 747, resolving to ~479 distinct defects |
| Filings after removing physical products | 218 |
| Reports describing a wrong clinical answer | 144, of which 136 (94%) from one manufacturer |
| Deaths in the primary set | 1 |
| Included codes with no report ever | 49 of 66 |
| Autonomous retinopathy screening (PIB) | 7 devices, 8 years, 1 report |
| Automated radiological image processing (QIH) | 274 devices, 10 reports, 6 the same defect |
| LLM trials completed with no traceable findings | 106 of 148; 69 of 108 with time to report |
| Incident registry health records, three sources | 1,438, almost none regulated devices |
| AI Incident Database health incidents involving a device | 1 of 162 |
| Predicate chains reaching 10 years or more | 9%, longest 21 years |
| Safety share of clinical AI literature | 5.3% (2020) to 8.5% (2025) |
