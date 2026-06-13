# CloudSentry — Insights into Cloud Activity

**Author:** Rishi R.

📓 **Main Notebook:** [cloudsentry_final.ipynb](cloudsentry_final.ipynb)
📊 **Evaluation Notebook:** [CloudSentryEvaluation.ipynb](CloudSentryEvaluation.ipynb)

---

## Executive Summary

### Project Overview and Goals

CloudSentry is an unsupervised anomaly detection system that monitors AWS CloudTrail logs for suspicious cloud activity — without ever requiring labeled training data. The system learns what "normal" looks like for each user and flags meaningful deviations from it.

The goal is to proactively catch threats such as unauthorized access, privilege escalation, and active attacks before they cause damage, while keeping false-positive rates low enough that a Security Operations Center (SOC) analyst can triage every alert efficiently.

Three complementary machine learning models were combined into a single detection pipeline:

| Model | What It Catches |
|---|---|
| Time Series (Rolling Z-Score) | Sudden spikes in API call volume — e.g., a user making 30× their normal number of calls in one minute |
| DBSCAN | Events that don't fit any recognizable cluster of normal behavior (density-based outliers) |
| Isolation Forest | Unusual combinations of timing, location, IP address, and API action across 11 engineered features |

Their outputs are fused with a two-rule OR: flag if at least 2 of 3 models agree the event is anomalous, or if the Isolation Forest score is in the top 0.5% of all events.

### Key Findings

CloudSentry was evaluated on a 30-day synthetic AWS CloudTrail dataset of approximately 18,600 events with roughly 1.5% injected anomalies.

- **Isolation Forest** was the strongest single model (F1 = **0.848**, Recall = 1.0) — catching every injected anomaly. Its continuous anomaly score produced near-perfect separation between benign and anomalous events. Its ROC-AUC of 1.0 is notable but reflects a synthetic-data artifact — see the Important Limitation section below.
- **DBSCAN** spatially isolated the injected anomalies in PCA-projected feature space, placing them in a distinct low-density region far from all normal clusters. It achieved strong recall (0.91) but lower precision, flagging 9% of all events as potential anomalies in its default configuration. Hyperparameter tuning improved its F1 from 0.25 to 0.41, the largest lift of any model.
- **Time Series** flagged volumetric bursts for the most-alerted user (user_07), but registered F1 = 0 against the injected ground truth. This is expected and by design — the synthetic anomalies are individual point-in-time events, not rate spikes, so the rolling z-score is detecting a different threat class entirely. In a real deployment with scripted attacks or credential-stuffing, this component would contribute meaningfully. See the Methodology section for a full explanation.
- **The fused CloudSentry verdict** is a creative and promising experiment that reflects real SOC practice — blending different signal types into one alert. In this particular dataset, however, Isolation Forest alone (F1 = 0.848) outperformed the fused model (F1 = 0.803). The fusion approach is best understood as a strong foundation to build on rather than a performance improvement in this controlled setting.
- **Hyperparameter tuning** via grid search improved DBSCAN's F1 by +0.15. Isolation Forest saw no lift, as its default parameters already closely matched the synthetic anomaly fraction.

| Model | Precision | Recall | F1 |
|---|---|---|---|
| Isolation Forest | 0.737 | 1.000 | **0.848** |
| CloudSentry Fused | 0.712 | 0.920 | 0.803 |
| DBSCAN | 0.147 | 0.909 | 0.254 |
| Time Series | 0.000 | 0.000 | 0.000 |

### ⚠️ Important Limitation: Synthetic Data & ROC-AUC = 1.0

All metrics above reflect performance on **synthetic, rule-based data**. Because the anomalies were injected using a fixed set of explicit rules (off-hours timestamps, foreign IPs, a predefined list of sensitive API actions), the models are learning to detect the generator's pattern rather than generalizing to real-world threats. Performance on actual, messy CloudTrail logs — where legitimate users sometimes call sensitive APIs, on-call engineers work overnight, and unusual IPs may belong to authorized cloud services — would very likely be lower.

