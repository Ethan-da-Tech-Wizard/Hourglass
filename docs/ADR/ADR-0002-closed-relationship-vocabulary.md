# ADR-0002: The canonical relationship vocabulary is closed, extended only by RFC

- **Status:** Accepted
- **Date:** 2026-07-03
- **Related:** [canonical-relationships.md](../Knowledge_Model/canonical-relationships.md),
  [RFC process](../RFC/README.md),
  [Wikidata prior art](../Research/prior-art.md#wikidata)

## Problem

Source language expresses connections with unbounded verb variety. If any string can become an
edge type, the graph fragments ("created"/"developed"/"invented" edges never unify), rules must
enumerate synonyms, and queries silently miss facts. Something must bound the edge vocabulary —
the question is how hard the boundary is.

## Options considered

1. **Open vocabulary** — any extracted verb becomes a relationship. Zero ingestion friction; but
   it recreates the fragmentation problem permanently and pushes synonym resolution onto every
   future query and rule. The compression and rule-generality goals die here.
2. **Open with post-hoc clustering** — mint freely, merge equivalents later (automatically or
   by review). Ingestion stays fast; but "later" is load-bearing: merging requires re-reading
   evidence to confirm meaning-equivalence, so the debt compounds, and automatic clustering is
   similarity-as-identity — a guess.
3. **Closed vocabulary, RFC-gated extension** — every relationship is declared (domain, range,
   properties, aliases, justification) and reviewed before use; unmapped verbs leave facts in
   candidate quarantine until the vocabulary catches up.

## Decision

Option 3. The vocabulary is closed; extensions go through the RFC process with a full
declaration; canonicalization maps source verbs to declared relationships via reviewed alias
lists; anything unmapped does not ingest as a verified grain.

## Reasoning

- Relationships are load-bearing schema: the rule engine trusts their declared properties
  (transitivity, functionality) and the entity resolver trusts their domain/range. Schema
  changes deserve review everywhere else in engineering; edges are schema here.
- Wikidata's property system — closed, proposal-gated, ~12k properties governing ~1.6B
  statements — is direct evidence that a closed vocabulary scales and that the review process,
  not the size limit, is what keeps a graph coherent.
- Merging two relationships later is mechanical (rewrite edge ids); *splitting* one that
  conflated two meanings requires re-reading all its evidence. The closed vocabulary makes the
  cheap mistake the likely one.
- The failure mode of option 3 is visible and bounded (a quarantine backlog and a pending RFC);
  the failure mode of options 1–2 is silent graph corruption.

## Tradeoffs

- Ingesting a new domain stalls on vocabulary RFCs — real friction, accepted deliberately.
- The RFC process needs to stay lightweight or it becomes the bottleneck that kills acquisition
  throughput; process weight is a monitored risk, not a fixed cost.
- Alias lists require continual curation (long-tail synonyms).

## Consequences

- A seed vocabulary ships with the design ([canonical-relationships.md](../Knowledge_Model/canonical-relationships.md#seed-vocabulary)).
- The acquirer must handle "verb not in vocabulary" as a normal, logged outcome feeding RFC
  prioritization.
- `IS_A` is permanently banned as a stored relationship (alias resolved by operand kinds to
  `INSTANCE_OF`/`SUBCLASS_OF`).
