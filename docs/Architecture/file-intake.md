# File intake: where uploaded information sits

**Problem:** knowledge arrives as files — PDFs, spreadsheets, CSVs, HTML, JSON exports, images,
eventually anything a user drops in. Each needs a permanent, tamper-evident home; most need
*conversion* before extraction (a PDF is not text; an XLSX is not a table); and some conversions
(OCR) are probabilistic. This document pins where every byte sits and which conversions are
trusted.

**Why it matters:** the source snapshot is the ground truth every proof bottoms out in
([provenance.md](../Knowledge_Model/provenance.md)). If intake is sloppy — files edited in
place, conversions overwriting originals, OCR text masquerading as the document — provenance is
corrupt at the root and nothing downstream can fix it.

## The invariant

> **The original uploaded bytes are stored exactly once, exactly as received, addressed by
> their hash, and never modified. Everything derived from them is a separate, rebuildable,
> provenance-linked artifact.**

Everything in this document is that sentence applied to specific formats.

## Physical layout

```
archive/                          # ORIGINALS — ground truth, append-only, backed up
└── ab/cd/abcdef…                 #   file named by its SHA-256 (= source id S-abcdef…),
                                  #   two-level fan-out dirs; bytes verbatim, any format

renditions/                       # DERIVED — rebuildable cache, deletable without loss
└── S-abcdef…/                    #   one directory per source
    ├── pdftext@2.1.txt           #   <converter-id>@<version>.<ext>
    ├── sheet-Population@1.0.tsv  #   one rendition per extracted sheet/table
    └── ocr-tesseract@5.3.txt     #   (v2 — see Vision below)
```

The database's source record ([provenance.md](../Knowledge_Model/provenance.md#source-model))
holds the metadata: original filename, media type, locator, retrieval time, declared version,
optional publisher signature. The *identity* is the hash — upload the same file twice and it is
the same source, automatically.

## Renditions: the conversion layer

A **rendition** is a derived view of a snapshot produced by a named, versioned converter:
PDF → text layer, XLSX sheet → TSV table, HTML → cleaned text + tables. Rules:

1. Renditions are hashed like everything else and recorded as
   `(source id, converter id@version) → rendition hash`. Same converter version + same
   original ⇒ same rendition bytes, reproducibly — converters MUST be deterministic (a
   converter with nondeterministic output is rejected at review).
