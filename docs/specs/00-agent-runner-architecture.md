# 00 — Agent runner: architecture and feasibility (discussion draft)

- Status: draft v0, 2026-09-16
- Source: [decision summary (sanitized)](../research/2026-09-16-runner-decision-summary.md)
- Project name: `agentd` (provisional, see §3)

## 1. Problem

Agents (today on Flue) work with data that neither fits in nor should pass through the model
context: spreadsheets, large tables, files, and images. Today, tabular attachments are often
truncated when converted to text, and existing virtual sandboxes do not give the model a clear
way to run transformations over complete datasets.

We need the agent to be able to:

1. Run self-generated JavaScript to join, filter, and verify complete data.
2. Keep datasets, artifacts, and execution results across agent pauses, even if Flue, the
   executor, or the pod restarts or is lost.
3. Retry without duplicating external effects (upload a file, write a result).
4. Do so under ordinary infrastructure constraints: normal pods with a custom image, managed
   buckets and databases; no privileged pods, no Docker-in-Docker, no `/dev/kvm`.

Flue keeps the agent’s history and progress. Nothing today keeps the agent’s *workspace*.
That gap is what this runner covers.

## 2. Goals and non-goals

### Goals

- An execution service for agents (*agent runner*) that provides **persistent per-task
  workspaces** and **bounded JS executions**.
- An execution sandbox on **celld** (Dynamic Workers), deployed in normal pods.
- **Durable state in PostgreSQL**: workspaces, datasets/artifacts (references and versions),
  executions and their attempts, permissions. Bytes in an **S3-compatible bucket**.
- **Execution idempotency**: the `executionId` survives Flue retries; if celld finished but the
  response was lost, the runner returns the already-recorded result.
- **Flue-first integration**, with a tool surface (`execute_js` and related) that the agent host
  can expose to the model.
- **Swappable agent core**: the same runner API should later serve Eve or Pi via thin adapters,
  without changing the data model.
- **Swappable executor**: celld behind a small interface, with exportable data (Postgres +
  bucket), so it can be replaced (e.g. QuickJS/Wasm or agentos-core) without migrating a
  proprietary filesystem.

### Non-goals (for now)

- Python, native packages, or a Linux shell inside the sandbox. JavaScript only.
- Hostile multi-tenant code isolation as the primary guarantee. celld is declared alpha and not
  safe for hostile code; the end user is internal and code goes through capability controls, pod
  limits, and result review.
- Replacing Flue: the runner does **not** store conversation history or agent decisions.
- Using celld’s Containers modality (requires Docker/Podman and privileged networking).
- Building a custom JS runtime or modifying celld’s Rust engine.
- Large joins against external analytical stores: those stay on the agent host with its
  credentials; the sandbox combines files and bounded results.

## 3. Naming note

The repo is provisionally named `agentd`. Design discussion proposed **Grove** as the project
name, with alternatives Hollow, Perch, **Cellkit** (makes the celld relationship explicit) and
Habitat, and a package scheme like `@agentd/grove`, `@agentd/grove-flue`, `@agentd/grove-eve`.

Decision pending. Suggested criterion: if the runner will grow the runtime beyond celld, prefer
a name with its own identity (Grove); if it will be a thin layer on celld, Cellkit. This
document uses "runner" as a neutral term.

## 4. Integration with celld

### What celld provides

- Rust engine; JS/TS programming surface compatible with the Workers / Durable Objects model;
  **Dynamic Workers** to run code at runtime.
- Runs as a process inside an image; no DinD or KVM needed for the Workers modality.
- Persistence and coordination backed by a bucket (`CELLD_DURABILITY=bucket` waits for the bucket
  write before confirming). S3 is among the supported providers.
- Capability model: outbound network can be blocked (`globalOutbound: null`) and specific
  capabilities exposed to dynamic code.
- Maintained under `denoland`, Apache-2.0, active releases (v0.5.0 on 2026-09-15).

### What it does not provide (and the runner must cover or verify)

- Dynamic Workers **do not enforce** the `cpuMs` and `subRequests` limits from their API. A
  `while (true)` or a huge memory allocation is contained only by pod limits, which are global to
  the process.
- Effective cancellation of an in-flight execution: still to be demonstrated.
- Isolation for hostile code: declared out of scope by the project.
- Security patches only for the latest version: requires staying close to releases.

### Responsibility split

