# Hourglass Master Progression Checklist

This is the single authoritative TODO list for the project. Every working session — human or
agent — takes items from here and checks them off here. If work happened and this file didn't
change, the work isn't done.

Derived from the [roadmap](docs/Roadmap/roadmap.md) (phases and gates) and the
[requirements traceability matrix](docs/Specifications/requirements-traceability.md) (REQ ids).

## Rules of this checklist — read before touching it

1. **Append-only.** Items are NEVER deleted, reordered, or renumbered. Item ids
   (`P2.4`) are permanent and referenceable from commits, PRs, and RFCs.
2. **Checking off:** change `[ ]` to `[x]` and append ` — YYYY-MM-DD, <commit hash>` on the
   same line. The check-off ships **in the same commit or PR as the work itself**.
3. **An item found to be obsolete or wrong** is struck through (`~~text~~`) with a note
   ` — superseded by RFC-NNNN` or ` — superseded by ADR-NNNN`. It stays in place forever.
4. **Adding items:** before the v1 freeze ([v1-scope.md](docs/Specifications/v1-scope.md)),
   by ordinary PR review; after the freeze, only via RFC. New items go at the END of their
   phase section with the next free number — never inserted between existing items.
5. **A phase gate item** (marked ⛩) may only be checked when every item above it in that phase
   is checked or struck. Gates are checked by a human, not an agent.