The **ROC-AUC = 1.0** reported for Isolation Forest should be treated with particular caution. It reflects how cleanly the injected anomalies cluster in feature space due to the generator's consistent rules — not proof that the model would generalise perfectly to real environments. All scores should be interpreted as an **upper bound** for this controlled scenario, not a production performance estimate.

### Conclusion

CloudSentry demonstrates that unsupervised machine learning can detect suspicious cloud activity without any labeled training data. **Isolation Forest is the strongest individual detector** on this dataset, and its performance held up through hyperparameter tuning. The fused CloudSentry verdict is a genuinely interesting experiment — combining multiple signal types is exactly the right intuition for a real SOC environment — but in this controlled setting it did not outperform Isolation Forest alone, and that distinction is important to state clearly. The most critical next step is validation against real CloudTrail logs with SOC-confirmed incidents, which would reveal how well the approach truly generalizes.

---

## Rationale

Cloud infrastructure is a prime target for attackers. A single compromised IAM credential can give an adversary access to databases, encryption keys, compute resources, and audit logs — and if the attacker disables CloudTrail logging (via `StopLogging` or `DeleteTrail`), the organization may not even know the breach happened.

Traditional rule-based security tools require defenders to anticipate every attack pattern in advance and write a rule for it. This creates a cat-and-mouse dynamic where novel attack techniques evade detection until a new rule is written. Machine learning-based anomaly detection inverts this: instead of asking "does this match a known bad pattern?", it asks "does this look like anything this user has done before?" — making it far more adaptable to novel threats.

Organizations using AWS at scale need automated, low-friction alerting that works without first collecting months of labeled incident data. CloudSentry addresses this need directly.

---

## Research Question

Can an ensemble of unsupervised machine learning models — combining time series analysis, density-based clustering, and isolation-based scoring — reliably detect suspicious activity in AWS CloudTrail logs without any labeled training data, while maintaining low enough false-positive rates to be actionable for a SOC analyst?

---

## Data Sources

Because real CloudTrail logs contain sensitive information about infrastructure, access patterns, and internal users, this project uses a **synthetic data generator** that produces realistic CloudTrail-style events.

**Dataset characteristics:**
- 30-day window (April 1 – May 1, 2026)
- 12 simulated users, each with a distinct home region, IP prefix, and work-hour profile
- ~18,600 total events; ~274 injected anomalies (~1.5%)
- 4 AWS regions: `us-east-1`, `us-west-2`, `eu-west-1`, `ap-south-1`

**Normal events** are drawn from 10 benign API actions (GetObject, ListBuckets, DescribeInstances, etc.) and arrive during each user's working hours from their typical IP range. Weekend volume is naturally 40% lower.

**Injected anomalies** are crafted with the following characteristics:
- Off-hours timestamps (1am–4am or 11pm)
- Foreign IP addresses outside each user's normal prefix
- Sensitive API actions: `DeleteUser`, `StopLogging`, `CreateAccessKey`, `Decrypt`, `DeleteTrail`, `PutBucketPolicy`, `AuthorizeSecurityGroupIngress`, `DeleteBucket`

The anomaly labels (`hidden_truth`) are immediately separated from the data after generation and stored aside. The models never see them — they are used only in the post-hoc validation step in Phase 5b.

**EDA Highlights:**
- Event volume is evenly distributed across all 12 users (~1,500–1,600 events each)
- The top 10 API actions are all benign and uniformly distributed, confirming the synthetic generator is behaving as designed
- The hourly distribution peaks between 10am–3pm, with very few events in the off-hours window (0–7am) — making off-hours timestamps a strong anomaly signal
- 47% of events originate from `ap-south-1`, reflecting the random user profile assignment

---

## Methodology

The notebook follows the **CRISP-DM** framework across six phases.

