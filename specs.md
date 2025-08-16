
---

### **Aegis Framework: Technical Specification v1.0**

This document provides a detailed technical and conceptual specification for the Aegis Framework, an operating system for verifiable reasoning and strategically-aligned decision-making.

**License:** MIT License
**Author/Architect:** [Your Name/Username Here, e.g., TeleoChrono]
**Version:** 1.0
**Status:** Conceptual Blueprint

---

### **1. Core Philosophy & The Problem of Strategic Drift**

#### **1.1. Core Values & Purpose**
The Aegis Framework is built on a foundation of three core values:

1.  **Verifiable Truth:** Knowledge must be distinguishable from opinion. The system’s primary function is to enforce this distinction by demanding proof.
2.  **Strategic Integrity:** Actions must serve intentions. The system is designed to relentlessly expose and close the gap between declared goals and actual outcomes.
3.  **Resilience through Falsifiability:** A system's strength is measured by its ability to withstand and learn from rigorous attempts to break it. Aegis is designed to be challenged.

#### **1.2. The Core Problem: The Intent-to-Outcome Gap**
In any complex endeavor, a systemic gap emerges between intentions and results. This is not due to a single failure, but a series of seemingly rational, locally-optimized actions that collectively lead to strategic drift.

- **The Illusion of Progress:** We declare noble goals and values, but our daily actions often create an illusion of progress without producing meaningful outcomes.
- **The Compounding of Uncertainty:** A project is a sequence of steps (`n` stages), where each step builds upon the last. Early-stage uncertainty or a minor error doesn't just add up; it compounds exponentially, corrupting the entire chain of reasoning. By the final stages, the initial goal is often unreachable, but it's too late and too costly to course-correct.
- **The Information Asymmetry of Time:** Early stages have low information but high leverage; late stages have high information but low leverage. A decision made at `step 1` has far greater consequences than a decision at `step n-1`.

Aegis solves this by inverting the flow of constraints. It recognizes that the **end-state is the most information-rich point in the process**. By defining the properties of a successful outcome (`EndAnchor`), we can propagate those requirements backward in time as non-negotiable constraints (`BackConstraints`) for earlier stages. This forces early, high-leverage steps to be compatible with the final, high-information state from the very beginning.

#### **1.3. Target Audience & Use Cases**
This framework is critical for groups operating in high-stakes, information-dense environments where the cost of being wrong is significant.
*   **Strategic Analysis & Intelligence:** Building verifiable reports for business, geopolitical, or scientific analysis.
*   **Product & System Design:** Making complex engineering or product decisions with a clear, auditable rationale, managing alternatives, and aligning with business goals.
*   - **AI Agent & LLM Orchestration:** Creating reliable and safe agentic systems where actions are governed by verifiable facts, executable policies, and strategic objectives, not just opaque model outputs.
*   **Regulatory & Compliance Audits:** Producing fully traceable documentation that proves adherence to legal, ethical, or brand standards.

---

### **2. Architectural Overview: The Three Pillars**

The framework is composed of three core layers.

#### **2.1. Pillar 1: AegisDB - The Epistemological Database**
The foundation is a database designed to store knowledge with proof. It enforces epistemological invariants at the transactional level. It is not just a data store; it is a **truth-enforcing engine**.

**Critical Invariants (Non-negotiable rules):**
1.  **Evidence-First:** An `artifact` of kind `claim` or `number` cannot be committed without at least one source. "Sourceless facts" are rejected by the database.
2.  **Unit & Temporal Safety:** Numerical artifacts must have defined `units`. The database's `UnitContract` system prevents operations on incompatible dimensions (e.g., `kg + m`) and requires explicit, dated conversions for compatible ones (e.g., `USD -> EUR`). All data has a temporal context (`valid_from`, `valid_to`).
3.  **Monotonic & Independent Validity:** The validity score (`v`) of an artifact is a monotonic function of its independent evidence. It is calculated as `v = 1 - Π(1 - w_i * level_i)`, where evidence from correlated sources is down-weighted or de-duplicated by a `Source-Independence Score` function.
4.  **Full Provenance & Reproducibility:** Every write operation is an immutable `JournalEvent`, forming a verifiable chain. This log includes `model_id`, `prompt_hash`, and `seed` for any AI-driven step, ensuring reproducibility.
5.  **Alternative Exclusivity:** Artifacts within an `alt_group_id` must have exactly one member in the `resolved` state.
6.  **Acyclicity:** The `requires` relationship graph must be a Directed Acyclic Graph (DAG). Changes to a parent node automatically propagate a `stale` flag to all descendants.

