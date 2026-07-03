# Answer envelope — normative

**Problem this spec fixes:** "every answer is proven or a classified refusal" must be a *shape*,
not a habit. This spec defines the only input the query path accepts and the only output it can
produce — making fabrication structurally unrepresentable at the API boundary
([ADR-0003](../ADR/ADR-0003-unknown-over-guessing.md)).

## Query

v1 queries are **structured goal patterns** — not natural language
([v1-scope.md](v1-scope.md#out--explicitly-not-in-v1)):

```
query :=
  goal        : (subject | ?var, relationship, object | ?var, qualifiers?)
  mode        : exists | first_proof | minimal_proof | enumerate(limit N)
  bindings_for: [?var, …]        (* for "what does Earth orbit?"-style questions *)
```

Examples:

- "Does Rex have lungs?" → `goal: (E-rex, HAS_PROPERTY, E-lungs), mode: first_proof`
- "What does Earth orbit?" → `goal: (E-earth, ORBITS, ?x), mode: enumerate(10), bindings_for: [?x]`

Exactly one relationship per goal; compound questions are the *client's* composition of queries
(KISS: the engine proves atoms; orchestration lives outside the trust boundary).

## Response

One record, always:

```
response :=
  outcome      : PROVEN | UNKNOWN | CONTRADICTED | INCOMPLETE | UNRESOLVABLE
  kb_version   : hash of the status-log head at snapshot time
  query_echo   : the query, canonicalized
  results      : [ { bindings, proof: ProofChain } ]      (* PROVEN only; else empty *)
  enumeration  : complete | truncated_at(N)               (* enumerate mode only *)
  conflicts    : [ conflict records + competing proofs ]  (* CONTRADICTED only *)
  blockers     : [ candidate-status grain ids ]           (* INCOMPLETE only *)
  error        : { class, detail }                        (* UNRESOLVABLE only:
                   unknown_entity | unknown_relationship | ill_typed | malformed *)
```

The five outcomes are the refusal taxonomy of
[unknown-handling.md](../Reasoning/unknown-handling.md#the-refusal-taxonomy), verbatim — this
spec is that document made into a type.

Field-level rules (MUST):

- `PROVEN` ⟹ at least one result, and every `proof` has been checked by the verifier *inside*
  the query path — an unchecked proof cannot leave the engine
  ([proof-engine.md](../Reasoning/proof-engine.md#verification)).
- `UNKNOWN` carries nothing but the outcome and `kb_version` — no "best guesses," no partial
  bindings, no similar-entity suggestions. (A *separate*, clearly-non-answer lookup API may
  offer entity search; it returns entities, never facts.)
- `CONTRADICTED` MUST include *all* sides' proofs/provenance, with corroboration counts
  displayed as counts ([contradiction-engine.md](../Reasoning/contradiction-engine.md#response-mark-preserve-explain--never-overwrite)).
- No field named `confidence`, `score`, `probability`, or `rank` exists in this schema, and
  adding one is a spec change requiring RFC — the ADR-0003 review rule, made greppable.

## kb_version

`kb_version` is the hash-chain head of the status log
([Security](../Security/integrity.md#immutable-history-the-hash-chained-audit-log)). It names
the exact belief-state the answer was computed against.

**The determinism contract, now testable:** for equal `kb_version` and equal canonicalized
query, the response MUST be byte-identical — across runs, machines, cache states, and
parallelism settings. This one sentence is the whole project's promise in executable form; it
is the first CI gate of Phase 2 ([roadmap](../Roadmap/roadmap.md#phase-2--rules-and-proof-engine)).

## Rendering

The renderer consumes exactly one `response` record and produces text. Per outcome it uses the
fixed phrasings of the refusal taxonomy; for `PROVEN` every emitted sentence maps to a proof
node ([Phase 4 gate](../Roadmap/roadmap.md#phase-4--renderer)). The renderer receives the
response by value — it holds no store handle of any kind
([storage-interface.md](storage-interface.md#two-handles-not-one) gives it none), so "the
renderer may never introduce information" is enforced by what it physically cannot reach.

## Conformance

`conformance/answers-v1/`: for the worked-example KB, a table of (query, expected full response
bytes) covering all five outcomes — the worked example already scripts PROVEN (Rex/lungs),
UNKNOWN (Rex/gills), and CONTRADICTED (Earth/Proxima); INCOMPLETE and UNRESOLVABLE fixtures are
one candidate-status grain and one unknown-entity query away. Byte equality, not structural
equality: formatting drift in a proof is identity drift somewhere upstream, and this test is
where it gets caught.
