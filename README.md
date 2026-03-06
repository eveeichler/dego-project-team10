# dego-project-team10
DEGO Course Project — Team 10

Team Members:
Eve-Fabiene Eichler (71784)
João Serrano (74929)
Chiara Nathani (71303)
Guilherme Morgado (56857)

# DEGO 2606 — Credit Application Governance Analysis
### Team 10 · Nova SBE MSc Business Analytics

> **Course:** Data Ethics & Governance Operations (DEGO 2606)  
> **Dataset:** NovaCred synthetic credit application pipeline — 502 raw records  
> **Regulatory Scope:** GDPR (EU) 2016/679 · EU AI Act (EU) 2024/1689



## Project Overview

This project conducts an end-to-end governance audit of the credit scoring system **NovaCred**. 

The goal is to tackle data quality, bias, and privacy violations in the credit application pipeline. We start from a raw JSON dataset and work through it in 3 stages:

1. **Data Engineering** — identify, quantify, and remediate all data quality issues using documented governance rationale
2. **Bias Detection** — apply fairness metrics (Disparate Impact ratio, Chi-Square, logistic regression) to detect gender and age-based discrimination
3. **Privacy & Compliance Audit** — map pipeline gaps to specific GDPR and EU AI Act obligations, demonstrate pseudonymisation techniques, and issue governance controls