### Phase 3 — Feature Engineering

Raw CloudTrail fields are converted into 11 numeric signals before modeling:

| Feature | Description |
|---|---|
| `hour`, `dayofweek` | When the event occurred |
| `is_offhours` | 1 if before 7am or after 7pm |
| `is_weekend` | 1 if Saturday or Sunday |
| `eventName_freq` | Global frequency of this API action in the dataset |
| `sourceIPAddress_freq` | Global frequency of this IP address |
| `awsRegion_freq` | Global frequency of this AWS region |
| `user_ip_familiarity` | How often this specific user has used this IP |
| `user_region_familiarity` | How often this specific user has operated from this region |

All features are normalized with `StandardScaler` before being passed to DBSCAN and Isolation Forest.

### Phase 4 — Three Detection Models

**Time Series (Rolling Z-Score):** For each `(user, action)` pair, API calls are aggregated into per-minute buckets. Each bucket is compared against the user's trailing 7-day rolling mean and standard deviation. Buckets more than 3 standard deviations above the baseline are flagged as volumetric anomalies.

> **Note on Time Series F1 = 0:** This model is designed to catch sustained rate spikes. The injected anomalies are single point-in-time events — one `DeleteUser` call at 2am — which do not generate a burst detectable by a rolling z-score. The F1 = 0 result is a data limitation, not a model failure. In a real deployment with credential-stuffing or scripted attacks, the time series component would be valuable.

**DBSCAN:** Groups events into dense clusters of typical behavior. Any event that does not fit any cluster (labeled `-1` by the algorithm) is treated as a potential anomaly. DBSCAN is sensitive to the `eps` (neighborhood radius) and `min_samples` parameters; default values flag ~9% of all events as noise, reflecting the algorithm's tendency toward high recall.

**Isolation Forest:** Builds an ensemble of random decision trees and isolates each point. Events that require fewer splits to isolate are assigned higher anomaly scores — reflecting the intuition that truly unusual events are easier to separate from the rest of the data.

### Phase 5 — Evaluation and Fusion

The three model outputs are fused with a two-rule OR:
- **Majority Vote:** Flag if at least 2 of 3 models agree the event is anomalous
- **Extreme Score:** Flag if the Isolation Forest score exceeds the 99.5th percentile

Cross-validation is implemented through grid search hyperparameter sweeps:
- DBSCAN: `eps` × `min_samples` (16 combinations)
- Isolation Forest: `contamination` × `n_estimators` (15 combinations)
- Time Series: `window_days` × `z_threshold` (12 combinations)

**Why F1?** F1 is the primary metric because the dataset is highly imbalanced (~1.5% anomalies). Accuracy would be misleadingly high even for a model that flags nothing. F1 balances precision (how many alerts are real) against recall (how many real anomalies are caught) — both of which matter in a SOC environment.

---

## Results

### Model Performance (Default Parameters)

| Model | Events Flagged | % Flagged | Precision | Recall | F1 | ROC-AUC |
|---|---|---|---|---|---|---|
| Isolation Forest | 372 | 2.00% | 0.737 | 1.000 | **0.848** | 1.0* |
| CloudSentry Fused | 354 | 1.90% | 0.712 | 0.920 | 0.803 | 1.0* |
| DBSCAN | 1,689 | 9.08% | 0.147 | 0.909 | 0.254 | — |
| Time Series | 52 | 0.28% | 0.000 | 0.000 | 0.000 | — |

*\*ROC-AUC = 1.0 is a synthetic-data artifact. The injected anomalies were created with a small, fixed set of discrete rules (specific off-hours, foreign IPs, a predefined list of sensitive API actions), which creates a clean cluster in feature space that Isolation Forest can separate almost perfectly. On real CloudTrail logs — where legitimate users call sensitive APIs, engineers work odd hours, and authorized cloud services use unfamiliar IPs — this score would be substantially lower and should not be taken as a production performance estimate.*

