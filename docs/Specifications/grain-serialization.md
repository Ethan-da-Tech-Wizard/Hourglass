# Grain serialization and identity — normative

**Problem this spec fixes:** grain ids are content hashes
([ADR-0001](../ADR/ADR-0001-atomic-grains.md)). A content hash is only an identity if the bytes
hashed are defined *exactly* — same fact, same bytes, same id, on every machine, forever. Any
ambiguity here (field order, unicode form, number formatting) silently forks identities and
breaks deduplication, caching, and tamper detection at once. This is the one spec that can never
be "roughly right."

## Encoding primitives

- All text is **UTF-8**, Unicode-normalized to **NFC**, before any other rule applies.
- Fields are framed as **netstrings**: `<byte-length in ASCII decimal>:<bytes>,`
  (e.g. `5:Earth,`). Netstrings need no escaping and concatenate unambiguously — this removes
  the entire class of delimiter/escaping bugs by construction (KISS).
- A canonical form is the concatenation of its netstring fields in the exact order given below.
  No whitespace outside netstring payloads. No optional fields — absent means an empty
  netstring (`0:,`).
- Hash function: **SHA-256**. Ids are the lowercase hex digest with a type prefix.

## Identifier formats

| Object | Id format | Hashed over |
|---|---|---|
| Grain | `G-<64 hex>` | canonical grain form (below) |
| Source | `S-<64 hex>` | the raw snapshot bytes, exactly as retrieved |
| Rule | `R-<64 hex>` | normalized rule text ([rule-syntax.md](rule-syntax.md#normalization)) |
| Proof chain | `P-<64 hex>` | canonical proof form (below) |
| Entity | `E<decimal>` | *not hashed* — allocated sequentially by the single writer, never reused, never re-meaning-ed |
| Relationship | its declared name, e.g. `INSTANCE_OF` | *not hashed* — the closed vocabulary is the namespace; renames are forbidden (a rename is a new relationship + RFC) |

Entities are the one non-content-addressed identity, deliberately: an entity's "content" (its
labels) is mutable data *about* it, not its identity
([entities.md](../Knowledge_Model/entities.md#entity-model)).

## Canonical grain form

Nine netstrings, in this order:

```
1: format tag        "hourglass-grain-v1"
2: subject           entity id            e.g. "E42"
3: relationship      relationship name    e.g. "ORBITS"
4: object kind       "entity" | "number" | "quantity" | "date" | "string" | "boolean"
5: object value      canonical literal form (below), or entity id
6..9: qualifiers     count, then per qualifier (key, value-kind, value) — keys sorted
                     bytewise ascending; duplicate keys are invalid
```

Concretely, `Earth ORBITS Sun` with no qualifiers, where Earth=E7, Sun=E9:

```
18:hourglass-grain-v1,2:E7,6:ORBITS,6:entity,2:E9,1:0,
```

`G-…` = SHA-256 of exactly those bytes. **Status, provenance, and timestamps are NOT part of the
hashed form** — they are metadata about the grain, not the fact
([grains.md](../Knowledge_Model/grains.md#what-a-grain-is)).

## Canonical literal forms

| Kind | Canonical form | Rules |
|---|---|---|
| number | ASCII decimal | no exponent; no leading zeros; no trailing fractional zeros; no `+`; `-0` is invalid; integers carry no `.` |
| quantity | `<number> <unit>` | one space; unit is the declared SI (or RFC-declared) symbol, e.g. `8848.86 m` |
| date | ISO 8601 + `@` + precision | precision ∈ `year|month|day|hour|minute|second`; e.g. `1969-07-20@day`, `1969@year`; always UTC (`Z`) when time is present |
| string | the NFC text, verbatim | netstring framing makes escaping unnecessary |
| boolean | `true` / `false` | |

The **original** source form (locale numbers, imperial units, prose dates) lives in the evidence
excerpt, untouched — canonicalization is for identity, provenance keeps the raw truth.

## Format versioning

The format tag is *inside* the hashed bytes. Changing anything in this spec means a new tag
(`…-v2`), which re-keys — deliberately loudly. A store may hold grains under multiple tags
during a migration; the migration procedure ships with the RFC that changes the format
([Specifications/README.md](README.md#change-policy)).

## Hash stability vs. entity merges — the resolved conflict

Grain hashes include entity ids; entity merges re-target grains. As originally drafted these two
facts contradicted each other ("re-pointing" a grain would change its bytes and therefore its
id, orphaning provenance, caches, and audit history keyed to the old id).

**Resolution ([ADR-0006](../ADR/ADR-0006-entity-merge-supersession.md)): grain records are
immutable, ids included.** A merge of E-dup into E-keep is executed as *supersession*: for each
grain referencing E-dup, admit a new grain (new hash) with E-keep substituted and all provenance
carried over; the old grain's status becomes `superseded`, and the status log links old id →
new id. Nothing is edited in place, ever; `superseded` joins the status lifecycle between
`verified` and `retracted` in trust order (usable for *auditing history*, never as a proof
leaf).

## Canonical proof form

A proof chain serializes bottom-up (children before parents), each node as netstrings:

- leaf: `4:leaf,` + grain id
- rule application: `4:rule,` + rule id + binding count + per binding (variable, value) sorted
  by variable name + child count + child node hashes in the premise order declared by the rule

The node hash is SHA-256 of that node's bytes; `P-…` is the root node's hash. Two proofs are
identical iff their roots hash equal — this is what makes the proof cache and the
[deterministic proof ordering](../Reasoning/proof-engine.md#multiple-proofs-and-minimal-proofs)
well-defined (proofs are ordered by their canonical bytes as the final tie-break).

## Conformance

An implementation passes this spec when: (1) the worked-example KB
([Examples](../Examples/worked-example.md)) produces bit-identical ids on two independent
implementations or runs; (2) a fixture file of (fact, expected bytes, expected id) triples —
to be committed as `conformance/grains-v1.tsv` when Phase 1 starts — round-trips exactly.
The fixture file is the spec's executable form; disagreement between them is a release blocker.
