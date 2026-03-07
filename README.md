# dego-project-team10
DEGO Course Project — Team 10

>**Team Members**:
Eve-Fabiene Eichler (71784)
João Serrano (74929)
Chiara Nathani (71303)
Guilherme Morgado (56857)

> **Dataset:** NovaCred synthetic credit application pipeline — 502 raw records  

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

## Project Overview

This project audits a synthetic credit application dataset from **NovaCred**, a fictional fintech whose ML-driven loan approval pipeline was flagged for regulatory review. The analysis covers three interconnected dimensions: data quality, algorithmic bias, and regulatory compliance.

The work is structured across three notebooks, each building on the preceding output:

1. **Data Engineering** — systematic detection and remediation of quality issues in the raw JSON dataset, with a documented governance rationale for each intervention
2. **Bias Detection** — statistical examination of whether gender and age substantively affect loan approval outcomes, including proxy variable screening to identify indirect discrimination
3. **Privacy & Compliance Audit** — mapping of identified issues to specific GDPR articles and EU AI Act obligations, pseudonymisation demonstration, and four concrete governance controls

The central finding: NovaCred's credit scoring system meets the definition of **High-Risk AI** under EU AI Act Annex III, Point 5(b), and is non-compliant with both GDPR and the EU AI Act across multiple provisions — including biased approval outcomes, absent audit trails, and plaintext personal data present in an analytical CSV.

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
| **Governance Officer**: Eve Eichler | PII classification, pseudonymisation demo, GDPR/AI Act mapping, controls |
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

The raw dataset was provided as deeply nested JSON, precluding direct CSV ingestion. `pd.json_normalize()` was applied to flatten standard fields, while the `spending_behavior` column — structured as an array of `{category, amount}` objects per applicant — required a separate pipeline: the array was exploded into individual rows, normalised, then pivoted to restore a single-row-per-applicant structure with one column per spending category. Final schema: 35 columns.

### Issues Detected

| # | Issue | Dimension | Count |
| :--- | :--- | :--- | :--- |
| 1 | Duplicate application records | Consistency | 2 exact duplicates (`app_042`, `app_001`) |
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

Given that the primary objective is a bias audit, data preservation was treated as a first-order concern — unnecessary record removal reduces statistical power for the fairness analysis. Each remediation applied the minimum intervention sufficient to resolve the specific issue, with an explicit governance rationale:

| Issue | Treatment | Rationale |
| :--- | :--- | :--- |
| Duplicate `_id` records | Drop duplicates, keep first | Prevents double-counting in statistical tests |
| Income stored as string | `pd.to_numeric(errors='coerce')` | Restore numeric type; merge with `annual_salary` field where missing |
| Gender inconsistency | Map `M → Male`, `F → Female` | Standardise categories for groupby and fairness calculations |
| Negative credit history | `abs()` | Most likely a sign typo — the magnitude is still valid |
| Negative savings balance | `.clip(lower=0)` | A savings balance cannot be negative; floored to zero while preserving the remainder of the record |
| Future timestamps | Set to `NaT` | The timestamp is wrong but the applicant's financial profile is fine |
| Shared SSNs | Drop **all** copies | Can't know which record is real — keeping any of them is a compliance risk |
| Malformed emails | Drop records | Can't verify or legally contact this person |
| Missing gender or DOB | Drop records | Without protected attributes the record is un-auditable for bias analysis |
| Inconsistent date formats | `pd.to_datetime(format='mixed')` | Pandas `mixed` mode handles both `-` and `/` separators in one pass |

**Final clean dataset: 485 records, 33 columns.**

### Data Quality Dimension Mapping

| Issue | Data Quality Dimension |
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
- Male: **66.8%** (out of 241 applicants, 161 approved)
- Female: **50.8%** (out of 244 applicants, 124 approved)
- Gap: **16.0 percentage points**

The key metric here is the **Disparate Impact ratio**, which compares the approval rate of the disadvantaged group against the privileged one:

> DI = Female approval rate / Male approval rate = 0.508 / 0.668 = **0.76**

Under the EEOC four-fifths rule, a DI below 0.80 constitutes evidence of adverse impact. At 0.76, NovaCred's system fails this threshold by a meaningful margin.

Statistical validation proceeded in two stages. A proportions Z-test (p = 0.000349) confirmed the disparity is not attributable to sampling variation. A logistic regression controlling for annual income, credit history, debt-to-income ratio, and savings balance further confirmed that the gender coefficient remains substantial (coef = 0.697) — the approval gap is not explained by differential financial profiles. Gender independently increases the log-odds of approval.

### Age-Based Disparity

