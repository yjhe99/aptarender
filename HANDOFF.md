# AptaRender — Developer Handoff Guide

Audience: a coding agent (or developer) picking up AptaRender next. Read this whole file
before writing code. **Feature #4 (two-strand hybridization), drag-to-reposition, and
undo/redo — all previously proposed below as future work — are now implemented.** This
version of the doc keeps the original architecture notes and adds what actually got
built, so both the "why" and the "how it turned out" are on record.

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
python3 -m http.server 8731
# then open http://localhost:8731/index.html
```

There is still **no browser test runner in the repo**. Since this doesn't have a build
step, tests were done ad hoc during development using Node + `jsdom` against the actual
file (see §9). If you add a real test suite, keep it optional / dev-only — don't add it
as a dependency of the shipped page.

## 3. Deployment & workflow

The app is a static single file served by **GitHub Pages** — there is no build step, so a
"deploy" is just the file being published. For a large feature, create a **feature
branch** and open a pull request rather than committing straight to the default branch.

## 4. Architecture of `index.html`

Rough top-to-bottom map (search for these anchors; line numbers drift):

- **`<style>`** — all CSS. Panel is a left sidebar (`#panel`) with a sticky tab navigator
  (`#pantop` / `#tabs`) and one `<section class="pane">` per tab, now including a
  **Complex** tab. A thin **`#panelResize`** divider sits between `#panel` and `#stage`
  and is draggable (see §8). Canvas is `#stage` > `svg#canvas`.
- **`<div id="panel">`** — the control panel, split into panes: `input`, `layout`,
  `letters`, `circles`, `lines`, `complex`, `export`. Every control has a stable `id`.
- **`<script>`**, in order:
  - `const M = {…}` — the **live model** for the currently-active single sequence.
  - `const CM = {…}` and `let complexes = []` — the **complex model** (see §7).
  - `parseStructure(seq, dot)` — validates and returns `{seq, n, pt, pk}`. `pt` is the
    **pair table**: `pt[i]` = index of i's partner, or `-1`. Only nested `()` supported
    (pseudoknot brackets throw). Rejects length mismatches / bad chars.
  - `computeLayout(n, pt, S, tailMode)` — **the layout engine.** See §4b — this changed
    significantly from the original design.
  - `render()` — builds the SVG from model coords. Dispatches to `renderComplex()` first
    if `CM.active`. Applies rotation in coordinate space so **glyphs stay upright** while
    the structure rotates. Draws (in order) H-bond rungs, backbone, then two base layers:
    all circles first, then all letters (so letters always sit above overlapping
    circles). Zoom/pan is an SVG group transform.
  - `fit()` — fits content to the canvas (guards against a zero-sized canvas via
    `requestAnimationFrame`). Dispatches to `fitComplex()` if `CM.active`.
  - `buildExportSVG()` / PNG export — regenerates a standalone SVG string (WYSIWYG with
    the on-screen render) for download; PNG rasterizes it at a chosen scale. Dispatches to
    `buildComplexExportSVG()` if `CM.active`.
  - **Multiple-sequences block** (`docs`, `active`, `saveActive`, `applyDoc`, `switchDoc`,
    `addDoc`, `deleteDoc`, `importFasta`, `parseFasta`, `readStyle`/`applyStyle`,
    `persist`). See §5. `applyDoc(i)` now always sets `CM.active=false` first.
    `deleteDoc(i)` now also removes/reindexes any saved complex that referenced the
    deleted sequence, with a hint shown to the user.
  - **Complex block** (`newComplex`, `openComplex`, `deleteComplex`, `renderComplexList`,
    `refreshComplexPickers`, `computeComplexLayout`, `computeSideHybridLayout`,
    `breakForDuplex`, `attachFlank`, etc.). See §7.
  - **Undo/redo block** (`history`, `pushHistory`, `applyHistorySnapshot`, `undo`, `redo`).
    See §6.
  - **Drag-to-reposition** wiring inside the `svg` mouse handlers (`baseDrag`,
    `branchIndicesIn`). See §6.
  - **Wiring** (`$("id").onclick = …`) and **init** (restores `localStorage`, else demo).

