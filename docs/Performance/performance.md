# Performance

**Problem:** determinism and provenance are non-negotiable; speed must be found *within* those
constraints. The risk runs both ways — a slow engine won't be used, and a fast-at-any-cost
engine will smuggle in nondeterminism (timeouts, races, cache-order effects).

**Why it matters:** the success criteria include compact storage and scalability, and several
"performance features" (caching, parallelism) are exactly where determinism traditionally leaks.
This document sets the rules before implementation exists to bend them.

**Standing rule: no optimization may change any answer.** Datalog's confluence gives us this for
evaluation strategy *by construction* ([rule-system.md](../Reasoning/rule-system.md#formal-basis-datalog-restricted));
everything below preserves it.

## Workload shape

Two very different paths ([overview](../Architecture/overview.md#the-seven-stage-pipeline)):

- **Write path** (acquisition): batch-friendly, latency-tolerant, throughput-bound. Heavy work
  (canonicalization, admission checks, contradiction detection, index maintenance) belongs here.
- **Read path** (queries): latency-sensitive, side-effect-free, embarrassingly cacheable.

The governing trade: **spend at write time to save at read time** — the KB is read far more
than written.

## Indexing

- **Grain indexes:** the three permutation indexes (SPO/POS/OSP) over the grain table —
  imported from triple-store research — make every backward-chaining probe a single B-tree
  range scan ([storage](../Storage/storage-research.md#recommendation-sqlite-first)).
- **Entity indexes:** alias → entity id (exact-match, the resolution hot path); class
  membership via the materialized `SUBCLASS_OF` closure.
- **Rule index:** THEN-pattern relationship → rules concluding it — backward chaining's "which
  rules can produce this goal?" lookup, kept in memory (the rule set is small by design).
- **Dependency index:** grain/rule id → dependent cached proofs and materialized facts. This is
  the TMS justification network ([proof-engine.md](../Reasoning/proof-engine.md#proof-caching-and-invalidation));
  it powers invalidation and contradiction blame, and it is maintained on the write path.

## Incremental updates and materialization

Never recompute from scratch what can be maintained incrementally:

- **Additions:** semi-naive evaluation — new derivations must use at least one new fact; the
  standard Datalog delta algorithm.
- **Deletions/retractions:** DRed (delete-and-rederive) or counting-based maintenance for the
  materialized hot closures. Counting is simpler and exact for non-recursive derivations; DRed
  handles the recursive closures. Choose per-materialization, document per-materialization.
- **Scope:** only declared-hot closures are materialized at all
  ([proof-engine.md](../Reasoning/proof-engine.md#query-evaluation-backward-chaining-with-selective-forward-materialization));
  everything else is computed on demand under tabling. Materialization is a cache with a
  maintenance algorithm, never a second source of truth.

## Proof caching

- Cache key: (goal pattern, bindings) → checked proof chain, tagged with every grain/rule id it
  references.
- Invalidation: exact, via the dependency index — any status change to a referenced id evicts.
  No TTLs, no heuristics: a stale proof is a wrong answer, so staleness is structurally
  impossible, not merely unlikely.
- Re-validation option: because proofs are checkable objects, a cache hit can be *re-verified*
  in linear time instead of trusted — the paranoid mode costs little and is the default until
  profiling argues otherwise.
- Storage: separate rebuildable store ([storage](../Storage/storage-research.md#future-work));
  losing the cache loses no knowledge.

## Parallelism without nondeterminism

- **Write path:** parallel acquisition/canonicalization workers feeding a single serialized
  admission point (one writer — matches SQLite's model and makes the audit log a total order).
- **Read path:** queries are independent and read-only — parallel across queries trivially.
  *Within* one query, parallel sub-goal exploration is allowed only with deterministic result
  selection: candidates complete in any order, but the returned proof is chosen by the defined
  total order on proofs ([proof-engine.md](../Reasoning/proof-engine.md#multiple-proofs-and-minimal-proofs)),
  never by completion order. Speed may vary run to run; answers may not.

## Memory and disk layout

- WAL-mode SQLite with memory-mapped reads; hot metadata (entities, rules, indexes) sized to
  page cache; snapshot archive on cold storage
  ([provenance](../Knowledge_Model/provenance.md#future-work)).
- **Lazy loading:** provenance excerpts and archived snapshots load only when a proof is
  *displayed* or audited — proving needs grain ids and statuses, not the evidence bytes.
- Compression: structural compression is the knowledge model's job
  ([grains.md](../Knowledge_Model/grains.md#structural-compression)); byte-level compression
  applies only to the cold archive (snapshots compress well and are read rarely).

## Measurement discipline

Before optimizing anything: a benchmark KB (generated, size-parameterized) and tracked metrics —
grains/sec admitted, p50/p99 query latency by proof depth, cache hit rate, invalidation fan-out,
bytes/grain on disk. "Storage remains compact" and "high performance" become numbers with
trend lines, or they are just adjectives. Regression gates in CI once implementation starts.

## Rejected alternatives

- **Query timeouts as a latency guarantee** — a timeout converts slowness into a wrong refusal
  ("Unknown" that isn't). Latency is managed by caching and materialization, never by giving up
  mid-proof.
- **Approximate/probabilistic indexes** (Bloom-filter negatives without verification, ANN
  lookups) in the answer path — approximation is guessing at the systems layer. Bloom filters
  are fine as *pre-filters* whose misses fall through to exact checks.
- **Distributed graph processing** now — no workload evidence; revisit triggers are defined in
  [storage-research.md](../Storage/storage-research.md#revisit-triggers-written-down-now-on-purpose).

## Future work

- Benchmark KB generator design.
- Tabling-engine memory policy (table space eviction interacts with determinism of *speed*,
  not answers — but must be characterized).
- Cost model for "declare this closure hot" decisions (query-frequency analytics driven).
