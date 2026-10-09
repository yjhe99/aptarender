# AptaRender — Developer Handoff Guide

Audience: a coding agent (or developer) picking up AptaRender next. Read this whole file
before writing code. **Start with §13** — the 2026-10-02 pass replaced the duplicated
single/complex rendering paths with one `strands()` abstraction, and several sections below
were corrected to match. Anything that draws, fits, exports, drags or selects must go
through `strands()` now; don't reintroduce a mode-specific copy.

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
  - `parseStructure(seq, dot)` — validates and returns `{seq, n, pt, nonWC}`. `pt` is the
    **pair table**: `pt[i]` = index of i's partner, or `-1`. `( )` are WC pairs; `[ ] { } < >`
    are non-WC pairs (flagged in `nonWC[]`). All must nest — crossing pseudoknots aren't
    supported. Rejects length mismatches / bad chars / mismatched bracket types.
  - `computeLayout(n, pt, S, tailMode)` — **the layout engine.** See §4b — this changed
    significantly from the original design.
  - `strands()` / `heteroPairs()` / `displayPoints()` — see §13. Present the current view
    (one strand in single mode, two in complex mode) uniformly.
  - `buildScene(interactive)` — the ONE description of the drawing: a flat list of
    `{t, a, text}` SVG element specs in paint order (hetero rungs → per-strand rungs →
    backbones → circles → numbers → letters). `interactive=true` adds hit targets, selection
    rings and bond highlights.
  - `render()` — turns `buildScene(true)` into DOM. Rotation is applied in coordinate space
    (`displayPoints`) so **glyphs stay upright**. Zoom/pan is an SVG group transform.
  - `buildExportSVG()` — serializes `buildScene(false)` into a standalone SVG string, so
    export is WYSIWYG by construction; PNG rasterizes it at a chosen scale.
  - `fit()` / `zoomToSelection()` / `viewToBox()` / `contentBox()` — view fitting.
  - **Multiple-sequences block** (`docs`, `active`, `saveActive`, `applyDoc`, `switchDoc`,
    `addDoc`, `deleteDoc`, `importFasta`, `parseFasta`, `readStyle`/`applyStyle`,
    `persist`). See §5. `applyDoc(i)` now always sets `CM.active=false` first.
    `deleteDoc(i)` now also removes/reindexes any saved complex that referenced the
    deleted sequence, with a hint shown to the user.
  - **Complex block** (`newComplex`, `openComplex`, `deleteComplex`, `renderComplexList`,
    `refreshComplexPickers`, `layoutComplex`, `validateRegion`, `computeComplexLayout`,
    `computeSideHybridLayout`, `breakForDuplex`, `attachFlank`, etc.). See §7.
  - **State block** (`serializeState`, `restoreState`, `persist`) and **undo/redo**
    (`histStack`, `pushHistory`, `applyHistorySnapshot`, `undo`, `redo`). See §6.
  - **Interaction** — `svg` mouse handlers (`baseDrag`, `marquee`, `branchIndicesIn`,
    `partIndicesIn`, `rotateSelection`). See §6.
  - **Wiring** (`$("id").onclick = …`) and **init** (restores `localStorage`, else demo).

### 4a. Coordinate model
- Model coords are in an abstract px space (base spacing = `SPACING`).
- Single-sequence mode: `M.x[i], M.y[i]` plus a per-base manual offset `M.offX[i]/M.offY[i]`.
  Complex mode: `CM.xA/yA` + `CM.offXA/offYA` for Strand A, the same with `B`.
- The **effective** coordinate is `effX(st,i)/effY(st,i)` = layout + offset, for a strand
  descriptor `st` from `strands()`. Never read the raw layout arrays when an offset might
  apply. (The old `ex/ey/cex/cey/cbx/cby` helpers are gone.)
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
  loops. A stem+loop is a U-turn in sequence, so after a top-level branch the cursor
  resumes from where its closing base `b` actually sits and the direction flips.
  **This forward/U-turn scheme is used ONLY when there is exactly one top-level stem.**
