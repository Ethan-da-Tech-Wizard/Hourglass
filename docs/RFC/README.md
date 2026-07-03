# RFC process

An RFC (request for comments) is a **proposal under discussion** — used when a change needs
review before anyone is entitled to act on it. An [ADR](../ADR/README.md) records a decision
already made; an accepted RFC that settles something significant produces one.

## What requires an RFC

- Adding or changing a **canonical relationship** (the vocabulary is closed —
  [ADR-0002](../ADR/ADR-0002-closed-relationship-vocabulary.md)). The RFC must contain the full
  declaration: domain, range, properties (with justification for each property claimed — a
  wrong `transitive` flag corrupts proofs), aliases, and the queries/rules that need it.
- Changing the **grain schema** or its canonical serialization (re-keys the store —
  [ADR-0001](../ADR/ADR-0001-atomic-grains.md)).
- Extending the **rule language** (must include termination and confluence arguments —
  [ADR-0005](../ADR/ADR-0005-datalog-style-rules.md)).
- Adding a **constraint type** to the contradiction engine, or a new admission policy class.
- Adding a term to the [glossary](../Glossary/README.md) that names a new concept.

Bug fixes, doc clarifications, new extractors within existing vocabulary, and implementation
choices inside one subsystem do not need an RFC.

## Process (deliberately lightweight)

1. Copy [RFC-0000-template.md](RFC-0000-template.md) to `RFC-NNNN-short-title.md` (next free
   number), status `Draft`.
2. Open a PR. Discussion happens on the PR.
3. Outcomes: `Accepted` (merge; write the ADR if the decision is architectural; implement),
   `Rejected` (merge anyway with status `Rejected` and the reason — rejected proposals are
   knowledge too and stop the same debate recurring), or `Withdrawn`.

The process exists to force declarations to be complete and reviewed, not to create ceremony.
If RFC latency ever becomes the acquisition bottleneck, fix the process — that risk is
explicitly tracked in ADR-0002's tradeoffs.

## Index

| RFC | Title | Status |
|---|---|---|
| — | *(none yet — the seed vocabulary and founding decisions shipped with the initial design as ADRs 0001–0005)* | |
