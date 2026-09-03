# Presentations

All decks in this directory are built with **deckkit** (`deckkit/`), a Marp theme
plus markdown-it plugin. Read `deckkit/README.md` for the component reference
before editing or creating a deck.

## Core rule

Slide files carry **content only**. No `<style scoped>` blocks, no inline
`style=` attributes, no ad-hoc `<div>` layout. If a slide needs something the
component vocabulary cannot express, add it to `deckkit/themes/deckkit.css` (and
a component in `deckkit/lib/` if it takes data) so every deck gets it.

Colours are named accents (`sky`, `emerald`, `purple`, `amber`, `rose`, `indigo`,
`slate`, `navy`), never hex in a slide file.

## Build

```bash
deckkit/bin/deck build <deck>/slides.md    # HTML
deckkit/bin/deck png   <deck>/slides.md    # one PNG per slide
deckkit/bin/deck watch <deck>/slides.md    # live preview
deckkit/bin/deck new   <deck>              # scaffold
```

## Toolchain on this machine

Node is user-local: `~/.local/share/node` symlinked into `~/.local/bin`, which is
already on `PATH`. `deckkit/node_modules` is not committed — after a fresh clone
run `cd deckkit && npm install --no-save` (`--no-save` keeps npm from rewriting
`package-lock.json` with a `license`/`bin` normalisation that is pure noise).

**Rendering needs a Chrome, and there isn't a system one.** Firefox is a snap and
puppeteer cannot drive it — it times out waiting for the debug endpoint. A
standalone Chrome for Testing lives in `~/.cache/puppeteer`. Ubuntu 24.04 also
disables unprivileged user namespaces, so Chrome refuses to start without
`CHROME_NO_SANDBOX`. Both are needed for `png`, `pdf` and `pptx`:

```bash
export CHROME_PATH=~/.cache/puppeteer/chrome/linux-152.0.7977.64/chrome-linux64/chrome
export CHROME_NO_SANDBOX=1
```

**Live preview without Chrome:** `deck watch` passes `--preview`, which opens a
Chrome window, so use marp's server mode instead. It converts on request, injects
its own reload client, and writes nothing into the repo:

```bash
./deckkit/node_modules/.bin/marp --no-stdin --config ./deckkit/marp.config.mjs \
    --server . --port 8080
```

Then open `http://localhost:8080/<deck>/slides.md`.

## Verify

Slides are a fixed 1280×720. After editing, render PNGs and **look at them** —
for vertical overflow, footer collisions and broken image paths. Several problems
here are invisible in the markdown and obvious in the PNG.

Density classes are for content that **overflows**. Reaching for `dense` or
`micro` on a slide that fits leaves it half empty; `compact middle` is the
working default for a content slide. When a slide genuinely runs long, step the
class down (`compact` → `dense` → `micro`) or drop the card `size=` a notch
rather than nudging spacing by hand.

## deckkit traps, each found the hard way

- **`bin/deck build` cannot retarget its output.** It hard-codes `-o
  <name>.html`, and a second `-o` makes marp-cli die with a stack trace rather
  than taking the last one. To build a deck to `index.html`, call marp directly:
  `deckkit/node_modules/.bin/marp --no-stdin --config deckkit/marp.config.mjs
  slides.md -o index.html`.
- **Component attributes have no escape syntax.** `title="say \"this\""` parses
  as `say \` and silently truncates — the info-string regex stops at the first
  inner quote. Use single quotes inside a double-quoted value.
- **`.footnote` is pinned bottom-left on the footer's own baseline**, with no
  width limit, so a long one runs underneath the footer text. Keep them to
  roughly 85 characters; anything longer belongs in a `callout slim` or a `note`,
  which reserve their own space.
- **Timeline node text wraps inside one fifth of the lane.** Keep each node's
  label, value and note to a few words or a node drops out of the track box and
  collides with whatever is below it.
- **Rebuild after editing, every time.** Checking a PNG while the `.html` is
  stale has wasted time here more than once.

Local images are referenced by relative path in the built HTML, not inlined —
any deployment must copy the asset files alongside the `.html`.
