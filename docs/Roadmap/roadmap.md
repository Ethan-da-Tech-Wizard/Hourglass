# Roadmap

Documentation precedes implementation — that is a directive, not a preference. Each phase has an
exit gate; a phase is done when its gate holds, not when its time is up. Phases build strictly
bottom-up: each one delivers something testable on its own, and nothing depends on a later
phase's magic.

This document explains the phases and their gates; the **live execution state** — itemized
tasks, check-offs with dates and commits, gate sign-offs — lives in
[CHECKLIST.md](../../CHECKLIST.md) at the repository root and is updated in the same commit as
the work it records.

## Phase 0 — Design and research *(current)*

The engineering notebook: architecture, knowledge model, reasoning design, storage decision,
prior-art survey, founding ADRs, worked example. This phase produced everything currently in
[docs/](../README.md).

**Exit gate:** a competent engineer can read the docs cold and describe every subsystem's
responsibility and interface without asking questions. The formerly-open items are now written
as normative specs: [grain serialization](../Specifications/grain-serialization.md),
[storage interface](../Specifications/storage-interface.md),
[rule syntax](../Specifications/rule-syntax.md),
[answer envelope](../Specifications/answer-envelope.md) — plus the
[v1 scope freeze](../Specifications/v1-scope.md) and the
[requirements traceability matrix](../Specifications/requirements-traceability.md), which is
reviewed at this and every later gate. Remaining before the gate: the cold-read review itself.

## Phase 1 — Grain store

SQLite schema, content-hash identity, status log, manual grain entry (curator CLI), entity
registry with alias resolution, seed relationship vocabulary loaded from declarations.
No inference yet — a provable *storage* layer first.

**Exit gate:** the [worked example](../Examples/worked-example.md) KB (grains G1–G5) can be
entered, deduplicated (three sources → one ORBITS grain), queried by pattern, and every grain's
provenance retrieved; the integrity audit (re-hash everything) passes.

## Phase 2 — Rules and proof engine

Datalog evaluator with tabled backward chaining; declaration-derived generic rules; proof-chain
construction; the small verifier; deterministic proof ordering.

**Exit gate:** "Does Rex have lungs?" returns the exact proof chain in the worked example;
"Does Rex have gills?" returns `Unknown. Insufficient evidence.`; answers are byte-identical
across runs and across evaluation-strategy settings.

## Phase 3 — Contradiction engine and provenance completion

Functional/disjointness constraints, incremental conflict detection at Link, dispute lifecycle,
conflict records, curator resolution flow, source snapshot archive, hash-chained status log.

**Exit gate:** the worked example's step 7 (Proxima injection) plays out exactly as documented:
both grains suspended, "Contradictory evidence exists" with both proof chains, curated
retraction logged append-only.

## Phase 4 — Renderer

Template-driven proof-to-prose translation; refusal phrasing per
[unknown-handling.md](../Reasoning/unknown-handling.md); the interface guarantee that the
renderer receives only the checked proof object.

**Exit gate:** every sentence in a rendered answer maps to a node of the proof chain (checked
mechanically: render → parse back → compare node coverage); refusals name the KB, not the world.

## Phase 5 — Acquisition (tiers 1–3)

Structured-source ingesters (tier 1), template extractors (tier 2), pattern extractors (tier 3),
quarantine queue and admission gates, extractor conformance suite
([parsing.md](../Research/parsing.md)).

**Exit gate:** a real structured source ingested end-to-end: bytes → snapshot → candidates →
verified canonical grains → provable answers, with every grain's extractor version recorded and
a deliberately-broken extractor's output recallable by that version.

## Phase 6 — Scale and hardening

Benchmark KB generator, performance baselines and CI regression gates
([performance.md](../Performance/performance.md#measurement-discipline)), proof cache with
TMS-style invalidation, materialized hot closures with incremental maintenance, integrity audit
job, poisoning tripwires ([Security](../Security/integrity.md)).

**Exit gate:** the metrics exist with trend lines (grains/sec, query p99 by proof depth,
bytes/grain), and a documented poisoning drill (inject a conflicting source, watch suspension →
enumeration → recall) passes.

## Deliberately unscheduled

Negative grains and stratified negation; tier-4 prose extraction under the containment rule;
multi-writer storage migration; KB distribution snapshots. Each is future work in its owning
document and enters the roadmap only through an RFC — an unscheduled list that quietly becomes
a promise is how projects lie to themselves.
