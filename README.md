# AptaRender

**Art-first, browser-based editor for 2D nucleic acid secondary-structure figures.**

You provide the sequence and its dot-bracket structure — AptaRender gives you a fully
editable 2D layout for making clean, publication-quality figures. It does **no structure
prediction**; it is purely a drawing/annotation tool for figures you already know.

### ▶ Live site: https://yjhe99.github.io/aptarender/

---

## Why

Prediction tools like mfold/UNAfold draw structures well, but they're rigid on the
aesthetic side — you can't easily resize fonts, rotate a structure while keeping the
letters upright, or color-code individual bases the way a figure needs. AptaRender
separates *drawing* from *prediction* so you have full control over how the figure looks.

## What it does

- **Input:** a sequence plus a nested dot-bracket structure — `( )` and `.`
- **Automatic radial layout** for hairpins, bulges, internal loops, and multiloops
- **Rotate** the whole structure while every letter stays upright
- **Font size** control (independent of the layout spacing)
- **Per-base letter color:** by A/U/G/C identity palette, a single monochrome color, or by hand-selecting bases
- **Base circles:** independent frame and fill — none / custom color / by base identity — plus per-base fill by selection, adjustable frame thickness
- **Per-hydrogen-bond coloring:** click any base-pair rung and recolor it
- **Free 5′/3′ tail layouts:** natural (curved), flatten, L-shape, or perpendicular
- **Backbone / rung colors, position numbering, custom canvas background**
- **Export:** SVG (fully editable — every base is its own text element) and PNG at a chosen scale, with an optional transparent background
- Your sequence and structure are **saved locally** and restored when you return

## How to use

1. Paste your **sequence** and its **dot-bracket structure** (or click *Hairpin* / *Multiloop* for a demo).
2. Click **Render**.
3. Style it with the panel on the left — colors, fonts, rotation, tails, circles.
4. **Export** as SVG or PNG.

## Updates

See [CHANGELOG.md](CHANGELOG.md) for the full version history.

## Feedback

Still under development — built mainly for my own figures, but I'd love to know whether
it's useful to other nucleic acid researchers too. Suggestions are very welcome.

Built by **Janet He** with Claude · [yjhe@utexas.edu](mailto:yjhe@utexas.edu?subject=AptaRender%20feedback)
