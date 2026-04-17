# Research Foundation: The Seven Sins & The GSI Framework

This document captures domain knowledge that informs design choices throughout the project — from selecting datasets to building the annotation framework to interpreting results. 

There are four key areas where expert domain knowledge shapes the work:

1. Existing datasets and evaluation metrics — understanding what corpora already exist, what's been measured, and whether existing scoring approaches translate to greenwashing classification specifically.
2. The brand landscape — understanding which companies operate in this space, what sectors they belong to, and what context shapes how their claims should be read. Whether the brand has a sustainability report. 
3. Annotation framework design — using domain knowledge to rank and label claims at the sentence level, identify high-signal material, and build out a consistent labeling intention across annotators.
4. Auditing evaluation results — using subject matter understanding to check model outputs and whether they align with overall climate sentiments. 

---

## Why Greenwashing Is Hard to Define

One of the core challenges of this project is that greenwashing doesn't have a universally agreed-upon definition. Across regulatory bodies, academic literature, and industry standards, the term gets applied to everything from outright fabrication to technically-true-but-misleading framing. This makes classification inherently fuzzy — a claim can be deceptive without being false.

There's also a self-undermining quality to the term itself: "greenwashing" has increasingly become a marketing and PR concept, which means brands are now sophisticated enough to anticipate and pre-empt the accusation. Claims are engineered to sound accountable while remaining vague. This raises the difficulty of the task considerably, because the most egregious greenwashing today often doesn't look like greenwashing on the surface.

For instance, H&M 

< image utilizing terms for democratizing fashion >

--- 

### The Seven Sins Framework 

The project uses the Seven Sins of Greenwashing taxonomy developed by TerraChoice (now UL Solutions), derived from a forensic audit of thousands of consumer products. It's one of the few industry-standard frameworks that operationalizes greenwashing into discrete, classifiable categories, which makes it a natural fit for an annotation scheme.
The seven sins are:

1. The Sin of the Hidden Trade-off
Definition: Claiming a product is "green" based on a single environmental attribute (e.g., recycled paper packaging) while ignoring more significant impacts (e.g., the high carbon footprint of overseas shipping).

- NLP Marker: High-frequency "sustainability" keywords concentrated in non-material sections (packaging, office waste) while material sections (manufacturing, Scope 3) remain vague.

2. The Sin of No Proof
Definition: An environmental claim that cannot be substantiated by easily accessible supporting information or by a reliable third-party certification.

- NLP Marker: Percentage-based claims (e.g., "50% more sustainable") lacking a specific link to a verified standard like GRS, FSC, or Oeko-Tex.

3. The Sin of Vagueness
Definition: A claim that is so poorly defined or broad that its real meaning is likely to be misunderstood by the consumer.

- NLP Marker: Overuse of unregulated adjectives: clean, eco-friendly, natural, ethical, conscious.

4. The Sin of Worshipping False Labels
Definition: Creating fake certifications or "eco-stamps" that mimic the look of a third-party audit to give a false sense of security.

- NLP Marker: Proprietary internal program names (e.g., "H&M Conscious Choice") placed in positions typically reserved for independent seals like the EU Ecolabel.

5. The Sin of Irrelevance
Definition: Making a claim that may be truthful but is unimportant or unhelpful for consumers seeking environmentally preferable products.

- NLP Marker: Highlighting compliance with laws passed decades ago, such as "CFC-free" or "Lead-free," where these substances are already banned.

6. The Sin of Lesser of Two Evils
Definition: Claims that may be true within the product category but risk distracting consumers from the greater environmental impacts of the category as a whole.

- NLP Marker: Promoting "Eco-friendly" versions of inherently high-impact products (e.g., "Sustainably sourced leather" or "Carbon-neutral air travel").

7. The Sin of Fibbing
Definition: Environmental claims that are simply false.

- NLP Marker: Blatant contradictions between PDF text and external "Ground Truth." For example, a company claiming "SBTi-validated Net Zero" when the SBTi Target Dashboard shows their commitment was removed.

--- 

## Domain Notes: The Sustainability Auditing Framework
This section establishes the expert "Ground Truth" used to validate the NLP-derived scores. In 2026, auditing has shifted from checking "if" a company reports to checking the "integrity" of the data through a multi-lens auditor approach.

