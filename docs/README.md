# Hourglass documentation map

This repository is the engineering notebook for Hourglass. Every document exists to answer a real
engineering question. If a document stops answering one, it is marked superseded with a pointer
to its replacement — never deleted (see the
[documentation preservation policy](#documentation-preservation-policy) below).

## Directory layout

| Directory | Purpose |
|---|---|
| [Architecture/](Architecture/) | System-level structure: subsystems, boundaries, data flow |
| [Research/](Research/) | Surveys of prior art and open investigations (parsing, provenance) |
| [Knowledge_Model/](Knowledge_Model/) | How knowledge is represented: grains, entities, relationships, provenance |
| [Storage/](Storage/) | Storage engine research and comparison |
| [Reasoning/](Reasoning/) | Rules, proof engine, contradiction handling, unknown handling |
| [Security/](Security/) | Integrity, tamper detection, malicious knowledge injection |
| [Performance/](Performance/) | Indexing, caching, incremental update research |
| [Examples/](Examples/) | End-to-end worked examples on concrete knowledge bases |
| [Specifications/](Specifications/) | **Normative** contracts: exact byte formats, interfaces, scope freeze, traceability matrix |
| [RFC/](RFC/) | Requests for comment: proposals that change the knowledge model or vocabulary |
| [ADR/](ADR/) | Architectural decision records: decisions already made, with reasoning |
| [Glossary/](Glossary/) | The project vocabulary, one definition per term |
| [Roadmap/](Roadmap/) | Phased plan from design to implementation |

## Reading order for a new contributor

1. [Glossary](Glossary/README.md) — learn the vocabulary first; the rest of the docs use it
   without re-defining it.
2. [Architecture overview](Architecture/overview.md) — the seven-stage pipeline and the
   subsystems that implement it.
3. [Grains](Knowledge_Model/grains.md) and
   [Canonical relationships](Knowledge_Model/canonical-relationships.md) — the data model.
4. [Rule system](Reasoning/rule-system.md) and [Proof engine](Reasoning/proof-engine.md) — how
   conclusions are derived.
5. [Worked example](Examples/worked-example.md) — everything above applied to a small concrete
   knowledge base.
6. [ADR index](ADR/README.md) — why the design is the way it is.
7. [Specifications](Specifications/README.md) — the normative contracts implementation codes
   against; read [v1-scope.md](Specifications/v1-scope.md) before proposing anything new.

## Documentation standards

Every research or design document should answer, in roughly this order:

1. **What problem exists?**
2. **Why does it matter?**
3. **Current approaches** (in prior art or in earlier drafts of this design).
4. **Advantages** and **disadvantages** of each approach.
5. **Engineering tradeoffs.**
6. **Recommended direction.**
7. **Rejected alternatives** — named explicitly, with the reason for rejection.
8. **Future work** — what is known to be unresolved.

Rules of the notebook:

- No placeholder documents. A document is created when there is content for it.
- No unsupported opinions. Design claims are backed by reasoning or a reference.
- Every document links to the documents it depends on and the ADRs it motivates.
- The [glossary](Glossary/README.md) is the single source of truth for terminology. New terms are
  added there before being used elsewhere.

## Documentation preservation policy

The notebook obeys the same law as the knowledge base it describes: **history is never erased.**
Grains are never edited or deleted, only status-changed
([ADR-0001](ADR/ADR-0001-atomic-grains.md), [ADR-0006](ADR/ADR-0006-entity-merge-supersession.md));
documentation gets the identical treatment.

1. **No document is ever deleted.** Not ADRs, not RFCs (including rejected ones), not checklist
   items, not glossary terms, not design docs. A document that no longer reflects the design is
   *superseded*, not removed.
2. **Decision records are append-only.** An accepted ADR or closed RFC is never edited beyond
   status annotations (`Superseded by ADR-NNNN`). Reversing a decision means writing a new ADR
   that names and supersedes the old one — the old reasoning remains readable forever, because
   knowing *why we used to think otherwise* is knowledge.
3. **Explanatory docs evolve by reviewed correction and extension**, with git history as the
   full audit trail. A superseded section gets an explicit note pointing to its replacement — a
   silent rewrite that makes an old decision look like it never happened is the documentation
   equivalent of silently overwriting a grain, and is treated as a defect in review.
4. **The [master checklist](../CHECKLIST.md) is append-only**: items are checked off with date
   and commit, or struck through with a superseding reference — never removed, reordered, or
   renumbered.
5. **Working agents are bound to this policy** by the repository-root
   [CLAUDE.md](../CLAUDE.md), which every agent session loads. Any PR that deletes a
   documentation file fails review on that ground alone.

## ADR vs RFC

- An **RFC** is a *proposal* — used when a change needs discussion before a decision (e.g. adding
  a canonical relationship, changing the grain schema). See [RFC/README.md](RFC/README.md).
- An **ADR** is a *record of a decision already taken* — problem, options, decision, reasoning,
  tradeoffs, consequences. See [ADR/README.md](ADR/README.md). An accepted RFC usually produces
  an ADR.