```diagram
┌──────────────────────────┐
│ Agent host / Flue        │ goal, history, decisions, permissions, tool gateway
└────────────┬─────────────┘
             │ SDK (TS) + Flue adapter
┌────────────▼─────────────┐      ┌──────────────┐
│ runner (control plane)   │─────▶│ PostgreSQL   │ workspaces, datasets, artifacts, executions
│ API, idempotency,        │      └──────────────┘
│ permissions, retries     │─────▶┌──────────────┐
└────────────┬─────────────┘      │ Bucket       │ bytes: CSV, JSON, XLSX, images, results
             │ HTTP               └──────▲───────┘
┌────────────▼─────────────┐             │
│ celld app (Worker + DO)  │─────────────┘  (celld’s own data, bucket durability)
│  ┌────────────────────┐  │
│  │ Dynamic Worker     │  │ agent JS; explicit capabilities: readDataset,
│  │ (one execution)    │──┼──▶ saveArtifact, callTool (via host gateway, read-only at first)
│  └────────────────────┘  │
└──────────────────────────┘
```

| Component | Keeps or runs |
|---|---|
| Flue | History, agent progress, pending decisions |
| runner + PostgreSQL | Source of truth for workspaces, versions, executions, permissions |
| Bucket | Full files as normal objects, addressed by content or version |
| celld app (one cell/DO per workspace) | Orchestration of the in-flight execution, working cache; **nothing that cannot be rebuilt from Postgres + bucket** |
| Dynamic Worker | Code for one execution, with bounded access to authorized inputs |

Design rule: **celld orchestrates, Postgres remembers**. celld’s own durability keeps its
internal state across restarts, but the runner does not depend on it to recover a task. That
keeps the executor swappable (see decision summary, point 4).

### Initial deployment

Normal pod, pinned image, unprivileged user, local writable directory, CPU/RAM limits, bucket
access, **one replica**. The runner can live in the same pod or another; starting in the same
one simplifies the pilot. Managed Postgres (runner-owned or a schema shared with the agent
host: see §8).

## 5. Durable model: PostgreSQL + bucket

Principles:

- One home for each fact: the **execution** lives in Postgres; its code, inputs (by version), and
  outputs are referenced from there. Bytes live in the bucket.
- **An execution finishes when its outputs are in the bucket and its Postgres record is
  confirmed**, in that order. Before that, for the agent it did not happen.
- Flue retries reuse the `executionId`. Recording the result does not by itself prevent duplicate
  external effects; destination operations (upload a file to another system) also need an
  idempotency key. They are modeled as pending *effects* with an outbox, not as direct calls from
  the Worker.
- Reproducibility: each execution stores code hash, exact input versions, runtime version
  (celld + image). True determinism also requires controlling time, randomness, and external
  responses; that remains a later improvement.
- Mid-flight JS variables and half-executed functions are **not** preserved: celld rebuilds
  objects after deactivation. What persists is files + metadata + confirmed results.

## 6. Path: Flue now, Eve / Pi later

### Flue (initial phase)

The agent host exposes tools backed by the runner SDK to the model. Design discussion proposed an
`execute_js` tool with these capabilities, available to code as functions:

| Capability | Behavior |
|---|---|
| Read datasets | References to files and results authorized for that task |
| Process | JS: joins, filters, aggregations, calculations, validations |
| Call tools | Controlled calls that go through the host gateway |
| Produce results | Small summary + CSV/JSON files; XLSX via a server-side tool |
| Transfer images | Download and upload to allowed destinations, with size and type limits |

Bytes do not pass through the model context: the model gets references and summaries.
Credentials for external stores and internal services stay on the agent host.

The Flue adapter is thin: it maps tool calls to runner operations and propagates the stable task
identity (`taskId` → `workspaceId`) and the `executionId` for idempotency. Flue does not change
its own persistence.

### Eve / Pi (later)

The runner contract (workspace, execution, artifact) does not know about Flue. An adapter for Eve
or Pi does the same mapping with their own tool and resume mechanisms. Condition: the runner API
must not leak Flue concepts (message types, history format). Eve and Pi APIs are not assumed in
this document; they will be evaluated when relevant.

## 7. Proposed components

Everything in TypeScript; agent-generated code is JavaScript.

| Package (provisional name) | Responsibility |
|---|---|
| `runner-core` | Shared contracts (`Workspace`, `DatasetRef`, `ArtifactRef`, `ExecutionResult`) with runtime validation; execution state machine |
| `runner-server` | Node service: HTTP API, Postgres, bucket, effects outbox, client toward celld |
| `runner-celld` | App deployed on celld: Worker + DO per workspace + Dynamic Worker launch with capabilities |
| `runner-executor` | Small interface that abstracts “run code with inputs and capabilities”; `runner-celld` is the first implementation |
| `runner-sdk` | TS client for Node (what the agent host consumes) |
| `runner-flue` | Adapter: Flue tools → SDK |
| `runner-eve`, `runner-pi` | Future adapters |

Minimal runner operations (proposed in design, **not yet implemented or fully specified**):

