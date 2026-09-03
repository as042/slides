---
marp: true
theme: deckkit
size: 16:9
paginate: true
header: 'Lab meeting | Target schema'
footer: 'sra-metadata-enrichment · schema consensus • 3 September 2026'
---

<!-- 16 slides. Every figure is measured — see CLAUDE.md for provenance. -->

<!-- _class: title -->

Decision meeting {.badge}

# What should a record actually hold?

## Three schemas exist, they disagree, and nothing downstream can be compared until they stop

::: presenter
Andrew

3 September 2026
:::

<!--
This is a decision meeting, not a status update. Two questions have to come out
of the room settled: what unit a record describes, and which biological fields
it carries. Everything else on the agenda is evidence for one of those two.

Say at the top that no configuration is being defended here. The schema is not
the pipeline.
-->

---

<!-- _class: compact middle -->

# The problem

Three schemas are in use across this project. Each was built for a different job, none was built to agree with the others, and every downstream number is quoted against one of them.

```stats accent=sky
3 | schemas in use | none a superset of another
2 | different record units | run-level and experiment-level
9 | benchmark columns | that our schema cannot express
```

::: cards cols=3 gap=18px border=top size=sm
### They were never reconciled {accent=slate}

Each schema is right for the job it was written for. They diverge on field *names*, on which biology they carry, and on what one row means.

### The costs are silent {accent=amber}

A field-name comparison across two of them compares different things and raises no error. A score against the third measures a schema that cannot hold the answer.

### The fix is cheap now {accent=emerald}

Adding a field is one line. Changing the record unit after a corpus run is not. This is the cheapest this decision will ever be.
:::

<!--
Do not let this turn into "which schema wins". None of them wins — two of the
three are not even trying to answer the same question. The output of the meeting
is a fourth thing that borrows from all three.
-->

---

<!-- _class: compact middle -->

# Three schemas, three answers

| | **`schema/sra.rs`** | **`schema/solr.rs`** | **`mela500`** |
| --- | --- | --- | --- |
| Whose | mine | colleague's | Metappuccino's |
| Fields | 62 + id | 60 + id | 15 + accession |
| **One row is** | **an experiment** | **a run** | **a run** |
| Names follow | SRA / NCBI | ENA portal columns | ad-hoc |
| Filled from | nested XML + papers | ENA flat export | hand-annotated |
| Provenance | per field | per field | none |
| Scale reached | 346 studies, 102,240 records | 40,575,282 documents | 500 rows |
| Sample-level fields | 32 | 28 | 15 |

::: note accent=amber
The two Rust schemas **invert two names**. In `solr.rs`, `study_accession` is the BioProject and `sample_accession` the BioSample; in `sra.rs` those are `bioproject_accession` and `biosample_accession`, and the plain names hold the SRA-internal SRP and SRS. A positional comparison across the two silently produces wrong-but-plausible values.
:::

<!--
The inversion is not a bug in either file — ENA gives the plain name to the
primary accession and `secondary_` to the SRA one. Both name the same object.
It is only dangerous when someone joins on field name, which is why every export
records its SchemaId and refuses to load a file from the other schema.

If asked why mela500 has no provenance: it is a gold set, not an output. The
absence is the point of comparison on the next slide, not a defect.
-->

---

<!-- _class: compact middle -->

# What each one gets right, and what it pays for

::: cards cols=3 gap=16px size=sm border=left
### `sra.rs` — faithful to the archive {tag="Experiment-level" accent=sky}

**Right:** carries the attribute bags, which are where the unharmonized biology lives. 32 sample-level fields. Every value carries how it was arrived at.

**Pays:** four fields have no source anywhere in the corpus. Cannot be joined to a run-level index without a lookup. One row per experiment collapses 21% of runs.

### `solr.rs` — speaks the group's language {tag="Run-level" accent=purple}

**Right:** exact set match with the TACC Solr core, so 40.5M documents can feed it directly. Run accessions make it joinable to anything.

