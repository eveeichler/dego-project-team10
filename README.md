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
