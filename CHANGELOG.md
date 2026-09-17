# Changelog

All notable changes to AptaRender. Newest first.

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
