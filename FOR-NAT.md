# FOR-NAT.md — Line Warp

Hey future Nat. Here's how this thing works, why it works, and what you'd want to know if you ever need to touch it again.

---

## What It Is

Line Warp is a single HTML file that takes a letter, renders it as a blurred heightmap, and uses that heightmap to vertically displace a stack of horizontal bands. The result looks like those classic striped logos where lines warp around a shape. You can tweak everything with sliders and export as PNG or SVG.

One file. No build step. No dependencies. Open it in a browser and go.

## The Architecture (It's Just a Canvas)

The whole thing is structured as a sidebar + canvas layout:

```
┌──────────┬──────────────────────┐
│ Controls │                      │
│ (280px)  │   Canvas (800x800)   │
│          │                      │
│ Sliders  │   Displaced lines    │
│ Buttons  │                      │
└──────────┴──────────────────────┘
```

On mobile (<700px), the sidebar stacks on top of the canvas. That's just a `flex-direction: column` media query.

### The Rendering Pipeline

Every time a slider moves, this happens:

1. **Render the letter** onto an offscreen canvas (white background, black letter)
2. **Blur it** using the browser's built-in `CanvasRenderingContext2D.filter = 'blur(Xpx)'` — this turns the hard-edged letter into a smooth dome-shaped heightmap
3. **Optionally blend in a ridge map** (distance transform) for sharper edges
4. **Read the pixel data** — each pixel's brightness = height at that point
5. **Draw displaced bands** — for each horizontal band, walk pixel-by-pixel across the top and bottom edges, look up the height at that position, and shift the y-coordinate up by `height * maxDisplacement`

The key insight: the letter never appears directly in the output. It's only used as a displacement source. The output is entirely made of horizontal bands that have been warped.

### The Heightmap

The heightmap is the core trick. When you render black text on white and blur it heavily, you get something like a topographic map — high in the middle of the letter, falling off to zero at the edges. The more blur, the smoother and wider the displacement field.

The `getHeight()` function samples this map:

```javascript
function getHeight(data, x, y) {
  return (255 - data[(iy * SIZE + ix) * 4]) / 255;
}
```

It inverts the value (black letter = 255 brightness after inversion = max height) and normalizes to 0-1. This height value gets multiplied by `maxDisplacement` to produce the actual pixel offset.

### The Ridge Effect (Distance Transform)

The default blurred heightmap creates smooth, rounded displacement. But sometimes you want sharper, more defined edges — like the letter is actually embossed into the surface rather than pushed through a rubber sheet.

That's what the Ridge slider does. It blends in a second heightmap based on a **distance transform** — for each pixel inside the letter, calculate how far it is from the nearest edge. Pixels deep inside the letter get high values, pixels near the edge get low values, creating a sharp ridge along the letter outline.

The distance transform uses a **chamfer approximation** — a two-pass algorithm (forward and backward) that's much faster than computing true Euclidean distances:

```
Forward pass: scan top-left to bottom-right
  For each pixel, compare with neighbors above and to the left
  Take the minimum distance

Backward pass: scan bottom-right to top-left
  For each pixel, compare with neighbors below and to the right
  Take the minimum distance
```

This gives an approximate Euclidean distance that's good enough for visual purposes. The 1.414 multiplier for diagonal neighbors approximates √2.

### Per-Line Rounded Corners

Each band has individually rounded corners using quadratic Bezier curves. The tricky part: the corner radius needs to work *with* the displacement. So the control points are placed at displaced positions, not flat positions.

The drawing order for each band:
1. Start at left edge, below top-left corner
2. Top-left corner arc (quadratic curve)
3. Walk the top edge left→right (pixel by pixel)
4. Top-right corner arc
5. Right edge straight down
6. Bottom-right corner arc
7. Walk the bottom edge right→left
8. Bottom-left corner arc
9. Close path

### SVG Export

The SVG export mirrors the canvas rendering logic exactly, but writes SVG path data instead of drawing to a canvas. Each band becomes a `<path>` element with moveTo, lineTo, and quadratic curve commands. The step size is 2px instead of 1px to keep file sizes reasonable without visible quality loss.

## How the Pieces Connect

```
User tweaks slider
       ↓
input/change event fires
       ↓
updateLabels() — update the numeric readouts
render() — redraw everything
       ↓
getParams() — read all control values
createDisplacementMap() — generate the heightmap
       ↓
For each band:
  Walk top edge, sample heightmap, offset y
  Walk bottom edge, same
  Fill the path
```

