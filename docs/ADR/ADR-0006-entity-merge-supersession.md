# ADR-0006: Entity merges supersede grains; no stored record is ever modified in place

- **Status:** Accepted
- **Date:** 2026-07-03
- **Related:** [grain-serialization.md](../Specifications/grain-serialization.md#hash-stability-vs-entity-merges--the-resolved-conflict),
  [entities.md](../Knowledge_Model/entities.md), [ADR-0001](ADR-0001-atomic-grains.md)

## Problem

Two commitments made independently turned out to conflict. ADR-0001: a grain's id is the
content hash of its bytes, which include its subject/object entity ids. Entities doc (as first
drafted): when two entities are found identical, "grains are re-pointed" to the surviving id.
Re-pointing changes a grain's bytes, therefore its hash, therefore its id — while provenance
rows, proof caches, audit-log entries, and conflict records still reference the old id. Either
those are all rewritten too (a cascading in-place mutation of exactly the records that are
supposed to be immutable), or the store silently violates "id = hash of content." This is the
kind of seam mismatch that surfaces as a mid-implementation redesign if not caught on paper.

## Options considered

1. **In-place re-point + cascade rewrite** — keeps one grain row per fact; but it is a
   destructive multi-table mutation, the audit log can only describe it rather than being it,
   and the id/content invariant needs a "temporarily false during merge" caveat. Complexity
   lands in the most safety-critical code path.
2. **Read-time redirect** — never touch grains; every lookup chases entity redirect chains.
   Grains stay byte-stable; but redirect logic smears into every read operation and every
   consumer (pattern matching, closure computation, contradiction checks must all think about
   dup-vs-canonical ids forever), and two grains stating the same fact via different aliases of
   one entity would *never* unify — deduplication quietly dies.
3. **Supersession** — a merge is executed as ordinary admissions + status changes: for each
   grain referencing the duplicate, admit the substituted grain (new id, provenance carried
   over, `derived-from` link in the status log), set the old grain to a new `superseded`
   status. One transaction. No record edited, ever.

## Decision

Option 3. `merge_entity` in the [storage interface](../Specifications/storage-interface.md#writestore-operations)
performs supersession atomically; `superseded` enters the status lifecycle (usable for history
and audit, never as a proof leaf); the merged entity remains, marked `merged-into`, so old ids
stay resolvable forever via `canonical_entity()`.

## Reasoning

- It composes from operations the system already has and already audits — admission and status
  change. No new mutation primitive, no new invariant, no special-case code path (KISS in the
  strong sense: the feature is a *usage* of the core, not an extension of it).
- The id/content invariant stays exceptionless, which the tamper-detection story depends on
  ([Security](../Security/integrity.md#tamper-detection-content-addressing-all-the-way-down)) —
  an integrity audit needs zero knowledge of merge history.
- Historical proofs referencing the old grain ids remain *replayable* (the superseded grains
  still exist, bytes intact) while new proofs automatically build on the new ids — exactly the
  "explain what we believed then, prove from what we believe now" split the project wants.
- Read paths stay dumb. All merge complexity is paid once, at write time, by the one serialized
  writer — consistent with the [write-heavy/read-simple](../Performance/performance.md#workload-shape)
  allocation used everywhere else.

## Tradeoffs

- A high-degree entity's merge admits many grains in one transaction — bounded, measurable
  write-path cost, accepted (merges are rare, curator-initiated events).
- Storage keeps both generations of every superseded grain — that is the price of immutable
  history, already accepted globally by ADR-0001.
- Proof caches keyed to superseded grains are invalidated wholesale via the dependency index —
  correct and already-designed behavior, just noting merges as a trigger.

## Consequences

- `superseded` is added to the grain status lifecycle and to
  [grain-serialization.md](../Specifications/grain-serialization.md); the proof engine's leaf
  filter (verified only) already excludes it with no code change.
- Entity *splits* remain out of v1 scope ([v1-scope.md](../Specifications/v1-scope.md)) — but
  this ADR defines the pattern a future split RFC must follow: supersession, never mutation.
- Standing design rule extracted from this incident: **any proposed operation described with
  the words "update," "re-point," or "rewrite" on stored records is mis-specified; find its
  supersession form.** Cheap to check in review, and it would have caught the original
  entities.md wording.
