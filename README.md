# healthcare-research

Research planning for machine-learning projects in healthcare AI.

## Contents

| Document | Summary |
|---|---|
| [`docs/mc-med-q1-q2-q9-deep-dive.md`](docs/mc-med-q1-q2-q9-deep-dive.md) | Deep dive on the chosen questions. **Q2:** a taxonomy of care-process leakage, plus experiments and hypotheses. **Q1:** privileged-waveform distillation, including selection-aware distillation for clinician-chosen monitoring. **Q9:** pulse-oximetry bias using ED blood gases and the bedside pleth. Covers prior work, study designs, decision gates and a combined timeline. |
| [`docs/mc-med-open-research-questions.md`](docs/mc-med-open-research-questions.md) | Open research questions on **MC-MED** (Stanford ED waveforms + EHR), checked against prior work (MC-BEC, CAREBench, MDS-ED). Twelve questions are ranked. The recommended flagship is privileged-waveform distillation with a care-process leakage audit. Also includes a first-week feasibility checklist, a revised 90-day plan, and open questions for the **MedAgentBench** side track. |
| [`docs/open-datasets-for-healthcare-ml-papers.md`](docs/open-datasets-for-healthcare-ml-papers.md) | Survey of open healthcare/biomedical datasets that are easier to publish ML papers on than the Allen Human Brain Atlas. It covers access tiers and a scorecard of ~35 datasets across EHR/ED, agent benchmarks, imaging, ECG/sleep, spatial omics and public health. It ends with the top 5 for this profile and a 90-day plan; **MC-MED** is the primary pick and **MedAgentBench** the quick parallel win. |
| [`docs/ahba-ml-research-report.md`](docs/ahba-ml-research-report.md) | Advisor-style assessment of the **Allen Human Brain Atlas** as a foundation for ML research. It covers: a dataset assessment; 18 ML research directions; a map of high-impact ML themes; 2023–2026 literature gaps; a PhD-oriented ranking; and detailed plans (hypotheses, baselines, experiments, compute, risks, 6-month roadmaps) for the top three projects. |

### Recommended project (short version)

**"Donors as Views": self-supervised spatial gene representations for brain-disorder gene prioritization.**

The atlas has only six donors, so the project treats each of its ~16k genes as a sample instead. Each donor's irregularly sampled expression map of a gene is a natural "view" for contrastive learning. The resulting embeddings are tested against text, protein and single-cell foundation-model embeddings, using temporal and understudied-gene holdouts. The full rationale and roadmap are in the report.
