# Glossary

The single source of truth for Hourglass terminology. Every other document uses these terms
without re-defining them. A term is added here (via PR) before it is used anywhere else.

Terms are ordered from the smallest unit of the system to the largest.

---

### Grain

The smallest atomic verified unit of knowledge. A grain asserts exactly one fact: a subject
entity, a canonical relationship, and an object (entity or literal value), plus optional
qualifiers (time, place, measurement unit) and mandatory provenance. Grains are the only thing
Hourglass "knows." Full anatomy: [Knowledge_Model/grains.md](../Knowledge_Model/grains.md).

### Canonical Grain

The normalized representation of a fact. All equivalent surface language ("the Earth revolves
around the Sun", "our planet circles the Sun") resolves to the same canonical grain
(`Earth ORBITS Sun`), while each original wording is preserved as evidence attached to it.

### Entity

A unique identifiable object — a thing, person, place, concept, or class. Every entity has
exactly one canonical identifier; all names and aliases map to it. See
[Knowledge_Model/entities.md](../Knowledge_Model/entities.md).

### Relationship

A canonical connection between entities, drawn from a closed, documented vocabulary
(`INSTANCE_OF`, `PART_OF`, `CREATED_BY`, …). Equivalent verbs in source text ("made",
"developed", "invented") normalize to one relationship. See
[Knowledge_Model/canonical-relationships.md](../Knowledge_Model/canonical-relationships.md).

### Literal

A non-entity value appearing as the object of a grain: a number, date, string, or measurement
(value + unit). Literals are typed and normalized (dates to ISO 8601, measurements to SI).

### Qualifier

A key–value refinement attached to a grain that scopes when/where/how the fact holds
(e.g. `valid_from: 1991`). Qualifiers are part of the grain's identity: the same triple with
different qualifiers is a different grain.

### Source

The origin of a grain: the document, dataset, or record it was extracted from, with retrieval
time, version, and content hash. See [Knowledge_Model/provenance.md](../Knowledge_Model/provenance.md).

### Evidence

The verbatim excerpt(s) of a source that support a grain, kept for provenance. Evidence is never
deleted by compression; only redundant *structure* is compressed away.

### Rule

A deterministic inference: IF a pattern of grains holds, THEN a conclusion grain holds. Rules are
explicit, stored, versioned objects — never learned weights. See
[Reasoning/rule-system.md](../Reasoning/rule-system.md).

### Proof Chain

The complete derivation of a conclusion: a directed acyclic graph whose leaves are verified
grains, whose internal steps are rule applications, and whose root is the conclusion. Every
answer Hourglass gives must carry a recoverable proof chain. See
[Reasoning/proof-engine.md](../Reasoning/proof-engine.md).

### Knowledge Graph

The graph formed by all entities (nodes) and grains (edges), plus the rules defined over them.
The knowledge graph is the entire long-term memory of Hourglass.

### Acquirer

The subsystem that ingests sources and proposes candidate grains. Candidates are *not* knowledge
until verified. See [Research/parsing.md](../Research/parsing.md).

### Verifier

The subsystem that validates grains before admission and independently re-checks every proof
chain before an answer is released. The verifier is deliberately small and separate from the
proof engine (see the de Bruijn criterion in
[Reasoning/proof-engine.md](../Reasoning/proof-engine.md#verification)).

### Proof Engine

The subsystem that searches for proof chains answering a query, using forward and backward
chaining over grains and rules.

### Contradiction Engine

The subsystem that detects conflicting grains, marks them disputed, and preserves both sides with
their provenance. Nothing is silently overwritten. See
[Reasoning/contradiction-engine.md](../Reasoning/contradiction-engine.md).

### Renderer

The subsystem that converts a verified proof chain into human-readable language. The renderer may
never introduce information not present in the proof. It runs *after* proving, never instead
of it.

### Hourglass

The overall deterministic reasoning engine: acquirer, verifier, knowledge graph, proof engine,
contradiction engine, and renderer, arranged as
`Acquire → Verify → Canonicalize → Compress → Link → Prove → Answer`.

---

## Terms deliberately not used

- **Confidence score** — forbidden in the answer path. A statement is proven, disputed, or
  unknown. See [ADR-0003](../ADR/ADR-0003-unknown-over-guessing.md).
- **IS_A** — ambiguous between class membership and subclassing; split into `INSTANCE_OF` and
  `SUBCLASS_OF`. See
  [canonical relationships](../Knowledge_Model/canonical-relationships.md#why-there-is-no-is_a).
- **Training** — Hourglass acquires and verifies knowledge; it does not fit parameters.
