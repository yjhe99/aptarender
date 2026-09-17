# AptaRender — Developer Handoff Guide

Audience: a coding agent (or developer) picking up AptaRender to implement **Feature #4:
two-strand partial hybridization**. Read this whole file before writing code.

---

## 1. What AptaRender is

An **art-first, browser-based editor for 2D nucleic-acid secondary-structure figures.**
The user supplies a **sequence** and a **dot-bracket structure**; the app draws an
editable 2D diagram for making publication-quality figures.

- It does **NO structure prediction.** The structure is user-supplied. Never add folding/MFE prediction.
- Priorities are **aesthetic control and clean vector export**, not biophysical accuracy.
- Live site: https://yjhe99.github.io/aptarender/

## 2. The single most important fact: it is ONE file, zero dependencies

The entire app is **`index.html`** — HTML + CSS + vanilla JS in one file. No build step,
no framework, no npm, no bundler, no external libraries. Keep it that way. Match the
existing code style (plain functions, terse but readable, comments where non-obvious).

To run it locally:
```bash
# from the repo folder
python3 -m http.server 8731
# then open http://localhost:8731/index.html
```
(Opening the file directly with `file://` also works for most things, but a local HTTP
server avoids browser file-access restrictions and matches how it's deployed.)

## 3. Deployment & workflow

The app is a static single file served by **GitHub Pages** — there is no build step, so a
"deploy" is just the file being published (a first build takes ~1 min, and the CDN may
cache the previous file briefly). For a large feature like #4, create a **feature branch**
and open a pull request rather than committing straight to the default branch.

## 4. Architecture of `index.html`

Rough top-to-bottom map (search for these anchors; line numbers drift):

- **`<style>`** — all CSS. Panel is a left sidebar (`#panel`) with a sticky tab navigator
  (`#pantop` / `#tabs`) and one `<section class="pane">` per tab. Canvas is `#stage` > `svg#canvas`.
- **`<div id="panel">`** — the control panel, split into panes: `input`, `layout`,
  `letters`, `circles`, `lines`, `export`. Every control has a stable `id`.
- **`<script>`**, in order:
  - `const M = {…}` — the **live model** for the currently-active sequence.
  - `parseStructure(seq, dot)` — validates and returns `{seq, n, pt, pk}`. `pt` is the
    **pair table**: `pt[i]` = index of i's partner, or `-1`. Only nested `()` supported
    (pseudoknot brackets throw). Rejects length mismatches / bad chars.
  - `computeLayout(n, pt, S, tailMode)` — **the layout engine.** Our own radial layout:
    loops placed on circles, helices as straight ladders, built recursively from the
    exterior loop outward (`drawLoop`, `buildStem`). Ends with a **tail post-process** that
    reshapes the free 5′/3′ ends (natural / flatten / L / perp), and a special case that
    linearizes the whole strand when there are no pairs. Returns `{x, y}` arrays (model
    coordinates, in px; `SPACING`≈42 between bases, independent of font size).
  - `render()` — builds the SVG from model coords. Applies rotation in coordinate space so
    **glyphs stay upright** while the structure rotates. Draws (in order) H-bond rungs,
    backbone, then two base layers: all circles first, then all letters (so letters always
    sit above overlapping circles). Zoom/pan is an SVG group transform.
  - `fit()` — fits content to the canvas (guards against a zero-sized canvas via
    `requestAnimationFrame`).
  - `buildExportSVG()` / PNG export — regenerates a standalone SVG string (WYSIWYG with
    the on-screen render) for download; PNG rasterizes it at a chosen scale.
  - **Multiple-sequences block** (`docs`, `active`, `saveActive`, `applyDoc`, `switchDoc`,
    `addDoc`, `deleteDoc`, `importFasta`, `parseFasta`, `readStyle`/`applyStyle`,
    `persist`). See §5.
  - **Wiring** (`$("id").onclick = …`) and **init** (restores `localStorage`, else demo).

### 4a. Coordinate model
- Model coords `M.x[i], M.y[i]` are in an abstract px space (base spacing = `SPACING`).
- On render, each point is rotated about the structure centroid by `M.rot`, then the whole
  group is translated/scaled by `M.panX/M.panY/M.zoom`. Text is drawn at the rotated point
  with **no per-glyph rotation** (that's the "text stays upright" trick).

## 5. Data model

`M` (active sequence):
- `seq` (string), `n` (length), `pt` (pair table), `x[]`, `y[]` (coords)
- `color[]` — per-base **letter** color
- `circleColor{}` — per-base circle-**fill** override (keyed by index)
- `bondColor{}` — per-H-bond color (keyed by the lower base index of the pair)
- `selected` (Set of base indices), `selBond` (Set of lower-index of selected rungs)
- `zoom, panX, panY, rot, font`, `palette{A,U,T,G,C}`

`docs` (multiple sequences): array of documents; `active` = index. Each doc =
`{name, seq, dot, color[], circleColor{}, bondColor{}, view:{zoom,panX,panY,rot}, style}`.
`style` is a snapshot of the global style controls (see `STYLE_IDS`), used when the
"Uniform style across all" checkbox is OFF. Switching docs = `saveActive()` then
`applyDoc(i)`. Everything persists to `localStorage["aptarender_docs"]`.

## 6. Conventions & gotchas (things that already bit us)

- **No dependencies, one file.** Don't introduce any.
- **Background is a real `<svg><rect>`**, not a CSS background — needed so the preview is
  WYSIWYG with export and immune to Chrome's Auto Dark Mode (which silently darkens white
  CSS backgrounds in the preview pane; the exported file is unaffected).
- **Two-pass base rendering**: draw all circles, then all letters, so letters aren't
  covered by a neighbor's filled circle. Do the same in `buildExportSVG`.
- **Clickable hit targets**: bases/bonds have an invisible transparent hit shape so they
  stay selectable even when the visible element has `fill:none`.
- **`fit()`** must guard against a zero-sized canvas (it reschedules via rAF) — otherwise
  zoom becomes 0 on first load.
- **`beforeunload`** saves state; `localStorage` is the only persistence (no backend).
- Keep every control's `id` stable — lots of wiring and the doc `style` snapshot depend on ids.
- Test by loading in a browser against the local server and checking the console for errors;
  there is no test suite. Verify visually and by inspecting rendered SVG elements.

---

## 7. FEATURE #4 — Two-strand partial hybridization

### Goal (from the maintainer)
Display **two sequences on one canvas** and let the user **manually set a partial
hybridization (hetero-duplex) region** between them. **The hetero-duplex has the highest
priority: any intramolecular stem-loop that conflicts with it must be broken.** In other
words, where strand A base-pairs with strand B, any A–A or B–B pair involving those bases
is removed, and the two strands are drawn joined by the intermolecular duplex.

Typical use cases: aptamer–target duplexes, toehold/strand-displacement, primer binding.

### Recommended prerequisite: drag-to-reposition (do this FIRST)
Before #4, implement **drag-to-reposition** — let the user drag a whole stem/loop
sub-branch (and individual bases / number labels) to new positions, with the rest anchored.
Reasons:
1. It's independently the most-requested editing capability (and covers the earlier request
   to move nucleotide-number labels).
