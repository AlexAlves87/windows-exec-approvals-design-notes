# Incremental Implementation Plan

This document describes a proposed incremental structure for introducing the new exec approvals system. It is a design map — a dependency ordering and phase breakdown — not a changelog or a delivery schedule. Each phase is defined by what it makes possible, not by when it happens.

The legacy approval path remains active and unchanged throughout all phases.

---

## Phase Structure

### Phase 1 — Local approval path (foundation)

Phase 1 is the minimum scope needed for the new path to be activatable and verifiable end-to-end. No gateway roundtrip is needed. No Settings UI is needed. Configuration at this phase is done manually via `exec-approvals.json`.

**What Phase 1 includes:**

- Routing seam in `HandleRunAsync` (feature flag, coordinator injection point)
- New store (`ExecApprovalsStore` / `exec-approvals.json`): read and write paths, atomic writes, cascade resolution
- Full evaluation pipeline: request validation, executable resolution, allowlist matching, state machine
- Coordinator (`ExecApprovalsCoordinator`): two-pass evaluation, prompt/fallback branch, side effects, final execution
- Local prompt dialog (`ExecApprovalDialog`, `IExecApprovalPromptHandler`)
- Fallback without UI (`canPresent`, `askFallback` policy)
- Minimum observability: structured logging per request (path, decision, reason code, correlation ID)
- Phase 1 concurrency: serialized coordinator access to shared mutable state

**What Phase 1 does not include:**

- Gateway approval cycle (`exec.approval.requested` / `exec.approval.resolve`)
- Settings UI
- `autoAllowSkills` / skill trust model

At the end of Phase 1, the new path is functional end-to-end for local requests. The feature flag defaults to off; the operator enables it deliberately. Disabling it restores the legacy path with no file migration required.

---

### Phase 2 — Gateway integration

Phase 2 adds the roundtrip approval cycle over the gateway WebSocket.

**What Phase 2 includes:**

- Operator side: handler for `exec.approval.requested` in the gateway client component; sender for `exec.approval.resolve`; expiry check; deduplication by request ID
- Node side: parsing of `approvalDecision` and `approved` fields in `system.run` params; delivery to coordinator
- Emission of `exec.denied`, `exec.started`, `exec.finished` events

At the end of Phase 2, the full remote approval cycle is functional: a remote operator can approve or deny a `system.run` command via the gateway before it executes.

---

### Phase 3 — Settings UI

Phase 3 adds a configuration window for the new store.

**What Phase 3 includes:**

- `ExecApprovalsSettingsWindow` as a separate `WindowEx`
- Editable fields: `security`, `ask`, `askFallback` for defaults and per-agent overrides
- Editable allowlist with pattern validation (path patterns only; basename-only rejected)
- Read-only display of usage metadata (`lastUsedAt`, `lastUsedCommand`, `lastResolvedPath`)
- Entry point from the ADVANCED section of the existing Settings window
- Deferred save/cancel semantics (not reactive-immediate)

---

### Phase 4 — Legacy cleanup (future, not planned)

Removal of `ExecApprovalPolicy` and `exec-policy.json`. Requires migration tooling and explicit activation planning. Not in scope for Phases 1–3.

---

## Phase 1 — Implementation Sequence

Phase 1 is itself broken into a fixed dependency sequence. Each step is independently buildable and leaves the legacy path untouched.

### Step 1 — Routing seam (inert)

Introduce the routing fork in `HandleRunAsync`. When no new-path coordinator is injected, the legacy path runs unchanged. When a coordinator is injected, calls route to a minimal stub that returns `unavailable`. The new path is inert by default — the stub is never wired in production at this step.

The seam created here (`IExecApprovalV2Handler`, routing in `HandleRunAsync`) is stable and not revisited by any subsequent step. Observability fields established here: correlation ID, selected path, decision, reason code.

**Depends on:** nothing  
**Unlocks:** all subsequent steps implement pipeline components or wire them into this seam

---

### Step 2 — Input validation

