# CODE_WALKTHROUGH.md — Forensic Execution Guide for `SR_GCN_Texas_Audit.ipynb`

## Purpose of This Document

This document is a **forensic, cell-by-cell walkthrough** of the notebook `SR_GCN_Texas_Audit.ipynb`.

The goal is not to restate code, but to explain:

- **What each cell does**
- **Why it exists** (audit rationale)
- **What artifacts it produces** (evidence)
- **Which downstream cells depend on it** (execution dependencies)
- **What integrity constraints it enforces** (audit safety)

The notebook is designed as a **single executable source of truth**.  
No external scripts are required to produce the authoritative artifacts.

---

## Global Execution Invariants

Before the cell-by-cell breakdown, the notebook enforces three global invariants:

1. **Single-Run Identity:** exactly one `RUN_ID` per runtime session.
2. **Immutable Evidence:** all artifacts are written under a unique run directory.
3. **Audit Contract First:** contracts and sensor semantics are persisted before any training evidence.

These invariants prevent “silent mixing” of runs and protect the evidence chain.

---

# CELL 01 — Session Identity, Drive Persistence, and Run Directory Layout

### What it does
- Generates an immutable run identifier:
  - `RUN_TIMESTAMP` (UTC)
  - `RUN_UUID` (short random)
  - `RUN_ID = srgnn_<timestamp>_<uuid>`
- Registers `CURRENT_RUN = RUN_ID` and **hard-fails** if a run already exists.
- Mounts Google Drive under `/content/drive`.
- Constructs the run directory:
  - `.../SR-GNN_AUDIT/runs/<RUN_ID>/`
- Creates subdirectories:
  - `artifacts/`, `metrics/`, `logs/`, `figures/`, `configs/`, `checks/`, `checkpoints/`, `tmp/`

### Why it exists (audit rationale)
This cell establishes the **forensic perimeter**:

- A run is uniquely identified.
- Evidence is guaranteed to be written in a run-scoped location.
- Evidence is separated from other runs by construction.

### Artifacts produced
- `artifacts/environment.json` — host/process/python metadata
- `artifacts/seed.json` — fixed seed

### Dependencies
All subsequent cells depend on:
- `RUN_ID`
- `RUN_ROOT`
- `DIRS`

### Integrity constraints
- `exist_ok=False` ensures a collision fails loudly.
- `CURRENT_RUN` guard prevents double-run contamination.

---

# CELL 01 (continued) — Environment & Seed Fixation

### What it does
- Sets deterministic seeds:
  - Python `random`
  - NumPy
  - Torch CPU
  - Torch CUDA (all devices) if available

### Why it exists
Reproducibility is a **first-class audit constraint**, not a convenience.

### Artifacts produced
- Already covered: `environment.json`, `seed.json`

---

# CELL 01 (continued) — Audit Contract Persistence

### What it does
Persists the run’s **audit contract** as a JSON artifact defining:

- audit spec version
- QTEES version
- scope (B0/B1 only)
- disallowed behaviors (simulation cannot drive verdict)
- governance flags (adaptive SOR disabled, KQ not trainable)

### Why it exists
This is a **governance constitution** that must exist *before* evidence.

### Artifact produced
- `artifacts/audit_contract.json`

### Downstream dependency
Later synthesis cells reference these contract fields to ensure the report remains scope-correct.

---

# CELL 01 (continued) — Dependency Bootstrap for PyG

### What it does
- Attempts `import torch_geometric`
- If missing:
  - installs `torch-geometric`, `torch-scatter`, `torch-sparse` using Torch-version-aligned wheels

### Why it exists
To prevent silent environment mismatch and reduce execution friction.

### Integrity note
A second raw `pip install ...` line appears later. This is redundant and may be removed for cleanliness, but does not affect correctness.

---

# CELL 02 — Sensor Contract Definition

### What it does
Defines the **sensor semantics** (IDs, domains, units, policies) for all metrics:

- `loss`
- `loss_variance`
- `grad_norm`
- `update_ratio` with policy `OMIT_IF_NOT_COMPUTABLE`
- `logits_entropy`
- `C` (coherence)
- `H_norm`
- `KQ`

