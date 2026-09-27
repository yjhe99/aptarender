# Changelog

All notable changes to AptaRender. Newest first.

## 2026-09-27
- **Two-strand hybridization ("Build a Complex", Feature #4):** display two sequences on
  one canvas and manually set a partial hetero-duplex region between them. New **Complex**
  tab: pick Strand A and Strand B (from your saved sequences), enter the A range and B
  range that hybridize, choose a mode, and click **Build complex**.
  - **Replace mode:** draws the hybridizing region as a fresh straight duplex ladder.
    Any intramolecular base pair in *either* strand that touches the hybridized bases is
    broken; each strand's surviving structure (if any) is re-attached as a flank off its
    own end of the duplex.
  - **Side-by-side mode:** Strand A's own structure and layout are left completely
    untouched — it's used as a fixed reference. Strand B is redrawn alongside it, with the
    hybridizing bases placed at a perpendicular offset from A's real coordinates. Only
    Strand B's own pairs that straddle the hybridized region are broken; Strand A is never
    modified. This offset is now computed dynamically — it scans the rest of Strand A's
    structure in that same stretch and pushes Strand B out just far enough to clear it,
    instead of a fixed distance that could land on top of another part of Strand A.
  - A saved complex appears in its own list, can be reopened, deleted, and exports to
    SVG/PNG like a normal sequence. The hetero-duplex rungs get their own color
    (**Duplex line** picker).
- **Drag-to-reposition:** click and drag any base to move it — and its whole paired
  branch — to a new position; the rest of the structure stays anchored. Hold **Alt**
  while dragging to move just that one base instead of its whole branch. Works in both
  single-sequence and Complex views.
- **Undo / Redo:** **Ctrl+Z** (Cmd+Z on Mac) to undo, **Ctrl+Y** or **Ctrl+Shift+Z** to
  redo. Covers rendering, styling, dragging, and sequence management — up to 100 steps.
- **Resizable control panel:** drag the thin divider between the panel and the canvas to
  widen or narrow it. Your chosen width is remembered.
- **Rendering fix — circle vs. straight line:** a run of unpaired bases now only draws as
  a circle/arc when it's actually *enclosed* by a matching `(`…`)` pair (a real hairpin,
  internal loop, or multiloop). Anything not enclosed by a pair — free 5′/3′ tails,
  linkers between separate stems, or an entire strand with no pairs at all — now draws as
  a straight line instead. Previously an unpaired strand (or a free tail, under "Natural"
  tail mode) could render as a full circle, which didn't make geometric sense, especially
  next to a real duplex in Complex mode.
  - As part of this, a top-level stem's two strands now continue straight off the
    backbone in the direction it was already heading, instead of turning 90°.
  - The **Natural** tail-mode option is relabeled "Natural (default layout)" — it no
    longer curves free tails; they're straight by default now, matching the fix above.

## 2026-09-16
- **Multiple sequences:** add sequences manually or import a FASTA / seq+structure file, switch between them, rename/delete, with a cap of 20. Each sequence keeps its own structure, colors, and view; a "Uniform style across all" toggle shares one look across every sequence or lets each keep its own. State persists in `localStorage` (`aptarender_docs`).
- **Reorganized control panel** into a tabbed top navigator (Input · Layout · Letters · Circles · Lines · Export) with a shared base-selection bar.
- **Fixed L-shape / perpendicular tail direction:** the 90° turn is now derived from the base-pair rung (the anchor base's own strand side) instead of a centroid test that went degenerate when a tail points radially outward — tails no longer fold back across the structure.

## 2026-09-14
- Added a **Clear** button that wipes the current input and its locally-saved copy. Saved input lives only in your own browser (`localStorage`); it is never uploaded and other visitors never see it.

## 2026-09-10
- Per-base **circle-fill color by selection** (mirrors the letter-color workflow) + *Reset fills*.
- **Input now persists** across page reloads (previously the demo example could overwrite your typed structure).
- Contact / credit line added.

## 2026-09-09
- Initial public release: parser, radial auto-layout, rotation with upright text, base-identity & selection coloring, per-H-bond coloring, base circles (frame + fill), adjustable frame thickness, four free-tail layouts, numbering, transparent-background option, and SVG/PNG export.