6. This file is subject to the
   [documentation preservation policy](docs/README.md#documentation-preservation-policy):
   never deleted, never rewritten wholesale, history never erased.

---

## Phase 0 — Design and research

- [x] P0.1 Architecture overview, pipeline, subsystem boundaries — 2026-07-03, 5deccc7
- [x] P0.2 Knowledge model: grains, entities, canonical relationships, provenance — 2026-07-03, 5deccc7
- [x] P0.3 Reasoning design: rule system, proof engine, contradiction engine, unknown handling — 2026-07-03, 5deccc7
- [x] P0.4 Research: prior-art survey, parsing strategy — 2026-07-03, 5deccc7
- [x] P0.5 Storage research and comparison — 2026-07-03, 5deccc7
- [x] P0.6 Security and performance design — 2026-07-03, 5deccc7
- [x] P0.7 Founding ADRs 0001–0005; RFC process; glossary; worked example; roadmap — 2026-07-03, 5deccc7
- [x] P0.8 Normative specs: grain serialization, storage interface, rule syntax, answer envelope — 2026-07-03, b8a7991
- [x] P0.9 v1 scope freeze document and change control — 2026-07-03, b8a7991
- [x] P0.10 Requirements traceability matrix (REQ-01..16) — 2026-07-03, b8a7991
- [x] P0.11 ADR-0006: entity-merge supersession (hash-stability conflict resolved) — 2026-07-03, b8a7991
- [x] P0.12 Master checklist, documentation preservation policy, agent working rules — 2026-07-04 (the commit that introduced this file)
- [ ] P0.13 Cold-read review: an engineer new to the project reads the docs and can describe every subsystem without asking questions; findings filed as issues
- [ ] P0.14 ⛩ **Phase 0 gate sign-off** (human): docs coherent, traceability matrix reviewed, v1 scope frozen from here on

## Phase 1 — Grain store

- [ ] P1.1 ADR-0007: implementation language and toolchain (with reasoning per ADR template)
- [ ] P1.2 Repository scaffolding: build, lint, test runner, CI skeleton
- [ ] P1.3 Conformance fixture `conformance/grains-v1.tsv` (fact → canonical bytes → id), covering every literal kind and qualifier cases
- [ ] P1.4 Canonical serializer + SHA-256 identity passing P1.3 bit-exactly ([spec](docs/Specifications/grain-serialization.md))
- [ ] P1.5 Storage interface defined in code: ReadStore / WriteStore types ([spec](docs/Specifications/storage-interface.md))
- [ ] P1.6 In-memory reference implementation of the storage interface
- [ ] P1.7 SQLite implementation: schema, SPO/POS/OSP indexes, WAL mode
- [ ] P1.8 Storage conformance suite (ordering determinism, atomicity under crash injection, lifecycle legality); both implementations pass
- [ ] P1.9 Entity registry: sequential id allocation, labels, aliases, exact-alias resolution with class narrowing
- [ ] P1.10 Relationship vocabulary loader from declaration files (seed vocabulary in-repo)
- [ ] P1.11 Source admission + content-addressed snapshot archive
- [ ] P1.12 Provenance storage: evidence excerpts with offsets, extractor method+version fields
- [ ] P1.13 Hash-chained append-only status log with chain verification
- [ ] P1.14 Curator CLI: admit grain/entity/rule, set status, inspect provenance
- [ ] P1.15 Integrity audit command: re-hash all stores, verify status-log chain (REQ-13 partial)
- [ ] P1.16 Worked-example KB scripted: G1–G5 entered, three phrasings dedup to one ORBITS grain, provenance queries answer all six questions (REQ-02, REQ-03, REQ-08, REQ-09)
- [ ] P1.17 ⛩ **Phase 1 gate** (human): P1.16 and P1.15 green in CI; traceability rows REQ-02/03/08/09/13 reviewed

## Phase 2 — Rules and proof engine

- [ ] P2.1 Rule parser + static validity checks per [rule-syntax.md](docs/Specifications/rule-syntax.md)
- [ ] P2.2 Rule normalizer + identity hashing; `conformance/rules-v1/` fixtures (valid + each invalid class)
- [ ] P2.3 Declaration-derived generic rules (transitive, symmetric, inverse) generated with provenance
- [ ] P2.4 Materialized `SUBCLASS_OF` closure with incremental insert maintenance
- [ ] P2.5 Tabled backward-chaining evaluator (terminates on recursive rules; verified-status leaf filter)
- [ ] P2.6 Proof-chain construction, canonical serialization, and hashing per spec
- [ ] P2.7 The verifier: independent proof checker, small enough to hand-audit; runs inside the query path
- [ ] P2.8 Deterministic proof ordering and minimal-proof selection (total order incl. canonical-bytes tie-break)
- [ ] P2.9 Answer envelope implementation: all five outcomes, kb_version from status-log head ([spec](docs/Specifications/answer-envelope.md))
- [ ] P2.10 `conformance/answers-v1/`: worked-example queries with expected full response bytes, all five outcomes
- [ ] P2.11 Determinism CI gate: byte-identical responses across runs and evaluation-strategy settings (REQ-01)
- [ ] P2.12 Store scan proves no derivable fact is stored as a base grain (REQ-07)
- [ ] P2.13 ⛩ **Phase 2 gate** (human): Rex/lungs exact documented chain; Rex/gills → UNKNOWN; REQ-01/04/05/07 rows green

## Phase 3 — Contradiction engine and provenance completion

- [ ] P3.1 Functional-constraint checking at Link, incremental (neighborhood-scoped)
- [ ] P3.2 `DISJOINT_WITH` constraint checking; disjointness declarations for seed vocabulary reviewed
- [ ] P3.3 Dispute lifecycle: suspension of all sides, conflict records with proofs and provenance
- [ ] P3.4 Derived-conflict blame: premise implication via dependency index
- [ ] P3.5 Curator resolution flow: retract/refine with append-only rationale
- [ ] P3.6 `merge_entity` supersession, atomic, per [ADR-0006](docs/ADR/ADR-0006-entity-merge-supersession.md); old proofs replayable, new proofs on new ids (REQ-14)
- [ ] P3.7 Supersession policy for time-qualified functional relationships ("latest as_of is current; older remain historical")
- [ ] P3.8 CONTRADICTED envelope path end-to-end with corroboration counts displayed
- [ ] P3.9 ⛩ **Phase 3 gate** (human): worked-example step 7 (Proxima injection) plays out exactly as documented; REQ-06/14 green

## Phase 4 — Renderer

- [ ] P4.1 Renderer receives response records by value; compile-time proof it holds no store handle (REQ-12 structural half)
- [ ] P4.2 Refusal templates for all four refusal outcomes, phrased as claims about this KB, not the world
- [ ] P4.3 PROVEN rendering: sentence-per-proof-node templates using current preferred labels
- [ ] P4.4 Mechanical coverage check: render → map back → every sentence maps to a proof node, every proof node reachable
- [ ] P4.5 ⛩ **Phase 4 gate** (human): coverage check green on all conformance answers; REQ-12 green

## Phase 5 — Acquisition (tiers 1–3)

- [ ] P5.1 Quarantine queue: candidate grains with held-ambiguity records, curator review flow
- [ ] P5.2 Tier-1 ingester: one real structured source (schema-mapped), end to end
- [ ] P5.3 Tier-2 template extractor framework + first format
- [ ] P5.4 Tier-3 pattern extractors: copular definitions, Hearst patterns, date/quantity normalizers
- [ ] P5.5 Extractor conformance suite: per-pattern corpus with expected grains; extractor version recorded in provenance
- [ ] P5.6 Recall-by-toolchain drill: break an extractor, enumerate and re-verify everything it produced (REQ-15)
- [ ] P5.7 ⛩ **Phase 5 gate** (human): real source → snapshot → candidates → verified grains → provable answers, fully traced; REQ-15 green — *also requires P5.8–P5.12 below (added pre-freeze 2026-07-04, after this gate item was numbered)*
- [ ] P5.8 Rendition/converter framework: deterministic converters (CSV, XLSX→TSV, JSON, HTML, born-digital PDF), hashed rebuildable renditions, trusted/untrusted flag, rendition field in provenance ([file-intake.md](docs/Architecture/file-intake.md))
- [ ] P5.9 Intake CLI: `ingest <file|url>` → archived snapshot, renditions run, quarantine report
- [ ] P5.10 Bulk intake: directory/batch ingestion, idempotent re-runs via content hash, per-format (not per-file) schema mappings, parallel conversion workers feeding the single admission point ([file-intake.md bulk intake](docs/Architecture/file-intake.md#bulk-intake))
- [ ] P5.11 OCR converter (contained): one OCR tool wrapped as an untrusted converter; renditions feed quarantine only; OCR-internal confidence dies at the converter boundary ([RFC-0001](docs/RFC/RFC-0001-confidence-thresholded-answering.md))
- [ ] P5.12 OCR confirmation flow in curator CLI: side-by-side page image vs. excerpt, confirm/reject recorded in status log; corroboration path as the non-human alternative

## Phase 6 — Scale and hardening

- [ ] P6.1 Benchmark KB generator (size-parameterized)
- [ ] P6.2 Metrics with CI trend gates: grains/sec admitted, p50/p99 by proof depth, bytes/grain, cache hit rate (REQ-10, REQ-11)
- [ ] P6.3 Proof cache with exact dependency-index invalidation; paranoid re-verify mode default
- [ ] P6.4 Deletion-safe closure maintenance (DRed or counting, documented per materialization)
- [ ] P6.5 Scheduled integrity audit job; byte-flip injection test detects tampering in store, archive, and log (REQ-13)
- [ ] P6.6 Poisoning drill: inject conflicting source → suspension → source enumeration → recall, documented as runbook
- [ ] P6.7 ⛩ **Phase 6 gate / v1 done** (human): every REQ row green, all gates held, no OUT-list item implemented

## Standing obligations — checked at every gate, never "done"

- [ ] S.1 Traceability matrix reviewed at each gate (this item is re-verified, not checked off)
- [ ] S.2 No commit implements anything on the [v1 OUT list](docs/Specifications/v1-scope.md#out--explicitly-not-in-v1)
- [ ] S.3 Docs updated in the same PR as the code they describe; preservation policy respected
- [ ] S.4 Every commit references the checklist item id(s) it advances
