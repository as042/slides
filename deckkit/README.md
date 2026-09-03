# deckkit

A Marp-based presentation system: write a **content-only markdown file**, get the
polished HTML deck. All styling lives in one theme (`themes/deckkit.css`); slide
files contain no CSS.

```bash
deckkit/bin/deck build  mydeck/slides.md    # -> slides.html
deckkit/bin/deck png    mydeck/slides.md    # -> slides.001.png, ...
deckkit/bin/deck pdf    mydeck/slides.md
deckkit/bin/deck watch  mydeck/slides.md    # live preview
deckkit/bin/deck new    mydeck              # scaffold a new deck
```

Reference deck: `../sra_metadata_rust_Sep_2026/slides.md`.

## Deck file skeleton

```markdown
---
marp: true
theme: deckkit
size: 16:9
paginate: true
header: '![logo](https://example.org/logo.svg) Series | Subtitle'
footer: 'Event • 22 May 2026'
---

<!-- _class: title -->

# Deck title

## Deck subtitle

---

<!-- _class: dense -->

# Slide title

One sentence of context — the paragraph right under an `h1` is styled as the lead
automatically.

...components...
```

Slides are separated by `---`. Per-slide classes go in a `<!-- _class: ... -->`
comment.

### Slide classes

| Class | Use |
| --- | --- |
| `title` | title slide: gradient wash, navy edge, no header/footer |
| `divider` | full-navy section break, for a statement slide |
| `compact` / `dense` / `micro` | density steps (20 / 19 / 18px) for busy slides |
| `headroom` | pulls the title block up when a slide is very full |
| `middle` | centres a sparse slide's content vertically |

Slides are a fixed 1280×720. When content grows, step the density class down
rather than hand-tuning spacing.

## Block components — `::: name`

Nest them freely; every level uses plain `:::`.

### `cards` — a grid of cards, one per `###`

```markdown
::: cards cols=2 gap=15px size=xs
### Recombination Screening (GARD) {tag="Control Module" accent=slate}

Identifies breakpoints to control for phylogenetic conflict.

- **Pre-flight Check:** Reduces false positive selection signals.

### Temporal Selection Spikes {tag="When? (Episodic)" accent=sky}

Detects transient evolutionary pressure.
:::
```

Container attributes (`cols`, `gap`, `ratio`, `accent`, `border`, `size`, `caps`,
`checks`, `thumb`) become defaults for every card; the `{...}` block on a
heading overrides them per card.

Drop a markdown image into a card body and it is framed automatically. Set
`thumb=120px` on the grid to crop every image to a uniform band — six full-page
screenshots are never legible on a slide, but six equal thumbnails read as a
sequence.

Card attributes: `tag`, `icon`, `subtitle`, `accent`, `border=left|top|none`,
`size=xs|sm|md|lg`, `caps` (uppercase title), `checks` (✓ bullets).

### `pillars` — linked cards under a brace

For "these N things, and the thing that unites them".

```markdown
::: pillars brace="Agents" accent=sky
### Data

NCBI · EBI · UCSC Genome Browser

### Tools + Workflows

BioConda · BioContainers · Workflows
:::
```

Cards are joined by connectors across the gap; `brace` draws the gathering
bracket and its label. `faint` greys the brace, `plain` drops the connectors.

### `card` — a single card

```markdown
::: card title="Why Analyze Glycoprotein M?" accent=navy size=md caps
Body markdown.
:::
```

### `cols` — columns split by `+++`

```markdown
::: cols ratio="1.15fr 0.85fr"
left column
+++
right column
:::
```

Without `+++` it is a plain grid wrapper (`cols=3`, `gap=30px`, …).

### `callout` — the accented strip

```markdown
::: callout title="Integration Strategy" icon="https://www.hyphy.org/images/logo.svg" dim slim
Body text.
:::
```

`icon` takes a URL (rendered as an image, `dim` darkens a light logo) or an emoji.
`slim` is the compact variant for the bottom of a dense slide.

### `note`, `box`, `figure`, `presenter`

```markdown
::: note accent=emerald
**Conclusion:** No significant change in selection intensity.
:::

::: box title="Summary Takeaways" accent=sky size=md
- Point one
:::

::: figure src="results/tree.png" h=360px
Optional caption
:::
```

`figure` takes `src`, `h`, `alt`, and `bare` (no frame).

## Data components — fenced blocks

Text fields accept inline markdown; ` // ` becomes a line break.