| Age Group | Approved | Denied | Approval Rate |
| :--- | :--- | :--- | :--- |
| Gen Z (18–25) | 13 | 27 | **32.5%** |
| Millennials (26–40) | 138 | 102 | **57.5%** |
| Gen X (41–60) | 114 | 62 | **64.8%** |
| Seniors (60+) | 20 | 9 | **69.0%** |

The ~25 percentage point gap between Gen Z and all other cohorts represents the most pronounced disparity in the dataset. A Chi-Square test confirmed that age group and approval outcome are statistically dependent (p = 0.0015). Logistic regression on continuous age yields a coefficient of 0.025 (OR = 1.026 per year), compounding to approximately **2.1× higher approval odds over a 30-year span** (OR = 2.13).

### Proxy Variables

Eliminating protected attributes from a model's feature set does not eliminate bias if correlated variables remain available to reproduce the same discrimination indirectly. All financial and geographic features were screened for proxy potential:

**Proxy for Gender:**

| Variable | Correlation with Gender |
| :--- | :--- |
| ZIP code (binary: NYC prefix 100 vs. other) | **r = 0.786** |
| Debt-to-income ratio | r = 0.023 |
| Annual income | r = −0.050 |

ZIP code functions as a near-perfect proxy for gender in this dataset (r = 0.786). When gender is excluded from the logistic regression, ZIP code absorbs the signal and becomes the dominant predictor — demonstrating that feature removal alone is an insufficient remediation strategy.

**Proxy for Age:**

| Variable | Correlation with Age |
| :--- | :--- |
| Credit history months | **r = 0.657** |
| Annual income | r = 0.389 |
| Savings balance | r = 0.276 |

Credit history length is structurally correlated with age as a function of elapsed time rather than financial behaviour. A younger applicant with 12 months of credit history is not inherently less creditworthy than an older applicant with 15 years — the difference reflects opportunity, not risk profile.

### Interaction Effects (Intersectionality)

We also tested whether being young and female compounds the disadvantage beyond either effect alone. The `age × gender` interaction term was non-significant (p = 0.802), so the two biases appear to operate independently in this dataset.

### Main Conclusions

- **Gender bias confirmed:** DI = 0.76, statistically significant (p = 0.000349), persists after controlling for all financial variables.
- **Age-based disparity confirmed:** Gen Z approval rate (32.5%) is ~25 pp below every other cohort; Chi-Square p = 0.0015.
- **Proxy discrimination identified:** ZIP code proxies for gender (r = 0.786); credit history proxies for age (r = 0.657) — both must be excluded from any model feature set.
- **No intersectional compounding** detected between age and gender effects.

---

## Notebook 03 — Privacy & Governance Audit

**Role:** Governance Officer  
**Input:** `clean_credit_applications.csv` (485 records)

### Part 1 — PII Identification & Classification

Under GDPR Art. 4(1), personal data is anything that can be used to identify a real person — directly or indirectly. We split the dataset's fields into two tiers:

| Field | Type | Sensitivity |
| :--- | :--- | :--- |
| `applicant_info.ssn` | Direct Identifier | **CRITICAL** |
| `applicant_info.full_name` | Direct Identifier | HIGH |
| `applicant_info.email` | Direct Identifier | HIGH |
| `applicant_info.ip_address` | Direct Identifier | HIGH |
| `applicant_info.date_of_birth` | Quasi-Identifier | MEDIUM |
| `applicant_info.zip_code` | Quasi-Identifier + Proxy | MEDIUM |
| `applicant_info.gender` | Quasi-Identifier + Protected | MEDIUM |

A re-identification test was conducted following the Sweeney (2000) methodology, combining ZIP code, gender, and date of birth as quasi-identifiers. **479 of 485 records (98.8%) are uniquely identifiable** using only those three fields, without recourse to SSN or full name. This rate exceeds Sweeney's 87% benchmark for the general US population and establishes that the dataset does not meet anonymisation standards — full GDPR obligations apply.

### Part 2 — Pseudonymisation Demonstration

| Technique | Reversible? | Applied To |
| :--- | :--- | :--- |
| Plain SHA-256 | No | Not used — deterministic, vulnerable to pre-computation attacks |
| **Salted SHA-256** | No | SSN, email, IP address |
| **Tokenization** | Yes (via lookup table) | Full name (`APPLICANT_001`, etc.) |

The salt is generated using `os.urandom(16)` and stored separately — this breaks the deterministic link between input and hash, making rainbow table attacks infeasible. Tokenization was chosen for names specifically because a compliance officer may need to recover the original value to respond to a GDPR Art. 17 erasure request.