### Why it exists
Auditors must not reverse-engineer metric meaning from code.
This cell externalizes meaning into a **stable contract**.

### Artifact produced
- `artifacts/sensor_contract.json`

### Dependencies
All metric exports and synthesis rely on contract-consistent metric keys.

---

# CELL 02.5 — Texas Dataset (WebKB) + GCN Bootstrap

### What it does
- Loads Texas WebKB via `torch_geometric.datasets.WebKB`.
- Moves data to `DEVICE`.
- Initializes a 2-layer GCN model and Adam optimizer.

### Why it exists
Establishes the **real-world dataset** and the model under audit.

### Fallback behavior
If `torch_geometric` is missing, it constructs a synthetic proxy **only to prevent crashes**.
This proxy is not intended to be used for authoritative results.

### Outputs
- Global objects: `data`, `model`, `optimizer`, `in_channels`, `out_channels`

### Integrity note
The audit contract forbids simulated evidence driving the verdict; therefore, environments must ensure PyG is installed.

---

# CELL 02.6 — System Sanity Verification

### What it does
Prints:
- Torch version
- CUDA availability
- Dataset tensor shapes
- Node/edge counts
- Model parameter count

### Why it exists
Provides immediate operator-visible validation that:

- the real dataset is loaded,
- tensors have expected shapes,
- the model is instantiated.

### Evidence
This cell is not persisted as a file by default; it is a runtime verification step.

---

# CELL 02.7 — Dual-Entropy Sensor Layer (C, H_norm, KQ)

### What it does
Defines deterministic metric functions:

1. **Coherence** `C(t)`:
   - cosine similarity of embeddings to centroid
   - clamped to [0,1] to satisfy domain contract

2. **Normalized entropy** `H_norm(t)`:
   - softmax probabilities
   - Shannon entropy
   - normalized by `log(K)` to map to [0,1]

3. **Quality** `KQ(t)`:
   - `KQ = C * (1 - H_norm)`

Defines `observe_step(step, out, loss)` that emits:
- `loss`
- `KQ`, `C`, `H_norm`, `logits_entropy`
- timestamp

### Why it exists
This cell defines the **audit sensor layer**:

- Metrics are deterministic.
- Domains are enforced.
- Output keys match sensor contract.

This prevents “metric drift” across runs.

---

# CELL 03 — SR-GCN Train Step Adapter (Texas-Bonded)

### What it does
Implements `train_step_audit()` as a single authoritative train step on the full graph:

- forward pass on **full graph**
- cross-entropy loss
- backward pass
- gradient norm calculation
- optimizer step
- parameter update ratio calculation
- rolling loss variance (window = 10)
- accuracy calculation

Returns a dictionary with:
- `loss` (tensor)
- `loss_val` (float)
- `loss_variance`
- `grad_norm`
- `update_ratio`
- `accuracy`
- `logits` (detached)
- `emb` (detached)

### Why it exists
This is the **single point of truth** for how training proceeds.
It is separated from governance logic so:

- training dynamics can be measured cleanly,
- governance interventions do not rewrite the learning rule.

### Integrity constraint
Update ratio includes epsilon floor to avoid divide-by-zero.

---

# CELL 03.5 — Baseline Evidence Collection (B0)

### What it does
Runs baseline training **without governance**:

- `NUM_B0_RUNS = 5`
- `STEPS_PER_B0 = 15`

For each B0 run:
- repeatedly calls `train_step_audit()`
- computes `KQ` via `observe_step()`
- collects arrays for each metric
- saves per-run mean statistics

Produces a baseline catalog:
- `loss_mean`
- `loss_variance_mean`
- `grad_norm_mean`
- `update_ratio_mean`
- `KQ_mean`

### Why it exists
B0 is the **empirical calibration phase** used to derive SOR bounds.

### Artifact produced
- `logs/b0_runs.json`

### Downstream dependencies
- Cell 04 (SOR calibration) consumes this file.
- Cell 05 (policy ceiling derivation) consumes this file.

---

# CELL 04 — Calibration Integrity Gate (SOR Derivation)

