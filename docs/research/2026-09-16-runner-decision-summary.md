# Decision summary — agent runner (celld / Flue)

- Status: sanitized public summary, 2026-09-16
- Origin: prior design discussion of runner, sandbox, celld, and Flue (no session IDs or machine paths)
- Canonical spec: [`docs/specs/00-agent-runner-architecture.md`](../specs/00-agent-runner-architecture.md)

This document is not a transcript. It summarizes the conclusions that feed the architecture draft.

## Conclusions

1. **Feasibility.** Using celld as a light JavaScript sandbox for an agent host is viable under typical deployment constraints: normal pods, custom images, managed buckets and databases; no privileged pods, no Docker-in-Docker, and no `/dev/kvm`.
2. **Scope.** JavaScript only, to transform, join, and validate data. May include downloading and forwarding images with limits. No Python. Hostile multi-tenant code isolation is not treated as the primary guarantee for the pilot.
3. **Persistence.** Flue keeps agent history and progress. Datasets, artifacts, and execution results must be persisted explicitly. PostgreSQL can be the source of truth for workspaces, versions, permissions, and executions; bytes live in a bucket. Pods stay disposable.
4. **Versus agentOS.** `agentos-core` can be used without Rivet Cloud / Rivet Actors as an executor, but it does not by itself provide a native durable Postgres + bucket model. celld aligns better with bucket durability. The executor should sit behind a thin interface so it can be replaced.
5. **Pilot posture.** Prefer celld first. Before production, demonstrate recovery after pod loss, effective cancellation, and CPU/memory containment.
6. **Naming.** The repo is provisionally `agentd`. Candidates: Grove (preferred in discussion), Cellkit, Hollow, Perch, Habitat. Decision pending.
7. **Language.** TypeScript for SDK, adapters, and the celld app; agent-generated code is JavaScript.
8. **celld surface.** Rust engine; API compatible with the Workers / Durable Objects model; Dynamic Workers for runtime code.

## Proposed model-facing capability (`execute_js`)

| Capability | Behavior |
|---|---|
| Read datasets | References to files and results authorized for the task |
| Process | JS: joins, filters, aggregations, calculations, validations |
| Call tools | Controlled calls via host gateway (read-only at first) |
| Produce results | Small summary + CSV/JSON; XLSX via server-side tool |
| Transfer images | Download/upload to allowed destinations, with limits |

Bytes do not pass through the model context. Credentials for external stores and internal services stay on the agent host.

## Minimal runner operations (proposed)

- `execute` — code, input refs, `executionId`
- `getExecution` — status or already-recorded result
- `listArtifacts` / `readDataset` — workspace materials
- `cancel` — actually stop the work
- (added in the spec) `putDataset` and outbox of idempotent *effects*

## Production decision criterion

Recovery after pod loss, real cancellation, and containment when running generated JS. If that fails in phase 0, reopen the executor comparison (e.g. QuickJS/Wasm or other options) before investing in full durability.
