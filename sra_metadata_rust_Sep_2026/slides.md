---
marp: true
theme: deckkit
size: 16:9
paginate: true
header: 'Lab meeting | Reconstructing SRA metadata'
footer: 'metadata-project · Rust implementation • 1 September 2026'
---

<!-- 17 slides. Every figure is measured — see CLAUDE.md for provenance. -->

<!-- _class: title -->

Lab meeting {.badge}

# Filling the blanks in SRA metadata

## Every inferred value carries the text it was read from

::: presenter
Andrew

1 September 2026
:::

<!--
Three parts: what the pipeline does, how it knows a filled value is real, and
whether any of it can be measured. The middle part is the one to stay awake for
— two designs failed there before the third worked.
-->

---

<!-- _class: compact middle -->

# The problem

The Sequence Read Archive stores raw sequencing reads with one submitter-typed metadata record per deposit. The records are thin.

```stats accent=sky
91,229,082 | sequencing runs | in SRA
5,733,257 | of them *Homo sapiens* | Logan v1.2, the authoritative count
```

::: cards cols=3 gap=18px border=top size=sm
### The fields are blank {accent=slate}

A few hundred characters per deposit. Tissue, disease, age, and treatment are left empty far more often than they are filled, and there is usually no link to the paper.

### The record is not empty {accent=sky}

Much of what is missing from the *fields* is present as free text — the submitter's own attribute bags, the sample titles, the study abstract. Mis-keyed rather than absent.

### The rest is in the paper {accent=emerald}

What no part of the record states, the manuscript behind the run often does.
:::

<!--
The middle card is the one to say out loud. Three of the four layers read the
record rather than the paper, and on the reference run they out-produce the
paper layer better than three to one — so "the answers are in the literature"
is the wrong mental model for what this pipeline does.

Deriving values from the reads themselves is a real fourth source, but it is a
separate workstream (sequence-derived) and nothing in this pipeline touches it.
Do not put it back on the slide.

The human count is from the librarysource/organism columns of the Logan v1.2
SRA table. An earlier draft said 6,904,165 — that is the sra_taxid assay-type
proxy, which counts differently. Use 5,733,257.
-->

---

<!-- _class: compact middle -->

# What the pipeline produces

One flat record per experiment: 62 fields, each carrying where its value came from.

::: cols ratio="1fr 1fr" gap=30px
## Input

A harvested corpus of archive records plus their open-access papers — **346 studies, 102,240 records**.

Nothing in the pipeline touches NCBI or Europe PMC. The harvest already happened; this starts from a file.

+++

## Output

One row per experiment, fully denormalized: study, sample, experiment, run, and submission values on the same record.

Every value is `Known`, `Missing`, or `Unknown` — where `Missing` means the submitter positively stated that the field does not apply.
:::

::: callout title="The part that matters" accent=purple
A filled field is only useful if somebody can check it. Every inferred value carries the span of text it was read out of, and the source that span came from.
:::

---

<!-- _class: compact middle -->

# Four layers, cheapest first

Each layer only fills fields no earlier layer settled. The ordering is a cost decision: run the free deterministic layers first and the paid ones pay only for what is left open.

::: cards cols=2 gap=18px size=sm border=left
### Direct {tag="D · free, offline" accent=emerald}

Reads the archive object graph. The only layer that *creates* records — everything it writes is something an archive stated outright, which is what makes these the anchors no later layer may overwrite.

### Harmonized {tag="H · free, offline" accent=emerald}

A synonym table over four submitter attribute bags, in priority order. The *value* is the submitter's and is trustworthy; only the key mapping is ours.

### LLMNaive {tag="N · one call per sample" accent=amber}

Study title and abstract, plus each sample's raw attribute bag. One call per study for study-level fields, one per distinct sample for everything else.

### LLMPaper {tag="P · one call per paper" accent=rose}

Up to 30,000 characters of open-access full text, Methods first. The most expensive evidence in the cascade, so it is only ever asked what nothing cheaper could answer.
:::

::: callout title="How a run is named" accent=slate slim
A cascade is written as its layer letters, in the order they run. `D H P N` is all four with the **paper layer ahead of the per-sample text layer** — which fills +0.76 more real values per record than `D H N P`, and costs 9–16% less.
:::

<!--
Nothing enforces "only fill what is open" as an invariant — it is a cost
decision, not a type-level guarantee. Layers are named rather than numbered
because the order is a property of the list, not of the enum.
-->

---

<!-- _class: compact middle -->

# What each layer actually delivers

Reference run: five studies, 52 records, all four layers, Sonnet 5 with thinking disabled, both model layers batched. 55 API calls, $0.2627.