- **Two or more top-level stems → `layoutBaseline`** (2026-10-06). The U-turn scheme sent
  the second stem's return rail straight back over the 5′ tail (bases exactly coincident),
  which is what made every Free-tails option look broken on multi-stem aptamers. Now: the
  backbone runs along y=0, each top-level stem is built standing up (-y) from its rung, and
  each branch is pushed right until all its bases are ≥ ~1·S from everything already placed
  (point-to-point clearance, not bounding boxes). Linker bases are spread evenly across the
  gap, so a bond can stretch beyond S when a big loop sits right at the baseline (e.g. a
  10-nt loop on a 2-bp stem → one ~2.5·S bond). Tails continue straight along the baseline.
- **Free tail reshaping** (`tailMode`: natural / flatten / L / perp) is a post-process
  (`placeTail`). The straight heading comes from the default layout itself (anchor → its
  first tail base), which is correct for both exterior layouts. (It used to come from the
  anchor's neighbour inside the stem, i.e. the helix axis — wrong for the baseline layout.)
  **`natural` is handled separately, by `naturalTails()`, and is drawn like a loop:**
  - *Single top-level stem* (`pt[a0]===b0`): the terminal rung (a0,b0) closes a virtual loop
    on the OUTSIDE of the stem, built with drawLoop's own geometry (chord S, r = S/2sin(π/m)).
    m = (#3′ tail) + (#5′ tail) + 2 + g, the 3′ tail runs round from b0 and the 5′ tail from
    a0 the other way, so they curve toward each other. g ≥ 2 empty slots always separate the
    two free tips (never overlap); g grows if a tail would come within 0.9·S of the body.
    A single tail (only 5′ or only 3′) uses the same circle.
  - *Two or more stems*: each tail is a tangent arc curving INWARD (toward the centroid of the
    non-tail bases, i.e. up toward the stems) with loop-sized curvature for its length
    (R = S/2sin(π/(len+2))); R grows ×1.12 until the tail is ≥ 0.9·S from every body base and
    from the tail already placed.
  `L`/`perp` turn OUTWARD (away from the partner / centroid), because a hard 90° inward turn
  would hit the partner strand. `relayout()` clears drag
  offsets on tail bases when the style changes. **"natural" curves again**: it
  was briefly a no-op ("default layout") on 2026-09-27, then restored to a gentle arc whose
  radius grows with tail length (`naturalArcRadius`, 2026-09-30) so long tails never close
  into a circle. The dropdown label is "Natural (curves inward)".

If you touch `computeLayout` again: the exterior-level rewrite is the part most likely to
surprise you. Test with (a) a fully unpaired sequence, (b) a single hairpin with tails on
both sides, and (c) two separate top-level hairpins joined by an unpaired linker — those
three cases cover the interesting behavior.

## 5. Data model

`M` (active single sequence): `seq`, `n`, `pt`, `nonWC`, `x[]`, `y[]`, `offX[]`, `offY[]`,
`color[]`, `circleColor{}`, `bondColor{}`, `selected` (Set), `selBond` (Set),
`zoom, panX, panY, rot, font`, `palette{A,U,T,G,C}`. The view fields (`zoom/pan/rot/font`)
are shared by complex mode.

`docs` (multiple sequences): array of `{name, seq, dot, color[], circleColor{}, bondColor{}, offset, view, style}`.
`active` = index. Switching docs = `saveActive()` then `applyDoc(i)`.

`CM` (active complex view): `active` (bool), `idx` (index into `complexes`),
`seqA/nA/ptA/nonWCA/xA/yA/offXA/offYA/selA`, the same set for B, and `dupLen`. A complex's
per-base colors are NOT in CM — they live in (and are edited directly on) its two source
docs. `complexes[]`: `{name, docA, docB, mode:"replace"|"side", region:{aStart,aEnd,bStart,bEnd}, offsetA, offsetB, view}`.

**`saveActive()` is mode-aware**: in complex view it saves the complex (offsets + view)
instead of writing the stale `M` over `docs[active]`. Call it freely before any action.

**Persistence**: `serializeState()` is the single serializer for BOTH `localStorage`
(`"aptarender_docs"`) and undo snapshots; `restoreState()` is its inverse, used by init
and undo/redo. It includes the global style and uniform flag. Add new persisted fields
there and nowhere else.

## 6. Drag-to-reposition & Undo/Redo (implemented)

**Mouse model on the canvas** (every hit target carries `data-s` = strand index into
`strands()`, and `data-i` = base index):

| Gesture | Moves / selects |
|---|---|
| drag a base | its whole branch — `branchIndicesIn(pt,i,true)`: contiguous `[min(i,pt[i]), max(i,pt[i])]` |
| Alt+drag | just that base |
| Ctrl/⌘+drag | its **part** — `partIndicesIn(pt,i)` (see below) |
| Ctrl/⌘+click | toggle that part in the selection |
| drag a selected base while ≥2 are selected | the whole selection (across both strands in complex mode) |
| click / Shift+click base | select one / toggle |
| drag background | pan |
| Shift+drag background | rubber-band select (adds to selection) |
| click background | clear selection |

Movement converts the mouse delta from screen to model space (undo zoom, then the view
rotation) and adds it to the strand's `offX/offY`. A drag that moved records one undo step.

**Parts** (`partIndicesIn`) are derived from `pt[]` on demand — never stored, so they can't
go stale: *stem* = the maximal stack of consecutive pairs containing `i` (both rails, NOT the
enclosed loop — plain drag already covers the subtree); *ring* = every unpaired base directly
in the loop enclosing `i` (sub-stems skipped); *free* = the contiguous unpaired run with no
enclosing pair (5′/3′ tail, linker).

**Rotate selection** (`rotateSelection(deg, pivot)`, Layout tab): rigid rotation of the
selected bases, positive = clockwise on screen. Pivot `"attach"` = midpoint of the selected
bases whose sequence neighbour is unselected, plus their selected pair partners — for a
stem branch that's its base rung, including a branch that starts at the strand's 5′ end.
`"centroid"` = centre of the selection. Applied as `offX/offY` deltas, so it composes with
earlier drags and is undoable.

**Zoom to selection** (`zoomToSelection`, "Zoom" in the selection bar): fits the view to
the selected bases' bounding box (capped at the wheel-zoom maximum, 30×); empty selection
falls back to Fit.

**Undo/Redo.** A full-state snapshot stack (`histStack`, `histIndex`, `HISTORY_LIMIT=100`)
of `serializeState()` JSON. `touch()` → `pushHistory()` after every mutating action
(dedupes identical snapshots). Continuous controls (sliders, color pickers) are wired with
`liveControl(id)`: re-render on `input`, one undo step on `change`. `Ctrl/⌘+Z` undo,
`Ctrl+Y` / `Ctrl+Shift+Z` redo — skipped only while a *text/number* field has focus.
`switchDoc` and `beforeunload` persist but are not undo points.

## 7. Feature #4 — Two-strand partial hybridization (implemented)

Two building modes are both implemented, driven by `cx.mode` (`"replace"` or `"side"`).
The mode → layout-function dispatch lives ONLY in `layoutComplex(c)` (used by build, open
and relayout); range checks live only in `validateRegion`.
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
   - Placement is done by `attachFlankContinuing`: the flank attaches **one normal bond**
     past the duplex end, continuing in the direction that strand was already heading, and
     only rotates away (smallest angle first, alternating sides) if it would land within
     `0.85·S` of anything already placed (`occupied`). (An earlier "stand-off = R+S" scheme
     described in older versions of this doc no longer exists.)

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
3. B's own flanks (outside `[bStart,bEnd]`) are placed with the same
   `attachFlankContinuing` as Replace mode, with all of A's points in `occupied` so they
   avoid Strand A.
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
- Multi-stem baseline layout (§4b) keeps every stem perpendicular to one straight line; a
  fan / arc arrangement of the exterior loop would avoid stretched linker bonds next to
  very wide loops.
- Everything in §12 below that's marked "not started."

## 12. 2026-09-30 change-request pass

A compiled change-request PDF came in with 2 bugs, 2 "internal" (dev-facing) requests, and
6 user requests. Fixed/shipped this pass, all covered by new jsdom scripts alongside
`smoke_test.js` (`test_overlap.js`, `test_lshape*.js`, `test_brackets.js`,
`test_numoffset.js` — not committed, but worth re-deriving if these regress):

- **Bug — long sequence overlaps in a circle instead of growing:** `computeLayout`'s
  `'natural'` tail style drew every unpaired base on a FIXED-radius arc (`Rc = S*3`). Past
  ~19 bases the arc's total sweep exceeds a full turn and starts reusing the same points —
  that's the "circle instead of a bigger circle" bug, and it hit both a fully-unpaired
  strand (the top of `computeLayout`, no stem at all) and a long free tail attached to a
  real stem (`placeTail`). Fixed by a new `naturalArcRadius(count)` helper: below a sweep
  cap (2.6 rad) the radius is unchanged (no visual difference for ordinary short tails);
  above it, the radius grows just enough to keep the sweep capped, so long runs spiral
  outward instead of closing on themselves. Verified up to 300 nt with no self-overlap and
  exact 1-bond-length backbone steps throughout.
- **Bug — "L-shape mode does not work" for a single sequence:** the corner math in
  `placeTail` itself was fine. What WAS broken: the fully-unpaired branch of `computeLayout`
  (no stem anywhere in the run) only special-cased `'natural'` and silently fell back to a
  dead-straight line for `'L'` and `'perp'` — i.e. those two modes were indistinguishable
  from `'flatten'` for a totally unpaired sequence. Both now get their own real shape there.
- **Bug — "L-shape mode does not work" for Complex mode (the actual, bigger bug):** a
  follow-up report with the exact repro (two hairpins hybridized via Replace mode) found
  the real defect, in `attachFlankContinuing`/`attachFlank`. Every flank (whatever's left of
  a strand beyond a duplex end) was rotated so its principal axis — computed by a point-
  cloud PCA fit — pointed along `continueDeg` (the "keep going straight" direction), and
  that "continue straight, no turn" angle essentially never collides with anything, so the
  search locked onto it FIRST regardless of tailMode. That silently flattened `'perp'` back
  onto the duplex axis (verified: literally byte-identical output to `'flatten'`) and
  distorted `'L'`'s corner into a smeared partial bend. Separately, the same function's "one
  bond straight, then turn" assumption is hard-coded to start from local index 0 — correct
  for a 3′ flank (attach at index 0) but backwards for a 5′ flank (attach at the LAST local
  index): the corner ended up at the free tip instead of at the duplex. Fixed by (a) running
  `computeLayout` on the reversed local sequence for a far-end attach and reversing the
  coordinates back, so "straight bond then turn" is always oriented from the true attach
  point, and (b) for a flank with no surviving structure at all under `'perp'`/`'L'`,
  building explicit candidate target angles (`continueDeg±90` for perp; `continueDeg` with
  the corner mirrored either way for L) using the flank's own first-bond direction as the
  alignment reference instead of PCA, then letting the existing collision-avoidance search
  pick whichever candidate clears the rest of the drawing. `attachFlank` gained an optional
  `refDegOverride` param for this; the untouched PCA path still runs for flanks that DO have
  real surviving structure (unchanged from before). Verified visually via a Playwright
  screenshot of the exact reported sequences/structures, and with `test_complex_tailmode.js`
  covering both Complex modes and both flank-attach orientations.
- **Bracket support (user request #3):** `parseStructure` now accepts `[ ]` and `{ }` (also
  `< >`, already scaffolded in the unused `OPEN`/`CLOSE` maps) as pair brackets alongside
  `( )`. They must still nest properly with everything else — this is NOT crossing-pair
  pseudoknot support, just alternate bracket characters so a non-WC pair doesn't have to be
  hand-edited into `()`. Mismatched bracket types (e.g. `[` closed by `)`) now get a
  specific error instead of a generic "unbalanced" one.
- **Differentiate non-WC pairs (user request #5):** any pair written with `[ ]`/`{ }`/`< >`
  is tagged in a new `nonWC[]` array (parallel to `pt[]`, true at the pair's lower index)
  and drawn as a **dashed** rung instead of solid, in both the canvas render and the
  single-sequence SVG export. **Not yet wired into Complex-mode rendering/export**
  (`renderComplex`, `buildComplexExportSVG`) — those still draw every rung solid regardless
  of bracket type. Doing that properly means threading `nonWC` through
  `breakForDuplex`/`sliceFlankPT`/`attachFlankContinuing` (index-preserving, so it's
  mechanical, just untouched so far) and both complex draw paths' `drawStrandBonds`/
  `strandBonds` helpers. **✅ Done 2026-10-02**: fell out of the `strands()` refactor —
  `CM.nonWCA/nonWCB` carry each strand's original flags, which stay index-aligned because
  `breakForDuplex` only clears `pt[]`.
- **Numbering (user request #2, "move the nt numbering")** — first built (2026-09-30) as a
  "Start numbering at" offset field; **that field was REMOVED 2026-10-07** at the requester's
  ask. What was actually wanted was labels that don't sit on bases: see
  `placeNumberLabels()` — each label is placed beside its base, in the direction away from the
  base's bonded neighbours/partner, trying 24 directions × 3 distances until it clears every
  base circle, bond/backbone line and earlier label (thin leader line if pushed past the first
  ring). Runs in display coords, so it follows rotation/drags; used by both canvas and export.
  A stale `numOffset` value in old saved styles is simply ignored.

**Status of the remaining items** (updated 2026-10-02 — see §13 for what shipped):

- ✅ **DONE 2026-10-02 — Modular parts (internal request #1).** Implemented as described
  in §6 (derived parts, Ctrl/⌘+drag, Ctrl/⌘+click, Shift+drag marquee, drag-selection).
  Original notes kept for context: grouping the structure into paired-stem /
  ring / free-end "parts" as first-class objects, `Ctrl+drag` to move a whole part, and
  drag-rectangle multi-select. Current drag model (see §6) already moves "a base + its
  whole paired branch" via `pt[]` recursion, which covers part of (a) implicitly, but there
  is no marquee-select and no `Ctrl` modifier binding today (only `Alt` for
  single-base-not-branch). A real implementation needs: (1) a lightweight "part" concept
  computed from `pt[]`/loop structure rather than a stored data model (keep it derived, not
  synced state, to avoid a second source of truth vs. `pt[]`), (2) a rubber-band select
  rectangle over base hit-targets, (3) `Ctrl+mousedown` on any base within a part starting
  a whole-part drag. Worth a dedicated pass with its own tests given how central dragging
  already is.
- ✅ **DONE 2026-10-02 — Rotate a selected part by a specified degree (internal request
  #2).** See §6 "Rotate selection". Original notes: the existing
  **Rotate** control (Layout tab) rotates the WHOLE structure around its centroid; there's
  no per-selection rotate today. Needs: a pivot choice (selection centroid vs. the part's
  attachment point — the latter reads more natural for "rotate this stem"), a numeric
  degree input, and applying it as an `offX`/`offY` delta per selected base (same channel
  drag already uses) so it composes with existing manual offsets instead of fighting them.
- **DMS/mutational-profile color upload (user request #1)** — importing a `color.txt` (or
  similar) to set per-base letter/circle color from external data. The per-base coloring
  plumbing already exists (`M.color[]`, `M.circleColor{}`) — this is really a small file-
  parsing + mapping feature (position → color, by whatever convention: hex per line, a
  value mapped through a scale, etc.) rather than a rendering change. Needs one thing from
  the user before building: **the exact file format** they actually export from their DMS
  pipeline (one color/value per position? a header? numeric reactivity mapped through a
  color scale, or literal colors?) — building the wrong parser is wasted work here.
- **Varna-style whole-helix dragging (user request #4)** — AptaRender's drag already moves
  a base's whole paired branch together (README's "drag-to-reposition" section), which is
  the same idea Varna's screenshot shows (drag one strand of a helix, the loop it encloses
  moves as a rigid unit). If this is still being requested after that existing behavior,
  it likely means something more specific — e.g. dragging a helix so it visually detaches
  and reflows as a "part" the way §"Modular parts" above describes (a helix as its own
  draggable unit independent of being nested under a bigger structure), or a rotate-while-
  dragging gesture. Worth clarifying with the requester against the current build before
  assuming which gap they mean.
- ✅ **DONE 2026-10-02 — Zoom in for selected base (user request #6).** See §6. Original
  notes: likely a "center + zoom
  canvas on `M.selected`" button using the existing `M.zoom`/`M.panX`/`M.panY` view state
  (no new data model needed, just a compute-bounding-box-of-selection-then-set-view
  helper). Lowest-risk item on this list if picked up next.

**Still open — need input from the requester before building:**
- **DMS/mutational-profile color upload (user request #1)** — blocked on the file format
  (see above).
- **Varna-style whole-helix dragging (user request #4)** — plain drag (branch) and
  Ctrl/⌘+drag (helix only) now cover both readings described above; confirm with the
  requester whether either matches, or whether they want rotate-while-dragging.
- **Per-label numbering drag** — only if "move the nt numbering" meant this (see above).

## 13. 2026-10-02 audit: redundancy & hard-coded logic

An audit of `index.html` for duplicated code and hard-coded logic that interfered with other
features. Every bug below was reproduced on the pre-audit file with a jsdom probe first,
then confirmed fixed.

**Structural refactor.**
- `render()`/`renderComplex()`, `buildExportSVG()`/`buildComplexExportSVG()`,
  `fit()`/`fitComplex()`, `contentSize()`/`complexContentSize()` and the
  `displayCoords`/`complexDisplayCoords` pairs were hand-copied twins, and had drifted:
  complex mode had no numbering and no dashed non-WC rungs. Replaced by `strands()` +
  `buildScene()` (one scene for canvas and export). The single-sequence SVG export was
  verified line-for-line identical to the pre-refactor output; the complex export differs
  only by the numbering labels it now gets.
- Selection, color-by-selection, select-all/clear, reset fills, reset positions, palette
  and monochrome each had an `if(CM.active){…A…B…} else {…M…}` branch — now one loop over
  `strands()`.
- Complex mode → layout dispatch was copied in three places → `layoutComplex()`. Region
  validation copied in both layout functions → `validateRegion()`.
- `#c8cdd6`, selection-ring / highlight / numbering colors were repeated literals →
  `FALLBACK_COLOR`, `SEL_RING`, `BOND_HI`, `NUM_COLOR`.
- Dead code removed: `pk`/`M.pkPairs`, `branchIndices`, `lum`, `applyPaletteNoRender`,
  no-op `oninput` handlers. `history` renamed `histStack` (it shadowed `window.history`).

**Bugs fixed.**
1. **Side-by-side complexes reloaded as Replace** — `persist()` didn't save `mode`.
2. **Style edits lost on reload in uniform mode (the default)** — the global style was
   only stored per-doc, and per-doc styles are ignored when uniform is on.
3. **Undo didn't revert style changes in uniform mode** — same root cause; snapshots now
   carry the global style. Fixed 1–3 by a single `serializeState()` used for both storage
   and undo.
4. **Sliders and color pickers never recorded undo steps or persisted** (font, rotation,
   frame width, all color pickers, numbering) → `liveControl()`.
5. **Complex-mode color edits could be overwritten** — Build, Add, Import and switching
   docs called `saveActive()` while a complex was open, writing the stale single-sequence
   `M` over the doc the complex had just recolored. `saveActive()` is now mode-aware.
6. **No way back from a complex to the same sequence's single view** — clicking the active
   doc was a no-op. Now it returns.
7. **Ctrl+Z ignored right after using a slider/color picker** — the keyboard guard skipped
   every `<input>`, not just text fields.
8. Render button / Clear while a complex was open edited the hidden single view; they now
   leave the complex first. Window resize / panel resize didn't redraw an open complex.
9. Strand-picker options were built with `innerHTML` from user-entered sequence names (an
   HTML-injection path via an imported FASTA header) → built with `textContent`.

**Tests** (scratch, not committed — recreate from §9's recipe): `tails.js`/`check.js`/
`test_tails.js`/`check2.js` (2026-10-06: every tail mode × single hairpin with 1, 2,
unequal and 20+20-nt tails, 2-/3-stem and wide-loop structures; asserts no two bases closer
than 0.85·S, all backbone steps = S, distinct shapes per mode), `probe.js` (bugs 1–3 +
export snapshot), `test_features.js` (parts, Ctrl+drag, selection drag, rotation rigidity
and pivot, zoom-to-selection centring, marquee, complex dashes/numbering, bug 5–7).

**Known limitations left as-is.**
- Changing tail mode re-lays out but doesn't renormalize manual offsets (both modes).
- Rotating a subset changes the whole-structure centroid, so when the view rotation is
  non-zero the drawing can shift slightly on screen; at 0° there's no shift.
- Per-bond click-recolor is still single-sequence only (complex strands have
  `selBond:null`); intramolecular bond colors set in single mode do carry over.
