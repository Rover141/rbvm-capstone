# Risk-Based Vulnerability Prioritization (RBVM Engine)

**Author:** Vishesh Grover  
**Repository:** [GitHub Repository](https://github.com/<your-username>/rbvm-capstone)  
**Notebooks:**
- [Exploratory Data Analysis & Baseline Model](notebooks/01_eda_and_baseline_model.ipynb)
- [Final Modeling, Hyperparameter Tuning & Evaluation](notebooks/02_final_modeling_and_evaluation.ipynb)

---

## Executive Summary

Enterprise cybersecurity operations face severe **alert fatigue**. Modern vulnerability scanners (such as Qualys, Tenable, and Rapid7) flag thousands of security flaws across corporate networks. Traditionally, security teams triage remediation using the industry-standard **Common Vulnerability Scoring System (CVSS)** base score, treating any flaw scored 7.0 or above as an emergency.

However, real-world threat telemetry reveals that **less than 10% of disclosed vulnerabilities are ever weaponized or actively exploited in the wild**. Organizations waste critical engineering hours patching theoretical vulnerabilities on low-value internal systems while genuinely dangerous flaws remain exposed.

This project delivers a **Risk-Based Vulnerability Management (RBVM) Engine** that combines technical scanner outputs with dynamic exploit threat intelligence (CISA Known Exploited Vulnerabilities, EPSS probabilities) and business asset criticality. Using machine learning classification, the engine isolates true risk and routes findings into an automated **24-Hour Emergency Remediation SLA** versus standard patching cycles.

---

## Business Problem & Understanding

### Current Challenges
1. **Alert Overload:** Over 40% of scanner findings are classified as "High" or "Critical" under CVSS, making remediation queues impossible to clear.
2. **Context Blindness:** A CVSS 9.0 vulnerability on an isolated internal test machine is often treated with higher urgency than an actively exploited CVSS 7.2 flaw on a public-facing payment gateway.
3. **Operational Drag:** Security and engineering teams spend hundreds of hours in manual triage meetings rather than remediating actual exposure points.

### Objective
Develop a predictive classification pipeline that accurately identifies vulnerabilities needing **immediate 24-hour remediation**, reducing patch ticket volume by at least **70%** while capturing over **95%** of high-consequence, actively exploited threats.

---

## Data Understanding & Governance

Due to corporate data privacy policies regarding internal IP architectures and proprietary asset inventories, this analysis utilizes a probabilistically generated dataset ($N=5,000$ records) modeled after enterprise vulnerability management platforms and public threat feeds:

| Feature Name | Type | Description |
| :--- | :--- | :--- |
| `vulnerability_id` | Identifier | Unique tracking identifier (e.g., `VULN-10042`) |
| `cve_id` | Identifier | Common Vulnerabilities and Exposures reference ID |
| `cvss_score` | Continuous (1.0–10.0) | Standard technical flaw severity score |
| `epss_score` | Continuous (0.0–1.0) | Exploit Prediction Scoring System probability of exploit in wild |
| `cisa_kev_listed` | Binary (0/1) | Flag indicating listing on CISA's Known Exploited Vulnerabilities catalog |
| `asset_criticality` | Categorical | Business impact tier (`Low`, `Medium`, `High`, `Critical`) |
| `internet_facing` | Binary (0/1) | Flag indicating whether the host is publicly reachable from the Internet |
| `patch_availability`| Binary (0/1) | Flag indicating if a vendor patch or workaround is published |
| `days_open` | Integer | Exposure duration (days since initial scanner detection) |
| **`action_required`** | **Binary (0/1)** | **Target Variable: 1 = Immediate 24h Emergency SLA, 0 = Standard Patch Cycle** |

---

## Data Cleaning & Feature Engineering

1. **Missing Data Imputation:** Scanner dropped-packet anomalies accounted for ~3% missing values in `cvss_score`. Missing records were resolved using median imputation grouped by `asset_criticality`.
2. **Contextual Risk Composite:** Engineered `risk_composite_score` calculated as:
   $$\text{Composite Score} = \text{CVSS} \times \text{EPSS} \times \text{Exposure Multiplier}$$
   where internet-facing assets receive a $1.5\times$ weight factor.
3. **Threat Intelligence Indicator:** Created `is_high_threat`, a binary heuristic identifying actively exploited flaws ($\text{CISA KEV} = 1$) paired with non-trivial exploit probability ($\text{EPSS} > 0.10$).

---

## Exploratory Data Analysis: Key Insights

* **Exploit Scarcity:** EPSS probabilities show a heavy right-skewed distribution. Fewer than 10% of vulnerabilities have an exploit probability $> 0.35$.
* **The CVSS Fallacy:** Numerous CVSS 9.0+ vulnerabilities lack exploit code and pose minimal immediate risk, while several CVSS 6.0–7.5 flaws with active CISA KEV listings are actively exploited by threat actors.
* **Surface Exposure Dominance:** Internet-facing assets account for over 70% of high composite risk scores, despite comprising only 25% of the total asset inventory.

---

## Model Evaluation & Comparison

### Evaluation Metric Selection
In cybersecurity operations, a **False Positive** (flagging a low-risk vulnerability for urgent review) costs a brief manual check. A **False Negative** (missing an actively exploited bug on a critical asset) risks an unmitigated breach. 

Therefore, **Recall on Class 1 (Action Required)** was chosen as the primary optimization metric, supported by **ROC-AUC** to measure overall discrimination.

### 5-Fold Stratified Cross-Validation Results

| Algorithm | Mean CV Recall | Recall Std | Mean CV ROC-AUC | ROC-AUC Std | Selection Status |
| :--- | :---: | :---: | :---: | :---: | :--- |
| **Baseline Decision Tree** | 0.938 | 0.018 | 0.957 | 0.012 | Baseline Reference |
| **Random Forest** | 0.946 | 0.014 | 0.982 | 0.007 | Candidate |
| **Tuned Gradient Boosting** | **0.962** | **0.009** | **0.989** | **0.004** | **Selected Production Model** |

### Final Test Set Performance (Tuned Gradient Boosting)
* **Recall (Emergency Class):** **0.958** (Captures ~96% of critical threats)
* **Precision:** **0.923** (Maintains high confidence; minimal false alarms)
* **ROC-AUC:** **0.989**
* **Workload Reduction:** Reclassifies **~73%** of scanner-flagged "High/Critical" tickets into routine patching cycles, isolating emergency action to high-risk assets.

---

## Business Findings & Non-Technical Takeaways

1. **Threat Feeds Outweigh Static Severity:** Dynamic signals (EPSS and CISA KEV) are far more predictive of risk than CVSS alone. Prioritizing based on threat intelligence eliminates substantial patching backlogs without increasing risk.
2. **Network Location Matters Most:** An unpatched vulnerability on an internal workstation poses fundamentally different risk than the same flaw on an edge firewall. Incorporating `internet_facing` context prevented hundreds of needless emergency calls.
3. **Measurable Efficiency Gains:** By reducing emergency tickets by ~73%, remediation teams can meet SLAs on the small subset of issues that attackers can actually leverage.

---

## Actionable Recommendations & Next Steps

### Operational Roadmap
1. **Implement Automated 2-Tier Routing:**
   * **Tier 1 (24-Hour SLA):** Vulnerabilities classified as `action_required = 1` by the RBVM Engine.
   * **Tier 2 (30-Day Standard Patch Cycle):** All remaining vulnerabilities (`action_required = 0`).
2. **ITSM Integration:** Integrate the inference pipeline into Jira or ServiceNow via webhooks to auto-tag tickets on ingestion.
3. **Live Feed Ingestion:** Connect scheduled daily updates from the CISA KEV JSON endpoint and FIRST.org EPSS API to dynamically re-score open findings.
4. **Future ML Iterations:** Collect historical Mean Time to Remediate (MTTR) by team to train regression models forecasting operational completion dates.
