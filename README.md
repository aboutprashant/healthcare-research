# healthcare-research

Research planning for machine-learning projects in healthcare AI.

## Contents

| Document | Summary |
|---|---|
| [`docs/ahba-ml-research-report.md`](docs/ahba-ml-research-report.md) | Advisor-style assessment of the **Allen Human Brain Atlas** as a foundation for ML research. It covers: a dataset assessment; 18 ML research directions; a map of high-impact ML themes; 2023–2026 literature gaps; a PhD-oriented ranking; and detailed plans (hypotheses, baselines, experiments, compute, risks, 6-month roadmaps) for the top three projects. |

### Recommended project (short version)

**"Donors as Views": self-supervised spatial gene representations for brain-disorder gene prioritization.**

The atlas has only six donors, so the project treats each of its ~16k genes as a sample instead. Each donor's irregularly sampled expression map of a gene is a natural "view" for contrastive learning. The resulting embeddings are tested against text, protein and single-cell foundation-model embeddings, using temporal and understudied-gene holdouts. The full rationale and roadmap are in the report.