| Layer | Per record | What it fills |
| --- | --- | --- |
| `Direct` | **29.00** | **What the archive registered** — accessions, titles, the library block, platform, spot and base counts |
| `Harmonized` | **2.50** | **What the submitter typed in a bag** — `strain` 85%, `cell_line` 42%, `tissue_type` 42% |
| `LLMNaive` | **2.60** | **What the record says outside its fields** — `cell_type` 48%, `treatment` 48%, `library_name` 48%, `age` 29% |
| `LLMPaper` | **0.81** | **What only the manuscript states** — `isolation_source` 44%, `age` 13%, `sequencing_method` 10% |

::: note accent=amber
`Direct` supplies 29 of the 34.91 values per record, and every one of them describes the deposit or the instrument rather than the sample. The **5.91** the other three layers add are the ones the project exists to produce. Running the free layers first is what keeps the bill at $0.26 — it is not evidence that the paid layers are optional.
:::

<!--
The raw counts, if anyone asks: 1,638 real values from the free layers and 177
from the paid ones, over the 52 records. Say them only with the composition
attached — on their own they invite the conclusion that the paid layers are not
worth running, which is the opposite of what the benchmark's stake partition
shows later in the deck.
-->

Mouse, *C. elegans*, and human only — these frequencies are this set's, not SRA's. {.footnote}

---

<!-- _class: compact middle -->

# The pipeline is a variable, not a program

This is a research harness, not a shippable tool. The unit of work is *two runs that differ in one thing*, so almost everything a run does is a knob rather than a decision baked into the code.

::: cards cols=2 gap=18px size=sm border=left
### What one run can change {accent=sky}

Which layers run, and in what order. Then, **independently per layer**: the model, the prompt, the reasoning effort, the thinking budget, the token ceiling, and whether calls go out batched or live.

### The target schema is one line {accent=purple}

Fields are declared through a macro that generates the struct, the field list, and every table derived from them. A colleague's ENA-named Solr core is a second schema, a different file, and the same pipeline.

### Every run saves what produced it {accent=emerald}

Each run writes its parameters and a provenance histogram beside its records. Two runs differing in one variable are only comparable if both were kept along with their settings.

### The analysis tools never spend {accent=amber}

Agreement across replicates, re-deriving groundings after the classifier changes, re-auditing stored evidence — all of it runs offline over saved runs, with no key and no network.
:::

::: callout title="There is no single 'right' configuration" accent=slate slim
Every number in this deck is a point in that space, measured against a stated setting — not the tuning of a finished product. The interesting output is the *difference* between two points, which is why the next few slides are all comparisons.
:::

<!--
Say this part out loud. The obvious question after the cascade slides is "so
what settings do you ship?", and the answer is that shipping settings is not
what this is for yet. Whatever is currently in the run configuration is a
scratchpad for the experiment in progress and should not be read as a
recommendation.
-->

---

<!-- _class: divider middle -->

# Filling a field is easy. Knowing whether the value is real is not.

Two designs failed at this before the third one worked.

---

<!-- _class: compact middle -->

# Two things we asked the model, and why both failed

Both attempts asked the model to describe its own work. Both produced a label that tracked the configuration rather than the evidence.

::: cols ratio="1fr 1fr" gap=28px
## Attempt 1 — ask for a confidence

`high` / `medium` / `low` on every inferred value, over 432 inferences:

```bars accent=rose unit="answers"
high | 391
medium | 41
low | 0
```

The largest error class came back uniformly `high`.

+++

## Attempt 2 — ask it to name what it did

`quoted` / `rephrased` / `inferred`:

```metrics accent=rose
One configuration, 400+ answers | zero `quoted`
Another configuration | constant `quoted`
Three runs, one identical setup | 90% / 59% / 19%
```

The label moved with the setup, not with the text.
:::

::: callout title="What both have in common" accent=amber slim
Both are self-reports, and a self-report about one's own process is not something a model can be held to.
:::

---

<!-- _class: compact middle -->

# The fix: the model no longer classifies anything

It returns one thing — `evidence`, the span it read the value out of. A function decides what that span supports, at assign time, against the exact string that call was shown.

| Grounding | What it means |
| --- | --- |
| `Quoted` | the span is in the evidence, and contains the value |
| `Grounded` | the span is in the evidence; the value is not in it |
| `Unsupported` | the span is nowhere in the evidence; the model invented it |
| `Ungrounded` | no span offered at all |
| `CitedInstructions` | **a diagnostic subset of `Unsupported`** — the span is not in the evidence, but it is in *our own prompt* |

::: cards cols=3 gap=16px size=sm border=top
### `Quoted` cannot exist unchecked {accent=emerald}

The classifier is its only constructor, so the variant and the evidence for it are created together.

### `Unsupported` values are stored, not refused {accent=amber}

A class refused at the door is a class nobody can count, and the rate is what the next round of this work needs.

### `CitedInstructions` diagnoses one bug {accent=rose}

