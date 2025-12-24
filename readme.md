# SR-GCN: Audit-First Regulated Graph Convolutional Network

## Executive Positioning

This repository presents a **single, fully reproducible, audit-first machine learning experiment** demonstrating how a Graph Convolutional Network (GCN) can be trained and governed under **explicit stability, risk, and resilience constraints**.

The work is intentionally **not** positioned as:
- a production ML framework,
- a reusable library,
- or an optimization-focused benchmark.

Instead, it is a **forensic-grade experimental artifact**, designed to answer one precise question:

> **Can a neural network training process be made externally auditable, stability-governed, and suitable for high-risk domains where failure modes matter more than peak accuracy?**

The answer provided by this repository is empirical, evidence-based, and fully traceable.

---

## Motivation: Why Audit-First ML

Modern machine learning systems increasingly operate in **high-risk environments**, including:

- financial decision systems,
- infrastructure and network control,
- safety-critical automation,
- regulated scientific and industrial domains.

In such environments, the dominant ML success metric — *accuracy* — is **insufficient**.

Key unresolved problems in conventional ML practice include:

- Lack of **formal stability boundaries** during training
- Absence of **runtime governance mechanisms**
- Inability to **forensically reconstruct** why a model behaved safely or unsafely
- Heavy reliance on post-hoc evaluation instead of **online risk control**

This experiment addresses these gaps by reframing training itself as a **regulated dynamic process**, not a black-box optimization.

---

## What This Repository Contains

This repository contains exactly **one authoritative experiment** and its complete evidence trail:

- A single execution notebook (`SR_GCN_Texas_Audit.ipynb`)
- A frozen archive of all generated artifacts (`artifacts.zip`)
- A formal mathematical specification (`MATH.md`)
- A line-by-line explanation of the code (`CODE_WALKTHROUGH.md`)
- A structured interpretation of the results (`RESULTS.md`)

There are **no hidden scripts**, **no auxiliary pipelines**, and **no post-processing steps** outside the notebook.

---

## Core Idea: Regulated Training Instead of Unconstrained Optimization

The experiment introduces a **governed training loop** built around four principles:

1. **Baseline Calibration (B0)**  
   A short, uncontrolled baseline phase is used to empirically measure the natural statistical operating region of the system.

2. **Safety Operating Region (SOR)**  
   From baseline evidence, explicit statistical bounds are derived for critical training metrics using robust estimators.

3. **Online Governance (B1)**  
   During the regulated run, all training dynamics are continuously monitored against the SOR.

4. **Deterministic Intervention & Recovery**  
   When violations occur, predefined governance actions are triggered, and recovery behavior is explicitly measured.

This transforms training from a best-effort optimization into a **controlled dynamical system**.

---

## Metrics That Matter (Beyond Accuracy)

In addition to standard task performance, the experiment tracks:

- **Loss Variance** — a proxy for optimization instability
- **Gradient Norm** — a signal of potential divergence
- **Update Ratio (Δθ / θ)** — magnitude of parameter perturbation
- **Entropy of Model Outputs** — uncertainty and dispersion
- **Coherence (C)** — structural alignment of representations
- **Quality Metric (KQ)** — a bounded, interpretable signal combining structure and entropy

These metrics are not used for tuning — they are used for **governance**.

---

## Event Governance & Resilience

Rather than preventing all violations, the system explicitly allows **controlled instability events**, provided that:

- violations are detected,
- interventions are applied deterministically,
- recovery occurs within bounded time.

Key resilience indicators include:

- **Event Violation Density (EVD)**
- **Recovery Time (RT)**
- **Abort Conditions**

The final audit verdict depends on these indicators — not on subjective interpretation.

---

## High-Risk Orientation

This repository is explicitly oriented toward **high-risk ML contexts**, where:

- silent failure is unacceptable,
- traceability is mandatory,
- and governance must be explicit, not implicit.

The experiment demonstrates *how* such systems can be built and evaluated, not merely *that* they can achieve good performance.

---

## How to Read This Repository

Recommended reading order:

1. **This README** — conceptual framing
2. **`MATH.md`** — formal definitions and theoretical grounding
3. **`SR_GCN_Texas_Audit.ipynb`** — executable evidence
4. **`RESULTS.md`** — interpretation of outcomes
5. **`CODE_WALKTHROUGH.md`** — implementation details

Each component serves a distinct purpose and should be read as such.

---

## What This Experiment Does *Not* Claim

- It does not claim optimal accuracy.
- It does not claim universal generalization.
- It does not propose a finished production system.

Instead, it provides **auditable evidence** that regulated, stability-aware training is both feasible and measurable.

---

## Intended Audience

This repository is intended for:

- ML auditors and reviewers
- Risk and safety engineers
- Researchers in robust and governed ML
- Organizations evaluating ML for regulated or safety-critical deployment

---

## License and Usage

This work is provided for research, audit, and evaluation purposes.  
Any reuse in production or safety-critical systems requires independent validation.

---

## Closing Statement

Machine learning systems should not be trusted because they perform well on average.

They should be trusted **only if their failure modes are observable, bounded, and governable**.

This repository is a concrete, reproducible step in that direction.

