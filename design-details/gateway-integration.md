# Gateway Integration

Gateway integration connects the WebSocket infrastructure with the exec approvals coordinator. It has two independent sides with distinct responsibilities.

---

## Architecture: Two Sides

### Operator side (app role)

The app acts as an operator: it receives `exec.approval.requested` events from the gateway, presents the approval UI (or applies fallback), and sends `exec.approval.resolve` with the decision.

**Component:** `OpenClawGatewayClient` (the component managing the app-side gateway connection). Not `WindowsNodeClient`.

**Flow:**
```
exec.approval.requested {id, request, createdAtMs, expiresAtMs}
  → expiry check: if now >= expiresAtMs → send resolve(id, deny) → return
  → deduplication: if id already seen → return
  → canPresent evaluation
      canPresent = true  → IExecApprovalPromptHandler.PromptAsync(request)
      canPresent = false → FallbackDecision(request, askFallback, allowlist)
  → send exec.approval.resolve(id, decision)
```

The `finally` block always sends `exec.approval.resolve` — even on exception. Leaving the gateway waiting indefinitely is not acceptable:

```
try   { obtain decision (prompt or fallback) }
catch { decision = deny }
finally { send exec.approval.resolve(id, decision) }
```

### Node side (node role)

The node receives `system.run` params that may include pre-resolved `approvalDecision` and `approved` fields (injected by the gateway after the operator completes the approval cycle).

**Component:** `WindowsNodeClient` / `HandleRunAsync`

**Parsing:**
```
approvalDecision: string?   // "allowOnce" | "allowAlways" | "deny" | unknown
approved:         bool?     // legacy compatibility field
```

Unknown values for `approvalDecision` parse to `null` (absence of decision) — not to `deny`. The semantic meaning of the decision is resolved by the coordinator and state machine, not at the parsing layer.

---

## Protocol Types

**`GatewayApprovalRequest`:**
```csharp
record GatewayApprovalRequest(
    string                  Id,
    ExecApprovalPromptRequest Request,
    long                    CreatedAtMs,
    long                    ExpiresAtMs
);
```

**`ExecApprovalPromptRequest`** (8 fields):
```
displayCommand:  string     // derived from argv, sanitized before display
cwd:             string?
host:            string?
security:        ExecSecurity
ask:             ExecAsk
agentId:         string
resolvedPath:    string?
sessionKey:      string?
```

Note: `allowAlwaysPatterns` is not in this struct. Those patterns come from `ExecApprovalEvaluation` and are consumed by the coordinator when persisting an `AllowAlways` decision. The presenter does not need them.

---

## Expiry and Deduplication

**Expiry check** occurs before any UI or fallback invocation. A request past its `expiresAtMs` is denied immediately and the result is sent as `exec.approval.resolve`. The user never sees a dialog for an already-expired request.

**Deduplication** by `id` prevents duplicate prompts from gateway retries or rebroadcasts. A set of seen IDs is maintained; any request with a previously-seen ID is discarded. The cleanup strategy for this set (TTL-based, circular buffer, etc.) is an implementation detail — the requirement that deduplication exists is fixed.

---

## Exec Events

The coordinator emits three events to the gateway after processing a request:

| Event | When | Key fields |
|-------|------|-----------|
| `exec.denied` | Any deny outcome | `sessionKey`, `runId`, `host`, `command`, `reason` |
| `exec.started` | Immediately before execution | `sessionKey`, `runId`, `host`, `command` |
| `exec.finished` | After execution completes | + `exitCode`, `timedOut`, `success`, `output` (truncated) |

`reason` in `exec.denied` maps to the typed deny codes: `security-deny`, `allowlist-miss`, `user-denied`.

These events are emitted from `SystemCapability` (or the coordinator), not from the transport layer. The transport layer provides the channel; the capability decides what to emit and when.

---

## Boundary with the Coordinator

The gateway integration layer and the coordinator are decoupled by design:

- The coordinator exposes an `approvalDecision: ExecApprovalDecision?` input field. If pre-resolved, the first evaluation pass may produce `allow` directly without entering the prompt/fallback branch.
- The coordinator is agnostic to the source of `approvalDecision` — whether it came from the gateway roundtrip or from a local prompt.
- The coordinator is the only component that calls `AddAllowlistEntry` or `RecordAllowlistUse`. The gateway layer does not call these.

---

## Open Design Points

The following points are deliberately surfaced as open. None of them block the core architecture or the Phase 1 implementation. Each is scoped to Phase 2 or later, or requires verification against platform or protocol behavior rather than a design decision that can be made in the abstract.

**Idle detection API.** `canPresent` on macOS uses `CGEventSource.secondsSinceLastEventType`. The Windows equivalent (`GetLastInputInfo()`) alone is insufficient — it does not detect a locked screen or a disconnected RDP session. Additional Win32 calls are needed: `OpenInputDesktop` + `GetUserObjectInformation`, `WTSQuerySessionInformation`, or `GetSystemMetrics(SM_REMOTESESSION)`. The required behavior is: fail-closed when desktop interactivity cannot be confirmed.

**Gateway ordering guarantee.** The protocol should guarantee that `exec.approval.resolve` reaches the gateway before the gateway injects `approvalDecision` into a `system.run` replay. If this ordering is not guaranteed, the coordinator may receive a stale or mismatched `approvalDecision`. This should be verified against the gateway protocol specification.

**Behavior when no operator is connected.** If the app is not active in the operator role, `exec.approval.requested` has no recipient. The fallback behavior (gateway timeout, automatic deny, or request retention) should be specified.

**Deduplication set cleanup.** Without cleanup, the seen-IDs set grows without bound during long sessions. A reasonable strategy: expire entries whose `expiresAtMs` is in the past on each insert, or use a bounded circular structure.

**`system.run.prepare` as a pre-approval stage.** The current protocol sends `system.run` with `approvalDecision` injected after the approval cycle completes. An alternative design (retaining the command at the gateway until approval, then releasing it) could simplify correlation but requires protocol changes. Not in scope for Phase 2.
