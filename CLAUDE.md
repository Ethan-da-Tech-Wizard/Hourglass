# Working rules for agents on the Hourglass project

Hourglass is a deterministic knowledge reasoning engine: nothing may be stated unless provable
from evidence; "Unknown. Insufficient evidence." is a correct answer. The design is complete and
frozen before implementation. Your job is to execute it, not to reinterpret it.

## Before doing anything

1. Open [CHECKLIST.md](CHECKLIST.md) — the master progression list. Work only on unchecked
   items in the current phase, respecting item order and phase gates (⛩ items are human-only).
2. Read [docs/Specifications/v1-scope.md](docs/Specifications/v1-scope.md). If what you're about
   to build is on the OUT list, stop — it needs an RFC, not code.
3. The normative specs in [docs/Specifications/](docs/Specifications/README.md) override any
   explanatory doc and override your own judgment about "better" formats or interfaces. A spec
   change is an RFC, never a silent deviation.

## Checklist protocol (mandatory)

- Every work commit/PR references the checklist item id(s) it advances (e.g. `P1.4`).
- When an item is complete, update CHECKLIST.md **in the same commit/PR**: `[ ]` → `[x]`,
  append ` — YYYY-MM-DD, <commit>`.
- Never delete, reorder, or renumber checklist items. Obsolete items are struck through with a
  pointer to the superseding RFC/ADR. New items go at the end of their phase section, and after
  the v1 freeze only an RFC may add them.

## Documentation preservation policy (mandatory)

All documentation is permanent record. Concretely:

- **Never delete** a documentation file, an ADR, an RFC, a checklist item, or a glossary term.
- **Never overwrite in a way that erases meaning**: ADRs and RFCs are append-only once accepted
  (a reversed decision gets a NEW ADR marked as superseding; the old one is only annotated
  `Superseded by ADR-NNNN`). Rejected RFCs are kept with their rejection reasons.
- Explanatory docs may be corrected and extended by reviewed PR, but a superseded design
  section gets an explicit supersession note pointing to its replacement — not a silent
  rewrite. When in doubt, add and annotate; don't remove.
- Full policy: [docs/README.md](docs/README.md#documentation-preservation-policy).

This mirrors the engine's own data model (grains are never edited, only status-changed —
[ADR-0001](docs/ADR/ADR-0001-atomic-grains.md), [ADR-0006](docs/ADR/ADR-0006-entity-merge-supersession.md)).
The notebook obeys the same law as the knowledge base.

## Engineering constraints you may not trade away

- **Determinism:** equal kb_version + equal query ⇒ byte-identical response
  ([answer-envelope.md](docs/Specifications/answer-envelope.md#kb_version)). No optimization,
  cache, or parallelization may change any answer.
- **No confidence scores** anywhere in the answer path — no field named confidence, score,
  probability, rank, weight, or trust ([ADR-0003](docs/ADR/ADR-0003-unknown-over-guessing.md)).
- **No delete operations** on stored knowledge; mutation is status change only.
- **KISS:** if your implementation of an item needs machinery the specs don't name, that's a
  design question — raise it (RFC or issue), don't improvise it.

## Definition of done for any task

Code + tests + spec-conformance fixtures green + docs updated + checklist item checked, all in
one reviewable unit. Partial work is left as an unchecked item with a note in the PR, never as
a half-checked box.
