# Deck: the Rust metadata pipeline (lab meeting, 1 September 2026)

Source of truth: `slides.md` (deckkit format — see `../CLAUDE.md` and
`../deckkit/README.md`).

Build — **note this deck outputs `index.html`, not `slides.html`**, so the folder
URL resolves on GitHub Pages without a redirect stub. `bin/deck build` hard-codes `-o slides.html` and passing a second `-o`
crashes marp-cli, so call marp directly:

```bash
../deckkit/node_modules/.bin/marp --no-stdin --config ../deckkit/marp.config.mjs \
    slides.md -o index.html
```

`../deckkit/bin/deck png slides.md` still works unchanged for the layout check.
Rendering needs a Chrome — Firefox cannot be driven by puppeteer here, and
Ubuntu 24.04 blocks unprivileged user namespaces, so:

```bash
export CHROME_PATH=~/.cache/puppeteer/chrome/linux-152.0.7977.64/chrome-linux64/chrome
export CHROME_NO_SANDBOX=1
```

18 slides. No external assets — every visual is a deckkit component, so the
built `index.html` is self-contained and needs nothing copied alongside it.

Listed on the site index at the repo root, newest-first at the top of the list.

> **First draft, not workshopped yet.** Two placeholders on the title slide:
> the presenter is "Andrew" with no surname, and the date is today's. Fix both
> before this is shown to anyone.

## What the deck covers

The subject is `sra-metadata-enrichment/andrew/rust` — the Rust implementation
of the metadata reconstruction pipeline (workstream 2, literature metadata
extraction).

**It is a research harness, not a shippable tool**, and the deck has to say so
or every number in it gets read as a product spec. Slide 6 exists for that. The
consequence for editing: whatever is in the run configuration on any given day
is a scratchpad for the experiment in progress, so **never put the current state
of `main.rs` on a slide, and never present one configuration as the recommended
one**. Configurations appear only as *named points being compared*.

The arc is three moves:

1. **What it does** (slides 2–6) — thin SRA records, one denormalized record per
   experiment with 62 fields, the four-layer cascade, and what each layer
   actually delivers, and the harness that makes all of it configurable.
2. **How it knows a value is real** (slides 7–10) — the two self-report designs
   that failed, the computed `Grounding` that replaced them, and the 77%
   fabrication rate that turned out to be a matcher bug.
3. **Whether any of it can be measured** (slides 13–16) — no gold set, so
   reproducibility instead; the fixture warning; and what one configuration
   scores. Then slide 17, which is speculative and marked as such, and the
   takeaways.

Cost and the spend guards (11–12) sit between parts 1 and 2 because the cascade
ordering only makes sense once the per-sample billing is on a slide.

## Slide map

| # | Slide |
| --- | --- |
| 1 | Title |
| 2 | The problem |
| 3 | What the pipeline produces |
| 4 | Four layers, cheapest first |
| 5 | What each layer actually delivers |
| 6 | The pipeline is a variable, not a program |
| 7 | *Divider* — filling a field is easy, knowing it is real is not |
| 8 | Two things we asked the model, and why both failed |
| 9 | The fix: the model no longer classifies anything |
| 10 | The 77% fabrication rate that was not there |
| 11 | What it costs |
| 12 | Four spend guards, failing independently |
| 13 | *Divider* — is any of it right? |
| 14 | What can be measured with no answer key |
| 15 | The benchmark, and the warning printed at the top of it |
| 16 | What one configuration scores |
| 17 | Three things I want to try next |
| 18 | Takeaways |

## Where the numbers come from

Every figure traces to the repo at `~/Documents/repos/sra-metadata-enrichment/andrew/rust`.

