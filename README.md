# green-claims-nlp

Automated detection of greenwashing-risk language in corporate sustainability reports using domain-adapted NLP. This project applies a fine-tuned ClimateBERT classifier to sustainability disclosures from fashion and clean beauty brands, testing whether greenwashing-risk language manifests differently across industries.

> **Status:** In development — DSAN 5550 course project, Georgetown University, Spring 2026.

---

## Overview

Environmental greenwashing — making unsubstantiated or misleading sustainability claims — is pervasive across consumer industries, but auditing it at scale remains largely manual. This project asks: *can NLP automatically detect greenwashing-risk language in corporate sustainability reports, and does that risk look different across industries?*

The central hypothesis is that greenwashing-risk language follows distinct linguistic patterns depending on industry: fashion brands tend to rely on vague systemic commitments, while clean beauty brands exploit unregulated terminology like *natural*, *clean*, and *non-toxic*.

---

## Brands

| Industry | Brand | Role |
|---|---|---|
| Fashion | H&M | Greenwashing case (regulatory ruling) |
| Fashion | Zara (Inditex) | Greenwashing case (regulatory scrutiny) |
| Fashion | Patagonia | Positive control (verified sustainability) |
| Clean Beauty | L'Oréal | Greenwashing case (hall of shame, FTC scrutiny) |
| Clean Beauty | Sephora | Greenwashing case (class action lawsuit) |
| Clean Beauty | Lush | Positive control (third-party verified) |

All reports are sourced from company investor relations pages (2024–2025 publications).

---

## Pipeline

```
PDF reports
    └── pdfplumber parsing
        └── ClimateBERT topic filter (retain env. passages only)
            └── Feature extraction
            │       ├── ClimateBERT embeddings (distilroberta-base-climate-f)
            │       ├── Hedge word density (custom lexicon)
            │       ├── Quantitative target detection (regex)
            │       ├── Flesch-Kincaid readability
            │       └── Unregulated beauty term flags (clean, natural, non-toxic, reef-safe)
            └── Classification
                    ├── Fine-tuned ClimateBERT (substantive vs. greenwashing-risk)
                    └── TF-IDF + logistic regression (baseline)
```

---

## Repository Structure

```
green-claims-nlp/
├── data/
│   ├── raw/                  # Original PDFs (not committed — see Data section)
│   └── extracted/            # Filtered sustainability passages (released as public dataset)
├── notebooks/
│   ├── 01_preprocessing.ipynb
│   ├── 02_feature_extraction.ipynb
│   ├── 03_classification.ipynb
│   └── 04_cross_industry_analysis.ipynb
├── src/
│   ├── parse.py              # PDF parsing with pdfplumber
│   ├── filter.py             # ClimateBERT topic filtering
│   ├── features.py           # Feature extraction
│   ├── classify.py           # Model training and inference
│   └── evaluate.py           # Metrics and external validation
├── models/                   # Saved model checkpoints (gitignored)
├── results/                  # Risk scores, evaluation outputs
├── pyproject.toml            # Project metadata and dependencies (uv)
├── uv.lock                   # Locked dependency versions
└── README.md
```

---

## Evaluation

**Intrinsic:** Precision, recall, and F1 on a held-out labeled test set drawn from the extracted passages.

**Extrinsic validation:**
- Fashion brands: CDP scores, Good On You ratings
- Clean beauty brands: EWG Skin Deep scores, documented FTC actions and legal outcomes

**Cross-industry analysis** is framed as an exploratory descriptive finding given the small corpus size (N=6). Company-level risk scores are compared across industries and checked against known positive controls (Patagonia, Lush).

Carbon footprint of model training and inference is tracked using [CodeCarbon](https://codecarbon.io/).

---

## Data

Raw PDFs are not committed to this repository due to copyright. To reproduce, download 2024–2025 sustainability reports directly from each company's investor relations page.

The **extracted and filtered passage sets** (environment-relevant text segments from each report) are released as a public dataset under `data/extracted/` — a secondary contribution of this project intended to lower the barrier for future greenwashing research.

---

## Models and External Resources

- [ClimateBERT distilroberta-base-climate-f](https://huggingface.co/climatebert/distilroberta-base-climate-f)
- [ClimateBERT Environmental Claims Dataset](https://huggingface.co/datasets/climatebert/environmental_claims)
- [Good On You Brand Ratings](https://goodonyou.eco/)
- [EWG Skin Deep Cosmetics Database](https://www.ewg.org/skindeep/)
- [CDP Open Data Portal](https://www.cdp.net/en/data)

---

## Setup

```bash
# Install uv if you don't have it
curl -LsSf https://astral.sh/uv/install.sh | sh

# Clone and enter the repo
git clone https://github.com/yourusername/green-claims-nlp.git
cd green-claims-nlp

# Create virtual environment and install all dependencies
uv sync
```

To run a notebook: 
```bash 
uv run <notebook-name> 
```


---

## Future Work

The most natural extension is a cross-layer analysis comparing greenwashing-risk language in formal sustainability reports versus consumer-facing product copy for the same brands — computing a *greenwashing gap score* that captures whether risk language is amplified when speaking to consumers rather than regulators. This divergence is expected to be most pronounced in the clean beauty context.

---

## Author

Mandy Sun — DSAN 5550, Georgetown University, Spring 2026


Notes on things to add: 
- scale of consumer & retails in climate change contribution 
- right now this only tracks for public-facing reports 

First, when you report brand-level scores, report them as proportions not raw counts — percentage of passages flagged rather than number of passages flagged. That at least controls for report length.
Second, in your Discussion section, add a limitation paragraph explicitly noting that reports follow different frameworks and vary significantly in length and structure, and that your passage-level analysis cannot control for deliberate omissions — a company that simply doesn't discuss a topic won't have passages flagged for that topic.
Third, consider doing a quick descriptive table showing how many passages each brand contributed after filtering — something like H&M: 340 passages, Patagonia: 180 passages, Lush: 90 passages. That gives the reader context for interpreting the scores and shows you're aware of the imbalance.


- can we spot corporate bs

quantified claim vs aspirational claim 


- determine a way to pull the table 

- self defined terms... .