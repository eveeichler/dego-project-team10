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
```

> **Note:** Notebook 02 reads `clean_credit_applications.csv` from the `data/` directory — run Notebook 01 first to generate it.

---