| Slide | Figure | Source |
| --- | --- | --- |
| 2 | 91,229,082 runs; 5,733,257 human | repo root `README.md` — Logan v1.2 `librarysource`/`organism` columns |
| 3 | 346 studies, 102,240 records; 62 fields | `rust/README.md`; `cost_findings.md` §1 |
| 5 | 29.00 / 2.50 / 2.60 / 0.81; 1,638 vs 177 for $0.26; 55 calls, $0.2627 | `rust/README.md`, "A reference run" (`runs/20260818T044453Z.json`) |
| 6 | per-layer knobs; the `schema!` macro and the second schema; the offline examples | `src/main.rs` header; `rust/README.md`, "The record" and the module map |
| 8 | 391 / 41 / 0 over 432 inferences; zero vs constant `quoted`; 90 / 59 / 19% | `rust/README.md`, "The record" |
| 9 | the five `Grounding` variants | `src/evidence.rs`, `Grounding::classify` |
| 10 | 176 of 229; the `Cell linesNIH 3T3` span | `src/evidence.rs`, the `contains_span` comment |
| 11 | $284.63; $0.003134/sample; $0.02154/paper; $886k / $147k / $129k; the two counting traps | `cost_findings.md` §1, §2, §3 |
| 12 | the four guards | `rust/README.md`, "Spend guards"; `src/main.rs` |
| 14 | 1.8× / 1.2×; 56.6–70.7 / 88.8 / 60.1%; 256 vs 84 | `benchmarking_plans.md` §1 |
| 15 | the fixture warning | `examples/benchmark.rs` header; `benchmarking_plans.md` §3 |
| 16 | 540 cells / 312 archive / 228 stake; 58.1%; 16.2–35.5%; 56.9–82.7%; 64.6–72.8% | `benchmarking/runs/`, see below |
| 17 | **nothing — see below** | author's own brainstorming, no source |
| 18 | 9–16% cheaper for `D H P N` | `layer_order_findings.md` §5; `rust/CLAUDE.md` |

### Slide 16 is computed from the run files, not quoted from a doc

The seven replicates are `benchmarking/runs/20260901T0240{50,59}Z`,
`…T0245{38,48,59}Z` and `…T0246{08,19}Z` — all `D H P N`, Haiku 4.5, thinking
disabled, unbatched, `max_spend` 0.25. The free-layer baseline is
`…T025939Z.json` (`direct`, `harmonized`, no models). Ranges are min–max of the
`grid`/`stake` accuracy and `stake` precision fields parsed out of
`params.note`.

**Do not widen that window.** An earlier group the same morning
(`…T0205*`, `…T0222*`) scores far worse — stake accuracy 0.0–20.2% against
16.2–35.5% — and something changed between them. Mixing the two groups produces
a range that describes two different systems.

## Claims the deck is built to defuse

Check any edit against these:

- **Never say the answers are in the literature.** Slide 2's three cards exist
  because an earlier draft said the missing values are "in the paper behind the
  run, or in the reads themselves — neither is in the record", which contradicts
  the deck's own slides 4 and 5. Three of the four layers read the *record*;
  `Harmonized` (2.50 per record) and `LLMNaive` (2.60) together out-produce
  `LLMPaper` (0.81) better than three to one. Most of what looks missing is
  mis-keyed free text in the submitter's own attribute bags, not absent.
- **Deriving values from the reads is not this pipeline.** It is the separate
  sequence-derived workstream, and nothing in this crate touches sequence data.
  It was on slide 2 once and was removed; do not put it back.
- **Slide 17 is the only slide with no sources, and it has to stay visibly so.**
  Its three ideas — retrieval in place of `LLMPaper`'s fixed 30,000-character
  prefix, embeddings where `Harmonized`'s synonym table stops, and a fifth
  prototype layer that flags a submitted value as incorrect — are the author's
  brainstorming. There is no design, no ticket and no measurement behind any of
  them. The lead line says "none of this is built, measured, or written down
  anywhere" and that sentence is load-bearing: without it the slide reads as a
  roadmap. **Do not add dates, effort estimates or an ordering.**
  - The third one **contradicts a stated constraint** rather than extending the
    design. `assign` never overwrites a settled field and `Direct`'s values are
    anchors; "never touch a submitter's value" is why `Harmonized` can be
    trusted without a grounding. The sketch is a *flag*, not an overwrite, and
    the callout on the slide exists to make that distinction survive Q&A.
  - An earlier version of this slide was the five-step plan from
    `benchmarking_plans.md` "Next steps". It was cut: it was a roadmap nobody
    had asked the deck to present, and compressing five paragraphs into timeline
    nodes of six words each destroyed the meaning. If a next-steps slide is ever
    wanted again, use cards with room for the reasoning — not the `timeline`
    component.
- **Never present the free/paid value counts as a verdict.** "1,638 real values
  for nothing against 177 for $0.26" is true and was the note on slide 5 and the
  first takeaway; on its own it reads as "the paid layers are not worth running",
  which is the opposite of the case. `Direct` supplies 29 of the 34.91 values per
  record and every one describes the deposit or the instrument, not the sample.
  The 5.91 from the other three layers are the fields the project exists to fill.
  Slide 5's third column now names *where each layer's evidence lives* rather
  than just listing fields, which is what keeps the comparison honest. The raw
  counts survive in that slide's presenter notes, with the composition attached.
