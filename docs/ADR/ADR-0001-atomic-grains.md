# ADR-0001: Knowledge is stored as atomic grains (triples + qualifiers), immutable except status

- **Status:** Accepted
- **Date:** 2026-07-03
- **Related:** [grains.md](../Knowledge_Model/grains.md),
  [provenance.md](../Knowledge_Model/provenance.md),
  [Wikidata prior art](../Research/prior-art.md#wikidata)

## Problem

Hourglass needs a unit of stored knowledge that supports deduplication (equivalent language →
one object), mechanical reasoning (rules must pattern-match it), provenance (every fact traceable
to sources), and safe update (change without silent loss of history). Sentences/paragraphs
support none of these; the unit had to be designed.

## Options considered

1. **Sentences/passages with retrieval** — trivial ingestion, nothing lost; but no unification
   of equivalent facts, no rule matching, no proof. This is a search engine.
2. **Plain triples (classic RDF)** — simple, deep prior art, index-friendly; but time,
   measurement units, and context require reification (≈4 extra triples per statement), which
   destroys the simplicity that motivated it.
3. **Per-relation n-ary tuples** — fully expressive; but every relation gets bespoke schema, and
   storage, rules, and proofs must all handle arbitrary arity: complexity spreads to every
   subsystem.
4. **Triples + qualifier map, content-hash identity, status-only mutability** — one uniform
   shape; qualifiers carry time/units/context; hash identity makes dedup a lookup and tampering
   self-evident.

## Decision

Option 4. A grain is (subject entity, canonical relationship, object entity-or-typed-literal,
qualifier map), identified by the content hash of its canonical byte form; provenance is a
mandatory attached list; the only mutable field is status
(candidate/verified/disputed/retracted); nothing is ever edited or deleted.

## Reasoning

- The shape is Wikidata's statement model, proven at 100M+ entities and a decade of operation —
  the strongest empirical evidence available for a knowledge-unit design.
- Content-hash identity makes the three hard problems cheaper simultaneously: deduplication
  (same fact = same id), tamper detection (id verifies content), and caching (proofs reference
  stable ids).
- Qualifiers-as-identity cleanly separates "different fact" (different time qualifier) from
  "contradiction" (same qualifiers, conflicting object) — the contradiction engine's central
  distinction falls out of the data model.
- Status-only mutability makes the audit trail complete by construction: every transition is an
  append, there is nothing else to log.

## Tradeoffs

- Canonical byte serialization must be specified exactly (field order, unicode normalization,
  qualifier ordering) and frozen early — any change re-keys the entire store.
- Atomic facts make discourse-level knowledge (arguments, narratives, uncertainty *statements*)
  awkward; accepted — that is not the v1 problem.
- More rows than a document store; mitigated by structural compression, and disk spent on
  provenance is a stated cost of the mission.

## Consequences

- Storage needs hash-keyed tables and the status log ([ADR-0004](ADR-0004-sqlite-first-storage.md)).
- The contradiction engine compares only same-(subject, relationship, qualifiers) grains.
- A grain-serialization spec document must be written before any implementation
  ([roadmap](../Roadmap/roadmap.md) phase 1 gate).
