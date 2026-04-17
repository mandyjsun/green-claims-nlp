# Annotation Guide — green-claims-nlp (v2.0 - 2026 Audit Edition)

---

## The Core Task
Evaluate corporate sustainability claims against the **Ground Truth** provided by third-party auditors (MSCI, Sustainalytics, FTI, CDP, Stand.earth). Your goal is to identify the mathematical and narrative gap between a brand's sentiment and its audited performance.

## Binary Labels

| Label | Name | Criteria |
|---|---|---|
| **0** | Substantive | Specific measurable goals with timelines; third-party verified (SBTi, GRI, ZDHC); concrete emissions data; named certifications (GOTS, FSC). **Must align with Auditor Grade (e.g., MSCI AAA/AA or CDP A).** |
| **1** | Greenwashing-risk | Vague commitments; "clean/natural" beauty terms; self-defined standards (Inditex "Life," H&M "Conscious") without third-party proof. **Triggers automatically if claim sentiment is high but Sustainalytics Risk is "Severe/High".** |
| **2** | Ambiguous | Genuine uncertainty — cannot reliably assign 0 or 1. (e.g., a brand is a B-Corp but currently under investigation or score is "Under Review"). |

---

## Divergence Triggers (The "Truth" Check)

Before labeling, cross-reference the claim with the **Metadata Master File**:

1. **The Disclosure Gap:** If a brand makes a "Transparency" claim but their **FTI score is < 30%** (e.g., Shein, Ralph Lauren) → **Label 1 (Sin of No Proof)**.
2. **The Carbon Gap:** If a brand claims "Climate Leadership" but their **CDP score is D or F** (e.g., L'Occitane's carbon math) → **Label 1 (Sin of Fibbing)**.
3. **The Chemical Gap:** If a beauty brand uses terms like "non-toxic" or "safe" but carries an **EWG Hazard Score > 7** → **Label 1 (Sin of Vagueness)**.
4. **The B-Corp Shield:** Claims from **B-Corps (Lush, Patagonia, Reformation)** are given a "Trust Baseline." Label 0 unless the claim is explicitly future-aspirational without a date.

---

## Edge Cases & Sector Rules

| Pattern | Rule |
|---|---|
| **"Clean" Beauty** | Always **Label 1** unless a specific chemical exclusion list (MRSL) or EWG Verification is cited. |
| **"Responsibly Sourced"** | Always **Label 1** unless it names the certification (e.g., "100% GRS Recycled Polyester"). |
| **"Net Zero by 2040/2050"** | **Label 1** if no interim milestones (2025/2030) are mentioned in the same paragraph. |
| **Circular Retail** | (e.g., Sephora's "Pact" recycling) **Label 0** if they provide the tonnage/volume of waste diverted; **Label 1** if they just mention the program exists. |

---

## Worked Examples

### Label 0 — Substantive (High Specificity)
* *"As a B-Corp, we achieved a 95.0 impact score, specifically reducing Scope 3 emissions by 12% in 2025."* (**Lush Example**)
* *"74% of our suppliers have achieved ZDHC Level 3 chemical compliance, up from 60% in 2024."* (**Kering Example**)

### Label 1 — Greenwashing-risk (Divergence/Vague)
* *"We are committed to a cleaner, more beautiful world for everyone."* (**Sephora Example - Pure fluff**)
* *"Our ambition is to use 100% preferred fibers by 2030."* (**Zara Example - "Preferred" is self-defined**)
* *"We prioritize the planet in everything we do."* (**Shein Example - Triggers GSI spike due to F score**)

---

## Metadata Fields to Note
- `auditor_conflict`: Mark as `True` if the claim sentiment is Positive but the Auditor Grade is a Laggard (C/D/F).
- `notes`: Briefly mention if you used a specific site (like **EWG** or **Impact-Reporting**) to debunk the claim.
