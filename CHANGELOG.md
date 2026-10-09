# Changelog

All notable changes to AptaRender. Newest first.

## 2026-10-07
- **Varna-style ring rotation** (Layout tab → **Lock ring**). Ctrl/⌘+click an unpaired base of a
  loop, click **Lock ring**, then drag any helix attached to that ring to swing it around the
  ring (or drag the closing stem to swing the rest of the molecule). The ring keeps its size;
  its unpaired bases re-spread evenly between the helices. Helices can't slide past each other,
  so nothing overlaps. **Esc** or **Unlock ring** to finish. Single-sequence view.
- **Color from file** (Letters tab): upload a .txt of `position [base] color [target]` lines
  (ranges like `5-9`, hex or named colors, `A:`/`B:` strand prefixes in a complex) to color
  letters, circle fills or both. **Download sample** writes a ready-to-edit file for the open
  structure, colored by part (stems / loops / free ends). Skipped lines and base-letter
  mismatches are reported.
- **Size of selected bases** (Letters tab): enlarge or shrink just the selected bases (50–300%,
  letter and circle together), with **Reset all sizes**. Saved, undoable, and exported. The
  font-size slider (now labeled "all bases") still scales everything.
- **Position numbers no longer overlap bases.** Each number is now placed beside its base in
  open space — on the outer side of a stem strand, outside a loop, past the end of a tail —
  checked against every base circle, line and other number. If a spot right next to the base
  isn't free, the number moves a little further out with a thin leader line back to its base.
  Follows rotation and dragging, and exports the same way.
- **Removed the "Start numbering at" offset field** (Lines tab). Numbers always start at 1.

## 2026-10-06
- **Fixed — Free tails options not working on multi-stem structures:** with two or more
  separate stems, the layout doubled back on itself so the 5′ and 3′ tails were drawn
  exactly on top of each other (and on top of a stem rail). Every tail style started from
  those overlapping positions, so none of them looked right. Structures with 2+ stems now
  use an open baseline: the backbone runs left to right and each stem stands up from it,
  spaced so loops and tails never overlap. Single-hairpin layouts are unchanged.
- **Natural tails now curve inward, shaped like a loop.** On a single hairpin, the 5′ and
  3′ tails are drawn around an open circle at the base of the stem, built the same way as the
  hairpin loop, so they curve toward each other and mirror the loop at the tip. At least two
  empty positions are always left between the two free ends, so they never meet or overlap,
  and the circle grows with tail length. On structures with 2+ stems, each tail curls inward
  toward the stems with a loop's curvature, opening up only as much as needed to clear the
  stems and the other tail. Option relabeled "Natural (curves inward)". L-shape and
  Perpendicular still turn outward, as labeled.
- Changing the tail style now clears any manual drags on the free-tail bases, so the new
  style shows cleanly (drags on stems and loops are kept).

## 2026-10-02
- **Box select & parts:** Shift+drag on the background draws a selection box.
  Ctrl/⌘+click a base to select its *part* — the helix it belongs to, the loop ring it sits
  in, or its free 5′/3′ end — and Ctrl/⌘+drag to move just that part. Dragging any selected
  base now moves the whole selection.
- **Rotate selection** (Layout tab): rotate the selected bases by a set number of degrees,
  either about the point where they attach to the rest of the structure (e.g. a stem's base
  pair) or about their own center. Undoable.
- **Zoom to selection:** new **Zoom** button in the selection bar centers and zooms the
  canvas on the selected bases.
- **Complex view now shows position numbering and dashed non-WC rungs**, same as a single
  sequence.
- **Fixed:** side-by-side complexes turned back into Replace complexes after a reload.
- **Fixed:** with "Uniform style" on (the default), style changes weren't saved across
  reloads and couldn't be undone.
- **Fixed:** font size, rotation, frame width, numbering and every color picker now record
  an undo step and are saved.
- **Fixed:** recoloring bases in a complex could be silently reverted after building
  another complex, adding/importing sequences, or switching sequences.
- **Fixed:** clicking the current sequence while a complex is open now returns to it.
- **Fixed:** Ctrl+Z didn't work right after moving a slider or color picker.
- Internal: the separate single-sequence and complex drawing/export code paths were merged
  into one, so the two views can't drift apart again.

## 2026-09-30
- **Fixed — "L-shape" and "Perpendicular" tail styles doing nothing in Complex mode:** a
  flank (whatever's left of a strand beyond a duplex end) was always re-oriented to
  continue straight out from the duplex before anything else was tried, no matter which
  tail style was selected — so a flank governed only by tailMode (no surviving structure of
  its own) always ended up flattened back onto the duplex axis, silently discarding any
  perpendicular turn or L-corner. Both styles now actually turn away from the duplex (with
  the existing collision-avoidance search still free to pick whichever side clears the rest
  of the drawing). Also fixed the same styles being backwards for a **5′ flank** specifically
  (the one whose duplex-adjacent base is the *last* base in that stretch, not the first) —
  the "one bond straight, then turn" shape was being built from the wrong end, putting the
  corner at the free tip instead of at the duplex. Verified on both Replace and Side-by-side
  Complex modes, and on flanks attached at either end of a duplex.
- **Fixed — long sequences overlapping into a closed circle instead of a bigger one:** the
  "Natural" tail style drew every unpaired run on a fixed-size arc, so past ~19 bases the
  arc wrapped past a full turn and later bases landed back on top of earlier ones. The arc
  now widens to keep spacing correct at any length — verified out to 300 nt with no
  overlap and exact-length backbone steps throughout. Affects both a fully-unpaired
  sequence and a long free tail hanging off a real stem.
- **Fixed — "L-shape" and "Perpendicular" tail styles doing nothing for a fully-unpaired
  sequence:** those two styles silently fell back to a plain straight line when there was
  no base pair anywhere in the structure to take a direction from. They now get their own
  distinct shape in that case too, matching how they already behaved for a tail attached to
  a real stem.
- **Non-Watson-Crick base pairs:** the structure box now accepts `[ ]` and `{ }` as pair
  brackets alongside `( )` — no more hand-editing a non-WC pair into parentheses just to
  render it. Any pair written with `[ ]`/`{ }` is drawn as a **dashed** rung instead of
  solid, so WC and non-WC pairs are visually distinct at a glance (single-sequence mode;
  not yet wired into Complex mode's rungs). Mismatched bracket types (e.g. `[` closed with
  `)`) now get a specific error message.
- **Manual numbering offset:** a new **Start numbering at** field next to **Number every**
  (Lines tab) shifts the displayed position numbers by a constant — e.g. start counting at
  101 to match a plasmid/genomic numbering convention — without touching the sequence
  itself. Persists like the other style settings.

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