#### **2.2. Pillar 2: AegisFlow - The Orchestration Engine**
This layer turns the static knowledge in AegisDB into a dynamic, goal-oriented process.

*   **ECL-lite (Evidence Contract Language):** A declarative YAML-based language for defining the entire analysis workflow. A contract is a machine-readable plan that specifies steps, data sources, quality gates (`asserts`), and the final `EndAnchor`.
*   **Scheduler (`Ω`):** A priority engine that manages the frontier of possible next actions. It uses a utility function to rank tasks:
    `priority = f(VOI, CoD, Risk, Cost, Novelty, rVOI)`
    where `rVOI` (Retrocausal Value of Information) is the expected improvement in the `EndAnchor` pass rate from performing an action.

#### **2.3. Pillar 3: TeleoGuard - The Strategic Alignment Layer**
This is the framework's strategic "conscience." It operates as a pre-check validation layer before the Scheduler commits to any action.

**Core Entities & Logic:**
*   **`EndAnchor` (EA):** The formal, machine-readable definition of success. A JSON object defining rules like `min_validity: 0.8`, `min_alternatives: 2`, `no_hard_violations: true`. Critically, Aegis supports `TeleoDiversity`: using an ensemble of multiple `EndAnchors` to represent different success scenarios or stakeholder interests.
*   **`BackConstraint` (BC):** An executable rule derived from an EA that is enforced on early-stage artifacts. The `backchain` operator in ECL-lite generates and applies these.
*   **`Penultimate Sentinel` (PS):** A specialized dry-run check targeting nodes at distance `d <= 2` from a terminal node. It ensures that the data required for the final, irreversible steps is of sufficient quality *before* those steps are reached.
*   **`EndMirror`:** A fast, cached simulation that answers: "Given the current state of the EvidenceGraph, what is our probability of passing the `EndAnchor` ensemble?" It returns a vector of margins against each EA.
*   **`Intent-Action Gap` (IAG):** A composite metric calculated before each action:
    `IAG = f(ΔUtility, VC_violations, ΔConsistency, ΔEA_margin, ΔRisk, ReversibilityBonus)`
    A negative IAG triggers a `soft-fail` (penalty, recommendation for alternatives) or a `hard-fail` (block, Socratic question to the user).

---

### **3. Technical Implementation Details**

#### **3.1. AegisDB Data Models (PostgreSQL DDL Sketch)**