2. It **de-risks #4 dramatically**: with manual dragging, the #4 auto-layout only needs to
   produce a *reasonable starting point*, not a perfect one. Getting a perfect automatic
   two-strand layout is the expensive part; dragging removes that requirement.

Suggested drag design: on mousedown on a base, determine the "branch" (the closed
sub-structure rooted at the nearest enclosing pair) and translate all its coordinates by
the mouse delta; keep the connecting backbone attached. Store manual offsets so re-render
/ rotation preserve them. Add a modifier or mode for single-base vs whole-branch drag.

### Suggested plan for #4 (phased)

**Data model.** Introduce a "complex" that references two docs (strand A, strand B) plus a
list of hetero-duplex pairings. Simplest first version: **one contiguous duplex region** —
`{aStart, aEnd, bStart, bEnd}` meaning A[aStart..aEnd] pairs antiparallel with
B[bStart..bEnd] (so A[aStart]–B[bEnd], …, A[aEnd]–B[bStart]). Validate equal lengths.

**Conflict resolution (easy).** Build each strand's `pt` normally, then delete any `()`
pair with an endpoint inside that strand's hybridized region. This "breaks" conflicting
stem-loops as required. Keep the surviving intramolecular structure on each flank.

**Layout (the hard part).** Do NOT try to reuse the single-strand radial layout as-is.
Recommended approach for the single-region case:
1. Place the **intermolecular duplex as a straight ladder** in the center (A left→right on
   top, B right→left on the bottom, antiparallel; rungs connect the hetero pairs).
2. Each strand now has up-to-two **flanks** hanging off the duplex ends: A's 5′ flank and 3′
   flank, B's 5′ and 3′ flank. Each flank is a normal single strand (possibly with its own
   surviving hairpins). **Lay out each flank with the existing `computeLayout`** in
   isolation, then **rotate/translate** it so its attachment base sits at the correct duplex
   end and it extends outward (away from the duplex). This "stitch the radial sub-layouts
   onto the duplex" approach reuses existing code and keeps the new math to coordinate
   transforms.
3. Render two backbones + the duplex ladder (style the hetero rungs distinctly, e.g. a
   different color/weight) + a strand-break indicator between the two 5′/3′ ends.
Because dragging exists, imperfect flank angles are fine — the user nudges them.

**UI.** In a new pane or the Input pane: pick strand A and strand B (from the `docs` list),
enter/select the A range and B range for the duplex, and a button to build the complex.
Show it as its own document type (a "complex"), consistent with how docs work. A
double-click into a strand to edit it (as the maintainer described) can reuse `applyDoc`.

**Coloring/editing across two strands.** The current per-base model is single-sequence
(indices into one `seq`). For a complex, either (a) keep two separate index spaces and tag
selections with which strand, or (b) build a combined coordinate/index list for rendering
and map back to per-strand indices. Option (a) is cleaner; render already just needs coord
arrays + color arrays.

**Scope for v1 (keep it small):**
- Exactly **two strands**, **one contiguous** hetero-duplex region.
- Defer: multiple separate duplex regions between the same two strands (they create
  inter-strand internal loops — much harder), 3+ strands, and any auto-detection of
  complementarity (it stays manual, matching the app's no-prediction philosophy).

### Definition of done for #4 v1
- User can create a two-strand complex, set one duplex region, and see both strands joined
  by the duplex with conflicting intramolecular pairs broken.
- The complex renders, exports to SVG/PNG, and its pieces can be dragged into place.
- No regression to single-sequence mode or the multi-sequence (#2) workflow.
- Still one file, no dependencies, no console errors.

---

## 8. What's in the repo
Everything needed is in the **git repo**:
- `index.html` — the app (all code).
- `README.md`, `CHANGELOG.md` — docs.
- `HANDOFF.md` — this guide.

There is **nothing outside git**: no secrets, API keys, env files, build config, or
database — it's a static single-file app persisted only in the browser's `localStorage`.
For a large feature like #4, branch off the default branch and open a PR.
