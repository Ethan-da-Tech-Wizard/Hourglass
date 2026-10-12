# Canonical relationships

**Problem:** natural language expresses the same connection in dozens of ways ("created",
"developed", "invented", "made", "authored"). If each verb becomes its own edge type, the graph
fragments: queries miss facts, rules multiply, deduplication fails.

**Why it matters:** relationship normalization is the second pillar of structural compression
(after entity resolution) and the precondition for a small, general rule set.

**Decision context:** the vocabulary is *closed* — a fixed, documented set, extended only through
the [RFC process](../RFC/README.md). See
[ADR-0002](../ADR/ADR-0002-closed-relationship-vocabulary.md) for the full reasoning; in short:
an open vocabulary re-creates the fragmentation problem one relationship at a time, and every
relationship the proof engine must traverse is code-adjacent — it deserves review.

## Relationship declaration format

Every canonical relationship is declared with:

```
id            : CREATED_BY
domain        : class the subject must belong to (or Any)
range         : class the object must belong to (or literal type)
properties    : transitive? symmetric? functional? inverse-functional?
inverse       : optional named inverse (CREATED_BY ↔ CREATOR_OF; store one, derive the other)
aliases       : surface verbs that normalize to it
why it exists : one paragraph — the queries/rules that need it
```

`domain`/`range` power entity disambiguation ([entities.md](entities.md#resolution-strategy-deterministic-first))
and admission checks. `properties` power the rule engine: transitivity and inverses are *generic
rules derived from declarations*, not hand-written per relationship.

## Seed vocabulary

| Relationship | Domain → Range | Properties | Normalizes (examples) | Why it exists |
|---|---|---|---|---|
| `INSTANCE_OF` | individual → class | — | "is a", "is one of the" | Class membership; the entry point of every inheritance rule |
| `SUBCLASS_OF` | class → class | transitive | "is a kind of", "is a type of" | Class hierarchy; carrier of rule inheritance |
| `PART_OF` | entity → entity | transitive | "is part of", "belongs to", "component of" | Mereology: engine PART_OF car; supports containment queries |
| `LOCATED_IN` | entity → place | transitive | "is in", "situated in", "found in" | Geographic containment: Paris LOCATED_IN France ⊢ … Europe |
| `CAUSES` | event/condition → event/condition | — (deliberately **not** transitive) | "causes", "leads to", "results in" | Causal claims; transitivity rejected because causal chains weaken with length — deriving `A CAUSES Z` through 10 hops asserts something no source said |
| `DEPENDS_ON` | entity → entity | — | "requires", "needs", "relies on" | Dependency reasoning; cycles allowed (mutual dependency exists in reality), so no DAG assumption |
| `OWNS` | agent → entity | — | "owns", "possesses", "property of" | Ownership; almost always needs `as_of` qualifier (ownership changes) |
| `USES` | agent/artifact → entity | — | "uses", "utilizes", "employs", "built with" | Usage/technology relations |
| `HAS_PROPERTY` | entity/class → property or literal | — | "has", "is [adjective]" | Attribute assignment; on a *class* subject it is the payload that inheritance rules distribute |
| `CREATED_BY` | artifact → agent | — | "created", "developed", "invented", "made", "authored", "designed" (inverted) | Attribution; direction fixed artifact→agent so every source phrasing lands identically |
| `ORBITS` | astronomical body → astronomical body | — | "orbits", "revolves around", "circles" | Present because it is the project's running example; also a template for domain-specific physical relations |

Literal-valued attributes (population, mass, dates) use `HAS_PROPERTY`-style specialized
relationships (`POPULATION`, `MASS`, `DATE_OF_BIRTH`) added by RFC as domains are ingested — a
flat `HAS_PROPERTY(x, "mass", 5.97e24 kg)` triple-with-string-key was rejected because it dodges
domain/range checking.

## Why there is no `IS_A`

"X is a Y" conflates two different relations with different logic:

- `Socrates IS_A human` — membership → `INSTANCE_OF` (not transitive: Socrates is not a
  *subclass* of anything).
- `Human IS_A mammal` — subclassing → `SUBCLASS_OF` (transitive).

Mixing them is the classic ontology bug (documented since Brachman's 1983 "What IS-A is and
isn't"): it lets `Socrates SUBCLASS_OF mammal` type-check and derivations go wrong silently.
`IS_A` is therefore an *alias* that canonicalization must resolve to one of the two using the
kinds of the operands — never a stored relationship.

## Normalization semantics

- Alias → canonical mapping happens at [pipeline stage 3](../Architecture/overview.md#3-canonicalize).
  The original verb is preserved in the evidence excerpt; nothing about the source is lost.
- Mapping must be **meaning-preserving, not merely plausible**. "Discovered" is *not* an alias
  of `CREATED_BY` (discovery ≠ creation); it waits for a `DISCOVERED_BY` RFC. When in doubt, a
  new relationship is proposed rather than a lossy merge — merging relationships later is easy
  (rewrite edges), splitting them is hard (re-read all the evidence).
- Directionality is fixed per relationship. Passive/active voice both normalize to the declared
  direction ("X was created by Y" and "Y created X" → `X CREATED_BY Y`).

## Tradeoffs

- A closed vocabulary means ingestion of a new domain stalls until its relationships pass RFC.
  Accepted: the stall is visible and bounded; vocabulary sprawl is forever.
- Declared properties (transitive etc.) are trusted by the rule engine, so a wrong declaration
  corrupts many proofs. Mitigation: property declarations require an ADR-grade justification in
  the RFC, and the worked example for the relationship must include a case the property enables.

## Rejected alternatives

- **Open vocabulary (any string is an edge)** — see ADR-0002.
- **Adopting OWL/RDFS wholesale** — powerful but drags in open-world semantics and description-
  logic complexity that conflict with our unknown-handling and KISS goals; we borrow the
  *concepts* (domain/range, transitivity, inverse) without the stack
  ([prior art](../Research/prior-art.md#semantic-web-rdf-rdfs-owl)).
- **Automatic relation extraction defining new relationships** — new edge types minted by an
  extractor are unreviewed schema changes; forbidden.

## Future work

- Qualifier schemas per relationship (e.g. `OWNS` requiring `as_of`).
- A documented decision procedure for "alias vs. new relationship" with worked borderline cases
  (discovered/invented, wrote/composed).
