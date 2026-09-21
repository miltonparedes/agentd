# 01 — Logical VFS and runtime gates

- Status: draft v1, 2026-09-20
- Parent: [00 — Agent runner architecture](00-agent-runner-architecture.md)
- Purpose: lock the durable-data shape and phase-0 gate closures without expanding the parent draft

## 1. Decision: logical VFS (catalog), not a filesystem layer

| Option | Verdict |
|---|---|
| Opaque refs only | Insufficient for model ergonomics, grants, listings, “what persists” |
| **Logical VFS (catalog)** | **Adopt** — path-like names → immutable versions → `blob_key` in Postgres; whole objects in the app bucket with ranged GET / multipart PUT |
| POSIX / mountable FS | Reject for MVP — inodes, partial writes, journals; second SoR; sandbox has no mount API |

**Phase 0 does not need a full FS layer.** Stub or minimally prototype the facade + streaming /
cancel / containment contracts. Durable catalog implementation is phase 1.

celld fact: `node:fs` = request-local `/tmp` + read-only `/bundle`. No mount API. What the agent
“sees” is a capability facade over the runner catalog by construction.

## 2. Ownership

| Concern | Owner |
|---|---|
| Catalog (names, versions, summaries, ACLs) | **runner** (Postgres) |
| Blob API (ranged GET, multipart PUT, hashing, publish) | **runner** |
| Reconciler, attempt fence, cancel semantics | **runner** |
| Sandbox + DO orchestration + loader facade | **celld** (rebuildable only) |
| System of record | Postgres + **app** bucket — never celld Workflows / KV / D1 / Queues / DO SQLite |

## 3. Closed gates (spec + phase-0 actions)

### Cancel / reconcile + attempt fence

- Reconciler in the runner: lease / heartbeat on `execution_attempts`.
- Monotonic fence via `active_attempt_no`; publish only when
  `status='running' AND active_attempt_no=$n`.
- Artifact keys **per attempt** (immutable staging); only the runner publishes.
- Tests: runner death, celld loss, cancel vs completion race, late publish.
- Cancel accepted ≠ compute finished — measure Facet `abort()`, CPU metrics, `/evict`.

### celld v0.5.1 limits + blobs / streaming / summary

- Verify `cpuMs` / `subRequests` on the pinned binary (`WorkerCode.limits` /
  `getEntrypoint()`); treat compat-doc “reject” as stale until disproven.
- Contract: ranged / multipart I/O, trusted SHA-256 on publish, bounded `summary jsonb`
  computed by the runner, idle-stream ~60 s keep-alive / windowed reads.
- Multipart abort + orphan GC for abandoned uploads.

### Two-bucket topology

- **Fleet** bucket: celld credentials only; reserved prefixes; not for app bytes.
- **App** bucket: runner credentials (prefer signed URLs in loader `props`).
- Prove cross-deny (fleet creds cannot write app keys and vice versa).

### Auth runner ↔ celld

- Public app listener: short-lived token (workspace / execution / attempt / exp).
- Runner callbacks authenticated.
- Operator listener private (no public ingress).

## 4. Copy / avoid (Cloudflare Workers + Rivet agentOS)

**Copy:** Workspace facade; execution separated from storage; content hashes; explicit “what
persists” table; lazy restore; path confinement; bytes off the control / model path.

**Avoid:** SQLite (DO or per-actor) as authoritative catalog; chunked S3 block stores; full VM /
POSIX root; plaintext credentials in the execution store; fleet-bucket credentials in app code;
Workflows / KV / D1 / Queues as SoR.

## 5. Phase-0 / draft checklist

- [ ] Logical VFS scope + “what persists” table in parent draft
- [ ] Leases, reconciler, attempt fence, `cancelling` semantics
- [ ] Physical termination + recovery after runner/celld loss
- [ ] Block late publish and overwrite of committed bytes (attempt-scoped keys)
- [ ] Verify `cpuMs` / `subRequests` on v0.5.1 binary + heap containment
- [ ] Streaming / ranges, multipart abort / orphan GC, checksums, bounded summaries
- [ ] Two buckets + separated credentials; cross-deny
- [ ] Auth runner ↔ celld; private operator listener
- [ ] Facade contract `WS.read` / `rows` / `readRange` / `write` / `list` / `tool` as swappable
- [ ] Phase-0 wording: no durable product catalog, but recovery experiments allowed
