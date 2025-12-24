# RESULTS.md — Audit Interpretation and Findings

## Scope of This Document

This document provides an **independent, audit-oriented interpretation** of the experimental results produced by the notebook `SR_GCN_Texas_Audit.ipynb`.

The goal is **not** to restate logs or code behavior, but to explain:

- what the observed results *mean*,
- how an external auditor should read them,
- what conclusions are justified — and which are *not*,
- how the system behaves under **baseline (B0)** and **regulated (B1)** regimes,
- whether the governance logic performs as claimed.

All interpretations in this document are grounded exclusively in persisted artifacts:

- `b0_runs.json`
- `sor_bounds.json`
- `policy_spec.json`
- `regulated_run_metrics.json`
- `cell09_audit_verdict.json`
- `comparative_analysis_0_vs_1.json`

No undocumented assumptions are introduced.

---

## 1. Experimental Structure Recap (Context for Interpretation)

The experiment is intentionally structured into **two distinct regimes**:

- **B0 — Baseline (Unregulated)**
- **B1 — Regulated (Governed)**

This separation is critical. The system does *not* attempt to compare “two training runs” in the usual ML sense. Instead, it evaluates:

> How a real GCN behaves **without governance**, and how the *same learning dynamics* behave **under explicit audit governance**.

The objective is **resilience and auditability**, not raw performance optimization.

---

## 2. Interpretation of B0 — Baseline Calibration Runs

### 2.1 What B0 Represents

B0 runs are **not training for convergence**. They serve a single purpose:

> To empirically characterize the *normal operating behavior* of the system *without any governance interventions*.

Each B0 run:

- uses the real Texas WebKB graph,
- executes a small, fixed number of steps (15),
- applies the same optimizer and model as B1,
- records raw dynamics without constraint or intervention.

### 2.2 Why Multiple B0 Runs Matter

Five independent B0 runs are used to:

- reduce sensitivity to random initialization effects,
- obtain a *distribution*, not a point estimate,
- support **robust statistics** (median and MAD).

This is essential for audit contexts, where single-run thresholds are considered fragile.

### 2.3 Observed Baseline Characteristics

From the baseline catalog:

- Loss steadily decreases across B0 runs (expected learning behavior).
- Gradient norms and update ratios show **natural variability**, including occasional higher values.
- KQ increases monotonically across runs, reflecting improving representation structure.

**Key interpretation:**

> The baseline system is *learning normally*, but exhibits non-zero volatility in all monitored metrics.

This volatility is *not* labeled as “bad” — it is **measured**, not judged, at this stage.

---

## 3. SOR — Stability Operating Region (Audit Meaning)

### 3.1 What SOR Is (and Is Not)

The Stability Operating Region (SOR) is:

- an **empirical envelope** derived from B0 behavior,
- defined per metric,
- frozen before B1 execution.

SOR is **not**:

- a safety guarantee,
- a proof of correctness,
- a theoretical bound.

It is a **data-backed expectation window**.

### 3.2 How to Read SOR Bounds

Each SOR bound consists of:

- `median`: typical baseline behavior,
- `epsilon`: allowed deviation ceiling (median + 3·MAD, floored).

Example interpretation:

> “If `grad_norm` exceeds its SOR epsilon, the system is behaving *outside* what was empirically observed during baseline operation.”

This framing is critical: **violations are contextual, not absolute errors**.

### 3.3 Why Safety Floors Matter

Safety floors prevent the pathological case where:

- a metric shows near-zero variance during baseline,
- resulting epsilon would be zero,
- causing every non-zero observation to be flagged.

From an audit perspective, this is a **false-positive prevention mechanism**.

---

## 4. B1 — Regulated Run Interpretation

### 4.1 What Changes in B1

In B1, *nothing changes* about:

- the model architecture,
- the optimizer,
- the loss function,
- the dataset.

What changes is **observability + governance**:

- metrics are continuously checked against SOR,
- violations are tracked over time,
- policy actions are applied deterministically.

This isolates governance effects from learning effects.

