# Allen Human Brain Atlas: Machine Learning Research Opportunities for a Healthcare-AI PhD Track

*An advisor-style assessment, written October 2026*

**Audience:** a software/ML engineer with healthcare-technology experience and little neuroscience background. They are doing a Master's now and plan to apply for a PhD in Healthcare AI, Medical AI, Computational Biology or Biomedical ML.

**Scope:** the Allen Human Brain Atlas (AHBA) download page (`human.brain-map.org/static/download`) and the Allen resources around it. The goal is ML methods research, not neuroscience discovery.

> **Verification note.** The research sandbox could not reach `brain-map.org` directly; the network proxy blocked it. The dataset facts below come from the published atlas papers, tool documentation and web search. Before you plan experiments around them, check the per-donor details (hemispheres, MRI/DTI availability, RNA-seq sample pairing) against the Allen Institute technical white papers on the atlas's Documentation tab. Citations I found and confirmed through search are linked. A few well-known older references are cited from memory and marked (†); please verify those too.

---

## TL;DR

1. **The AHBA is a reference atlas, not a clinical dataset.** It covers 6 neurotypical adult donors, 3,702 tissue samples, about 20k genes, MRI coordinates for every sample, plus ISH images and a histology atlas. If you treat *donors* as the training samples you have n = 6, which is a dead end for most ML. Instead, treat it as one of three things:
   - a **gene-level dataset** (n ≈ 15–20k genes),
   - a **sparse spatial field** to model,
   - a **normative prior** joined to clinical data.
2. **The best ML projects** come from what makes the atlas statistically hard: sparse, irregular sampling; multiple donors with only partially overlapping views; strong spatial autocorrelation (which inflates false positives); and no outcome labels. These are general problems in biomedical ML, so methods built here transfer.
3. **The three recommended projects:**
   - **(A) "Donors as Views":** self-supervised spatial gene representations for prioritizing brain-disorder genes, evaluated with temporal and literature-bias holdouts.
   - **(B) Probabilistic neural fields:** donor-aware neural fields with calibrated uncertainty for sparse transcriptomic atlases.
   - **(C) Statistically guarded LLM agents:** a verifiable benchmark plus guardrails for agents doing imaging-transcriptomics analysis.
4. **If you can do only one project, do (A).** It solves the n = 6 problem by design. Its ML contribution is clear (a new SSL view construction, multimodal foundation-model comparison, leakage-aware evaluation). Its healthcare link is concrete: genetically supported drug targets succeed about 2.6× more often ([Minikel et al., *Nature* 2024](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11096124/)). It needs one GPU and almost no neuroscience, and it produces a workshop paper in about 3–4 months and a full paper in about 6–7.

---

## Step 1 — Dataset Assessment

### 1.1 What is on the download page (the AHBA family)

