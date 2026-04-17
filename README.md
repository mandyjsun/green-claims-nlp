# green-claims-nlp

Automated detection of greenwashing-risk language in corporate sustainability reports using NLP. A dual-track classifier — semantic kNN and lexical TF-IDF — is trained on manually annotated passages from 16 fashion and cosmetics brands to distinguish substantiated sustainability claims from vague, unverifiable ones.

> **Status:** DSAN 5550 course project, Georgetown University, Spring 2026.

---

## The Problem

Fashion and cosmetics are two of the most aggressively marketed "sustainable" industries on the planet. The fashion industry alone accounts for 2–8% of global carbon emissions, yet apparel sector emissions grew 7.5% in 2023 even as brands ramped up green messaging. A 2021 study found 42% of all green claims were exaggerated, false, or deceitful. Auditing this at scale is currently a manual process — this project automates it.

---

## Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│  16 brand PDFs (sustainability reports, 2023–2025)              │
└───────────────────────────┬─────────────────────────────────────┘
                            │ pymupdf4llm → Markdown
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  Rolling-window segmentation                                    │
│  window = 5 sentences · stride = 3 · min = 3 sentences         │
│  ~7,000 raw passages                                            │
└───────────────────────────┬─────────────────────────────────────┘
                            │ ClimateBERT topic filter
                            │ (distilroberta-base-climate-detector)
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  4,485 climate-relevant passages                                │
└────────────┬─────────────────────────────┬──────────────────────┘
             │                             │
             ▼                             ▼