A run produced six answers citing `organism Rattus norvegicus` — a string from a worked example in our prompt — for a study of human monocytes. Both variants mean ungrounded; the split says whether the prompt invites copying, which we can fix, or the model invents evidence, which we cannot.
:::

<!--
CitedInstructions is not a fifth outcome — it is the same branch as Unsupported
with one extra test, so it can only ever be a subset of it. As a bare
Unsupported the Rattus case took a manual trace through the run file to find.

The attribution is length-guarded on purpose: the worked examples hold realistic
values, so a short span collides with them by coincidence rather than by
copying.
-->

---

<!-- _class: compact middle -->

# The 77% fabrication rate that was not there

The first live run under the new design reported **176 unsupported answers out of 229**. That would have been the headline finding. Every one of the 176 was a false negative, and the model had invented nothing.

::: cols ratio="1fr 1fr" gap=28px
## The cause

The matcher padded its needle so it could only match on token boundaries. For a short *value* that is exactly right — without it, `USA` matches inside `USAGE`.

For a *span* it is wrong: the boundary has to exist in the text being searched, and extracted paper text routinely has none.

+++

## The span that exposed it

Copied perfectly from a methods section that reads:

```
...Cell linesNIH 3T3 cells were cultured in...
```

A heading fused to the sentence by whatever produced the text. The pad then demanded a word break before `NIH` that the paper does not contain.
:::

::: callout title="Why this is on a slide" accent=purple slim
A measurement this large, this wrong, and this plausible is exactly what the design exists to catch. It was found by looking at the answers, not by a test.
:::

---

<!-- _class: compact middle -->

# What it costs

Cost scales with **samples**, not with studies or papers: the per-sample layer is 97.5% of the bill.

```stats accent=sky cols=3
$284.63 | whole corpus | 346 studies, 102,240 records
$0.003134 | per sample | the number that scales
$0.02154 | per paper | one call per study
```

| All of SRA — ~1.08M studies, ~46M experiments | projected |
| --- | --- |
| every study looks like our corpus (295.5 records each) | ~$886,000 |
| SRA's real volume (42.8 records each), every study open-access | ~$147,000 |
| SRA's real volume, real ~17% open-access rate | **~$129,000** |

::: callout title="Two counting traps behind the denominator" accent=amber slim
`esearch db=sra` returns 46,059,823 — one per *experiment*, overstating studies by ~43×. ENA's `read_study` returns one row per *run*. Use `result=study`: ~1.08 million.
:::

<!--
The corpus is 6.9x heavier per study than SRA at large; the open-access filter
selected for substantial studies. That single assumption is the entire gap
between the $886k and $147k rows.
-->

---

<!-- _class: compact middle -->

# Four spend guards, failing independently

::: cards cols=2 gap=18px size=sm border=left
### `SPEND=1` {tag="1" accent=emerald}

The model layers are not constructed without it, so a bare `cargo run` is free **by construction** rather than by a flag that could be read the wrong way.

### The confirmation prompt {tag="2" accent=sky}

Prices the *actual plan* before anything is sent and requires a typed `y`. End-of-input declines, so a piped or unattended run cannot spend.

### The budget ledger {tag="3" accent=amber}

Meters what has actually been billed and refuses the next call at the ceiling. It cannot un-spend the call that crossed the line.

### Volume caps {tag="4" accent=slate}

Applied *before* the layers run, so a paid layer is never asked about a record that is going to be discarded anyway.
:::

::: callout title="Why 2 and 3 are not redundant" accent=purple slim
A ledger learns a call's cost only after paying for it, and it stops a run part-way — leaving a half-finished paid layer nobody can compare against anything. The estimate answers the question a ledger cannot: should this begin at all.
:::

---

<!-- _class: divider middle -->

# Is any of it right?

There is no gold set. But accuracy is not the only thing that can be measured.

---

<!-- _class: compact middle -->

# What can be measured with no answer key

Repeated runs of one configuration. Agreement proves nothing about correctness — but disagreement *proves* unreliability, and the modal fraction bounds accuracy from above with no labels at all.

| configuration | fill spread | ordinary-value ceiling |
| --- | --- | --- |
| `D H P N` — full cascade (n=4) | 1.8× | 56.6 – 70.7% |
| `D H N` — text layer isolated (n=3) | 1.2× | **88.8%** |
| `D H P` — paper layer isolated (n=3) | 1.2× | 60.1% |

`D` Direct · `H` Harmonized · `P` LLMPaper · `N` LLMNaive, in the order they run. {.footnote}

::: cards cols=2 gap=18px size=sm border=top
### Isolating a layer removes the compounding {accent=sky}

A layer is only asked about fields the layer before it left open, so upstream variance changes the downstream *denominator* rather than adding to it.

### Three quarters of the output is a declared absence {accent=amber}