| Component | What it is | Scale | ML-relevant notes |
|---|---|---|---|
| **Microarray (normalized)** | Genome-wide bulk expression from dissected tissue samples. Custom Agilent 8×60K array. | **6 donors, 3,702 samples, 58,692 probes** ([Ruffle et al. 2024](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11267301/)). After probe→gene mapping, ≈20.7k genes (Wagstyl et al. use 20,781). After standard intensity filtering in `abagen`, ≈15–16k genes. | Per donor: `MicroarrayExpression.csv`, `PACall.csv` (present/absent calls), `Probes.csv`, `SampleAnnot.csv` (structure IDs, native MRI voxel + **MNI coordinates**), `Ontology.csv`. Cross-donor normalized. A few GB in total. |
| **RNA-seq** | Bulk RNA-seq on a subset of structures. | **2 donors** (H0351.2001, H0351.2002), on the order of ~120 samples. | Tiny, but it gives a cross-platform view of the same brains. Check sample-level pairing with the microarray in the white paper. |
| **MRI / DTI** | Post-mortem T1w/T2w MRI per donor; diffusion data for donors where available. NIfTI. | 6 donors. | Every microarray sample has coordinates in its donor's MRI and in MNI space. This is what makes AHBA a bridge between molecules and imaging. |
| **ISH (in situ hybridization)** | Cellular-resolution images of single-gene expression on tissue sections, plus Nissl. | Five studies: **Cortex** (~1,000 genes in visual and temporal cortex), **Subcortex**, **Neurotransmitter**, **Schizophrenia** (60 genes in DLPFC, >50 control + SCZ cases), **Autism** (25 genes in frontal/temporal/occipital cortex, 11 ASD + 11 controls). | The **only disease-labelled data** in the AHBA. Accessed through the Allen API, not a single zip. Source: [Allen ISH documentation via search](https://community.brain-map.org/t/human-brain-atlas-in-situ-hybridization-ish-data/2872). |
| **Allen Human Reference Atlas** | Ding et al. 2016: 1,356 Nissl/IHC plates at 1 µm/pixel, MRI and DWI from one brain, with comprehensive structure annotation. **3D 2020 version:** 141 structures on MNI ICBM 2009b. | 1 brain (histology); 1 MNI parcellation. | A segmentation and annotation resource. Large-format histology. ([README](https://download.alleninstitute.org/informatics-archive/allen_human_reference_atlas_3d_2020/version_1/README.pdf)) |
| **Structure ontology** | Hierarchical tree of brain structures. | Thousands of nodes. | A natural hierarchy and graph for labels and priors. |

**Donors.** Six adults with no known neuropsychiatric or neuropathological history: five male, one female, aged roughly 24–57 (†). Samples per donor are 946, 893, 363, 529, 470 and 501 (sum 3,702). **Only two donors (H0351.2001 and .2002) were sampled in both hemispheres; the other four are left hemisphere only** (†, verify).

### 1.2 Labels and metadata available

- **Intrinsic:**
  - structure label per sample (hierarchical ontology)
  - 3D coordinates (native and MNI)
  - donor ID, age, sex
  - probe annotations and present/absent calls
  - MRI per donor
- **Disease labels:** none in the microarray (all controls). The ISH Schizophrenia and Autism studies have case/control labels, but only for a few dozen genes and small donor counts.
- **External labels you can join (this is where ML tasks come from):**
  - *Gene-level:* GO; cell-type markers (e.g., [Siletti et al. 2023](https://www.pubmed.ncbi.nlm.nih.gov/37824663/), 3M+ nuclei); disease genes (SFARI Gene, SCHEMA, PGC/AD GWAS, GWAS Catalog, Open Targets — all versioned and dated, which enables temporal holdouts); PubMed counts per gene (to measure literature bias).
  - *Region-level:* ENIGMA case–control effect maps (ENIGMA Toolbox †); PET receptor and other maps (neuromaps †); HCP connectomes.

### 1.3 Data quality considerations

1. **Processing choices change results.** Probe re-annotation, probe selection, intensity filtering, sample→region assignment, normalization and donor handling all alter downstream findings. See Arnatkevičiūtė et al. 2019 (†) and `abagen` (Markello et al. 2021, †). *For ML, the preprocessing pipeline is a hyperparameter; report sensitivity to it.*
2. **Donor effects.** Inter-donor variance is large and only six donors estimate it. Normalization removes some biology along with batch effects.
3. **Post-mortem factors.** Post-mortem interval, agonal state, RNA integrity, age range and male skew all add noise that can't be recovered.
4. **Non-uniform sampling.** About 1,300 cortical samples sit across six left hemispheres, while some structures have only a few samples. There is also registration error of a few mm in MNI space.
5. **Microarray limitations.** Dynamic range is limited, probes cross-hybridize, and many genes sit near background.
6. **Spatial autocorrelation.** Nearby samples look alike. Naive gene-category enrichment on spatial atlases showed **>500× average inflation of false positives** for GO categories ([Fulcher et al., *Nat Commun* 2021](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8113439/)).
7. **Bulk tissue.** Each sample mixes cell types, so expression gradients partly reflect composition (neuron/glia ratio, white-matter contamination).
8. **Low effective dimensionality.** About 20k genes collapse onto a few dominant axes ([Dear et al., *Nat Neurosci* 2024](https://link.springer.com/10.1038/s41593-024-01624-4) found three).

### 1.4 Existing research and common uses

- **Atlas biology:** Hawrylycz 2012 *Nature* and 2015 *Nat Neurosci* (†) introduced "differential stability" and canonical gene modules.
- **Imaging transcriptomics** (the dominant use) correlates regional gene expression with MRI phenotypes, disease maps and PET maps:
  - Burt 2018 (†)
  - [Hansen 2022, *Nat Commun*](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC9365855): cross-disorder cortical abnormalities
  - Seidlitz 2020 (†)
  - Tooling: `abagen`, ENIGMA Toolbox, neuromaps, BrainSpace, BrainSMASH, and [eigenstrapping](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12330862/) (Koussis et al., *Imaging Neurosci* 2025)
  - Inference pitfalls: Fulcher 2021; [Wei et al. 2022](https://www.biorxiv.org/content/10.1101/2021.02.22.432228.full.pdf)
- **Transcriptional axes and continuous maps:**
  - [Dear et al. 2024](https://link.springer.com/10.1038/s41593-024-01624-4): three generalizable cortical components linked to autism and schizophrenia
  - [Wagstyl et al. 2024, *eLife*](https://elifesciences.org/articles/86933): MAGICC, vertex-level maps for >20k genes
  - Gryglewski et al. 2018 *NeuroImage*: variogram/kriging interpolation to voxel maps
- **ML on AHBA (thin so far):**
  - [Ruffle et al. 2024, *HBM*](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11267301/): deep autoencoders give the best compressed representation
  - [Yu et al. 2025, arXiv:2506.11158](https://arxiv.org/html/2506.11158v1): implicit neural representations to interpolate AD-risk genes brain-wide
  - [BrainAlign, *Nat Commun* 2024](https://ideas.repec.org/a/nat/natcom/v15y2024i1d10.1038_s41467-024-50608-2.html): SSL human–mouse alignment with heterogeneous GNNs
  - [Negi & Guda 2017, *Sci Rep*](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC5429860/): simple classifiers predicting autism and Parkinson's genes from AHBA expression

### 1.5 Limitations from an ML perspective

| Limitation | Consequence | Design response |
|---|---|---|
| n = 6 donors | Donor-level generalization rests on 6 folds; you can't learn inter-individual distributions. | Make **genes**, **sample sets** or **region pairs** the unit. Use AHBA as a prior for clinical data. |
| No outcome labels | No supervised clinical tasks inside AHBA. | Use external labels (disease genes, PET/ENIGMA maps) or self-supervision. |
| Spatial autocorrelation and donor clustering | Random cross-validation leaks, so performance is inflated. | Spatially blocked and leave-one-donor-out CV; spatial-null baselines. |
| Donors sampled at different locations | No pixel-aligned views across donors. | Set/point encoders, neural fields, or parcellation. |
| Hemisphere and sex imbalance | Bias, and weak fairness or generalization claims. | Report it; restrict claims to the left hemisphere. |
| Legacy microarray; small RNA-seq | Platform shift relative to modern data. | Cross-platform calibration; validate on GTEx, PsychENCODE or snRNA-seq. |
| Post-mortem bulk vs in-vivo MRI | Expression "predicted from MRI" is population-level, not patient-level. | Don't overclaim individual "virtual transcriptomics". |
| Biological ground truth needs wet lab | Hard to "validate discoveries". | Pick tasks with **computational ground truth**: held-out samples or donors, database labels, **temporal holdouts**, planted signals. |

### 1.6 What makes AHBA unique

- It is the **only anatomically comprehensive, genome-wide, whole-brain human expression atlas with a 3D coordinate for every sample**, registered to MRI. Compare:
  - GTEx: many donors, ~13 brain regions, no coordinates
  - BrainSpan: developmental, few regions
  - Single-cell atlases: cells, no tissue space
  - UK Biobank: imaging, no expression
  - Spatial transcriptomics: 2D, mm-scale fields of view
- **It is multi-scale:** bulk expression, cellular ISH, MRI/DTI and a histology atlas.
- **It is the de facto standard reference** for imaging transcriptomics, so a better method built on it gets an immediate, large audience.
- **Its data is natively geometric** (3D fields, surfaces, graphs), not a tabular matrix. That's a natural setting for modern geometric deep learning.

### 1.7 Choose your unit of analysis — the key decision

| Unit | n | Verdict |
|---|---|---|
| Donors | 6 | ✗ Not viable as a training set. |
| Tissue samples | 3,702 (correlated) | ~ Fine for field modelling, with spatial CV. |
| Regions (parcellated) | 68–400 | ~ Small, but the standard unit for imaging joins. |
| **Genes** | **~15–20k** | ✓ **Large, labelable from databases, ideal for SSL.** |
| Gene × donor views | ~90–120k | ✓ Natural multi-view SSL. |
| Region pairs (edges) | 10³–10⁵ | ✓ Graph/link tasks, but distance confounds them. |

This table drives the rest of the report.

---

## Step 2 — 18 ML Research Directions

### 2.1 Overview

| # | Short title | Primary ML task type | Novelty | Pub. potential | Difficulty | Timeline |
|---|---|---|---|---|---|---|
| 1 | Donors-as-views SSL gene embeddings for disease-gene prioritization | SSL / Representation / Multimodal | High | High | Medium | 5–7 mo |
| 2 | Probabilistic, donor-aware neural fields for transcriptomic atlases | Regression / Generative (INR) / Uncertainty | Med-High | High | Med-High | 6–8 mo |
| 3 | Statistically guarded LLM agents for imaging transcriptomics | Agentic AI / Evaluation | Med-High | Med-High | Medium | 4–6 mo |
| 4 | Spatial leakage in brain-map ML: benchmark and protocol | Evaluation methodology / Regression | Medium | High | Low-Med | 3–5 mo |
| 5 | Transcriptomic positional encodings for neuroimaging disease models | Graph learning / Transfer / XAI | Med-High | Medium | Medium | 5–7 mo |
| 6 | Cross-species contrastive transfer (mouse ISH ↔ human AHBA) | Transfer / Contrastive | Medium | Medium | Med-High | 6–9 mo |
| 7 | Active spatial sampling for atlas construction | Active learning / Bayesian design / RL | Med-High | Medium | Med-High | 6–9 mo |
| 8 | MRI → transcriptome "virtual transcriptomics" with honest evaluation | Multimodal regression | Medium | Low-Med | High | 6–9 mo |
| 9 | ISH-CLIP: vision–language alignment of ISH images and gene knowledge | Vision-language / Contrastive | Med-High | Medium | Med-High | 6–9 mo |
| 10 | Pathology foundation models → neuroanatomy, plus few-shot disease ISH | Transfer / Few-shot / MIL | Medium | Med-High | Low-Med | 3–5 mo |
| 11 | Spatially aware Bayesian cell-type deconvolution with uncertainty | Probabilistic modelling | Medium | Medium | Med-High | 6–9 mo |
| 12 | Cross-platform translation (microarray ↔ RNA-seq ↔ GTEx) | Domain adaptation | Low-Med | Low-Med | Low-Med | 3–4 mo |
| 13 | Learned generative null models for brain maps | Generative / Statistical ML | Med-High | Medium | High | 8–12 mo |
| 14 | Heterogeneous-graph atlas completion (missing hemispheres/regions) | Graph learning / Tensor completion | Medium | Medium | Medium | 4–6 mo |
| 15 | Causal disentanglement: cell composition vs intrinsic regulation | Causal representation learning | High | Low-Med | High | 9–12 mo |
| 16 | Federated, spatially aware imaging-transcriptomics | Federated learning | Medium | Low-Med | Medium | 5–7 mo |
| 17 | Atlas-grounded RAG / tool-use QA with verifiable answers | RAG / LLM evaluation | Medium | Med-High | Low-Med | 3–4 mo |
| 18 | Transcriptome → connectome link prediction, distance-debiased | Graph learning (link prediction) | Low-Med | Low-Med | Medium | 4–6 mo |

### 2.2 Idea cards

#### Idea 1 — Donors as Views: self-supervised spatial gene representations for brain-disorder gene prioritization
- **Problem.** Today's gene embeddings come from text (GenePT), protein sequence (ESM), networks (STRING) or dissociated cells (Geneformer, scGPT, TranscriptFormer). None encode *where in the intact human brain* a gene is expressed. Can SSL on AHBA learn **donor-invariant spatial gene representations** that improve prioritization of neuropsychiatric and neurodegenerative risk genes, especially *understudied* ones and genes discovered *after* the training cutoff?
- **Task type.** Self-supervised / contrastive representation learning; multimodal fusion; positive–unlabeled few-shot classification downstream.
- **Why it's ML, not domain science.**
  - The contribution is a new **view construction**: the same gene measured in different donors at *different, irregular* locations forms a natural positive pair.
  - Set/point encoders handle irregular spatial samples.
  - A **leakage-aware evaluation protocol** covers temporal, literature-bias and paralog splits.
  - A principled answer to "what does spatial expression add beyond foundation models?"
  - The biology comes from external labels; you don't need to interpret it.
- **Significance.** Genetic support raises drug-program success about 2.6× ([Minikel 2024](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11096124/)). CNS is the therapeutic area with the highest failure rates.
- **Novelty.** **High.** I found no published donor-as-view SSL on AHBA. Prior gene-level work uses simple classifiers on AHBA ([Negi & Guda 2017](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC5429860/)) or developmental BrainSpan networks ([forecASD](https://www.biorxiv.org/content/10.1101/370601v3.full), [DeepND](https://www.biorxiv.org/content/10.1101/2020.06.13.150201.full.pdf)).
- **Publication potential.** High: a workshop paper in about 3–4 months; a full paper at ISMB, PSB, ML4H, *Bioinformatics* or *Genome Biology* class venues.
- **Difficulty.** Medium. **Timeline:** 5–7 months.

#### Idea 2 — Probabilistic, donor-aware neural fields for sparse transcriptomic atlases
- **Problem.** Every imaging-transcriptomics result depends on interpolating about 500 samples per donor to voxels, vertices or parcels. Current methods produce point estimates without calibrated uncertainty and ignore donor variation: nearest-sample assignment, smoothing, kriging (Gryglewski 2018), MAGICC, and the deterministic INR of [Yu et al. 2025](https://arxiv.org/html/2506.11158v1).
- **Task type.** Regression; implicit neural representations; uncertainty estimation; conformal prediction.
- **Why it's ML.** Four method components:
  - a conditional neural field with **donor latent codes** (auto-decoder)
  - a **low-rank multi-output field** that scales to all ~15k genes
  - **conformal calibration** under spatial dependence with only 6 exchangeable units
  - **uncertainty propagation** into downstream hypothesis tests

  Each is an open methodological problem.
- **Significance.** Every downstream disease or PET association inherits interpolation error. Calibrated uncertainty reduces false-positive "mechanisms".
- **Novelty.** Medium-high. Related work: INR on AHBA (Yu 2025); INR for spatial transcriptomics ([SUICA, ICML 2025](https://proceedings.mlr.press/v267/zhu25af.html)); [Bayesian neural fields](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11390735) (general purpose); [GPSA](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10482692/) (GP alignment). The donor-aware, all-gene, calibrated, downstream-validated combination appears open.
- **Publication potential.** High (MICCAI, IPMI, MIDL, *NeuroImage*/*Imaging Neuroscience*, *MedIA*).
- **Difficulty.** Medium-high. **Timeline:** 6–8 months.

#### Idea 3 — Statistically guarded LLM agents for imaging transcriptomics (benchmark + guardrails)
- **Problem.** Autonomous bio-agents now run analyses end to end: [Biomni/BixBench](https://arxiv.org/pdf/2503.00096v2), [SpatialAgent](https://www.biorxiv.org/content/10.1101/2025.04.03.646459v1), [CellVoyager](https://neuroscience.stanford.edu/node/5300), [NeuroAgent](https://arxiv.org/pdf/2605.06584). Imaging transcriptomics is full of statistical traps: spatial autocorrelation, GCEA inflation, donor leakage. **Do agents fall into these traps, and can tool-level guardrails prevent it?**
- **Task type.** Agentic AI; LLM evaluation; trustworthy AI for science.
- **Why it's ML.** A **procedurally generated, contamination-resistant benchmark** with planted-signal and null tasks, so ground truth is known by construction. Plus a measurable **guardrail method** (validity-enforcing tool APIs and a statistical critic) and calibration analysis of agent claims.
- **Significance.** The same failure modes (dependence, leakage, multiple testing) affect clinical data analysis by agents.
- **Novelty.** Medium-high. General hypothesis-testing reliability is starting to be studied ([P-Bench/Fisher-R1, 2026](https://huggingface.co/papers/2608.07437)), and BixBench showed agents at about 17% open-answer accuracy. Nothing I found targets *spatially dependent biomedical data*.
- **Publication potential.** Medium-high (NeurIPS Datasets & Benchmarks, agents-for-science workshops, ML4H).
- **Difficulty.** Medium. **Timeline:** 4–6 months.

#### Idea 4 — Spatial leakage in brain-map ML: benchmark and evaluation protocol
- **Problem.** Models that predict brain maps (disease effects, PET, connectivity) from AHBA features are usually evaluated with random CV over regions, which leaks through spatial autocorrelation. Spin and surrogate nulls exist for *correlations* but aren't standard for *predictive ML*.
- **Task type.** Evaluation methodology; regression.
- **Why it's ML.** It defines spatially blocked and surrogate-based CV for predictive models and quantifies inflation across model classes (linear → PLS → GBM → GNN → foundation-model features).
- **Significance.** Reproducibility of claims about disease mechanisms.
- **Novelty.** Medium. Analogues exist: connectome leakage ([Rosenblatt et al., *Nat Commun* 2024](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10901797/)) and spatial CV in remote sensing ([Kattenborn 2022](https://rsc4earth.de/publication/kattenborn-spatially-2022/)). The brain-map synthesis is open.
- **Publication potential.** High (*NeuroImage*, *Imaging Neuroscience*, *HBM*, ML4H).
- **Difficulty.** Low-medium. **Timeline:** 3–5 months. *Best used as the evaluation backbone for ideas 1, 2 and 5.*

#### Idea 5 — Transcriptomic positional encodings: do molecular priors help neuroimaging disease models?
- **Problem.** AHBA region features are *identical for every subject*. Injecting them into GNN or transformer disease classifiers is therefore a **biological positional encoding**. Does biology help beyond learned, Laplacian or random-smooth encodings, especially with little data and across sites?
- **Task type.** Graph learning; transfer; explainable AI.
- **Why it's ML.** A clean ablation that controls for smoothness: AHBA encoding vs learned vs Laplacian-eigenmap vs **spin-permuted AHBA** (keeps spatial smoothness, breaks the biology). It tests generalization under site shift (ABIDE sites; ADNI → OASIS).
- **Significance.** Direct clinical AI (autism, Alzheimer's) and biologically grounded explanations.
- **Novelty.** Medium-high in framing; the spin-permuted control is the new piece.
- **Publication potential.** Medium; null results are a real risk.
- **Difficulty.** Medium. **Timeline:** 5–7 months (ABIDE is open after registration; ADNI needs an application).

#### Idea 6 — Cross-species contrastive transfer: mouse ISH (~20k genes) ↔ human AHBA
- **Problem.** The mouse atlas has dense 3D ISH for about 20k genes (Lein 2007 †); human data is sparse bulk. Learn joint gene and region embeddings that transfer mouse knowledge to human, with uncertainty about non-conserved genes.
- **Task type.** Transfer learning; contrastive; optimal transport.
- **Why it's ML.** Alignment of partially paired domains: orthologs anchor genes, regions are unpaired.
- **Significance.** The translational validity of mouse models matters for CNS drug failure.
- **Novelty.** Medium; [BrainAlign (2024)](https://ideas.repec.org/a/nat/natcom/v15y2024i1d10.1038_s41467-024-50608-2.html) already covers much of this.
- **Publication potential.** Medium. **Difficulty:** Medium-high. **Timeline:** 6–9 months.

#### Idea 7 — Active spatial sampling: where should the next atlas sample?
- **Problem.** Future human atlases have limited budgets. Choose tissue locations (and gene panels) that best reconstruct whole-brain expression, simulated by subsampling AHBA donors.
- **Task type.** Active learning; Bayesian experimental design; RL/bandits.
- **Why it's ML.** Acquisition functions over neural-field posteriors (Idea 2); non-myopic design.
- **Significance.** Efficient use of brain-bank tissue; spatial-transcriptomics panel design.
- **Novelty.** Medium-high. **Publication potential:** Medium. **Difficulty:** Medium-high. **Timeline:** 6–9 months (best as a follow-on to Idea 2).

#### Idea 8 — MRI → transcriptome "virtual transcriptomics" with honest donor-level evaluation
- **Problem.** Predict local expression from MRI patches (T1w, T2w, DTI) plus coordinates, and test whether any **individual-level** signal exists beyond the population map.
- **Task type.** Multimodal regression; cross-modal learning.
- **Why it's ML.** Splitting predictable variance into positional and individual parts under n = 6.
- **Significance.** Very high if positive (molecular priors from routine MRI), but likely limited.
- **Novelty.** Medium. **Publication potential:** Low-medium (high risk of a negative result). **Difficulty:** High. **Timeline:** 6–9 months.

#### Idea 9 — ISH-CLIP: vision–language alignment of ISH images and gene knowledge
- **Problem.** Align cellular-resolution ISH images with gene text and protein embeddings for zero-shot retrieval ("which gene gives this laminar pattern?"). Sources: human cortex ISH, ~1,000 genes (Zeng 2012 †); mouse ISH, ~20k genes.
- **Task type.** Vision-language; contrastive; zero-shot.
- **Why it's ML.** CLIP with one "caption" per gene, heavy imbalance, and cross-species image shift.
- **Significance.** Moderate; a step toward neuropathology vision-language models.
- **Novelty.** Medium-high. **Publication potential:** Medium. **Difficulty:** Medium-high (image engineering through the API). **Timeline:** 6–9 months.

#### Idea 10 — Do pathology foundation models transfer to neuroanatomy? (+ few-shot disease ISH)
- **Problem.** Benchmark frozen pathology foundation models (UNI/UNI2, CONCH, Virchow2, H-optimus, Phikon-v2) against general vision models (DINOv2/v3) on four tasks:
  - structure classification on Allen Reference Atlas Nissl plates
  - cortical-layer classification
  - ISH pattern classification
  - few-shot schizophrenia/autism vs control on the ISH disease studies
- **Task type.** Transfer learning; few-shot; multiple-instance learning.
- **Why it's ML.** Measures the domain gap for models trained on cancer H&E, plus donor-level few-shot MIL with uncertainty.
- **Significance.** Neuropathology AI (dementia diagnosis) needs to know this. [NeuroFM (2025)](https://arxiv.org/abs/2512.05993v1) argues that general foundation models miss neurodegenerative morphology.
- **Novelty.** Medium. **Publication potential:** Medium-high (MIDL, ISBI, MICCAI workshops, *J Pathol Inform*). **Difficulty:** Low-medium. **Timeline:** 3–5 months.

#### Idea 11 — Spatially aware Bayesian cell-type deconvolution with uncertainty
- **Problem.** Estimate per-sample cell-type proportions from bulk microarray using snRNA-seq references (Siletti 2023), with a spatial GP prior, platform-shift terms and calibrated uncertainty.
- **Task type.** Probabilistic modelling; amortized inference.
- **Why it's ML.** A hierarchical model under platform shift with spatial priors.
- **Significance.** Cell-type maps are used to interpret regional vulnerability.
- **Novelty.** Medium (deconvolution is crowded, e.g., [DeTREM](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10507917/)). **Publication potential:** Medium. **Difficulty:** Medium-high, mostly because validation is hard. **Timeline:** 6–9 months.

#### Idea 12 — Cross-platform translation (microarray ↔ RNA-seq ↔ GTEx/PsychENCODE)
- **Problem.** Calibrate AHBA microarray to RNA-seq scale using the RNA-seq subset, and harmonize with GTEx brain to borrow estimates of inter-individual variance.
- **Task type.** Domain adaptation.
- **Novelty.** Low-medium. **Publication potential:** Low-medium. **Difficulty:** Low-medium. **Timeline:** 3–4 months. *A good warm-up or engineering contribution, not a flagship.*

#### Idea 13 — Learned generative null models for brain maps
- **Problem.** Existing spatial nulls (spin, BrainSMASH, Moran, [eigenstrapping](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12330862/)) carry assumptions, and volumetric or subcortical maps lack good nulls. Train generative models (score-based or GP-hybrid) to sample surrogate maps, with formal checks of false-positive-rate control.
- **Task type.** Generative modelling; statistical ML.
- **Novelty.** Medium-high. **Publication potential:** Medium. **Difficulty:** High (needs statistical theory). **Timeline:** 8–12 months.

#### Idea 14 — Heterogeneous-graph atlas completion
- **Problem.** Complete the donor × region × gene tensor: four donors lack the right hemisphere, and some structures are unsampled. Use a region graph (adjacency, connectome) and a gene graph (PPI, co-expression).
- **Task type.** Graph learning; tensor completion under structured missingness (not at random).
- **Novelty.** Medium. **Publication potential:** Medium. **Difficulty:** Medium. **Timeline:** 4–6 months.

#### Idea 15 — Causal disentanglement: cell composition vs cell-intrinsic regulation
- **Problem.** Separate composition-driven from regulation-driven spatial gradients, using snRNA-seq priors.
- **Task type.** Causal representation learning with identifiability.
- **Novelty.** High, but hard to validate and needs neuroscience expertise. **Publication potential:** Low-medium. **Difficulty:** High. **Timeline:** 9–12 months. *Not recommended for your profile now.*

#### Idea 16 — Federated, spatially aware imaging-transcriptomics
- **Problem.** Consortia like ENIGMA can't pool individual MRI. AHBA can serve as a public, shared prior. Design federated, spatially aware association and prediction methods, simulated with ABIDE sites as clients.
- **Task type.** Federated learning; differential privacy.
- **Honest note.** AHBA itself has no privacy issue; the federated story lives entirely on the clinical side.
- **Novelty.** Medium. **Publication potential:** Low-medium. **Difficulty:** Medium. **Timeline:** 5–7 months.

#### Idea 17 — Atlas-grounded RAG / tool-use QA with verifiable answers
- **Problem.** LLMs hallucinate about gene expression and anatomy. Auto-generate questions whose answers are computed from AHBA, then compare closed-book, literature-RAG and data-tool-use LLMs on accuracy, abstention and calibration.
- **Task type.** RAG; LLM evaluation.
- **Novelty.** Medium. **Publication potential:** Medium-high at workshops. **Difficulty:** Low-medium. **Timeline:** 3–4 months. *Natural sub-module of Idea 3.*

#### Idea 18 — Transcriptome → connectome link prediction, distance-debiased
- **Problem.** Correlated gene expression predicts connectivity (Richiardi 2015 †), but spatial proximity confounds it. Use GNN link prediction with distance-matched negatives and spatial CV.
- **Task type.** Graph learning.
- **Novelty.** Low-medium (well studied; the confound is known). **Publication potential:** Low-medium. **Difficulty:** Medium. **Timeline:** 4–6 months.

---

## Step 3 — High-impact ML themes: where AHBA really fits

| Theme | Fit | Best instantiation | Advisor's comment |
|---|---|---|---|
| **Self-supervised learning** | **Strong** | 1, 9 | Donors are free, natural augmentations. Masked-sample modelling over irregular sample sets is a clean pretext task. |
| **Multimodal representation learning** | **Strong** | 1 (spatial + text + protein + single-cell gene views), 8, 9 | Gene-level fusion is where multimodality is both feasible and well posed. |
| **Contrastive learning** | **Strong** | 1, 6, 9 | Positives by construction (same gene, other donor; ortholog; image ↔ text). Watch for false negatives among co-expressed genes. |
| **Graph neural networks** | Moderate | 5, 14, 18 | Brain graphs are small (≤400 nodes); a GNN has to beat MLPs and positional baselines to earn its place. |
| **Vision-language models** | Moderate | 9, 10 | Images are real (ISH, Nissl), but paired text is thin; gene descriptions are the "captions". |
| **Biological foundation models** | **Strong for evaluation/adaptation**, weak for pretraining | 1, 10 | AHBA is far too small to *pretrain* a foundation model, and ideal to **probe** one ("does Geneformer know brain spatial biology?"). See the [2026 harmonised FM benchmark](https://arxiv.org/abs/2607.17227) and the [IBM gene-property benchmark](https://arxiv.org/pdf/2412.04075). |
| **RAG for biomedical knowledge** | Moderate | 17 | Strongest when answers are *computed from data*, so hallucination becomes measurable. |
| **Agentic AI for science** | **Strong** | 3 | The atlas has well-documented pitfalls, so agent failures are measurable and consequential. |
| **Uncertainty-aware healthcare AI** | **Strong** | 2, 11, 1 (calibrated prioritization) | Sparse sampling and six donors make uncertainty a first-class question. |
| **Explainable AI** | Moderate | 5; *variant:* use AHBA disease-gene maps as weak ground truth to **evaluate XAI saliency** of MRI classifiers | Interesting, but "biological plausibility" as an XAI metric needs careful framing. |
| **Synthetic data generation** | Weak–moderate | 13 | You can't learn a donor distribution from 6 donors. Generating *null* maps is the defensible use. |
| **Federated learning** | Weak (conceptual) | 16 | Only meaningful on the clinical side; AHBA is public. |
| **Transfer across biological domains** | **Strong** | 6, 10, 12 | Mouse → human, cancer H&E → brain histology, microarray → RNA-seq. |
| **Few-shot learning** | Moderate | 1 (few positive genes, positive–unlabeled), 10 (22–50 ISH donors) | Disease gene lists are small and positive–unlabeled, a good fit for few-shot and PU methods. |
| **Multimodal transformers** | Moderate | 1, 2 | Treat tissue samples as tokens with coordinate encodings (set transformers). |

---

## Step 4 — Literature gap analysis (top opportunities)

> Difficulty of reaching publishable novelty: **Low** = a clear gap you can fill with solid engineering; **Medium** = needs a sharp angle and good baselines; **High** = crowded, or needs theory or wet lab.

### 4.1 Gene representations from spatial atlases (Idea 1)
- **Recent work.**
  - Gene embeddings from text: [GenePT (2023)](https://www.biorxiv.org/content/10.1101/2023.10.16.562533.full.pdf).
  - Cross-species single-cell models: [TranscriptFormer (CZI, 2025)](https://www.biorxiv.org/content/10.1101/2025.04.25.650731.full.pdf).
  - Benchmarks: [gene-property benchmark (IBM, 2024)](https://arxiv.org/pdf/2412.04075) found expression-based models win on *localization* tasks while text and protein models win on regulatory and genomic tasks. A [harmonised benchmark (2026)](https://arxiv.org/abs/2607.17227) found no foundation model dominates.
  - Bias: [Brechtmann et al. 2023](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10629286/) showed literature-derived embeddings look good largely because benchmarks favour well-studied genes, and they don't help on GWAS signals. A [2025 survey on LLM gene prioritization](https://arxiv.org/pdf/2501.18794) documents the same popularity bias.
  - Brain-specific: [BrainAlign 2024](https://ideas.repec.org/a/nat/natcom/v15y2024i1d10.1038_s41467-024-50608-2.html) (human–mouse SSL); [Ruffle 2024](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11267301/) (autoencoders over voxel-wise AHBA; sample-level, not gene-level, SSL); [DeepND](https://www.biorxiv.org/content/10.1101/2020.06.13.150201.full.pdf) and [forecASD](https://www.biorxiv.org/content/10.1101/370601v3.full) (developmental BrainSpan networks for autism/ID genes).
- **Already solved.** Generic gene embeddings exist. Simple AHBA-feature classifiers predict some disease genes (Negi & Guda 2017). Literature bias is documented.
- **Gaps.**
  1. No SSL that uses **multi-donor spatial views** of each gene.
  2. No test of **what spatial expression adds beyond foundation-model embeddings** under **temporal** and **understudied-gene** holdouts, which is exactly where literature-derived embeddings should fail.
  3. Disease-gene prioritization is rarely treated as **positive–unlabeled** learning with calibrated rankings.
- **Novelty difficulty: Low–Medium.** The gap is clear. The risk is empirical, not conceptual.

### 4.2 Continuous transcriptomic maps with uncertainty (Idea 2)
- **Recent work.**
  - [Wagstyl 2024 (MAGICC)](https://elifesciences.org/articles/86933): vertex-level cortical maps.
  - [Yu et al. 2025](https://arxiv.org/html/2506.11158v1): INR interpolation of the top-100 AD-risk genes.
  - [SUICA, ICML 2025](https://proceedings.mlr.press/v267/zhu25af.html): INRs for spatial transcriptomics.
  - [BayesNF, 2024](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11390735): Bayesian neural fields for spatiotemporal data.
  - [GPSA, 2023](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10482692/): deep-GP alignment across samples.
  - [Eigenstrapping, 2025](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12330862/): improved spatial nulls.
- **Already solved.** Deterministic dense maps (smoothing, kriging, INR); cortical-surface maps; better nulls for *given* maps.
- **Gaps.**
  1. **Donor-level variation** modelled explicitly (latent codes) rather than averaged away.
  2. **Calibrated uncertainty** with coverage guarantees under spatial dependence and only 6 donors (a real conformal-prediction research problem).
  3. **All-gene scalability** through low-rank fields.
  4. **Propagating map uncertainty** into downstream association tests and measuring the effect on false-positive rate. To my reading of the available abstracts, the 2025 INR work does not focus on 1, 2 or 4; read the full paper to confirm.
- **Novelty difficulty: Medium.** You must clearly beat or complement Yu 2025 and MAGICC.

### 4.3 LLM agents for imaging-transcriptomics analysis (Idea 3)
- **Recent work.**
  - [BixBench (2025)](https://arxiv.org/pdf/2503.00096v2): frontier models reached ~17% open-answer accuracy.
  - Biomni (2025, †).
  - [SpatialAgent (2025)](https://www.biorxiv.org/content/10.1101/2025.04.03.646459v1), [CellVoyager (2025)](https://neuroscience.stanford.edu/node/5300), [NeuroAgent (2026)](https://arxiv.org/pdf/2605.06584): neuroimaging preprocessing and ADNI classification.
  - Statistical reliability: [P-Bench / Fisher-R1 (2026)](https://huggingface.co/papers/2608.07437); ["LLM hacking"/p-hacking](https://arxiv.org/pdf/2606.27687).
  - BrainBench (Luo et al., *Nat Hum Behav* 2024 †): LLMs predicting neuroscience results.
- **Already solved.** Agents can run bioinformatics pipelines; general benchmarks exist; generic statistical-reliability benchmarks are emerging.
- **Gaps.**
  1. No benchmark of agents on **spatially dependent** biomedical inference, where the textbook test is wrong.
  2. No **planted-signal / null-task** design for this domain, which gives exact FDR and power.
  3. No **guardrail method** evaluated on the statistical-validity vs discovery-power trade-off.
- **Novelty difficulty: Medium.** The field moves fast, so you need a procedurally generated benchmark that stays meaningful as models change.

### 4.4 Spatial leakage in predictive brain-map ML (Idea 4)
- **Recent work.** [Rosenblatt et al. 2024](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10901797/) on connectome leakage; [Fulcher 2021](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8113439/) and [Wei 2022](https://www.biorxiv.org/content/10.1101/2021.02.22.432228.full.pdf) on inference; [Hansen 2022](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC9365855) on predicting disorder maps from molecular features (uses spatial nulls); remote-sensing spatial CV ([Kattenborn 2022](https://rsc4earth.de/publication/kattenborn-spatially-2022/)).
- **Gap.** No standard **predictive** protocol (spatially blocked CV, surrogate-target CV) with quantified inflation across model families on brain maps.
- **Novelty difficulty: Low.** It's a rigor paper, so its novelty is "important and missing" rather than methodological.

### 4.5 Molecular priors in neuroimaging disease models (Idea 5)
- **Recent work.** Brain MRI foundation models such as [BrainIAC (*Nat Neurosci* 2026)](https://www.medrxiv.org/content/10.1101/2024.12.02.24317992.full.pdf). Many GNN-on-connectome classifiers. Studies linking disease-model weights to cell-type transcriptomic features after training, not as priors.
- **Gap.** A controlled test of whether **biology** (not just smoothness or position) in AHBA-derived encodings improves sample efficiency, cross-site generalization or explanation alignment. The spin-permuted control is the key missing experiment.
- **Novelty difficulty: Medium.** The method is easy. The risk is a null result, so pre-register hypotheses.

### 4.6 Pathology foundation models → neuroanatomy and neuro-ISH (Idea 10)
- **Recent work.** UNI and CONCH (*Nat Med* 2024 †; see [MGB announcement](https://news.massgeneralbrigham.org/en/mass-general-brigham-researchers-develop-ai-foundation-models-to-advance-pathology)); multi-FM benchmarks on cancer cohorts; [NeuroFM (2025)](https://arxiv.org/abs/2512.05993v1) for neuropathology.
- **Gap.** No systematic transfer benchmark on **Nissl cytoarchitecture and ISH** (non-H&E, non-cancer), and no few-shot disease detection on the AHBA ISH disease studies.
- **Novelty difficulty: Low–Medium.** Mostly a benchmark contribution, and quick to execute.

### 4.7 Cross-species transfer (Idea 6)
- **Recent work.** [BrainAlign (2024)](https://ideas.repec.org/a/nat/natcom/v15y2024i1d10.1038_s41467-024-50608-2.html); TranscriptFormer's cross-species embeddings; [SpatialSSL (NeurIPS 2023)](https://neurips.cc/virtual/2023/75729) on mouse whole-brain spatial transcriptomics.
- **Gap.** Uncertainty about which genes or regions are conserved, and downstream translational evaluation.
- **Novelty difficulty: Medium–High.** BrainAlign covers the core idea.

---

## Step 5 — PhD-oriented ranking

Scores run 1–5 per criterion; the total is out of 30. Criteria:

- **C1** — fit for a Healthcare AI PhD application
- **C2** — likelihood of publication
- **C3** — strength of the ML contribution
- **C4** — technical depth
- **C5** — fit for an SWE moving into research
- **C6** — long-term career value

| Rank | # | Idea | C1 | C2 | C3 | C4 | C5 | C6 | **Total** |
|---|---|---|---|---|---|---|---|---|---|
| 1 | 1 | Donors-as-views SSL gene embeddings | 5 | 4 | 5 | 4 | 5 | 5 | **28** |
| 2 | 2 | Probabilistic donor-aware neural fields | 4 | 4 | 4 | 5 | 4 | 4 | **25** |
| 3 | 3 | Statistically guarded agents | 4 | 4 | 3 | 4 | 5 | 5 | **25** |
| 4 | 5 | Transcriptomic positional encodings (clinical MRI) | 5 | 3 | 4 | 4 | 4 | 4 | **24** |
| 5 | 4 | Spatial leakage benchmark | 4 | 5 | 3 | 3 | 5 | 3 | **23** |
| 6 | 10 | Pathology FM → neuroanatomy + few-shot ISH | 4 | 4 | 3 | 3 | 5 | 4 | **23** |
| 7 | 8 | MRI → transcriptome | 4 | 2 | 4 | 4 | 3 | 4 | **21** |
| 8 | 9 | ISH-CLIP | 3 | 3 | 4 | 4 | 4 | 3 | **21** |
| 9 | 14 | Graph atlas completion | 3 | 3 | 4 | 4 | 4 | 3 | **21** |
| 10 | 17 | Atlas-grounded RAG QA | 3 | 4 | 2 | 3 | 5 | 4 | **21** |
| 11 | 6 | Cross-species transfer | 3 | 3 | 4 | 4 | 3 | 3 | **20** |
| 12 | 7 | Active spatial sampling | 3 | 3 | 4 | 4 | 3 | 3 | **20** |
| 13 | 13 | Learned null models | 3 | 3 | 4 | 5 | 2 | 3 | **20** |
| 14 | 11 | Bayesian deconvolution | 3 | 3 | 4 | 4 | 2 | 3 | **19** |
| 15 | 15 | Causal disentanglement | 3 | 2 | 4 | 5 | 1 | 3 | **18** |
| 16 | 16 | Federated imaging-transcriptomics | 3 | 2 | 3 | 3 | 4 | 3 | **18** |
| 17 | 18 | Connectome link prediction | 2 | 3 | 3 | 3 | 4 | 2 | **17** |
| 18 | 12 | Cross-platform translation | 2 | 3 | 2 | 2 | 4 | 2 | **15** |

**Top three per criterion:**
- **PhD fit:** 1, 5, then 2/3/4/8/10
- **Publication likelihood:** 4, then 1/2/3/10/17
- **ML contribution:** 1, then 2/5/6/7/8/9/11/13/14/15
- **Technical depth:** 2, 13, 15
- **SWE-transition fit:** 1, 3, 4/10/17
- **Career value:** 1, 3, then 2/5/8/10/17

*Ties at 25 (ideas 2 and 3) are broken in favour of ML rigor.*

---

## Step 6 — Final recommendation: top three projects

These three fit together as one research program, which is useful for a statement of purpose: **"Trustworthy machine learning for sparse, multi-subject biomedical atlases."**

- **(A)** learns representations of *genes*.
- **(B)** models *space* with calibrated uncertainty.
- **(C)** makes *automated analysis* statistically valid.

Idea 4 (spatial-leakage protocol) is the shared evaluation backbone for all three.

### Project A — "Donors as Views": self-supervised spatial gene representations for brain-disorder gene prioritization

**Research hypotheses (pre-register these):**
- **H1 (representation).** Contrastive SSL, with positives defined as *the same gene measured in different donors*, yields gene embeddings with higher cross-donor retrieval (gene-identity top-1 accuracy) than PCA or autoencoders on raw or parcellated expression.
- **H2 (downstream).** Under **temporal** and **understudied-gene** holdouts, spatial SSL embeddings improve brain-disorder gene prioritization (AUPRC, recall@k) over raw-expression baselines. Fused with foundation-model embeddings (text, protein, single-cell), they beat the best single modality, with the largest gains on genes with few publications.
- **H3 (control).** The gains are not explained by trivial gene-level covariates: mean expression, spatial autocorrelation (Moran's I or variogram parameters), gene length, PubMed count.

**Data:**
- AHBA microarray, processed with `abagen`, in two forms:
  - per-(gene, donor) **point sets**: MNI xyz + expression value + ontology label
  - per-donor **parcellated** matrices (e.g., Schaefer-400 + subcortex)
- Optional: Allen Mouse Brain Atlas 3D ISH expression grids (ortholog views).
- Labels:
  - SFARI Gene (archived releases with dates)
  - SCHEMA (Singh 2022 †) and PGC3 prioritized genes (Trubetskoy 2022 †)
  - AD GWAS genes (Bellenguez 2022 †)
  - Open Targets Platform historical releases (for temporal splits)
  - cell-type markers from Siletti 2023
  - GO-slim
- Foundation-model gene embeddings: GenePT, Geneformer and scGPT token embeddings, TranscriptFormer, ESM-2 (protein).

**Model sketch:**
```
For gene g, donor d:  S_{g,d} = {(x_i, e_i, s_i)}   x = MNI coord (Fourier features),
                                                    e = expression, s = ontology embedding
Encoder f: Set Transformer / Point Transformer over S_{g,d}  ->  z_{g,d} in R^256
Positives: (z_{g,d}, z_{g,d'}), d != d'  (15 donor pairs per gene)
Loss:      InfoNCE (debiased; soft-positives for highly co-expressed genes)
           + masked-sample reconstruction (MAE-style auxiliary)
Augment:   random sample dropout (30-50%), region-block masking,
           coordinate jitter (~registration error), hemisphere mirroring
Gene embedding: z_g = mean_d z_{g,d}  (or attention pooling)
Fusion:    late fusion / gated cross-attention with FM embeddings; modality dropout
Downstream: linear probe, kNN, PU-learning head (nnPU) with calibration
```

**Baselines:**
- *Expression-based:* donor-averaged parcellated vector + PCA; differential-stability-weighted features; autoencoder (Ruffle-style); node2vec on the AHBA co-expression network.
- *Other modalities:* STRING node2vec; GenePT; Geneformer/scGPT; TranscriptFormer; ESM-2; Gene2vec.
- *Domain baselines:* BrainSpan-feature model (forecASD-like).
- *Covariate-only:* expression level, Moran's I, gene length, PubMed count.
- *Random embedding.*

**Proposed novel contributions:**
1. **Donor-as-view SSL** for irregularly sampled, multi-subject spatial biology. It applies to SEA-AD, BICAN and any multi-donor atlas.
2. **Leakage-aware gene-prioritization benchmark** for brain disorders: random, paralog-grouped, literature-bias and **temporal** splits, framed as positive–unlabeled. Released as a pip package with frozen splits.
3. **Modality attribution:** what spatial atlas information adds beyond text, protein and single-cell foundation models, and *for which genes*.
4. A **donor-scaling curve**: how embedding quality scales from 2 to 6 donors. This is useful for planning future atlases.

**Experimental design:**
- **Splits:**
  - S1 random stratified 5-fold (reported, not primary)
  - S2 paralog-family grouped
  - S3 literature-bias (train on the top 60% of genes by PubMed count, test on the bottom 20%)
  - S4 temporal (labels frozen at, e.g., a 2019–2020 release; positives are genes first added in 2021–2026; negatives are never-associated genes as of the latest release, with a PU caveat)
- **Diseases:** autism, schizophrenia, epilepsy, Alzheimer's, Parkinson's, depression (plus a non-brain control disease to test specificity).
- **Metrics:** AUPRC, recall@k, enrichment in the top 1%, calibration (ECE after Platt or isotonic scaling), bootstrap 95% CIs over genes, 5 seeds; paired significance by permutation.
- **Ablations:** view type (donor vs random subsample), encoder (set transformer vs MLP on parcels), loss (InfoNCE vs VICReg vs MAE-only), parcellation resolution, number of training donors, coordinate noise.
- **Sanity checks:** a covariate-only model; label permutation; spatially smooth surrogate maps substituted for real gene maps (embeddings should lose disease signal).

**Compute:** one 24 GB GPU (RTX 4090, A10G or L4). The data is tiny (~16k genes × 6 donors × ≤950 points). SSL runs take hours; ESM-2 embeddings for ~20k proteins take about 1–2 GPU-hours. Expect under 200 GPU-hours in total (about $100–300 on cloud).

**Risks and mitigations:**

| Risk | Mitigation |
|---|---|
| SSL doesn't beat PCA | Go/no-go gate at week 6. If it fails, make the benchmark and modality analysis the main contribution; that's still publishable. |
| Small, noisy positive sets (e.g., about 200–250 high-confidence SFARI genes) | PU framing; several diseases; CIs; avoid overfitting through linear probes. |
| Spatial-autocorrelation confound | Covariate baselines and surrogate-map control (H3). |
| False negatives in contrastive loss (co-expression modules) | Debiased or soft-positive InfoNCE; ablate. |
| Effort assembling temporal labels | Start in month 1. SFARI archives, Open Targets historical FTP releases and GWAS Catalog dates are all public. |
| Preprocessing sensitivity | Two `abagen` configurations; report variance. |

**6-month roadmap:**

| Month | Work | Deliverable / gate |
|---|---|---|
| 1 | `abagen` pipeline; point-set and parcellated tensors; label sets with dates; FM embeddings; covariate table | Reproducible data package; first linear-probe baseline (week 2) |
| 2 | Benchmark splits (S1–S4); all baselines; evaluation harness with CIs | Baseline leaderboard; draft benchmark section |
| 3 | SSL v1 (set transformer + InfoNCE); retrieval metrics; first downstream results | **Week-6 gate (H1);** internal tech report |
| 4 | Fusion with FMs; PU head; ablations; donor-scaling curve | Results for H2 and H3; **workshop submission** (e.g., MLGenX, LMRL, ML4H findings) |
| 5 | Robustness (preprocessing, parcellation); understudied-gene analysis; candidate-gene case studies with literature check | Full paper draft |
| 6 | Package release (pip + Hugging Face embeddings + frozen splits); arXiv; full submission | ISMB/ECCB, PSB, ML4H or a journal (*Bioinformatics*, *Briefings in Bioinformatics*); repo with DOI |

**Target venues** (check current CFPs; deadlines move):
- *ML venues:* ML4H; ICLR/ICML/NeurIPS workshops (MLGenX, LMRL, FM4Science, AIDrugX).
- *Computational biology:* ISMB/ECCB (*Bioinformatics* proceedings); PSB (strong for students); RECOMB.
- *Journals:* *Bioinformatics*, *Briefings in Bioinformatics*, *Cell Systems*, *Genome Biology*, *Nature Communications* (only if the results are strong).

---

### Project B — Probabilistic, donor-aware neural fields for sparse transcriptomic atlases

**Research hypotheses:**
- **H1.** A donor-conditioned, low-rank neural field reduces leave-one-donor-out error (RMSE / Pearson per gene) over nearest-sample, Gaussian smoothing, kriging/GP and deterministic INR baselines.
- **H2.** Its predictive intervals are **calibrated**: empirical coverage within ±3 percentage points of nominal at 80% and 90%, under both leave-one-donor-out and spatially blocked holdouts.
- **H3.** Propagating map uncertainty into imaging-transcriptomic association tests (multiple imputation plus spatial nulls) **reduces false-positive rate** on null maps at comparable power on planted signals.

**Model:**
`E(x, g, d) = Σ_k w_{g,k} · φ_k(x; z_d)`
- φ: a shared neural-field basis (hash-grid or Fourier-feature MLP).
- z_d: donor latent (auto-decoder, DeepSDF-style, FiLM conditioning).
- w_g: gene loadings, which make it scale to all ~16k genes.
- Heteroscedastic Student-t likelihood.
- Uncertainty from a deep ensemble or Laplace approximation, compared against BayesNF.
- Conformal calibration with donors and spatial blocks as calibration units (spatially weighted/localized conformal). This is a real research question, because six exchangeable units give coarse quantiles.
- **Few-shot donor adaptation:** given k anchor samples from a new donor, optimize z_d. This links directly to Idea 7 (active sampling).

**Baselines:** `abagen` nearest-sample assignment; Gaussian smoothing / MAGICC; variogram kriging (Gryglewski 2018); sparse GP (GPyTorch); GPSA; INR (Yu et al. 2025); SUICA; BayesNF; low-rank matrix completion.

**Novel contributions:**
1. Donor-latent, multi-output neural fields for multi-subject sparse biology.
2. Calibrated (conformal) uncertainty under spatial dependence with few subjects.
3. End-to-end propagation of uncertainty into downstream inference, with measured false-positive-rate effects.
4. Released all-gene posterior maps (mean + variance) in neuromaps-compatible format. A practical community resource is a strong PhD signal.

**Experimental design:**
- **Tasks:**
  - T1: within-donor spatially blocked holdout (whole structures or k-means spatial blocks).
  - T2: leave-one-donor-out, with 0, 5, 10 or 50 anchor samples.
  - T3: external weak validation — correspondence of receptor and transporter **genes** with **PET** maps (e.g., HTR1A ↔ 5-HT1A, SLC6A4 ↔ 5-HTT, DRD2 ↔ D2, SLC6A3 ↔ DAT, OPRM1 ↔ MOR, CNR1 ↔ CB1; Hansen 2022 *Nat Neurosci* †; maps in neuromaps), especially subcortically, where interpolation matters most.
  - T4: downstream inference on ENIGMA case–control maps, with FPR on null maps (eigenstrapping/BrainSMASH surrogates) and power on planted associations.
- **Metrics:** RMSE, Pearson, CRPS, NLL, interval coverage and width, FPR and power.

**Compute:** one 24 GB GPU. Hash-grid fields train in minutes per configuration; ensembles over all genes come from the low-rank design. Under 300 GPU-hours in total.

**Risks:**

| Risk | Mitigation |
|---|---|
| Overlap with Yu et al. 2025 | Differentiate on donor latents, calibration, all-gene scaling and downstream inference. Contact the authors early; it may lead to a collaboration. |
| Registration error dominates | Model coordinate noise explicitly as an input-noise augmentation. |
| No dense ground truth | Rely on held-out donors, spatial blocks and PET correspondence. Be explicit about this in the paper. |
| Conformal with 6 units is coarse | Make it a stated research contribution (spatial-block calibration) rather than a hidden weakness. |

**6-month roadmap:**

| Month | Work | Gate |
|---|---|---|
| 1 | Data in native and MNI space; tissue masks from donor MRI; baselines (nearest, smoothing, kriging, GP) | Baseline table for T1/T2 |
| 2 | Neural field v1 (single gene); then low-rank multi-gene; donor latents | Beats GP on T2 for ≥60% of genes |
| 3 | Uncertainty (ensembles / Laplace / BayesNF); conformal calibration; coverage study | Coverage within ±3 points |
| 4 | Few-shot donor adaptation; PET correspondence (T3) | **MIDL / ISBI-style short paper** |
| 5 | Downstream inference (T4): multiple imputation + spatial nulls; FPR/power | Main result for H3 |
| 6 | Release posterior maps + code; full paper | MICCAI, IPMI or *NeuroImage* / *Imaging Neuroscience* / *MedIA* |

**Target venues:** MICCAI, IPMI, MIDL, ISBI; *Medical Image Analysis*, *IEEE TMI*, *NeuroImage*, *Imaging Neuroscience*; AISTATS if the conformal component becomes strong.

---

### Project C — Statistically guarded LLM agents for imaging transcriptomics

**Research hypotheses:**
- **H1.** Without guardrails, frontier LLM agents doing open-ended AHBA imaging-transcriptomics analyses report "significant" findings on **≥20% of null tasks**, where no true association exists by construction. The main causes are ignored spatial autocorrelation, naive GCEA and donor leakage.
- **H2.** A guardrail layer cuts the false-discovery rate by **≥50%** while losing **≤10%** power on planted-signal tasks. The layer has two parts: typed tools that default to spatial-null tests, and a statistical-critic agent that audits code and claims.
- **H3.** Agents' stated confidence is poorly calibrated to correctness without guardrails and better calibrated with them.

**Benchmark design ("AtlasBench"), ~300 tasks:**
- **F1** data retrieval and processing (≈60): "rank these genes by putamen expression across donors", with exact answers.
- **F2** planted-signal association (≈80): synthetic target maps with known gene associations and controlled spatial autocorrelation.
- **F3** null tasks (≈80): surrogate maps with *no* true association. These measure FDR exactly.
- **F4** enrichment tasks (≈40): planted vs null gene-category effects (the Fulcher-style trap).
- **F5** replication of published findings (≈20–40), with rubric grading and a small expert review.

All tasks are **procedurally generated from seeds**, so the benchmark is refreshable and resists contamination.

**Conditions:** {closed-book, generic Python, domain tools, domain tools + guardrails} × {2–3 frontier API models + 2 open-weight models} × 3 seeds.

**Metrics:** accuracy; FDR on F3; power on F2; a validity rubric (spatial null used? multiple comparisons handled? donors handled?); calibration of stated confidence; cost and latency.

**Baselines:** ReAct / code-execution agents; Biomni-style general agent; a "statistician system prompt" baseline (prompting only, no tools); the human-expert pipeline (`abagen` + spin/eigenstrapping + Fulcher-style ensemble nulls).

**Novel contributions:**
1. A first benchmark for agents on **spatially dependent biomedical inference**, with exact FDR and power by construction.
2. A **guardrail method** with a measured validity vs discovery trade-off.
3. Released sandbox, tool APIs and generator.

**Compute and cost:** CPU sandbox (Docker) plus API calls. Roughly 300 tasks × 4–5 models × 3 seeds × ~$0.3–1 per run, about **$1.5–4k**. Shrink it with a 100-task core set, or apply for research API credits if a program is available. Open-weight models need one or two GPUs or a hosted endpoint.

**Risks:**

| Risk | Mitigation |
|---|---|
| Model churn makes results stale | The generator lets you re-run on new models; the guardrail result is the durable part. |
| Grading open-ended answers | Most tasks have numeric or set answers; use LLM-judge only for F5, with human audit. |
| Overlap with general benchmarks (P-Bench, BixBench) | Position this as domain-specific dependence-aware inference; cite and contrast. |
| API cost | Core subset; caching; early stopping. |

**6-month roadmap:**

| Month | Work | Gate |
|---|---|---|
| 1 | Sandbox + tool API (`abagen`, neuromaps, nulls); task generator for F1–F3 | 50 validated tasks |
| 2 | F4–F5; grading harness; pilot with one model | Pilot shows a measurable FDR on nulls |
| 3 | Full runs: all models × conditions (without guardrails) | H1 result |
| 4 | Guardrail layer (typed tools + critic); re-runs | H2/H3 result; **workshop paper** |
| 5 | Error taxonomy; calibration analysis; expert review of F5 | Paper draft |
| 6 | Public release (leaderboard, generator); full submission | NeurIPS Datasets & Benchmarks or ML4H; *Bioinformatics* / *Nat Comput Sci* if strong |

---

### Strong alternative if you're targeting clinical-imaging labs: Idea 5 (transcriptomic positional encodings)

If your target PhD labs are mainly in medical imaging or clinical neuroimaging, Idea 5 is the most *clinically framed*. It trains disease classifiers on ABIDE and ADNI with AHBA-derived encodings, and uses the spin-permuted-biology control. It can also reuse Project A's region embeddings: Project A learns *gene* embeddings, and the transposed view gives *region* embeddings. So it works well as a second paper after A.

---

## The single highest-leverage project: Project A ("Donors as Views")

If only one project is possible, this is it.

| Criterion | Why Project A wins |
|---|---|
| **Publishability** | Several publishable units, each standing alone: (1) a leakage-aware brain-disorder gene-prioritization benchmark; (2) a new SSL objective; (3) a modality-attribution analysis of foundation models. A negative result on (2) still leaves (1) and (3) publishable. Natural ladder: workshop paper by month 4, full paper by month 6–7. |
| **ML rigor** | Clear hypotheses (H1–H3); strong baselines across five modalities; temporal, literature-bias and paralog splits; positive–unlabeled framing; covariate and surrogate controls; CIs and seeds. This is what PhD committees read as "thinks like a researcher". |
| **Healthcare relevance** | Gene prioritization feeds target discovery, and genetic support raises clinical success about 2.6× ([Minikel 2024](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11096124/)), in CNS, where drugs fail most. You can state the translational story in one sentence without overclaiming. |
| **PhD application value** | It sits where three hiring areas meet: foundation models and representation learning, biomedical ML, and healthcare AI. It opens a research agenda ("SSL for multi-subject spatial biology") that extends to SEA-AD (84 AD donors with a continuous pathology score; [Gabitto 2024](https://pmc.ncbi.nlm.nih.gov/articles/PMC11614693)), BICAN and spatial transcriptomics. That's the kind of continuation a PhD statement of purpose should show. |
| **Feasibility (solo, 6–12 months)** | Avoids n = 6 by using about 16k genes as samples. All data and labels are public with no access applications. One consumer GPU. Neuroscience knowledge required is close to zero, since labels come from curated databases and `abagen` handles anatomy. Your engineering strengths (pipelines, packaging, reproducibility) become a visible contribution: a pip-installable benchmark. |

**Why not the others as the only project:**
- **B** is more incremental relative to Yu 2025 and MAGICC, and needs more spatial-statistics expertise to make the conformal part convincing.
- **C** is trendy, but its ML-methods contribution is thinner, results age fast as models change, and API costs are real.

**The 12-month plan:** A (months 1–6), then B or Idea 5 (months 6–12), reusing A's pipeline. Together they form a coherent first-author portfolio.

### Your first two weeks (concrete)

```python
# pip install abagen nilearn neuromaps  (check current docs for exact signatures)
import abagen

# 1) Sample-level data: expression (samples x genes) + MNI coordinates, all donors
expr, coords = abagen.get_samples_in_mask(mask=None)

# 2) Parcellated, per-donor matrices (regions x genes), keeping donors separate
atlas = abagen.fetch_desikan_killiany()
per_donor = abagen.get_expression_data(atlas['image'], atlas['info'], return_donors=True)
```

1. **Days 1–3:** run the above, then build (gene, donor) point sets and parcellated tensors, and cache them as Parquet or Zarr.
2. **Days 4–6:** download SFARI (current release plus one 2019/2020 archive) and Open Targets (one historical and one current release), plus NCBI `gene2pubmed` for literature counts.
3. **Days 7–10:** first baseline — logistic regression on donor-averaged 400-region vectors (PCA 64) predicting SFARI score 1–2 genes, under a random split and an understudied-gene split. **This early number tells you whether a signal exists.**
4. **Days 11–14:** add GenePT and ESM-2 embeddings as competing baselines, and write a 2-page internal memo of the results. This memo becomes the seed of your paper's Table 1.

---

## Companion datasets (to add clinical labels or scale)

| Dataset | What it adds | Access |
|---|---|---|
| ENIGMA Toolbox (Larivière 2021 †) | Case–control cortical and subcortical effect maps for many disorders; preprocessed AHBA | Open (pip) |
| neuromaps (Markello 2022 †) | PET receptor, metabolic and functional maps in standard spaces | Open (pip) |
| [Siletti 2023](https://www.pubmed.ncbi.nlm.nih.gov/37824663/) / [HCA Brain v1.0](https://data.humancellatlas.org/hca-bio-networks/nervous-system/atlases/brain-v1-0) | 3M+ nuclei, ~100 dissections: cell-type references and markers | Open (CELLxGENE) |
| [SEA-AD](https://pmc.ncbi.nlm.nih.gov/articles/PMC11614693) | 84 donors across the AD spectrum; snRNA/ATAC-seq (~8M nuclei); continuous pathology score | Open |
| [Aging, Dementia & TBI](https://aging.brain-map.org/overview/home) | RNA-seq of 377 samples from 107 donors (dementia/TBI labels), 4 regions | Open |
| Allen Mouse Brain Atlas (Lein 2007 †) | ~20k-gene 3D ISH: cross-species views | Open (API) |
| GTEx / BrainSpan / PsychENCODE | Inter-individual and developmental variation | Open / controlled for genotypes |
| ABIDE I/II | ~2k subjects, autism vs control, multi-site MRI | Open after registration |
| ADNI / OASIS-3 | AD MRI/PET cohorts | Data-use application |
| SFARI Gene, Open Targets, GWAS Catalog | Versioned disease–gene labels (temporal splits) | Open |

---

## Honest advisor notes

1. **Don't promise biological discoveries.** Frame contributions as methods, benchmarks and resources. Any candidate genes are *hypotheses* you check against the literature, not findings.
2. **Don't claim patient-level molecular inference from AHBA.** Six post-mortem neurotypical brains provide population priors only.
3. **Treat preprocessing as part of the experiment.** Reviewers in this area know the Arnatkevičiūtė/`abagen` literature and will check.
4. **Spatial autocorrelation is the default reviewer objection.** Build spatial nulls and blocked CV in from day one; it's also your best rigor signal.
5. **For the PhD application,** the deliverables that matter most are: one first-author arXiv paper plus a workshop acceptance, a clean open-source package, and a research statement that describes a *program* (A → B/5 → SEA-AD), not a one-off project.

---

## References

Links are to sources found during this review. Entries marked † are standard references cited from memory; verify them before citing.

**Dataset and resources**
- Hawrylycz MJ et al. An anatomically comprehensive atlas of the adult human brain transcriptome. *Nature* 2012. †
- Hawrylycz M et al. Canonical genetic signatures of the adult human brain. *Nat Neurosci* 2015. †
- Zeng H et al. Large-scale cellular-resolution gene profiling in human neocortex. *Cell* 2012. †
- Ding S-L et al. Comprehensive cellular-resolution atlas of the adult human brain. *J Comp Neurol* 2016. [Allen Human Reference Atlas 3D README](https://download.alleninstitute.org/informatics-archive/allen_human_reference_atlas_3d_2020/version_1/README.pdf)
- Allen Human Brain Atlas ISH studies (Cortex, Subcortex, Neurotransmitter, Schizophrenia, Autism). [Allen community documentation](https://community.brain-map.org/t/human-brain-atlas-in-situ-hybridization-ish-data/2872)
- Lein ES et al. Genome-wide atlas of gene expression in the adult mouse brain. *Nature* 2007. †
- Siletti K et al. Transcriptomic diversity of cell types across the adult human brain. *Science* 2023. [PubMed](https://www.pubmed.ncbi.nlm.nih.gov/37824663/)
- Gabitto MI, Travaglini KJ et al. Integrated multimodal cell atlas of Alzheimer's disease (SEA-AD). *Nat Neurosci* 2024. [PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC11614693)
- Allen Aging, Dementia and TBI Study. [Overview](https://aging.brain-map.org/overview/home)

**Imaging transcriptomics, methods and pitfalls**
- Arnatkevičiūtė A, Fulcher BD, Fornito A. A practical guide to linking brain-wide gene expression and neuroimaging data. *NeuroImage* 2019. †
- Markello RD et al. Standardizing workflows in imaging transcriptomics with the abagen toolbox. *eLife* 2021. †
- Markello RD et al. neuromaps. *Nat Methods* 2022. †
- Larivière S et al. The ENIGMA Toolbox. *Nat Methods* 2021. †
- Fulcher BD, Arnatkevičiūtė A, Fornito A. Overcoming false-positive gene-category enrichment… *Nat Commun* 2021. [PMC](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC8113439/)
- Wei Y et al. Statistical testing in transcriptomic-neuroimaging studies… (2021 preprint; *Hum Brain Mapp* 2022). [bioRxiv](https://www.biorxiv.org/content/10.1101/2021.02.22.432228.full.pdf)
- Koussis NC et al. Generation of surrogate brain maps preserving spatial autocorrelation through random rotation of geometric eigenmodes. *Imaging Neuroscience* 2025. [PMC](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12330862/)
- Burt JB et al. Hierarchy of transcriptomic specialization across human cortex… *Nat Neurosci* 2018. †
- Burt JB et al. Generative modeling of brain maps with spatial autocorrelation (BrainSMASH). *NeuroImage* 2020. †
- Alexander-Bloch AF et al. On testing for spatial correspondence between maps… (spin test). *NeuroImage* 2018. †
- Gryglewski G et al. Spatial analysis and high resolution mapping of the human whole-brain transcriptome… *NeuroImage* 2018.
- Dear R et al. Cortical gene expression architecture links healthy neurodevelopment to the imaging, transcriptomics and genetics of autism and schizophrenia. *Nat Neurosci* 2024. [Springer](https://link.springer.com/10.1038/s41593-024-01624-4)
- Wagstyl K et al. Transcriptional cartography integrates multiscale biology of the human cortex. *eLife* 2024. [eLife](https://elifesciences.org/articles/86933)
- Hansen JY et al. Local molecular and global connectomic contributions to cross-disorder cortical abnormalities. *Nat Commun* 2022. [PMC](https://pmc.ncbi.nlm.nih.gov/articles/PMC9365855)
- Hansen JY et al. Mapping neurotransmitter systems to the structural and functional organization of the human neocortex. *Nat Neurosci* 2022. †
- Seidlitz J et al. Transcriptomic and cellular decoding of regional brain vulnerability to neurogenetic disorders. *Nat Commun* 2020. †
- Richiardi J et al. Correlated gene expression supports synchronous activity in brain networks. *Science* 2015. †
- Rosenblatt M et al. Data leakage inflates prediction performance in connectome-based machine learning models. *Nat Commun* 2024. [PMC](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10901797/)
- Kattenborn T et al. Spatially autocorrelated training and validation samples inflate performance assessment of CNNs. 2022. [Link](https://rsc4earth.de/publication/kattenborn-spatially-2022/)

**ML on brain atlases and spatial fields**
- Ruffle JK et al. Compressed representation of brain genetic transcription. *Hum Brain Mapp* 2024. [PMC](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11267301/)
- Yu X, Torok J, Pandya S, Pal S, Singh V, Raj A. Brain-wide interpolation and conditioning of gene expression in the human brain using Implicit Neural Representations. arXiv:2506.11158, 2025. [arXiv](https://arxiv.org/html/2506.11158v1)
- Zhu et al. SUICA: Learning super-high dimensional sparse implicit neural representations for spatial transcriptomics. *ICML* 2025. [PMLR](https://proceedings.mlr.press/v267/zhu25af.html)
- Saad F et al. Scalable spatiotemporal prediction with Bayesian neural fields. *Nat Commun* 2024. [PMC](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11390735)
- Jones A, Townes FW, Li D, Engelhardt BE. Alignment of spatial genomics data using deep Gaussian processes (GPSA). *Nat Methods* 2023. [PMC](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10482692/)
- BrainAlign: self-supervised whole-brain alignment of spatial transcriptomics between humans and mice. *Nat Commun* 2024. [RePEc](https://ideas.repec.org/a/nat/natcom/v15y2024i1d10.1038_s41467-024-50608-2.html)
- SpatialSSL: Whole-brain spatial transcriptomics in the mouse brain with self-supervised learning. NeurIPS 2023. [NeurIPS](https://neurips.cc/virtual/2023/75729)

**Gene representations, foundation models, prioritization**
- Chen Y, Zou J. GenePT. bioRxiv 2023. [bioRxiv](https://www.biorxiv.org/content/10.1101/2023.10.16.562533.full.pdf)
- TranscriptFormer: a cross-species generative cell atlas across 1.5 billion years of evolution. bioRxiv 2025. [bioRxiv](https://www.biorxiv.org/content/10.1101/2025.04.25.650731.full.pdf)
- Does your model understand genes? A benchmark of gene properties for biological and text models. arXiv:2412.04075, 2024. [arXiv](https://arxiv.org/pdf/2412.04075)
- Harmonised benchmarking of foundation models for single-cell and spatial transcriptomics… arXiv:2607.17227, 2026. [arXiv](https://arxiv.org/abs/2607.17227)
- Brechtmann F et al. Evaluation of input data modality choices on functional gene embeddings. *NAR Genom Bioinform* 2023. [PMC](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10629286/)
- Survey and improvement strategies for gene prioritization with large language models. arXiv:2501.18794, 2025. [arXiv](https://arxiv.org/pdf/2501.18794)
- Negi SK, Guda C. Global gene expression profiling of healthy human brain and its application in studying neurological disorders. *Sci Rep* 2017. [PMC](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC5429860/)
- Brueggeman L et al. forecASD. *Genome Med* 2020. [bioRxiv](https://www.biorxiv.org/content/10.1101/370601v3.full)
- Beyreli I et al. DeepND: deep multitask learning of gene risk for comorbid neurodevelopmental disorders. 2022. [bioRxiv](https://www.biorxiv.org/content/10.1101/2020.06.13.150201.full.pdf)
- Minikel EV et al. Refining the impact of genetic evidence on clinical success. *Nature* 2024. [PMC](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC11096124/)
- Singh T et al. (SCHEMA) *Nature* 2022 †; Trubetskoy V et al. (PGC3 SCZ) *Nature* 2022 †; Bellenguez C et al. (AD GWAS) *Nat Genet* 2022 †.

**Agents, LLMs, and reliability**
- BixBench: a comprehensive benchmark for LLM-based agents in computational biology. arXiv:2503.00096, 2025. [arXiv](https://arxiv.org/pdf/2503.00096v2)
- Huang K et al. Biomni: a general-purpose biomedical AI agent. bioRxiv 2025. †
- SpatialAgent: an autonomous AI agent for spatial biology. bioRxiv 2025. [bioRxiv](https://www.biorxiv.org/content/10.1101/2025.04.03.646459v1)
- CellVoyager. 2025. [Stanford news](https://neuroscience.stanford.edu/node/5300)
- NeuroAgent: LLM agents for multimodal neuroimaging analysis and research. arXiv:2605.06584, 2026. [arXiv](https://arxiv.org/pdf/2605.06584)
- Training LLM agents for reliable hypothesis testing (P-Bench, Fisher-R1). arXiv:2608.07437, 2026. [HF Papers](https://huggingface.co/papers/2608.07437)
- Mitigating LLM-based p-hacking by preregistering for the next LLM. arXiv:2606.27687, 2026. [arXiv](https://arxiv.org/pdf/2606.27687)
- Luo X et al. Large language models surpass human experts in predicting neuroscience results (BrainBench). *Nat Hum Behav* 2024. †

**Imaging and pathology foundation models**
- BrainIAC: a foundation model for generalized brain MRI analysis. *Nat Neurosci* 2026 (medRxiv 2024). [medRxiv](https://www.medrxiv.org/content/10.1101/2024.12.02.24317992.full.pdf)
- Chen RJ et al. UNI. *Nat Med* 2024 †; Lu MY et al. CONCH. *Nat Med* 2024 †. [MGB announcement](https://news.massgeneralbrigham.org/en/mass-general-brigham-researchers-develop-ai-foundation-models-to-advance-pathology)
- NeuroFM: neuropathology foundation model. arXiv:2512.05993, 2025. [arXiv](https://arxiv.org/abs/2512.05993v1)
- DeTREM: bulk brain tissue deconvolution with snRNA-seq bias correction. *BMC Bioinformatics* 2023. [PMC](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC10507917/)
