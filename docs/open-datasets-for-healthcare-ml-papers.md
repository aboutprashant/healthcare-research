# Open Datasets That Are Easier to Publish On (Healthcare AI / Biomedical ML)

*Companion to [`ahba-ml-research-report.md`](ahba-ml-research-report.md). Survey done October 2026.*

**Question:** compared with the Allen Human Brain Atlas (AHBA), which open datasets make it *easier to write and publish ML research papers*, given a software/ML engineer profile with healthcare-systems and public-health experience aiming for a Healthcare AI PhD?

> **Method and caveats.** I compiled this from web searches of dataset pages, dataset papers and benchmark papers, all linked under Sources. Dataset sizes and access terms change, so confirm the current version and licence on each dataset's page before you plan around it. The scores are my own judgement, made to compare options. They are not measured quantities.

---

## TL;DR

- **For writing papers, almost every dataset below is easier to use than AHBA.** AHBA has n = 6 donors and no outcome labels. These datasets have thousands to millions of samples, real clinical labels, published baselines and active communities.
- **The single most useful step this week:** get **PhysioNet credentialed access**. It needs a free CITI human-subjects course, an application and per-dataset agreements. One credential unlocks MIMIC-IV, MC-MED, eICU, INSPIRE and more.
- **The five best fits for you:**

| # | Dataset family | Why it suits you | Time to first result |
|---|---|---|---|
| 1 | **MC-MED** (+ MIMIC-IV / MIMIC-IV-ED / eICU) | New 2025 multimodal emergency-department dataset: continuous waveforms plus EHR. Few published baselines, high clinical relevance, recognized by reviewers. | ~3–5 weeks, after credentialing |
| 2 | **MedAgentBench / HealthAgentBench** | Agents working inside FHIR-based EHR environments. Directly matches healthcare-systems engineering, and there's no data-access wait. | ~1–2 weeks |
| 3 | **Open 12-lead ECG at scale** (MIMIC-IV-ECG, CODE-15%, PTB-XL) + PhysioNet Challenge | Fully open, about 1.2M ECGs across sites, cheap 1D compute, yearly challenge with proceedings. | ~1–2 weeks |
| 4 | **HEST-1k / STimage-1K4M** (histology ↔ spatial transcriptomics) | The data-rich successor to your AHBA spatial-biology interest, with NeurIPS benchmarks. | ~2–3 weeks |
| 5 | **CDC FluSight hub + NHANES** | Matches your public-health background. Prospective forecasting is the strongest defence against data leakage. | ~2 weeks |

- **My single pick:** **MC-MED** for the main paper. While credentialing is pending, start a **MedAgentBench** side project. Details are in the last section.

---

## 1. What makes a dataset "paper-friendly"

| Criterion | Why reviewers and advisors care | AHBA for comparison |
|---|---|---|
| **Access friction** | Weeks lost waiting for approvals; licences that block release of code or models | Open, but tiny |
| **Labels and scale** | Supervised tasks, enough power for confidence intervals, subgroup analyses | 6 donors, no outcomes |
| **Benchmark maturity** | Published baselines and standard splits let reviewers judge your gain | Few ML baselines |
| **Headroom** | Saturated benchmarks (e.g., PTB-XL superclass, MedQA, MIMIC mortality) make novelty hard | High, but hard to measure |
| **Natural shifts** | Multiple sites, scanners or years give generalization, fairness and federated-learning papers for free | None |
| **Compute** | One GPU vs a cluster | Light |
| **Fit with you** | EHR, FHIR, pipelines, public health | Low |

### Access tiers (fastest first)

| Tier | Typical process | Examples |
|---|---|---|
| **Open download** | Accept a licence | PTB-XL, CODE-15%, MIMIC-IV-ECG, VitalDB, PMC-Patients, CELLxGENE, Tahoe-100M, NHANES, FluSight, OpenMind, SLICE-3D |
| **Gated / registration** | Hugging Face or website sign-up, research-use agreement | CT-RATE, HEST-1k, CheXpert Plus, NSRR, FairVision |
| **Credentialed (PhysioNet)** | CITI "Data or Specimens Only Research" course + credentialing application + per-dataset DUA. Usually days to a couple of weeks. | MIMIC-IV, MC-MED, eICU, INSPIRE |
| **Committee application** | Proposal reviewed | ADNI (reviewed "generally within two weeks"), EHRSHOT (Stanford AIMI DUA) |
| **Institutional agreement** | Your university must sign | All of Us (DURA can take months) |
| **Paid** | Fee | UK Biobank (student data-only fee £500 + VAT; standard tiers £3k–£9k) |

