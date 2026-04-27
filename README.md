# Exec Approvals for Windows — Design Notes

## About this package

This is a curated, standalone version of a larger body of design and investigation work on porting an exec approvals system from macOS to Windows. It is not a raw dump of working notes. The goal was to extract and preserve the technical direction, key decisions, and incremental structure in a form that can be read, shared, and used independently of the context in which the work was done.

The package covers design, not implementation status. Some parts describe decisions that have been fully worked out; others describe components that are designed but not yet built, or that are deliberately deferred. The documents should be read as a **design and implementation map** — a basis for continuing the work, not a description of a finished system.

---

## Document Status

**Type:** Architecture and design package  
**Not:** Product documentation, release notes, or implementation guide  

What the documents contain:
- Problem framing and gap analysis against the existing Windows implementation
- A complete module-level design for the new approval system
- An incremental implementation plan with explicit dependency ordering
- Key design decisions with rationale and alternatives considered
- Detailed subsystem notes for persistence, evaluation, and gateway integration

What the documents do not contain:
- Implementation status tracking
- API reference or usage instructions
- Test coverage or CI status
- Any assumption that the system described is fully built

---

## Problem

The Windows node executes `system.run` commands from agents using a basic glob-pattern rule list (`ExecApprovalPolicy`). The macOS implementation provides a richer model: per-agent security tiers (`security`/`ask`/`allowlist`), interactive prompting, allowlist-based approval with persistent tracking, and fallback behavior when the UI is unavailable. This work adapts that model to Windows idioms while keeping the existing rule-list behavior intact as a separately-activatable path.

The two models are conceptually incompatible and cannot be merged:

| Dimension | macOS model | Windows legacy model |
|-----------|-------------|----------------------|
| Model | security × ask × allowlist | ordered rule list |
| Scope | per-agent cascade | global |
| Interactive prompt | fully implemented | stub only |
| Env sanitization | inside evaluator | before policy check |
| Shell chain | single-level extract | multi-level BFS |

---

## Contents

- **`architecture-overview.md`** — System design, components, invariants, and the relationship between the legacy path and the new approval model.
- **`incremental-plan.md`** — Phased technical map for introducing the new system without disrupting the existing path.
- **`decisions.md`** — Key design decisions, alternatives considered, tradeoffs, and rationale.
- **`design-details/persistence-design.md`** — Store schema, atomic writes, cascade resolution, and platform differences from the macOS implementation.
- **`design-details/evaluation-pipeline.md`** — The four evaluation submodules (request validation, executable resolution, allowlist matching, state machine) and their pipeline contracts.
- **`design-details/gateway-integration.md`** — The dual-sided gateway architecture (operator side and node side) and protocol design.

---

## Guiding Decisions

1. The existing rule-list model (`ExecApprovalPolicy`) is preserved unchanged. The new system coexists as a separate, independently-activatable path — no migration, no replacement in the first wave.
2. The macOS implementation is the behavioral reference — not a literal translation, but a conceptual port adapted to C#/WinUI idioms and Windows platform constraints.
3. The new path is fail-closed: any error or ambiguity in the approval pipeline produces a typed deny, never a silent fallback to the legacy path.
4. The evaluation pipeline is deterministic and side-effect-free. Side effects (allowlist writes, usage tracking) occur only after a final allow decision.
5. Core approval logic (validation, resolution, evaluation, matching) is testable without any WinUI dependency.
6. The new path is activated only by an explicit configuration signal — not by file presence, not by code presence.

---

## How to Read This

Start with `architecture-overview.md` for the overall model and invariants. Read `incremental-plan.md` to understand the dependency structure and what each phase unlocks. Read `decisions.md` for the reasoning behind non-obvious choices. The `design-details/` directory contains subsystem-level notes that go deeper than the overview documents.