2. **Evidence offsets point into the rendition**, and the provenance entry names which one
   ([per-grain provenance](../Knowledge_Model/provenance.md#per-grain-provenance-entry) gains a
   `rendition` field alongside `extraction method`). The chain a human can walk:
   grain → excerpt at offsets in rendition R → produced by converter C@v from snapshot S →
   bytes on disk that re-hash to S. No link is a matter of trust.
3. Renditions are cache: deletable, rebuildable from `archive/` + the converter at the recorded
   version. Backups cover `archive/` + `kb.sqlite`; renditions and proof caches are explicitly
   excluded — same rebuildable/ground-truth split as everywhere else
   ([codebase-layout.md](codebase-layout.md#write-path--read-path-mapped-onto-modules)).
4. A converter upgrade (pdftext 2.1 → 2.2) creates *new* renditions; old ones referenced by
   existing grain provenance are kept (they are what the evidence offsets mean). This is the
   extractor-versioning discipline extended one layer down.

## Per-format plan

| Format | Converter → rendition | Tier ([parsing.md](../Research/parsing.md#source-difficulty-ladder)) | Trust status |
|---|---|---|---|
| CSV / TSV | none needed (already text) | 1 | deterministic |
| XLSX / ODS | sheet → TSV per sheet; formulas stored as *values + formula text* (the value is what the source asserted; the formula is context) | 1–2 | deterministic |
| JSON / XML / API dumps | pretty-printed normal form | 1 | deterministic |
| HTML | boilerplate-stripped text + extracted tables | 2 | deterministic |
| PDF (born-digital) | embedded text layer + per-page offsets | 2–3 | deterministic |
| PDF (scanned) / images | **OCR** — see Vision below | — | **probabilistic: v1, contained (quarantine-only)** |
| Audio / video | out of scope entirely; not on any roadmap | — | — |

An unrecognized format still archives fine (the snapshot layer is format-blind); it simply has
no converter, so it yields no candidates until one exists. Storage never blocks on
understanding.

## Vision / OCR: the honest KISS answer

**The use case is real but narrow:** scanned PDFs, faxed/photographed documents, images of
tables — sources where the knowledge is genuinely trapped in pixels. That is the *only* vision
use case in scope-of-philosophy: "describe this photo" produces impressions, not evidence, and
has no place here.

**Why OCR is not v1:** OCR is a probabilistic converter — it guesses characters (l/1/I, 0/O,
digits in tables — precisely where factual damage is maximal). The
[containment rule](../Research/parsing.md#where-probabilistic-methods-may-be-required--and-the-containment-rule)
therefore applies at the *converter* layer, one layer below where we first wrote it:

1. An OCR rendition is marked **untrusted**. Trusted (deterministic) renditions can feed
   auto-admission per normal policy; untrusted renditions can only feed **quarantine**.
2. A grain extracted from an OCR rendition reaches `verified` only when a human confirms the
   excerpt against the page image (the conformance UI shows both side by side), or an
   independent non-OCR source corroborates it. The OCR engine's per-character confidence
   numbers die at the rendition boundary — they never touch grain status
   ([ADR-0003](../ADR/ADR-0003-unknown-over-guessing.md)).
3. Provenance stays intact either way: the archived snapshot is the *image*; the excerpt
   offsets live in the OCR rendition; the human check is recorded like any verification.

**Scope decision (amended 2026-07-04, pre-freeze, owner decision):** OCR ships **in v1**, under
exactly the containment above. The build order stays socket-first: the rendition layer with its
trusted/untrusted flag (checklist P5.8) lands before the OCR converter (P5.11) and the
side-by-side confirmation flow in the curator CLI (P5.12). The containment terms are not
adjustable at implementation time — OCR-derived grains reach `verified` only through human
confirmation or independent corroboration, per
[RFC-0001](../RFC/RFC-0001-confidence-thresholded-answering.md); any loosening is a new RFC.

## Bulk intake

"A massive amount of spreadsheets" is the intended happy path, and the design absorbs it
without new machinery:

- **Idempotent by construction.** Sources are content-addressed, so re-ingesting a directory
  costs one hash per file; unchanged files short-circuit, changed files become new source
  versions. Point the intake at the same folder nightly and only novelty does work.
- **Grain-level dedup does the rest.** A thousand sheets repeating the same fact converge on
  one grain with a thousand provenance entries — corroboration accumulating for free
  ([grains.md](../Knowledge_Model/grains.md#what-a-grain-is)).
- **Throughput lives on the write path**, which is allowed to be slow-and-checked
  ([performance.md](../Performance/performance.md#workload-shape)): parallel conversion and
  extraction workers, one serialized admission point. The `grains/sec admitted` metric (P6.2)
  is the number to watch; nothing about bulk load touches the read path or determinism.
- One schema mapping per spreadsheet *format* (not per file): map "column B is POPULATION"
  once, apply it to every file that matches the format (P5.10).

## "Training data" — what it means here

Hourglass **never trains** — there are no weights, so nothing in the system is training data in
the ML sense ([glossary: terms deliberately not used](../Glossary/README.md#terms-deliberately-not-used)).
But every asset that role usually covers has a designated home already:

| What people mean by "training data" | Hourglass equivalent | Lives in |
|---|---|---|
| The raw corpus the system learns from | Source snapshots — the system doesn't learn from them, it *cites* them | `archive/` |
| Labeled examples defining correct behavior | Conformance fixtures — executable spec: input → exact expected bytes | `conformance/` in git |
| Evaluation/test sets | Extractor corpora: per-pattern documents with expected grains (P5.5) | `conformance/extractors/` |
| Fine-tuning / feedback data | Curator decisions — quarantine verdicts, conflict resolutions, all in the append-only status log | `kb.sqlite` |
| Model checkpoints | *(none — nothing to checkpoint)* | — |

If a v2 statistical proposer (tier-4 extraction, OCR post-correction) ever needs actual ML
training data, it is built and trained **outside this repository and outside the trusted path**,
and only its *proposals* enter — through quarantine, like everything else. The knowledge base
itself is never anyone's training set from the inside.

## Rejected alternatives

- **Storing uploads in the database as blobs** — bloats the hot store, breaks the
  cheap-backup/rehash-audit story; rejected in
  [provenance.md](../Knowledge_Model/provenance.md#future-work) and settled here.
- **Converting on upload and keeping only the conversion** — destroys ground truth; a
  conversion bug becomes permanent data loss. The original is sacred; conversions are cache.
- **One universal "document → text" converter** — formats differ in what must be preserved
  (table structure, page offsets, sheet names); per-format converters are each trivial, and
  trivial-per-format beats clever-universal (KISS).
- **General multimodal vision models** ("read this chart", "what does this diagram imply") —
  interpretation, not evidence extraction; philosophically out, not merely deferred.

## Future work

- Rendition schema addition to the provenance entry (fold into the Phase 1 provenance work,
  P1.12 — the field exists from day one so OCR needs no migration).
- Intake CLI UX: `hourglass ingest <file|url>` → source id + "what converters ran, what's in
  quarantine."
- Retention/tiering for `archive/` at scale (cold storage) — already tracked in
  [provenance future work](../Knowledge_Model/provenance.md#future-work).
