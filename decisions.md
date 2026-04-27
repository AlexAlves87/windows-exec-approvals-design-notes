# Key Design Decisions

---

## Decision 1: Conceptual port from macOS, not a literal translation

**Decision:** The macOS exec approvals system is the behavioral reference. The Windows implementation must be idiomatic C#/WinUI, structurally independent, and designed for incremental integration.

**Why:** Swift and C# differ in concurrency model (actors vs. Tasks), type system, and runtime. A literal translation produces unidiomatic C# and imports macOS-specific assumptions that do not hold on Windows — blocking modal dialogs (`NSAlert.runModal()`), Unix socket IPC, POSIX permissions, `CGEventSource` for idle detection. These must be redesigned for the Windows platform.

**Tradeoff:** Semantic parity with macOS is maintained where it matters for correctness (pipeline order, fail-closed rules, canonical identity). Platform-specific mechanisms (async dialogs, Win32 idle detection, NTFS atomic writes) diverge by design.

---

## Decision 2: The existing rule-list model is not replaced

**Decision:** `ExecApprovalPolicy` and `exec-policy.json` remain intact and unmodified. The new model adds a parallel path. No data migration between the two models occurs in the first wave.

**Why:** `ExecApprovalPolicy` is in production use. Replacing it requires migration logic and regression risk. Additive design with temporary coexistence avoids this risk and keeps each increment independently verifiable.

**Tradeoff:** `HandleRunAsync` contains two branches during Phases 1–3. This is intentional, bounded complexity. The cost is clarity in the entry point; the benefit is zero regression risk on existing behavior.

**Alternatives considered:**
- *Extend `ExecApprovalPolicy` with interactive prompt* — rejected. The rule-list model has no concept of per-agent cascade, allowlist tiers, or security dimensions. Retrofitting produces unclear semantics.
- *Replace atomically with migration tooling* — rejected for Phase 1. Blocks the first verifiable increment on migration infrastructure that is not yet designed.

---

## Decision 3: Explicit feature flag activation

**Decision:** The new path is activated only by `ExecApprovalsNewPathEnabled: true` in `settings.json`. File presence, class presence, or store presence do not activate it. Default is `false`.

**Why:** The store file may exist from machine sync, backup, or prior testing. Activation must be an operator decision, not a filesystem side effect.

**Fail-closed behavior:**
```
if ExecApprovalsNewPathEnabled == true:
    if coordinator available:
        coordinator.HandleAsync(request)
    else:
        deny(code: "unavailable")   ← never fall back to legacy
else:
    legacy path
```

If the coordinator throws an unhandled exception during a request: deny. No retry via legacy.

**Alternatives considered:**
- *Activate by presence of `exec-approvals.json`* — rejected. File may exist without deliberate intent.
- *Activate by code presence (always-on new path)* — rejected. Prevents independent verification of each increment.

---

## Decision 4: Fail-closed in all error paths of the new path

**Decision:** When the new path is active, any failure produces an explicit deny. There is no fallback to the legacy path.

**Why:** The operator who enables the new path does so deliberately. A failure that silently degrades to the legacy path is more dangerous than an explicit deny — it removes the operator's ability to observe that the new path failed, and it may bypass policy the operator intended to enforce.

**Critical dependencies that fail-close:**
- Store unavailable → deny
- Coordinator not initialized → deny (`unavailable`)
- Validator cannot process request → deny
- `resolveReadOnly` fails → deny
- Prompt required but `canPresent = false` and `askFallback` resolves to deny → deny

**Alternatives considered:**
- *Degrade to legacy on new-path failure* — rejected. Silent degradation is more dangerous than explicit deny when the operator has made a deliberate activation decision.

---

## Decision 5: Evaluation is decomposed into four independent submodules

**Decision:** The evaluation module is not a single component. It comprises four independently-designed and independently-testable submodules: request validation, executable resolution, allowlist matching, and state machine.

**Why:** Request validation addresses a security boundary with no equivalent in the current Windows codebase (`rawCommand` coherence check). Treating evaluation as a block would obscure this gap. Each submodule has a different risk profile, different test surface, and different dependency requirements.

**Tradeoff:** More design work upfront. The benefit is that security-critical components (request validation, allowlist matching) receive explicit design attention rather than being absorbed into a monolithic evaluator.

---

## Decision 6: `rawCommand` from the agent is untrusted

**Decision:** The `rawCommand` field sent by an agent is treated as untrusted input. It is never used directly for display, allowlist matching, or as input to execution. The request validator is the only component that reads it, and only to verify coherence with `argv`. All display text is derived from `argv`.

**Why:** An agent can construct a `rawCommand` that differs from what will actually execute (`argv`). If `rawCommand` is shown in an approval dialog, the user approves what they see — not what runs. This is a visual prompt injection vector.

**Implementation:** The validator produces a `displayCommand` from `argv` using safe quoting. The prompt presenter receives only `displayCommand`. The type system (`ExecHostValidatedRequest`) prevents `rawCommand` from reaching downstream components.

---

## Decision 7: Executable resolution precedes allowlist matching

**Decision:** The allowlist matcher operates on resolved executable paths, not on command strings. The match target is `resolvedPath ?? rawExecutable`. Basename-only patterns are invalid and fail-closed.

**Why:** Matching on command strings (as the legacy model does) is insufficient for a path-based allowlist: the same string can resolve to different executables depending on PATH, PATHEXT, and working directory. Resolution first ensures that the allowlist pattern applies to the actual executable that will run.

