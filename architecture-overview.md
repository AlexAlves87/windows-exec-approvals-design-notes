# Architecture Overview

## Goals

- Introduce a per-agent, tiered approval model for `system.run` execution on Windows.
- Preserve the existing rule-list behavior (`ExecApprovalPolicy`) unchanged and active by default.
- Enable incremental, independently-verifiable introduction of each new component.
- Maintain a fail-closed posture: any failure in the new path denies, not permits.

---

## Legacy Path and New Seam

The entry point for `system.run` execution is `SystemCapability.HandleRunAsync`. This method currently runs the legacy rule-list check and then executes the command.

The new design introduces a routing branch at this point:

```
HandleRunAsync
  ├── if ExecApprovalsNewPathEnabled == true
  │     └── ExecApprovalsCoordinator.HandleAsync(request)
  │           (fail-closed if coordinator unavailable)
  └── else
        └── legacy path (ExecApprovalPolicy, unchanged)
```

This routing point is introduced once and never modified by subsequent changes. The new path is inert by default: the feature flag is `false` until the operator enables it explicitly. The legacy path is never affected by changes to the new system.

If the new path is active and the coordinator is unavailable, the response is a typed error (`code: "unavailable"`) — not a silent fallback to legacy.

---

## Module Map

The system is organized into eight logical modules:

| Module | Responsibility |
|--------|---------------|
| Domain / Contracts | Enums, types, and contracts shared across all modules |
| Persistence | `exec-approvals.json` store: schema, atomic writes, cascade resolution |
| Evaluation | Four submodules — see below |
| Prompting | Dialog presenter and `IExecApprovalPromptHandler` interface |
| Fallback without UI | `canPresent` determination and `askFallback` policy |
| Execution integration | End-to-end coordinator: orchestrates all modules into a single flow |
| Gateway integration | Transport: `exec.approval.requested` / `exec.approval.resolve` cycle |
| Settings UI | `ExecApprovalsSettingsWindow` for configuring the store |

**Evaluation** is treated as four distinct submodules, each independently testable:

| Submodule | Responsibility |
|-----------|---------------|
| Request validation | Transforms raw request into a typed validated request; verifies `rawCommand` coherence |
| Executable resolution | Produces `ExecCommandResolution` from argv; handles PATH, PATHEXT, shell chains |
| Allowlist matching | Compares resolved executables against `ExecAllowlistEntry` patterns |
| State machine | Pure decision function: given evaluation context and optional approval decision, produces `allow`, `deny`, or `requiresPrompt` |

---

## Configuration Model

The new model uses a per-agent cascade:

```json
{
  "version": 1,
  "defaults": { "security": "deny", "ask": "on-miss", "askFallback": "deny" },
  "agents": {
    "main": { "security": "allowlist", "ask": "on-miss", "allowlist": [...] },
    "*":    { "security": "deny" }
  }
}
```

**`security`** — the tier for this agent:
- `deny` — unconditional deny, no prompt possible
- `allowlist` — allow if the executable matches the allowlist; prompt on miss (subject to `ask`)
- `full` — allow all commands without matching

**`ask`** — when to prompt the user:
- `on-miss` — prompt if `security=allowlist` and no allowlist match
- `always` — always prompt regardless of allowlist
- `off` — never prompt; allowlist miss → silent deny

**`askFallback`** — decision when the UI cannot be presented (screen locked, no active session):
- `deny` (default) — deny the command
- `allowlist` — allow if in allowlist, deny otherwise
- `full` — allow without prompting

Field resolution uses a four-level cascade: agent entry → wildcard (`*`) entry → defaults → hardcoded system defaults.

---

## Evaluation Pipeline

Every `system.run` request through the new path follows this fixed sequence:

```
1. Validate input            → ExecHostValidatedRequest   (or deny: validation-failed)
2. Normalize command form    → detect shell wrappers
3. Resolve executable        → ExecCommandResolution      (or deny: resolution-failed)
4. Build canonical identity  → ExecApprovalEvaluation
5. Evaluate (pass 1)         → allow / deny / requiresPrompt
6. Prompt or fallback        → only if requiresPrompt; produces ExecApprovalDecision
7. Evaluate (pass 2)         → allow / deny
8. Side effects              → allowlist write, usage tracking (after final allow only)
9. Execute                   → run command with validated argv and sanitized env
```

A typed deny at any step terminates the pipeline. Side effects never precede the final decision.

The evaluation context (`ExecApprovalEvaluation`) is built once and reused across both passes. The state machine is stateless; the coordinator manages the two-pass flow.

---

## Typed Outcomes

Every deny from the new path carries a stable code:

