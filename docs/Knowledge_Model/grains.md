# Grains: the atomic unit of knowledge

**Problem:** we need a unit of knowledge that is (a) small enough to deduplicate and reason over
mechanically, (b) rich enough to carry time, measurement, and provenance, and (c) stable enough
that the same fact from ten sources becomes one stored object.

**Why it matters:** every subsystem — storage, proof engine, contradiction engine, renderer —
operates on grains. If the grain is wrong, everything downstream is wrong.

## What a grain is

A grain asserts exactly one fact:

```
subject      : entity id
relationship : canonical relationship id
object       : entity id | typed literal
qualifiers   : { key → typed literal }        (optional, part of identity)
```

plus metadata that is *not* part of the fact's identity:

```
grain id     : content hash of (subject, relationship, object, qualifiers) in canonical byte form
status       : candidate | verified | disputed | retracted
provenance   : list of (source id, evidence excerpt, extracted_at, extraction method)
created_at   : first time this grain entered the system
```

Design points:

- **Grain id = content hash.** Deterministic identity: the same canonical fact hashes to the
  same id no matter which source produced it, which makes deduplication a lookup, not a search.
  It also gives tamper-evidence for free ([Security](../Security/integrity.md)).
- **Provenance is a list.** Ten sources asserting `Earth ORBITS Sun` produce one grain with ten
  provenance entries. Corroboration is countable and displayable without ever becoming a
  "confidence score."
- **Status is the only mutable field.** Grains are never edited or deleted; they change status.
  Retraction keeps the grain and its history ([contradiction engine](../Reasoning/contradiction-engine.md)).

## Why triples + qualifiers, not paragraphs and not n-ary tuples

Approaches considered:

| Approach | Advantages | Disadvantages |
|---|---|---|
| Store sentences/paragraphs | Trivial ingestion; nothing lost | No deduplication; no mechanical reasoning; equivalent wordings never unify; this is what retrieval systems do and it cannot support proof |
| Plain triples (RDF-style) | Simple; huge prior art; easy indexing | Cannot express time ("population *in 2020*"), units, or context without reification gymnastics |
| Full n-ary tuples per relation | Expressive | Every relation gets its own schema; storage and proof engine must handle arbitrary arity; complexity spreads everywhere (violates KISS) |
| **Triples + qualifier map** | One uniform shape; time/units/context expressible; deduplication still works | Slightly more complex identity (qualifiers are part of the hash) |

**Decision:** triples + qualifiers ([ADR-0001](../ADR/ADR-0001-atomic-grains.md)). This is the
model Wikidata converged on after a decade of operating the largest public knowledge graph
(statement = property–value + qualifiers + references), and its fit is empirical evidence, not
fashion — see [prior art](../Research/prior-art.md#wikidata).

Qualifiers are part of grain identity because `population = 8,000,000 [as_of: 2020]` and
`population = 8,400,000 [as_of: 2024]` are *different facts*, not a contradiction. The
contradiction engine only compares grains whose subject, relationship, and qualifiers match.

## Literals

Objects that are not entities are typed literals, normalized at canonicalization:

| Type | Normal form | Example |
|---|---|---|
| number | decimal, no locale formatting | `8804` |
| quantity | value + SI (or declared) unit | `8848.86 m` |
| date/time | ISO 8601, with explicit precision | `1969-07-20` (day precision) |
| string | NFC Unicode, verbatim | `"Hamlet"` |
| boolean | true/false | `true` |

Precision is stored explicitly (`1969` at year precision is not a contradiction of `1969-07-20`).
Unit conversion happens at canonicalization, and the original evidence keeps the original form.

## Structural compression

Hourglass compresses **structure, not language**, and never the information required for proof:

1. **Grain deduplication** — content-hash identity makes equivalent facts one row; evidence
   accumulates on it.
2. **Entity deduplication** — all aliases resolve to one entity id
   ([entities.md](entities.md)).
3. **Relationship normalization** — "created / developed / invented / authored" → `CREATED_BY`
   ([canonical-relationships.md](canonical-relationships.md)).
4. **Rule inheritance** — `Dog has lungs`, `Cat has lungs`, `Horse has lungs` are never stored;
   `Mammal HAS_PROPERTY lungs` plus the inheritance rule derives them on demand
   ([rule-system.md](../Reasoning/rule-system.md)). This is the largest single compression
   mechanism, and it is exact, not lossy.
5. **Shared proof sub-DAGs** — proof chains reference grain ids; common sub-proofs are stored
   once in the proof cache ([performance](../Performance/performance.md)).

Explicitly **not** compressed: evidence excerpts and source records. They are the ground truth
that proofs bottom out in. Disk spent on provenance is the cost of being able to say *why*.

## Grain lifecycle

```
            admission checks pass                    conflict detected
 candidate ─────────────────────────▶ verified ─────────────────────────▶ disputed
     │                                   │                                   │
     │ rejected (reason logged)          │ superseded / source withdrawn     │ resolved
     ▼                                   ▼                                   ▼
  (kept in quarantine log)           retracted ◀──────────────────────  verified | retracted
```

Only `verified` grains may appear as leaves of a proof chain. The proof engine treats `disputed`
grains as unusable and reports "Contradictory evidence exists" if a query can only be answered
through them ([unknown handling](../Reasoning/unknown-handling.md)).

## Rejected alternatives

- **Storing embeddings alongside grains** — rejected for the core: similarity is not identity,
  and nothing in the proof path may depend on approximate matching. (May be revisited strictly
  as an *acquisition-time suggestion* mechanism, outside the trusted path — see
  [parsing](../Research/parsing.md#where-probabilistic-methods-may-be-required).)
- **Mutable grains with an edit log** — rejected: status-only mutation is simpler to reason
  about and makes audit trivial.
- **Free-form relationship strings** — rejected; see
  [ADR-0002](../ADR/ADR-0002-closed-relationship-vocabulary.md).

## Future work

- Negative grains (`Earth NOT ORBITS Moon`): needed for stating known falsehoods explicitly;
  interacts with unknown handling and stratified negation. Deferred — see
  [rule-system.md](../Reasoning/rule-system.md#negation).
- Grain schemas per relationship (required qualifiers, e.g. `POPULATION` requiring `as_of`).
- Compression measurement: define the metric (grains stored vs. facts derivable) before
  implementation so compression claims are testable.