### 4a. Coordinate model
- Model coords are in an abstract px space (base spacing = `SPACING`).
- Single-sequence mode: `M.x[i], M.y[i]` plus a per-base manual drag offset,
  `M.offX[i]/M.offY[i]`. The **effective** coordinate used everywhere for
  rendering/rotation is `ex(i)/ey(i)` = layout coord + offset — never read `M.x/M.y`
  directly if a drag offset might apply.
- Complex mode: `CM.xA/yA` + `CM.offXA/offYA` for Strand A, `CM.xB/yB` + `CM.offXB/offYB`
  for Strand B, read through `cex(i)/cey(i)` and `cbx(i)/cby(i)` respectively.
- On render, points are rotated about the structure's centroid by `M.rot` (single) using
  the effective coords, then the whole group is translated/scaled by `M.panX/M.panY/M.zoom`.
  Text is drawn at the rotated point with **no per-glyph rotation** (that's the "text
  stays upright" trick). Complex mode shares this same `rot/zoom/panX/panY/font` view
  state, computed over both strands' combined centroid.

### 4b. The layout engine — how it actually draws a structure now

`computeLayout(n, pt, S, tailMode)` is the core algorithm. There's a hard rule baked into
it now, driven directly by user feedback:

> **A run of unpaired bases only gets drawn as a circle/arc if it's actually enclosed by
> a real `(`…`)` pair.** Anything not enclosed by a pair — free 5′/3′ tails, linkers
> between separate top-level stems, or an entire strand with no pairs at all — is drawn
> as a straight line instead.

Concretely:
- **`drawLoop(ci,cj,aoutX,aoutY)`** — unchanged in spirit from the original design. Only
  ever called for content strictly between a real closing pair `(ci,cj)`, so it's the
  correct place for circular/radial placement and nothing here needed to change.
- **The "exterior level"** (the top of `computeLayout`, content with no enclosing pair at
  all) was rewritten from a circular "virtual opening" arrangement to a simple forward-
  flowing cursor: unpaired top-level bases are placed one `S` apart along a straight line
  (`curX += S` each step). A top-level base pair's two rails are placed **continuing in
  that same forward direction** — not turning 90° off the backbone — offset from each
  other by one rung-width (`S`) perpendicular to the direction of travel. `buildStem` is
  handed a synthetic "center" point positioned to force its outward direction to be
  "keep going forward" rather than the loop-relative direction it computes for nested
  loops. After the branch, the cursor resumes just past whichever rail extends furthest,
  back at the pre-stem baseline y.
  - **Known limitation:** if a sequence has *multiple* separate top-level stems, the
    backbone's y-baseline after each one is reset to where it was before that stem, so
    consecutive top-level stems don't compound-drift, but the very last visible
    stretch after a stem sits at a slightly different y than logically "continuous" —
    acceptable in practice (drag fixes it), not geometrically perfect.
- **Free tail reshaping** (`tailMode`: flatten / L / perp) is a post-process, unchanged in
  approach — it still works because it derives its direction from the actual positions of
  the anchor and its neighbor, not from any assumption about how the exterior level was
  laid out. The **"natural"** tail mode is now just "leave the default layout as-is" — it
  no longer curves tails (the default layout is already straight for anything unenclosed,
  per the rule above), so its label was changed from "Natural (curved)" to
  "Natural (default layout)".

If you touch `computeLayout` again: the exterior-level rewrite is the part most likely to
surprise you. Test with (a) a fully unpaired sequence, (b) a single hairpin with tails on
both sides, and (c) two separate top-level hairpins joined by an unpaired linker — those
three cases cover the interesting behavior.

## 5. Data model

`M` (active single sequence): `seq`, `n`, `pt`, `x[]`, `y[]`, `offX[]`, `offY[]` (drag
offsets), `color[]`, `circleColor{}`, `bondColor{}`, `selected` (Set), `selBond` (Set),
`zoom, panX, panY, rot, font`, `palette{A,U,T,G,C}`.

`docs` (multiple sequences): array of `{name, seq, dot, color[], circleColor{}, bondColor{}, view, style}`.
`active` = index. Switching docs = `saveActive()` then `applyDoc(i)`. Persists to
`localStorage["aptarender_docs"]`, which now also stores `complexes`.

`CM` (active complex view — mirrors `M` but for two strands): `active` (bool), `idx`
(index into `complexes`), `seqA/nA/ptA/xA/yA/offXA/offYA/selA`, the same set for B, and
`dupLen`. `complexes[]`: array of `{name, docA, docB, region:{aStart,aEnd,bStart,bEnd}, mode}`
where `mode` is `"replace"` or `"side"`.

## 6. Drag-to-reposition & Undo/Redo (implemented)

**Drag-to-reposition.** On `mousedown` on a base, `branchIndicesIn(pt, i, wholeBranch)`
computes the set of indices to move: if the clicked base is paired and `wholeBranch` is
true (the default — false only when Alt is held), it's the whole contiguous range
`[min(i,pt[i]), max(i,pt[i])]` (this works because nested dot-bracket structure guarantees
a paired base's entire sub-structure is exactly that contiguous range). Dragging adds the
mouse delta (converted from screen space to model space, accounting for the current
rotation/zoom) to `M.offX/offY` (or `CM.offXA/offYA` / `CM.offXB/offYB` in complex mode,
chosen via a `data-strand` attribute on the clicked SVG element) for every index in that
set. Offsets are applied at render time via `ex(i)/ey(i)` (or `cex/cey`, `cbx/cby`) — they
survive re-render, rotation, and are included in export.

**Undo/Redo.** A full-state snapshot stack (`history[]`, `histIndex`, `HISTORY_LIMIT=100`).
`touch()` — called after basically every mutating action — calls `pushHistory()` instead
of the old plain `persist()`. `Ctrl+Z`/`Cmd+Z` calls `undo()`, `Ctrl+Y` or `Ctrl+Shift+Z`
calls `redo()`, both guarded so they don't hijack a focused `<textarea>`'s native undo.
`switchDoc` and `beforeunload` deliberately still call plain `persist()` (switching docs
isn't itself an undoable "edit", and saving on unload shouldn't push a history entry).

## 7. Feature #4 — Two-strand partial hybridization (implemented)

Two building modes are both implemented, driven by `cx.mode` (`"replace"` or `"side"`).
Both take a single contiguous region `{aStart,aEnd,bStart,bEnd}` (validated equal length,
≥2nt) and both output `{xA,yA,xB,yB,ptA,ptB,dupLen,broken,warning}` for `renderComplex()`.

### `computeComplexLayout` — Replace mode
Matches the original plan in this doc almost exactly:
1. `breakForDuplex(pt,s,e)` deletes any `()` pair with either endpoint inside a strand's
   own hybridized range — applied to **both** strands.
2. The hybridizing region is drawn as a fresh straight ladder (ignoring whatever the
   strands' own layouts would have put there).
3. Each strand's surviving structure (if any) on either side of the duplex is laid out
   independently via the ordinary `computeLayout`, then rotated/translated into place by
   `attachFlank` so its attachment base sits at the correct duplex end.
   - `attachFlank`'s rotation heuristic is **PCA-based** (principal axis of the flank's
     own point cloud), not "farthest single point" — more robust when a flank is lopsided
     (a small stem remnant + a big loop + a long leftover tail all on one side, which
     happens whenever the hybridized region only partially overlaps an existing stem).
   - The **stand-off distance** placing a flank near the duplex end is **adaptive**:
     `standoff = R + S`, where `R` is the flank's own farthest reach from its attach
     point. A fixed stand-off could leave a flank's farthest point closer to the *other*
     side of the duplex than to its own attach point, causing overlap; this guarantees
     (by the triangle inequality) every flank point stays clear regardless of rotation.

### `computeSideHybridLayout` — Side-by-side mode
Strand A is **never modified or re-laid-out** — `computeLayout` is run on it completely
normally, and its real coordinates are used directly as a geometric reference (not
rebuilt via a rigid transform of some independently-computed shape, which was tried first
and failed: an unpaired stretch doesn't lay out as a straight line on its own, so
anchoring only its two endpoints left everything in between curved/off-axis).
1. Only Strand B's pairs that straddle `[bStart,bEnd]` are broken (`breakForDuplex`,
   applied to B only).
2. Each hybridizing base of B is placed at a **dynamically-computed perpendicular offset**
   from A's corresponding real base, along the direction of A's own arm
   (`A[aStart..aEnd]`). The offset distance and which side of the arm to use are chosen by
   scanning the rest of A's own layout (excluding the arm itself) for anything else A
   occupies within B's longitudinal span, and pushing B out past the farthest such point
   plus a margin — not a fixed distance, which broke as soon as A's own layout put
   another stem or loop at a similar distance from the arm. (See `reach(sign)` in the
   function.)
3. B's own flanking unpaired ends (outside `[bStart,bEnd]`) are laid out the same
   `doFlank`/`attachFlank` way as Replace mode, fanned out on the **same side** as the
   chosen offset so they don't cross back over Strand A.
4. If A's chosen range has no base pairs of its own, a `warning` is returned (not thrown —
   building still proceeds) suggesting the user probably meant to swap A and B: this mode
   only makes sense when A contributes a real existing arm to offset from.