- `execute` — code, input refs, `executionId`.
- `getExecution` — status or already-produced result.
- `listArtifacts` / `readDataset` — materials of the authorized workspace.
- `cancel` — actually stop the work.

Added as needs spotted in this design: `putDataset` (register an input from the host) and an
idempotent *effects* mechanism for external transfers.

## 8. Data schema (sketch)

Names and columns are indicative; the real schema is defined in phase 1.

```text
workspaces        id, owner (task/agent), created_at, status, policy (allowed capabilities)
datasets          id, workspace_id, name, current_version_id
dataset_versions  id, dataset_id, blob_key, size, content_hash, mime, created_by_execution_id?
artifacts         id, workspace_id, execution_id, name, blob_key, size, content_hash, mime
executions        id (client executionId), workspace_id, status, code_hash, code_blob_key,
                  input_version_ids[], runtime_version, requested_at, finished_at, result_summary
execution_attempts id, execution_id, attempt_no, executor, started_at, ended_at, outcome, error
effects           id, execution_id, kind, idempotency_key, target, status, attempts, last_error
permissions       workspace_id, principal, scope (read/write/tools)
```

`executions` states: `pending → running → (succeeded | failed | cancelled)`. A repeated
`execute` with the same `executionId` returns the current status without starting another run.
`execution_attempts` separates runner retries from the logical execution.

Blobs: full objects in the bucket, key derived from `workspace_id` + content hash. No proprietary
fragmentation: the workspace rebuilds from Postgres + bucket alone.

## 9. Risks and open questions

### Risks

| Risk | Mitigation / test |
|---|---|
| celld alpha; patches only on latest version | Pin version, follow releases, swappable executor interface |
| No `cpuMs`/`subRequests` limits on Dynamic Workers | Infinite-loop and memory tests; pod limits; isolate per replica if needed; measure impact on other executions |
| Cancellation not effective | Explicit phase-0 test; if it fails, restart process/pod as last resort and document it |
| Generated code influenced by external files | Minimal capabilities, `globalOutbound: null`, read-only gateway at first |
| Pod loss mid-execution | “Bucket then Postgres” rule; `execution_attempts`; Flue and celld restart test |
| Duplicate external effects on retry | Outbox + destination idempotency key |
| Bucket consistency requirements for celld | Verify against the real deployment bucket |
| Workers ↔ celld compatibility gaps | Limit the surface used; test in the pod |

### Open questions

1. Final project and package names (§3).
2. Runner-owned Postgres vs a schema inside the host’s Postgres? Owned isolates lifecycle; shared
   simplifies permissions and joins with host permissions.
3. How does `callTool` reach the host gateway from the Dynamic Worker: direct HTTP bridge with a
   per-execution token, or always through the runner? The second option centralizes audit.
4. How much state to leave in the cell? Proposal: in-flight orchestration only. Is it worth the
   cell keeping anything durable if Postgres already has it?
5. Multi-replica celld and workspace → replica affinity: out of MVP, but it constrains the API.
6. Max dataset and artifact size per execution; version retention policy.
7. What resume contract Eve and Pi require; not yet investigated.
8. Comparison with QuickJS/Wasm and agentos-core as alternate executors: do it in phase 0 or
   defer until celld fails a test?

## 10. Phased MVP

Each phase ends with something runnable and a validating test.

**Phase 0 — Technical feasibility (spike, no durability)**
- celld in a normal pod with a custom image; a minimal app that launches a Dynamic Worker with a
  sample transform.
- Tests: infinite loop, memory excess, unauthorized network access, cancellation.
- Output: report on what was contained, what was not, and resource cost.

**Phase 1 — Durable workspace + idempotent execution**
- Postgres + bucket with the §8 schema; `execute`, `getExecution`, `cancel`, `listArtifacts`,
  `readDataset`, `putDataset`.
- Primary design test: load a spreadsheet, normalize identifiers, join with another table, save
  `differences.csv`, **restart Flue and celld**, continue the same task with the same data
  version, without losing files or repeating effects.
- Idempotency test: repeat `execute` with the same `executionId` after losing the response.

**Phase 2 — Flue adapter on the agent host**
- `execute_js` tool with `readDataset`, `saveArtifact`, `callTool` (read-only).
- Real case: “join this sheet with its related tables and produce the differences.”

**Phase 3 — External effects and hardening**
- Image tools (download/upload) with outbox and destination idempotency.
- Per-execution limits, observability (per-execution metrics, logs), retention.

**Phase 4 — Second agent core**
- Eve or Pi adapter on the same SDK; validate that the API did not leak Flue concepts.

Production decision criterion: recovery after pod loss, real cancellation, and containment of
resource use when running generated JS. If phase 0 fails on cancellation or containment, reopen
the executor comparison before phase 1.
