# Parsing: from sources to candidate grains

**Problem:** knowledge arrives as documents, tables, and records; Hourglass consumes canonical
grains. Something must bridge the gap — and that something is the part of the pipeline most
tempted to guess.

**Why it matters:** parsing quality bounds knowledge growth (the expert-systems acquisition
bottleneck — [prior art](prior-art.md#symbolic-ai-and-expert-systems)), and parsing is where
probabilistic methods knock loudest. The directive is explicit: investigate deterministic
methods first; if insufficient, document exactly where and why probabilistic help is needed.
This document does both.

## Framing that changes the problem: output is *candidates*

The acquirer's output is **candidate grains**, quarantined until verification
([overview](../Architecture/overview.md#1-acquire)). Parsing therefore does not need to be
perfect — it needs to be *honest*: propose only what the evidence excerpt actually supports,
attach the excerpt, and refuse to propose when unsure. Precision over recall, always: a missed
fact is a gap (visible, fixable by better sources or patterns); a wrong admitted fact is
corruption.

## Source difficulty ladder

| Tier | Source type | Extraction character |
|---|---|---|
| 1 | Databases, CSV/tables with schemas, APIs | Column → relationship mapping written once per schema; fully deterministic; ~zero ambiguity |
| 2 | Semi-structured documents: infoboxes, key-value blocks, citations, standards documents | Template extractors per format; deterministic; ambiguity limited to entity resolution |
| 3 | Controlled prose: encyclopedic first sentences, definitions, glossaries | Pattern-based extraction (below); deterministic with modest recall |
| 4 | Open prose: narrative text, news, papers' discussion sections | Full NLP problem: ambiguity, coreference, implicature |

**Strategy: descend the ladder only as needed.** Tiers 1–2 contain enormous factual mass
(scientific databases, gazetteers, bibliographic records, Wikidata itself) and can be ingested
with fully deterministic extractors. Hourglass's early knowledge growth is planned there — not
because prose is unimportant, but because tier-1/2 facts arrive with structure that *is*
evidence. Tier 4 is explicitly out of scope until the rest of the engine exists.

## Deterministic methods for tier 3

- **Lexico-syntactic patterns** (Hearst patterns and descendants): "X is a Y that…",
  "X, such as Y and Z", "X consists of Y" — each pattern maps to a canonical relationship and
  fires only on exact structural match. Well-studied since 1992; high precision, low recall —
  the right trade for us.
- **Copular-definition parsing** for glossaries/encyclopedic openers ("Mercury is the first
  planet from the Sun") — a small grammar over a deterministic parse.
- **Deterministic normalizers** for dates, quantities, and units (ISO 8601, SI) — solved,
  library-grade problems.
- **Apposition and parenthetical patterns** for aliases ("Mercury (the Roman god)") feeding
  entity alias lists, not fact grains.

Every extractor states its pattern id and version in the grain's provenance
([provenance.md](../Knowledge_Model/provenance.md#per-grain-provenance-entry)), so a bad pattern's
output is recallable — extractors are treated like compiler versions.

## The documented hard problems

Honest accounting of where deterministic methods strain, per the directive:

| Challenge | Deterministic answer | Residual gap |
|---|---|---|
| **Ambiguity** (structural/word sense) | Fire only on unambiguous structural matches; else skip | Recall loss in tier 3–4; acceptable |
| **Pronouns/coreference** | Don't resolve; patterns spanning a pronoun don't fire | Large recall loss in tier 4 prose — the single biggest gap |
| **Synonyms** | Alias tables for entities ([entities.md](../Knowledge_Model/entities.md)) and relationship alias lists ([canonical-relationships.md](../Knowledge_Model/canonical-relationships.md)); curated, versioned | Long-tail synonyms need continual curation |
| **Dates** | ISO 8601 + explicit precision; relative dates ("last Tuesday") resolved only against the source's own timestamp, else skipped | Vague expressions ("in ancient times") — skipped, correctly |
| **Measurements** | Unit-aware normalization to SI; original preserved in evidence | Ambiguous units ("tons") require source-locale rules or skip |
| **References/citations** | Citation graph extracted deterministically (tier 2 problem); claims inherit *their* source, i.e. cited-paper claims are provenance-attributed to the cited paper only when the citation is itself retrieved | Claim-to-citation alignment in prose is tier-4 hard |

## Where probabilistic methods may be required — and the containment rule

Deterministic parsing is *sufficient for tiers 1–3* and insufficient for tier 4 (open-prose
relation extraction and coreference are unsolved deterministically; forty years of NLP says
so). If tier 4 recall ever justifies the complexity, the containment rule is:

> A statistical model may **propose** candidate grains (with evidence spans). It may never
> admit, verify, canonicalize, or appear in a proof. Its proposals enter the same quarantine as
> every other candidate and pass the same evidence-based verification — a human or a
> corroborating deterministic extraction confirms the excerpt actually asserts the grain.

Under this rule a neural extractor is a *suggestion engine for curators*, structurally identical
to the fuzzy entity-merge proposer ([entities.md](../Knowledge_Model/entities.md#resolution-strategy-deterministic-first)):
useful, untrusted, outside the proof path. The philosophy is preserved because verification —
not extraction — is what confers grain status.

## Rejected alternatives

- **LLM-based extraction as the primary acquirer** — maximum recall, but proposals arrive
  fluent and unmoored from evidence spans; verifying them costs more than pattern extraction
  saves, and the failure mode (plausible fabricated grains flooding quarantine) attacks the
  system's weakest joint. Reconsider only under the containment rule, only for tier 4.
- **Ingesting raw sentences as grains "to canonicalize later"** — deferred canonicalization
  never happens; violates the grain philosophy directly.

## Future work

- Extractor conformance suite: per pattern, a test corpus with expected grains — extractors are
  code and get regression tests.
- Tier-2 format library roadmap (which infobox/record formats first — driven by the refusal
  analytics of [unknown-handling.md](../Reasoning/unknown-handling.md#future-work)).
- Study of deterministic coreference subsets (e.g. same-sentence appositive antecedents) that
  may safely recover some tier-4 recall.
