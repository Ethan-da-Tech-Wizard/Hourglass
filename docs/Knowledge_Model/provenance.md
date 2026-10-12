# Provenance and source tracking

**Problem:** a proof chain bottoms out in grains; grains must bottom out in something outside the
system. If we cannot say where a grain came from, when, and in what form, then "proven" only
means "derivable from things we chose to believe" — traceability breaks at the last step.

**Why it matters:** source traceability is a success criterion of the project ("Can every grain
be located?"). It is also the substrate for the contradiction engine (which source said what,
when) and for security (tamper detection, poisoning response).

## Source model

```
source id      : content hash of the retrieved bytes (or record snapshot)
kind           : document | dataset | database record | table | manual entry
locator        : URL / DOI / ISBN / file path / query — how to find it again
retrieved_at   : timestamp of acquisition
version        : source-declared version or edition, if any
snapshot       : the retrieved bytes, archived (or a pointer into an archive store)
signature      : optional publisher signature ([Security](../Security/integrity.md))
```

Design points:

- **Sources are immutable snapshots.** "The Wikipedia article on Mercury" is not a source; *the
  bytes retrieved at time T* are. If the page changes, a new acquisition creates a new source.
  This is what makes "outdated information" a solvable versioning problem instead of a mystery
  ([contradiction engine](../Reasoning/contradiction-engine.md#outdated-information)).
- **Content-hash ids** make source identity deterministic and tampering detectable: re-hash the
  snapshot, compare.
- **Manual entry is a source kind**, not a backdoor: a human asserting a grain creates a source
  record naming the human and the timestamp, subject to the same admission checks.

## Per-grain provenance entry

Each grain carries one or more provenance entries
([grains.md](grains.md#what-a-grain-is)):

```
source id          : which snapshot
evidence excerpt   : verbatim span(s) supporting the fact, with offsets into the snapshot
extracted_at       : when the acquirer produced the candidate
extraction method  : which parser/pattern/rule produced it, and its version
verified_at        : when admission checks passed, and which checks ran
```

`extraction method` matters more than it looks: when a parser bug is found, every grain it
produced can be located and re-verified. This is the knowledge-base equivalent of a compiler
recording which version compiled each object file.

## What this gives each success criterion

| Question | Answered by |
|---|---|
| Where did this grain come from? | provenance → source id → locator + snapshot |
| When was it obtained? | `retrieved_at` / `extracted_at` |
| What version does it belong to? | source `version` + snapshot hash |
| Has it been verified? | grain status + `verified_at` + checks run |
| What are its dependencies? | for derived conclusions: the proof chain; for base grains: none — that is what "base" means |
| What supports it? | evidence excerpts, verbatim |

## Corroboration, not confidence

Multiple independent sources asserting the same canonical grain append provenance entries to the
one grain. The system can report "asserted by 4 independent sources" and list them. It never
converts that count into a probability, and corroboration count never changes what is provable —
one verified source admits a grain; zero sources means no grain
([ADR-0003](../ADR/ADR-0003-unknown-over-guessing.md)). Admission policy *may* require N
independent sources for certain source kinds (that is a verification gate, applied before
grain status becomes `verified`, not a score attached to answers).

"Independent" needs a real definition (mirrors and syndicated copies are not independent);
deferred to future work, tracked here so it is not forgotten.

## Prior art borrowed

- **W3C PROV** — the Entity/Activity/Agent decomposition maps cleanly: snapshot = prov:Entity,
  extraction = prov:Activity, parser or human = prov:Agent. We adopt the decomposition and the
  vocabulary of derivation without adopting RDF serialization
  ([prior art](../Research/prior-art.md#provenance-systems-w3c-prov)).
- **Scientific citation practice** — evidence excerpts with locators are quotations with page
  numbers; the design goal is that a human can check any grain the way a referee checks a
  citation.

## Rejected alternatives

- **Storing only locators (URLs) without snapshots** — link rot and silent edits destroy
  reproducibility; a proof that bottoms out in a 404 is not a proof.
- **Trust scores per source** — source reliability weighting is a confidence score in disguise.
  Source *policies* (e.g. "this source kind requires corroboration") are allowed because they
  gate admission deterministically; per-source numeric trust that scales belief is not.
- **Deduplicating evidence excerpts away during compression** — provenance is exempt from
  compression by design ([grains.md](grains.md#structural-compression)).

## Future work

- Archive store design: snapshots dominate disk usage; needs tiering (hot metadata in the main
  store, snapshot bytes in content-addressed cold storage) —
  [Storage](../Storage/storage-research.md#archive-store).
- Independence criteria for corroboration (syndication detection, shared-upstream detection).
- Source withdrawal procedure: what happens to grains whose only source is retracted by its
  publisher (cascade: grain → retracted, dependent proof caches invalidated).
