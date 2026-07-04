# Requirements traceability matrix

**Problem this spec fixes:** "all requirements work together" is only checkable if every
requirement from the founding directive maps to (1) the design doc that answers it, (2) the
normative spec that pins it, (3) the phase that builds it, and (4) the test that proves it —
with no orphans in either direction. This matrix is that map. It is reviewed at every phase
gate; a red or missing cell blocks the gate.

Requirement ids are stable (`REQ-nn`) so commits, RFCs, and test names can reference them.

## The matrix

| ID | Requirement (from the directive) | Design home | Normative pin | Phase | Acceptance test |
|---|---|---|---|---|---|
| REQ-01 | Deterministic reasoning — same KB, same question ⇒ same answer, always | [ADR-0005](../ADR/ADR-0005-datalog-style-rules.md), [proof-engine.md](../Reasoning/proof-engine.md) | [answer-envelope.md](answer-envelope.md#kb_version) determinism contract | 2 | Byte-identical responses across runs, machines, cache states, parallelism settings (CI gate) |
| REQ-02 | Atomic fact storage — grains, not paragraphs | [grains.md](../Knowledge_Model/grains.md) | [grain-serialization.md](grain-serialization.md) | 1 | Conformance fixture `grains-v1.tsv` round-trips bit-exactly |
| REQ-03 | Canonicalization — equivalent language ⇒ one grain, evidence preserved | [canonical-relationships.md](../Knowledge_Model/canonical-relationships.md), [entities.md](../Knowledge_Model/entities.md) | grain-serialization + [storage-interface.md](storage-interface.md) dedup-on-admit | 1 | Worked example step 4: three phrasings ⇒ one grain id, three provenance rows |
| REQ-04 | Proof-chain generation — every answer carries a recoverable proof | [proof-engine.md](../Reasoning/proof-engine.md) | [answer-envelope.md](answer-envelope.md#response) (PROVEN ⟹ checked proof) | 2 | Rex/lungs query returns the exact documented chain; verifier re-check passes |
| REQ-05 | Unknown handling — never fabricate; classified refusals | [unknown-handling.md](../Reasoning/unknown-handling.md), [ADR-0003](../ADR/ADR-0003-unknown-over-guessing.md) | answer-envelope outcome enum; no confidence fields | 2 | Rex/gills ⇒ `UNKNOWN`; schema grep: no confidence/score/rank field exists |
| REQ-06 | Contradiction detection — conflicts surfaced, never silently overwritten | [contradiction-engine.md](../Reasoning/contradiction-engine.md) | storage-interface (no delete op; status transitions checked) | 3 | Worked example step 7: both grains suspended, `CONTRADICTED` with both proofs, curated resolution logged |
| REQ-07 | Rule inheritance — general facts stored once, instances derived | [rule-system.md](../Reasoning/rule-system.md) | [rule-syntax.md](rule-syntax.md) | 2 | KB contains no `Dog HAS_PROPERTY lungs` grain, yet the query proves; store scan confirms absence |
| REQ-08 | Source traceability — every grain: origin, time, version, verification, evidence | [provenance.md](../Knowledge_Model/provenance.md), [evidence-viewer.md](../Architecture/evidence-viewer.md) | storage-interface `evidence()`, `admit_source()` | 1, 5 | Every grain in the example KB answers all six provenance questions via API; evidence viewer round-trip: offsets → location map → highlight lands on the recorded excerpt in the original (P5.14) |
| REQ-09 | Canonical relationship mapping — closed vocabulary, documented reasons | [ADR-0002](../ADR/ADR-0002-closed-relationship-vocabulary.md) | rule-syntax validity rule 2; storage-interface validation rule 3 | 1 | Admission of a grain/rule with undeclared relationship is rejected with the documented error |
| REQ-10 | Storage efficiency — compress structure, never proof-required information | [grains.md](../Knowledge_Model/grains.md#structural-compression) | grain-serialization (identity = dedup) | 6 | bytes/grain and grains-vs-derivable-facts metrics tracked with CI trend gates |
| REQ-11 | High scalability without sacrificing determinism | [performance.md](../Performance/performance.md), [ADR-0004](../ADR/ADR-0004-sqlite-first-storage.md) | storage-interface (swappable engine; ordering guarantees) | 6 | Benchmark KB: p99 latency by proof depth within budget; REQ-01 test still green under load |
| REQ-12 | Human-readable explanations from proof only | renderer sections of [overview](../Architecture/overview.md#7-answer) | [answer-envelope.md](answer-envelope.md#rendering) (renderer holds no store handle) | 4 | Mechanical check: every rendered sentence maps to a proof node; renderer compiles with no store dependency |
| REQ-13 | Immutable, tamper-evident history | [Security](../Security/integrity.md) | grain-serialization (content ids), storage-interface (no delete; hash-chained log) | 1, 6 | Integrity audit passes; injected byte-flip in store/archive/log is detected |
| REQ-14 | Safe knowledge update — merges, retraction, supersession without loss | [ADR-0006](../ADR/ADR-0006-entity-merge-supersession.md) | storage-interface `merge_entity`, status lifecycle | 3 | Merge test: old proofs replayable, new proofs use new ids, audit chain intact |
| REQ-15 | Deterministic parsing first; probabilistic only as contained proposers | [parsing.md](../Research/parsing.md) | [v1-scope.md](v1-scope.md) (tier 1–3 only; tier 4 OUT) | 5 | Extractor conformance suite; provenance records extractor+version for every grain |
| REQ-16 | Engineering simplicity — every subsystem understandable, no magic | KISS sections throughout; [docs/README.md](../README.md#documentation-standards) | [Specifications/README.md](README.md) (six seams, no more) | 0, every gate | Phase 0 gate: cold-read test by an engineer new to the project |

## Reverse check — no orphan machinery

Read the matrix bottom-up quarterly and at each gate: every spec section, every storage
operation, every status value must be reachable from some REQ row. Machinery that no requirement
demands is scope creep wearing an engineering costume — it gets an RFC justifying a new REQ, or
it gets deleted. (This check is what keeps "tight-knit" true over time, not just at design.)

## Known intentional gaps

Requirements the directive names that v1 *defers*, with their parking place — listed here so
the matrix cannot be accused of quietly dropping them:

- Proven negative answers (needs negative grains) — [v1-scope OUT](v1-scope.md#out--explicitly-not-in-v1), REQ candidate for v2.
- Prose-scale acquisition (tier 4) — same.
- Multi-writer scale-out — same, with revisit triggers already written.