**Pays:** built from ENA's flat 195-column export, which **drops the attribute bags entirely** — so as a *source* it can only ever fill what was already a column. Uses 60 of 195 available columns.

### `mela500` — the only one with answers {tag="Run-level" accent=emerald}

**Right:** actual labels, hand-written, on the fields clinical users ask for — disease, organ, treatment, response.

**Pays:** 15 fields, no provenance, and `unknown` conflates *not in the record* with *not knowable*. It is a scoring target, not a schema.
:::

::: callout title="The through-line" accent=slate slim
Each schema is a projection of a different source, and each source is lossy in its own direction. A consensus schema has to name which source it trusts per field — not average the three.
:::

---

<!-- _class: divider middle -->

# Two decisions, and they are independent

What one row means, and which fields it holds. Settling either does not settle the other.

---

<!-- _class: compact middle -->

# Decision 1 — is a record a run, or an experiment?

`sra.rs` emits one row per experiment. The Solr core and the benchmark both emit one row per run. Measured over the whole corpus:

```stats accent=purple
117,915 | runs | in 346 studies
102,240 | experiments | what the Rust pipeline emits
1.153 | runs per experiment | mean
```

```bars accent=purple max=102240 unit="experiments"
1 run | 93,010 | holds 93,010 runs — 78.9% of every run in the corpus
2 runs | 6,703 | holds 13,406 runs | accent=amber
3 runs | 1,952 | holds 5,856 runs | accent=amber
4+ runs | 575 | holds 5,643 runs, up to 30 in one experiment | accent=amber
```

::: note accent=amber
**Both framings are true, and they point opposite ways.** 91% of *experiments* hold exactly one run, so experiment-level rows are lossless for almost every experiment. But **21% of *runs*** sit inside a multi-run experiment, so one run in five loses its own row. Which number is the right one depends on whether the unit of analysis is a run.
:::

<!--
This is the slide the meeting turns on. Do not present 91% on its own — it is
the number that makes collapsing look free, and it is the experiment-side view
of the same fact that 21% of runs get merged.

The 15,675-row difference between 117,915 and 102,240 is what disappears.
-->

---

<!-- _class: dense -->

# What collapsing runs actually costs

Not biology. Every biological field in the schema is **sample-level**, and a sample never splits across runs.

::: cols ratio="1fr 1fr" gap=28px
## What is genuinely run-level

Of 62 fields in `sra.rs`, **four** sit at run level, and two of those have no source in this corpus at all:

```metrics accent=sky
total_spots | summed across runs
total_bases | summed across runs
submitted_format | no source
submitted_read_type | no source
```

So the live loss is **two additive counts** — recoverable by summing, unrecoverable per run.

+++

## What actually hurts

The run accession itself. `sra.rs` has no `run_accession` field; `solr.rs` has both `run_accession` and `run_alias`.

::: note accent=rose
Without it, an experiment-level record **cannot be joined** to the 40.5M-document Solr core or scored against `mela500` — both keyed by run — without a separate lookup.
:::
:::

::: callout title="The cheap resolution" accent=emerald slim
Keep experiment-level rows and **add `run_accession` as a list**. The unit stays where the biology is, the join key survives, and per-run counts remain the only real casualty. Worth ten minutes of argument before the field list.
:::

<!--
mela500 cannot settle this for us: its 500 runs map to 500 distinct BioSamples
and 469 BioProjects, so it is one-run-per-sample by construction and never
exercises the multi-run case. Say so if someone offers it as evidence.
-->

---

<!-- _class: divider middle -->

# Decision 2 — which biological fields

The archive says one thing about what is worth recording. The benchmark says another.

---

<!-- _class: compact middle -->

# The benchmark asks for nine fields we cannot write down

`mela500` is what this project is scored against. Its answer key has 15 data columns. `sra.rs` can express six of them.

