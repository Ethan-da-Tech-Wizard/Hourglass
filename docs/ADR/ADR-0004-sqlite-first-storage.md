# ADR-0004: SQLite + content-addressed archive as the initial storage engine

- **Status:** Accepted
- **Date:** 2026-07-03
- **Related:** [storage-research.md](../Storage/storage-research.md),
  [performance.md](../Performance/performance.md)

## Problem

Grains, entities, rules, sources, provenance, and the audit log need a home before
implementation starts. Requirements derived from the design: ACID writes (grain + provenance
atomically), point-lookup-heavy read pattern (backward chaining probes), append-only history,
operational simplicity, and inspectability with standard tools. Full comparison:
[storage-research.md](../Storage/storage-research.md#candidate-comparison).

## Options considered

1. **SQLite** — zero-config single file, ACID/WAL, ubiquitous tooling, decades-stable format;
   weaker recursive-query optimization, single-writer.
2. **PostgreSQL** — everything SQLite does plus real concurrency; at the cost of a server
   dependency for a project with, today, zero concurrent writers.
3. **Property graph database (Neo4j class)** — traversal-optimized; heavy operational footprint,
   and the differentiating workload (rule application, proof construction) cannot be delegated
   to it anyway — it would accelerate the part that isn't the bottleneck.
4. **RDF triple store** — the right index layout natively; but qualifiers/status/provenance
   demand reification, and SPARQL is not our rule semantics.
5. **Custom storage engine** — maximal fit in theory; unjustifiable engineering and bus-factor
   cost with no evidence any requirement is unserved by existing engines.

## Decision

SQLite in WAL mode for all structured data (entities, grains, provenance, rules, status log),
with the three triple-permutation indexes (SPO/POS/OSP) imported from triple-store research;
source snapshots in a content-addressed file tree (git-object-style) outside the database; all
access through a storage interface so the engine choice stays swappable.

## Reasoning

- KISS is a stated project value, and SQLite is the simplest system meeting every actual
  requirement. Single-writer matches the architecture (one serialized admission point —
  [performance.md](../Performance/performance.md#parallelism-without-nondeterminism)) rather
  than constraining it.
- The read pattern is B-tree range scans, which SQLite does excellently; the known weakness
  (recursive CTE optimization) is off the hot path because hot closures are materialized and
  incrementally maintained by our own code regardless of engine.
- Inspectability is a real integrity feature: any engineer can audit the KB with the sqlite3
  CLI; a bespoke or niche format would make the audit trail practically unauditable.
- Single-file database + content-addressed archive gives a trivial, verifiable backup story
  (copy + re-hash).

## Tradeoffs

- A future multi-writer world (distributed acquisition) forces a migration to PostgreSQL —
  accepted, with the trigger condition written down rather than discovered in a crisis.
- No horizontal read scaling; irrelevant at design-phase scale.
- We forgo native graph-query convenience; our traversal code is explicit instead — which the
  explainability goal arguably prefers.

## Consequences

- Revisit triggers (defined in
  [storage-research.md](../Storage/storage-research.md#revisit-triggers-written-down-now-on-purpose)):
  concurrent writer processes become real, or working set outgrows one machine.
- A storage-interface specification precedes implementation (roadmap phase 1).
- The proof cache lives in a separate, fully rebuildable store — losing it must never lose
  knowledge.
