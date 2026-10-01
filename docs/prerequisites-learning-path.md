# Prerequisites Before Starting Q2 + Q1 on MC-MED

*A learning path for someone **without** a healthcare background, plus a compressed track for someone who has one. Written October 2026.*

Project recap ([deep dive](mc-med-q1-q2-q9-deep-dive.md)):
- **Q2:** audit how much ED "deterioration prediction" comes from clinicians' actions rather than patient physiology.
- **Q1:** use bedside waveforms as *training-only* privileged information to build a better EHR-only model, validated externally on MIMIC-IV-ED.

---

## TL;DR

- **You don't need to become a clinician.** You need to be **"ED-literate"**: know the workflow, what deterioration looks like in the data, and what clinicians *do* about it. That's maybe 5% of medical training, but it's the 5% that decides your labels and your leakage audit.
- **The most important prerequisite is not medicine. It's clinical-prediction study design:** index times, prediction windows, leakage, calibration, external validation. Q2 *is* a study-design paper, so this module matters most.
- **Plan on ~6 weeks part-time (8–10 h/week)** for a newcomer, or **~3 weeks** for someone with healthcare-IT experience. Do it **during the PhysioNet credentialing wait**, so it costs no calendar time.
- **Turn the learning into project code:** the capstone (Section 5) builds the full Q2 pipeline on the **open MIMIC-IV-ED demo**. When MC-MED access arrives, you swap the data source.

---

## 1. Principle: learn by decision, not by syllabus

Every design decision in Q1+Q2 needs specific knowledge. Learn what blocks a decision now; learn the rest when you reach it.

| Project decision | Knowledge needed | Module |
|---|---|---|
| Can I legally download and process the data? Can I use an LLM on it? | Credentialing, DUA, PhysioNet LLM guidance | **M0** |
| When do predictions happen (triage? hourly?) and over what horizon? | ED workflow; prediction-study design | **M1, M4** |
| What counts as "deterioration"? (vital-sign thresholds vs ICU admission/vasopressors) | Vital signs, shock, respiratory failure, sepsis; label design | **M1, M4** |
| Which actions count as "escalation" (the cut-off for pre-recognition evaluation)? | ED interventions and order types | **M1** (+ a clinician) |
| What does each timestamp mean (ordered, collected, resulted, administered)? | EHR data semantics | **M2** |
| How do I build tiered features: physiology → process → actions? | EHR structure + process bias | **M2, M5** |
| How do I turn waveforms into teacher inputs? | ECG/PPG/respiration basics, signal quality, WFDB | **M3** |
| How do I evaluate honestly? (AUPRC, calibration, decision curves, alert burden, subgroups) | Clinical prediction evaluation | **M4** |
| How do I correct for "only monitored patients have waveforms"? | Propensity scores / inverse weighting; invariance | **M5, M6** |
| How do I distil teacher → student? | Knowledge distillation, LUPI, missing modalities | **M6** |
| How do I report it so clinical-ML reviewers accept it? | TRIPOD+AI, PROBAST+AI, pre-registration | **M4, M7** |

---

## 2. The modules

Each module lists **why it matters**, **core concepts**, **resources**, a **hands-on exercise**, and a **"ready when"** test. Times are part-time estimates: newcomer / healthcare-IT background.

