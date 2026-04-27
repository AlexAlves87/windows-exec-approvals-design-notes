# Evaluation Pipeline

The evaluation pipeline transforms a raw `system.run` request into a typed policy decision. It comprises four submodules in a fixed order. Each submodule is independently testable without UI or gateway dependencies.

---

## 1. Request Validation

**Input:** `ExecHostRequest` (raw agent request)  
**Output:** `ExecHostValidatedRequest` (proof type) or deny (`validation-failed`)

The validator is the only component that reads `rawCommand` from the agent. All other components receive only the validated output.

```csharp
record ExecHostValidatedRequest(
    IReadOnlyList<string> Command,          // trimmed, non-empty argv
    string                DisplayCommand,   // canonical form derived from argv
    string?               EvaluationRawCommand
);
```

**`DisplayCommand`** is always generated from `argv` using safe quoting — never from the agent's `rawCommand`. This is the string shown in the approval dialog.

**`EvaluationRawCommand`** is produced by the validator under specific conditions:

| Scenario | `EvaluationRawCommand` |
|----------|----------------------|
| Shell wrapper + rawCommand == canonical form | `null` (conservative) |
| Shell wrapper + rawCommand == inner payload form | `rawCommand` |
| Shell wrapper + rawCommand == null | `null` |
| Direct exec + rawCommand present and verified | `rawCommand` |
| Direct exec + rawCommand == null | `null` |

**Coherence check:** when the agent sends `rawCommand`, the validator verifies it equals either the canonical form or the accepted inner payload form. Any other value produces `validation-failed`. The validator fails closed — an unrecognized form is not accepted.

**`mustBindDisplayToFullArgv`:** set to `true` when argv contains inline env assignments (`VAR=val cmd`), `env` options that modify the environment (`-u`, `--unset`), or positional argv carriers in the shell payload (`$0`, `$1`, `"$@"`). When `true`, only the canonical full-argv form of `rawCommand` is accepted.

**Shell wrapper detection** uses single-level detection on `argv` only, passing `null` for rawCommand to the wrapper parser. The multi-level BFS expansion of the existing Windows shell wrapper parser is not invoked at this stage.

**`ExecEnvSanitizer`** is a prerequisite, not part of this submodule. The sanitizer runs before validation and blocks dangerous environment variable overrides. The validator does not duplicate this logic.

---

## 2. Executable Resolution

**Input:** `ExecHostValidatedRequest`, cwd, sanitized env  
**Output:** `ExecCommandResolution` or `[]` (fail-closed)

```csharp
readonly record struct ExecCommandResolution {
    string  RawExecutable { get; init; }   // first effective token, trimmed
    string? ResolvedPath  { get; init; }   // best available absolute path; null if undeterminable
    string  ExecutableName{ get; init; }   // basename of ResolvedPath or RawExecutable
    string? Cwd           { get; init; }   // working directory context
}
```

**Three distinct functions:**

`resolve(command, cwd, env)` → `ExecCommandResolution?`  
Singular resolution of the main executable. Used by the state machine and display. Unwraps transparent `env` dispatch (no modifier flags). Returns `null` if the command is empty or unresolvable.

`resolveForAllowlist(command, rawCommand, cwd, env)` → `IReadOnlyList<ExecCommandResolution>`  
Multi-segment resolution for allowlist matching. Detects shell wrappers, splits the command chain, produces one resolution per segment. Fail-closed: any ambiguity, unresolvable segment, or dangerous construct returns `[]`.

`resolveAllowAlwaysPatterns(command, cwd, env)` → `IReadOnlyList<string>`  
Produces pattern suggestions for the "Allow Always" UX. This is a UX helper, not a security decision. Unlike `resolveForAllowlist`, this function unwraps `env` with modifiers to show the inner executable. Deduplicated.

**`ResolvedPath` resolution rules:**

| Input form | `ResolvedPath` |
|------------|---------------|
| Fully-qualified absolute (`Path.IsPathFullyQualified`) | `Path.GetFullPath(input)` — normalizes `.`/`..` |
| Drive-relative (`\Windows\...`) | `Path.GetFullPath(input, effectiveCwd)` — treated as relative |
| Relative (contains separator, not fully-qualified) | `Path.GetFullPath(input, effectiveCwd)` |
| Basename (no separator) | First PATH search hit with PATHEXT probing; attempt 8.3 → long path normalization |
| Basename not in PATH | `null` |

PATH and PATHEXT come from the sanitized env, with fallback to the process environment. This ensures resolution and execution use the same PATH.

**Fail-closed conditions for `resolveForAllowlist`:**

