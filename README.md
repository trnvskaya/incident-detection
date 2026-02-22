# Predictive Alerting for Cloud Metrics (PoC)

This repository contains a Proof of Concept (PoC) for a predictive alerting system. The objective is to predict whether a server incident (e.g., a crash) will occur within a future time horizon based on a historical window of time-series telemetry data.

## 1. Problem Formulation

The core challenge is transforming an unsupervised time-series anomaly detection problem into a **supervised binary classification** task. This is achieved using a **Sliding Window** approach:
* **Historical Window (`W = 30`):** The model observes the past 30 time steps (minutes) of system metrics.
* **Prediction Horizon (`H = 10`):** The model predicts if at least one critical incident will occur in the *next* 10 time steps.

If an incident falls within the horizon `H`, the preceding window `W` is labeled as `1` (positive class). Otherwise, it is labeled as `0` (normal state).

## 2. Data Generation & Preprocessing

Since real-world cloud incident datasets are highly specific and often lack clean anomaly precursor labels, I generated a synthetic dataset representing a typical cloud metric (`CPU_Load`):
* **Baseline:** A sine wave representing daily seasonality combined with Gaussian noise.
* **Anomalies:** 50 deterministic incidents were injected. Crucially, each incident is preceded by an "anomaly build-up" (a sharp, unnatural spike in CPU load over 10 time steps). This simulates real-world memory leaks or traffic spikes.
* **Train/Test Split:** A strict **chronological split** (80% Train / 20% Test) was used. Standard random splitting is an anti-pattern in time-series forecasting as it causes future data leakage.

*Note on Class Imbalance:* The positive class represents ~7% of the generated windows. This maintains a realistic class imbalance challenge while providing enough True Positives for the model to train and for stable evaluation of the Precision-Recall tradeoff.

## 3. Modeling: Random Forest vs. LightGBM

To ensure a rigorous evaluation, I trained and hyperparameter-tuned two tree-based architectures:
1. **Random Forest Classifier** (Bagging ensemble)
2. **LightGBM Classifier** (Gradient Boosting framework)

Both models utilized `class_weight='balanced'` to heavily penalize missing actual incidents (False Negatives).

### Evaluation Metrics (Test Set)
In a predictive alerting system, `Accuracy` is misleading. The evaluation focuses on:
* **Recall:** Catching the real crashes before they happen (preventing downtime).
* **Precision:** Ensuring alerts are genuine (minimizing "Alert Fatigue" for on-call engineers).

| Metric / Outcome | Random Forest (Tuned) | LightGBM (Tuned) | Impact of LightGBM |
| :--- | :--- | :--- | :--- |
| **True Positives (Caught Incidents)** | 90 | **94** | Caught 4 additional incident windows. |
| **False Negatives (Missed Incidents)**| 23 | **19** | Reduced missed crashes by ~17%. |
| **False Positives (False Alarms)**| 17 | **11** | Reduced false alarms by ~35%. |
| **Precision** | 0.84 | **0.90** | Significantly more reliable alerts. |
| **Recall** | 0.80 | **0.83** | Better overall sensitivity to anomalies. |
| **F1-Score** | 0.82 | **0.86** | Superior overall balance. |

## 4. Conclusion & Business Decision

**The tuned LightGBM is the strict winner for this predictive alerting pipeline.** Even when both models are hyperparameter-optimized, LightGBM's sequential boosting architecture proves to be inherently better at isolating the "anomaly build-up" pattern than Random Forest's parallel bagging approach. LightGBM achieved a Pareto improvement: it is better at catching real anomalies while simultaneously generating fewer false alarms. 

In a real-world Cloud Infrastructure environment, this translates directly to less server downtime and significantly reduced "Alert Fatigue" for DevOps engineers. Furthermore, LightGBM's computational efficiency makes it the optimal choice for scaling to millions of rows of real-time telemetry data.
