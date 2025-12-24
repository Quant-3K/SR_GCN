# Mathematical Foundations of the SR-GCN Audit Experiment

## Scope and Purpose

This document provides the **formal mathematical foundation** for the SR-GCN audit-first experiment.  
Its purpose is to define, unambiguously and without reference to implementation details, the quantities, metrics, and statistical procedures used to:

- characterize training dynamics,
- define stability boundaries,
- govern runtime behavior,
- and derive an audit-safe verdict.

This document is intentionally **implementation-agnostic**.  
All formulas defined here must be traceable to observable quantities produced by the execution notebook.

---

## 1. Problem Setting

We consider supervised learning on a fixed graph:

- A graph \( G = (V, E) \) with \(|V| = N\) nodes and edges \(E\)
- Node features \( X \in \mathbb{R}^{N \times d} \)
- Labels \( y \in \{1, \dots, K\}^N \)

A **Graph Convolutional Network (GCN)** parameterized by \(\theta\) defines a function:

\[
 f_\theta : (G, X) \rightarrow P(Y|X)
\]

Training proceeds via gradient-based optimization on a fixed dataset.

---

## 2. Why Accuracy Is Insufficient

Let \(A(t)\) denote task accuracy at training step \(t\).

While \(A(t)\) measures predictive performance, it is **not a sufficient indicator of safety or stability**, because:

- high accuracy can coexist with unstable optimization,
- instability may not immediately affect predictions,
- catastrophic divergence often manifests first in internal dynamics.

Therefore, the audit focuses on **training dynamics**, not solely on outcomes.

---

## 3. Primary Observables

At each training step \(t\), the following raw quantities are observed:

- Loss: \( L(t) \in \mathbb{R}^+ \)
- Gradient vector: \( \nabla_\theta L(t) \)
- Parameter vector: \( \theta(t) \)
- Model output distribution: \( p(t) = P_\theta(Y|X) \)

All derived metrics are functions of these observables.

---

## 4. Gradient Norm

The gradient norm is defined as:

\[
\|g(t)\|_2 = \left\| \nabla_\theta L(t) \right\|_2
\]

It serves as a first-order indicator of optimization stress and potential divergence.

---

## 5. Parameter Update Ratio

Let \( \theta(t) \) and \( \theta(t+1) \) denote parameters before and after an optimizer step.

The **update ratio** is defined as:

\[
R(t) = \frac{\|\theta(t+1) - \theta(t)\|_2}{\|\theta(t)\|_2 + \varepsilon}
\]

where \( \varepsilon > 0 \) prevents division by zero.

This quantity measures the **relative magnitude of parameter perturbation**.

---

## 6. Loss Variance (Local Instability)

To capture short-term instability, a sliding window variance of the loss is computed.

Given a window of size \(w\):

\[
\mathrm{Var}_L(t) = \frac{1}{w-1} \sum_{i=t-w+1}^{t} \left( L(i) - \bar{L}_t \right)^2
\]

where \( \bar{L}_t \) is the mean loss in the window.

Loss variance is treated as a **second-order instability signal**.

---

## 7. Entropy of Model Outputs

Let \( p(t) = (p_1, \dots, p_K) \) be the predicted class distribution.

Shannon entropy is defined as:

\[
H(t) = - \sum_{k=1}^{K} p_k \log p_k
\]

To ensure comparability across tasks, entropy is normalized:

\[
H_{\mathrm{norm}}(t) = \frac{H(t)}{\log K} \in [0,1]
\]

---

## 8. Coherence Measure

A coherence score \(C(t) \in [0,1]\) is defined as a bounded measure of structural alignment in model representations.

Formally:

\[
C(t) = \mathrm{clip}(\cos(\phi(t)), 0, 1)
\]

where \(\phi(t)\) denotes an internal representation similarity angle.

The exact computation is fixed and deterministic, ensuring reproducibility.

---

## 9. Quality Metric (KQ)

The **Quality metric (KQ)** combines structure and uncertainty:

\[
\boxed{\quad KQ(t) = C(t) \cdot \bigl(1 - H_{\mathrm{norm}}(t)\bigr) \quad}
\]

Properties:

- \(KQ(t) \in [0,1]\)
- Penalizes confident noise and uncertain structure
- Increases only when coherence improves *and* entropy decreases

KQ is **not trainable** and **not optimized directly**.

---

## 10. Baseline Calibration (B0)

A short baseline phase is executed without governance.

Let \(M = \{m_1, \dots, m_n\}\) denote a metric observed during B0.

Robust statistics are computed:

- Median: \( \tilde{m} \)
- Median Absolute Deviation (MAD):

\[
\mathrm{MAD}(M) = \mathrm{median}(|m_i - \tilde{m}|)
\]

---

## 11. Safety Operating Region (SOR)

For each governed metric, the **Safety Operating Region** is defined as:

\[
\mathrm{SOR}_{\epsilon} = \tilde{m} + 3 \cdot \mathrm{MAD}
\]

This yields an empirical, data-driven stability boundary.

No hard-coded constants are used beyond the robustness multiplier.

---

## 12. Event Violation Density (EVD)

Let \(T\) be the total number of training steps in the regulated run.

Let \(V\) be the number of steps where at least one SOR boundary is violated.

The **Event Violation Density** is defined as:

\[
\mathrm{EVD} = \frac{V}{T}
\]

This quantity captures the *frequency* of instability events.

---

## 13. Recovery Time (RT)

For each violation event \(e\):

\[
RT(e) = t_{\mathrm{recovered}} - t_{\mathrm{violation}}
\]

Summary statistics (min, median, p95, max) characterize system resilience.

---

## 14. Governance and Verdict Logic

Let \( \epsilon_{\mathrm{EVD}} \) denote the maximum allowed EVD derived from B0.

The audit verdict is determined as:

- **PASS** if:
  - training completes without abort, and
  - \( \mathrm{EVD} \leq \epsilon_{\mathrm{EVD}} \)

- **FAIL** otherwise

No subjective interpretation is involved.

---

## 15. Interpretation Boundary

The mathematics defined here:

- specifies *what is measured*,
- defines *how stability is quantified*,
- and constrains *how conclusions may be drawn*.

It does **not** claim universality, optimality, or completeness.

---

## Closing Remark

The central principle of this experiment is simple:

> *A learning system is only trustworthy if its internal dynamics are measurable, bounded, and governable.*

This document formalizes that principle mathematically.

