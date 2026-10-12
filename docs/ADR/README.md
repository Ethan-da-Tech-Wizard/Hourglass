# Architectural Decision Records

Every significant engineering choice is recorded here: the problem, the options considered, the
decision, the reasoning, the tradeoffs accepted, and the consequences. ADRs are append-only —
a reversed decision gets a *new* ADR that supersedes the old one; the old one is marked
superseded but never deleted. (The knowledge base never silently overwrites; neither does the
notebook.)

Template: [ADR-0000-template.md](ADR-0000-template.md). Proposals still under discussion belong
in [RFC/](../RFC/README.md); an accepted RFC that decides something produces an ADR.

## Index

| ADR | Title | Status |
|---|---|---|
| [0001](ADR-0001-atomic-grains.md) | Knowledge is stored as atomic grains (triples + qualifiers), immutable except status | Accepted |
| [0002](ADR-0002-closed-relationship-vocabulary.md) | The canonical relationship vocabulary is closed, extended only by RFC | Accepted |
| [0003](ADR-0003-unknown-over-guessing.md) | No confidence scores in the answer path; refusal is a correct output | Accepted |
| [0004](ADR-0004-sqlite-first-storage.md) | SQLite + content-addressed archive as the initial storage engine | Accepted |
| [0005](ADR-0005-datalog-style-rules.md) | The rule language is Datalog (no function symbols, no negation, range-restricted) | Accepted |
| [0006](ADR-0006-entity-merge-supersession.md) | Entity merges supersede grains; no stored record is ever modified in place | Accepted |