### M0 — Compliance and data access (day 1; it's on the critical path) · *2–4 h / 1–2 h*
- **Why:** credentialing is the longest wait. Mistakes here (e.g., pasting MIMIC rows into a chatbot) can cost you access.
- **Concepts:** human-subjects research training; de-identified vs identifiable data; data use agreements (no redistribution, no sharing with third parties); responsible LLM use with credentialed data.
- **Do:**
  1. Create a PhysioNet account.
  2. Complete the **CITI "Data or Specimens Only Research"** course.
  3. Apply for credentialing.
  4. Read the [PhysioNet guidance on LLMs and online services](https://physionet.org/news/post/llm-responsible-use/). In short: don't send credentialed data to third-party services; prefer locally deployed models; use a cloud LLM only with verified zero-retention, no-training terms.
- **Ready when:** you can explain what you may and may not do with MC-MED data, including with LLM tools.

### M1 — ED clinical context ("ED literacy") · *~10 h / ~4 h*
- **Why:** labels, prediction times and the escalation-action list all come from here.
- **Core concepts:**
  - **ED patient journey:** arrival → **triage** (ESI levels 1–5) → evaluation (labs, imaging) → treatment → **disposition** (discharge, ward admission, ICU, transfer, death) → possible **return visit**.
  - **Vital signs and abnormal ranges:** heart rate, blood pressure (incl. MAP), respiratory rate, SpO2, temperature, mental status.
  - **What deterioration is:** shock (hypotension, poor perfusion), respiratory failure (hypoxaemia, high respiratory rate), sepsis (infection + organ dysfunction; Sepsis-3), arrhythmia, altered consciousness.
  - **Early warning scores** (the baselines reviewers expect): **NEWS2**, MEWS, qSOFA. Learn how they are computed.
  - **Escalation actions:** supplemental oxygen and non-invasive ventilation, intubation, IV fluid boluses, **vasopressors**, ICU consult, rapid response, arterial line, blood cultures + lactate (sepsis workup).
- **Resources:**
  - [MIT 6.S897 *Machine Learning for Healthcare*](https://ocw.mit.edu/courses/6-s897-machine-learning-for-healthcare-spring-2019/resources/lecture-1-what-makes-healthcare-unique/), lectures 1–3 (what makes healthcare unique; overview of clinical care; clinical data).
  - The NEWS2 chart and guidance from the Royal College of Physicians (free).
  - Singer et al., *The Third International Consensus Definitions for Sepsis and Septic Shock (Sepsis-3)*, JAMA 2016.
  - The ESI triage handbook: ENA's 5th edition (2023) is the current version, sold through ENA. An overview of the five ESI levels is enough for this project.
  - Free emergency-medicine references such as LITFL and WikEM for looking up terms as you meet them.
- **Hands-on:** take 5 ED stays from the MIMIC-IV-ED demo and write each patient's story in plain English (arrival → triage → vitals → meds → disposition). Mark the moment you, as a reader, would have become worried.
- **Ready when:** you can draft a list of 10–15 escalation actions with reasons, and explain why "ICU admission" is a *clinician decision*, not a pure physiological outcome.
- **Strongly recommended:** a 1-hour conversation with an emergency physician or nurse to review your escalation list and label definitions.

### M2 — Clinical data literacy (EHR structure and timestamps) · *~10 h / ~3 h*
- **Why:** Q2 is fundamentally about *what each data element means and when it became knowable*.
- **Core concepts:**
  - **Encounter structure:** patient → visit/stay → events.
  - **Timestamp semantics:** order **placed** vs specimen **collected** vs result **available** vs medication **administered** vs **charted** (documentation lag). A feature is usable at time *t* only if it was *known* at *t*.
  - **Code systems:** ICD-10 (diagnoses), LOINC (labs), RxNorm / NDC / GSN (medications), units.
  - **Data quality:** missingness patterns, outliers, unit errors, duplicates, copy-forward documentation.
  - **Process bias:** whether and when something is measured carries information ([Agniel et al., BMJ 2018](https://www.bmj.com/content/361/bmj.k1479.abstract)).
  - **Data models:** OMOP (*Book of OHDSI*, free), FHIR, and **MEDS**, the minimal event format for ML (see the [MIMIC-IV demo in MEDS](https://physionet.org/content/mimic-iv-demo-meds/)).
- **Resources:**
  - *Secondary Analysis of Electronic Health Records* (MIT Critical Data, Springer, open access): the chapters on data, cohort selection and pitfalls.
  - The MIMIC-IV and MIMIC-IV-ED documentation (table-by-table).
  - **Practice data:** the [MIMIC-IV-ED demo](https://physionet.org/content/mimic-iv-ed-demo/) (100 patients, open, no credentialing) and the MIMIC-IV clinical demo.
- **Hands-on:** load the MIMIC-IV-ED demo into DuckDB or Postgres. Build a per-stay **event timeline** (vitals, meds, diagnoses, disposition) with explicit "known-at" times.
- **Ready when:** for any feature, you can state the earliest time it was known, and spot at least two timestamp traps in the demo.

### M3 — Physiological signals (ECG, PPG, respiration) · *~12 h / ~10 h*
- **Why:** the Q1 teacher is built on bedside waveforms. This is new ground even for most healthcare-IT engineers.
- **Core concepts:**
  - **ECG:** P-QRS-T waves; lead II; heart rate from R-peaks; **heart-rate variability**; common rhythms (sinus, atrial fibrillation); noise sources (baseline wander, motion, electrode artefacts).
  - **PPG / pleth:** what the pulse oximeter waveform represents; pulse amplitude as a rough perfusion proxy; morphology; motion artefact; why the bedside pleth is usually filtered and normalized.
  - **Respiration:** impedance pneumography; respiratory rate estimation.
  - **Signal processing basics:** sampling rate, filtering, peak detection, windowing, **signal-quality indices**, resampling and alignment with EHR time.
  - **Formats:** WFDB headers and records; numerics vs waveforms.
- **Tools:** `wfdb` (Python), **NeuroKit2** (ECG/PPG/respiration processing), [**pyPPG**](https://pyppg.readthedocs.io/) (74 PPG biomarkers from fiducial points).
- **Practice data:** [**BIDMC PPG and Respiration**](https://www.physionet.org/content/bidmc/): 53 ICU recordings of 8 minutes each, with ECG, PPG and impedance respiration at 125 Hz plus 1 Hz HR/RR/SpO2 and annotated breaths. Open, and very close to MC-MED's channels.
- **Hands-on:** on BIDMC, compute HR from ECG R-peaks and RR from the respiration signal. Compare them with the monitor's 1 Hz numerics, compute a signal-quality index, and produce windowed features (HRV, pulse amplitude, RR variability).
- **Ready when:** you can turn a raw WFDB record into a clean, quality-filtered feature table aligned to timestamps, and explain where each feature could fail.

### M4 — Clinical prediction study design and evaluation · *~15 h / ~10 h* ★ most important
- **Why:** Q2 is a study-design contribution, and reviewers at ML4H, CHIL and JAMIA judge clinical-ML papers mainly on this.
- **Core concepts:**
  - **The anatomy of a prediction task:** cohort → **index (prediction) time** → observation window → (gap) → **prediction horizon** → outcome definition. Draw this diagram for every task.
  - **Splits:** patient-level; **temporal**; **external validation** (internal-external, cross-site).
  - **Leakage taxonomy** ([Kapoor & Narayanan, *Patterns* 2023](https://reproducible.cs.princeton.edu/)): illegitimate features, temporal leakage, non-independence, sampling bias.
  - **Metrics:**
    - AUROC vs **AUPRC** (the baseline equals prevalence);
    - **calibration** (calibration curve, slope and intercept, ECE);
    - **decision-curve analysis / net benefit**;
    - **alert burden** (alerts per true event at a fixed alert rate);
    - lead time; subgroup performance; bootstrap confidence intervals.
  - **Reporting and appraisal:** **TRIPOD+AI** (BMJ 2024) for reporting; **PROBAST+AI** (2025) for risk of bias.
  - **Benchmark hygiene:** cohort definitions and preprocessing often matter more than model class ([YAIB, ICLR 2024](https://iclr.cc/virtual/2024/poster/17800)).
- **Resources:**
  - Collins et al., *Evaluation of clinical prediction models*, BMJ 2024 ([part 1](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10772854/); parts 2–3 on external validation and sample size).
  - Van Calster et al., *Calibration: the Achilles heel of predictive analytics*, BMC Medicine 2019.
  - Vickers et al., a step-by-step guide to interpreting decision curve analysis (*Diagn Progn Res* 2019).
  - Steyerberg, *Clinical Prediction Models* (2nd ed., 2019), as a reference book.
  - Harutyunyan et al., *Multitask learning and benchmarking with clinical time series data*, Sci Data 2019 (the MIMIC benchmark pattern).
  - Saito & Rehmsmeier, *The precision-recall plot is more informative than the ROC plot…*, PLoS ONE 2015.
- **Hands-on:** write a one-page **task specification** for "ED deterioration within 12 h": cohort, index times, windows, outcome, exclusions, splits, metrics. Then implement an evaluation harness (AUROC/AUPRC with bootstrap CIs, a calibration plot, net benefit, alerts per true event) on synthetic data.
- **Ready when:** you can explain why an AUROC of 0.85 can be clinically useless (poor calibration, low prevalence, too many alerts, no lead time) and draw the timeline diagram for your task from memory.

### M5 — Process bias, shortcuts and causal basics · *~12 h / ~8 h*
- **Why:** this is the conceptual core of Q2, and the propensity tools are needed for the selection-aware distillation in Q1.
- **Core concepts:**
  - **"Looking over clinicians' shoulders":** models that learn clinician behaviour ([Beaulieu-Jones et al. 2021](https://pubmed.ncbi.nlm.nih.gov/33785839/)).
  - **Post-recognition evaluation inflation:** Kamran et al., NEJM AI 2024 on the Epic sepsis model ([coverage](https://healthitanalytics.com/news/epic-sepsis-model-predictions-may-have-limited-clinical-utility)).
  - **Treatment-averted outcomes:** [the early warning paradox](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11790821/); **prediction under interventions** ([van Geloven et al.](https://arxiv.org/pdf/2402.17366)).
  - **Shortcut learning:** Geirhos et al., *Nature Machine Intelligence* 2020.
  - **Causal basics:** confounding; selection bias; **propensity scores** and **inverse probability weighting**; competing risks and censoring.
- **Resources:**
  - Hernán & Robins, *Causal Inference: What If* (free online). Read chapters 1–3 (definitions), 7–8 (confounding, selection bias) and 12 (IP weighting).
  - MIT 6.S897 lectures on causal inference.
- **Hands-on:**
  1. On the MIMIC-IV-ED demo, build a "process-only" feature set (counts and timing of orders and meds, no values) next to a physiology-only set.
  2. Write a short simulation in which treatment prevents the outcome, and show how naive evaluation misleads.
  3. Fit a propensity model for "receives an intervention" from triage features and compute IP weights.
- **Ready when:** you can explain to a non-expert, with an example, how a model can reach high AUROC while adding no value beyond clinicians, and how IP weighting changes what a training set "represents".

### M6 — ML methods for Q1 · *~12 h / ~8 h* (if you already know deep learning)
- **Core concepts and papers:**
  - **Knowledge distillation:** Hinton, Vinyals & Dean, *Distilling the Knowledge in a Neural Network*, 2015 (soft labels, temperature).
  - **Learning using privileged information:** Vapnik & Vashist, *Neural Networks* 2009.
  - **Generalized distillation:** [Lopez-Paz, Bottou, Schölkopf & Vapnik, ICLR 2016](https://arxiv.org/abs/1511.03643). This is the core reference.
  - **Multimodal learning with missing modalities:** modality dropout; [MIND (TMLR 2025)](https://www.arxiv.org/abs/2502.01158); [modality imbalance on EHR + CXR (2026)](https://arxiv.org/abs/2602.23614).
  - **Irregular clinical time series:** carry-forward + masks + time-delta features; GRU-D (Che et al. 2018); transformer-based event models; gradient-boosted trees on windowed snapshots (a strong baseline, often hard to beat).
  - **Invariance penalties** for "the student must not encode monitoring status": domain-adversarial training (DANN, Ganin et al., JMLR 2016).
  - **Temporal stability of risk scores:** [CAREBench (2025)](https://arxiv.org/html/2510.14286v1).
- **Hands-on:** in a toy setting (synthetic, or tabular + derived "waveform" features on BIDMC), train teacher and student, then compare soft-label KD, feature alignment and IP-weighted KD. That's your method code, ready for MC-MED.
- **Ready when:** you can write the generalized-distillation loss, explain what the temperature does, and explain why naive distillation could teach "monitored = sick".

### M7 — Research craft for clinical ML · *ongoing*
- **Pre-registration:** write hypotheses, primary endpoints and analysis plan before touching test data (an OSF registration or a dated doc in the repo).
- **Reproducibility:** config-driven pipelines, fixed seeds, experiment tracking, data versioning (hashes of cohort files), a `make`-style end-to-end run.
- **Working with clinicians:** bring concrete artefacts (timeline diagram, escalation list, label definitions), not open questions.
- **Reading clinical-ML papers:** read MC-BEC and CAREBench *code*, not just the papers. Their task definitions are your baselines.
- **Fairness basics:** subgroup reporting and why it matters (Obermeyer et al., *Science* 2019; Chen et al., *Annual Review of Biomedical Data Science* 2021).
- **Venues and format:** look at a few accepted ML4H and CHIL papers to see what reviewers expect (data card, cohort flow diagram, TRIPOD+AI checklist).

---

## 3. What you can skip (for now)

- Anatomy, pharmacology, detailed pathophysiology. Look things up as you meet them.
- 12-lead ECG interpretation beyond the basics (MC-MED has a single bedside lead).
- Imaging, clinical NLP, genomics.
- Pulse-oximetry physics and blood gases (Q9 is deferred).
- Advanced causal inference (g-methods, target trials). Basic IP weighting is enough for Q1/Q2.
- Reinforcement learning, federated learning, LLM agents (the MedAgentBench side track has its own prerequisites).

---

## 4. Schedule

### Newcomer track (~6 weeks, 8–10 h/week), overlapping credentialing

| Week | Focus | Output |
|---|---|---|
| 1 | **M0** (CITI, apply) + **M1** first half + 6.S897 lectures 1–3 | Credentialing submitted; plain-English stories of 5 ED stays |
| 2 | **M1** second half + **M2** | Draft escalation-action list; MIMIC-IV-ED demo timelines |
| 3 | **M4** (study design + metrics) | Task specification + evaluation harness |
| 4 | **M5** (process bias + causal basics) | Process-only vs physiology-only demo; IP-weighting notebook |
| 5 | **M3** (signals) on BIDMC | Waveform feature pipeline with signal-quality filtering |
| 6 | **M6** + capstone integration + clinician review | Toy teacher/student; pre-registration draft |

### Healthcare-IT track (~3 weeks) — for you

| Week | Focus | Notes |
|---|---|---|
| 1 | M0 + skim M1/M2 + **M4** | You know EHR structure and codes; spend the time on *timestamp semantics* and study design |
| 2 | **M5 + M3** | Process bias and signals are probably the most new to you |
| 3 | **M6** + capstone | Ends with a working pipeline on the demo data and a pre-registration draft |

---

## 5. Capstone: build the Q2 pipeline on open demo data

Learning becomes project progress. With 100 patients the *results* are meaningless, so the goal is **correct, tested code**:

1. **Load** the MIMIC-IV-ED demo (+ MIMIC-IV clinical demo) into DuckDB; convert to a MEDS-style event table.
2. **Define** index times (triage, hourly), the outcome (ICU admission / death within *k* hours), and the **escalation-action list**.
3. **Build tiered features:** T0 physiology → T1 triage → T2 lab values (at result time) → T3 process (order existence/timing) → T4 actions.
4. **Implement the pre-recognition filter:** keep only predictions made before the first escalation action.
5. **Evaluation harness:** AUROC, AUPRC, calibration, net benefit, alerts per true event, bootstrap CIs, subgroup tables.
6. **Waveform module** (BIDMC): windowed ECG/PPG/respiration features with signal-quality gating.
7. **Distillation module** (toy): teacher with waveform features, student without; soft-label KD, feature alignment and IP-weighted KD.
8. **Tests:** unit tests asserting that **no feature uses information after its index time**. That check is the backbone of Q2.

When MC-MED access arrives, steps 1 and 6 change their data source; everything else carries over.

---

## 6. Readiness self-test

You're ready to start on MC-MED when you can answer these without notes:

1. What do *triage*, *ESI* and *disposition* mean, and which MC-MED fields correspond to them?
2. Give three examples of escalation actions, and explain why each could leak the outcome "ICU admission".
3. For a lab result, what's the difference between order time, collection time and result time? Which one may a feature use?
4. Draw the timeline for "deterioration within 12 h of each hourly index time", including the observation window and horizon.
5. Your outcome prevalence is 3%. What is the AUPRC of a random model, and why report AUPRC at all?
6. What does a calibration slope of 0.6 mean, and what would you do about it?
7. How would you measure alarm burden for a deployed early-warning model?
8. Explain Agniel et al. 2018 and Beaulieu-Jones et al. 2021 in two sentences each.
9. Why might the Epic sepsis model look good overall but poor before clinicians act?
10. Load a WFDB record, find R-peaks, compute heart rate, and flag low-quality segments. What can go wrong?
11. Write the generalized-distillation loss and explain the role of λ and temperature.
12. Only monitored patients have waveforms. Why can naive distillation teach "monitored = sick", and how do IP weighting or an adversarial penalty help?
13. What do TRIPOD+AI and PROBAST+AI require that most ML papers omit?
14. What are you allowed to do with MC-MED data and LLM tools under the PhysioNet DUA?

---

## Sources

**Courses and books**
- MIT 6.S897 *Machine Learning for Healthcare* (Sontag & Szolovits, 2019): https://ocw.mit.edu/courses/6-s897-machine-learning-for-healthcare-spring-2019/resources/lecture-1-what-makes-healthcare-unique/
- Hernán & Robins, *Causal Inference: What If* (free online book) †
- *Secondary Analysis of Electronic Health Records* (MIT Critical Data, Springer, open access) †
- Steyerberg, *Clinical Prediction Models* (2nd ed., 2019) †

**Data and tools**
- MIMIC-IV-ED demo (open, 100 patients): https://physionet.org/content/mimic-iv-ed-demo/
- MIMIC-IV demo in MEDS format: https://physionet.org/content/mimic-iv-demo-meds/
- BIDMC PPG and Respiration dataset: https://www.physionet.org/content/bidmc/
- pyPPG: https://pyppg.readthedocs.io/ · paper: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11003363/
- NeuroKit2; wfdb-python †
- YAIB (ICLR 2024): https://iclr.cc/virtual/2024/poster/17800

**Compliance**
- PhysioNet: use of MIMIC data with LLMs and online services: https://physionet.org/news/post/llm-responsible-use/

**Study design, evaluation, reporting**
- Collins et al., *Evaluation of clinical prediction models (part 1)*, BMJ 2024: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10772854/
- TRIPOD+AI (BMJ 2024) and PROBAST+AI (2025); protocol: https://bmjopen.bmj.com/content/11/7/e048008
- Kapoor & Narayanan, *Leakage and the reproducibility crisis in ML-based science* (Patterns 2023): https://reproducible.cs.princeton.edu/
- Van Calster et al., calibration (BMC Med 2019) †; Vickers et al., decision curves (Diagn Progn Res 2019) †; Saito & Rehmsmeier, PR curves (PLoS ONE 2015) †; Harutyunyan et al. (Sci Data 2019) †

**Process bias and causal**
- Agniel, Kohane & Weber (BMJ 2018): https://www.bmj.com/content/361/bmj.k1479.abstract
- Beaulieu-Jones et al. (npj Digit Med 2021): https://pubmed.ncbi.nlm.nih.gov/33785839/
- Kamran et al. (NEJM AI 2024), coverage: https://healthitanalytics.com/news/epic-sepsis-model-predictions-may-have-limited-clinical-utility
- The early warning paradox (npj Digit Med 2025): https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11790821/
- van Geloven et al., prediction under interventions: https://arxiv.org/pdf/2402.17366
- Geirhos et al., shortcut learning (Nat Mach Intell 2020) †

**Methods for Q1**
- Lopez-Paz et al., generalized distillation (ICLR 2016): https://arxiv.org/abs/1511.03643
- Hinton et al., knowledge distillation (2015) †; Vapnik & Vashist, LUPI (2009) †; Ganin et al., DANN (JMLR 2016) †; Che et al., GRU-D (2018) †
- MIND (TMLR 2025): https://www.arxiv.org/abs/2502.01158
- When does multimodal learning help (2026): https://arxiv.org/abs/2602.23614
- CAREBench (2025): https://arxiv.org/html/2510.14286v1

**Clinical references**
- Sepsis-3 (Singer et al., JAMA 2016) †; NEWS2 (Royal College of Physicians) †; ESI Handbook 5th ed. (ENA, 2023) †

† Standard references cited from memory; verify the exact citation before using it in a paper.