**Windows-specific considerations:**
- Path separator normalization (`\` → `/`) applied to both pattern and target
- Case-insensitive matching with `ToLowerInvariant()` (locale-independent)
- PATHEXT probing for basenames; resolved path includes the extension
- No symlink/junction resolution in v1 (consistent with macOS; shim-based package managers require this)
- Paths with `:` in non-volume-prefix positions are rejected as unsupported path forms

**Deliberately deferred:** The `?` glob wildcard behavior on Windows is an open security decision. On macOS, `?` matches any single character including path separators. On Windows, that behavior would allow cross-directory matches that could widen the allowlist surface unexpectedly. The preferred Windows behavior is `? → [^/]` (single character, no separator crossing). This is not left unresolved by oversight — it requires an explicit decision at implementation time because it is a security boundary, not a usability detail.

---

## Decision 8: `displayCommand` is always sanitized before rendering

**Decision:** Before any text derived from `displayCommand` is rendered in the approval dialog, it passes through `ExecApprovalCommandDisplaySanitizer`. This function escapes codepoints in the Unicode `Cf` (Format) category and four additional Hangul filler codepoints that are visually invisible but not classified as `Cf`.

**Why:** Invisible Unicode characters — BiDi overrides, zero-width spaces, Hangul fillers — can make a displayed command look different from what it contains. In an approval dialog, this is a direct attack surface: the user approves a visual representation that does not match the actual command string.

**Escape format:** `\u{XXXX}` (hex uppercase, no zero-padding), consistent with macOS. The sanitizer escapes to visible text — it does not strip. Stripping would hide that invisible content was present.

**Sanitizer scope:** Applied only in the display path. The command that executes is always `validatedRequest.Command` (the argv IReadOnlyList), which is structurally separate from any display string.

---

## Decision 9: Store uses atomic writes

**Decision:** Writes to `exec-approvals.json` use write-to-temp + `File.Move(tmp, dest, overwrite: true)`. `File.WriteAllText` is not used.

**Why:** `File.WriteAllText` is not atomic. If the process dies mid-write, the file is left in a partial state. The `File.Move` pattern with `overwrite: true` on NTFS uses `MoveFileExW` with `MOVEFILE_REPLACE_EXISTING`, which is atomic at the MFT level.

**Tradeoff:** `File.Move` can fail transiently if an external process (antivirus, indexer) holds the file open. The implementation must handle `IOException` on the move with bounded retry or controlled degradation.

---

## Decision 10: Phase 1 serializes coordinator access to shared mutable state

**Decision:** Concurrent `system.run` requests do not execute concurrently through the new coordinator in Phase 1. Access to shared mutable state (store, allowlist cache) is serialized using `SemaphoreSlim(1,1)`.

**Why:** Concurrent execution without coordination risks inconsistent evaluation results and shared state corruption. Phase 1 takes the conservative position for correctness. Moving to a concurrent or lock-free model is an explicit Phase 2+ decision.

**Tradeoff:** Throughput of concurrent requests is limited in Phase 1. This is acceptable given that exec approvals involve user interaction (prompts) which already serializes naturally in practice.

---

## Decision 11: Gateway integration and Settings UI are not Phase 1 prerequisites

**Decision:** The coordinator handles the local path (direct prompting, `askFallback`) without a gateway roundtrip. `exec-approvals.json` can be edited manually. Both gateway integration and Settings UI are Phase 2+ work.

**Why:** Blocking Phase 1 on Phase 2 delays the first end-to-end verifiable increment without a security benefit. The local path is a complete, meaningful capability on its own.

---

## Decision 12: `lastUsedAt` is stored as `double?` (Unix epoch ms)

**Decision:** The `lastUsedAt` field in allowlist entries uses `double?` in C#, matching the macOS representation (`Double`, Unix epoch in milliseconds).

**Why:** Diverging to `long?` produces a different JSON number type from macOS without functional benefit. The precision of `double` (IEEE 754, exact integers up to 2^53) covers Unix epoch milliseconds until year 2286 without loss.

---

## Risk Register

| Risk | Severity | Mitigation |
|------|----------|-----------|
| `allowlistSatisfied` computed inconsistently across callers | High | Derive `AllowlistSatisfied`, `AllowlistMatch`, and `Resolution` in the constructor of `ExecApprovalEvaluation`, not in callers |
| Evaluation precedence order reordered | High | Fixed five-step order: security-deny → user-denied → requiresAsk → allowlist-miss → allow. Not reordered. |
| `rawCommand` reaching display or allowlist | High | `ExecHostValidatedRequest` does not expose `rawCommand`; presenter receives only `displayCommand` |
| Sanitizer bypassed in display path | High | Sanitizer applied in code-behind before any bind; not delegated to ViewModel or XAML |
| Store write corruption on crash | Medium | Atomic write pattern; `File.Move` with NTFS guarantees |
| `canPresent` incorrect under lock screen | Medium | `GetLastInputInfo` alone is insufficient; desktop interactivity must be checked independently |
| Concurrent requests produce duplicate prompts | Medium | Coordinator serializes in Phase 1; concurrent policy is explicit Phase 2 decision |
| `approvalDecision` unknown string treated as `deny` | Medium | Unknown values parse to `null` (absence of decision), not to `deny`; `null` is handled by `requiresAsk` |
| Side effects triggered before final decision | Medium | Pipeline order enforced: side effects are step 8, after final allow decision |
| 8.3 short path names bypass allowlist matching | Low | Attempt long-path normalization via `GetLongPathName` for resolved basenames; document if deferred |