`D H P` produced 256 absence cells against 84 ordinary values. A gold set that recorded only *values* would say nothing about most of what this emits.
:::

---

<!-- _class: compact middle -->

# The benchmark, and the warning printed at the top of it

A run is scored against an answer key for one deliberately degraded study. The key was written by a model reading the same record and the same paper, with a similar prompt.

::: callout title="The annotator is the system under test" icon="‼️" accent=rose
The errors are correlated exactly where it matters — whether `sex` is "not applicable" on a passaged cell line is the same judgment in both. Any systematic bias in the annotation becomes the definition of correct, and nothing in the scoring can tell a shared mistake from a shared insight.
:::

::: cards cols=2 gap=18px size=sm border=top
### A run scoring 100% is a red flag {accent=rose}

Not a success. The informative output is the disagreement list, which is an adjudication queue rather than a score.

### So it is a fixture, not a gold set {accent=emerald}

Not a demotion — it is the step that comes first anyway. To check that the grid works and the label classes round-trip, it does not need to be *right*, it needs to be *well-formed*.
:::

---

<!-- _class: compact middle -->

# What one configuration scores

One study, nine records, 540 cells. Seven replicates of a single point in that space — Haiku 4.5, `D H P N`, chosen because it is cheap to replicate rather than because it is recommended. The grid is fixed before the run, so the system cannot choose its own denominator.

| partition | what it is | free layers alone | with both model layers |
| --- | --- | --- | --- |
| `archive` | 312 cells the free layers already settled | 100% | 100% |
| `stake` | 228 cells only a model could reach | **0.9%** | **16.2 – 35.5%** |
| whole grid | all 540 | 58.1% | 64.6 – 72.8% |

::: callout title="Read the stake row, not the grid row" accent=amber slim
The whole-grid figure is diluted by 312 cells that were never a question anyone asked a model. Precision on the cells the models did settle is **56.9 – 82.7%** — that is the number with something at risk in it.
:::

::: note accent=slate
The unit of output is a **range**, not a number: five identical runs once produced 1, 2, 10, 29, and 56 answers, and two of three findings quoted from single runs did not survive replication.
:::

---

<!-- _class: compact middle -->

# Three things I want to try next

**None of this is built, measured, or written down anywhere.** It is where I would like to point the harness — and the third would change a rule the project currently runs on.

::: cards cols=3 gap=16px size=sm border=top
### Retrieval instead of a fixed prefix {tag="RAG" accent=sky}

`LLMPaper` sends up to 30,000 characters of the paper, Methods first, and hopes the answer is inside. Retrieval would pick passages against the fields actually open on that study — fewer tokens, and evidence chosen for the question rather than by position.

### Embeddings where the table stops {tag="Vectors" accent=purple}

`Harmonized` maps submitter keys through a hand-written synonym table: it matches or it does not. Similarity would catch the near-misses — more dynamic than a fixed table, and far cheaper per comparison than asking a model. It could grade `Grounded` too.

### Marking a submitted value wrong {tag="the big one" accent=rose}

Every layer today may only fill what is *empty*. A fifth, prototype layer would ask a model to flag an existing entry as **incorrect** — routine in metadata enhancement, and something this code cannot currently express at all.
:::

::: callout title="Why the third is not just another layer" accent=amber slim
`assign` never overwrites a settled field, and `Direct`'s values are anchors no later layer may touch — the cascade is *built* on that. Contesting a submitter's value needs somewhere to record a disagreement rather than replace a value, and a much higher bar of evidence than filling a blank.
:::

<!--
Brainstorming, not a roadmap. There is no design for any of these, no ticket and
no measurement, and the slide says so in its lead — do not let it be read as a
commitment or a schedule.

The third is the one to spend question time on. "Never touch a submitter's
value" is not an incidental rule: it is why `Harmonized` can be trusted without
grounding, and why `Direct` is described as an anchor. Expect the objection that
a model contesting human-entered data is exactly the thing this project has been
careful not to do — that objection is correct, which is why the sketch is a
*flag*, not an overwrite.
-->

---

<!-- _class: compact middle -->

# Takeaways

::: cards cols=1 gap=14px size=md border=left
### The cheap layers do the volume; the paid layers do the biology {accent=emerald}

`Direct` alone is 29 of the 34.91 values per record and all of it is registration metadata. Ordering the cascade cheapest-first is what holds the bill to $0.26 — the paid layers are still the ones filling the fields anybody came for.

### A label is an opinion; a span is a statement about a string {accent=purple}

The model no longer classifies its own work. It cites, and the citation is checked against exactly what it was shown — so `Quoted` cannot be claimed, only earned.

### The measurement is the binding constraint, not the pipeline {accent=amber}

Reproducibility caps any accuracy number a gold set could ever report. Fix that denominator before buying labels, and then buy far fewer than the obvious plan calls for.
:::
