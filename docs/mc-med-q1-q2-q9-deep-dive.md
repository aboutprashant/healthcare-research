# Deep Dive: Q1 + Q2 (privileged waveforms, leakage audit) and Q9 (pulse-oximetry bias)

*Companion to [`mc-med-open-research-questions.md`](mc-med-open-research-questions.md). Written October 2026.*

> **Method and caveats.** These designs are built from web searches of the papers linked under Sources. physionet.org, arXiv and PMC full texts were blocked from the research sandbox, so a few dataset details (waveform channel format, which lab tests exist, label definitions in MC-BEC and CAREBench) **must be confirmed once you have credentialed access**. Each design has an explicit **decision gate** for that. Any numbers written as targets are pre-registration proposals, not predictions of results.

---

## Part 0 — How the three questions fit together

```
                   ┌──────────────────────────────────────────────┐
                   │  Q2  Leakage audit                            │
                   │  "Is the model reading the patient,           │
                   │   or reading the clinicians?"                 │
                   │  → defines an honest evaluation regime:       │
                   │    predictions made BEFORE clinicians act     │
                   └───────────────┬──────────────────────────────┘
                                   │ evaluation backbone
                   ┌───────────────▼──────────────────────────────┐
                   │  Q1  Privileged-waveform distillation         │
                   │  "Teach an EHR-only model with waveforms      │
                   │   it will never see at deployment"            │
                   │  → shows value exactly where Q2 says          │
                   │    action-based shortcuts fail                │
                   └──────────────────────────────────────────────┘

                   ┌──────────────────────────────────────────────┐
                   │  Q9  Pulse-oximetry bias (separate paper)     │
                   │  Reuses: waveform pipeline (Q1) +             │
                   │          order-timestamp machinery (Q2)       │
                   │  Gate: are there enough arterial blood gases? │
                   └──────────────────────────────────────────────┘
```

**The recommended sequence:**
- **Paper 1 = Q2 + Q1**, one combined paper. Q2 alone is publishable; Q1 needs Q2 to be convincing.
- **Paper 2 = Q9**, if the week-1 data check passes. If it doesn't, Q9 moves to fallback datasets.

---

## Part 1 — Q2: the care-process leakage audit

### 1.1 The idea in plain language

In a hospital record, much of the data is not a measurement of the patient. It is a trace of **what clinicians did because they were already worried**: ordering a lactate, paging the ICU, starting a vasopressor, putting the patient on a continuous monitor. A model that sees those traces can predict "deterioration" very accurately without ever noticing anything the clinicians hadn't. It is reading over their shoulders, not adding new information.

### 1.2 Evidence that this is real (and how big it can be)