### Hyperparameter Tuning Results

| Model | F1 (Default) | F1 (Tuned) | Lift |
|---|---|---|---|
| Isolation Forest | 0.848 | 0.848 | 0.000 |
| CloudSentry Fused | 0.803 | 0.730 | −0.072 |
| DBSCAN | 0.254 | **0.407** | +0.154 |
| Time Series | 0.000 | 0.000 | 0.000 |

DBSCAN showed the largest benefit from tuning (best: `eps=1.3`, `min_samples=10`). Isolation Forest's default `contamination=0.02` already matched the true anomaly fraction closely, so tuning produced no additional lift — another sign that the synthetic data is too regular, since in real logs the optimal contamination value would be unknown. The fused model's F1 actually drops after tuning (from 0.803 to 0.730) because the tuned DBSCAN flags a larger and noisier event set, which the majority-vote rule then mixes with the strong Isolation Forest signal — pulling combined precision down. This further confirms that **Isolation Forest alone is the strongest-performing model on this dataset**.

### Key Visual Findings

**From `cloudsentry_final.ipynb`:**

**EDA Overview** — Event volume is balanced across all 12 users. The hourly distribution clearly shows off-hours events as rare, making them a strong anomaly signal. The `ap-south-1` region dominates, reflecting user profile randomization.

**Isolation Forest Score Distribution** — The score histogram shows near-perfect separation: benign events cluster between −0.25 and −0.05, while injected anomalies cluster above 0.05. This visual confirms the strong ROC-AUC score and illustrates why the model performs well on this synthetic dataset.

**DBSCAN Clusters (PCA)** — Normal events form two dense elongated clusters in PCA space. Injected anomalies (red circles) fall in a distinct low-density region far to the lower-left, confirming the feature engineering successfully separates the two populations geometrically.

**Time Series — user_07** — The per-minute API call rate for the most-alerted user shows occasional spikes to 3 calls/minute. CloudSentry alerts (red dots) mark events flagged by the fused verdict, driven by Isolation Forest rather than the time series component.

**From `CloudSentryEvaluation.ipynb`:**

**Confusion Matrices** — A 4-panel subplot showing TP, FP, TN, and FN counts for every model side by side. Isolation Forest shows zero false negatives (perfect recall); DBSCAN shows a large FP count; Time Series shows zero detections of the injected anomalies.

**ROC Curve** — The full Isolation Forest ROC curve plotted alongside single-point markers for DBSCAN, Time Series, and the fused verdict, illustrating how the continuous score gives Isolation Forest a decisive advantage over binary-threshold models.

**Permutation Feature Importance** — A horizontal bar chart showing which of the 11 engineered features most impact Isolation Forest's ROC-AUC when shuffled. `is_offhours`, `sourceIPAddress_freq`, and `user_ip_familiarity` emerge as the most critical signals — directly reflecting how the synthetic anomalies were injected.

**False Positive Score Distribution** — A histogram comparing the anomaly scores of true positives vs. false positives within the set of events Isolation Forest flagged, showing that false positives cluster at lower score values than true positives and could be filtered with a higher threshold.

**Per-User Alert Breakdown** — A stacked bar chart showing total events, CloudSentry alerts, and injected anomaly counts per user, useful for identifying which users would be prioritised for SOC investigation.

---

## Next Steps and Recommendations

1. **Validate against real CloudTrail logs.** The single most important next step is testing CloudSentry against actual AWS environments with SOC-confirmed incidents. Real data will expose gaps in feature engineering and likely require recalibration of all three model thresholds. Work with a security team to collect a labeled set of confirmed malicious events.

2. **Add more anomaly types to the synthetic generator.** The current generator only injects individual point-in-time events. Adding volumetric burst scenarios (e.g., a credential-stuffing attack that fires hundreds of `AssumeRole` calls in one minute) would let the time series model contribute meaningfully to validation and give a more complete picture of ensemble performance.

