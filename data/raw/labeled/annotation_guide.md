# Annotation Guide — green-claims-nlp

**Version:** 1.0  
**Author:** [your name]  
**Date:** [date]

---

## Binary labels

| Label | Name | Criteria |
|---|---|---|
| **0** | Substantive | Specific measurable goals with timelines; third-party verified claims (SBTi, GRI, Deloitte, ZDHC); concrete emissions data with methodology disclosed; named certification standards (not proprietary) |
| **1** | Greenwashing-risk | Vague commitments without quantified targets; unverifiable claims; self-defined proprietary standards without third-party verification; unregulated beauty terms as sustainability claims; relative comparisons without benchmarks; future-tense aspirations without timelines |
| **2** | Ambiguous | Genuine uncertainty — cannot reliably assign 0 or 1 (excluded from model training; tracked separately) |

---

## Edge cases

| Pattern | Rule |
|---|---|
| Self-defined standards ("lower-impact", "sustainably sourced", "responsibly made") | Label 1 unless linked to a named third-party standard |
| Relative claims without external benchmarks ("reduced by X% vs our own 2019 baseline") | Label 0 if methodology disclosed; label 1 if baseline is undefined |
| Future-tense commitments | Label 0 if quantified with timeline; label 1 if aspirational only |
| Own-operations vs supply-chain claims | Same criteria apply — note distinction in the `scope` metadata field |
| Proprietary program membership (Better Cotton Initiative, Fashion Pact) | Label 1 unless specific outcome metrics reported |
| Unregulated beauty terms (*clean*, *natural*, *non-toxic*, *reef-safe*) | Label 1 for beauty industry; note in `notes` field |

---

## Worked examples

### Label 0 — Substantive
1. "Scope 1 and 2 emissions reduced 88% vs 2018 baseline, validated by SBTi."
2. "97.1% MRSL compliance for chemical inputs, measured via ZDHC Gateway."
3. "100% of cotton sourced via Better Cotton Initiative, with BCI-verified outcome metrics for 2024."
4. "GRI 305-1 disclosure: 12,450 tCO2e scope 1 emissions, third-party assured by Deloitte."
5. "SBTi-aligned 1.5°C target: absolute scope 1+2 reduction of 50% by 2030 vs 2019 base year."

### Label 1 — Greenwashing-risk
1. "We are committed to decarbonising our supply chain." (no timeline, no metric)
2. "Made with lower-impact fibres." (Inditex self-defined, no third-party standard)
3. "Sustainably sourced materials." (H&M self-defined standard)
4. "The positive power of fashion." (aspirational brand language)
5. "Our ambition is to become climate positive by 2040." (no interim milestones or current baselines)

### Label 2 — Ambiguous
1. [Add your own examples with one-sentence explanation of uncertainty]

---

## Notes
- Record uncertainty using `notes` field; write one sentence explaining any label 2 assignment
- Do not retroactively change labels after test set is locked
