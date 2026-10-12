# Storage research

**Problem:** choose where grains, entities, rules, sources, and proof caches live. The choice
constrains everything operational: traversal speed, audit trail durability, backup story,
contributor onboarding.

**Why it matters:** storage is the easiest place to buy complexity we don't need (distributed
graph clusters) or to paint ourselves into a corner (a bespoke format nobody can inspect).

## Requirements, derived from the design

1. Store ~10⁶–10⁹ grains (triples + qualifiers), entities, rules, sources, provenance entries.
2. Fast lookups by subject / by (subject, relationship) / by object — the access pattern of
   backward chaining ([proof-engine.md](../Reasoning/proof-engine.md)).
3. Recursive traversal (transitive closures) with acceptable performance.
4. ACID: a grain and its provenance must land atomically; the audit log must not lie.
5. Deterministic behavior; simple operational story; inspectable by a competent engineer with
   standard tools (KISS).
6. Immutable history support (status changes, never destructive updates).

## Candidate comparison

| | Simplicity | Speed (our workload) | Memory | Disk | Maintainability | Proof efficiency |
|---|---|---|---|---|---|---|
| **SQLite** | ★★★★★ single file, zero config | ★★★★ superb point lookups; recursive CTEs adequate | ★★★★ pages + mmap option | ★★★★ compact; standard vacuum | ★★★★★ ubiquitous tooling, stable format for decades | ★★★★ with the right indexes (below) |
| **PostgreSQL** | ★★★ server, roles, config | ★★★★ similar + better concurrency | ★★★ | ★★★★ | ★★★★ mature, but an ops dependency | ★★★★ |
| **Property graph DB (Neo4j etc.)** | ★★ JVM server, own query language | ★★★★★ traversal-optimized | ★★ | ★★★ | ★★ niche expertise, license questions | ★★★★ traversal wins, but proof needs rule application, not just pathfinding — the DB can't do the part that matters |
| **RDF triple store (Jena/Virtuoso/Oxigraph)** | ★★ | ★★★★ SPO/POS/OSP indexing is exactly right | ★★★ | ★★★ | ★★ | ★★★ SPARQL lacks our rule semantics; reification needed for qualifiers/status fights the model |
| **Compressed static graph formats (HDT, CSR)** | ★★★ | ★★★★★ read | ★★★★★ | ★★★★★ | ★★★ | ★★ read-only — no admission pipeline, no status changes |
| **Custom engine** | ★ | ？ | ？ | ？ | ★ bus-factor disaster | ？ |

Key insight from the comparison: our differentiating workload — *rule application and proof
construction* — lives in the proof engine regardless of storage. No database performs it for us.
Therefore storage should be chosen on simplicity, integrity, and index fit, not on "graph"
branding. The triple-store literature's lasting contribution is the **index layout** (multiple
permutation indexes), which is portable to any B-tree store.

## Recommendation: SQLite first

Decision recorded in [ADR-0004](../ADR/ADR-0004-sqlite-first-storage.md). Sketch of the schema:

```sql
entities  (id PRIMARY KEY, kind, status, description)
labels    (entity_id, lang, text, is_preferred)         -- + aliases with source_id
grains    (id PRIMARY KEY,            -- content hash
           subject_id, relationship_id, object_kind, object_ref, qualifiers_canon,
           status, created_at)
provenance(grain_id, source_id, excerpt, offsets, extracted_at, method, method_version)
sources   (id PRIMARY KEY,            -- content hash of snapshot
           kind, locator, retrieved_at, version, snapshot_ref, signature)
rules     (id PRIMARY KEY, body_canon, status, why, created_at)
status_log(object_id, old_status, new_status, at, actor, reason)   -- append-only
```

Index plan (the SPO/POS/OSP idea, relationally):

```sql
CREATE INDEX g_spo ON grains(subject_id, relationship_id, object_ref);
CREATE INDEX g_pos ON grains(relationship_id, object_ref, subject_id);
CREATE INDEX g_osp ON grains(object_ref, subject_id, relationship_id);
```

Backward chaining's typical probe — "all grains with subject S and relationship R" — is one
B-tree range scan. Transitive closures: recursive CTEs for cold paths; the declared-hot closures
are materialized and incrementally maintained anyway
([proof-engine.md](../Reasoning/proof-engine.md#query-evaluation-backward-chaining-with-selective-forward-materialization)),
so SQLite's mediocre recursive-query optimizer is off the hot path by design.

Also fits: WAL mode (crash-safe, readers don't block the writer — matching our
one-write-pipeline/many-readers architecture), memory-mapped reads, `PRAGMA
integrity_check` for corruption audits, and single-file backups (copy = snapshot).

### Archive store

Source snapshots (bytes) do **not** go in SQLite rows: a content-addressed directory tree
(`archive/ab/cd/abcd…` named by hash) keeps the database small and hot while snapshots stay
verifiable by re-hashing. Same pattern as Git's object store — prior art enough.

## Revisit triggers (written down now, on purpose)

Move to PostgreSQL when *either*: multiple concurrent writer processes become real (distributed
acquisition workers), or the working set outgrows one machine's page cache and read replicas are
needed. Move the grain table to a specialized layout only if profiling shows index scans — not
rule application — dominate query time. Any move is mechanical because every consumer goes
through a storage interface, not raw SQL.

## Rejected alternatives (summary; reasoning above)

- **Neo4j/property graphs:** operational weight for a traversal advantage our workload doesn't
  center on; the proof engine can't be delegated to it anyway.
- **RDF stores:** the qualifier/status/provenance model demands reification gymnastics; SPARQL
  is not our rule language; we take their index layout instead.
- **HDT/CSR compressed graphs:** wrong mutability model for a living admission pipeline;
  attractive later as a *distribution* format for read-only KB releases (future work).
- **Custom engine now:** premature in the strongest sense; nothing in the requirements is
  unserved by SQLite + files.

## Future work

- Storage interface definition (the API the proof engine and pipeline code against).
- KB release/distribution format study (HDT-like read-only snapshot exports).
- Proof-cache store: likely a separate SQLite file (different lifecycle: fully rebuildable) —
  [performance.md](../Performance/performance.md#proof-caching).