**Tip.** The **MIMIC-IV demo** and **MIMIC-IV demo in MEDS format** are open on PhysioNet, so you can write your whole pipeline before credentialing comes through.

---

## 2. Scorecard

Five criteria, 1–5 each: **Acc** = access ease, **Lab** = labels/scale, **Bench** = benchmark maturity, **Head** = headroom/novelty room, **Fit** = fit with your profile. The total is out of 25. AHBA scores **14** on the same scale.

### 2.1 Clinical EHR, ICU, ED and perioperative

| Dataset | Key facts | Access | Acc | Lab | Bench | Head | Fit | **Total** |
|---|---|---|---|---|---|---|---|---|
| [**MC-MED**](https://physionet.org/content/mc-med) (2025) | **118,385 adult ED visits** (Stanford, 2020–22). Continuous vitals, **waveforms (ECG, PPG, respiration)**, demographics, history, orders, meds, labs, imaging results, outcomes. | Credentialed | 3 | 5 | 3 | 5 | 5 | **21** |
| [**MIMIC-IV v3.1**](https://physionet.org/content/mimiciv/) | ~**364k individuals, 546k hospitalizations, 94k ICU stays**. Modules for ED, notes, ECG, CXR, echo, waveforms. | Credentialed | 3 | 5 | 5 | 2 | 5 | **20** |
| [**eICU-CRD**](https://physionet.org/content/eicu-crd/) | **200,859 ICU encounters, 139,367 patients, 208 hospitals** (2014–15). Natural multi-site splits. | Credentialed | 3 | 4 | 4 | 3 | 4 | **18** |
| [**INSPIRE**](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11192876/) | ~**130k surgical cases** (Korea, 2011–20). OR/ward/ICU vitals, labs (±6 months), meds, complications, mortality. | Credentialed | 3 | 4 | 2 | 4 | 3 | **16** |
| [**VitalDB**](https://physionet.org/content/vitaldb/) | **6,388 surgeries**, 486k tracks, 500 Hz waveforms, labs. Python API. [VitalBench (2025)](https://arxiv.org/pdf/2511.13757) adds a benchmark. | **Open** | 5 | 3 | 4 | 3 | 3 | **18** |
| [**EHRSHOT**](https://arxiv.org/pdf/2307.02028) | **6,739 longitudinal patients**, 15 few-shot tasks, released 141M-parameter EHR foundation model (CLMBR-T). | Stanford AIMI DUA | 2 | 3 | 5 | 3 | 5 | **18** |
| [**MEDS / MEDS-DEV**](https://physionet.org/content/mimic-iv-demo-meds/) | Standard EHR event format plus decentralized benchmark; used at 15+ institutions. | Open tooling | — | — | — | — | — | *infrastructure* |
| [**All of Us**](https://support.researchallofus.org/hc/en-us/articles/35013049400468) | EHR + wearables + surveys + genomics, very large. | Institutional DURA | 1 | 5 | 3 | 4 | 4 | **17** |
| [**UK Biobank**](https://www.ukbiobank.ac.uk/use-our-data/fees/) | 500k participants; imaging, omics, linked records. | Paid (£500 student) | 2 | 5 | 4 | 3 | 3 | **17** |

### 2.2 Healthcare agents and LLM benchmarks

| Dataset | Key facts | Access | Acc | Lab | Bench | Head | Fit | **Total** |
|---|---|---|---|---|---|---|---|---|
| [**MedAgentBench**](https://arxiv.org/html/2501.14654v2) (NEJM AI 2025) | **FHIR-compliant** virtual EHR: 100 patients, 700k+ records, **300 clinician-written tasks**. Best model **69.67%**. | Open | 5 | 3 | 4 | 4 | 5 | **21** |
| [**HealthAgentBench**](https://arxiv.org/abs/2606.31179) (2026) | **54 tasks across 7 categories** (EHR, notes, CXR, CT, whole-slide images). Strongest agent averages only **42%**. | Open code; some environments may need credentialed data | 4 | 3 | 4 | 5 | 4 | **20** |
| [**HealthBench**](https://arxiv.org/html/2505.08775v1) (2025) | **5,000** multi-turn health conversations, **48,562 rubric criteria**, 262 physicians from 60 countries. | Open | 5 | 4 | 5 | 2 | 3 | **19** |
| [**MedHELM**](https://arxiv.org/pdf/2505.23802) (2025) | Taxonomy of 121 tasks; 37 evaluations; LLM-jury grading. | Open framework; some sets need DUA | 4 | 4 | 5 | 2 | 3 | **18** |
| [**PMC-Patients**](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10728216/) | **167k patient summaries**, 3.1M patient–article and 293k patient–patient relevance labels. Built for retrieval and RAG. | **Open** (CC BY-NC-SA) | 5 | 5 | 4 | 3 | 4 | **21** |
| [**BioASQ14 @ CLEF 2026**](https://bioasq.org/) | Six shared tasks (QA, clinical summarization, clinical coding, information extraction). Workshop papers for participants. | Registration | 4 | 4 | 5 | 3 | 4 | **20** |

### 2.3 Medical imaging (radiology, dermatology, ophthalmology, neuro)

| Dataset | Key facts | Access | Acc | Lab | Bench | Head | Fit | **Total** |
|---|---|---|---|---|---|---|---|---|
| [**ReXGradient-160K**](https://arxiv.org/html/2505.00228v1) (2025) | **160k chest X-ray studies + reports, 109,487 patients, 3 US health systems, 79 sites**. Private test set on ReXrank. | Open (Hugging Face) | 5 | 5 | 4 | 3 | 3 | **20** |
| [**CheXpert Plus**](https://arxiv.org/pdf/2405.19538) (2024) | **223,228 image–report pairs**, 64,725 patients, demographics, 14 pathology labels. | Registration (AIMI) | 4 | 5 | 4 | 2 | 3 | **18** |
| [**CT-RATE**](https://huggingface.co/datasets/ibrahimhamamci/CT-RATE/blob/refs%2Fpr%2F102/README.md) | **25,692 chest CTs** (50,188 reconstructions), 21,304 patients, reports + abnormality labels. 3D, so GPU-heavy. | Gated (Hugging Face) | 4 | 4 | 4 | 3 | 3 | **18** |
| [**Merlin release**](https://arxiv.org/html/2406.06512v1) (*Nature* 2026) | **25,494 abdominal CT–report pairs** released with model and code. | Public release (check licence) | 4 | 4 | 3 | 4 | 3 | **18** |
| [**AbdomenAtlas 3.0**](https://arxiv.org/html/2501.04678v2) (ICCV 2025) | **9,262 CT + per-voxel tumour mask + report** triplets. | Open | 5 | 3 | 3 | 4 | 2 | **17** |
| [**OpenMind**](https://arxiv.org/html/2412.17041v2) (ICCV 2025) | **114k 3D brain MRI volumes**, 23 modalities, 800 OpenNeuro datasets, CC-BY. A pre-training corpus. | **Open** | 5 | 3 | 3 | 4 | 3 | **18** |
| [**ISIC 2024 / SLICE-3D**](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11324883/) | **401,059 lesion crops** with smartphone-like quality, from 7+ centres, with rich metadata. | **Open** | 5 | 5 | 4 | 3 | 3 | **20** |
| [**Harvard-FairVision30k / FairVLMed**](https://arxiv.org/html/2403.19949v2) | **30k** subjects (10k each of AMD, DR, glaucoma) with **6 demographic attributes**. FairVLMed has 10k image + clinical-note pairs. | Registration | 4 | 4 | 4 | 3 | 3 | **18** |
| [**ADNI**](https://adni.loni.usc.edu/?p=68) | Alzheimer's MRI/PET/biomarkers, longitudinal. | Application (~2 weeks) | 3 | 4 | 5 | 2 | 3 | **17** |

### 2.4 Physiological signals, sleep and wearables

| Dataset | Key facts | Access | Acc | Lab | Bench | Head | Fit | **Total** |
|---|---|---|---|---|---|---|---|---|
| [**MIMIC-IV-ECG**](https://registry.opendata.aws/mimic-iv-ecg/) | **~800k 12-lead diagnostic ECGs, ~160k patients**, 500 Hz, machine reports. Linking to MIMIC-IV outcomes needs credentials. | **Open** (ODbL) | 5 | 5 | 3 | 4 | 4 | **21** |
| [**CODE-15%**](https://zenodo.org/records/4916206) | **345,779 ECGs, 233,770 patients** (Brazil telehealth, 2010–16). | **Open** (CC BY 4.0) | 5 | 5 | 4 | 3 | 4 | **21** |
| [**PTB-XL**](https://www.physionet.org/content/ptb-xl/1.0.3/) | **21,799 ECGs, 18,869 patients**, 71 SCP statements. The standard benchmark, now near-saturated. | **Open** | 5 | 3 | 5 | 1 | 4 | **18** |
| [**OpenECG**](https://arxiv.org/html/2503.00711v1) (2025) | Benchmark protocol over **1.2M public ECGs from 9 centres**, with leave-one-dataset-out evaluation. | Open | 5 | 5 | 4 | 3 | 4 | *protocol* |
| [**PhysioNet Challenge 2026**](https://moody-challenge.physionet.org/2026/) | **Polysomnography → future cognitive impairment.** Multicentre EEG/ECG/respiration + sleep annotations. Results published in CinC proceedings. | Open (challenge) | 4 | 4 | 5 | 4 | 3 | **20** |
| [**NSRR (SHHS etc.)**](https://sleepdata.org/datasets/shhs) | Tens of thousands of sleep studies (EDF + outcomes), e.g., SHHS with >8,000 PSGs. | Free, DUA | 4 | 4 | 3 | 4 | 3 | **18** |
| [**GLOBEM**](https://proceedings.neurips.cc/paper_files/paper/2022/hash/9c7e8a0821dfcb58a9a83cbd37cc8131-Abstract.html) | **700+ user-years, 497 users** of phone and wearable sensing, with well-being and depression labels; tests cross-year generalization. | PhysioNet (check licence) | 3 | 3 | 4 | 4 | 4 | **18** |

### 2.5 Pathology, spatial omics and single-cell

| Dataset | Key facts | Access | Acc | Lab | Bench | Head | Fit | **Total** |
|---|---|---|---|---|---|---|---|---|
| [**HEST-1k**](https://arxiv.org/html/2406.16192v2) (NeurIPS 2024 D&B) | **1,108+ spatial-transcriptomics profiles, each paired with an H&E whole-slide image** (later versions list more); 131 cohorts, 25 organs. **HEST-Benchmark** (9 tasks) for gene-expression-from-histology. | Gated (Hugging Face) | 4 | 4 | 5 | 4 | 3 | **20** |
| [**STimage-1K4M**](https://proceedings.neurips.cc/paper_files/paper/2024/hash/3ef2b740cb22dcce67c20989cb3d3fce-Abstract.html) (NeurIPS 2024 D&B) | **1,149 slides, 4.29M spot–tile pairs**, 50 tissues (**22% brain**). | **Open** | 5 | 4 | 3 | 4 | 3 | **19** |
| [**Tahoe-100M**](https://www.biorxiv.org/content/10.1101/2025.02.20.639398.full.pdf) + [**Virtual Cell Challenge**](https://arcinstitute.org/news/virtual-cell-challenge-2026) | 100M cells, 50 cancer cell lines, ~1,100 drugs. The 2026 challenge is zero-shot prediction across unseen cell lines. | **Open** | 5 | 5 | 5 | 3 | 2 | **20** |
| [**CELLxGENE Census**](https://tessl.io/registry/skills/github/K-Dense-AI/scientific-agent-skills/cellxgene-census) | **217M+ cells (125M+ unique), 1,845 datasets** (2025-11-08 LTS release). | **Open** | 5 | 5 | 3 | 3 | 2 | **18** |
| SEA-AD (see AHBA report) | 84 Alzheimer's donors, ~8M nuclei, continuous pathology score. | **Open** | 5 | 4 | 3 | 4 | 2 | **18** |

### 2.6 Public health and population

| Dataset | Key facts | Access | Acc | Lab | Bench | Head | Fit | **Total** |
|---|---|---|---|---|---|---|---|---|
| [**CDC FluSight hub**](https://github.com/cdcepi/FluSight-forecast-hub) | Weekly probabilistic nowcasts and forecasts of flu hospital admissions and ED-visit share for every US state, plus all submitted forecasts and the ensemble. **Prospective** evaluation. | **Open** (GitHub) | 5 | 3 | 5 | 3 | 5 | **21** |
| **NHANES** | National survey: labs, exams, diet, accelerometry; public mortality linkage. ML benchmarks emerging, e.g., [NHANES accelerometry cardiometabolic benchmark (2026)](https://arxiv.org/abs/2606.30702). | **Open** | 5 | 4 | 3 | 3 | 5 | **20** |

### 2.7 Federated and multi-site

| Dataset | Key facts | Access | Total |
|---|---|---|---|
| [**FLamby**](https://proceedings.neurips.cc/paper_files/paper/2022/hash/232eee8ef411a0a316efa298d7be3c2b-Abstract.html) (NeurIPS 2022 D&B) | 7 healthcare datasets with **natural cross-silo splits** and baseline code. | Mixed (some components need separate access) | **19** |
| *Natural multi-site splits elsewhere* | eICU (208 hospitals), ReXGradient (79 sites), SLICE-3D (7+ centres), OpenECG (9 centres), MC-MED vs MIMIC-IV-ED | — | — |

---

## 3. Top 5 for your profile, with concrete paper angles

I chose these using the scorecard *plus* clinical weight and PhD-application value. So they aren't simply the five highest totals: PMC-Patients also scores 21 but is a narrower retrieval/RAG resource, and it appears under honourable mentions.

### 1) MC-MED (with MIMIC-IV / MIMIC-IV-ED / eICU for external validation) — strongest overall
- **Why:**
  - It was released March 2025, so few baselines exist and novelty is cheap.
  - It is one of the few large datasets that combines **continuous physiological waveforms with full ED EHR context and outcomes**.
  - Emergency-department deterioration is a clinically important problem that reviewers understand immediately.
  - It is messy, multimodal clinical data, which is where your engineering experience is a real advantage.
- **Paper angles:**
  - **Multimodal deterioration forecasting** (ICU admission, decompensation, critical intervention) from waveforms + vitals + orders, with **missing-modality robustness**: waveforms are often absent at other hospitals. Externally validate the EHR-only branch on MIMIC-IV-ED.
  - **Calibrated early-warning scores** with conformal prediction and alert-burden analysis (alerts per true event). This frames the work as healthcare-systems research.
  - **Self-supervised waveform encoders** (PPG/ECG/respiration) pretrained on MC-MED and evaluated label-efficiently.
- **Venues:** ML4H, CHIL, MLHC, AMIA; *npj Digital Medicine*, *JAMIA*, *JBI*.
- **Watch-outs:**
  - It is single-centre, so pair it with MIMIC-IV for external validation.
  - Define prediction times carefully to avoid label leakage from future orders.
  - Check whether new papers like [this 2025 preprint on adverse-event prediction](https://arxiv.org/pdf/2510.14286) already use MC-MED.

### 2) MedAgentBench / HealthAgentBench — fastest to a paper, best SWE fit
- **Why:**
  - It's open with no wait.
  - It's **FHIR-native**, which is the health-IT stack you already know.
  - There is clear headroom: 69.67% best on MedAgentBench and 42% on HealthAgentBench.
  - Agentic AI is a top hiring and PhD topic in 2026.
- **Paper angles:**
  - **Robustness of EHR agents to realistic FHIR messiness:** perturb the environment with unit variants, missing codes, duplicate resources and paginated or failing APIs, then measure the drop. This is a benchmark contribution only a health-IT engineer would think to build.
  - **Verifier-guided or constrained FHIR agents:** schema-validated actions and a "safety critic" for orders, measuring the safety vs success trade-off.
  - **Small open-weight models plus tool-use fine-tuning** on synthetic trajectories: can a 7–8B model approach frontier performance at a fraction of the cost?
- **Venues:** ML4H, NeurIPS Datasets & Benchmarks, AMIA, *JAMIA*, *NEJM AI*.
- **Watch-outs:** results date quickly as models change, so make the *environment and method* the contribution, not a leaderboard snapshot.

### 3) Open ECG at scale (MIMIC-IV-ECG + CODE-15% + PTB-XL, OpenECG protocol) + PhysioNet Challenge
- **Why:**
  - It's fully open and large: about 800k + 346k + 22k ECGs.
  - It's multi-country and multi-site, and 1D signals train on one GPU.
  - Reviewer-recognized benchmarks exist.
  - The annual **PhysioNet Challenge** leads to a **CinC proceedings paper**, which is one of the most accessible first publications. The **2026 topic, sleep → cognitive impairment**, connects to your brain interest.
- **Paper angles:**
  - **Site-robust, calibrated ECG foundation models:** leave-one-site-out evaluation, as in the OpenECG protocol, plus conformal triage.
  - **ECG–text contrastive pretraining** using MIMIC-IV-ECG machine reports, then zero-shot transfer to CODE-15% labels.
  - **Label-efficient fine-tuning** and **subgroup fairness** across countries.
- **Venues:** CinC, ML4H, CHIL, *IEEE JBHI*, *npj Digital Medicine*.
- **Watch-outs:** PTB-XL classification alone is saturated, so the paper has to be about generalization, calibration or pretraining.

### 4) HEST-1k / STimage-1K4M — the data-rich successor to your AHBA interest
- **Why:**
  - Same spatial-biology flavour as the AHBA plan, but with **1,000+ slides of paired histology and expression**.
  - NeurIPS benchmarks and baselines already exist.
  - STimage-1K4M is about 22% brain tissue.
- **Paper angles:**
  - **Port the "Donors as Views" idea:** use *slides/patients as views* to learn gene representations, then test on HEST-Benchmark.
  - **Uncertainty-calibrated gene-expression prediction from H&E** under cross-organ and cross-platform shift.
  - **Which pathology foundation model features carry molecular information?**
- **Venues:** NeurIPS D&B, MICCAI, ICML/ICLR workshops (MLGenX, LMRL), *Bioinformatics*.
- **Watch-outs:** this is an active, competitive area, so be precise about your splits (patient-level, cohort-level) and compute budget.

### 5) CDC FluSight hub + NHANES — leverages your public-health background
- **Why:** both are open. FluSight gives **prospective, real-time evaluation**: your forecasts are scored against data that didn't exist when you submitted. That's the strongest possible defence against leakage, and few ML papers can claim it.
- **Paper angles:**
  - **Time-series foundation models** (e.g., Chronos, TimesFM, Moirai) vs FluSight ensembles, evaluated prospectively.
  - **Conformal calibration of probabilistic forecasts.**
  - On NHANES: **tabular foundation models** (TabPFN-style) for cardiometabolic risk from accelerometry plus labs, with survey-weighted fairness analysis.
- **Venues:** *PLOS Comp Bio*, *Epidemics*, ML4H, KDD Health Day, *AJE*.
- **Watch-outs:** less ML prestige than imaging or EHR. Make the ML contribution (calibration, foundation-model evaluation) explicit.

### Honourable mentions
- **Radiology vision-language** (ReXGradient-160K, CheXpert Plus, CT-RATE, Merlin, AbdomenAtlas 3.0): lots of data, but crowded and, for 3D CT, GPU-heavy. The best angle is *multi-site robustness and fairness of report generation* (ReXGradient has 79 sites).
- **Tahoe-100M / Virtual Cell Challenge:** high visibility in computational biology, but very competitive and more compute-heavy.
- **SLICE-3D (ISIC 2024):** smartphone-like skin images with metadata from 7+ centres; good for fairness and telehealth framing.
- **PMC-Patients:** open, large, ideal for RAG and retrieval papers about clinical decision support.
- **OpenMind + ADNI:** brain MRI self-supervision with clinical downstream labels.
- **FLamby / eICU:** if you want federated learning to be your theme.

---

## 4. Single pick and a 90-day plan

**Primary project: MC-MED.** Its novelty headroom (new dataset), clinical weight (ED deterioration), ML depth (multimodal waveforms + EHR, missing modalities, calibration) and fit with your engineering background together beat every other option. MIMIC-IV gives reviewer-proof external validation through the same PhysioNet credential.

**Parallel quick win: MedAgentBench.** Start it during the credentialing wait. It needs no data access, uses your FHIR expertise, and can become a workshop paper in about 3 months.

| Weeks | MC-MED track | MedAgentBench track |
|---|---|---|
| 0–2 | Complete CITI training and PhysioNet credentialing; build the pipeline on the open **MIMIC-IV demo / MEDS demo** | Run baseline agents; reproduce published numbers |
| 3–6 | Cohort and task definitions (prediction times, leakage audit); EHR-only baselines (GBM, MEDS-compatible transformer) | Build the FHIR-perturbation suite; measure robustness drop |
| 7–10 | Waveform encoders (SSL); multimodal fusion with modality dropout; calibration | Verifier / constrained-action method; ablations |
| 11–13 | External validation (EHR branch on MIMIC-IV-ED); subgroup analysis; draft | **Workshop submission** (ML4H findings, or a NeurIPS/ICLR workshop) |

---

## 5. How this connects to the AHBA plan

- AHBA stays a good *secondary* computational-biology project; Project A in the AHBA report is still valid. For a *first* paper and a Healthcare-AI PhD application, though, a clinically labelled dataset is the better bet.
- **Continuity options:**
  - The "Donors as Views" method transfers to **HEST-1k** (patients as views).
  - Your brain interest continues through the **PhysioNet 2026 sleep → cognitive-impairment challenge**, **OpenMind/ADNI** and **SEA-AD**.
  - AHBA can serve as a *prior* inside any brain-imaging project.

---

## 6. Practical pitfalls

1. **Saturation:** PTB-XL superclass classification, MIMIC in-hospital mortality, MedQA-style multiple-choice QA, and CheXpert 5-label classification are near ceiling. Choose tasks where gains are still measurable.
2. **Leakage:** split by patient and by time; check that features don't use information recorded after the prediction time. For agents, don't let the benchmark leak into prompts.
3. **Licences:** several datasets are **non-commercial** (CC BY-NC, e.g., PMC-Patients) or **gated**. Fine for academic papers, but check before releasing models trained on them.
4. **Credentialed data + LLM APIs:** PhysioNet's data-use agreement forbids sharing credentialed data with third parties. Read PhysioNet's guidance on using online LLM services *before* sending any MIMIC or MC-MED text to an API; local or open-weight models avoid the issue.
5. **External validation:** reviewers in clinical ML expect it. Pick dataset pairs that share a task: MC-MED ↔ MIMIC-IV-ED, MIMIC-IV ↔ eICU, CODE-15% ↔ MIMIC-IV-ECG ↔ PTB-XL, ReXGradient ↔ CheXpert Plus.

---

## Sources

**EHR / ICU / ED / perioperative**
- MIMIC-IV (PhysioNet): https://physionet.org/content/mimiciv/
- MC-MED (PhysioNet): https://physionet.org/content/mc-med
- eICU-CRD: https://physionet.org/content/eicu-crd/ · paper: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6132188/
- INSPIRE: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11192876/ · https://www.physionet.org/content/inspire/
- VitalDB: https://physionet.org/content/vitaldb/ · https://www.ncbi.nlm.nih.gov/pmc/articles/PMC9178032/ · VitalBench: https://arxiv.org/pdf/2511.13757
- EHRSHOT: https://arxiv.org/pdf/2307.02028 · https://github.com/som-shahlab/ehrshot-benchmark
- MEDS: https://www.iclr.cc/virtual/2024/23574 · MIMIC-IV demo (MEDS): https://physionet.org/content/mimic-iv-demo-meds/
- All of Us access: https://support.researchallofus.org/hc/en-us/articles/35013049400468
- UK Biobank fees: https://www.ukbiobank.ac.uk/use-our-data/fees/
- MC-MED-related preprint: https://arxiv.org/pdf/2510.14286

**Agents / LLM benchmarks**
- MedAgentBench: https://arxiv.org/html/2501.14654v2
- HealthAgentBench: https://arxiv.org/abs/2606.31179 · https://microsoft.github.io/HealthAgentBench/
- HealthBench: https://arxiv.org/html/2505.08775v1
- MedHELM: https://arxiv.org/pdf/2505.23802
- PMC-Patients: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10728216/
- BioASQ: https://bioasq.org/

**Imaging**
- ReXGradient-160K: https://arxiv.org/html/2505.00228v1
- CheXpert Plus: https://arxiv.org/pdf/2405.19538 · https://aimi.stanford.edu/datasets/chexpert-plus
- CT-RATE: https://huggingface.co/datasets/ibrahimhamamci/CT-RATE/blob/refs%2Fpr%2F102/README.md
- Merlin: https://arxiv.org/html/2406.06512v1 · NIH news: https://www.nih.gov/news-events/news-releases/automated-ct-scan-analysis-could-fast-track-clinical-assessments
- AbdomenAtlas 3.0 / RadGPT: https://arxiv.org/html/2501.04678v2
- OpenMind: https://arxiv.org/html/2412.17041v2
- SLICE-3D / ISIC 2024: https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11324883/ · https://challenge2024.isic-archive.com
- Harvard-FairVision / FairVLMed: https://arxiv.org/html/2403.19949v2 · https://openaccess.thecvf.com/content/CVPR2024/html/Luo_FairCLIP_Harnessing_Fairness_in_Vision-Language_Learning_CVPR_2024_paper.html
- ADNI access: https://adni.loni.usc.edu/?p=68

**Signals / sleep / wearables**
- MIMIC-IV-ECG: https://registry.opendata.aws/mimic-iv-ecg/
- CODE-15%: https://zenodo.org/records/4916206
- PTB-XL: https://www.physionet.org/content/ptb-xl/1.0.3/ · https://www.ncbi.nlm.nih.gov/pmc/articles/PMC7248071/
- OpenECG: https://arxiv.org/html/2503.00711v1
- PhysioNet Challenge 2026: https://moody-challenge.physionet.org/2026/
- NSRR: https://sleepdata.org/datasets/shhs · https://sleepdata.org/about
- GLOBEM: https://proceedings.neurips.cc/paper_files/paper/2022/hash/9c7e8a0821dfcb58a9a83cbd37cc8131-Abstract.html

**Pathology / spatial / single-cell**
- HEST-1k: https://arxiv.org/html/2406.16192v2
- STimage-1K4M: https://proceedings.neurips.cc/paper_files/paper/2024/hash/3ef2b740cb22dcce67c20989cb3d3fce-Abstract.html
- Tahoe-100M: https://www.biorxiv.org/content/10.1101/2025.02.20.639398.full.pdf · Arc Virtual Cell Atlas: https://arcinstitute.org/news/arc-virtual-cell-atlas-launch
- Virtual Cell Challenge 2025 wrap-up / 2026: https://arcinstitute.org/news/virtual-cell-challenge-2025-wrap-up · https://arcinstitute.org/news/virtual-cell-challenge-2026
- CELLxGENE Census (release summary): https://tessl.io/registry/skills/github/K-Dense-AI/scientific-agent-skills/cellxgene-census

**Public health**
- CDC FluSight forecast hub: https://github.com/cdcepi/FluSight-forecast-hub
- NHANES accelerometry benchmark: https://arxiv.org/abs/2606.30702

**Federated**
- FLamby: https://proceedings.neurips.cc/paper_files/paper/2022/hash/232eee8ef411a0a316efa298d7be3c2b-Abstract.html
