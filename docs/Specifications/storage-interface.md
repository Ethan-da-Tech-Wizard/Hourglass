# Storage interface — normative

**Problem this spec fixes:** "all codebases line up" is achieved by making every subsystem meet
storage at one narrow, named interface. No subsystem may touch SQL, files, or caches directly;
the engine choice stays swappable ([ADR-0004](../ADR/ADR-0004-sqlite-first-storage.md)) and the
architecture's read/write separation is enforced by *types*, not discipline.

## Two handles, not one

The interface is split so that misuse is unrepresentable:

- **`ReadStore`** — all lookup operations. This is the *only* handle the proof engine, verifier,
  contradiction checks, and renderer-adjacent code ever receive.
- **`WriteStore`** — extends ReadStore with admission/status operations. Exactly one WriteStore
  exists per KB, held by the single-threaded admission loop
  ([performance.md](../Performance/performance.md#parallelism-without-nondeterminism)).

There is **no delete operation in either interface.** Not "don't call it" — it does not exist
([grains.md](../Knowledge_Model/grains.md#what-a-grain-is), [Security](../Security/integrity.md#immutable-history-the-hash-chained-audit-log)).

## ReadStore operations

```
grain(id)                      → grain record | absent
grains_matching(pattern)       → ordered stream of grain records
    pattern = (subject | ?, relationship | ?, object | ?, qualifiers | ?)
    ordering: ascending grain id (total, stable — determinism depends on this)
entity(id)                     → entity record (follows no redirects; what you ask is what you get)
resolve_alias(text, class?)    → exact-match entity ids (0, 1, or many; caller handles ambiguity)
canonical_entity(id)           → id after following merge redirects to the live entity
relationship(name)             → declaration record (domain, range, properties, aliases)
rules_concluding(relationship) → verified rules whose THEN uses it, ascending rule id
rule(id)                       → rule record | absent
source(id)                     → source record | absent
evidence(grain_id)             → provenance entries (loads excerpts lazily — hot path never needs bytes)
status(id)                     → current status of any grain/rule/entity
status_log(after_seq?)         → ordered stream of status-log records (audit walk)
closure(relationship, from_id) → ids reachable via the materialized closure (SUBCLASS_OF in v1)
kb_version()                   → hash of the status-log head (defined in answer-envelope.md)
```

Guarantees every ReadStore implementation MUST provide:

1. **Snapshot reads.** A logical operation (one proof search) sees one KB version throughout —
   SQLite WAL gives this per read-transaction; the interface makes it explicit.
2. **Deterministic ordering.** Every multi-result operation returns a defined total order.
   No operation ever exposes hash-map/insertion order.
3. **Status honesty.** `grains_matching` returns records *with* status; filtering to `verified`
   is the caller's stated act (the proof engine filters; the contradiction engine deliberately
   doesn't).

## WriteStore operations

```
admit_source(kind, locator, retrieved_at, version, snapshot_bytes, signature?) → source id
admit_grain(canonical fields, provenance entries)   → grain id        (status: candidate)
admit_rule(normalized text, why, provenance)        → rule id         (status: candidate)
admit_entity(kind, labels, description)             → entity id
add_alias(entity_id, text, source_id)
set_status(id, new_status, actor, reason)           → status-log seq  (the ONLY mutation)
merge_entity(dup_id, keep_id, actor, reason)        → supersession report (per ADR-0006: new
                                                      grain ids minted, old ones → superseded)
```

Guarantees:

1. **Atomicity.** `admit_grain` lands grain + provenance + status-log entry in one transaction
   or not at all. `merge_entity` is one transaction for the whole supersession set.
2. **Append-only audit.** Every operation above writes exactly one (or, for merge, N linked)
   hash-chained status-log records ([Security](../Security/integrity.md#immutable-history-the-hash-chained-audit-log)).
3. **Validation at the door.** `admit_*` MUST reject: unknown relationships, ill-typed literals,
   domain/range violations, malformed canonical bytes, duplicate grain ids (which instead
   append provenance to the existing grain and report it — deduplication is an admission
   behavior, not a caller concern).
4. **Status transitions are checked** against the lifecycle graph
   ([grains.md](../Knowledge_Model/grains.md#grain-lifecycle)); illegal transitions
   (retracted → verified) are errors.

## What is deliberately NOT in the interface

- **No free-form query pass-through** (no `execute_sql`). The moment one exists, every caller
  couples to the engine and the swap-ability of ADR-0004 is fiction.
- **No proof-cache operations.** The cache is a separate, fully-rebuildable store owned by the
  proof engine ([performance.md](../Performance/performance.md#proof-caching)); conflating it
  with knowledge storage would let a cache bug corrupt knowledge.
- **No bulk update / transform operations.** Every change is an admission or a status change,
  each individually audited. Bulk ingestion is a loop, and its slowness is write-path slowness,
  which we explicitly accept ([performance.md](../Performance/performance.md#workload-shape)).

## Conformance

The interface ships (Phase 1) with a conformance test suite runnable against any implementation:
lifecycle legality, ordering determinism (two runs, byte-identical streams), atomicity under
injected mid-transaction crash, audit-chain verification after every mutation, and the
worked-example scenario end-to-end. The in-memory reference implementation and the SQLite
implementation MUST both pass it; that pair keeps the interface honest (an interface with one
implementation quietly becomes that implementation).
