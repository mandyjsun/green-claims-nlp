# Project plan — green-claims-nlp

Detecting greenwashing-risk language in corporate sustainability reports using a sentence-transformer + cosine kNN pipeline, benchmarked against TF-IDF and fine-tuned ClimateBERT.

**Timeline:** Spring 2026 · DSAN 5550, Georgetown University  
**Method anchor:** Sampson, Cotterill & Au (NeurIPS 2022) — stacked TF-IDF + kNN outperforms fine-tuned ClimateBERT (AUC 0.884 vs 0.865)

---

## Carbon footprint tracking

All computation in this project is tracked for carbon emissions using [CodeCarbon](https://codecarbon.io/). This serves two purposes: (1) it is good research practice to report the environmental cost of NLP work, especially in a project about environmental claims; (2) it allows us to report kWh and CO₂e in the paper, consistent with emerging norms in climate-adjacent NLP research.

### Setup

Install and import at the top of every notebook that runs model training or inference:

```python
pip install codecarbon

from codecarbon import EmissionsTracker

tracker = EmissionsTracker(
    project_name="green-claims-nlp",
    output_dir="results/carbon/",
    output_file="emissions.csv",        # appends across runs
    log_level="error"                   # suppress verbose output
)
```

### Usage pattern

Wrap every substantive computation block:

```python
tracker.start()

# --- your code here ---
# e.g. encoding all passages, training kNN, running ClimateBERT inference

emissions = tracker.stop()
print(f"Emissions: {emissions:.4f} kg CO2e")
```

### What to track

Track each of the following separately so you can report them individually:

| Block | Notebook | Label |
|---|---|---|
| ClimateBERT topic filtering (Phase 2) | `01_preprocessing.ipynb` | `climatebert_filter` |
| Sentence-transformer encoding (Phase 4) | `03_features.ipynb` | `embedding_corpus` |
| kNN training + inference (Phase 5) | `04_modeling.ipynb` | `knn_train_predict` |
| TF-IDF + LR training + inference (Phase 5) | `04_modeling.ipynb` | `tfidf_train_predict` |
| ClimateBERT classification inference (Phase 5) | `04_modeling.ipynb` | `climatebert_classify` |

### Reporting

In the paper/poster, report total kWh consumed and total kg CO₂e for the full pipeline. Note the hardware used (CPU vs GPU, cloud vs local). Cite CodeCarbon (Courty et al., 2023).

> Courty, B., Schmidt, V., Goyal-Kamal, et al. (2023). *CodeCarbon: Estimate and Track Carbon Emissions from Machine Learning Computing.* https://github.com/mlco2/codecarbon

### Output
- `results/carbon/emissions.csv` — per-run emissions log (appended automatically by CodeCarbon)

---

## Corpus

| Industry | Brand | Role | Notes |
|---|---|---|---|
| Fashion | H&M | Greenwashing (regulatory ruling) | 2025 report |
| Fashion | Zara (Inditex) | Greenwashing (regulatory scrutiny) | 2025 report (FY2025, pub. March 2026), 165 pages — note fiscal year mismatch with H&M |
| Fashion | Patagonia | Positive control (verified) | 2024 report, ~130 pages |
| Clean Beauty | L'Oréal | Greenwashing (hall of shame) | 2024 report |
| Clean Beauty | Sephora | Greenwashing (class action) | 2024 report |
| Clean Beauty | Lush | Positive control (verified) | 2025 All-One! report |

**Corpus balance note:** Patagonia (~130 pages) and Lush (~60-80 pages) are smaller than the greenwashing brands (~165 pages each). All brand-level scores must be reported as proportions (share of passages flagged), not raw counts. Report passage counts per brand in a descriptive table in the paper.

**Fiscal year note:** All reports are 2025 publications. The Inditex report covers FY2025; confirm the fiscal year coverage for other brands and note any mismatches in the Limitations section.

---

## Notebook structure

```
notebooks/
├── 01_preprocessing.ipynb
├── 02_annotation.ipynb
├── 03_features.ipynb
├── 04_modeling.ipynb
├── 05_analysis.ipynb
└── 06_evaluation.ipynb
```

---

## Phase 0 — Pilot reading (complete)
**Week 1 — completed prior to coding**

This phase is real analytical work that informs the annotation guide, the hedge word lexicon, and the supplementary features. It is documented here as a completed phase.

### Completed
- [x] Deep-read H&M 2024 Annual & Sustainability Report (166 pages)
- [x] Deep-read Inditex 2025 Non-Financial Information Report (165 pages)
- [x] Identified primary greenwashing risk patterns per brand:
  - H&M: aspirational brand language ("positive power of fashion", "liberating fashion for the many"); "sustainably sourced" as self-defined standard; scope 1+2 progress masking slow scope 3 improvement
  - Inditex: proprietary definitional manipulation ("lower-impact fibres" — self-defined, 88% claim rests entirely on own criteria); own-operations decarbonisation masking slow supply chain progress
- [x] Identified key terminology for annotation guide and lexicon
- [x] Identified own-operations vs supply-chain passage distinction as key analytical feature
- [x] Confirmed that rolling 10-sentence windows are appropriate given multi-sentence hedging patterns observed in both reports

### Key insight from pilot reading
Both H&M and Inditex show a systematic pattern: strong progress on own operations (renewable energy, scope 1+2 emissions) paired with much weaker progress on supply chain (scope 3). Companies can make truthful own-operations claims while their total footprint barely moves. This own-operations vs supply-chain distinction must be operationalised as a feature and discussed in the paper.

---

## Phase 1 — Data collection

### Goals
- Download all 6 sustainability reports as PDFs
- Log provenance for each

### Tasks
- [x] Download H&M 2025 sustainability report 
- [x] Download Zara / Inditex 2025 sustainability report
- [x] Download Patagonia 2025 environmental + social initiative report
- [x] Download L'Oréal 2025 sustainability report
- [x] Download Sephora 2024 sustainability report (most recent)
- [x] Download Lush 2025 All-One! report
- [ ] Record source URL, access date, page count, and year for each in `data/raw/sources.csv`
- [ ] Confirm report years and note any fiscal year mismatches in sources.csv

### Output
- `data/raw/` — 6 PDF files + `sources.csv` — provenance log with columns: brand, industry, role, url, access_date, page_count, fiscal_year, publication_year, notes

---

## Phase 2 — Preprocessing + filtering
**Week 2 · `01_preprocessing.ipynb`**

### Goals
- Parse raw PDFs into clean text
- Build overlapping paragraph windows for context
- Filter down to environment-relevant passages only

### Tasks
- [ ] Parse each PDF with `pdfplumber`, extract text page by page
- [ ] Sentence-tokenize using `nltk` or `spacy`
- [ ] Build rolling 10-sentence windows with 5-sentence stride (Sampson et al. §3.2)
- [ ] Apply secondary length filter: drop windows shorter than 3 sentences to exclude decorative mentions and section headers that ClimateBERT may retain
- [ ] Apply ClimateBERT topic classifier (`climatebert/distilroberta-base-climate-detector`) to each window — **wrap in CodeCarbon tracker, label `climatebert_filter`**
- [ ] Retain only windows classified as climate-relevant
- [ ] Tag each retained passage with brand name and industry
- [ ] Record passage count per brand in a descriptive table — flag any brand where count differs by more than 3x from the median
- [ ] Save filtered passages to `data/raw/extracted/passages.jsonl`
- [ ] Release `data/raw/extracted/` as public dataset (secondary contribution)

### Notes
- Do not attempt to "restructure" reports — the pipeline works on any PDF regardless of format
- Rolling window adds context so the model sees hedged language that spans sentences, not just individual sentences in isolation
- Front matter (CEO letters, brand profiles, financial tables) will be parsed — ClimateBERT filter plus length filter will remove most non-substantive mentions

### Output
- `data/raw/extracted/passages.jsonl` — filtered passages with brand/industry metadata
- `data/raw/extracted/passage_counts.csv` — passage count per brand for reporting

---

## Phase 3 — Annotation
**Week 2–3 · `02_annotation.ipynb`**

### Goals
- Write the annotation guide before labeling anything
- Build a labeled dataset of ~100 passages for training and evaluation
- Lock train/test split before touching any model

### Tasks

#### Step 1: Write annotation_guide.md first
- [ ] Write `data/raw/labeled/annotation_guide.md` before labeling any passages
- [ ] Guide must define the binary label criteria:
  - **Label 1 — greenwashing-risk:** vague commitments without quantified targets; unverifiable claims; self-defined proprietary standards without third-party verification; unregulated beauty terms used as sustainability claims; relative comparisons without benchmarks; future-tense aspirations without timelines
  - **Label 0 — substantive:** specific measurable goals with timelines; third-party verified claims (SBTi, GRI, Deloitte assurance, ZDHC); concrete emissions data with methodology disclosed; named certification standards (not proprietary ones)
- [ ] Guide must include worked examples for each label — minimum 5 per label drawn from pilot reading:
  - Greenwashing-risk examples: "lower-impact fibres" (Inditex self-defined), "sustainably sourced" (H&M self-defined), "positive power of fashion", "we are committed to decarbonising our supply chain"
  - Substantive examples: "scope 1 and 2 emissions reduced 88% vs 2018 baseline, SBTi-validated", "97.1% MRSL compliance for chemical inputs, measured via ZDHC Gateway"
- [ ] Guide must address these edge cases explicitly:
  - Self-defined standards ("lower-impact", "sustainably sourced", "responsibly made") → label 1 unless linked to a named third-party standard
  - Relative claims without external benchmarks ("reduced by X% vs our own 2019 baseline") → label 0 if methodology is disclosed, label 1 if baseline is undefined
  - Future tense commitments → label 0 if quantified with timeline, label 1 if aspirational only
  - Own-operations vs supply-chain claims → same criteria apply, but note the distinction in metadata
  - Proprietary program membership (Better Cotton Initiative, Fashion Pact) → label 1 unless specific outcome metrics are reported
- [ ] Guide must define a third working label: **label 2 — ambiguous** for genuine uncertainty; collapse to binary for modeling but track separately

#### Step 2: Sample and label
- [ ] Sample ~100 passages from `passages.jsonl`, stratified across all 6 brands (~17 per brand) and across passage types (aspirational framing / quantified claim / policy/procedure / supply chain / product claim)
- [ ] Label each passage using annotation_guide.md — apply label 2 (ambiguous) freely, do not force binary when uncertain
- [ ] For label 2 passages: write a one-sentence note explaining the uncertainty
- [ ] Hold out 20 passages (20%) as locked test set — never adjust after this point
- [ ] Save labeled set to `data/raw/labeled/labeled_passages.jsonl`
- [ ] Save test set separately to `data/raw/labeled/test_set.jsonl`

#### Step 3: Inter-annotator check (if possible)
- [ ] If a second annotator is available, ask them to label 20 passages independently using the annotation_guide
- [ ] Compute Cohen's kappa — even a partial check strengthens the paper
- [ ] Report kappa in the paper regardless of value; low kappa is a finding, not a failure

### Notes
- The test set must be locked now. Do not re-examine or adjust it based on model results — ever
- Label 2 passages are excluded from model training but reported in the paper as evidence of task difficulty

### Output
- `data/raw/labeled/annotation_guide.md` — criteria, worked examples, edge cases (written first)
- `data/raw/labeled/labeled_passages.jsonl` — ~80 training examples (labels 0 and 1 only)
- `data/raw/labeled/ambiguous_passages.jsonl` — label 2 passages, excluded from training
- `data/raw/labeled/test_set.jsonl` — 20 held-out test examples

---

## Phase 4 — Feature extraction
**Week 3 · `03_features.ipynb`**

### Goals
- Produce the sentence-transformer embeddings that power kNN
- Compute supplementary features for qualitative analysis and cross-industry comparison

### Tasks

#### Embeddings (kNN backbone)
- [ ] Load `sentence-transformers/all-MiniLM-L6-v2`
- [ ] Encode all passages in `passages.jsonl` → `corpus_embeddings.npy` — **wrap in CodeCarbon tracker, label `embedding_corpus`**
- [ ] Encode all labeled passages → `labeled_embeddings.npy`
- [ ] L2-normalize all embeddings for cosine similarity

#### Supplementary features
- [ ] Build hedge word lexicon from greenwashing literature + manual review of pilot documents:
  - Base: Loughran & McDonald (2011) uncertainty wordlist
  - Domain additions from pilot reading: *committed to*, *working towards*, *aims to*, *strives to*, *exploring*, *ambition is*, *journey towards*, *we believe*, *we strive*, *in a sustainable way*, *positive impact*, *conscious*, *responsible*
- [ ] Compute hedge word density per passage (proportion of lexicon tokens)
- [ ] Build proprietary standard flag: passages containing self-defined sustainability vocabulary without a named third-party standard:
  - Flag terms: *lower-impact*, *sustainably sourced*, *responsibly made*, *conscious choice*, *better* (as modifier before material/cotton/wool), *eco-designed*
  - Do not flag if same sentence contains named standard: *SBTi*, *GRI*, *ZDHC*, *FSC*, *bluesign*, *Oeko-Tex*, *Fair Trade*, *B Corp*
- [ ] Regex for quantitative targets: detect percentages, year references, measurable metrics
- [ ] Build scope flag: does passage refer to own operations or supply chain?
  - Own operations keywords: *our stores*, *our offices*, *scope 1*, *scope 2*, *own operations*, *our facilities*, *our headquarters*
  - Supply chain keywords: *supply chain*, *suppliers*, *scope 3*, *value chain*, *tier 1*, *tier 2*, *upstream*, *downstream*, *manufacturers*
  - Label: `own_ops` / `supply_chain` / `mixed` / `unclear`
- [ ] Compute Flesch-Kincaid grade level per passage
- [ ] Build clean beauty unregulated term flag: *clean*, *natural*, *non-toxic*, *reef-safe*, *green*, *pure*, *eco*, *gentle*, *plant-based* (flag only for beauty industry passages)
- [ ] Save all features to `data/raw/features/features.parquet`

### Notes
- Supplementary features are not inputs to kNN — they are used in Phase 6 to interpret and explain model outputs
- The scope flag and proprietary standard flag are new features identified during pilot reading — they operationalise the key theoretical insights from the corpus analysis

### Output
- `data/raw/features/corpus_embeddings.npy`
- `data/raw/features/labeled_embeddings.npy`
- `data/raw/features/features.parquet`

---

## Phase 5 — Modelling (3 parallel tracks)
**Week 4–5 · `04_modeling.ipynb`**

### Goals
- Train and run all three classifiers on the full corpus
- kNN is the primary model; TF-IDF and ClimateBERT are comparison benchmarks

### Track A — kNN (primary model)

```python
from sklearn.neighbors import KNeighborsClassifier
from sklearn.preprocessing import normalize
from codecarbon import EmissionsTracker

tracker = EmissionsTracker(project_name="green-claims-nlp",
                           output_dir="results/carbon/",
                           output_file="emissions.csv")
tracker.start()

X_train = normalize(labeled_embeddings)
X_test  = normalize(corpus_embeddings)

knn = KNeighborsClassifier(n_neighbors=5, metric="cosine")
knn.fit(X_train, labels)

predictions   = knn.predict(X_test)
probabilities = knn.predict_proba(X_test)  # [:, 1] = greenwash probability

emissions = tracker.stop()  # label: knn_train_predict
```

**Nearest-neighbor interpretability:**

```python
distances, indices = knn.kneighbors(X_test)
# indices[i] → row indices into labeled set → human-readable nearest examples
```

Tasks:
- [ ] Tune k (try k = 3, 5, 7, 10) on validation passages, pick best by F1
- [ ] Check probability calibration: plot calibration curve on validation set
- [ ] Run on full corpus, save predictions + probabilities
- [ ] For every passage with greenwash prob > 0.6, save top-3 nearest labeled neighbors
- [ ] Save results to `results/knn_predictions.parquet`

### Track B — TF-IDF + logistic regression (baseline)

```python
from sklearn.pipeline import Pipeline
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression

tracker.start()

baseline = Pipeline([
    ("tfidf", TfidfVectorizer(max_features=5000, ngram_range=(1, 2))),
    ("clf",   LogisticRegression(max_iter=1000))
])

baseline.fit(train_texts, train_labels)
baseline_preds = baseline.predict(corpus_texts)

emissions = tracker.stop()  # label: tfidf_train_predict
```

Tasks:
- [ ] Fit on training passages, predict on full corpus
- [ ] Save results to `results/tfidf_predictions.parquet`

### Track C — ClimateBERT (comparison)

**Important note:** `climatebert/distilroberta-base-climate-f` classifies climate-related framing, not greenwashing per se. Its output is not directly comparable to Tracks A and B as a greenwashing classifier. Use it as a framing/sentiment comparison signal only — document this distinction in the paper.

```python
from transformers import pipeline

tracker.start()

climatebert = pipeline(
    "text-classification",
    model="climatebert/distilroberta-base-climate-f",
    truncation=True,
    max_length=512
)

cb_preds = [climatebert(p)[0] for p in corpus_passages]

emissions = tracker.stop()  # label: climatebert_classify
```

Tasks:
- [ ] Run inference on all passages — **wrap in CodeCarbon tracker**
- [ ] Document in notebook that ClimateBERT output is a framing signal, not a greenwashing label
- [ ] Save results to `results/climatebert_predictions.parquet`

### Output
- `results/knn_predictions.parquet`
- `results/tfidf_predictions.parquet`
- `results/climatebert_predictions.parquet`
- `results/carbon/emissions.csv` — updated with all three runs

---

## Phase 6 — Cross-industry analysis
**Week 5–6 · `05_analysis.ipynb`**

### Goals
- Aggregate to brand-level risk scores
- Use nearest-neighbor retrieval to illustrate findings qualitatively
- Test cross-industry hypotheses about language patterns

### Tasks

#### Brand-level risk scores
- [ ] Compute mean greenwash probability per brand from kNN predictions
- [ ] **Report as proportions, not raw counts** — divide by total passages per brand
- [ ] Include passage count per brand in the results table
- [ ] Rank all 6 brands — check that Lush and Patagonia score lowest
- [ ] Compare kNN scores to TF-IDF and ClimateBERT scores for consistency

#### Own-operations vs supply-chain analysis (new)
- [ ] Split all passages by scope flag (own_ops / supply_chain)
- [ ] Compute greenwash probability separately for own-operations and supply-chain passages, per brand
- [ ] Test hypothesis: supply-chain passages score higher across all brands including positive controls
- [ ] This operationalises the key insight from pilot reading: companies can truthfully report own-operations progress while supply-chain claims remain vague

#### Proprietary standard analysis (new)
- [ ] Compute proprietary standard flag rate per brand
- [ ] Test hypothesis: Inditex and H&M use more self-defined terminology than Patagonia and Lush
- [ ] Cross-tabulate proprietary standard flag with greenwash probability

#### Nearest-neighbor qualitative analysis
- [ ] For each brand, pull the 5 highest-scoring passages
- [ ] For each, show top-3 nearest labeled training examples and cosine distances
- [ ] Write 1–2 sentence interpretation of why each passage was flagged
- [ ] This becomes the qualitative evidence section of the paper

#### Cross-industry feature comparison
- [ ] Compare hedge word density: fashion brands vs clean beauty brands
- [ ] Compare quantitative target presence: fashion vs beauty
- [ ] Compare unregulated beauty term frequency: beauty industry only
- [ ] Compare Flesch-Kincaid readability across brands
- [ ] Test refined hypothesis:
  - Fashion brands show higher hedge density in **supply chain** passages specifically
  - Clean beauty brands show higher unregulated terminology density in **product-level** claims

### Output
- `results/brand_risk_scores.csv` — brand, passage_count, mean_greenwash_prob, own_ops_prob, supply_chain_prob
- `results/qualitative_examples.md`
- `results/feature_comparison.csv`

---

## Phase 7 — Evaluation + write-up
**Week 6–7 · `06_evaluation.ipynb`**

### Goals
- Report intrinsic metrics on held-out test set
- Validate against external benchmarks
- Report carbon footprint of the full pipeline
- Produce poster / paper

### Tasks

#### Intrinsic evaluation
- [ ] Evaluate all three models on `test_set.jsonl`
- [ ] Report precision, recall, F1 for each model
- [ ] Report ROC-AUC for comparability with Sampson et al.
- [ ] Produce confusion matrices
- [ ] Report proportion of label-2 (ambiguous) passages excluded from training

#### Extrinsic validation
- [ ] Fashion: compare kNN brand scores to Good On You ratings (H&M, Zara, Patagonia)
- [ ] Beauty: compare to specific regulatory outcomes (Sephora class action, L'Oréal EU complaints)
- [ ] Include in paper: "We treat extrinsic validation as a plausibility check, not a formal hypothesis test, given N=6."

#### Carbon footprint reporting
- [ ] Load `results/carbon/emissions.csv`
- [ ] Summarise total kWh and total kg CO₂e for the full pipeline
- [ ] Report per-component breakdown
- [ ] Note hardware (CPU vs GPU, local vs cloud)
- [ ] Include in paper: "We report the carbon cost of this pipeline as a matter of research transparency, consistent with emerging norms in climate-adjacent NLP."

#### Write-up sections
- [ ] **Methods:** cite Sampson et al. (2022), Alonso-Robisco et al. (2023), Spokoyny et al. (2024); describe annotation guide and inter-annotator reliability; describe scope flag and proprietary standard flag as novel features
- [ ] **Results:** model comparison table; brand risk score table with passage counts; own-ops vs supply-chain breakdown
- [ ] **Discussion:** caveat N=6; qualitative nearest-neighbor analysis; cross-industry patterns; own-operations vs supply-chain as key finding; proprietary standard terminology as a distinct greenwashing mechanism
- [ ] **Limitations:**
  - Small labeled set; single annotator
  - Self-defined standards are not distinguished from third-party verified claims by surface language alone — a fundamental limitation of any lexical/embedding approach
  - Report length imbalance across brands
  - Fiscal year coverage may differ across brands even if all reports are 2025 publications — verify and disclose
  - Consumer-facing product copy not included
  - Binary label is a simplification of a continuous spectrum
- [ ] **Future work:** greenwashing gap score (reports vs product copy); scale to 20+ brands; formal inter-annotator study

### Output
- `results/evaluation_report.md`
- `results/carbon/carbon_summary.md`
- Poster / paper draft

---

## Key methodological decisions

| Decision | Rationale |
|---|---|
| kNN as primary model | Best performing in Sampson et al. (AUC 0.884); works well with small labeled sets; interpretable via nearest-neighbor retrieval |
| Sentence-transformers over BERT encoder | Embedding space optimized for semantic similarity, not just token prediction |
| Rolling 10-sentence windows | Adds context for hedged language that spans sentences (Sampson et al. §3.2) |
| Minimum 3-sentence window length filter | Removes section headers and decorative mentions that ClimateBERT may retain |
| 3 models in parallel | kNN is primary; TF-IDF and ClimateBERT are benchmarks; ClimateBERT Track C is a framing signal, not a greenwashing classifier |
| Locked test set at annotation | Prevents data leakage; test set never seen during development |
| Annotation guide written before any labeling | Ensures consistent decisions; reduces annotator drift; required for reproducibility |
| Own-ops vs supply-chain scope flag | Operationalises key insight from pilot reading: companies can make truthful own-operations claims while supply-chain claims remain vague |
| Proprietary standard flag | Captures definitional manipulation — companies creating self-defined vocabulary to make ambiguous claims sound specific |
| N=6 framed as exploratory | Per professor feedback — findings are descriptive, not hypothesis tests |
| CodeCarbon throughout | Research transparency; consistent with norms in climate-adjacent NLP |

---

## Data directory structure

```
data/
└── raw/
    ├── sources.csv
    ├── hm-2025.pdf
    ├── zara-2025.pdf
    ├── patagonia-2025.pdf
    ├── loreal-2025.pdf
    ├── sephora-2024.pdf
    ├── lush-2025.pdf
    ├── extracted/
    │   ├── passages.jsonl
    │   └── passage_counts.csv
    ├── labeled/
    │   ├── annotation_guide.md        ← written first
    │   ├── labeled_passages.jsonl
    │   ├── ambiguous_passages.jsonl
    │   └── test_set.jsonl
    └── features/
        ├── corpus_embeddings.npy
        ├── labeled_embeddings.npy
        └── features.parquet

results/
├── knn_predictions.parquet
├── tfidf_predictions.parquet
├── climatebert_predictions.parquet
├── brand_risk_scores.csv
├── feature_comparison.csv
├── qualitative_examples.md
├── evaluation_report.md
└── carbon/
    ├── emissions.csv
    └── carbon_summary.md
```

---

## References

- Sampson, Cotterill & Au (2022). TCFD-NLP: Assessing alignment of climate disclosures using NLP. *NeurIPS 2022 Climate Change AI Workshop.*
- Alonso-Robisco, Carbó & Marqués (2023). Machine learning methods in climate finance: A systematic review. *Banco de España Working Paper 2310.* `doi:10.53479/29594`
- Spokoyny, Callaghan & Schimanski (2024). NLP models for climate policy analysis. *Climate Change AI Summer School.* `doi:10.5281/zenodo.12533572`
- Webersinke, Kraus, Bingler & Leippold (2021). ClimateBERT: A pretrained language model for climate-related text. `arXiv:2110.12010`
- Bingler, Kraus, Leippold & Webersinke (2022). Cheap talk and cherry-picking: What ClimateBERT has to say on corporate climate risk disclosures. *Finance Research Letters.*
- Courty, B., Schmidt, V., Goyal-Kamal, et al. (2023). CodeCarbon: Estimate and Track Carbon Emissions from Machine Learning Computing. https://github.com/mlco2/codecarbon