### What's still scoped out (deliberately, per the original plan)
- Multiple separate duplex regions between the same two strands.
- Three or more strands.
- Any automatic complementarity detection — everything is manual, matching the app's
  no-prediction philosophy.

## 8. Resizable panel (implemented)

`#panelResize` is a thin draggable div between `#panel` and `#stage`. Plain mouse
event listeners resize `#panel`'s inline `style.width` (clamped 200px–80vw) and persist
the chosen width to `localStorage["aptarender_panelw"]`. No new dependency.

## 9. Testing approach used during development (no browser available)

Two levels, both throwaway scripts (not committed — recreate as needed):
- **Isolated geometry tests:** extract just `parseStructure`/`computeLayout`/etc. as a
  string slice out of `index.html` and `eval()` it in plain Node, then assert on the
  returned coordinate arrays directly (spread checks, collinearity checks, distances
  between specific bases). Fast, no dependencies.
- **Full jsdom integration tests:** `npm install jsdom`, load the real `index.html` with
  `runScripts:"dangerously"`, patch `canvas.clientWidth/clientHeight` (jsdom always
  reports 0, which would otherwise spin `fit()` forever via `requestAnimationFrame`), and
  drive the actual UI (click buttons, set `<textarea>` values, read `M`/`CM`/`docs` back
  out via an `expose()` helper that injects a follow-up `<script>` tag reading the shared
  top-level `const`/`let` bindings — those don't attach to `window` in a classic script,
  but multiple classic `<script>` tags in the same document do share the same lexical
  scope, which is what makes this trick work). Call `process.exit()` explicitly at the
  end to avoid hanging on any pending timer.

## 10. What's in the repo
- `index.html` — the app (all code).
- `README.md`, `CHANGELOG.md` — docs.
- `HANDOFF.md` — this guide.

There is **nothing outside git**: no secrets, API keys, env files, build config, or
database — it's a static single-file app persisted only in the browser's `localStorage`.
For a large feature, branch off the default branch and open a PR.

## 11. Ideas for what's next (not started)

- Multiple hetero-duplex regions on the same pair of strands (creates inter-strand
  internal loops between them — meaningfully harder layout problem than the single-region
  case).
- Three-or-more-strand complexes.
- A proper committed test suite (the jsdom approach in §9 works fine as a starting point).
- The exterior-level "y-baseline reset after each stem" limitation noted in §4b, if
  multi-stem single-molecule structures turn out to be common in practice and the current
  behavior looks off often enough to be worth a more careful fix.