```sql
-- Artifacts: The core nodes of the knowledge graph
CREATE TABLE artifact (
    id TEXT PRIMARY KEY,
    kind TEXT NOT NULL CHECK (kind IN ('claim', 'number', 'table', 'report', 'decision', 'latent_risk')),
    content JSONB NOT NULL,
    units TEXT,
    v DOUBLE PRECISION DEFAULT 0 CHECK (v BETWEEN 0 AND 1),
    r DOUBLE PRECISION DEFAULT 0 CHECK (r BETWEEN 0 AND 1),
    sources JSONB NOT NULL,
    owner TEXT,
    state TEXT DEFAULT 'draft' CHECK (state IN ('draft', 'resolved', 'stale', 'invalidated')),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    valid_from TIMESTAMPTZ,
    valid_to TIMESTAMPTZ,
    alt_group_id TEXT
);

-- Edges: The relationships between artifacts
CREATE TABLE edge (
    src TEXT REFERENCES artifact(id) ON DELETE CASCADE,
    dst TEXT REFERENCES artifact(id) ON DELETE CASCADE,
    type TEXT NOT NULL CHECK (type IN ('supports', 'contradicts', 'requires', 'refines', 'alternative_of')),
    PRIMARY KEY (src, dst, type)
);

-- EvidenceItem: Atomic pieces of proof
CREATE TABLE evidence_item (
    id TEXT PRIMARY KEY,
    artifact_id TEXT NOT NULL REFERENCES artifact(id) ON DELETE CASCADE,
    level TEXT NOT NULL CHECK (level IN ('opinion', 'citation', 'replicated', 'unit_test', 'simulation', 'formal_proof')),
    weight DOUBLE PRECISION NOT NULL CHECK (weight BETWEEN 0 AND 1),
    source_url TEXT NOT NULL,
    source_hash TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- JournalEvent: The immutable log of all actions
CREATE TABLE journal_event (
    id BIGSERIAL PRIMARY KEY,
    operation TEXT NOT NULL,
    inputs JSONB,
    outputs JSONB,
    agent_id TEXT,
    model_id TEXT,
    prompt_hash TEXT,
    seed BIGINT,
    timestamp TIMESTAMPTZ DEFAULT NOW(),
    prev_hash TEXT
);
-- ... other tables for ValueConstraint, EndAnchor, etc.
```

#### **3.2. ECL-lite Operators (Key Examples)**

A contract is a list of steps. Each step is an operator.

```yaml
steps:
  # Setup: Define the success criteria for multiple scenarios
  - end_anchor: {id: "base_scenario", rules: {min_v: 0.7, ...}}
  - end_anchor: {id: "optimistic_scenario", rules: {min_v: 0.8, ...}}

  # Data Ingestion & Processing
  - fetch_docs: {source: "./data", tag: "raw_reports"}
  - extract_facts: {from: "raw_reports", mode: "llm|regex", tag: "extracted_claims"}
  - pin_model: {model_id: "claude-3-opus-20240229", seed: 42} # Ensures reproducibility

  # Enforcement of Invariants & Constraints
  - assert_source_independence: {target: "extracted_claims", min_sources: 3, min_score: 0.7}
  - assert_time_window: {from: "2024-01-01", to: "2024-12-31"}
  - backchain: {from: ["base_scenario", "optimistic_scenario"], to_stage: "extraction", strict: true}

  # Decision & Reporting
  - generate_alternatives: {for_decision: "invest_or_not", count: 3}
  - alt_parity: {group: "invest_or_not", min_score: 0.8} # Ensure alternatives are equally vetted
  - endmirror_ensemble: {anchors: ["base_scenario", "optimistic_scenario"]}
  - assemble_report: {format: "html", manifest: true}
```

#### **3.3. Critical Algorithms & Formulas**
*   **Validity Aggregation (`v`):** `v = 1 - Π_i (1 - (w_i * level_weight_i * independence_score_i))` where `independence_score` is a function of source domain/author/timestamp clustering.
*   **Scheduler Utility Function (`F_p`):** `priority = w_u*u*(1-r) + w_voi*VOI + w_cod*CoD + w_rvoi*rVOI - w_cost*cost`
*   **IAG Calculation:** A weighted sum of negative changes in utility, consistency, EA margins, and risk, offset by a bonus for reversible actions.

---

### **4. Critical Considerations & Nuances**