| Study | Finding |
|---|---|
| [Agniel, Kohane & Weber, *BMJ* 2018](https://www.bmj.com/content/361/bmj.k1479.abstract) | Across 272 lab tests and 669,452 patients, **just the existence of a test order** was associated with survival for 233 tests (86%). **When** a test was ordered predicted 3-year survival better than its **result** for 118 of 174 tests (68%). |
| [Beaulieu-Jones et al., *npj Digit Med* 2021](https://pubmed.ncbi.nlm.nih.gov/33785839/) | On 42.9M admissions, models using **only clinician-initiated billing charges** from day 1 reached AUC **0.89** for in-hospital mortality, 0.82 for prolonged stay and 0.71 for readmission, close to full-EHR benchmarks. Title: *"standing on, or looking over, the shoulders of clinicians?"* |
| [Kamran et al., *NEJM AI* 2024](https://healthitanalytics.com/news/epic-sepsis-model-predictions-may-have-limited-clinical-utility) | The **Epic Sepsis Model** looked much worse when evaluated only on predictions made **before** treatment signals (antibiotics, fluids, blood cultures, lactate). Most of its apparent skill came after clinicians had already recognized sepsis. |
| [The early warning paradox, *npj Digit Med* 2025](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11790821/) | Existing early-warning systems trigger interventions that **prevent** the very outcomes models are trained to predict, so standard evaluation can be paradoxical. |
| [van Geloven et al. (*Ann Intern Med* 2025; preprint)](https://arxiv.org/pdf/2402.17366) | Treatments given after the prediction time blur what a risk model means. Models meant to support decisions need **"prediction under interventions"**. |

**What hasn't been done:** nobody has quantified these effects on a **multimodal ED dataset**, where even *having* waveforms is a clinician decision (monitoring), and where we can separate **physiology** from **process** with minute-level timestamps. MC-MED makes that possible.

### 1.3 A taxonomy of leakage channels in ED data

This taxonomy is itself a paper contribution.

| Channel | Example in MC-MED | Why it leaks |
|---|---|---|
| **L1 Action features** | Orders, medication administrations (vasopressors, antibiotics, oxygen), consults | They encode clinician concern |
| **L2 Measurement-process features** | Whether or when a lab is ordered; how often vitals are charted; **being on continuous monitoring** | The decision to measure is itself informative (Agniel 2018) |
| **L3 Action-defined labels** | Outcome = "ICU admission" or "vasopressor started" | The label *is* a clinician decision |
| **L4 Treatment-averted outcomes** | Early fluids or oxygen prevent the decompensation the model was meant to predict | Interventions change the outcome (early warning paradox) |
| **L5 Timestamp leakage** | Using result times vs order times wrongly; documentation lag; rounding | Future information enters the features |
| **L6 Post-recognition evaluation** | Scoring predictions made after clinicians already acted | Inflates apparent early-warning value (Kamran 2024) |

### 1.4 Experiments

| ID | Experiment | What it shows |
|---|---|---|
| **E1** | **Feature-tier ladder:** T0 physiology only (charted vitals, numerics, waveforms) → T1 + triage (ESI, chief complaint, demographics) → T2 + lab *values* at result time → T3 + *measurement-process* features (order existence and timing, charting frequency, monitoring indicator) → T4 + *actions* (meds, orders, consults) | How much performance each channel buys |
| **E2** | **Process-only model** (T3 + T4 features with **no values**), the ED analogue of Beaulieu-Jones | Share of full-model skill that comes from clinician behaviour alone |
| **E3** | **Pre-recognition evaluation:** score only predictions made **before the first escalation action**. The escalation-action list should be pre-specified with a clinician. | The Kamran-style drop |
| **E4** | **Label variants:** *physiology-defined* deterioration (sustained hypotension, hypoxaemia, tachycardia thresholds from monitor numerics) vs *action-defined* (ICU admission, vasopressors, ventilation) | Whether model rankings change with the label definition |
| **E5** | **Monitoring shortcut:** how much does "on continuous monitoring (yes/no) + time since arrival" alone predict? Does the waveform model's advantage survive **within** the monitored-only stratum? | Separates the selection effect from real signal |
| **E6** *(stretch)* | **Interventions as competing events / treatment-naive risk:** censor at first intervention, or reweight with inverse probability of treatment | First step toward "prediction under interventions" |

**Pre-registered hypotheses:**
- **H2a.** A process-only model (E2) recovers most of the full model's AUROC on action-defined labels. I propose a pre-registered threshold of ≥80%; adjust it with a clinician before you look at the data.
- **H2b.** Under pre-recognition evaluation (E3), AUROC/AUPRC of T3–T4 models drops more than that of T0–T2 models.
- **H2c.** On physiology-defined labels (E4), the gap between process-heavy and physiology-heavy models narrows. *That* is the regime where waveforms (Q1) should matter.

### 1.5 Deliverables
1. A **leakage card**: a one-page checklist reporting L1–L6 for any ED/EHR prediction paper.
2. Open-source **tiered feature builders** and a **pre-recognition evaluation harness**, in MEDS format if feasible.
3. Results on MC-MED, with a **replication on MIMIC-IV-ED** for the tiers that exist there.

---

## Part 2 — Q1: privileged-waveform distillation

### 2.1 The idea in plain language

MC-MED has something almost no other hospital stores: **continuous bedside waveforms** (ECG lead II, pulse oximeter pleth, respiration). If you build a model that *needs* waveforms, it can only be deployed in hospitals that archive them, which is very few. Instead:

1. Train a **teacher** that sees waveforms + EHR.
2. Train a **student** that sees **only** routinely stored data (triage, charted vitals, labs), but is trained to **mimic the teacher**.
3. Deploy the student anywhere, and test it externally on **MIMIC-IV-ED**, which has no ED waveforms.

This is **learning using privileged information (LUPI)**: extra information available at training time but not at test time. [Lopez-Paz, Bottou, Schölkopf & Vapnik (ICLR 2016)](https://arxiv.org/abs/1511.03643) unified it with knowledge distillation as **"generalized distillation"**.

### 2.2 Why it can work (intuition)
- The teacher's **soft predictions** carry information about the patient's latent physiological state: "this one looks fine on paper but the PPG perfusion is collapsing". Hard 0/1 labels don't carry that. The student learns *which* EHR patterns are reliable warning signs and which are noise.
- In the theory behind LUPI, privileged information can make learning **more sample-efficient**. Practically, distillation acts as a **data-dependent regularizer** that points the student toward physiology rather than process shortcuts. That's the link to Q2.

### 2.3 The new methodological twist: privileged information that is *missing not at random*

Standard LUPI assumes privileged information exists for **all** training examples. In MC-MED, waveforms exist only for patients clinicians **chose to monitor**: sicker patients, certain chief complaints. So:
- the teacher only learns on a *selected* subpopulation;
- the student must be applied to **everyone**, including unmonitored patients;
- the monitoring decision itself is a leakage channel (L2).

**Open problem:** how to distill from a privileged modality whose presence is a clinical decision, without teaching the student "monitored = sick". I didn't find prior work addressing this; it's the core novelty claim (verify with a final literature check before writing).

### 2.4 Candidate methods (simple → advanced)

| Method | How it works | Why include it |
|---|---|---|
| **M0** EHR-only | Standard training on EHR features | Baseline |
| **M1** Modality dropout | Multimodal model that sees EHR-only inputs part of the time | The current default for "missing modalities" |
| **M2** **Waveform-derived auxiliary targets** | Student also predicts teacher-side physiological summaries (heart-rate variability, PPG amplitude and perfusion proxies, respiratory-rate variability) from EHR inputs | Simplest LUPI; interpretable |
| **M3** **Soft-label distillation** | Loss = (1−λ)·CE(y, student) + λ·KL(teacher_T ‖ student_T), on monitored visits | Classic generalized distillation |
| **M4** **Representation alignment** | Student predicts the teacher's waveform embedding ("imagine the waveform") | Transfers richer information than soft labels |
| **M5** **Selection-aware distillation** *(novel)* | Weight the distillation loss by the inverse propensity of being monitored (estimated from triage data), and/or add an adversarial penalty so the student's representation can't predict monitoring status | Directly addresses the MNAR selection problem |
| **Upper bound** | Teacher evaluated on monitored visits | Shows how much information is available to distil |

**Related work to position against:**
- [MIND (TMLR 2025)](https://www.arxiv.org/abs/2502.01158): distils unimodal teachers into a *multimodal* student for compression. The opposite direction.
- [MMPKD (2025)](https://arxiv.org/abs/2508.06558v1): text or tabular teacher → chest X-ray student.
- [Variational KD for CXR with EHR](https://arxiv.org/pdf/2103.10825): EHR privileged during training.
- [LuPIET](https://astro.paperswithcode.com/paper/improving-text-based-early-prediction-by): later clinical text as privileged information for early prediction.
- ECG→PPG cross-modal distillation for wearables.

**The gap:** none of these do *bedside physiological waveforms → deployable EHR student* under *clinician-selected* privileged data, with external validation.

### 2.5 Study design

| Element | Specification |
|---|---|
| **Cohort** | Adult MC-MED visits. Exclusions decided before modelling (e.g., very short stays, missing triage). |
| **Prediction times** | Triage; then rolling hourly. Primary analysis = **pre-recognition** predictions (from Q2, E3). |
| **Outcomes** | (a) physiology-defined deterioration within *k* hours (from monitor numerics); (b) action-defined deterioration (ICU admission, vasopressors, ventilation, death); (c) admission. Align definitions with MC-BEC/CAREBench after reading their task code. |
| **Student inputs** | Fields that also exist in **MIMIC-IV-ED**: triage vitals, ESI, chief complaint, charted vitals, labs, home meds, demographics. Harmonize once. |
| **Teacher inputs** | Student inputs + waveforms. Start with **handcrafted/pretrained features** (HRV, PPG morphology, respiratory rate); move to learned encoders later. |
| **Splits** | Patient-level; **temporal** (train Sept 2020–2021, test 2022); external test on MIMIC-IV-ED (student only). |
| **Metrics** | AUROC, **AUPRC**, calibration (ECE, slope/intercept), decision curves (net benefit), **alerts per true event at fixed alert budgets**, lead time, [CAREBench stability](https://arxiv.org/html/2510.14286v1); subgroup metrics; patient-level bootstrap CIs; 5 seeds. |
| **Key analyses** | Student-vs-EHR-only gain overall, **for unmonitored visits**, **pre- vs post-recognition**, and **externally**. Does distillation reduce the student's reliance on process features (Q2 tiers)? |

**Pre-registered hypotheses:**
- **H1a.** M3/M4/M5 students beat M0 and M1 on pre-recognition AUPRC on MC-MED's temporal test set.
- **H1b.** The improvement carries over to **MIMIC-IV-ED**, even if smaller.
- **H1c.** Selection-aware distillation (M5) beats naive distillation (M3/M4) on **unmonitored** visits and reduces the student's dependence on the monitoring indicator.
- **H1d.** Distilled students are **more temporally stable** (fewer alert flips) at equal sensitivity.

### 2.6 Threats to validity and mitigations

| Threat | Mitigation |
|---|---|
| Waveform coverage too low or biased | Week-1 check; report coverage by acuity; M5 is designed for exactly this |
| Label mismatch MC-MED ↔ MIMIC-IV-ED | One harmonized label spec; sensitivity analyses on alternative definitions |
| Gains are tiny | The pre-recognition and unmonitored subsets are where gains should concentrate; report effect sizes with CIs. A clean negative result plus the Q2 audit is still a paper. |
| Waveform data volume | Precompute windowed features or embeddings once; store as Parquet/Zarr |
| COVID-era shift | Temporal split plus a robustness section |

### 2.7 What Paper 1 looks like

| Section | Content |
|---|---|
| **Title (draft)** | *Learning from monitors you won't have: privileged-waveform distillation for deployable, leakage-audited ED deterioration prediction* |
| **Contributions** | (1) the L1–L6 leakage taxonomy and audit on a multimodal ED dataset; (2) selection-aware privileged distillation; (3) external validation of a waveform-trained, EHR-deployable model; (4) open code and harmonized MC-MED ↔ MIMIC-IV-ED tasks |
| **Main figures** | Feature-tier ladder (E1); pre- vs post-recognition performance (E3); student gains by monitored/unmonitored × internal/external; stability vs sensitivity curves |
| **Venues** | ML4H, CHIL, MLHC; *npj Digital Medicine*, *JAMIA*, *JBI* (check current deadlines) |

---

## Part 3 — Q9: pulse-oximetry bias, using the raw PPG

### 3.1 The problem in plain language

A pulse oximeter shines red and infrared light through the finger and estimates oxygen saturation (**SpO2**) from how much of each is absorbed. Melanin also absorbs light, and devices were largely calibrated on lighter-skinned volunteers. So in patients with darker skin, oximeters can **overestimate** oxygen. A patient can be dangerously low (**occult hypoxaemia**) while the monitor looks reassuring.

- [Sjoding et al., *NEJM* 2020](https://psnet.ahrq.gov/issue/racial-bias-pulse-oximetry-measurement): among patients with SpO2 **92–96%**, true arterial saturation (SaO2) was **<88%** in **11.7% of Black vs 3.6% of White** patients (University of Michigan cohort, 10,789 pairs; a multicentre cohort of 37,308 pairs confirmed it).
- [Epic Research](https://epicresearch.org/articles/black-patients-32-more-likely-than-white-patients-to-experience-occult-hypoxemia-which-may-result-in-delayed-care): non-Hispanic Black patients were 32% more likely than White patients to have occult hypoxaemia.
- **Regulators responded:** the [FDA's January 2025 draft guidance](https://www.hhs.gov/guidance/sites/default/files/hhs-guidance-documents/FDA/pulse-oximeters-for-medical-purposes-draft-guidance_0.pdf.pdf) asks for clinical testing in **≥150 participants** across **Monk Skin Tone** groups.

### 3.2 What already exists (so we don't duplicate it)

| Resource / study | What it offers | Limitation that leaves room for us |
|---|---|---|
| [**BOLD**](https://www.physionet.org/content/blood-gas-oximetry/) (*Sci Data* 2024) | **49,099** SpO2–SaO2 pairs **within 5 minutes**, SaO2 70–100%, from MIMIC-III, MIMIC-IV and eICU; ~25% minority patients; credentialed | **ICU only**; charted SpO2 numbers, **no raw PPG** |
| [**OpenOximetry**](https://physionet.org/content/openox-repo/) (*Sci Data* 2025) | Controlled lab desaturation studies with **processed and unprocessed PPG**, co-oximetry SaO2, **measured skin colour**; restricted access | Healthy volunteers in a lab, not sick ED patients |
| [Patwari et al., IEEE CHASE 2023](https://arxiv.org/pdf/2210.04990) | Errors have **higher variance** for Black patients, so **no single race-based correction factor** gives equal hypoxaemia detection | Argues for **non-race-based** approaches |
| Tabular ML on BOLD (e.g., [Quantifying racial bias in SpO2 with ML](https://scitepress.org/PublishedPapers/2025/131170)) | SpO2 + covariates → SaO2 with tree models; race often used as an input | Race as a correction input is ethically contested; no waveform information |
| Kim et al., *Anesth Analg* 2023 ([abstract](https://experts.colorado.edu/display/pubid_517944)) | Transfer-learned network improved SpO2 accuracy for African American patients by 23% | Small; race-stratified training |
| [MIMIC-IV Waveform DB](https://physionet.org/content/mimic4wdb/0.1.0/waves/p139/) | ICU bedside PPG linkable to MIMIC-IV labs (v0.1.0: 200 records; larger release planned) | ICU; small in the current release |

**The gap MC-MED could fill:** a **real-world ED population** with **continuous SpO2 numerics** (so pairing can be much tighter than 5 minutes), the **raw bedside pleth waveform** at the moment of the blood draw, and **order timestamps** that let us measure *downstream care consequences*. To my knowledge, no published dataset combines all three.

### 3.3 Two caveats that shape the research question

1. **The bedside "Pleth" channel probably isn't the raw red/IR signal.**
   - Monitors typically export one filtered, normalized pleth waveform (usually AC-coupled).
   - The separate red and infrared intensities and DC levels that SpO2 is computed from are generally **not** in it. Verify this in the WFDB headers and documentation.
   - **Consequence:** we probably *can't* re-compute SpO2 from the waveform. What the waveform *can* tell us is **signal quality, perfusion and morphology**, which are known drivers of oximeter error.
   - **So the right question is not** "re-estimate SaO2 from PPG". **It is:** *"Can we tell when an SpO2 reading is untrustworthy, and flag possible occult hypoxaemia, using signal quality, perfusion and clinical context, without using race as an input?"*
2. **Race is not skin tone.** MC-MED records race and ethnicity, not measured pigmentation. Use race **only to audit** fairness, never as a model input. Use OpenOximetry (measured skin colour) to test the signal-level hypotheses.

### 3.4 Study design: three phases, each publishable

**Phase A — Descriptive (clinical paper; lowest risk)**
- Build SpO2–SaO2 pairs: an **arterial** blood gas with co-oximetry SaO2, matched to monitor SpO2 at the draw time (e.g., median over ±1–2 min; sensitivity analysis at ±5 min to match BOLD).
- Estimate bias (Bland–Altman), accuracy root-mean-square error (**A_rms**, the FDA metric), and **occult hypoxaemia** rates (Sjoding definition) by race and ethnicity, in an **ED** population.
- **The consequence analysis.** This reuses Q2 machinery and is the novel clinical piece. Among patients with occult hypoxaemia vs true normoxaemia, compare:
  - **time to supplemental-oxygen order**
  - escalation of care
  - disposition

  Does occult hypoxaemia measurably delay ED care?
- *Venues:* *JAMA Network Open*, *Annals of Emergency Medicine*, *Academic Emergency Medicine*.

**Phase B — ML: race-blind "SpO2 reliability" model**
- *Inputs:* SpO2 trend; PPG features (signal-quality index, amplitude and perfusion proxies, beat-to-beat variability, morphology embeddings from a PPG foundation model such as [PaPaGei](https://iclr.cc/virtual/2025/poster/28573)); heart rate, blood pressure, temperature (vasoconstriction); haemoglobin, carboxy- and methaemoglobin if available; clinical context. **No race.**
- *Outputs:* (i) P(occult hypoxaemia | current SpO2), and (ii) a **calibrated prediction interval for SaO2**, using conformal prediction.
- *Fairness evaluation:* false-negative rate for hypoxaemia, interval coverage and calibration **by race group**. Compare:
  - SpO2 alone
  - SpO2 + context
  - the race-blind signal model
  - (for reference only) a race-aware model

  **Key question:** does the race-blind waveform model close the detection gap?
- *Venues:* ML4H, CHIL, FAccT, *npj Digital Medicine*.

**Phase C — External and mechanistic checks**
- Test whether the signal features that predict SpO2 error in the ED also relate to **measured skin tone** in OpenOximetry, and replicate on MIMIC-IV-WDB + labs (ICU).

### 3.5 Hypotheses
- **H9a.** ED occult-hypoxaemia rates differ by race and ethnicity, in the same direction as Sjoding 2020 and BOLD.
- **H9b.** Occult hypoxaemia is associated with **later supplemental-oxygen orders** and escalation, after adjusting for acuity.
- **H9c.** A race-blind model using PPG quality/perfusion + context detects occult hypoxaemia better than SpO2 alone and **reduces the between-group gap in false-negative rate**.

### 3.6 The week-1 decision gate

| Check | Go | Pivot |
|---|---|---|
| Count of **arterial** gases with **co-oximetry SaO2** (not venous) | Enough pairs per race group for precise rates. Rule of thumb: ~200–400 pairs per group gives about ±3 percentage points on a 5–10% rate. | Few arterial gases → Phase A as a small descriptive note; move Phase B to **BOLD + MIMIC-IV-WDB + OpenOximetry** |
| SpO2 numerics present at the draw time | Continuous SpO2 within ±2 min for most pairs | Use charted SpO2 with ±5 min (BOLD-style) |
| Pleth channel characteristics | Pleth present at the draw time with usable quality | Without a usable pleth, Phase B becomes tabular (context-only) |
| COHb / MetHb / haemoglobin in labs | Present → include as confounders | Note as a limitation |
| Race/ethnicity completeness | >90% recorded | Report missingness; sensitivity analysis |

**Note on arterial vs venous gases:** EDs draw **venous** gases far more often than ICUs, so arterial pairs may be scarce. That's why this gate matters, and why the BOLD / OpenOximetry fallback is built in.

---

## Part 4 — Combined timeline (after credentialing)

| Weeks | Paper 1 (Q2 → Q1) | Paper 2 (Q9) |
|---|---|---|
| 1 | Data card: timestamps (order vs result vs administration), waveform coverage, label derivability | **Decision gate (3.6):** ABG counts, pleth format, SpO2 numerics |
| 2–3 | Cohort, labels, escalation-action list (with a clinician), MIMIC-IV-ED harmonization | Pairing pipeline (ABG ↔ SpO2 ↔ pleth window) |
| 4–6 | **Q2 experiments E1–E5**; draft the leakage card | Phase A descriptive analysis + consequence analysis |
| 7–8 | Teacher (waveform features → pretrained embeddings); M0–M2 | Draft Phase A (short clinical paper) |
| 9–10 | M3–M5 distillation; stability; ablations | Phase B features, models and fairness evaluation |
| 11–12 | External validation on MIMIC-IV-ED; subgroups; temporal shift | Phase C checks (OpenOximetry / MIMIC-IV-WDB) if access is approved |
| 13–14 | Write and submit Paper 1 | Continue Phase B → Paper 2 |

**Apply early for:** BOLD (credentialed) and **OpenOximetry (restricted)** access, so the Q9 fallback is ready if the gate fails.

---

## Part 5 — What to learn first (minimal clinical background)

| Topic | Why | Effort |
|---|---|---|
| ED workflow: triage (ESI), disposition, escalation | Label and escalation-action definitions (Q2) | 1–2 days |
| Early warning scores (NEWS2, MEWS, qSOFA) | Baselines reviewers expect | 1 day |
| Pulse-oximetry physics (ratio-of-ratios, perfusion index, motion artefacts) | Q9 framing; knowing what the pleth can and can't tell you | 1–2 days |
| ABG vs VBG; co-oximetry; COHb / MetHb | Q9 pairing validity | 1 day |
| Prediction under interventions / competing risks | E6 and the discussion | 2–3 days |
| Generalized distillation (Lopez-Paz 2016) | Q1 method | 1 day |

**A clinical collaborator is strongly recommended**, ideally an emergency physician, to sign off on the escalation-action list, label definitions and Phase A interpretation. Asking the MC-MED dataset authors clarifying questions through PhysioNet's channels is also reasonable.

---

## Sources

**Q2 — leakage and process bias**
- Agniel, Kohane, Weber. *BMJ* 2018: https://www.bmj.com/content/361/bmj.k1479.abstract
- Beaulieu-Jones et al. *npj Digit Med* 2021: https://pubmed.ncbi.nlm.nih.gov/33785839/
- Kamran et al. *NEJM AI* 2024 (coverage): https://healthitanalytics.com/news/epic-sepsis-model-predictions-may-have-limited-clinical-utility
- The early warning paradox. *npj Digit Med* 2025: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11790821/
- van Geloven et al. Causal blind spots in risk prediction: https://arxiv.org/pdf/2402.17366
- CAREBench (stability on MC-MED): https://arxiv.org/html/2510.14286v1

**Q1 — privileged information and distillation**
- Lopez-Paz, Bottou, Schölkopf, Vapnik. Unifying distillation and privileged information. ICLR 2016: https://arxiv.org/abs/1511.03643
- MIND (TMLR 2025): https://www.arxiv.org/abs/2502.01158
- MMPKD (2025): https://arxiv.org/abs/2508.06558v1
- Variational knowledge distillation for CXR with EHR: https://arxiv.org/pdf/2103.10825
- LuPIET (privileged time-series text): https://astro.paperswithcode.com/paper/improving-text-based-early-prediction-by
- MC-BEC: https://arxiv.org/abs/2311.04937 · MC-MED: https://physionet.org/content/mc-med

**Q9 — pulse oximetry**
- Sjoding et al. *NEJM* 2020 (summary): https://psnet.ahrq.gov/issue/racial-bias-pulse-oximetry-measurement
- Epic Research on occult hypoxaemia: https://epicresearch.org/articles/black-patients-32-more-likely-than-white-patients-to-experience-occult-hypoxemia-which-may-result-in-delayed-care
- BOLD (*Sci Data* 2024): https://www.physionet.org/content/blood-gas-oximetry/
- OpenOximetry (*Sci Data* 2025): https://physionet.org/content/openox-repo/
- Patwari et al. Race-based correction cannot fix disparities (IEEE CHASE 2023): https://arxiv.org/pdf/2210.04990
- Quantifying racial bias in SpO2 with ML (2025): https://scitepress.org/PublishedPapers/2025/131170
- Kim et al. Transfer learning for pulse-oximetry bias (*Anesth Analg* 2023): https://experts.colorado.edu/display/pubid_517944
- FDA draft guidance on pulse oximeters (Jan 2025): https://www.hhs.gov/guidance/sites/default/files/hhs-guidance-documents/FDA/pulse-oximeters-for-medical-purposes-draft-guidance_0.pdf.pdf
- MIMIC-IV Waveform Database: https://physionet.org/content/mimic4wdb/0.1.0/waves/p139/
- PaPaGei PPG foundation model (ICLR 2025): https://iclr.cc/virtual/2025/poster/28573