### 1. The Greenwashing Severity Index (GSI) Logic

The primary metric for this project is the GSI.

- **The Discrepancy Theory:** GSI measures the statistical distance between Corporate Self-Representation (High-sentiment, vague text) and External Public Narratives (Hard metrics and auditor grades).

- **Model Application:** If Track C (ClimateBERT) shows "Highly Positive Sentiment" but Sustainalytics shows "High Risk," the GSI value increases, flagging the passage as a high-probability Sin of Fibbing.

### 2. The Auditor Comparison Matrix

To avoid "Black Box" bias, we use different auditors to infer specific corporate behaviors:

| Auditor | Primary Inference | NLP Auditing Role |
|---|---|---|
| **B Corp** | Operational Integrity | Acts as a binary filter. B Corp text is treated as the "Gold Standard" for **Substantive (Label 0)** claims. |
| **FTI (2025)** | Disclosure Transparency | High FTI scores validate that the brand isn't hiding its supply chain. |
| **CDP (2025/26)** | Carbon Math Rigor | If a brand claims "Net Zero" but lacks a CDP 'A' grade, the claim is flagged as unverified. |
| **Stand.earth (2025)** | Fossil Fuel Progress | It detects if a brand promotes "recycled plastic" while still using coal-powered factories. |
| **MSCI / Sustainalytics** | Financial ESG Risk |  A "safe investment" grade (AAA) may still hide poor environmental performance. However, these metrics are used to inform investors on risks. |

### 3. Hard Metric "Triggers"
The annotation framework prioritizes sentences containing Hard Units. A claim is only considered Substantive if it includes:

- tCO2e: Absolute carbon reduction.

- % PCR: Post-Consumer Recycled content (vs. vague "recycled" claims).

- ZDHC / MRSL: Specific chemical safety standards.

- SBTi Validation Date: To ensure the target isn't a "Sin of Irrelevance."


--- 

### Brand Landscape & Control Groups

The corpus is structured to teach the model the difference between marketing fluff and technical disclosure, using a three-tier label design.

| Label | Fashion | Beauty / Cosmetics |
|---|---|---|
| **Label 0 — Substantive** | Patagonia, Eileen Fisher, Puma, Kering | Lush, L'Occitane |
| **Label 1 — Greenwashing Risk** | H&M, Zara, Shein, Ralph Lauren | Unilever, L'Oréal |
| **Label 2 — Edge Cases** | Everlane, Reformation, Lululemon | Sephora |

**Label 0 selection criteria:** B Corp status (score above 80), verified CDP A-List status, 
or third-party assured technical annexes signed by auditors (PwC/Deloitte Limited Assurance 
Reports).

**Label 1 selection criteria:** High volume of self-defined standards — "Conscious Choice," 
"Green Sciences" — that lack linkage to ISO, GRI, or ESRS standards.

**Label 2 selection criteria:** Brands that market sustainability as a core identity but hold 
no recognized third-party certifications, or whose claims rely solely on retailer-defined 
standards such as "Clean at Sephora.


### Other Cool Sites I Found 

As I was exploring auditing sites, I found a few more cool websites to look at that are not included above. Below I linked a few and wrote a little note on *why.* 

📊 [The Impact Reporting Archive](https://impact-reporting.com/) 

A global archive for business impact report sharing, research, and inspiration. 

- **Why it’s cool:** I like that it promotes sustainability reporting by providing resources through blogs with tips and tricks. You can be a small business and submit your own report! You can also view other small business reports. 

🏷️ [Good On You](https://directory.goodonyou.eco/) 

Discover thousands of brands' sustainability reports. 

* **Why it's cool:** The site that exposed me to the concept greenwashing in high school. It remains the most user-friendly way to see a synthesized view of the fashion industry. It includes popular brands with youth, including online brands like Cider that are harder to find on other rating sites. Ratings are human-curated and already synthesize labor, environment, and animal welfare — so it maps naturally onto greenwashing detection.

🧪 [EWG Skin Deep](https://www.ewg.org/skindeep/) 

Massive database of over 100,000 products rated on ingredient safety

- **Why it's cool:** You can search a specific product and see a hazard score from 1 to 10 based on scientific toxicity data.