- **Never quote the whole-grid number as accuracy.** 58.1% → 64.6–72.8% is
  diluted by 312 of 540 cells that the free layers had already settled and no
  model was ever asked about. Slide 16's callout exists to say so; if the table
  ever loses the `stake` row, the slide is misleading rather than merely
  incomplete.
- **Grid precision is not the precision.** The runs report 91.9–95.9% on the
  whole grid, which is inflated by that same trivially-perfect archive
  partition. The deck quotes **56.9–82.7%**, the `stake` figure. Do not swap in
  the larger number because it looks better.
- **The benchmark key is a fixture, not a gold set** — slide 15 is the whole
  point, and `examples/benchmark.rs` prints the same warning at the top of its
  own source. A score from it is an adjudication queue getting longer, not an
  accuracy measurement. **No accuracy number from it goes in an abstract.**
- **The 77% on slide 10 is a bug, not a finding.** The slide title says "that was
  not there" for exactly that reason. Never let the number appear without its
  resolution on the same slide.
- **Five identical runs produced 1, 2, 10, 29, and 56 answers.** Any number on
  slide 16 that stops being a range is wrong.
- **Never show the current `main.rs` configuration.** It is a scratchpad for
  whatever experiment is running that week — the layer order in it has been
  `DHN` while the findings recommend `DHPN` — and putting it on a slide would
  present an arbitrary state as a recommendation. Configurations belong on
  slides only as named points in a comparison, always with their `n`.
- **`LLMPaper` does not "run last".** Slide 4's card said so in an early draft,
  which contradicted the `D H P N` used on slides 14 and 16 and the +0.76 / 9–16%
  claim in takeaway 1 of slide 18. In the measured-best order the paper layer is
  **third** and the per-sample text layer is last. The upstream `rust/README.md`
  narrates the layers as D → H → N → P and says "runs last" there, so the
  wrong claim is easy to re-import: it describes the declared listing order, not
  the order the findings recommend. The card now says "the most expensive
  evidence in the cascade", which is true under either ordering.
- **The `D H P N` notation is defined on slide 4 and nowhere else.** Each card
  carries its letter in the tag (`D · free, offline`), and the "How a run is
  named" callout underneath states the convention plus the ordering result.
  Slide 14 carries a one-line footnote reminder because the notation reappears
  eight slides later. If slide 4 is ever cut or reordered, the notation has to
  land somewhere before slide 14.
- **`CitedInstructions` goes last in slide 9's table, and is labelled a subset.**
  It sat third in an early draft, which put the rarest and least consequential
  variant in the middle of the reading order and implied it was a fifth
  independent outcome. In `classify` it is the *same branch* as `Unsupported` —
  span not in the evidence — with one extra test: the span is long enough, and it
  appears in our own prompt. It can only ever be a subset. The table now reads
  `Quoted` → `Grounded` → `Unsupported` → `Ungrounded`, then `CitedInstructions`
  underneath as the diagnostic split. Do not promote it back up the table.
- **`Grounded` is not "paraphrased".** Slide 9 says it asserts only that the
  citation is real — there is no synonym table, no fuzzy match, no entailment
  check. It was called `Rephrased`, and that name claimed a relationship nothing
  had established.

## Layout notes, learned by rendering

- Every content slide is `compact middle`. A first pass used `dense` and `micro`
  and every slide came out half empty — the density steps are for content that
  overflows, and this deck's slides do not.
- **`.footnote` is pinned bottom-left on the footer's own baseline.** A long one
  runs underneath the footer text. Keep them to roughly 85 characters (slide 5
  is at the limit); anything longer belongs in a `callout slim` or a `note`,
  which is where slides 10 and 15 ended up.
- **Timeline node text wraps inside one fifth of the lane.** Keep the label,
  value and note to a few words each or the third node drops out of the track
  box and into whatever is below it.

## Language

**American English, and the serial (Oxford) comma.** Both are deliberate, and both
differ from the rest of this repo — deckkit's own README and the other decks use
British spellings (`colours`, `normalisation`, `analyse`). Keep this deck
American and leave the neighbours alone.

Words that have already been corrected once, so they are the ones to watch on the
next edit: *denormalized* (not denormalised), *judgment* (not judgement). The
serial comma bites hardest in the run-count list on slide 16 and in the
five-part record description on slide 3.

## Tone

Same audience as the paper-linkage deck: working scientists who do not know this
codebase. Acronyms expanded on first use, no rhetorical framing, and every
number on a slide traceable to the table above. The argument is deliberately
self-critical — three of the seventeen slides are about a measurement that was
wrong — because that is what the project's own documents are like, and the deck
would misrepresent the work if it were confident where the source is not.