| Code | Meaning |
|------|---------|
| `unavailable` | Coordinator not ready; check initialization |
| `security-deny` | Agent's `security` is `deny` — unconditional |
| `allowlist-miss` | `security=allowlist`, `ask=off`, no allowlist match |
| `user-denied` | User explicitly denied the prompt |
| `validation-failed` | Request structure failed validation |
| `resolution-failed` | Executable could not be resolved unambiguously |

These codes are machine-readable and logged with every request.

---

## Observability

Every request handled by the new path must log:
- Correlation identifier
- Selected path (new or legacy)
- Canonical command identity
- Decision and reason code
- Whether fallback was applied

This is a requirement for Phase 1, not an optional addition.

---

## Design Invariants

These constraints apply to every component in the new path:

**No silent fallback.** If the new path is enabled, any failure produces a typed deny. It never silently falls back to the legacy path.

**Routing stays thin.** `HandleRunAsync` selects the path but contains no approval logic. All decision logic lives in the coordinator and its dependencies.

**Legacy isolation.** New code must not mutate, reinterpret, or extend `ExecApprovalPolicy`, `exec-policy.json`, or any current legacy type.

**Separate storage.** The new path uses `exec-approvals.json` — a distinct file with a distinct schema. The two files never interact.

**Resolution precedes approval.** The executable must be validated and resolved before any allowlist check or user prompt. Raw shell text is never the source of truth for an approval decision.

**No guessing on ambiguity.** If argv parsing, wrapper expansion, or executable resolution is ambiguous, the result is a typed deny.

**No self-healing store.** Malformed JSON or an unsupported `version` in `exec-approvals.json` must emit a structured warning and apply default-deny until the file is corrected.

**UI-free core.** Validation, resolution, matching, persistence, and the state machine are testable without WinUI. No WinUI types appear in the evaluation pipeline.

**Prompting is an adapter.** The prompt UI sits behind `IExecApprovalPromptHandler`. The coordinator depends on the interface, not the implementation.

**Evaluation is deterministic.** Given the same canonical input, store state, and configuration, the evaluation result is identical across runs.

**Canonical command identity.** A single canonical representation of the command (resolved executable + argv) is produced during validation and reused across evaluation, logging, prompting, and execution. No module reinterprets the command independently.

**Evaluation is side-effect-free.** The evaluation phase does not mutate persistent or shared state. Side effects are permitted only after a final allow decision.

**Concurrency safety.** Concurrent requests must not cause inconsistent evaluation results or corrupt shared state. In Phase 1, access to shared mutable state is serialized.

**Pipeline order is mandatory.** The nine-step pipeline above is fixed. No step may be skipped or reordered.

---

## Key Type Contracts

**`ExecCommandResolution`** — immutable record produced by executable resolution:
```csharp
readonly record struct ExecCommandResolution {
    string  RawExecutable { get; init; }   // first effective token, trimmed
    string? ResolvedPath  { get; init; }   // absolute path if determinable; null otherwise
    string  ExecutableName{ get; init; }   // basename of ResolvedPath or RawExecutable
    string? Cwd           { get; init; }   // working directory context
}
```

**`ExecHostValidatedRequest`** — proof type produced by request validation:
```csharp
record ExecHostValidatedRequest(
    IReadOnlyList<string> Command,          // cleaned argv
    string                DisplayCommand,   // canonical display string, never rawCommand
    string?               EvaluationRawCommand
);
```

**`ExecHostPolicyDecision`** — three-case outcome of the state machine:
```
deny(error: ExecHostError { code, reason })
requiresPrompt
allow(approvedByAsk: bool)
```

**`IExecApprovalPromptHandler`** — interface for the prompt presenter:
```csharp
interface IExecApprovalPromptHandler {
    Task<ExecApprovalDecision> PromptAsync(ExecApprovalPromptRequest request);
}
```

---

## Security Boundaries

**`rawCommand` from the agent is untrusted.** It is never used directly for display, allowlist matching, or execution. The validator is the only component that reads it, and only to verify coherence with argv. The display string shown to the user (`displayCommand`) is always generated from argv, never from `rawCommand`.

**Display text is sanitized before rendering.** Before any text derived from `displayCommand` is shown in the approval dialog, it passes through a Unicode sanitizer (`ExecApprovalCommandDisplaySanitizer`) that escapes invisible and format characters (Unicode `Cf` category + four Hangul filler codepoints). This prevents visual prompt injection attacks.

**Env is sanitized before use.** `ExecEnvSanitizer` blocks dangerous environment variable overrides (PATH, PATHEXT, BASH_ENV, LD_*, and others) before the request reaches the evaluation pipeline. The sanitized env is the source for both executable resolution and command execution.

**Executed command is always `validatedRequest.Command`.** The coordinator executes the argv produced by the validator, with the env produced by the sanitizer. No other source is used.