- Command substitution detectable in shell payload (`$(`, `` ` ``, `$\n(`)
- Env manipulation before shell wrapper (`BASH_ENV=`, `env -u`, `env -`)
- Empty wrapper payload
- `-EncodedCommand` / `-enc` / `-ec` in any segment payload position
- Any segment without a valid resolution
- `env -` (dash replaces entire env)

**Symlinks and junctions:** not resolved in v1. Package managers on Windows (Scoop, Volta, nvm-windows) install binaries as shims. Resolving symlinks would produce a `ResolvedPath` to the target, while the allowlist entry was created for the shim path — resulting in spurious allowlist misses.

**8.3 short names:** attempt normalization to long path via `GetLongPathName` for results of PATH search (where file existence is already confirmed). For absolute paths without existence verification, no normalization attempt. Document if deferred.

**Paths with `:` in non-standard positions:** paths where `:` appears outside the volume prefix or UNC root are rejected fail-closed as unsupported path forms.

**`allowAlwaysPatterns`** is produced by this submodule, not by the matcher or state machine. The state machine passes these patterns to prompting as-is.

---

## 3. Allowlist Matching

**Input:** `IReadOnlyList<ExecAllowlistEntry>`, `ExecCommandResolution`  
**Output:** matching entry or `null`; for multi-resolution: `IReadOnlyList<ExecAllowlistEntry>` or `[]`

The matcher compares a resolved executable against allowlist entries. It is a pure function with no side effects, no PATH lookup, and no shell parsing.

**Match target:** `resolvedPath ?? rawExecutable`. Never the full command string.

**`match(entries, resolution)`** → first matching entry or `null`.

**`matchAll(entries, resolutions)`** → one entry per resolution in order, or `[]` if any resolution fails to match. Strict all-or-nothing semantics.

**Glob semantics:**
- `*` — one path segment (does not cross separators); equivalent to `[^/]*`
- `**` — zero or more segments (crosses separators); equivalent to `.*`
- `?` — **not finalized**; preferred behavior is `[^/]` (one character, no separator crossing); this is an open security decision

**Normalization:** separator normalization (`\` → `/`) applied to both pattern and target using the same function. Case-insensitive comparison using `ToLowerInvariant()` (locale-independent, not `ToLower()`).

**Fail-closed validation:** basename-only patterns (no `/`, `\`, or `~`) are invalid. An invalid pattern does not match anything — it never produces allow.

**Allowlist satisfaction:** `allowlistSatisfied` is derived in the `ExecApprovalEvaluation` constructor, not recomputed by callers:
```
AllowlistSatisfied = Security == allowlist
    && AllowlistResolutions.Count > 0
    && AllowlistMatches.Count == AllowlistResolutions.Count
```

---

## 4. State Machine

**Input:** `ExecApprovalEvaluation` context, `ExecApprovalDecision?`  
**Output:** `ExecHostPolicyDecision` (three cases: `deny`, `requiresPrompt`, `allow`)

The state machine is a stateless, pure decision function. It does not read the store, perform I/O, invoke UI, or execute side effects.

```csharp
ExecHostPolicyDecision evaluate(ExecApprovalEvaluation context, ExecApprovalDecision? approvalDecision)
```

**Fixed precedence order:**

| Step | Condition | Outcome |
|------|-----------|---------|
| 1 | `context.Security == deny` | `deny("security-deny")` |
| 2 | `approvalDecision == deny` | `deny("user-denied")` |
| 3 | `requiresAsk(...) && approvalDecision == null` | `requiresPrompt` |
| 4 | `security == allowlist && !allowlistSatisfied && !skillAllow && !approvedByAsk` | `deny("allowlist-miss")` |
| 5 | (default) | `allow(approvedByAsk: approvalDecision != null)` |

This order must not be changed. Step 1 (`security=deny`) is an absolute override — no user decision can bypass it. Step 4 fires only when `ask=off` (otherwise step 3 would have fired first).

**`requiresAsk` helper:**
```
requiresAsk(ask, security, allowlistMatch, skillAllow) → bool
  ask == always                                           → true
  ask == onMiss && security == allowlist
      && allowlistMatch == null && !skillAllow            → true
  otherwise                                               → false
```

**`approvalDecision` parsing:** unknown strings parse to `null` (absence of decision), not to `deny`. Treating unknown as `deny` would block the first pass (which always arrives without a decision) for all requests.

**`ExecApprovalEvaluation` construction:** the coordinator builds the context once before the first pass and reuses it for the second pass. The context is immutable between passes. Derived fields (`AllowlistSatisfied`, `AllowlistMatch`, `Resolution`) are computed in the constructor:

```
AllowlistSatisfied = Security == allowlist
    && AllowlistResolutions.Count > 0
    && AllowlistMatches.Count == AllowlistResolutions.Count

AllowlistMatch = AllowlistSatisfied ? AllowlistMatches.FirstOrDefault() : null

Resolution = AllowlistResolutions.Count > 0 ? AllowlistResolutions[0] : null
```

Computing these in the constructor prevents inconsistencies across callers.

**`skillAllow`:** part of the conceptual model (macOS: `SkillBinsCache.shared.currentTrust()`). Hardcoded to `false` in v1. The parameter is kept in the function signature for future activation.

**`askFallback`:** not an input to the state machine. It is consumed by the Fallback without UI module when `requiresPrompt && !canPresent`.

**Two-pass flow (managed by the coordinator):**
```
pass 1: evaluate(context, null)           → typically requiresPrompt
        → coordinator: prompt user or apply fallback
pass 2: evaluate(context, userDecision)   → allow or deny
```

If pass 2 returns `requiresPrompt`, this is an invariant violation — the coordinator produces `deny(INVALID_REQUEST)` without retry.

---

## Security Notes

**Command identity is canonical.** A single `ExecCommandResolution` is produced during resolution and reused across evaluation, logging, prompting, and execution. No module reinterprets the command independently.

**The evaluation is side-effect-free.** No module in the evaluation pipeline may write to the store, update the allowlist, or record usage. These are deferred to after the final allow decision.

**The state machine never receives raw request input.** `ExecApprovalEvaluation` is built by the coordinator from validated and resolved data. The state machine function receives only the context and the optional approval decision.

**`AllowlistMatch != null` ≠ `AllowlistSatisfied`.** A partial match (some resolutions matched, not all) produces `AllowlistMatches` with entries but `AllowlistSatisfied = false`. The `requiresAsk(onMiss)` check depends on `AllowlistMatch`, which is `null` when `!AllowlistSatisfied`. Getting this wrong would suppress prompts for partially-matched commands.