Implement the validate-input phase as a standalone, UI-free component. Input: raw request args. Output: typed result (`validation-failed` or `ExecHostValidatedRequest`). No resolution, no evaluation.

**Depends on:** Step 1 (typed result contract)  
**Unlocks:** Step 3

---

### Step 3 — Normalization, resolution, canonical identity

Implement: normalize command form (detect shell wrappers), resolve executable to full path, build `ExecCommandResolution`. Typed deny (`resolution-failed`) on ambiguity or unresolvable input. Canonical identity is a boundary — nothing downstream may reinterpret the command independently.

**Depends on:** Step 2  
**Unlocks:** Step 4

---

### Step 4 — Store read path

Implement `ExecApprovalsStore` read path: load `exec-approvals.json`, apply cascade resolution for a given agent, return `ExecApprovalsResolved`. Malformed JSON or unsupported version → emit structured warning, apply default-deny. No write path yet.

**Depends on:** Step 3 (canonical identity needed for allowlist key lookups)  
**Unlocks:** Step 5

---

### Step 5 — Evaluator

Implement the state machine: given `ExecApprovalEvaluation` context and optional `ExecApprovalDecision`, produce `allow` / `deny` / `requiresPrompt`. Deterministic and side-effect-free. The `ExecApprovalEvaluation` context is built by the coordinator (Step 7) — this step implements only the pure decision function.

**Depends on:** Steps 3, 4  
**Unlocks:** Step 6

---

### Step 6 — Prompt adapter interface and stub

Define `IExecApprovalPromptHandler`. Add a stub implementation (always returns `user-denied`). No WinUI. The prompt is only invoked after canonical identity exists and evaluation produced `requiresPrompt`.

**Depends on:** Step 5  
**Unlocks:** Step 7

---

### Step 7 — Coordinator implementation (not yet wired in production)

Implement `ExecApprovalsCoordinator` behind the existing `IExecApprovalV2Handler` interface. Wire the full pipeline: validate → normalize → resolve → canonical identity → evaluate (pass 1) → prompt/fallback → evaluate (pass 2) → side effects. Full observability. Write-path side effects are stubbed. The coordinator is tested in isolation but not injected in production — the null handler from Step 1 remains active.

**Depends on:** Steps 2, 3, 4, 5, 6  
**Unlocks:** Step 8

---

### Step 8 — Wire coordinator into production (feature flag off by default)

Inject `ExecApprovalsCoordinator` into production via the wiring point established in Step 1. The feature flag (`ExecApprovalsNewPathEnabled`) remains `false` by default. The new path is reachable but not active unless deliberately enabled.

**Depends on:** Step 7  
**Unlocks:** Step 9

---

### Step 9 — Store write path and side effects

Add the write path to the store. Wire side effects in the coordinator: allowlist entry persistence and usage tracking, both conditional on a final allow decision. This closes the persistence loop — the new path is fully functional end-to-end for local requests.

**Depends on:** Step 8  
**Unlocks:** Phase 2 (gateway integration), Phase 3 (Settings UI)

---

## Dependency Graph (Phase 1)

```
Step 1 ─── Step 2 ─── Step 3 ─── Step 4 ──┐
                                            ├── Step 5 ─── Step 6 ─── Step 7 ─── Step 8 ─── Step 9
                               Step 3 ──────┘
```

---

## Activation Design

The new path is controlled by a single boolean flag (`ExecApprovalsNewPathEnabled`) in the application settings. This flag defaults to `false`. Enabling it routes `system.run` requests through the new coordinator; disabling it restores the legacy path immediately, with no file migration.

The design intent is that activation is always an explicit operator decision. The presence of `exec-approvals.json` on disk, or the presence of the new coordinator classes in the build, does not activate the new path on its own.

If the new path is active and the coordinator fails for any reason, the response is a typed deny with code `unavailable` — not a silent fallback to legacy. This is a deliberate fail-closed design (see `decisions.md`, Decision 4).

Structured logs include a correlation ID per request, the selected path, the decision, and the reason code. These are the primary diagnostic surface for any operational issue with the new path.