::: cols ratio="0.9fr 1.1fr" gap=26px
## In the schema

`library_selection` · `cell_line` · `cell_type` · `treatment` · `age` · `sex`

## Missing

`disease` · `organ` · `biopsy_site` · `biopsy_type` · `is_cancer` · `treatment_time` · `response` · `ethnicity` · `sequencing_source`
:::

::: note accent=rose
A schema that cannot represent its own benchmark's answer key **cannot be scored against it.** Whatever else is decided today, this is the part with no defensible status quo.
:::

::: callout title="One of the nine is not a field at all" accent=amber slim
`sequencing_source` looked like `library_source` renamed. It is not — it is the bulk-vs-single-cell axis. Cross-tabulated over all 500 runs, `library_source` recovers **27 of the 60** single-cell runs, with zero false positives: a perfect positive signal and a useless negative one.
:::

---

<!-- _class: compact middle -->

# How the gap was measured

Not from a wish list. Every attribute bag in the corpus was read and counted, then resolved against the schema.

::: cards cols=2 gap=18px size=sm border=top
### What was counted {accent=sky}

88,502 sample bags, 88,560 BioSample bags, 102,240 experiment bags. Keys normalised with the pipeline's own `normalize_key`, then resolved against 63 declarations plus 29 `SYNONYMS` rows.

**46 keys resolve. 505 do not**, across 474,148 instances.

### Ranked by study spread, not count {accent=emerald}

One study can put a key on 20,000 samples. Candidates are ranked by **how many of the 346 studies use the key with a real value.**

Absence terms are excluded — they occupy the same slot as a value, and counting them inflates every figure.
:::

::: callout title="Why the absence filter changes the answer" accent=amber slim
`lat_lon` is only **26% real** — 31,514 of its 42,700 instances are *"missing"*. `ecotype` is 29%, `breed` 27%. A raw count ranks all three far too high.
:::

<!--
The ENA cross-reference matters for the next slide: every candidate was checked
against ENA's live 195-column catalogue, so "already an ENA column" means adding
it is alignment with INSDC and with the Solr core, not our own invention.
-->

---

<!-- _class: compact middle -->

# Free first — five synonym rows, no schema change

These are not missing fields. They are missing rows in `SYNONYMS`: keys that already have a home and are currently being sent to a paid model layer to guess at.

```bars accent=emerald max=346 unit="studies"
gender → sex | 10 | 14,199 values, 100% real, all male/female
host_age → age | 7 | 10,456 real values
host_tissue_sampled → tissue_type | 3 | 9,348 values, 100% real
organism → scientific_name | 2 | 94 values
host_subject_id → host | 2 | arguable — identifies an individual, not a species
```

::: note accent=emerald
**22 of 346 studies** gain a value from this, at a cost of one line each. Settle it outside the meeting regardless of everything below.
:::

---

<!-- _class: compact middle -->

# Tier 1 — the six worth adding

Each is used with a real value by 24+ studies, each carries a meaning no current field holds, and five of the six are already ENA columns.

```bars accent=sky max=346 unit="studies"
genotype | 75 | ENA: host_genotype · 100% real
isolate | 47 | ENA: isolate · the individual, a join key
lat_lon | 39 | ENA: lat, lon · MIxS-mandatory · only 26% real
disease | 29 | ENA: disease · **also a benchmark column**
timepoint | 26 | no ENA column · **benchmark's treatment_time**
cultivar / ecotype | 24 | ENA: cultivar, ecotype, variety · weakest of the six
```

::: note accent=sky
Taken together these put a real value into **184 of 346 studies**. They are not independent decisions — five of six can be supplied by the Solr core, and two are columns the benchmark already requires.
:::

<!--
`timepoint` needs explicit synonym rows: `timepoint` and `time point` do NOT
collapse to one key under normalize_key, so a field alone would miss half its
own data. Mention only if the field list gets adopted.
-->

---

<!-- _class: dense middle -->