### `metrics` and `stats`

`metrics` is label/value rows on a dashed rule; `stats` is a row of headline
figures (`value | label | note`).

```
​```metrics accent=sky
Human Branches dN/dS | 0.0224
P-value (Threshold 0.05): 0.3104 (NS)
​```

​```stats accent=sky
1,920 | pathogen taxa | bacteria, viruses and fungi
5,060 | genome assemblies | with annotations
​```
```

### `bars`

A single-series horizontal bar chart — one row per category, `label | value |
note` plus optional trailing `k=v` fields. Bar lengths scale against `max=`
(default: the largest value in the block), so the geometry is the data.

```
​```bars accent=sky unit="pairs" wide
0.00 | 21 | no surname in common
0.75–1.00 | 9 | accent=emerald
​```
```

`accent=` on a row recolours that bar to separate a highlighted band from the
rest; `wide` is the roomier variant for a chart that carries a slide alone.
Every bar is direct-labelled with its value, so the chart never relies on
colour alone.

### `timeline`

```
​```timeline
track: Hantavirus Submissions | Active Monitoring (n=994) | accent=sky
node: 10% | Oct 2025 | 174 hits | Peak Submissions // (Spring Onset) | lg
node: 80% | Mid May '26 | 0 hits | Complete Gap | open
​```
```

Node fields: `position | label | value | note | size`. Sizes: `lg`, `md`, `sm`,
`open` (hollow dashed dot).

### `genemap`

Positions are real coordinates; percentages are derived from `length=`, so
segments and markers stay aligned.

```
​```genemap length=1137 accent=purple
segment: Gn Head (1–512) | 1-512 | #3B82F6
segment: | 648-651 | #EF4444 | title="Cleavage Motif"
ticks: 1, 512, 648, 1000, 1137
mark: 71
mark: 978 | 978: ACA | raise
​```
```

`mark: position | label | raise` — `raise` staggers a marker that would collide
with its neighbour. A segment can force a width with `w=0.8%`.

### `embed` — a live widget

Inlines a self-contained HTML fragment (markup + `<style>` + `<script>`) at
build time, for something a static image cannot carry. `w` and `scale` size a
widget built for a browser viewport down onto the 1280×720 slide — pass units,
`w=1300px` not `w=1300`.

```
​```embed src="assets/orbit-demo.html" w=1300px scale=0.80 h=485px
​```
```

Paths resolve relative to the deck directory. Animation runs in the HTML deck;
a PNG export captures whatever frame it lands on.

### `pills`, `motif`, `brandbar`

```
​```pills
22 | Gn Head
685 | Gc Head
​```

​```motif accent=purple label="Site 649 (Middle Alanine)"
W - A - [A] - S - A
​```

​```brandbar
https://brc-analytics.org/logo/brc.svg | 28 | invert
https://datamonkey.org/assets/img/header-logo.svg | 24
​```
```

`[X]` highlights a motif position. `invert` flips a dark logo white for the
brandbar's navy plate.

## Inline

- `{{logo T:67 L:33}}` — sequence-logo stack; works inside table cells.
  A residue without a percentage takes the remaining share.
- `[text]{.class}` — span with classes. Useful ones: `.badge`, `.mono`, `.dim`,
  `.muted`, `.center`, `.big` and `.xl` for statement text, colours `.c-sky`
  `.c-emerald` `.c-purple` `.c-amber` `.c-rose`, accents `.accent-purple`
  (pairs with `.badge`).
- `::: div .todo` — a dashed placeholder block for a template slide, so unfinished
  spots are impossible to miss.
- Markdown tables are styled by the theme — use them instead of HTML tables.

## Colour

Accents are named, not hex: `sky`, `emerald`, `purple`, `amber`, `rose`,
`indigo`, `slate`, `navy`. Pass `accent=` to any component; it drives the border,
bullets, tags, metric values and marker colours together.

Re-brand a deck without touching the theme by overriding tokens in front matter:

```yaml
style: |
  :root { --sky: #B91C1C; --ink-strong: #450A0A; }
```

## Layout

`lib/` holds the markdown-it plugin — `container.mjs` (`:::` parsing),
`components.mjs` (block components), `fences.mjs` (data components),
`inline.mjs` (`{{...}}`), `attrs.mjs` (shared parsing). `marp.config.mjs` wires
the plugin and theme into marp-cli.

Local images are referenced by relative path in the built HTML, not inlined —
ship the asset files alongside the deployed `.html`.
