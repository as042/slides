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

## Decks and tools

| | Occasion |
| --- | --- |
| `field-picker.html` | Target schema field picker &mdash; 206 candidate fields for SRA metadata enrichment, with prevalence, synonyms, overlaps and real example values. Not a deck; a standalone single-file tool, no build step |
| `sra_metadata_rust_Sep_2026` | Filling the blanks in SRA metadata &mdash; the Rust reconstruction harness. Lab meeting, Sep 2026 |

The decks inherited from the upstream `nekrut/slides` fork were removed; only
Andrew's own material is kept. `deckkit/` stays, because it is the build system
this deck compiles with rather than a deck of its own.