NovaCred's credit scoring system is classified as **High-Risk AI** under EU AI Act Annex III, Point 5(b). The pipeline is currently **non-compliant with both GDPR and the EU AI Act**.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Repository Structure](#repository-structure)
3. [Team & Roles](#team--roles)
4. [How to Run](#how-to-run)
5. [Notebook 01 — Data Quality](#notebook-01--data-quality)
6. [Notebook 02 — Bias & Fairness Analysis](#notebook-02--bias--fairness-analysis)
7. [Notebook 03 — Privacy & Governance Audit](#notebook-03--privacy--governance-audit)
8. [Key Findings Summary](#key-findings-summary)
9. [Regulatory Exposure](#regulatory-exposure)

---

## Repository Structure

```
dego-project-team10/
│
├── data/
│   ├── raw_credit_applications.json        # Original dataset (502 records, nested JSON)
│   └── clean_credit_applications.csv       # Output of Notebook 01 (485 records)
│
├── notebooks/
│   ├── 01-data-quality.ipynb               # Data Engineering
│   ├── 02-data-analysis.ipynb              # Bias & Fairness Analysis
│   └── 03-privacy-governance.ipynb         # Privacy & Compliance Audit
│
└── README.md
```
---

## Team & Roles

| Role | Responsibilities |
| :--- | :--- |
| **Data Engineer**: Guilherme Morgado | JSON parsing, nested data flattening, issue detection, cleaning pipeline |
| **Data Scientist**: João Serrano | Disparate Impact analysis, statistical testing, logistic regression, proxy screening |
| **Governance Officer**: Eve Eichler| PII classification, pseudonymisation demo, GDPR/AI Act mapping, controls |
| **Product Lead**: Chiara Nathani | Presentation, cross-notebook synthesis, repository coordination |

---

## How to Run

**Requirements:** Python 3.9+, Jupyter Notebook

```bash
# 1. Clone the repository
git clone https://github.com/eveeichler/dego-project-team10.git
cd dego-project-team10

# 2. Install dependencies
pip install pandas numpy scipy statsmodels scikit-learn matplotlib

# 3. Run notebooks in order (each builds on the previous output)
jupyter notebook notebooks/01-data-quality.ipynb
jupyter notebook notebooks/02-data-analysis.ipynb
jupyter notebook notebooks/03-privacy-governance.ipynb
```

> **Note:** Notebook 02 reads `clean_credit_applications.csv` from the `data/` directory — run Notebook 01 first to generate it.

---

## Notebook 01 — Data Quality

**Role:** Data Engineer  
**Input:** `raw_credit_applications.json` (502 records)  
**Output:** `clean_credit_applications.csv` (485 records)

### Data Loading & Handling

The raw dataset consists of deeply nested JSON objects. We used `pd.json_normalize()` to flatten the standard fields, then separately exploded and pivoted the `spending_behavior` array (a list of `{category, amount}` dicts) into one-hot-style spending columns. This yielded a flat dataframe with 35 columns.

### Issues Detected

| # | Issue | Dimension | Count |
| :--- | :--- | :--- | :--- |
| 1 | Duplicate application records | Consistency | 2 exact duplicates (apps `app_042`, `app_001`) |
| 2 | Annual income stored as string | Validity | 8 records |
| 3 | Missing fields | Completeness | See breakdown below |
| 4 | Inconsistent gender coding (`M`/`Male`, `F`/`Female`) | Consistency | 111 records affected |
| 5 | Negative credit history months | Validity | 2 records (`-10`, `-3`) |
| 6 | Negative savings balance | Validity | 1 record (`-5,000`) |
| 7 | Future processing timestamps | Validity | 2 records (2026 and 2027) |
| 8 | Shared SSNs across different applicants | Accuracy | 3 SSNs shared across 6 records |
| 9 | Credit history begins before age 18 | Validity | 2 records |
| 10 | Malformed email addresses | Validity | 4 records |
| 11 | Inconsistent date formats (`DD/MM/YYYY` vs `YYYY-MM-DD` vs `YYYY/MM/DD`) | Consistency | 157 records |

**Missing values breakdown (selected fields):**

| Field | Missing |
| :--- | :--- |
| `processing_timestamp` | 440 / 502 (88%) |
| `loan_purpose` | 452 / 502 (90%) |
| `financials.annual_salary` | 497 / 502 (99%) |
| `decision.interest_rate` | 210 / 502 (42%) |
| `applicant_info.email` | 7 |
| `applicant_info.ssn` | 5 |
| `applicant_info.date_of_birth` | 5 |

### Remediation Rationale

Each remediation step follows explicit governance logic — our primary objective is a demographic bias audit, so data preservation and compliance are balanced:

| Issue | Treatment | Rationale |
| :--- | :--- | :--- |
| Duplicate `_id` records | Drop duplicates, keep first | Prevents double-counting in statistical tests |
| Income stored as string | `pd.to_numeric(errors='coerce')` | Restore numeric type; merge with `annual_salary` field where missing |
| Gender inconsistency | Map `M → Male`, `F → Female` | Standardise categories for groupby and fairness calculations |
| Negative credit history | `abs()` | Logically impossible value treated as a data entry sign typo |
| Negative savings balance | `.clip(lower=0)` | A savings account cannot hold a negative balance; floor to zero |
| Future timestamps | Set to `NaT` | Invalid timestamps isolated; demographic data preserved |
| Shared SSNs | Drop **all** copies | Identity fraud / pipeline corruption risk; keeping any instance is a compliance liability |
| Malformed emails | Drop records | Non-verifiable contact data treated as low-quality / potentially fraudulent |
| Missing gender or DOB | Drop records | Without protected attributes, records are un-auditable for bias analysis |
| Inconsistent date formats | `pd.to_datetime(format='mixed')` | Pandas `mixed` mode resolves both `-` and `/` separators |

**Final clean dataset: 485 records, 33 columns.**

### Data Quality Dimension Mapping

| Issue | GDPR Dimension |
| :--- | :--- |
| Missing fields | Completeness |
| Duplicate records, gender coding inconsistency, mixed date formats | Consistency |
| String-typed income, impossible numerical values, malformed emails | Validity |
| Shared SSNs across distinct applicants | Accuracy |

---

## Notebook 02 — Bias & Fairness Analysis

**Role:** Data Scientist  
**Input:** `clean_credit_applications.csv` (485 records)

### Gender Disparate Impact

**Approval rates:**
- Male: **65.9%** (out of 248 male applicants)
- Female: **50.6%** (out of 241 female applicants)
- Difference: **15.3 percentage points**

**Disparate Impact Ratio** (Female / Male):

$$DI = \frac{0.506}{0.659} = \mathbf{0.77}$$

DI < 0.80 — adverse impact is presumed under the EEOC four-fifths rule.

**Proportions Z-test:** p = 0.0007 (two-sided) → the gender approval gap is **statistically significant** at the 5% level.

**Logistic regression (controlling for financial variables):** `gender_Male` remains a significant predictor of approval (coef = 0.624, p = 0.043), confirming the disparity persists even after controlling for income, credit history, debt-to-income ratio, and savings balance.

### Age-Based Disparity

**Approval by age group:**

| Age Group | Approved | Denied | Approval Rate |
| :--- | :--- | :--- | :--- |
| Gen Z (18–25) | 13 | 27 | **32.5%** |
| Millennials (26–40) | 139 | 102 | **57.7%** |
| Gen X (41–60) | 115 | 64 | **64.2%** |
| Seniors (60+) | 20 | 9 | **69.0%** |

**Chi-Square test:** p = 0.0019 → age group and approval outcome are **not independent** (significant at 5% level).

**Logistic regression (age as continuous variable):** The odds ratio for age was 1.025 (p = 0.005) — small on its own, but it compounds over time. Over 30 years that adds up to an odds ratio of 2.09, which is a significant structural disadvantage for younger applicants.

**Linear trend:** The trendline across approval rates by age has a clear upward slope — older applicants are consistently more likely to be approved.

### Proxy Variable Screening

Removing protected attributes from a model does not automatically remove the bias — other features can reproduce the same discriminatory outcomes if they are correlated with gender or age. We screened the financial and geographic features to check for this.

**Proxy for Gender:**

| Variable | Correlation with Gender |
| :--- | :--- |
| ZIP code (binary: NYC prefix 100 vs. other) | **r = 0.783** |
| Annual income | r = -0.050 |
| Credit history months | r = -0.024 |

ZIP code is a near-perfect proxy for gender in this dataset. When gender is removed from logistic regression, ZIP code becomes strongly significant (coef = 0.585, p = 0.002) — confirming it absorbs the gender signal.

**Proxy for Age:**

| Variable | Correlation with Age |
| :--- | :--- |
| Credit history months | **r = 0.659** |
| Annual income | r = 0.386 |
| Savings balance | r = 0.272 |

Credit history length is strongly correlated with age — younger applicants are structurally disadvantaged by this feature regardless of their actual creditworthiness.

### Interaction Effects (Intersectionality)

A logistic regression including an `age × gender` interaction term found the interaction to be non-significant (p = 0.802). Gender and age effects appear **independent** — there is no strong evidence of compounded intersectional bias in this dataset.

### Main Conclusions

- **Gender bias confirmed:** DI = 0.77, statistically significant, persists after financial controls.
- **Age-based disparity confirmed:** Gen Z approval rate (32.5%) is 25.2 pp below Millennials; Chi-Square p = 0.002.
- **Proxy discrimination identified:** ZIP code proxies for gender; credit history proxies for age — both must be excluded from any model features.
- **No intersectional compounding** detected between age and gender.

---

## Notebook 03 — Privacy & Governance Audit

**Role:** Governance Officer  
**Input:** `clean_credit_applications.csv` (485 records)

### Part 1 — PII Identification & Classification

Seven fields in the dataset carry personal data under GDPR Art. 4(1):

| Field | Type | Sensitivity |
| :--- | :--- | :--- |
| `applicant_info.ssn` | Direct Identifier | **CRITICAL** |
| `applicant_info.full_name` | Direct Identifier | HIGH |
| `applicant_info.email` | Direct Identifier | HIGH |
| `applicant_info.ip_address` | Direct Identifier | HIGH |
| `applicant_info.date_of_birth` | Quasi-Identifier | MEDIUM |
| `applicant_info.zip_code` | Quasi-Identifier + Proxy | MEDIUM |
| `applicant_info.gender` | Quasi-Identifier + Protected | MEDIUM |

**Re-identification test (Sweeney methodology):** Using ZIP code, gender, and date of birth in combination, every single applicant in the clean dataset is uniquely identifiable. **Re-identification rate = 100%.**  
→ Quasi-identifier generalisation is mandatory before any analytical export.

### Part 2 — Pseudonymisation Demonstration

Three techniques are compared and applied:

| Technique | Reversible? | Applied To |
| :--- | :--- | :--- |
| Plain SHA-256 | No | — (not used; deterministic = vulnerable to pre-computation) |
| **Salted SHA-256** | No | SSN, email, IP address |
| **Tokenization** | Yes (via lookup table) | Full name (`APPLICANT_001`, etc.) |

The salted hash uses a randomly generated salt (`os.urandom(16)`) stored separately — making rainbow table attacks infeasible even when the input space (e.g., all valid SSN formats) is known.

**k-Anonymity verification** after generalisation (age bands + ZIP prefix truncation):

| Metric | Value |
| :--- | :--- |
| Minimum k | 1 |
| Maximum k | 118 |
| Average k | 23.1 |
| Groups below k = 5 (at-risk) | 9 of 21 |

Generalisation raised average k from 1 to 23.1, but 9 small demographic cohorts (mostly in ZIP prefix 300) remain below the internal governance threshold of k ≥ 5. Differential Privacy (Laplace mechanism for continuous variables, coin-flip technique for categorical) is required before any external release of these records.

### Part 3 — Regulatory Compliance Mapping

#### GDPR Violations

| Finding | Article | Severity |
| :--- | :--- | :--- |
| Plaintext SSN, name, email, IP in analytical CSV | Art. 5(1)(f) — Integrity & Confidentiality / Art. 5(1)(c) — Data Minimisation | CRITICAL |
| 440/502 records (88%) missing `processing_timestamp` — retention periods unenforceable | Art. 5(1)(e) — Storage Limitation | HIGH |
| Gambling, Adult Entertainment spending collected without documented lawful basis | Art. 6, Art. 5(1)(b) — Purpose Limitation, Art. 5(1)(c) — Data Minimisation | HIGH |
| 157 inconsistent date formats; shared SSNs; impossible values | Art. 5(1)(d) — Accuracy | MEDIUM |

#### EU AI Act Classification

Credit scoring is **explicitly enumerated** as High-Risk AI under EU AI Act Annex III, Point 5(b). This classification triggers mandatory obligations — all three are currently unmet:

| Article | Obligation | Status |
| :--- | :--- | :--- |
| Art. 10 | Training data examined and corrected for bias | **NON-COMPLIANT** — Gender DI = 0.77; Gen Z approval gap = 25.2 pp |
| Art. 13 | Automated decisions traceable to a specific model version | **NON-COMPLIANT** — No `model_version` field |
| Art. 14 | Human oversight and override mechanism | **NON-COMPLIANT** — No `human_review_flag`; most common rejection reason is `algorithm_risk_score` |

#### Mandatory Governance Fields Audit

All six fields required to demonstrate lawful processing under GDPR Art. 5(2) are **absent** from the dataset:

| Field | Regulatory Basis | Status |
| :--- | :--- | :--- |
| `consent_timestamp` | GDPR Art. 6(1)(a), Art. 7 | MISSING |
| `processing_purpose` | GDPR Art. 5(1)(b) | MISSING |
| `retention_until` | GDPR Art. 5(1)(e) | MISSING |
| `data_source` | GDPR Art. 14 | MISSING |
| `human_review_flag` | GDPR Art. 22 | MISSING |
| `model_version` | EU AI Act Art. 13 | MISSING |

### Part 4 — Governance Controls

**Control 1 — Privacy by Design Pipeline Restructuring** *(Immediate)*  
Pseudonymisation needs to happen at ingestion — before data is written anywhere. The pipeline should produce two separately managed outputs: a secure PII store (compliance team only) and a clean analytical dataset (data science team). Plaintext identifiers should never make it through to the analytical layer.

**Control 2 — Mandatory Governance Fields** *(Before next data collection cycle)*  
The six fields identified above need to be added to the schema with ingestion-level validation. Any record missing a mandatory field should be quarantined before it enters the decision pipeline — their absence makes lawful processing impossible to demonstrate.

**Control 3 — Bias Monitoring & Feature Governance** *(Before any retraining)*  
ZIP code and credit history months need to come out of the model feature set entirely. Beyond that, a pre-deployment bias gate should block any model where DI drops below 0.80 for any protected attribute. A model card documenting training data, DI ratios, and excluded features should accompany every deployment.

**Control 4 — Three-Tier Human Oversight** *(Before further production deployment)*  
The current binary auto-reject setup needs to be replaced with a three-tier review architecture:
- **Tier 1 (Green):** Automated approval for applications that clearly meet all thresholds
- **Tier 2 (Yellow):** Mandatory human review when a proxy variable triggers or DI falls below 0.80 for the applicant's group
- **Tier 3 (Red):** A clear pathway for applicants to contest an automated rejection — required under GDPR Art. 22

---

## Key Findings Summary

| # | Finding | Regulatory Impact |
| :--- | :--- | :--- |
| 1 | All 4 direct identifiers in plaintext in a public repository | GDPR Art. 5(1)(f) — **CRITICAL** |
| 2 | Sensitive lifestyle spending data collected without documented lawful basis | GDPR Art. 5(1)(b)(c) — HIGH |
| 3 | 88% of records missing `processing_timestamp` | GDPR Art. 5(1)(e) — HIGH |
| 4 | Gender DI ratio = 0.77; Gen Z approval rate 25.2 pp below Millennials | EU AI Act Art. 10 — **CRITICAL** |
| 5 | ZIP code is a structural proxy for gender; credit history proxies for age | EU AI Act Art. 10 — HIGH |
| 6 | No human oversight mechanism; fully automated rejections in operation | GDPR Art. 22 / EU AI Act Art. 14 — HIGH |
| 7 | All 6 mandatory governance fields absent | GDPR Art. 5(2) / EU AI Act Art. 13 — **CRITICAL** |
| 8 | Shared SSNs, malformed emails, impossible values in raw pipeline | GDPR Art. 5(1)(d) — MEDIUM |

---

## Regulatory Exposure

| Regulation | Provision | Maximum Fine |
| :--- | :--- | :--- |
| GDPR | Art. 83(5) — violations of Arts. 5, 25 | €20M or 4% of global annual turnover |
| EU AI Act | Art. 99(4) — High-Risk system non-compliance | €15M or 3% of global annual turnover |

**Immediate actions required (in priority order):**
1. Remove plaintext CSV from Git repository history
2. Implement pseudonymisation at pipeline ingestion
3. Drop `spending_Adult Entertainment`, `spending_Gambling`, `spending_Alcohol` fields
4. Exclude ZIP code and credit history from all model features
5. Add 6 mandatory governance fields with ingestion-level validation
6. Implement Three-Tier Human Oversight before any further production deployment