### What it does
Loads `logs/b0_runs.json` and computes robust bounds for governed metrics:

- `median`
- `MAD` (median absolute deviation)
- `epsilon = median + 3*MAD` (or median scaling if MAD=0)

Applies **safety floors** to prevent epsilon=0 traps.

Creates and freezes SOR:
- status `CALIBRATION_FROZEN`
- timestamp
- method string
- per-metric bounds

### Why it exists
Defines the system’s **empirical stability operating region**.

This transforms governance from heuristic thresholds into evidence-derived boundaries.

### Artifact produced
- `artifacts/sor_bounds.json`

### Dependency
Cells 05–11 rely on the frozen SOR bounds.

---

# CELL 05 — Control Policy Skeleton (B1)

### What it does
Constructs the regulated policy using SOR calibration:

- Loads `sor_bounds.json` and hashes it (`sor_hash`).
- Computes an EVD-related ceiling derived from B0 distribution:
  - uses `loss_variance_mean` as calibration base
  - computes median and MAD
  - defines `EVD_DYNAMIC_CEILING = median + 3*MAD` (with fallback)

Defines a 3-level deterministic policy:
- **L1**: mild LR scale-down
- **L2**: aggressive LR scale-down
- **L3**: abort training if instability persists

### Why it exists
This cell defines governance **before** the regulated run executes.

It ensures:
- governance cannot be retrofitted to “fit” the outcome,
- thresholds are derived from baseline evidence.

### Artifacts produced
- `artifacts/policy_spec.json`
- `artifacts/thresholds.json`

---

# CELL 06 — Regulated Training Loop (B1)

### What it does
Executes regulated training for a fixed horizon (`NUM_STEPS = 200`).

At each step:

1. Calls `train_step_audit()` to produce raw dynamics.
2. Calls `observe_step()` to compute `KQ`, `C`, `H_norm`, entropy.
3. Adds governed metrics (`loss_variance`, `grad_norm`, `update_ratio`, `accuracy`).
4. Detects violations:
   - `violating_metrics = {m : obs[m] > epsilon_m}`
5. Maintains consecutive violation counters per metric.
6. Applies cooldown logic.
7. Maps max consecutive violations to policy level:
   - L1 / L2 / L3
8. Applies action:
   - LR multiplier scaling
   - or abort

Persists a per-step record including:
- metrics
- LR multiplier
- violation set
- max consecutive violation
- worst metric
- status/level

### Why it exists
This cell is the **core demonstration**:

- governance decisions are deterministic,
- bounded by evidence (SOR),
- and measurable via event-based resilience metrics.

### Artifact produced
- `metrics/regulated_run_metrics.json`

---

# CELL 09 — Authoritative Verdict Engine (Real Evidence Only)

### What it does
Computes event-based resilience metrics from `regulated_run_metrics.json`:

1. Identifies violation steps.
2. Segments into events (closed vs open).
3. Computes:
   - **EVD** = violating_steps / total_steps
   - **RT stats** from closed events
   - abort signal via open event or ABORT status

Loads dynamic ceiling from `policy_spec.json`:
- `EVD_CEILING_DYNAMIC`

Derives final verdict:
- If Abort triggered: PASS (governance acted)
- Else PASS iff `EVD <= ceiling`

### Why it exists
This is the **single authoritative verdict source**.

No simulation evidence is used.
No hardcoded ceiling is used.

### Artifact produced
- `logs/cell09_audit_verdict.json`

---

# CELL 11 — Human-Readable Master Audit Synthesis (v2.8)

### What it does
Generates a structured, audit-safe report printed to console:

- Part A: executive summary
  - SOR bounds (median + eps)
  - B1 training dynamics (ranges, means)
  - resilience summary (RT and EVD)
  - final verdict

- Part B: deep evidence
  - B0 catalog table
  - B1 intervention chronology

Loads:
- `sor_bounds.json`
- `b0_runs.json`
- `cell09_audit_verdict.json`
- `thresholds.json`
- `regulated_run_metrics.json`

Synchronizes EVD ceiling display with `thresholds.json`.

### Why it exists
Auditors often require a **human-readable synthesis** that is:

- derived entirely from stored artifacts,
- explicit about sources,
- and consistent across runs.

This cell is designed to be “audit-safe”: it reports only what can be backed by persisted evidence.

### Evidence
Primarily console output; the authoritative evidence remains the underlying JSON artifacts.

---

# CELL 12 — CSV Metrics Export

### What it does
Flattens `regulated_run_metrics.json` into a single CSV:

- discovers all keys across records
- enforces a priority header ordering
- fills missing keys with empty strings

### Why it exists
CSV export supports:

- external auditing tools
- spreadsheet review
- ingestion into third-party analysis workflows

### Artifact produced
- `metrics/audit_metrics_export.csv`

### Integrity note (minor)
The code attempts to read a top-level `sensor_id` from `sensor_contract.json`, but the contract structure stores sensor IDs per sensor entry. This does not impact the CSV content, but provenance printing may show `unknown`. For strict correctness, this should be adjusted.

---

# CELL 15 — Comparative Analysis (Baseline vs Regulated)

### What it does
Loads:
- `logs/b0_runs.json`
- `metrics/regulated_run_metrics.json`

Computes mean metrics for:
- baseline (B0)
- regulated (B1)

Produces a comparison table and persists:
- baseline stats
- regulated stats
- stability gain percent
- quality improvement percent

### Why it exists
Provides an audit-friendly comparison:

- what changed between unregulated and regulated regimes
- whether governance improved stability

### Artifact produced
- `logs/comparative_analysis_0_vs_1.json`

---

# CELL 16 — Artifact Export & Download (Audit Archive)

### What it does
Zips key directories:
- `logs/`
- `artifacts/`
- `metrics/`

Outputs a run-scoped archive:
- `audit_evidence_<RUN_ID>.zip`

Triggers download in Colab when possible.

### Why it exists
Provides a **single immutable evidence bundle** for:

- external auditors
- third-party reviewers
- archival storage

This enforces the “forensic handoff” step.

---

# CELL 17 — Audit Visualizations (B1 Training Dynamics)

### What it does
Loads `regulated_run_metrics.json` and plots:

- Loss vs step
- Accuracy vs step
- KQ vs step

Marks governance intervention steps with vertical lines.

Saves:
- `figures/b1_training_dynamics.png`

### Why it exists
High-risk reviews often require visual evidence for:

- stability over time
- event localization
- governance interventions

Plots are supportive, not authoritative (the JSON remains source of truth).

---

## Cross-Cell Dependency Graph (High-Level)

- Cell 01 → defines run identity and persistence
- Cell 02 → defines sensor semantics
- Cell 02.5 → loads dataset + model
- Cell 02.7 → defines derived metrics (C, H_norm, KQ)
- Cell 03 → defines training step observables
- Cell 03.5 → generates baseline evidence (B0)
- Cell 04 → freezes SOR
- Cell 05 → freezes policy spec + thresholds
- Cell 06 → executes regulated run (B1)
- Cell 09 → derives verdict from B1
- Cell 11 → human-readable synthesis
- Cell 12/15/16/17 → exports and supporting analyses

---

## Forensic Guarantees Provided by the Notebook

1. **Run isolation:** unique directory, hard-fail on collision.
2. **Evidence immutability:** artifacts written once per run.
3. **Contract-first:** audit and sensor contracts written before training evidence.
4. **Real-evidence verdict:** verdict derived only from persisted B1 metrics.
5. **Data-driven thresholds:** SOR bounds and policy ceiling derived from B0 evidence.

---

## Known Minor Cleanups (Non-Blocking)

These do not affect the validity of the experiment, but improve audit polish:

1. Remove redundant raw `pip install` line after the conditional installer.
2. Fix CSV export provenance printing to correctly extract sensor IDs from the sensor contract structure.
3. Optionally persist Cell 11 console synthesis to `logs/` as a text artifact for completeness.

---

## Closing Note

This notebook is designed to function as a **self-contained audit instrument**.

The authoritative chain is:

> **Notebook execution → persisted JSON metrics → deterministic verdict → human-readable synthesis**

This separation is intentional and supports external auditing without trusting undocumented steps.