All controls use both `input` (fires continuously while dragging) and `change` events, wired up in a single loop over `controls`. This means the canvas updates in real-time as you drag any slider.

## Hosting Setup

The tool lives in two places:

1. **GitHub repo** (`nmemodi/line-warp`) — the source of truth, open source, MIT licensed
2. **natemodi.com** (`public/line-warp/index.html`) — the live demo, served as a static file by Astro

The Astro site serves anything in `public/` as-is — no build processing, no templating. So the HTML file works identically in both locations. If you update the tool, remember to copy `index.html` to both places.

The nav dropdown on natemodi.com uses CSS-only hover (`:hover` on `.nav-dropdown` shows `.nav-dropdown-menu`). No JavaScript needed for simple dropdowns.

## Technologies and Why

- **Raw Canvas API** — No libraries because (a) the rendering is simple enough to do directly, and (b) keeping it dependency-free means the tool is truly one file you can open anywhere
- **CSS `filter: blur()`** — Instead of implementing Gaussian blur in JavaScript (which would be slow), we use the browser's GPU-accelerated blur on an offscreen canvas. This is the same trick game developers use: render to a texture, apply a post-process effect
- **Chamfer distance transform** — A simple, fast approximation to Euclidean distance transform. Two passes, O(n) per pass, works great for this use case
- **Inline base64 favicon** — To keep the tool as a single file with no external dependencies, the favicon is a data URI. It was generated by actually running the tool and capturing a 64x64 canvas

## Lessons and Gotchas

### The blur matters more than you'd think

The blur radius is the single most important parameter for how the output looks. Too little blur and you get hard displacement edges that look bad. Too much and the letter shape gets lost. The default (28px) works for most letters, but letters with thin features (like 'I' or 'L') need less blur.

### Canvas `filter` works on draw operations, not existing content

When you do `ctx.filter = 'blur(28px)'; ctx.drawImage(source, 0, 0)`, it blurs the *source* as it's drawn. You can't blur content that's already on the canvas. That's why the code creates a separate offscreen canvas for blurring — render the letter on canvas A, then draw canvas A onto canvas B with the blur filter active.

### Text rendering is font-dependent

The exact output depends heavily on what fonts are available on the user's system. Helvetica Neue renders differently on macOS vs Windows. There's no good fix for this — web fonts would add a dependency, and the tool is about the *shape* of displacement, not pixel-perfect reproduction.

### Distance transform edges need smoothing

The chamfer distance transform produces visible pixelation artifacts at the edges. That's why the code applies a light blur (`max(3, blur * 0.15)px`) to the distance field before blending it with the smooth heightmap. Without this, the ridge effect looks jagged.

### Offset X/Y applies to the letter, not the lines

The offset controls shift where the letter is rendered on the heightmap canvas. They're applied as arguments to `fillText()`: `g.fillText(p.letter, p.offsetX, p.offsetY)`. The lines don't move — they just respond to the shifted heightmap.

### Don't forget: the heightmap is inverted

Black text on white background → the text area has pixel value 0. But we want the text area to have maximum height. So `getHeight()` inverts: `(255 - pixelValue) / 255`. This catches people every time they look at the code.

### SVG paths can get huge

With 800px width and 1px step size, each band would be ~1600 lineTo commands × 2 edges = 3200 path segments × 13 bands = 41,600 path segments. Using 2px steps in SVG export cuts this in half with no visible difference.

## How Good Engineers Think About This

The core lesson here is **separation of concerns in a creative tool**: the displacement map is completely independent of the line rendering. You could swap in any heightmap source — an image, a gradient, procedural noise — and the line rendering code wouldn't change. The heightmap is the interface between "what shape do you want" and "how do we draw it."

Another pattern worth noting: using the browser as your rendering engine. Instead of implementing blur from scratch, we lean on `CanvasRenderingContext2D.filter`. Instead of computing font metrics, we render text and read the pixels. The browser already has fast, well-tested implementations of these things — use them.

Finally: single-file tools have a distribution advantage that can't be overstated. No npm install, no build step, no CORS issues, no hosting requirements. Email the HTML file. Open it from your desktop. Put it on a USB drive. The simplicity of deployment *is* a feature.