*   **The DSL Adoption Problem:** A declarative language like ECL-lite can be a barrier. **Solution:** The system must provide a Level 1 interface (a UI wizard or guided forms) that generates the ECL-lite contract automatically. The DSL is for power users (Level 2).
*   **The Goodhart's Law Risk:** As soon as a metric (`v`) becomes a target, it ceases to be a good measure. **Solution:** The system includes `Anti-Goodhart` mechanisms: `v` is never just a number but is always paired with its `explain_prov` trail. The system also requires `v-calibration` against benchmark datasets to detect metric inflation.
*   **Performance at Scale:** Graph traversal can be slow. **Solution:** AegisDB is designed for a hybrid model. The core data lives in a relational database with strong indexing. Graph queries for provenance (`explain_prov`) or complex relationships are performed on-demand or through materialized views/caches for critical paths.
*   **Human-in-the-Loop:** This is not a fully autonomous system. It is a **cognitive exoskeleton**. Its purpose is to augment, not replace, human judgment. `Socratic Questions` generated by the IAG check are a critical interface for ensuring the human user understands the risks and trade-offs of their actions.

---

### **5. Anti-Requirements (What This System Is Not)**

*   **It is not a "naked" AI wrapper.** It does not accept opaque outputs from models without forcing them through a rigorous process of sourcing, validation, and unit checking.
*   **It is not a magic black box.** Every output is fully reproducible and traceable back to its foundational evidence and the contract that generated it.
*   **It does not assume the final goal is correct.** `TeleoDiversity` and the `Q_hypothetical_failures` queries are designed to stress-test the declared goals themselves, allowing for strategic pivots based on evidence.

Понял, нужно создать техническую спецификацию, которая будет служить глубоким, детальным документом для инженеров, архитекторов и продакт-менеджеров. Она должна не только описывать "что" и "как", но и глубоко раскрывать "зачем", опираясь на твои философские и практические инсайты. Я вытащу все технические детали из твоих заметок, структурирую их, добавлю недостающие концептуальные мосты и отвечу на твои ключевые вопросы.

Отличная идея. Это именно то, что нужно профессионалам, чтобы быстро "пощупать" систему и понять ее структуру без необходимости читать многостраничный документ. Это концентрированная выжимка ключевых параметров, контрактов и инвариантов.

---

### **Aegis Framework: Architect's Cheat Sheet v1.0**

This document provides a high-level summary of the key components, parameters, and invariants of the Aegis Framework. It is intended for architects, senior engineers, and technical leads to rapidly understand the system's structure and design principles.

#### **I. AegisDB: The Epistemological Store**

The database layer. Its primary role is to enforce truthfulness and traceability at the storage level.

**`Artifact` (The Node)**
*   **`kind`**: [Enum] Critical type information (`claim`, `number`, `decision`, `report`, `latent_risk`). Defines the artifact's behavior and validation rules.
*   **`state`**: [State Machine] The core workflow state (`draft`, `resolved`, `stale`, `invalidated`). Transitions are managed by system triggers.
*   **`metrics`**: [Struct] The core epistemological vector `{v, r}`. Validity (`v`) is calculated by the DB, risk (`r`) is often a model- or user-supplied input.
*   **`units` & `temporal_validity`**: [String/TimestampTZ] Non-negotiable fields for numerical and time-sensitive data. `units` must map to a `UnitContract`. `valid_from`/`valid_to` define the data's relevance window.
*   **`sources`**: [JSONB/Array] A mandatory, non-empty list of source URLs/citations. The core of the **Evidence-First** invariant.

**`EvidenceItem` (The Proof)**
*   **`artifact_id`**: [FK] The claim this item substantiates.
*   **`level`**: [Enum] The quality of the evidence (`opinion`, `citation`, `replicated`, `unit_test`, `simulation`, `formal_proof`). This is a key input for the validity (`v`) calculation.
*   **`source_hash`**: [String] A hash of the source content at the time of ingestion. Critical for detecting source drift and ensuring **reproducibility**.
*   **`independence_score`**: [Float] A calculated metric (0-1) indicating how independent this source is from other evidence for the same artifact. Crucial for preventing **validity inflation**.

**`JournalEvent` (The Immutable Log)**
*   **`operation`**: [String] The name of the function/action performed (e.g., `extract_facts`, `assess_validity`).
*   **`reproducibility_payload`**: [JSONB] The "magic" that ensures determinism. Must contain `model_id`, `prompt_hash`, and `seed` for any LLM-based operation.
*   **`prev_hash`**: [String] A hash of the previous journal event, creating a verifiable chain of custody (blockchain-like).

