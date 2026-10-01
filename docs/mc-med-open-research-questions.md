# MC-MED: Open Research Questions (plus the MedAgentBench side track)

*Companion to [`open-datasets-for-healthcare-ml-papers.md`](open-datasets-for-healthcare-ml-papers.md). Written October 2026.*

> **Method and caveats.** Direct access to physionet.org and arXiv was blocked from the research sandbox, so the facts below come from web search over the dataset page, the dataset and benchmark papers, and follow-up work (all under Sources). Several questions depend on facts you can only confirm once you have credentialed access, such as waveform coverage and which lab tests are present. Section 6 is a **first-week feasibility checklist** that decides which questions are viable. Run it before committing.

---

## TL;DR

**The important correction to our earlier plan:** MC-MED's predecessor benchmark, **MC-BEC (NeurIPS 2023)**, already evaluates multimodal use, **robustness to missing data**, and **fairness** on these Stanford ED visits, for decompensation, disposition and revisit tasks. **CAREBench (2025)** already uses MC-MED for decompensation and sepsis, adding a **temporal-stability** metric. So "multimodal fusion + missing-modality robustness" is **not novel enough on its own**. We need sharper questions.

**The three strongest open questions:**

| # | Question | Why it's open | Type of contribution |
|---|---|---|---|
| **Q1** | **Can waveforms that exist only at training time make EHR-only models better?** Most EDs never store bedside waveforms. Train with privileged waveform data, deploy without it, and validate externally on MIMIC-IV-ED. | Prior work treats missing waveforms as a robustness nuisance, not as *privileged information* to distil from. | ML method + external validation |
| **Q2** | **How much of "deterioration prediction" is predicting the clinicians' behaviour rather than the patient's state?** Orders, monitoring decisions and treatments all leak the outcome. | Known in general (the "early warning paradox"), but never quantified on a multimodal ED dataset where monitoring itself is a clinician decision. | Evaluation rigor / leakage audit |
| **Q3** | **Do biosignal foundation models transfer to ED bedside monitors?** 12-lead ECG models (ECG-FM, ECGFounder) and wearable PPG models (PaPaGei, Pulse-PPG) were trained on different signals. Does a model pretrained on MC-MED itself win? | MC-MED has continuous single-lead ECG, PPG and respiration at ED scale, a domain no released foundation model was trained on. | Benchmark + self-supervised learning |

**Recommended flagship paper:** Q1 as the method, with Q2's leakage audit as the evaluation backbone:

> *"Learning from monitors you won't have: privileged-waveform distillation for deployable, leakage-audited ED deterioration prediction."*

Q3 is the natural second paper. A health-equity variant (**pulse-oximetry bias correction from the raw PPG waveform**, Q9) is potentially the highest-impact question, but it **depends on whether paired arterial blood gases exist**. Check that in week 1.

---

## 1. What MC-MED contains (as of v1.0.1)

| Aspect | What I could confirm | Verify after access |
|---|---|---|
| Scale | **118,385 adult ED visits**, ~**70K unique patients**, Stanford Health Care, **Sept 2020 – Sept 2022** | Exact patient count; any date shifting |
| Waveforms | Continuous bedside-monitor **ECG (lead II)**, **PPG (Pleth)**, **respiration**, in WFDB format, segmented by visit | Sampling rates (read the header files); **what share of visits have waveforms**; segment lengths |
| Numerics | Continuously monitored vital signs | Sampling interval; SpO2, HR, RR, BP coverage |
| EHR tables | Visits (demographics, arrival, triage, chief complaint, diagnosis, disposition, timestamps, payor); home meds; in-ED orders (labs, imaging, meds, procedures, with times); labs (values, ranges, result times); medication administrations; imaging results; past medical history | Exact file names and columns; whether **order times vs result times** are both present; radiology report text |
| Outcomes | Visit outcomes / disposition | How decompensation / ICU transfer / revisit can be derived |
| Access | PhysioNet credentialed | DUA terms on LLM API use |

**Known structural features that create research questions:**
1. **Monitoring is a clinical decision.** Waveforms exist only for patients the team chose to monitor, so their absence is informative, not random.
2. **It covers the COVID and post-COVID era** (2020–22), so there's built-in temporal shift.
3. **It's a single centre**, so external validation must come from elsewhere: MIMIC-IV-ED, plus MIMIC-IV-ECG through **MDS-ED**.
4. **Bedside waveforms differ from diagnostic ECGs:** single lead, long, noisy, with motion artefacts.