3. **Improve the fusion strategy.** The current majority-vote + extreme-score rule does not weight models by their demonstrated reliability. A learned meta-classifier (e.g., logistic regression trained on the three models' output scores) could improve precision at the same recall level — though it would require labeled data to train.

4. **Add UEBA-style behavioral features.** The current feature set treats each event independently. Richer features — such as rolling counts of sensitive API calls per user per hour, or a flag for "first time this user has ever called this action" — would capture behavioral drift over time and likely improve detection of slow-moving threats.

5. **Build an operational alerting layer.** CloudSentry currently exports results to a CSV. A production deployment would benefit from a live integration with AWS EventBridge or a SIEM (e.g., Splunk, Elastic) that can route `cloudsentry_alert = 1` events to analyst queues in real time.

---

## Outline of Project

```
cloudsentry_final.ipynb              # Main analysis notebook (run top to bottom)
CloudSentryEvaluation.ipynb          # Standalone evaluation notebook (run independently)
cloudsentry_results.csv              # Per-event alert output (generated on run)
plot_eda_overview.png                # 4-panel EDA overview
plot_timeseries_baseline.png         # Time series visualization for most-alerted user
plot_dbscan_clusters.png             # DBSCAN clusters in PCA space
plot_isolation_forest_scores.png     # Isolation Forest score distribution
plot_eval_confusion_matrices.png     # Confusion matrices for all four models
plot_eval_roc_curve.png              # ROC curve with binary model comparison points
plot_eval_score_distribution.png     # Benign vs. anomalous score histogram with threshold
plot_eval_feature_importance.png     # Permutation feature importance bar chart
plot_eval_fp_vs_tp_scores.png        # False positive vs. true positive score distribution
plot_eval_timeseries.png             # Time series chart with alert overlay (evaluation)
plot_eval_per_user_alerts.png        # Per-user event volume, alerts, and anomaly counts
README.md                            # This file
```

**Main notebook structure (CRISP-DM):**
- [Phase 1 — Business Understanding](cloudsentry_final.ipynb)
- [Phase 2 — Data Understanding & Synthetic Data Generation](cloudsentry_final.ipynb)
- [Phase 2b — Data Cleaning & EDA](cloudsentry_final.ipynb)
- [Phase 3 — Feature Engineering & Data Preparation](cloudsentry_final.ipynb)
- [Phase 4 — Modeling (Time Series · DBSCAN · Isolation Forest)](cloudsentry_final.ipynb)
- [Phase 5a — Evaluation: Alert Summary](cloudsentry_final.ipynb)
- [Phase 5b — Model Comparison vs. Hidden Ground Truth](cloudsentry_final.ipynb)
- [Phase 5c — Hyperparameter Tuning](cloudsentry_final.ipynb)

**Evaluation notebook structure:**
- [Section 1 — Setup (data, features, model fitting)](CloudSentryEvaluation.ipynb)
- [Section 2 — Model Evaluation (metrics, confusion matrices, ROC curve, score distribution, Time Series analysis)](CloudSentryEvaluation.ipynb)
- [Section 3 — Model Interpretation (feature importance, example predictions, false positive analysis, per-user breakdown, tuning recap)](CloudSentryEvaluation.ipynb)
- [Section 4 — Conclusion & Next Steps](CloudSentryEvaluation.ipynb)

---

## Requirements

```
numpy
pandas
matplotlib
seaborn
scikit-learn
```

Install with:
```bash
pip install numpy pandas matplotlib seaborn scikit-learn
```

> `seaborn` is required by `CloudSentryEvaluation.ipynb` for plot styling. The main notebook (`cloudsentry_final.ipynb`) uses only `matplotlib`.

---

## Contact and Further Information

**Rishi R.**

Project developed as part of the UC Berkeley Professional Certificate in Machine Learning & Artificial Intelligence Capstone.

For questions about this project or the CloudSentry approach, please reach out via the repository.