**k-Anonymity after generalising DOB to age bands and truncating ZIP to 3-digit prefix:**

| Metric | Value |
| :--- | :--- |
| Total k-groups | 31 |
| Minimum k | 1 |
| Maximum k | 75 |
| Average k | 15.6 |
| Groups below k = 5 (at-risk) | 18 of 31 |

Average k improved significantly after generalisation, but 18 of 31 cohorts still fall below k ≥ 5 — particularly in lower-density ZIP prefixes. Those groups would need **Differential Privacy** (Laplace mechanism) before the data could safely be shared externally.

### Part 3 — Regulatory Compliance Mapping

#### GDPR Violations

| Finding | Article | Severity |
| :--- | :--- | :--- |
| Plaintext SSN, name, email, IP in analytical CSV | Art. 5(1)(f) — Integrity & Confidentiality / Art. 5(1)(c) — Data Minimisation | CRITICAL |
| 440/502 records (88%) missing `processing_timestamp` | Art. 5(1)(e) — Storage Limitation | HIGH |
| Gambling & Adult Entertainment spending collected without documented lawful basis | Art. 6, Art. 5(1)(b) — Purpose Limitation | HIGH |
| 157 inconsistent date formats; shared SSNs; impossible values | Art. 5(1)(d) — Accuracy | MEDIUM |

#### EU AI Act Classification

Credit scoring is explicitly listed as High-Risk AI under EU AI Act **Annex III, Point 5(b)** — this applies automatically, not by judgment call. All three mandatory obligations are currently unmet:

| Article | Obligation | Status |
| :--- | :--- | :--- |
| Art. 10 | Training data examined and corrected for bias | **NON-COMPLIANT** — Gender DI = 0.76; Gen Z gap ≈ 25 pp |
| Art. 13 | Automated decisions traceable to a specific model version | **NON-COMPLIANT** — No `model_version` field anywhere |
| Art. 14 | Human oversight and override mechanism | **NON-COMPLIANT** — No `human_review_flag`; 161 rejections driven by `algorithm_risk_score` alone |

#### Mandatory Governance Fields Audit

GDPR Art. 5(2) — the Accountability Principle — means it's not enough to be compliant, you have to be able to prove it. All six fields that would allow NovaCred to do that are absent:

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
Pseudonymisation needs to happen at ingestion — before data is written anywhere. Split the pipeline output into a secure PII store (compliance team only) and a clean analytical dataset (data science team). No plaintext identifiers should ever reach the analytical layer.

**Control 2 — Mandatory Governance Fields** *(Before next data collection cycle)*  
Add the six fields above to the data schema with ingestion-level validation. Any record missing a mandatory field should be quarantined before it enters the decision pipeline.

**Control 3 — Bias Monitoring & Feature Governance** *(Before any retraining)*  
Remove ZIP code and credit history months from the model feature set entirely. Before every deployment, compute DI ratios on a holdout set — if DI drops below 0.80 for any protected attribute, block deployment until resolved.

**Control 4 — Three-Tier Human Oversight** *(Before further production deployment)*  
Replace the current binary auto-reject setup with a three-tier architecture:
- **Tier 1 (Green):** Automated approval for clearly compliant applications
- **Tier 2 (Yellow):** Mandatory human review when a proxy variable triggers or DI falls below 0.80
- **Tier 3 (Red):** Full manual review pathway for contested rejections — required under GDPR Art. 22

---

## Key Findings Summary

| # | Finding | Regulatory Impact |
| :--- | :--- | :--- |
| 1 | All 4 direct identifiers in plaintext in analytical CSV | GDPR Art. 5(1)(f) — **CRITICAL** |
| 2 | Sensitive lifestyle spending data collected without documented lawful basis | GDPR Art. 5(1)(b)(c) — HIGH |
| 3 | 88% of records missing `processing_timestamp` | GDPR Art. 5(1)(e) — HIGH |
| 4 | Gender DI = 0.76; Gen Z approval rate ~25 pp below Millennials | EU AI Act Art. 10 — **CRITICAL** |
| 5 | ZIP code proxies for gender (r = 0.786); credit history proxies for age (r = 0.657) | EU AI Act Art. 10 — HIGH |
| 6 | No human oversight mechanism; 161 rejections driven by `algorithm_risk_score` alone | GDPR Art. 22 / EU AI Act Art. 14 — HIGH |
| 7 | All 6 mandatory governance fields absent | GDPR Art. 5(2) / EU AI Act Art. 13 — **CRITICAL** |
| 8 | 98.8% re-identification rate from ZIP + gender + DOB alone | GDPR Art. 5(1)(f) — HIGH |

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
