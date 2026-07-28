# Petal Catch

A pixel-art catch game rendered inside a handheld-console shell. One self-contained
HTML file, no build step, no dependencies, works offline from `file://`.

The player moves left/right and catches flowers (or uploaded faces) in a basket.
Miss three and the run ends.

## Running it

Open `petal-catch.html` directly in a browser, or use VS Code's Live Server /
`npx serve` for a local server. A server isn't required today, but it will be if
assets get split out of the HTML (see "Suggested first task").

## Files

| File | What it is |
|---|---|
| `petal-catch.html` | The game. Everything lives here. |
| `petal-catch-yellow-shell.html` | Identical, but with the original flat-yellow console shell instead of the rainbow gradient. Kept as a revert point — don't edit it, just copy over from it if the rainbow gets dropped. |

## Architecture

One IIFE at the bottom of the file, split into numbered sections. Read the section
headers before hunting for anything.

```
0  CONSTANTS          frame/screen dimensions, sprite geometry, palette
1  CANVAS             ctx setup, makeCanvas() offscreen helper
2  RESPONSIVE         fit() — scale factor + grain-density compensation
3  BITMAP FONT        3x5 glyphs drawn as fillRects
4  ITEM SPRITES       flower pixel maps -> baked canvases; deck() picks the pool
4b PHOTO -> HEAD      oval crop, framing UI, palette snap, outline
5  BACKGROUND         scene baked once to an offscreen canvas; clouds drift live
6  CHARACTER          the two embedded sprite layers
7  GAME STATE         player, items, settled, puffs, pops, score, lives
8  INPUT              keyboard / D-pad / drag all write to one `input` object
9  SCREEN SWITCHING   overlay state machine + the faces picker wiring
10 UPDATE             movement, spawning, catch + miss detection
11 DRAW               render order (see below)
12 LOOP               rAF with delta time
```

The DOM is the console: `.shell` is the plastic body, `.plate` the title bar,
`.screen` holds the canvas plus the overlay screens, `.controls` holds the D-pad
and buttons. The start / game-over / faces / framing screens are **DOM overlays**
positioned over the canvas, not canvas-drawn — so their text stays crisp at any
scale and they're normal HTML to edit.

## Measured constants — do not guess these

These came from pixel-inspecting the actual sprite exports. Changing the art means
re-measuring, not estimating.

```js
SPR_W = 50, SPR_H = 92   // character sprite, native pixels
FRONT_Y = 53             // where the basket-front layer sits on the body
CATCH = { ox:4, oy:46, w:42, h:13 }   // hitbox, relative to sprite origin
```

The three gribble layer exports all sat on the same 100x100 canvas spanning
x 25-75, so cropping them with one shared box left them aligned at (0,0). If new
layers get exported, keep that shared-canvas property or the alignment breaks.

Press **`** (backtick) in game to toggle a debug overlay drawing the hitbox in red
and the miss line in blue. Use it whenever catch feel is off — don't tune by eye.

## Gotchas

**Render order is the whole depth illusion.** In `render()`:

```
background -> body -> falling items -> settled items -> basket front
```

The basket front is a separate 50x22 sprite drawn last, so anything in the basket
sinks behind the real woven silhouette. Reordering these breaks the effect. An
earlier version clipped a rectangle out of the full sprite instead — that was
wrong, because the rim is curved.

**Integer scaling.** `fit()` floors the scale factor once it's >= 2. Fractional
scaling makes pixel art shimmer. Don't make the canvas fluid; scale the whole
console as one unit.

**Grain density.** The shell's SVG noise is tiled at `--gs: 46/scale` because
`transform: scale()` scales background images too. Without that compensation the
grain renders 2-3x too coarse.

**Canvas path state.** `beginPath()` wipes any subpath already queued. A helper
that calls it internally will silently destroy a rect you were about to subtract
from. This caused an inverted oval mask once — the fix was clipping and drawing
twice instead of an even-odd fill.

**Photo downscaling.** In `buildOvalSprite()`, `imageSmoothingEnabled` is **true**
for the shrink so each output pixel averages the photo. Everywhere else it's
false. Nearest-neighbour on a photo samples one arbitrary pixel per cell and looks
like noise. Turn smoothing off *after* the shrink, for drawing.

**Sprite sizes vary.** Every falling thing is `{ c, w, h, tint }` and items carry
the object directly. Flowers are 8x8, faces 16x20. Never hardcode item dimensions;
use `sp.w` / `sp.h`.

## Palette

Everything is ENDESGA 32 — that's the palette the character was drawn on. `P` holds
the named subset in use; `PAL32` holds all 32 as RGB triples for snapping uploaded
photos. New colours should come from that set, not be invented.

## How to add things

**A button on the console.** Add it inside `.controls` in the HTML, style it near
`.abtn` in the CSS, wire a listener in section 9. The control strip is 66 logical
px tall starting at `top:346px` and is absolutely positioned — check there's room,
or grow `FRAME_H` (currently 412) and the `.controls` offset together.

**A new falling item.** Add a pixel map next to `SHAPE_A` / `SHAPE_B`, add an entry
to `FLOWERS`, done — `bake()` and `deck()` pick it up automatically.

**A new screen.** Copy the `.view` pattern: a div inside `.screen`, add its id to
`show()`, extend the `overlay` state (`null | 'faces' | 'crop'`).

**Sound.** There's none yet. Web Audio oscillators suit the aesthetic better than
sample files and keep the single-file property.

## Suggested first task

The three base64 blobs (character, basket front, shell gradient) total ~7KB of
unreadable string sitting in the middle of the file. They cost context on every
read and make diffs noisy.

Worth extracting into `assets.js` as named exports, leaving the HTML readable.
That needs a local server, since ES modules don't load over `file://`. If the
offline-single-file property matters more, leave them alone.
