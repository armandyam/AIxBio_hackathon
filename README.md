# Accurate but Brittle: Single-Nucleotide Adversarial Attacks on Genomic Foundation Models

Submission to the [Apart Research AIxBio Hackathon](https://apartresearch.com/sprints/aixbio-hackathon-2026-04-24-to-2026-04-26) (2026-04-24 to 2026-04-26).

Paper: [`AIxBio_paper.pdf`](AIxBio_paper.pdf)

## TL;DR

Genomic foundation models (GFMs) are state-of-the-art on sequence classification but are rarely co-evaluated on adversarial robustness, compute cost, and biological plausibility. Under random single-nucleotide substitutions — a conservative, biologically-grounded attack — we observe a sizeable accuracy–robustness gap:

- **DNABERT-2:** 90.0% accuracy at **7.78% conditional ASR** (95% Wilson CI 5.7–10.5%)
- **XGBoost on 5-d compositional features:** 82.8% accuracy at **1.01% ASR** (CI 0.4–2.3%)

That is roughly an **order-of-magnitude reduction in ASR for a 7.2-point accuracy cost**, and DNABERT-2 takes ~24 min on GPU vs ~4–6 min on CPU for XGBoost on 500 sequences. We also show that gradient-based NLP attacks from prior genomic benchmarks (BertAttack, TextFooler, FIMBA via GenoArmory) produce biologically implausible sequences (length drift, GC collapse, non-ACGT characters) that likely inflate reported vulnerability.

Random SNV is among the weakest attack regimes, so all reported ASRs should be read as **lower bounds**.

## Research questions

- **RQ1:** How should adversarial robustness be evaluated for genomic sequences?
- **RQ2:** Do GFMs justify their complexity when considering both accuracy and robustness?
- **RQ3:** Are gradient-based attacks from NLP/CV appropriate for genomics?

## Contributions

1. A biologically-grounded evaluation protocol for adversarial robustness in genomics.
2. A side-by-side GFM vs. classical ML comparison on robustness rather than accuracy alone.
3. A decision-boundary-based working hypothesis for the observed accuracy–robustness gap.
4. A quantitative biological-plausibility audit of an existing genomic attack benchmark (GenoArmory).
5. Deployment-context-aware model-selection guidance.

## Setup

- **Task:** binary promoter classification on 300 bp sequences.
- **Dataset:** GUE promoter benchmark (Umarov & Solovyev) — 47,356 train / 11,839 test / 500 attack sequences.
- **Attack:** up to 1,000 random single-nucleotide substitutions per correctly-classified sequence (gradient-free, model-agnostic).
- **Metrics:** accuracy, conditional ASR with 95% Wilson CIs, training and inference wall-clock on each model's native hardware, plus 12 biological-plausibility metrics (length, GC content, per-base frequencies, Shannon entropy, CpG O/E, TATA/GC/CCAAT/Initiator motifs, homopolymer fraction, non-ACGT count).

## Headline results

| Model            | Features        | Acc (%) | ASR % [95% CI]   | Train     | Inference (500 seq) |
|------------------|-----------------|---------|------------------|-----------|---------------------|
| DNABERT-2        | Learned (768-d) | 90.0    | 7.78 [5.7, 10.5] | Hours GPU | 24 min (GPU)        |
| NTv2-250M        | Learned (768-d) | 92.4    | 24.9 [21.3, 29.0]| Hours GPU | 130 min (GPU)       |
| CatBoost k-mer   | 4-mer (256-d)   | 86.2    | 9.73 [7.4, 12.6] | Min CPU   | 16.7 min (CPU)      |
| LinearSVC k-mer  | 4-mer (256-d)   | 85.7    | 5.53 [3.9, 7.9]  | Min CPU   | 8.8 min (CPU)       |
| XGBoost k-mer    | 4-mer (256-d)   | 85.8    | 5.35 [3.7, 7.7]  | Min CPU   | 5 min (CPU)         |
| RF k-mer         | 4-mer (256-d)   | 85.4    | 8.31 [6.2, 11.0] | Min CPU   | 3.1 hrs (CPU)       |
| SVC comp         | GC%, bases (5-d)| 82.8    | 0.54 [0.2, 1.6]  | Min CPU   | 20.6 min (CPU)      |
| XGBoost comp     | GC%, bases (5-d)| 82.8    | 1.01 [0.4, 2.3]  | Min CPU   | 4 min (CPU)         |
| RF comp          | GC%, bases (5-d)| 80.5    | 22.6 [19.1, 26.5]| Min CPU   | 3.0 hrs (CPU)       |

No evaluated model simultaneously satisfies the illustrative 95%-accuracy / 5%-ASR reference frame drawn from medical-diagnostic and biometric practice — the high-accuracy / low-ASR quadrant is empty.

## Decision-boundary working hypothesis

GFMs learn smooth, locally-approximable decision boundaries in a high-dimensional embedding space — useful for accuracy, but exposing larger perturbation surfaces. Compositional classifiers operate in low-dimensional spaces (GC%, per-base frequencies move by at most ±0.3% under a single substitution) where single-nucleotide edits rarely cross thresholds. We treat this as a hypothesis, not a proven mechanism; falsifying it would require local Lipschitz / input-Jacobian estimates, embedding-space displacement distributions, and margin-distribution comparisons which we leave to follow-up work.

## Biological plausibility audit (GenoArmory)

Analyzing 250 GenoArmory adversarial sequences against the GUE original and our SNV-perturbed set:

| Metric         | Original     | GenoArmory   | Our SNV       |
|----------------|--------------|--------------|---------------|
| Length (bp)    | 300.0 ± 0.0  | 306.0 ± 43.8 | 300.0 ± 0.0   |
| GC content     | 0.612 ± 0.121| 0.543 ± 0.078| 0.629 ± 0.105 |
| Shannon 2-mer  | 3.732        | 3.887        | 3.730         |
| Non-ACGT       | 0.000        | 0.002        | 0.000         |
| CpG O/E        | 0.640        | 0.791        | 0.685         |

GenoArmory's BPE-token-level attacks systematically violate fixed-length gene structure (~43× std-dev increase), shift mean GC content 11 points toward the 50% random-DNA baseline, and introduce non-ACGT characters. Krishnan et al.'s biologically-constrained genetic algorithm reports 30% ASR (75/250) on this setting versus 66.71% reported for GenoArmory — the latter's higher numbers partly reflect biologically implausible sequences rather than true model vulnerability.

## Recommendations by use case

| Application            | Priority         | Model           | Rationale                          |
|------------------------|------------------|-----------------|------------------------------------|
| Biosecurity screening  | Robustness       | XGBoost comp    | 1.01% ASR, scalable                |
| Research / discovery   | Accuracy         | XGBoost k-mer   | 85.8% acc, 5 min inference         |
| Clinical diagnosis     | Accuracy + Robust| Adversarial GFM | 90% needed, training helps         |
| Embedded devices       | Efficiency       | XGBoost comp    | CPU-only, 4 min, robust            |

## Infrastructure

- **Hardware:** single NVIDIA A10 (24 GB) for DNABERT-2 / NTv2 fine-tuning and inference; 32-core AMD EPYC CPU node with 128 GB RAM for the classical baselines and the attack loop on classical models.
- **Software:** Python 3.10, PyTorch 2.0, HuggingFace Transformers 4.36, scikit-learn 1.3, XGBoost 1.7, CatBoost 1.2, NumPy 1.26.
- **Seed:** fixed seed 42; single-seed results (no multi-seed bootstrapping — reflected in the Wilson CIs).

## Limitations

Single task (promoter classification on GUE); random SNV is a weak attack regime so all ASRs are lower bounds; the 500-sequence attack budget yields wide Wilson CIs at low ASR (e.g. 0.4–2.3% for XGBoost compositional); we do not apply adversarial training to the classical baselines; transfer-learning regimes are not evaluated; the decision-boundary reading is a working hypothesis, not a tested mechanism.