# Tier 2 — the ones worth the meeting

Each has a real constituency and a real objection. The pattern to notice: the benchmark wants them, the archive barely carries them.

| Candidate | Studies | For | Against |
| --- | --- | --- | --- |
| `chip_target` | 20 | three ENA columns; a ChIP record loses its point without it | meaningless on ~90% of records |
| `replicate` | 15 | design structure nothing else records | no ENA column; values are prose |
| `culture_collection` | 11 | structured and verifiable — `ATCC:BAA-894`, 97% real | microbial studies only |
| `serotype` | 10 | 99% real, semi-controlled | pathogen surveillance only |
| `host_common_name` | 9 | 100% real, heavily used where used | **derivable from `host_tax_id`** — a lookup, not a field |
| `cell_subtype` | 8 | single-cell work needs a finer axis than `cell_type` | boundary with `cell_type` is a judgement call |
| `body_site` / `organ` | 6 | **two benchmark columns want it** | overlaps `tissue_type` *and* `isolation_source` |
| `ethnicity` | 4 | **benchmark column; blocks the annotation pass** | almost absent from the archive; governance questions |
| `single_cell` | 6 | **benchmark column**; free and exact where the archive states it | a derived boolean, unlike everything else here |

---

<!-- _class: dense -->

# What adding a field costs

Adding a field is one line in a macro, which makes it look free. It is not: every non-blind field joins the menu both model layers are asked to fill, on every call.

::: cols ratio="1fr 1fr" gap=28px
## The precedent

`sequencing_method` collided with two fields the archive already fills better — `instrument_model` and `library_strategy`.

```metrics accent=rose
Submitter values in the corpus | 4,704, all instruments
Model-written values across saved runs | 457
Population shared with submitters' | none
```

It became the largest single error source left in the paper layer, and is now blind to both model layers.

+++

## The one to leave alone

`source_name` — **220 of 346 studies**, 42% of samples, the most-used unmapped key by a wide margin.

::: note accent=slate
Values include *Fibroblast* (a cell type), *Hypothalamus* (a tissue), *whole worms* (an organism). Pinning it to one field would be wrong roughly two thirds of the time.
:::

It stays out of the schema — and is the highest-value *input* to a layer that can read it.
:::

::: callout title="The rule this implies" accent=amber slim
A new field that overlaps an existing one will fail the way `sequencing_method` did. That is the real argument against `body_site` and `cell_subtype`, and it is not about how often submitters use them.
:::

---

<!-- _class: compact middle -->

# What I would put to the room

::: cards cols=2 gap=18px size=sm border=left checks
### Settle now, outside the meeting {accent=emerald}

- The five synonym rows — no schema change, no argument available
- Whether `run_accession` joins the record as a list

### Adopt as one block {accent=sky}

- Tier 1's six fields: `genotype`, `isolate`, `lat_lon`, `disease`, `timepoint`, `cultivar`
- 184 of 346 studies gain a value; five of six come from ENA

### Decide in the room {accent=amber}

- **The record unit.** 91% of experiments vs 21% of runs — pick which one the project counts in
- The four Tier 2 fields where the benchmark and the archive disagree

### Do not decide today {accent=slate}

- Which of the nine benchmark gaps are *fields* versus *values derived from fields we have*
- `single_cell` is the test case for that question
:::

::: callout title="The question underneath all of it" accent=navy slim
Every field here is a claim about what a record is *for*. The archive, the Solr core and the benchmark each answer that differently — so name the source we trust per field, rather than trying to satisfy all three.
:::

<!--
If time runs short, the two that must not be skipped are the record unit and the
nine-column benchmark gap. Everything else can be settled asynchronously on the
evidence in the artifact.
-->

---

<!-- _class: divider middle -->

# The evidence behind every figure here

`andrew/rust/insdc_structure_findings.md` · the field-candidate artifact · measured against `oa_corpus_full.json`, 3 September 2026