┌────────────────────────┐   ┌─────────────────────────────────────┐
│  Stratified sample     │   │  Feature extraction (all 4,485)     │
│  20 / brand → 320      │   │  · sentence embeddings (MiniLM)     │
│  Manual annotation     │   │  · vague language score             │
│  ├── 92  → train       │   │  · red-flag proprietary terms       │
│  ├── 48  → test (locked│   │  · scope (own-ops vs supply-chain)  │
│  └── 180 → excluded    │   │  · readability grade (FK)           │
└──────────┬─────────────┘   │  · unregulated beauty claims        │
           │                 └──────────────┬──────────────────────┘
           ▼                                │
┌─────────────────────────────────────────────────────────────────┐
│  Modeling                                                       │
│                                                                 │
│  Track A — kNN (k=7, cosine)        Track B — TF-IDF + LR      │
│  semantic similarity to labeled     lexical pattern matching    │
│  training passages                  (unigrams + bigrams)        │
│                                                                 │
│  Track C — ClimateBERT commitment classifier (corroborating)   │
└───────────────────────────┬─────────────────────────────────────┘
                            │ predict on all 4,485 passages
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  Analysis & Evaluation                                          │
│  · brand-level risk scores      · supply-chain vs own-ops       │
│  · cross-track agreement        · top predictive words          │
│  · bootstrap CIs (n=1,000)      · extrinsic validation          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Brands

| Industry | Brand | Role | Passages |
|---|---|---|---|
| Fashion | H&M | Greenwashing (ASA ruling 2022) | 553 |
| Fashion | Zara | Greenwashing (EU scrutiny) | 524 |
| Fashion | Shein | Greenwashing | 307 |
| Fashion | Lululemon | Greenwashing | 235 |
| Fashion | Everlane | Greenwashing | 198 |
| Fashion | Ralph Lauren | Greenwashing | 172 |
| Fashion | Reformation | Ambiguous | 171 |
| Fashion | Puma | Ambiguous | 758 |
| Fashion | Kering | Greenwashing | 89 |
| Fashion | Patagonia | Positive control (B Corp) | 448 |
| Fashion | Eileen Fisher | Positive control (B Corp) | 26 |
| Clean Beauty | Unilever | Greenwashing | 492 |
| Clean Beauty | L'Oréal | Greenwashing (EU complaint 2022) | 23 |
| Clean Beauty | Sephora | Greenwashing (class action 2022) | 139 |
| Clean Beauty | L'Occitane | Positive control (B Corp) | 279 |
| Clean Beauty | Lush | Positive control (cruelty-free certified) | 71 |

Role assignment uses B Corp certification as the primary proxy for positive controls. Raw PDFs are not committed to this repository due to copyright.

---

## Annotation Schema

320 passages were manually labeled (20 per brand, stratified). Labels:

| Label | Name | Description | Count |
|---|---|---|---|
| 0 | Substantive | Specific, measurable, or third-party verified | 98 |
| 1 | Greenwashing-risk | Vague, unverified, or contradicted by auditor ratings | 42 |
| 2 | Ambiguous | Genuine uncertainty — excluded from modeling | 180 |

The 56% ambiguity rate is itself a finding: the majority of sustainability language resists clean classification.

Train / test split: **92 train · 48 test (locked)**

---

## Results

### Model Performance

| Model | Macro F1 | AUC-ROC | AUC-PR |
|---|---|---|---|
| kNN (k=7) | 0.714 | 0.874 | 0.674 |
| TF-IDF + LR | **0.758** | **0.891** | **0.724** |

Bootstrap 95% CIs (n=1,000 · 48 test passages):

| Model | F1 CI | AUC-ROC CI |
|---|---|---|
| kNN | [0.564, 0.829] | [0.762, 0.953] |
| TF-IDF + LR | [0.613, 0.885] | [0.782, 0.976] |

Random baseline macro F1 ≈ 0.44 given 2.3:1 class imbalance. Both lower bounds clear this comfortably. CIs overlap entirely — no reliable difference between models at this test set size.

### Cross-Track Agreement

| When both tracks say... | Agreement rate | Passages |
|---|---|---|
| Substantive | 93% | 2,837 |
| Greenwashing-risk | 35% | 1,648 |

kNN casts a wider net (1,648 greenwash flags vs TF-IDF's 573). Dual-track agreement functions as a confidence filter — passages both models flag are higher-confidence predictions.

### Top Predictive Words (TF-IDF coefficients)

```
GREENWASHING-RISK                    SUBSTANTIVE
─────────────────────────────────    ────────────────────────────────
+ supply chain    + committed         - double materiality  - scope
+ responsible     + communities       - data                - analysis
+ circularity     + innovation        - financial           - statement
+ future          + impact            - report              - policies
+ materials       + operations        - costs               - strategy
```

Greenwashing-risk language is aspirational and relational. Substantive language is formal, quantitative, and tied to recognised disclosure frameworks (double materiality is a technical EU CSRD concept).

### Supply-Chain vs Own-Operations

Supply-chain passages score higher greenwash risk than own-operations passages in **10 of 12 brands**. Brands make credible claims about what they directly control and vague claims about what they don't.

```
Brand          Own-ops prob   Supply-chain prob   Gap
────────────── ────────────   ─────────────────   ─────
L'Oréal             —              0.679            —
Patagonia         0.321            0.574          +0.253
Ralph Lauren      0.286            0.502          +0.216
H&M               0.379            0.503          +0.124
Lululemon         0.429            0.516          +0.087
Everlane          0.443            0.532          +0.089
```

### Red-Flag Proprietary Term Rates

Brands with the highest use of self-defined sustainability labels lacking third-party verification:

```
Lululemon     ████████████████  13.6%
Everlane      ███████████████   13.1%
H&M           ██████████        9.8%
Shein         █████████         9.1%
Zara          ████████          7.8%
Eileen Fisher ████████          7.7%
──────────────────────────────
Lush          █                 0.8%   ← positive control
L'Occitane    █                 1.1%   ← positive control
```

---

## Repository Structure

```
green-claims-nlp/
│
├── data/
│   ├── raw/                          # PDFs (not committed — copyright)
│   │   ├── hm-2025.pdf
│   │   ├── zara-2025.pdf
│   │   └── ...
│   ├── extracted/                    # Phase 1 outputs
│   │   ├── passages.jsonl            # 4,485 climate-relevant passages
│   │   ├── passage_counts.csv        # Per-brand passage counts
│   │   └── sentences_cache/          # Per-brand sentence cache (pkl)
│   ├── labeled/                      # Phase 2 outputs
│   │   ├── to_annotate.csv           # Annotation working file
│   │   ├── labeled_passages.jsonl    # 92 training passages
│   │   ├── test_set.jsonl            # 48 held-out test passages (locked)
│   │   └── ambiguous_passages.jsonl  # 180 excluded passages
│   └── features/                     # Phase 3 outputs
│       ├── features.parquet          # Handcrafted features (4,485 rows)
│       ├── corpus_embeddings.npy     # MiniLM embeddings (4,485 × 384)
│       └── labeled_embeddings.npy    # MiniLM embeddings (92 × 384)
│
├── notebooks/
│   ├── 01_preprocessing.ipynb        # PDF parsing, segmentation, topic filter
│   ├── 02_annotation.ipynb           # Stratified sampling, label loading, train/test split
│   ├── 03_features.ipynb             # Embeddings + handcrafted features
│   ├── 04_modeling.ipynb             # kNN, TF-IDF+LR, ClimateBERT commitment
│   ├── 05_analysis.ipynb             # Brand/industry analysis, visualisations
│   └── 06_evaluation.ipynb           # Test set metrics, bootstrap CIs, top words
│
├── results/
│   ├── knn_predictions.parquet       # kNN scores for all 4,485 passages
│   ├── tfidf_predictions.parquet     # TF-IDF scores for all 4,485 passages
│   ├── climatebert_predictions.parquet
│   ├── brand_risk_scores.csv         # Aggregated brand-level risk scores
│   ├── feature_comparison.csv        # Fashion vs clean beauty feature breakdown
│   ├── evaluation_metrics.csv        # Test set precision / recall / F1 / AUC
│   ├── cross_track_disagreements.csv # Passages where Track A and B disagree
│   ├── qualitative_examples.md       # Top flagged passages with NN rationale
│   ├── best_k.txt                    # Optimal k from cross-validation
│   ├── tsne_embeddings.png           # Full-corpus t-SNE
│   ├── tsne_labeled_only.png         # t-SNE on 92 labeled passages
│   ├── feature_scatter_labeled.png   # Vague language vs specific numbers scatter
│   ├── tfidf_top_words.png           # LR coefficient chart
│   ├── corpus_balance.png
│   └── carbon/
│       ├── emissions.csv
│       └── carbon_summary.md
│
├── pyproject.toml                    # Dependencies (uv)
├── uv.lock
└── README.md
```

---

## Limitations

- **Brand-level aggregation is unreliable.** Passage-level predictions do not aggregate cleanly to brand-level rankings. kNN captures semantic topic similarity — all sustainability reports discuss the same topics — so positive controls can score high alongside bad actors. Brand-level risk scores should be treated as exploratory, not conclusive.
- **Small training set.** 92 labeled training passages across 16 brands. Wide bootstrap CIs and the high ambiguity rate reflect this constraint honestly.
- **Reports vary in structure and length.** The pipeline cannot control for deliberate omission — a brand that simply doesn't discuss a topic won't have passages flagged for it.
- **Public-facing reports only.** Analysis is limited to formal sustainability disclosures, not consumer-facing product copy, advertising, or social media.

---

## Setup

```bash
# Install uv if you don't have it
curl -LsSf https://astral.sh/uv/install.sh | sh

# Clone and enter the repo
git clone https://github.com/mandyjsun/green-claims-nlp.git
cd green-claims-nlp

# Create virtual environment and install all dependencies
uv sync
```

Run notebooks in order: `01` → `02` → `03` → `04` → `05` → `06`.
Each notebook reads outputs from the previous phase and writes to `data/` or `results/`.

---

## Carbon Footprint

Total pipeline cost: **0.0003 kg CO₂e** — run entirely on CPU (2021 MacBook Pro, Apple M1).
Tracked per phase using [CodeCarbon](https://codecarbon.io/). Full breakdown in `results/carbon/carbon_summary.md`.

---

## Author

Mandy Sun — DSAN 5550, Georgetown University, Spring 2026