---

## 5. Interpretation of Governance Interventions

### 5.1 Observed Intervention Summary

From the intervention chronology:

- Exactly **one intervention** occurred.
- The trigger metric was `grad_norm`.
- The applied level was **L1 (mild stabilization)**.
- No repeated or escalating violations followed.

### 5.2 How an Auditor Should Read This

This pattern is **healthy**, not suspicious:

- A single, short-lived deviation is expected in stochastic optimization.
- Immediate recovery indicates that the system is *not fragile*.
- The absence of escalation demonstrates proportional governance.

In audit terms:

> The system detected a deviation, responded minimally, and stabilized.

This is precisely the intended behavior of tiered governance.

---

## 6. EVD — Evidence Density (Violation Density)

### 6.1 What EVD Measures

EVD is defined as:

> fraction of training steps that belong to violation states.

It answers the question:

> “How often does the system operate outside its expected stability envelope?”

### 6.2 Observed EVD Interpretation

In this run:

- EVD = 0.005
- Dynamic ceiling ≈ 0.007

Meaning:

- Only 0.5% of steps were violating.
- This is **below** the data-derived governance ceiling.

From an audit standpoint:

> Deviations are *rare* and *bounded*.

---

## 7. RT — Recovery Time Interpretation

### 7.1 What RT Represents

Recovery Time (RT) measures:

- how many consecutive steps the system remains in violation before recovering.

It is a **temporal resilience metric**, not a performance metric.

### 7.2 Observed RT Profile

- RT_min = 1
- RT_median = 1
- RT_max = 1

Interpretation:

> Every detected deviation was corrected within a single step.

This indicates:

- fast feedback effectiveness,
- no sustained instability,
- no hidden oscillatory behavior.

---

## 8. Abort Logic and Its Audit Meaning

### 8.1 Abort Does Not Mean Failure

In this framework:

- an abort is considered a **successful governance outcome**,
- it signals that unsafe behavior was *detected and stopped*.

### 8.2 Observed Outcome

- No abort was triggered.
- No open violation events remained.

Interpretation:

> The system remained governable throughout the run.

---

## 9. Comparative Analysis (B0 vs B1)

### 9.1 What the Comparison Is — and Is Not

The comparison:

- does **not** claim improved generalization,
- does **not** claim better final accuracy.

It evaluates:

> Whether governance altered stability-related properties.

### 9.2 Observed Effects

- Loss variance decreased under B1.
- KQ increased modestly.

Interpretation:

> Governance introduced *stability without suppressing learning*.

This is a critical property for high-risk deployments.

---

## 10. Final Verdict Interpretation

The final verdict is **PASS**.

This means:

- the system stayed within empirically justified stability bounds,
- governance acted when needed,
- no uncontrolled behavior was observed,
- all conclusions are traceable to persisted evidence.

**PASS does not mean**:

- the model is universally safe,
- the model is optimal,
- the model is certified for deployment.

It means:

> Under the audited conditions, the system behaved as designed, and governance claims are supported by evidence.

---

## 11. How an External Auditor Is Expected to Read This

An external auditor should conclude:

1. The experiment is **well-scoped** and not over-claimed.
2. Thresholds are **data-derived**, not arbitrary.
3. Governance logic is **deterministic and traceable**.
4. Results are **reproducible** given the same environment.
5. Evidence artifacts are sufficient for independent review.

---

## 12. Explicit Non-Claims (Important)

This experiment does **not** claim:

- robustness to adversarial attacks,
- formal verification of safety,
- generalization guarantees beyond Texas WebKB,
- performance superiority over other GCN variants.

These exclusions are intentional and protect audit integrity.

---

## Closing Statement

This experiment demonstrates a **practical, evidence-driven approach to auditing neural network stability and governance**.

Its value lies not in accuracy numbers, but in:

- explicit observability,
- bounded deviations,
- measurable recovery,
- and traceable decision logic.

These properties are essential for **high-risk, regulated, or safety-sensitive ML deployments**.

