# Persistence Design

## File

The new store uses `exec-approvals.json` at `%LOCALAPPDATA%\OpenClawTray\exec-approvals.json`. This is separate from the legacy `exec-policy.json` and from `settings.json`. The two approval files coexist without interference.

## Schema

```json
{
  "version": 1,
  "socket": { "path": null, "token": null },
  "defaults": {
    "security": "deny",
    "ask": "on-miss",
    "askFallback": "deny",
    "autoAllowSkills": false
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "allowlist": [
        {
          "id": "3F2A1B00-...",
          "pattern": "C:/Windows/System32/rg.exe",
          "lastUsedAt": 1714000000000.0,
          "lastUsedCommand": "rg foo",
          "lastResolvedPath": "C:/Windows/System32/rg.exe"
        }
      ]
    },
    "*": { "security": "deny" }
  }
}
```

All fields in `defaults` and per-agent entries are optional. `agents` itself is optional.

## JSON serialization

- `JsonNamingPolicy.CamelCase` for properties
- `JsonStringEnumConverter(JsonNamingPolicy.KebabCaseLower)` for enums — produces `"deny"`, `"on-miss"`, `"allow-once"`, etc.
- `DefaultIgnoreCondition.WhenWritingNull` to omit null fields
- `WriteIndented = true` for human readability
- Separate `JsonSerializerOptions` instance — not shared with the legacy policy serializer

## Version handling

If `version != 1` when loading: emit a structured warning log (file path + error description) and apply default-deny for all requests until the file is corrected. The store does not self-heal; a file with an unsupported version is not silently upgraded.

## `socket` field

The `socket` field is present in the schema for structural compatibility with the macOS store format. On Windows, `socket.path` and `socket.token` are not auto-generated at store initialization. The field is preserved if it already exists in the file; its functional use is deferred to the gateway integration phase.

## `lastUsedAt` type

`double?` in C#, JSON number. Stores Unix epoch in milliseconds (matching macOS). `double` (IEEE 754) represents exact integers up to 2^53, which covers millisecond timestamps until year 2286.

## Cascade resolution

For a given `agentId`, the resolved config is:

```
field_value = agents[agentId].field
           ?? agents["*"].field
           ?? defaults.field
           ?? hardcoded_system_default
```

Hardcoded system defaults:
- `security` → `deny`
- `ask` → `onMiss`
- `askFallback` → `deny`
- `autoAllowSkills` → `false`

`agentId` null or empty resolves to `"main"` internally.

The resolved allowlist is: `wildcardEntry.allowlist + agentEntry.allowlist`, normalized with `dropInvalid: true`.

## Two resolution APIs

**`resolveReadOnly(agentId)`** — loads the file without side effects. Used by the evaluation pipeline. Never writes.

**`resolve(agentId)`** — calls `ensureFile()` internally, which may create the file, initialize the `agents` map, and write. Used during startup or Settings UI operations. Not used by the evaluation pipeline.

The evaluation pipeline always calls `resolveReadOnly`. Using `resolve` in the evaluation path would generate unnecessary writes on every request.

## Normalization on load

When loading, the store normalizes the file content:

- Trim `socket.path` and `socket.token`; set to null if empty after trim
- Migrate `agents["default"]` → `agents["main"]` with merge; existing `"main"` fields take precedence
- Normalize allowlist entries per agent: keep invalid non-empty entries, discard completely empty entries
- Force `version: 1` in the normalized output

## Atomic write

```csharp
var tmp = Path.Combine(dir, $".exec-approvals-{Guid.NewGuid():N}.tmp");
await File.WriteAllTextAsync(tmp, json);
File.Move(tmp, destPath, overwrite: true);
```

`File.Move` with `overwrite: true` on NTFS uses `MoveFileExW` with `MOVEFILE_REPLACE_EXISTING`, which is atomic at the MFT level. Handle `IOException` on the move with bounded retry; do not leave `tmp` files on failure.

## Concurrency

`SemaphoreSlim(1, 1)` serializes write operations (the `updateFile` pattern: ensureFile → mutate → saveFile). This protects intra-process coordination. It does not protect against simultaneous writes from multiple processes (e.g., tray and CLI) accessing the same file. Cross-process coordination is a design point for future implementation.

## Snapshot / hash (optional)

For skip-rewrite-if-unchanged: compute SHA256 of the JSON before and after normalization; only write if the hash changed. Useful in `ensureFile` to avoid unnecessary disk writes. Hash parity with macOS is not required.

## Allowlist entries

Each `ExecAllowlistEntry` has a stable `id` (GUID), a `pattern` (path pattern for allowlist matching), and optional usage metadata:

```csharp
record ExecAllowlistEntry(
    Guid    Id,
    string  Pattern,
    double? LastUsedAt,       // Unix epoch ms
    string? LastUsedCommand,
    string? LastResolvedPath
);
```

The `id` is immutable once created. Patterns are validated before insertion — basename-only patterns are rejected.

## Legacy coexistence

`exec-policy.json` and `exec-approvals.json` coexist without interaction. The new store never reads or writes `exec-policy.json`. The legacy `ExecApprovalPolicy` type is untouched. No automatic data migration occurs.
