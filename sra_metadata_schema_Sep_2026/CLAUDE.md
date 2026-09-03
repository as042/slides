# Deck: agreeing on a target schema (decision meeting, 3 September 2026)

Source of truth: `slides.md` (deckkit format — see `../CLAUDE.md` and
`../deckkit/README.md`).

Build — like `sra_metadata_rust_Sep_2026`, this deck should output `index.html`
rather than `slides.html`, so the folder URL resolves on GitHub Pages without a
redirect stub. `bin/deck build` hard-codes `-o slides.html` and passing a second
`-o` crashes marp-cli, so call marp directly:

```bash
../deckkit/node_modules/.bin/marp --no-stdin --config ../deckkit/marp.config.mjs \
    slides.md -o index.html
```

`../deckkit/bin/deck png slides.md` renders the layout check. Rendering needs a
Chrome; `/usr/bin/google-chrome` is present on this machine:

```bash
export CHROME_PATH=/usr/bin/google-chrome
export CHROME_NO_SANDBOX=1
```

> **Node lives in a conda env on this machine** — there is no system node, so
> put it on PATH before any build:
> `export PATH=/home/andrew/miniconda3/envs/_galaxy_/bin:$PATH`
> (`deckkit/npm install` has been run against it.)

**Built and rendered 3 September 2026; all 16 slides inspected.** Density was set
from the renders, not guessed:

| Slide | Class | Why |
| --- | --- | --- |
| 14 — what adding a field costs | `dense` | at `compact` the closing callout **collided with the footer** |
| 13 — Tier 2 (9-row table) | `dense middle` | `micro` fitted but wasted ~200px; `dense` fills it |
| 3, 4 — schema comparison | `compact middle` | `dense`+`size=xs` left ~250px dead at the bottom |

Re-render and look at **14** first after any edit to it — it is the one slide
that has already overflowed once.

> **Placeholder:** the title slide's presenter is "Andrew" with no surname, and
> the date is the day it was drafted. Same placeholder as the sibling deck.

16 slides. No external assets — every visual is a deckkit component, so the
built `index.html` is self-contained.

Listed on the site index at the repo root, newest-first at the top.

## What the deck is for

It is a **decision meeting**, not a status update. Two things have to come out of
the room settled:

1. **The record unit** — one row per run, or per experiment.
2. **Which biological fields** the target schema carries.

Everything else on the deck is evidence for one of those two. The companion
long-form document is the field-candidate artifact, which carries the full tier
tables and the per-field reasoning; this deck is the subset that needs a room.

**Do not let it become "which schema wins".** Two of the three schemas are not
answering the same question, and the intended output is a fourth thing that
borrows from all three. Slide 4's callout is where that is said.

## Slide map

| # | Slide |
| --- | --- |
| 1 | Title |
| 2 | The problem — three schemas, never reconciled |
| 3 | Three schemas, three answers (comparison table) |
| 4 | What each one gets right, and what it pays for |
| 5 | *Divider* — two decisions, and they are independent |
| 6 | Decision 1 — is a record a run, or an experiment? |
| 7 | What collapsing runs actually costs |
| 8 | *Divider* — decision 2, which biological fields |
| 9 | The benchmark asks for nine fields we cannot write down |
| 10 | How the gap was measured |
| 11 | Free first — five synonym rows |
| 12 | Tier 1 — the six worth adding |
| 13 | Tier 2 — the ones worth the meeting |
| 14 | What adding a field costs |
| 15 | What I would put to the room |
| 16 | *Divider* — where the evidence lives |

## Provenance of every figure

All measured on 2026-09-02 / 09-03 against
`sra-metadata-enrichment/andrew/rust/oa_corpus_full.json` (`format_version` 2 —
346 studies, 102,240 experiments, 88,560 samples) unless noted.

**Slide 2–4, schema comparison.** Field counts are declarations in each
`schema!` invocation, counted by level: `sra.rs` = id + 62 fields (Study 6,
Submission 7, Sample 32, Experiment 12, Run 4, Record 1); `solr.rs` = id + 60
(Sample 28, Run 6). `mela500` = `run_accession` + 15 data columns, from
`benchmark/datasets/metappuccino-mela500/gold_500.tsv`. Solr core size
(40,575,282 documents) and the 195-column ENA figure are from
`andrew/rust/insdc_structure_findings.md`. **The name inversion between the two
Rust schemas is real and documented** in `andrew/rust/CLAUDE.md` — do not "fix"
either file to agree with the other.

**Slide 6, runs per experiment.** Counted by parsing every `"runs": [...]` array
in the corpus: 117,915 runs across 102,240 experiments, mean 1.153. Distribution
— 1 run 93,010 experiments (90.97%), 2 runs 6,703, 3 runs 1,952, 4+ runs 575,
max 30. Runs held by multi-run experiments: 24,905 of 117,915 = **21.12%**.

> ‼️ **Both framings are true and they point opposite ways.** 91% is the
> experiment-side view, 21% the run-side view of the same fact. Never present
> the 91% alone — it makes collapsing look free.

**Slide 7, run-level fields.** From `schema/sra.rs`: four fields at `Run` level,
of which `submitted_format` and `submitted_read_type` are both marked "No source
in this corpus". `solr.rs` additionally carries `run_accession` and `run_alias`.
mela500 cannot settle the unit question — its 500 runs map to 500 distinct
BioSamples and 469 BioProjects, so it never exercises the multi-run case.

**Slide 9, the benchmark gap.** `gold_500.tsv` header, cross-referenced by hand
against `sra.rs`. Six of 15 data columns are in the schema. The
`sequencing_source` cross-tab — `library_source` recovers 27 of 60 single-cell
runs, 0 false positives — is over all 500 runs joined to
`sra_records_500.json.zst`.

**Slide 10, method.** 88,502 sample bags + 88,560 BioSample bags + 102,240
experiment bags. Keys normalised with a faithful port of
`harmonized::normalize_key`, resolved against 63 declarations + 29 `SYNONYMS`
rows: 46 resolve, 505 do not, 474,148 instances. Absence terms excluded
throughout — `lat_lon` is 11,186 real against 31,514 absence/blank (26% real),
`ecotype` 29%, `breed` 27%.

**Slides 11–13, the tiers.** Study spread is a set union of study accessions
where the key appears **with a real value**, over the 346 studies. Tier 0 union
22/346; Tier 1 union 184/346. Per-candidate figures are in the artifact.

**Slide 14, the cost.** `sequencing_method`: 4,704 submitter occurrences in the
corpus, all instrument names; 457 model-written values across saved runs. Both
from the field's comment block in `schema/sra.rs`. `source_name`: 220/346
studies, 37,532 instances, 42% of samples; the polysemy examples are quoted from
`harmonized.rs`.

## Editing constraints

- **Named accents only**, never hex. Content-only slides — no `<style scoped>`,
  no inline `style=`, no ad-hoc `<div>`.
- **Never put the current state of `main.rs` on a slide.** Same rule as the
  sibling deck: the run configuration is a scratchpad for whatever experiment is
  in progress, and this deck is about the schema, not a configuration.
- The corpus is **346 open-access-paper studies**, not a random sample of SRA.
  Any fill-rate figure quoted as archive-wide should come from the Solr index
  instead (`collection_date` 36%, `host` 36%, `strain` 14%).
- Study spread measures **use, not correctness**. A submitter writing
  `cultivar = McIntosh` says the key is live, not that the value is right.
