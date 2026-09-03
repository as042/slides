# slides

Presentation decks and the system that builds them.

Each deck is a directory containing a `slides.md` written in plain content
markdown — no CSS, no layout HTML. `deckkit/` turns it into a polished HTML
deck: a Marp theme plus a markdown-it plugin providing the component
vocabulary (cards, columns, callouts, timelines, figures, gene maps).

## Layout

```
deckkit/                        the build system
  themes/deckkit.css            the design system — all visual decisions live here
  lib/                          markdown-it plugin: ::: blocks, ```data fences, {{inline}}
  templates/deck.md             starter deck
  README.md                     component reference — start here to author a deck
  bin/deck                      CLI

<deck-name>/                    one directory per deck
  slides.md                     the deck source
  CLAUDE.md                     deck-specific notes and constraints
  ...assets the deck references
```

## Build

```bash
deckkit/bin/deck build <deck>/slides.md    # -> slides.html
deckkit/bin/deck png   <deck>/slides.md    # one PNG per slide, for checking layout
deckkit/bin/deck watch <deck>/slides.md    # live-reloading preview
deckkit/bin/deck pdf   <deck>/slides.md
deckkit/bin/deck new   <deck>              # scaffold from the template
```

First run needs dependencies: `cd deckkit && npm install`.

## Authoring

`deckkit/README.md` is the reference. The short version — a deck is front
matter, then slides separated by `---`:

````markdown
---
marp: true
theme: deckkit
size: 16:9
paginate: true
---

<!-- _class: dense -->

# Slide title

The paragraph under the title is styled as the lead automatically.

::: cards cols=2 accent=sky
### First card {tag="Category"}

Body markdown, **bold**, bullets — all normal.
:::
````

Two rules keep decks consistent: slide files carry content only — anything
visual belongs in `deckkit/themes/deckkit.css` so every deck inherits it — and
colours are named accents (`sky`, `emerald`, `purple`, `amber`, `rose`,
`indigo`, `slate`, `navy`), never hex.

## Decks

| Deck | Occasion |
| --- | --- |
| `asv2026` | American Society for Virology 2026 — BRC Analytics, Galaxy workflows, Orbit, Logan/LexicMap. 15 min |
| `toolsfortomorrow_May_23_2026` | Andes Hantavirus glycoprotein selection analysis — BRC-Analytics / Datamonkey / HyPhy, May 2026 |
| `paper_sra_linkage_Aug_2026` | Linking papers to the sequencing data they generated — ground truth for a metadata-prediction experiment. Lab meeting, Aug 2026 |
| `sra_metadata_schema_Sep_2026` | Agreeing on a target schema — three schemas that disagree, run- versus experiment-level records, and which biological fields to add. Decision meeting, Sep 2026 |
