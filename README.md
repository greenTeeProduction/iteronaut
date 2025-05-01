# Iteronaut: Self-Contained Software Actor for BGS Iterations

[![CI](https://github.com/greenTeeProduction/iteronaut/actions/workflows/ci.yml/badge.svg)](https://github.com/greenTeeProduction/iteronaut/actions/workflows/ci.yml)

## 1. Mission Statement & Scope

Iteronaut is a **self-contained software actor** designed to execute the full lifecycle of iterations within the Bulk-Governance Simulator (BGS) programme. It ingests an "Iteration Prompt" and drives the seven-stage process (PIS → SPI → SEP → STE → SSV → SPA → SGR → SRF) for each iteration.

Key guarantees include:

1.  **Deterministic Reproducibility**: Same prompt and seed yield identical artefacts.
2.  **Hard-Stop Safety**: Progress halts without gate approval; automatic rollback is enforced.
3.  **Governance Compliance**: Real-time signals and adherence to Charter thresholds.

---

## 2. High-Level Architecture (HLA)

```
┌───────────────────────────────┐      ┌───────────────────────────┐
│ 1  Prompt-IO Layer            │◄─────┤ Governance UI / Council   │
├───────────────────────────────┤      └───────────────────────────┘
│ 2  Control Orchestrator       │
│   ├── 2a Planner              │
│   ├── 2b Executor             │
│   └── 2c Sentinel Watchdog    │
├───────────────────────────────┤
│ 3  Domain Modules             │
│   ├── Workspace Manager (WSM) │
│   ├── Dependency Manager      │
│   ├── Activity Runner         │
│   ├── Gate Validator          │
│   ├── Metrics Engine          │
│   ├── Artefact Publisher      │
│   └── Rollback Handler        │
├───────────────────────────────┤
│ 4  Persistence & Audit        │
│   ├── SQLite event log        │
│   ├── Artefact registry CSV   │
│   └── Metrics store (Parquet) │
└───────────────────────────────┘
```

Components communicate asynchronously via **typed message bus events** (Pydantic models over `asyncio.Queue`).

---

## 3. Key Design Principles

| Principle                | Mechanism                                                              | Rationale                                      |
| ------------------------ | ---------------------------------------------------------------------- | ---------------------------------------------- |
| **Fail-closed**          | Single gate failure triggers auto-rollback and pipeline pause.         | Governance demands existential-risk containment. |
| **Pure-function gates**  | Validators read logs/artefacts but **never** mutate state.             | Guarantees deterministic audits.               |
| **Separation of duties** | Planner decides; Executor does; Sentinel monitors.                     | Reduces coupling, eases reasoning.             |
| **Immutable inputs**     | IPC and dependencies hashed at intake; no in-flight edits.             | Blocks prompt-poisoning.                       |
| **Continuous self-audit**| Every atomic action emits a signed JSON envelope with millis timestamps. | Enables chain-of-trust replay.                 |

---

## 4. Control Flow Overview

1.  **Planner**: Loads and verifies the Iteration Prompt, compiles it into a plan (structured Task objects with deterministic seeds), and sends it to the Executor.
2.  **Executor**: Sets up a workspace, prepares the environment, runs tasks sequentially within namespaced containers, checks gates after each task, validates the final state, publishes artefacts, and reports status. Rolls back on any failure.
3.  **Sentinel Watchdog**: Monitors stage progress via the event log. Issues warnings (SIGINT) and eventual hard kills (SIGKILL) if stages exceed expected timeouts, marking the iteration as `stalled`.

---

## 5. Domain Modules

| Module               | Core Functionality                                             |
| -------------------- | -------------------------------------------------------------- |
| **Workspace Manager**| Manages temporary Git branches & Docker network namespaces.    |
| **Dependency Manager** | Pins dependencies using `pip --require-hashes`, fingerprints wheels. |
| **Activity Runner**  | Executes tasks using appropriate back-ends (shell, Python, container). |
| **Gate Validator**   | Runs pure-function validation scripts (`gates/gate_*.py`).      |
| **Metrics Engine**   | Evaluates metrics using Pandas and Pydantic schemas.           |
| **Artefact Publisher** | Uploads artefacts (e.g., to S3) and records checksums.         |
| **Rollback Handler** | Executes rollback scripts, reverts Git, cleans up artefacts.   |

All modules operate within a context providing workspace details, metadata, and logging.

---

## 6. Typed Event Model

Events like `StageMessage` and `MetricEvent` (Pydantic models) are used for internal communication and reporting. All messages are SHA-256-signed with the agent's ED25519 key, and re-signed by the Sentinel before being forwarded to the Governance UI via Kafka.

---

## 7. Governance Integration

-   **Reporting**: Successful iterations send summary (`iter_summary`) and optional trigger (`trigger_event`) messages to a dedicated Kafka topic (`gov-signals`).
-   **Control**: The Governance Council can pause/resume the Iteronaut pipeline via WebSocket commands.

---

## 8. Error Handling & Safety

-   **Automatic Rollback**: Triggered by dependency issues, gate failures, checksum mismatches.
-   **Pipeline Pause/Freeze**: Triggered by critical KPI breaches or high-severity security vulnerabilities.
-   **Watchdog Timeout**: Leads to SIGINT/SIGKILL and `stalled` status.
-   **Resource Limits**: Enforced via cgroups; breaches trigger task kill and rollback.
-   Escalations are reported to the Governance UI with severity indicators.

---

## 9. Testing & Verification

| Layer         | Tool                 | Frequency           |
| ------------- | -------------------- | ------------------- |
| Unit          | `pytest` (+ coverage)| On commit           |
| Integration   | `behave` BDD         | Nightly             |
| End-to-end    | `make replay ITER=X` | Weekly              |
| Chaos         | `tox-chaos` plugin   | Monthly             |
| Security      | Trivy, detect-secrets| CI pipeline (GHA)   |

Tests run on self-hosted GitHub Actions runners replicating production environment constraints.

---

## 10. Bootstrapping

1.  Clone the repository: `git clone git@github.com:greenTeeProduction/iteronaut.git`
2.  Generate agent key: `scripts/gen_agent_key.sh`
3.  Install dependencies: `pip install -r requirements.lock`
4.  Configure message bus (e.g., Kafka): `scripts/setup_kafka.sh`
5.  Launch the agent: `python run_agent.py --mode daemon`
6.  Monitor via Grafana: `http://localhost:3000/dashboard`

---

## 11. Extensibility

Adding new iterations primarily involves:

-   Creating an Iteration Prompt Configuration (IPC) YAML file in `iterations/`.
-   Registering the prompt's SHA hash.
-   Implementing necessary pure-function gate scripts (`gate_*.py`).

The core Iteronaut agent logic remains largely unchanged, ensuring scalability.

---

## 12. Security Hardening

Planned enhancements include key rotation, reproducible builds (Nix), OPA network policies, supply-chain attestations (Sigstore), and runtime Seccomp profiles.