**`ValueConstraint` (The Guardrail)**
*   **`severity`**: [Enum] `hard` (blocks transaction on failure) vs. `soft` (raises flag/penalty). The core of the **executable ethics** model.
*   **`rule_expr`**: [String/Expression] The machine-readable rule to be evaluated against an artifact's content or metadata.
*   **`scope`**: [String] Defines which `Artifact.kind` or tags this constraint applies to.

---

#### **II. AegisFlow: The Orchestration Engine**

The process layer. It uses contracts to drive the workflow and a scheduler to prioritize actions.

**`ECL-lite Contract` (The Plan)**
*   **`end_anchors`**: [Array of Objects] A list of one or more `EndAnchor` definitions. The presence of multiple anchors enables **TeleoDiversity**.
*   **`steps`**: [DAG] A directed acyclic graph of operations to be performed. The structure defines the flow of execution.
*   **`asserts`**: [Array of Rules] Quality gates that must be passed at the end of the contract execution for the final artifact to be marked `resolved`.

**`Scheduler (Ω)` (The Brain)**
*   **`priority_function_weights`**: [Config Struct] The tunable weights for the utility function, e.g., `{w_voi, w_cod, w_rvoi, w_cost, w_novelty}`. This is the primary control knob for the system's strategic behavior.
*   **`frontier`**: [State Object] The set of currently possible actions/steps the scheduler can choose from.
*   **`budget_constraints`**: [Config] Limits on `cost` or `latency` that the scheduler must adhere to.

---

#### **III. TeleoGuard: The Strategic Alignment Layer**

The validation layer that operates via reverse causality.

**`EndAnchor` (The Goal)**
*   **`target`**: [String] The `Artifact.kind` (e.g., `report`, `decision`) to which these rules apply.
*   **`rules`**: [JSONB Object] The set of conditions for a successful outcome. Critical parameters include `min_v`, `max_r`, `min_alternatives`, `required_provenance_depth`, and `no_hard_vc_violations`.
*   **`mode`**: [Enum] `strict`, `balanced`, `fast`. A preset that adjusts the "Escalation by Proximity" parameters.

**`BackConstraint` (The Reverse Rule)**
*   **`anchor_id`**: [FK] The `EndAnchor` that generated this constraint.
*   **`applies_to`**: [Selector] A selector defining the scope, e.g., `stage:extraction` or `type:number`.
*   **`rule_expr`**: [Expression] The rule to be enforced (e.g., `units IS NOT NULL`, `source.date >= '2024-01-01'`).

**`IAG (Intent-Action Gap)` (The Drift Metric)**
*   **`component_weights`**: [Config Struct] The weights for each factor in the IAG calculation, e.g., `{w_goal, w_vc, w_consistency, w_ea_margin, w_risk, w_reversibility}`. Allows tuning the "sensitivity" of the system's conscience.
*   **`disposition_thresholds`**: [Config] The IAG score thresholds that trigger `soft-fail` vs. `hard-fail`.
*   **`socratic_question_map`**: [Config] A mapping from failure types (e.g., `EA_MARGIN_DECREASED`) to specific, templated questions posed to the human operator.

---

#### **IV. System-Wide Concepts & Metrics**

*   **`Resilience Score`**: [Float] A single metric (0-1) produced by the `Anti-System` after running a suite of adversarial tests (`Adversarial 30`). Measures the system's robustness against known failure modes.
*   **`EA-PassRate`**: [Float] The percentage of workflows that pass their `EndAnchor` gates on the first attempt. The primary KPI for TeleoGuard's effectiveness.
*   **`Cost-per-Valid-Fact`**: [Float] An economic metric tracking the resource cost (time, money) to produce a `resolved` artifact with `v` above a certain threshold.