---

## 2. What has already been done (don't duplicate it)

| Work | Data | What it covers | What it leaves open |
|---|---|---|---|
| [**MC-BEC**](https://arxiv.org/abs/2311.04937) (NeurIPS 2023 D&B; Chen, Kansal, …, Kim, Rajpurkar) | Stanford ED, 100K+ visits (the MC-MED precursor) | Tasks: **decompensation, disposition, ED revisit**; multimodal inputs incl. ECG/PPG and imaging-report text; evaluates **multimodal use, missing-data robustness, fairness** | Privileged-information training; care-process leakage; deployability without waveforms; biosignal foundation-model transfer |
| [**CAREBench**](https://arxiv.org/html/2510.14286v1) (2025; Keoliya et al., UPenn) | MC-MED (decompensation, sepsis), MIMIC-IV, EHRSHOT | **Temporal stability** of risk trajectories (local-Lipschitz metric); classical, deep and zero-shot LLM baselines; methods "struggle to jointly optimize accuracy and stability" with "poor recall at high-precision operating points" | *Methods* that improve stability; alarm burden at fixed alert budgets |
| [**MDS-ED**](https://arxiv.org/html/2407.17856v4) (2024; ECG + EHR emergency benchmark) | MIMIC-IV-ED + MIMIC-IV-ECG (12-lead) | Discharge diagnoses (AUROC >0.8 for 609/1,428 conditions) and **15 deterioration targets** (14 with AUROC >0.8) | Continuous monitoring; cross-site transfer to/from MC-MED |
| [**When does multimodal learning help?**](https://arxiv.org/abs/2602.23614) (2026) | MIMIC-IV + MIMIC-CXR | Fusion helps when modalities are complete; **modality imbalance** from rich EHR "architectural complexity alone cannot overcome" | The waveform analogue; imbalance-aware training |
| [**Early warning paradox**](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11790821/) (npj Digit Med 2025) | Hospital early warning systems | Existing alert systems *avert* the outcomes models are trained to predict, so standard evaluation can be paradoxical | Quantification on ED multimodal data; evaluation protocol |
| [**LLM ED-triage bias**](https://arxiv.org/pdf/2504.16273) (2025); [**LLM triage benchmark**](https://arxiv.org/html/2509.26351v1) (GenAI4Health @ NeurIPS 2025) | ED triage data | LLM capability plus bias in triage; hospital-rich vs field-like regimes | Waveform-aware or tool-augmented LLM triage |
| Biosignal foundation models: [**ECG-FM**](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12530324/) (1.5M 12-lead ECGs), [**PaPaGei**](https://iclr.cc/virtual/2025/poster/28573) (ICLR 2025, 57k hours of PPG), [**Pulse-PPG**](https://arxiv.org/html/2502.01108v1) (field-trained) | Diagnostic 12-lead ECG; public or wearable PPG | Open weights; strong label efficiency in their own domains | **Transfer to ED bedside single-lead ECG / Pleth / respiration** |

**Bottom line:** the most crowded ground is "fusion + missing data + fairness" and "decompensation AUROC". The clearest open space is **deployability** (privileged information, leakage, stability, alarm burden), **bedside-signal representation learning**, and **equity in monitoring signals**.

---

## 3. Catalogue of open questions

Each question has the open problem, an approach, the ML contribution, feasibility, risk and a novelty rating. Ratings are **H/M/L** for novelty and feasibility.

### Q1. Privileged waveforms: train with monitors, deploy without them ★ flagship
- **Problem.** Waveforms are the richest signal in MC-MED, but most hospitals, including MIMIC-IV-ED, have no continuous waveforms at prediction time. Can a waveform-informed *teacher* make a deployable *EHR-only student* better, better calibrated and more stable?
- **Approach.** Learning using privileged information (LUPI):
  - knowledge distillation from a multimodal teacher to an EHR+vitals student;
  - representation alignment (the student predicts the teacher's waveform embedding);
  - a comparison with standard modality dropout.
  - Evaluate the student on MC-MED (held-out *time* split) **and on MIMIC-IV-ED**, where only the student can run. That gives true external validation of the deployable model.
- **ML contribution.** LUPI and distillation for irregular clinical time series under missing-not-at-random modalities; a principled answer to "is the waveform's value transferable?"
- **Novelty H · Feasibility H** (one GPU; waveforms only for the teacher).
- **Risks.**
  - The gains may be small. Report effect sizes with confidence intervals and show *where* the student improves (subgroups, lead times).
  - Label definitions must match across MC-MED and MIMIC-IV-ED. Define them once and harmonize, ideally in MEDS format.

### Q2. Are we predicting patients or clinicians? A care-process leakage audit ★ evaluation backbone
- **Problem.** In ED data, many inputs are **downstream of clinicians' judgement** that a patient is deteriorating: an ICU consult order, a lactate order, a vasopressor, starting continuous monitoring, an arterial line. Models exploit these, inflating AUROC while adding little clinical value. The early-warning paradox adds that interventions *prevent* the labelled outcome.
- **Approach.** Build input tiers and measure the drop at each:
  - **T0** physiology only (vitals/waveforms)
  - **T1** + triage
  - **T2** + results
  - **T3** + *orders and actions*

  Also:
  - "action-blind" re-timing, i.e. predicting before the first escalation order;
  - treating interventions as competing or composite events;
  - **lead-time–matched** comparisons;
  - the **"monitoring-indicator" shortcut** test: how much does "has waveform" alone predict?
- **ML contribution.** A reusable leakage taxonomy and protocol for multimodal clinical prediction. Reviewers increasingly ask for exactly this.
- **Novelty M–H · Feasibility H.**
- **Risk.** Low. It's mostly careful engineering, which is your strength.

### Q3. Do biosignal foundation models transfer to ED bedside monitors?
- **Problem.** Open ECG foundation models are trained on 10-second 12-lead diagnostic ECGs, and PPG models on wearable or curated clinical PPG. ED monitors produce *long, single-lead, artefact-heavy* ECG, Pleth and respiration.
- **Approach.**
  - Benchmark frozen and fine-tuned ECG-FM / ECGFounder / HuBERT-ECG and PaPaGei / Pulse-PPG against **SSL pretrained on MC-MED's unlabeled waveform hours** (masked modelling, contrastive, JEPA-style).
  - Tasks: decompensation, disposition, rhythm or derived-vital targets.
  - Measure label efficiency at 1%, 10% and 100% of labels.
- **ML contribution.** Domain-gap measurement plus the first ED-monitor SSL model; a waveform-encoder benchmark others can reuse.
- **Novelty H · Feasibility M–H.** Waveform I/O at scale is engineering-heavy; budget storage and dataloaders.
- **Risk.** Respiration channels may be noisy or sparse. Report per-channel availability.

### Q4. Stability-aware early warning: fewer flip-flopping alarms at the same lead time
- **Problem.** CAREBench showed current methods can't jointly optimize accuracy and temporal stability on MC-MED. Unstable risk scores cause alarm fatigue.
- **Approach.** Train-time stability:
  - temporal-smoothness and Lipschitz regularization;
  - state-space or continuous-time models with hysteresis;
  - "evidence-gated" updates where risk changes only when new data arrives.

  Evaluate on CAREBench's stability metric plus **alerts per true event at a fixed alert budget** and lead time.
- **ML contribution.** A method for a metric that is defined but currently unoptimized.
- **Novelty M–H · Feasibility H.** Combines well with Q1.

### Q5. Selection-aware value of continuous monitoring
- **Problem.** Does continuous monitoring add predictive value beyond nurse-charted vitals, *for whom*, and can a model recommend **who should be monitored**? Monitored patients are a selected group (confounding by indication).
- **Approach.** Compare within comparable strata (propensity of being monitored from triage data), value-of-information analysis, and a monitoring-allocation policy evaluated off-policy.
- **ML contribution.** Causal / off-policy evaluation applied to an operational decision.
- **Novelty H · Feasibility M.** It needs careful causal assumptions; a good *second-year* question.

### Q6. Robustness to the pandemic-era temporal shift
- **Problem.** 2020–22 spans COVID waves with changing case mix and triage practices. Do models trained on early months degrade later? Which recalibration or updating strategies work?
- **Approach.** Rolling-origin temporal evaluation, drift detection, online recalibration, COVID-status subgroups.
- **Novelty M · Feasibility H.** A good robustness section within Q1, rather than a standalone paper.

### Q7. Early disposition (admission) prediction for patient flow
- **Problem.** Predicting admission early can cut ED boarding, an operations problem that fits your health-systems background. Prediction alone isn't enough: the value depends on capacity.
- **Approach.** Admission prediction updated over time, evaluated with **decision-analytic metrics** (net benefit, hours of earlier bed requests) plus a simple queueing simulation.
- **ML contribution.** Prediction evaluated as part of a decision.
- **Novelty M · Feasibility H.** Strong for informatics venues (AMIA, JAMIA).

### Q8. Safe discharge: physiology at departure and 72-hour returns
- **Problem.** Do vitals and waveform trends in the *last hour before discharge* predict bounce-back revisits beyond triage and EHR features?
- **Novelty M–H · Feasibility M.** It depends on revisit labels and waveform coverage near discharge.

### Q9. Equity in monitoring signals: pulse-oximetry bias from the raw PPG ★ highest impact if feasible
- **Problem.** Pulse oximeters overestimate oxygen saturation in patients with darker skin, causing occult hypoxaemia (well documented since 2020). MC-MED has the **raw PPG waveform**, continuous SpO2 and demographics. If it also has **arterial blood gases (SaO2)** close in time, it's one of the few public datasets that can study **waveform-level bias correction**.
- **Approach.** Pair SpO2/PPG windows with ABG SaO2. Quantify the bias by group, train PPG-based correction or occult-hypoxaemia detection, and report calibration by subgroup.
- **ML contribution.** Fairness-aware signal modelling with a real-world harm endpoint.
- **Novelty H · Feasibility: unknown.** It depends on ABG availability and timing; that's the first item on the week-1 checklist.
- **Risk.** Race and ethnicity are poor proxies for skin pigmentation. State this limitation explicitly.

### Q10. Time-series-aware LLMs for ED triage and deterioration
- **Problem.** CAREBench found zero-shot LLMs struggle on these tasks. Can tool-augmented LLMs, which call waveform encoders and lab trend tools, match specialized models while producing grounded explanations?
- **Novelty M · Feasibility M.** Watch the PhysioNet DUA rules on sending credentialed data to external LLM APIs; use local open-weight models.

### Q11. Cross-site harmonized ED benchmark (MC-MED ↔ MIMIC-IV-ED / MDS-ED)
- **Problem.** No common, harmonized task set exists across the two largest open ED datasets.
- **Approach.** Shared task definitions plus a **MEDS ETL for MC-MED**; transfer in both directions.
- **Contribution.** An infrastructure and benchmark paper; the community values ETLs.
- **Novelty M · Feasibility H.** A natural by-product of Q1. Check whether a MEDS ETL for MC-MED already exists.

### Q12. Imbalance-aware multimodal training
- **Problem.** The 2026 EHR+CXR benchmark found rich EHR causes **modality imbalance** that architecture can't fix. Does the same hold for waveforms vs EHR? Do gradient-modulation or balanced-learning methods help?
- **Novelty M · Feasibility H.** A good ablation section for Q1.

### Ranking for your 90-day window

| Question | Novelty | Feasibility | Clinical weight | Fit with your profile | Verdict |
|---|---|---|---|---|---|
| **Q1 Privileged waveforms** | H | H | H | H | **Flagship method** |
| **Q2 Leakage audit** | M–H | H | H | **Very H** | **Flagship evaluation backbone** |
| **Q3 Biosignal FM transfer** | H | M–H | M | M | **Paper #2** |
| **Q9 Pulse-ox bias from PPG** | H | ? | **Very H** | M | **Check week 1; pivot if data supports it** |
| Q4 Stability | M–H | H | H | M | Fold into Q1 |
| Q7 Disposition / flow | M | H | M–H | **Very H** | Informatics-venue spin-off |
| Q6 Temporal shift · Q12 Imbalance | M | H | M | M | Sections within Q1 |
| Q11 Cross-site harmonization | M | H | M | H | By-product → resource paper |
| Q5 Monitoring allocation · Q8 Discharge · Q10 LLMs | M–H | M | M–H | M | Later |

---

## 4. Flagship paper design (Q1 + Q2)

**Working title:** *Learning from monitors you won't have: privileged-waveform distillation for deployable, leakage-audited ED deterioration prediction.*

**Hypotheses (pre-register):**
- **H1.** An EHR+vitals student distilled from a waveform-informed teacher improves AUPRC and calibration over an EHR-only model trained normally. This should hold on MC-MED's temporally held-out test set *and* externally on MIMIC-IV-ED.
- **H2.** Gains are largest for **early prediction times** (before escalation orders) and in **unmonitored or low-acuity strata**, where waveform-derived knowledge is most "new".
- **H3.** Under the leakage audit, standard models lose a substantial share of performance when action-derived inputs are removed. The distilled student loses less, because it relies more on physiology.

**Tasks** (define once; harmonize with MIMIC-IV-ED):
- Decompensation or critical intervention within *k* hours (in line with the MC-BEC/CAREBench definitions).
- ICU admission from the ED.
- Admission (disposition).

**Prediction times:** triage, +1h, +2h, rolling hourly. Also an **"before first escalation order"** cut-off (Q2).

**Splits:** patient-level and **temporal** (train 2020–21, test 2022). External test on MIMIC-IV-ED with EHR inputs only.

**Models:**
- *Baselines:* triage scores (e.g., ESI-based), NEWS2-style rule scores, GBM on tabular snapshots, a sequence transformer.
- *Teachers:* waveform encoders (from Q3 work or ECG-FM/PaPaGei features) fused with EHR.
- *Students:* the same EHR architecture trained with distillation and representation-alignment losses, compared against modality dropout.

**Metrics:**
- AUROC, **AUPRC**, calibration (ECE, calibration slope), net benefit.
- **Alerts per true event at fixed alert budgets**, lead time.
- CAREBench **stability**.
- Subgroups (sex, age, race/ethnicity, payor, language).
- Bootstrap CIs; multiple seeds.

**Leakage audit (Q2):** the T0–T3 input tiers; the "has-waveform" shortcut model; action-blind timing; interventions as competing events.

**Ablations:** loss variants; teacher strength; waveform channels (ECG / Pleth / Resp); prediction horizon; modality-imbalance remedies (Q12).

**Venues:** ML4H (full or findings), CHIL, MLHC; journal versions in *npj Digital Medicine*, *JAMIA*, *JBI*.

---

## 5. Revised 90-day plan

| Weeks | MC-MED track (updated) | MedAgentBench track |
|---|---|---|
| 0–2 | CITI + credentialing; pipeline on MIMIC-IV demo / MEDS demo; read MC-BEC, CAREBench, MDS-ED closely | Reproduce baselines on **v1 and v2** with current models (published numbers are from early 2025) |
| 3–4 | **Week-1-after-access checklist (Section 6)**; cohort, task and prediction-time definitions; MEDS ETL | Build the FHIR-realism perturbation suite (Section 7) |
| 5–6 | **Leakage audit (Q2)** + EHR-only baselines; temporal split | Measure robustness drop; severity-weighted scoring |
| 7–8 | Waveform teacher (pretrained-FM features first, then light SSL) | Verify-before-write guardrail |
| 9–10 | **Privileged distillation (Q1)**, stability metrics (Q4), modality-imbalance ablation (Q12) | Ablations; error taxonomy |
| 11–13 | **External validation on MIMIC-IV-ED**; subgroups; temporal-shift section; draft | **Workshop submission** |

---

## 6. First-week feasibility checklist (after access)

These ten checks decide which questions are viable. Turn the answers into a one-page "data card" for the paper.

1. **Waveform coverage:** % of visits with ECG, Pleth and Resp; hours per visit; distribution by acuity. *(Q1, Q3, Q5)*
2. **Sampling rates and quality:** from WFDB headers; fraction of flat-line or artefact segments. *(Q3)*
3. **Timestamp fidelity:** order *placed* vs *resulted* vs *administered* times; time zone or date shifting. *(Q2, all tasks)*
4. **Outcome derivability:** can decompensation, ICU admission and 72h revisit be derived consistently with MC-BEC/CAREBench? *(Q1, Q8)*
5. **ABG availability:** arterial SaO2 results and how close in time they are to SpO2 readings. *(Q9 go/no-go)*
6. **Demographics:** race, ethnicity, language and payor completeness. *(fairness, Q9)*
7. **Temporal coverage:** visit counts by month; COVID markers. *(Q6)*
8. **Free text:** whether radiology reports or chief-complaint text are present. *(Q10)*
9. **Harmonization map to MIMIC-IV-ED:** which variables align (triage vitals, ESI, labs, dispositions). *(Q1 external validation, Q11)*
10. **Existing code:** search for MEDS ETLs or loaders for MC-MED, and the MC-BEC/CAREBench repos for task definitions to reuse. *(saves weeks)*

---

## 7. MedAgentBench side track: open questions

**State of the field:**
- [MedAgentBench](https://arxiv.org/html/2501.14654v2) has 300 tasks; the best model in early 2025 reached **69.67%**. Models were strong on *queries* and weaker on *actions* (orders, documentation).
- **MedAgentBench v2** added tools, memory and 300 new multi-step tasks.
- [FHIR-AgentBench](https://arxiv.org/html/2509.19319v2) (2,931 questions on FHIR) shows that retrieving data from intricate FHIR resources is a major bottleneck.
- [MedAgentGym](https://arxiv.org/abs/2506.04405v2) (NeurIPS 2025) showed a 7B model gaining +36% from supervised fine-tuning and +42% from reinforcement learning on verifiable biomedical coding tasks.
- [HealthAgentBench](https://arxiv.org/abs/2606.31179) (2026) best average is 42%.

**Open questions:**

| # | Question | Why open | Contribution |
|---|---|---|---|
| A1 | **How robust are EHR agents to real-world FHIR messiness?** (US Core profile variants, LOINC/RxNorm code variants, unit differences, pagination, missing or partial resources, transient API errors) | Benchmarks use clean, idealized servers; real FHIR deployments are messy | **FHIR-realism perturbation suite** + robustness curves ★ flagship |
| A2 | **Severity-weighted evaluation:** a wrong-patient or wrong-dose order is not "one failed task" | Success rate treats all failures equally | Clinically weighted scoring + an error taxonomy |
| A3 | **Verify-before-write:** can schema and profile validation, dry-run simulation and confirmation steps close the query/action gap without hurting success? | Action tasks lag queries | Guardrail method; safety vs success trade-off |
| A4 | **Abstain or escalate:** do agents know when to ask a human? | Calibration and abstention are rarely measured | Selective-execution metrics |
| A5 | **Small open models + reinforcement learning on verifiable FHIR tasks** (privacy-preserving, on-premises) | MedAgentGym shows it works for coding; not yet shown for FHIR order entry | Training recipe + cost analysis |
| A6 | **Does v2's "memory from failures" generalize, or overfit to the benchmark?** | Memory can memorize task specifics | Held-out-workflow evaluation |

**Flagship:** **A1 + A2 + A3**: *"FHIR-Real: robustness and safety of EHR agents under realistic interoperability perturbations, with severity-weighted scoring and verify-before-write guardrails."* It fits your health-IT background exactly, needs no patient-data access, and the perturbation suite stays useful as models improve.

**Note:** the published leaderboard numbers are from early 2025. Newer models may score much higher, so reproduce first (weeks 0–2). If the original benchmark is near saturation, the perturbation suite becomes even more valuable.

---

## Sources

**MC-MED and direct prior work**
- MC-MED (PhysioNet): https://physionet.org/content/mc-med
- MC-BEC (NeurIPS 2023 D&B): https://arxiv.org/abs/2311.04937 · https://neurips.cc/virtual/2023/poster/73542
- CAREBench / Stable prediction of adverse events: https://arxiv.org/html/2510.14286v1
- MDS-ED (waveforms in emergency care benchmark): https://arxiv.org/html/2407.17856v4
- When does multimodal learning help in healthcare? (EHR + CXR, 2026): https://arxiv.org/abs/2602.23614
- The early warning paradox (npj Digit Med 2025): https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11790821/
- LLMs for ED triage, capability and bias: https://arxiv.org/pdf/2504.16273
- LLM-assisted emergency triage benchmark: https://arxiv.org/html/2509.26351v1
- MedM2T (EHR + ECG, MIMIC-IV): https://arxiv.org/pdf/2510.27321

**Biosignal foundation models**
- ECG-FM: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12530324/
- PaPaGei (ICLR 2025): https://iclr.cc/virtual/2025/poster/28573
- Pulse-PPG: https://arxiv.org/html/2502.01108v1

**Agents**
- MedAgentBench: https://arxiv.org/html/2501.14654v2
- FHIR-AgentBench: https://arxiv.org/html/2509.19319v2
- MedAgentGym: https://arxiv.org/abs/2506.04405v2
- HealthAgentBench: https://arxiv.org/abs/2606.31179
- Medmarks: https://arxiv.org/pdf/2605.01417

**Background (from memory; verify before citing)**
- Sjoding MW et al. Racial bias in pulse oximetry measurement. *NEJM* 2020.
- Vapnik V, Vashist A. A new learning paradigm: learning using privileged information. *Neural Networks* 2009.
