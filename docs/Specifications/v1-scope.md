# v1 scope — frozen

**Purpose:** make "no new features once code begins" an enforceable rule instead of a hope.
This document is the single answer to "is X in v1?" If X is not in the IN list, it is out, and
the change-control rule at the bottom is the only way in.

Freeze point: this document freezes when Phase 1 implementation starts
([roadmap](../Roadmap/roadmap.md)). Until then it may be amended by ordinary PR review.

## IN — v1 builds exactly this

**Knowledge store (Phase 1)**
- Grains per [grain-serialization.md](grain-serialization.md): triple + qualifiers, typed
  literals (number, quantity, date, string, boolean), content-hash identity, status lifecycle
  (candidate / verified / disputed / retracted / superseded).
- Entities with opaque ids, aliases, merge-by-supersession
  ([ADR-0006](../ADR/ADR-0006-entity-merge-supersession.md)).
- The seed relationship vocabulary
  ([canonical-relationships.md](../Knowledge_Model/canonical-relationships.md#seed-vocabulary))
  loaded from declaration files; vocabulary changes only via RFC.
- Provenance: sources as hashed immutable snapshots, evidence excerpts with offsets, extractor
  method+version per grain.
- Append-only, hash-chained status log. Single-writer admission.
- Curator CLI for manual grain/entity/rule entry and status changes.

**Reasoning (Phase 2)**
- Datalog rules per [rule-syntax.md](rule-syntax.md); declaration-derived generic rules
  (transitive, symmetric, inverse); tabled backward chaining; materialized `SUBCLASS_OF`
  closure with incremental maintenance.
- Proof chains as serialized, hashed DAGs; the independent verifier; deterministic proof
  ordering and minimal-proof selection per
  [proof-engine.md](../Reasoning/proof-engine.md#multiple-proofs-and-minimal-proofs).
- The full refusal taxonomy per [answer-envelope.md](answer-envelope.md).

**Integrity (Phase 3)**
- Functional and `DISJOINT_WITH` constraint checking, incremental, at Link.
- Dispute lifecycle, conflict records, curated resolution, supersession policy for
  time-qualified functional relationships.

**Output (Phase 4)**
- Template renderer over proof chains and refusals; the proof-object-only interface.

**Acquisition (Phase 5)** — tiers 1–3 only ([parsing.md](../Research/parsing.md)):
structured-source ingesters, template extractors, deterministic pattern extractors, quarantine
queue, extractor conformance suite, and the file-intake layer — content-addressed snapshot
archive plus the rendition/converter framework with its trusted/untrusted flag
([file-intake.md](../Architecture/file-intake.md)). Deterministic converters:
CSV/XLSX/JSON/HTML/born-digital PDF, including **bulk/batch spreadsheet intake** (idempotent by
content hash). One contained probabilistic converter: **OCR** for scanned documents — untrusted
renditions feeding quarantine only; admission requires human confirmation against the page
image or independent corroboration; OCR-internal confidence never leaves the converter
(amended IN 2026-07-04, pre-freeze, owner decision; containment per
[file-intake.md](../Architecture/file-intake.md#vision--ocr-the-honest-kiss-answer) and
[RFC-0001](../RFC/RFC-0001-confidence-thresholded-answering.md)).

**Hardening (Phase 6)**
- Proof cache with exact invalidation; benchmark KB + CI regression gates; integrity audit job;
  systematic-conflict (poisoning) analytics.

## OUT — explicitly not in v1

Each item is out *by decision*, with the doc that parks it:

| Out | Parked in |
|---|---|
| Negative grains; any negation in rules | [rule-system.md](../Reasoning/rule-system.md#negation) |
| Arithmetic, aggregation, or built-ins in rules | [ADR-0005](../ADR/ADR-0005-datalog-style-rules.md#tradeoffs) |
| Tier-4 prose extraction; any statistical extractor | [parsing.md](../Research/parsing.md#where-probabilistic-methods-may-be-required--and-the-containment-rule) |
| ~~OCR / vision converters (the untrusted-rendition socket ships in v1; the OCR appliance is v2)~~ — moved to IN 2026-07-04, pre-freeze, owner decision; OCR ships in v1 under containment (quarantine-only, human confirm or corroboration) | [file-intake.md](../Architecture/file-intake.md#vision--ocr-the-honest-kiss-answer) |
| Audio/video ingestion; general multimodal interpretation ("describe this image") | [file-intake.md](../Architecture/file-intake.md#per-format-plan) — the latter is out of philosophy, not just out of scope |
| Natural-language *questions* — v1 queries are structured goal patterns only | [answer-envelope.md](answer-envelope.md#query) |
| Multi-writer / distributed storage; PostgreSQL migration | [storage-research.md](../Storage/storage-research.md#revisit-triggers-written-down-now-on-purpose) |
| Read-only KB distribution snapshots (HDT-like) | [storage-research.md](../Storage/storage-research.md#future-work) |
| Entity *splits* (one entity discovered to be two) — v1 supports merge only; splits are manual curation | [entities.md](../Knowledge_Model/entities.md#future-work) |
| Access control / multi-curator auth | [Security](../Security/integrity.md#future-work) |
| Cross-language labels beyond storage of them | [entities.md](../Knowledge_Model/entities.md#future-work) |
| Web UI of any kind — v1 surfaces are CLI + library API | this document |

Note what the OUT list implies about honesty of the IN list: v1 answers structured queries over
deterministically-ingested knowledge with proofs. It is not a chatbot and will not feel like
one. That is the product ([unknown-handling.md](../Reasoning/unknown-handling.md#tradeoffs)).

## Change control

After the freeze:

1. Any scope addition, removal, or spec change requires an **RFC** ([process](../RFC/README.md)),
   and the RFC must state which phase gate it invalidates and what re-work it causes.
2. The default disposition of every proposal is **v2 backlog**, not v1. The bar for touching v1
   is "a phase gate cannot be met without it" — i.e. the freeze protects working code from
   ideas, and only *blockers* get through.
3. Bug fixes against a spec are not scope changes; where the spec itself is the bug, rule 1
   applies.

## Definition of done for v1

Every row of the [requirements traceability matrix](requirements-traceability.md) green: each
requirement's acceptance test passing at its owning phase gate, all six phase gates held, no
OUT-list item implemented.
