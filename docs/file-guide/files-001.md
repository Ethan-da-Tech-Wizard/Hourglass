# Files 1 of 1 — Hourglass

[All pages](../FILE_GUIDE.md) · [Project guide](../PROJECT_GUIDE.md)

| File | Role and useful details |
| --- | --- |
| [.gitignore](../../.gitignore) | Defines local data, credentials, dependencies, and generated outputs that Git should leave untracked. |
| [CHECKLIST.md](../../CHECKLIST.md) | Documentation: Hourglass Master Progression Checklist. Covers Rules of this checklist — read before touching it; Phase 0 — Design and research; Phase 1 — Grain store. |
| [CLAUDE.md](../../CLAUDE.md) | Documentation: Working rules for agents on the Hourglass project. Covers Before doing anything; Checklist protocol (mandatory); Documentation preservation policy (mandatory). |
| [LICENSE](../../LICENSE) | Preserved license, copyright, or attribution terms; read the full text before reusing the covered component. |
| [PROJECT_NOTES.md](../../PROJECT_NOTES.md) | Preserved detailed setup, usage, architecture, and development notes from the earlier README. |
| [README.md](../../README.md) | Project overview, current status, entry points, setup navigation, and documentation links. |
| [docs/ADR/ADR-0000-template.md](../../docs/ADR/ADR-0000-template.md) | Documentation: ADR-NNNN:. Covers Problem; Options considered; Decision. |
| [docs/ADR/ADR-0001-atomic-grains.md](../../docs/ADR/ADR-0001-atomic-grains.md) | Documentation: ADR-0001: Knowledge is stored as atomic grains (triples + qualifiers), immutable except status. Covers Problem; Options considered; Decision. |
| [docs/ADR/ADR-0002-closed-relationship-vocabulary.md](../../docs/ADR/ADR-0002-closed-relationship-vocabulary.md) | Documentation: ADR-0002: The canonical relationship vocabulary is closed, extended only by RFC. Covers Problem; Options considered; Decision. |
| [docs/ADR/ADR-0003-unknown-over-guessing.md](../../docs/ADR/ADR-0003-unknown-over-guessing.md) | Documentation: ADR-0003: No confidence scores in the answer path; refusal is a correct output. Covers Problem; Options considered; Decision. |
| [docs/ADR/ADR-0004-sqlite-first-storage.md](../../docs/ADR/ADR-0004-sqlite-first-storage.md) | Documentation: ADR-0004: SQLite + content-addressed archive as the initial storage engine. Covers Problem; Options considered; Decision. |
| [docs/ADR/ADR-0005-datalog-style-rules.md](../../docs/ADR/ADR-0005-datalog-style-rules.md) | Documentation: ADR-0005: The rule language is Datalog — no function symbols, no negation, range-restricted. Covers Problem; Options considered; Decision. |
| [docs/ADR/ADR-0006-entity-merge-supersession.md](../../docs/ADR/ADR-0006-entity-merge-supersession.md) | Documentation: ADR-0006: Entity merges supersede grains; no stored record is ever modified in place. Covers Problem; Options considered; Decision. |
| [docs/ADR/README.md](../../docs/ADR/README.md) | Documentation: Architectural Decision Records. Covers Index. |
| [docs/Architecture/codebase-layout.md](../../docs/Architecture/codebase-layout.md) | Documentation: Codebase layout — the visual architecture. Covers Repository tree (planned); Module dependency graph; The four load-bearing constraints in that graph. |
| [docs/Architecture/evidence-viewer.md](../../docs/Architecture/evidence-viewer.md) | Documentation: Evidence viewer: showing the user where information came from. Covers The chain the viewer walks; Location maps: the one new artifact; Presentation (v1). |
| [docs/Architecture/file-intake.md](../../docs/Architecture/file-intake.md) | Documentation: File intake: where uploaded information sits. Covers The invariant; Physical layout; Renditions: the conversion layer. |
| [docs/Architecture/overview.md](../../docs/Architecture/overview.md) | Documentation: Architecture overview. Covers The seven-stage pipeline; Stage responsibilities; 1. Acquire. |
| [docs/DEPENDENCIES.csv](../../docs/DEPENDENCIES.csv) | Machine-readable inventory of declared and locked dependencies, versions, reported licenses, and evidence URLs. |
| [docs/Examples/worked-example.md](../../docs/Examples/worked-example.md) | Documentation: Worked example: the whole pipeline on a small knowledge base. Covers 1. Sources arrive; 2. Acquire → candidate grains; 3. Verify + Canonicalize. |
| [docs/FILE_GUIDE.md](../../docs/FILE_GUIDE.md) | Index of the complete tracked-file guide, with one entry for every file and links to the paginated descriptions. |
| [docs/Glossary/README.md](../../docs/Glossary/README.md) | Documentation: Glossary. Covers Grain; Canonical Grain; Entity. |
| [docs/Knowledge_Model/canonical-relationships.md](../../docs/Knowledge_Model/canonical-relationships.md) | Documentation: Canonical relationships. Covers Relationship declaration format; Seed vocabulary; Why there is no `IS_A`. |
| [docs/Knowledge_Model/entities.md](../../docs/Knowledge_Model/entities.md) | Documentation: Entities and entity resolution. Covers Entity model; Resolution strategy (deterministic first); Tradeoffs. |
| [docs/Knowledge_Model/grains.md](../../docs/Knowledge_Model/grains.md) | Documentation: Grains: the atomic unit of knowledge. Covers What a grain is; Why triples + qualifiers, not paragraphs and not n-ary tuples; Literals. |
| [docs/Knowledge_Model/provenance.md](../../docs/Knowledge_Model/provenance.md) | Documentation: Provenance and source tracking. Covers Source model; Per-grain provenance entry; What this gives each success criterion. |
| [docs/LICENSE_REVIEW.md](../../docs/LICENSE_REVIEW.md) | Original-code license, preserved component notices, dependency-license evidence, and unresolved distribution questions. |
| [docs/PROJECT_GUIDE.md](../../docs/PROJECT_GUIDE.md) | Maintainer orientation: purpose, important entry points, source layout, commands, and project-specific boundaries. |
| [docs/Performance/performance.md](../../docs/Performance/performance.md) | Documentation: Performance. Covers Workload shape; Indexing; Incremental updates and materialization. |
| [docs/README.md](../../docs/README.md) | Documentation: Hourglass documentation map. Covers Directory layout; Reading order for a new contributor; Documentation standards. |
| [docs/RFC/README.md](../../docs/RFC/README.md) | Documentation: RFC process. Covers What requires an RFC; Process (deliberately lightweight); Index. |
| [docs/RFC/RFC-0000-template.md](../../docs/RFC/RFC-0000-template.md) | Documentation: RFC-NNNN:. Covers Problem; Proposal; Alternatives. |
| [docs/RFC/RFC-0001-confidence-thresholded-answering.md](../../docs/RFC/RFC-0001-confidence-thresholded-answering.md) | Documentation: RFC-0001: Confidence-thresholded answering. Covers Problem; Proposal (as considered); Why it was rejected. |
| [docs/Reasoning/contradiction-engine.md](../../docs/Reasoning/contradiction-engine.md) | Documentation: Contradiction engine. Covers What counts as a contradiction; Detection: at the write path, incrementally; Response: mark, preserve, explain — never overwrite. |
| [docs/Reasoning/proof-engine.md](../../docs/Reasoning/proof-engine.md) | Documentation: Proof engine. Covers The proof chain object; Query evaluation: backward chaining, with selective forward materialization; Multiple proofs and minimal proofs. |
| [docs/Reasoning/rule-system.md](../../docs/Reasoning/rule-system.md) | Documentation: Rule system. Covers Rule form; Formal basis: Datalog, restricted; Property-derived rules. |
| [docs/Reasoning/unknown-handling.md](../../docs/Reasoning/unknown-handling.md) | Documentation: Unknown handling. Covers The refusal taxonomy; Open world for facts, closed world for search; Where guessing is structurally excluded. |
| [docs/Research/parsing.md](../../docs/Research/parsing.md) | Documentation: Parsing: from sources to candidate grains. Covers Framing that changes the problem: output is *candidates*; Source difficulty ladder; Deterministic methods for tier 3. |
| [docs/Research/prior-art.md](../../docs/Research/prior-art.md) | Documentation: Prior art survey. Covers Symbolic AI and expert systems; Cyc; Logic programming: Prolog, Datalog, XSB. |
| [docs/Roadmap/roadmap.md](../../docs/Roadmap/roadmap.md) | Documentation: Roadmap. Covers Phase 0 — Design and research *(current)*; Phase 1 — Grain store; Phase 2 — Rules and proof engine. |
| [docs/Security/integrity.md](../../docs/Security/integrity.md) | Documentation: Security: integrity, tamper detection, and knowledge injection. Covers Threat model (first cut); Tamper detection: content addressing all the way down; Immutable history: the hash-chained audit log. |
| [docs/Specifications/README.md](../../docs/Specifications/README.md) | Documentation: Specifications. Covers The contract set; Why these six and no more; Change policy. |
| [docs/Specifications/answer-envelope.md](../../docs/Specifications/answer-envelope.md) | Documentation: Answer envelope — normative. Covers Query; Response; kb_version. |
| [docs/Specifications/grain-serialization.md](../../docs/Specifications/grain-serialization.md) | Documentation: Grain serialization and identity — normative. Covers Encoding primitives; Identifier formats; Canonical grain form. |
| [docs/Specifications/requirements-traceability.md](../../docs/Specifications/requirements-traceability.md) | Documentation: Requirements traceability matrix. Covers The matrix; Reverse check — no orphan machinery; Known intentional gaps. |
| [docs/Specifications/rule-syntax.md](../../docs/Specifications/rule-syntax.md) | Documentation: Rule syntax — normative. Covers Grammar (EBNF); Static validity — rejected at parse, not at runtime; The `*` closure sugar. |
| [docs/Specifications/storage-interface.md](../../docs/Specifications/storage-interface.md) | Documentation: Storage interface — normative. Covers Two handles, not one; ReadStore operations; WriteStore operations. |
| [docs/Specifications/v1-scope.md](../../docs/Specifications/v1-scope.md) | Documentation: v1 scope — frozen. Covers IN — v1 builds exactly this; OUT — explicitly not in v1; Change control. |
| [docs/Storage/storage-research.md](../../docs/Storage/storage-research.md) | Documentation: Storage research. Covers Requirements, derived from the design; Candidate comparison; Recommendation: SQLite first. |
| [docs/file-guide/files-001.md](../../docs/file-guide/files-001.md) | One page of the tracked-file catalog; links each listed path to its role and source-derived details. |
