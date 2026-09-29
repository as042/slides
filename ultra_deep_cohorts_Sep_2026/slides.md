---
marp: true
theme: deckkit
size: 16:9
paginate: true
header: 'Ultra-Deep Cohorts for Planetary Scale Evolution'
footer: 'Logan · Disassembler · HyphAeon • September 2026'
---

<!-- 13 slides. Converted 1:1 from the source .pptx draft — see CLAUDE.md. -->

<!-- _class: title -->

# Ultra-Deep Cohorts for Planetary Scale Evolution

## Andrew Seacord¹, Rayan Chikhi³, Danielle Callan², Steven Weaver², Patrik Smeds¹, Sergei L. Kosakovsky Pond², Anton Nekrutenko¹

::: presenter
¹The Pennsylvania State University · ²Temple University · ³Institut Pasteur
:::

::: cols ratio="1fr 1fr 1fr" gap=40px
::: figure src="assets/pennstate.png" h=90px bare
:::
+++
::: figure src="assets/temple.png" h=90px bare
:::
+++
::: figure src="assets/pasteur.png" h=90px bare
:::
:::

---

<!-- _class: compact middle -->

# Goal / Problem

**Goal:** Planetary-scale genomic analysis, especially for selection analysis of rapidly-evolving pathogens.

**Problem:** 2 bottlenecks

::: cards cols=2 gap=18px border=left size=sm
### SRA metadata {tag="Bottleneck 1" accent=amber}

The Sequence Read Archive has incomplete and often inaccurate metadata. Discovery of specific sequences is extremely challenging.

### Selection analysis {tag="Bottleneck 2" accent=rose}

Evolutionary selection analysis is slow. Contemporary software relies on asymptotically-complex algorithms that do not scale.
:::

---

<!-- _class: compact middle -->

# Sequence Discovery — Logan

Assemblers: Reads → Unitigs → Contigs → Sequence

::: cards cols=3 gap=18px border=top size=sm
### Logan {accent=sky}

Stores unitigs and contigs for every SRA accession.

### Kmindex {accent=emerald}

Unitigs can be searched via Bloom filters.

### LexicMap {accent=purple}

Contigs can be searched via alignment.
:::

---

<!-- _class: compact middle -->

# Large-Scale Selection Analysis — HyphAeon

::: cols ratio="1.35fr 0.65fr" gap=30px
HyphAeon is a small neural network that can recognize patterns in codons thousands of times faster than traditional tree-based methods.

- HyphAeon does selection analysis (meme, epistasis, etc.)
- The partner module, **ChronAeon**, does molecular clock dating
- Neural networks bypass the bottleneck of asymptotically-complex tree searches

+++

::: figure src="assets/hyphaeon-logo.png" h=340px bare
:::
:::

---

<!-- _class: compact middle -->

# Bridging the Gap — Disassembler

How do we feed Logan into HyphAeon?

::: cols ratio="1fr 1fr" gap=28px
- Logan offers unitigs and contigs
- HyphAeon requires MSA
- The **Disassembler** package: a Rust/Python tool suite for turning Logan output into a HyphAeon-ready MSA

Two main pathways:

::: cards cols=1 gap=10px size=xs border=left
### LexicMap pathway {accent=purple}

quick, broad analysis

### Kmindex pathway {accent=emerald}

slow, meticulous analysis
:::

+++

::: figure src="assets/logan-disassembler.png" h=340px bare
:::
:::

---

<!-- _class: compact middle -->

# The Kmindex Pathway

Kmindex is a tool for calculating the percentage of shared k-mers between a query and reference.

::: cards cols=1 gap=12px size=sm border=left
### Query with a protein CDS {accent=emerald}

Disassembler uses a protein coding sequence as a query and runs Kmindex to compare it against Logan's pre-computed Bloom filters.

### Result {accent=sky}

List of SRA accessions and their k-mer % identities to the query.

### Downstream {accent=purple}

Matching accessions → stream unitigs from AWS S3 → **Logan Walker** → MSA.
:::

---

<!-- _class: compact middle -->

# Logan Walker

::: figure src="assets/logan-walker-pipeline.png" h=280px bare
:::

- Unitigs are parsed into a **Compressed Sparse Row (CSR)** graph
- "Anchor" regions are identified by additional k-mer matching of query
- Subgraphs are made by expanding out from anchors and then are aligned by **GraphAligner**
- A legitimate path through the alignment is stitched together to make a haplotype

---

<!-- _class: compact middle -->

# The LexicMap Pathway

::: figure src="assets/lexicmap-pipeline.png" h=280px bare
:::

- LexicMap goes straight for the alignments
- **contigs** vs. **unitigs**

---

<!-- _class: compact middle -->

# Disassembler is not an Assembler

::: cards cols=2 gap=18px border=left size=sm
### Assembly is about compressing {accent=slate}

Reads → Unitigs → Contigs. Logan has done this for us already.

### Disassembler goes backwards {accent=sky}

Takes these pre-assembled intermediates and tries to find the information that gets hidden in the usual compression steps.

### Assemblers do not have a general target {accent=amber}

### Disassembler constructs everything based on the query sequence {accent=purple}
:::

::: callout title="Result" accent=emerald slim
Pre-computed, searchable Logan sequences → realistic haplotypes per accession that can be aligned and analyzed via HyphAeon.
:::

---

<!-- _class: compact middle -->

# SARS-CoV-2 Spike Protein Case Study

Kmindex pathway was tested via 11-strain panel of Covid spike CDS. Goal was to develop intuition regarding the kmer identity percentage.

::: figure src="assets/kmer-identity-panel.png" h=300px bare
:::

::: figure src="assets/reconstructed-coverage.png" h=180px bare
:::

---

<!-- _class: compact middle -->

# SARS-CoV-2 LexicMap RBD Case Study

::: figure src="assets/rbd-disposition.png" h=520px bare
:::

---

<!-- _class: compact middle -->

# SARS-CoV-2 LexicMap RBD Case Study — Map

::: figure src="assets/rbd-msa-map.png" h=560px bare
:::

---

<!-- _class: compact middle -->

# What is Next?

::: cards cols=2 gap=18px border=left size=sm
### Galaxy tools/workflows {accent=sky}

### HyphAeon and ChronAeon analysis {accent=purple}

### Intra-accession variance analysis {accent=emerald}

### Case studies for fast-diverging genomes {accent=amber}

e.g. HIV
:::

::: cols ratio="1fr 1fr 1fr" gap=40px
::: figure src="assets/pennstate.png" h=70px bare
:::
+++
::: figure src="assets/temple.png" h=70px bare
:::
+++
::: figure src="assets/pasteur.png" h=70px bare
:::
:::
