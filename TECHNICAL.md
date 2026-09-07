# TECHNICAL.md — how `ascii-city.html` works

A from-scratch technical reference to the renderer, written to teach the math and the trick
behind each subsystem rather than to log decisions. This document explains *what the mechanism
is and how to rebuild it*. Every claim below cites a function name and an approximate line
number in `ascii-city.html` — that refers to the fuller, commented development source (not
`index.html` in this repo, which is a stripped release build with different line numbers); if
a citation looks wrong against `index.html`, grep for the function name, which is unchanged
between the two.

Each chapter has two layers. **The idea** is plain language, no code, for a reader who wants
to understand what problem is being solved. **The mechanism** is the actual formula or
algorithm, for a reader who wants to reimplement it.

---

## 1. Determinism & the seeded RNG

### The idea

Everything in this city — where the streets run, how tall each building is, which windows
are lit, when a signal turns red — comes from a single number: the seed. Feed the same seed
in twice and you get the same city, the same traffic pattern, the same flickering sign, down
to the byte. This matters for two practical reasons: a bug report can be reproduced exactly by
quoting a seed, and every measurement in this project's own test suite (a run count, a digest,
a "did this change move anything") is only meaningful if nothing *except* the change under
test can move the numbers. A single stray call to the platform's ordinary random-number
generator anywhere in the codebase would make every seed a lie.

The generator used is `mulberry32`, a well-known small, fast, public-domain PRNG. It is not
cryptographically strong and does not need to be — its only job is to produce a long,
well-distributed sequence of numbers from one 32-bit integer, deterministically.

### The mechanism

```js
function mulberry32(a) {
  return function () {
    a |= 0; a = a + 0x6D2B79F5 | 0;
    let t = Math.imul(a ^ a >>> 15, 1 | a);
    t = t + Math.imul(t ^ t >>> 7, 61 | t) ^ t;
    return ((t ^ t >>> 14) >>> 0) / 4294967296;
  };
}
```
(`ascii-city.html` ~386–393.) Called once per generation with the seed as `a`; the returned
closure is the `rnd()` used throughout world generation (lots, massing, sign text, prop
placement). The seed itself comes from the URL hash (`#seed=12345`) or a default, read by
`hashSeedFromHash()` immediately below it (~394–397) — never from `Math.random()`.

**The invariant:** `Math.random()` must not appear anywhere in the codebase. A second, separate
seeded stream (`mulberry32` reseeded independently) is used for audio noise buffers, kept apart
from the generation stream precisely so that adding a new generation-time random draw doesn't
silently reseed audio, and vice versa.

**What breaks it, and why it's subtle.** The danger isn't a rogue `Math.random()` call (that's
easy to grep for) — it's inserting a *new consumer of the same seeded stream* in the middle of
an existing sequence of draws. `rnd()` is a stream, not a table: every call advances it by
exactly one step, so a new `rnd()` call inserted before an existing one shifts every draw after
it, changing every building's height, seed, and placement downstream even though the code
"looks" unrelated. This actually happened once in this project's history — inserting a
sign-band consumer moved the tree count from 860 to 669 — which is why several later features
(door placement, address derivation, the material selector, the filesystem-city mapping)
deliberately draw from `hash3(...)` (a stateless hash of explicit inputs, ~5158) instead of
`rnd()` wherever they need per-object randomness that must not perturb generation. The rule
that falls out: **if a feature doesn't need to change the shape of the city, it must not touch
`rnd()`.** The observable proof this rule is being followed is a "world digest" — a checksum of
`cellHeight`/`cellType`/etc. — that a test suite asserts is unchanged across such a feature's
on/off switch.

---

## 2. The projection formula & the height-span raycaster

This is the centerpiece: the single formula every visible thing in the city is drawn through,
and the traversal algorithm that decides what's visible at all.

### The idea

The renderer draws one vertical strip of the screen per character *column* — 192 of them,
left to right — by firing one imaginary ray per column out from the camera, in the horizontal
plane, using classic 2D grid raycasting (the "Wolfenstein" trick: the world is a grid of solid
or empty cells, and DDA — Digital Differential Analysis — walks the ray from grid-line to
grid-line rather than in small fixed steps, so it never overshoots or undershoots a cell
boundary and costs one arithmetic step per cell crossed, not per pixel).

The interesting design decision isn't the ray-marching — it's *what happens when the ray hits
something*. A naive raycaster stops at the very first solid cell each ray touches and draws
only that. That fails as soon as a tall building stands behind a short one: the short building
blocks the ray, so the tall building's upper floors — which should be visible rising above the
short one — never even get considered, and the screen shows empty sky where a skyscraper
should be. This was caught early, in a throwaway pre-production spike, and it is why this
renderer instead lets every ray keep traveling *past* buildings it has already drawn, stacking
up "spans" of wall as it goes, stopping only when the column is either fully covered by
what's already drawn, or a hard distance/count limit is reached.

The trick that makes this cheap is a single fact about the city: every building rises from the
same flat ground. That means a farther building can only ever poke out *above* whatever is
already covering that column on screen — never below or beside it — because a taller,
farther object's silhouette always sits closer to the horizon line than a shorter, nearer
one's. So the renderer doesn't need to track *which rows of the column are already drawn*
(an expensive per-row bookkeeping structure); it only needs to track the single topmost row
filled so far, and test each new candidate span against that one number.

### The mechanism

**The projection formula.** Given the camera's field of view and the fixed character grid,
two focal constants are derived once (and recomputed live whenever FOV or aspect changes, in
`recomputeProjection()`, ~284–288):

```js
FOCAL_COLS = COLS / (2 * Math.tan(hFov / 2));     // ≈125.11 at defaults
FOCAL_ROWS = FOCAL_COLS / CELL_ASPECT;             // ≈83.41 — CELL_ASPECT = CELL_H/CELL_W = 1.5
```

`FOCAL_ROWS` differs from `FOCAL_COLS` by exactly the character cell's aspect ratio (12px
tall / 8px wide = 1.5), because character cells are not square — a wall's vertical extent in
*rows* has to be scaled down relative to its horizontal extent in *columns* by that factor, or
the whole city stretches vertically by 50%. This is spec §9.2's single most-likely bug, and it
is why `CELL_ASPECT` is treated as a projection term rather than a rendering detail: it appears
inside the same formula that places every wall, not as a separate post-processing stretch.

Three formulas, sharing `FOCAL_ROWS` and a live `HORIZON` (§9.5 — pitch is implemented as a
y-shear of this one variable, never a true 3D rotation, so verticals never converge):

```js
const rowTopAt    = (h, d) => HORIZON - (h - CAM_Z) * FOCAL_ROWS / d;   // top row of a wall of height h at depth d
const rowBottomAt = d      => HORIZON + CAM_Z * FOCAL_ROWS / d;         // ground-contact row at depth d
const depthAtRow  = r      => CAM_Z * FOCAL_ROWS / (r - HORIZON);       // inverse of rowBottomAt
```
(`ascii-city.html` 372–376.) `depthAtRow` is used by floor casting (Chapter 3) to go the other
direction: given a screen row below the horizon, what world depth does it correspond to.

**Row spans must be computed unclamped, as floats**, and the façade `v` coordinate (which
selects what texture detail to draw) derived from those unclamped values *before* the
iteration range is clipped to the visible screen. Clamping first and deriving `v` from the
clamped numbers makes façade detail slide vertically as the player approaches a wall, because
the clamped top/bottom no longer correspond to the wall's true world extent.

**Height-span traversal — the `coverTop` trick.** `castColumns()` (~4227) runs the DDA loop per
column:

```js
let coverTop = ROWS;                 // topmost row filled so far — the ONE scalar
let lastBuilding = -1, spans = 0;
while (coverTop > 0 && depth < MAX_DEPTH && spans < MAX_SPANS) {
  // ... DDA step to next grid line, yielding perpendicular `depth` and cell (mapX,mapY) ...
  const h = cellHeight[idx];
  if (h === 0) continue;
  const b = cellBuilding[idx];
  if (b === lastBuilding) continue;      // farther face of the same building — free
  lastBuilding = b; spans++;

  const top = rowTopAt(h, depth), bot = rowBottomAt(depth);
  const yTop = Math.max(0, top);
  const yBot = Math.min(bot, coverTop);  // clipped by whatever is already nearer
  if (yBot > yTop) emitSpan(col, yTop, yBot, depth, b, side, ...);
  if (top < coverTop) coverTop = Math.max(0, top);
}
```
(Compressed from `castColumns`, ~4245–4379; `emitSpan` is the write function at ~6160.)

Two properties fall out with no extra branching:
- A farther building that is **shorter** produces `top >= coverTop`, so `yBot <= yTop` and the
  span is silently skipped — the height test alone suffices, with no separate "is this taller
  than what's already drawn" comparison needed.
- Consecutive DDA steps that land in cells belonging to the **same** building are free (`b ===
  lastBuilding`), so a long flat façade costs exactly one span regardless of how many grid
  cells it spans — the cost is proportional to the number of *distinct visible surfaces* in a
  column, not to distance traveled.

`MAX_SPANS` (8) bounds worst-case per-column cost; in practice `coverTop > 0` is the dominant
early exit, because any near wall reaching the top of the screen ends the column immediately —
the common case looking down an ordinary street.

**The invariant this whole scheme depends on:** every wall rises from a single shared ground
plane (`z = 0`), so a farther, taller wall's screen extent can only ever push *toward the
horizon from below* into rows nothing else has claimed — never appear as an isolated island
below something already drawn. The moment geometry violates this (an overhang, a floating
object, two solid intervals stacked in one column) the single-scalar `coverTop` test becomes
wrong, because "already covered" is no longer a single contiguous run from the bottom of the
column. This is exactly the constraint `ENVELOPE.md` names as the reason bridges, overhangs,
and stacked rooms are unrepresentable without a fundamentally different data structure (one
height *interval* per cell instead of one height *scalar*) — see Chapter 13.

**Why first-hit traversal was withdrawn.** The Phase 0 spike (`spike-height-span.html`,
retained as a regression viewpoint) placed a 6-unit slab at depth 37.5 directly in front of a
22-unit tower at depth 43.5. A first-hit raycaster stops at the slab and never even samples the
tower, leaving 25 rows of screen that should show the tower's upper floors as empty sky
instead. In motion this reads as a rendering bug — buildings popping in and out — not as a
stylistic choice, which is why height-span traversal (the ray continuing past occluded cells)
was made mandatory from Phase 1 onward rather than treated as an optional refinement.

---

## 3. Floor casting

### The idea

Where the wall raycaster fires one ray per *screen column* through the world, floor casting
works the opposite way: it walks one *screen row* at a time (every row below the horizon) and
asks what point on the flat ground plane maps to that row, then samples the ground material
there. This is the standard "horizontal-plane" trick from old row-based engines: because the
ground is perfectly flat, a screen row corresponds to a fixed world depth regardless of screen
column, so the depth only needs to be computed once per row rather than once per screen cell.
That single depth is then used to walk across the row in world-space steps, sampling asphalt,
lane paint, sidewalk seams, and crosswalk markings as it goes.

The genuinely hard part of floor casting isn't projecting it correctly — it's that the ground
plane's perspective is *extremely* non-linear: a screen row just below the horizon can
represent a point 70+ world units away, while the row at the very bottom of the screen is only
2 units away. A fixed-frequency ground pattern (say, a repeating lane-dash texture) sampled
naively at that non-linear rate will alias badly near the horizon — a single screen row there
covers so much world distance that the pattern flickers and shimmers as the camera moves even
by a fraction of a unit, the classic "moiré" failure of any perspective texture without
mipmapping.

### The mechanism

For each screen row `r` below the horizon:

```js
rowDepth = CAM_Z * FOCAL_ROWS / (r - HORIZON);     // = depthAtRow(r), Chapter 2
```
(`castFloorRow()`, ~6361–6362, calls `depthAtRow(r + 0.5)` directly — the same formula
Chapter 2 derives from the wall projection, run in reverse. This is the contrast with Chapter
2's traversal: walls are *one ray per column*, sampled at a variable depth found by DDA; floor
is *one depth per row*, sampled across a variable set of world (x,y) coordinates found by
stepping perpendicular to the view direction at that fixed depth.)

**The anti-aliasing fix — pattern LOD.** Rather than sampling the ground pattern at the raw
world coordinate, the sample point is quantized to a step size that grows with depth (a
logarithmic level-of-detail, computed once per row and applied to every sample coordinate on
that row, not just the noise lookup):

```js
const lod  = Math.max(0, Math.floor(Math.log2(1 + rowDepth / 6)));
const step = 1 << lod;                       // world-space quantum, in quarter-cells
const sx = Math.floor(wx * 4 / step) * step;
const sy = Math.floor(wy * 4 / step) * step;
```
This is the ground-plane equivalent of a mipmap: at each LOD increase, one tier of fine detail
(lane dashes, then sidewalk seams, then asphalt speckle) is dropped, so a row that represents
tens of world units per screen cell samples a coarser, stable pattern instead of a fine one
that would flicker under sub-pixel camera motion.

**The invariant, and how it was found to be violated.** Every ground-pattern decision — lane
dashes, seams, stripes — must read the *quantized* coordinate, not the raw one, even when
deciding *whether* to draw a feature (not just *what* to draw). Quantizing only the noise
lookup while testing raw coordinates for feature placement was measured to change 56% of
ground glyphs per frame under a camera creep of just 0.004 cells; quantizing the coordinate
consistently everywhere it's used dropped that to 2.1%, scaling proportionally with actual
movement thereafter (10.7% at 0.01 cells/frame). The lesson generalizes: a stable *function* of
world coordinates is not the same guarantee as stable *output* if the function's input hasn't
itself been made stable first.

---

## 4. World generation

### The idea

The city isn't hand-authored — it's grown from the seed by a fixed sequence of steps: lay down
a grid of wide avenues and narrower streets, mark sidewalks, subdivide whatever land is left
into building lots, then decide each lot's height, footprint shape, and materials. The single
constraint that shapes almost every other layout decision is a purely geometric one: how far
away do you need to stand before the top of a tall building comes into view at all? Below that
distance, the building's roof is projected *above* the top of the screen (or, more precisely,
above where the visible horizon geometry allows) — so a maze of short blocks can never show a
skyline no matter how tall the buildings actually are; the tallness would be wasted.

The generator also refuses to extrude every building as a flat rectangular box straight up —
that reads as a uniform "bar chart" skyline. Instead a lot's footprint can step in as it rises
(setbacks), or carry a narrower tower rising from a wider podium, or even be carved with a
concave notch or a diagonal corner cut. The surprising finding here is that none of this
richer shaping required any change to the *renderer* at all — it fell entirely out of how the
world data (a per-cell height array) is populated, because the renderer was already built to
handle arbitrary per-cell heights rather than one height per building.

### The mechanism

**The sightline budget.** From the projection formula (Chapter 2), a rooftop at world height
`h` only becomes visible once the required depth clears the point where its top row would
otherwise sit above the screen:

```
depth >= (h - CAM_Z) * FOCAL_ROWS / (ROWS / 2)
       = (h - 0.85) * 83.41 / 36
       = (h - 0.85) * 2.317
```
(spec §8.1; the constant folds in `FOCAL_ROWS` and half the screen height, both fixed once
`recomputeProjection()` has run.) A 24-unit tower (the generator's max height) needs
`(24 - 0.85) * 2.317 ≈ 53.6` cells of clear depth before its roof is visible — so avenues are
laid out at least 56 cells long, unbroken, specifically so the tallest buildings can read as a
skyline rather than as a wall the moment the player looks down one. This formula must always
be evaluated against `HORIZON0` (the neutral, un-pitched horizon), never the live, pitch-shifted
`HORIZON` — otherwise a generation-time layout constraint would depend on which way the player
happens to be looking when the city is built.

**Layout sequence** (`generate()`, following spec §8.4): stamp a boundary ring of unbroken
maximum-height slabs 16 cells deep around the whole 256×256 grid (so the map edge is never
visible within the 96-cell fog cutoff); carve avenues (width 6, every 32 cells) then streets
(width 4, every 8 cells within blocks); mark sidewalks and crosswalks; partition remaining
space into lots by recursive binary split (minimum 3×3); assign each lot a height (biased
toward avenue intersections, plus low-frequency noise so the skyline has structure) and a
massing style; place props (lamps, trees, signal poles) with spacing phased to the block
rather than to the world origin (spacing that shares a factor with the road pitch can silently
land every instance of a prop *inside* a road band and place zero of them — measured once at
spacing 16); build traffic lane loops and pedestrian waypoint graphs; then run `validate()`,
which asserts the layout is actually usable — every avenue traversable end-to-end, every lane
graph a closed cycle, no building overlapping a road — and fails loudly rather than shipping a
broken seed.

**Non-rectangular massing, and why it needs no renderer support.** `stampMassing()` (~2013)
writes per-cell heights directly — slab (uniform), setback (steps down toward the lot edge by
Chebyshev ring), tower (a podium across the whole lot with a narrower inset shaft), courtyard
(a hollow core carved to height 0), plus L-notch and chamfer corner cuts on suitable lots.
Carved cells get `cellHeight = 0` and `CELL_EMPTY` — there is no separate "is this cell inside
the footprint" flag; `cellHeight` alone is authoritative. The reason this composes for free
with everything in Chapters 2–3 is that the façade texture coordinate `u` (Chapter 5, Chapter
7) is derived from the struck face's *absolute world coordinate*, never from a per-building
bounding box or footprint index:

```js
uWorld = cam.y + depth * rdy    // (or the x-analog on the other axis) — world space, not footprint-relative
```
A concave corner produced by a courtyard or notch carve is therefore not a special case for the
sampler at all — it's simply another wall segment with its own `uWorld` values, textured the
same way a straight wall would be. `tools/test-footprints.js` verifies this end-to-end: carving
a building that's on screen leaves all 13,824 framebuffer cells whose *geometry* didn't move
byte-identical, which would not hold if the window lattice were anchored to the footprint
rather than to world space (removing cells would re-phase the lattice on the surviving walls).

**The invariant:** §9.4's coverage scalar (Chapter 2) rests on walls rising from a shared
ground plane, not on the footprint being convex — so nothing about non-rectangular massing
threatens the raycaster's core assumption. The generalizable point: keeping façade sampling
anchored in world space, rather than in any per-object local frame, is what makes footprint
complexity a pure generation-time concern.

---

## 5. The character framebuffer & palette quantization

### The idea

Instead of drawing directly to a canvas each frame, the renderer first fills an intermediate,
purely logical grid — one character glyph, one color, one depth value, and one "how much does
this resist fog" value, per cell of a 192×72 grid — and only converts that grid to actual pixel
drawing calls afterward (Chapter 6). This separation is what makes the whole thing fast: the
expensive part (deciding what glyph and color belongs at each cell, tested against everything
else that might be nearer) never has to think about how it will eventually be drawn, and the
drawing step never has to re-derive anything about the world.

The other major discipline here is color. Instead of computing a full, continuous RGB value
for every cell (as a naive "true color" ASCII renderer would), every color that can appear
anywhere in the city is drawn from a fixed table of exactly 128 entries, decided once at
startup. This sounds like a limitation, but it's what makes the renderer's core performance
trick (Chapter 6) possible at all: identical colors can be recognized instantly (by comparing
two small integers) and batched into a single draw call, which is meaningless if every cell's
color is a unique float-precision RGB triple that will essentially never exactly match its
neighbor's.

### The mechanism

**The framebuffer** is four parallel typed arrays, indexed `row * COLS + col`, allocated once
and never reallocated inside the render loop:

```js
fbGlyph = new Uint8Array(COLS * ROWS);     // index into GLYPH_SET
fbColor = new Uint8Array(COLS * ROWS);     // index into the 128-entry PALETTE
fbDepth = new Float32Array(COLS * ROWS);   // perpendicular depth; Infinity = empty/sky
fbEmis  = new Uint8Array(COLS * ROWS);     // 0..255, fog resistance (Chapter 7/emissive)
```
Every caster — walls (`emitSpan`, ~6160), floor (`castFloorRow`, ~6361), props/cars/pedestrians
— writes through one shared function, `fbPut(col, row, glyph, colorIdx, depth, emis)` (~5288),
which performs the actual depth test and rejects the write if the new depth does not beat what
is already stored:

```js
function fbPut(col, row, glyph, colorIdx, depth, emis) {
  const i = row * COLS + col;
  if (depth >= fbDepth[i]) return;      // per-cell depth test
  fbDepth[i] = depth; fbGlyph[i] = glyph; fbColor[i] = colorIdx; fbEmis[i] = emis || 0;
}
```
A parallel, identically-shaped function, `pickSet(kind, depth, id, sub)` (~7544), runs the same
`depth < PICK.depth` comparison against a single global `PICK` record rather than the
framebuffer arrays. Callers that could win the crosshair cell call both `fbPut` and `pickSet`
at their own write site, with the same depth value, so the two structures can never disagree
about which write actually won. This is why the crosshair pick (Chapter 11) can be captured
exactly at the moment a cell wins its depth test, rather than needing a second, separate
derivation later.

**The palette.** 14 hue families × 8 brightness shades, plus 16 "specials" (signal reds/greens,
headlights, pure black, haze) for things that need a fixed identity rather than a family-plus-
shade encoding:

```js
const HUE_FAMILIES = 14, SHADES = 8, SPECIALS = 16;
const PALETTE_SIZE = HUE_FAMILIES * SHADES + SPECIALS;   // 128
```
`fbColor` stores a `Uint8` index into this table — **no RGB value is ever computed inside the
render loop.** Distance fades a cell *toward black* by lowering its shade within the same hue
family, never by desaturating toward gray, so a building keeps its identity color as it recedes
into fog rather than washing to neutral.

**Why the top of the ramp bends hot.** The shade-to-lightness curve used to be one straight
line, which meant the brightest shade any neon sign or window could reach (shade 7) still only
measured luma 101 of 255 — barely above middle gray. Every rule reserving shades 6–7 for actual
light sources was being followed correctly; the city was simply incapable of ever drawing
something that looked *bright*, so hue alone separated buildings and brightness carried no
information. The fix, discovered only by rendering a frame offline and looking at it (no
aggregate statistic caught this, because "shade 7 is brighter than shade 5" was true on the
broken build too) bends the lightness curve upward specifically for shades 6–7 — e.g. magenta
shade 7 moved from luma 101 to luma 157 — while leaving shades 0–5 untouched, so the structural
hierarchy those shades carry is undisturbed. The reservation of shades 6–7 to genuinely
emissive things (`STRUCT_TOP` caps ordinary structure at shade 5; only signage-grade emissive
reaches shade 7) is what keeps this from being a global brightness boost — it targets exactly
the ~10% of a frame's ink that is meant to read as light.

**Why continuous per-cell RGB is banned, precisely.** A 128-entry palette bounds two things
simultaneously: the number of distinct `fillStyle` values the presenter (Chapter 6) will ever
need to set, which is what makes run-length batching produce long runs instead of one run per
cell; and the size of the glyph-atlas fallback path (`|GLYPH_SET| × 128` pre-rendered tiles),
which would be unbounded if color were continuous.

**Dithering is keyed by row only**, never by column — an 8-entry van der Corput-style sequence
over `row & 7` applied to the fractional part of a shade computation at a brightness boundary.
This spreads a shade transition across several rows rather than producing a hard visual pop.
Keying it by column instead (or by both axes) was measured to cost 3429 runs per frame against
a 1200-run budget, because it makes the shade — and therefore the color index — flip every
cell or two along a single wall, which the row-batched presenter (Chapter 6) cannot merge. The
general rule this proves: **dither across the axis you are *not* batching runs along.**

---

## 6. The presenter

### The idea

Once the framebuffer is fully populated for a frame, it has to actually become pixels on
screen. The naive approach — one `fillText()` call per character cell — would mean thousands
of canvas draw calls per frame, each with real per-call overhead in a browser's 2D context.
The trick this renderer uses instead is to notice that huge stretches of a single row very
often share the exact same color (a flat wall, a stretch of asphalt, an unlit sky band), and
to walk each row accumulating consecutive cells of identical color into one "run," setting the
canvas fill color exactly once per run and drawing the whole run's worth of characters in a
single `fillText()` call.

The payoff of this scheme reaches further than just speed. Because runs are grouped purely by
*color*, and never by *glyph identity*, changing which glyph is drawn at a cell is completely
free from a performance standpoint, even if the glyph varies from cell to cell within a run —
as long as the color stays the same, it is still one run. This is the load-bearing fact behind
several later features: an entire visual style (hatching, cladding textures, animated color-
table effects) can be layered onto the city essentially for free, provided the feature only
ever touches the glyph channel and leaves color alone.

### The mechanism

`present()` (~8522) walks each row, comparing each cell's `fbColor` to the previous cell's; on
a change (or end of row) it flushes the accumulated run as one `ctx.fillText()` call after one
`ctx.fillStyle =` assignment. A run's cost, empirically, is roughly one unit against a measured
budget of **≈1,072 runs per millisecond** (captured with the in-page `P` benchmark, §16.2 —
30 seeded street locations × 8 yaws, two frames each, the second kept, `present_ms` fit against
`runs`), against a 5 ms frame budget — so roughly 5,000 runs would exhaust the budget, though
that figure is explicitly flagged in the spec as an extrapolation beyond observed data and
should not be quoted as a hard ceiling. Measured runs/frame at defaults: median ≈1118–1218,
p90 ≈1568–1724, worst ≈2028–2182 across sampled viewpoints — comfortably inside budget even at
the tail.

**Why color-table animation is free.** Because every color a cell can hold is an *index* into
the 128-entry palette rather than a baked-in value, an effect like a time-of-day cycle, a
flicker, or a "cycle the palette" demo effect can rewrite the palette's RGB *lookup table*
once per frame rather than touching a single cell of the framebuffer. `present()` still walks
exactly the same runs it would have anyway; only the meaning of each index changed. This is
the same principle in the opposite channel from the run-batching trick above: batching exploits
*color* being cheap to compare, and color-table animation exploits *color* being resolved to
RGB only at the very last step, inside `present()`, rather than at the point each cell was
written.

**The invariant the whole scheme rests on:** `present()` breaks a run on *color*, never on
*glyph*. This single fact is cited repeatedly across later chapters (façade materials, the
monolith's face texture, coherent hatching) as the reason a texture, a rotating mark, or a
material can vary freely from cell to cell "for free" — provided it stays inside the glyph
channel and does not perturb `fbColor`. A feature that reads as free in the run counter can
still cost real CPU in the *sampler* that decides which glyph to draw — the run counter is a
presenter metric and is structurally blind to sampler cost (see Chapter 7's CLAD material,
where a naive per-cell noise sampler measured +34% wall-clock cost against a run delta of
exactly zero).

---

## 7. Façade materials

### The idea

A building's wall isn't a flat color — it carries a texture, chosen from one of several
generators that each produce a different visual character while obeying the same hard rule:
they only ever touch the *glyph* a cell displays, never its color, so they're free by
Chapter 6's rule regardless of how visually busy they look.

**Wang tiling** solves a specific problem: how do you tile a wall with a repeating decorative
pattern so that neighboring tiles always visually agree at their shared edge, without either
computing a global solution or falling back to one repeating tile that reads as obviously
mechanical? The trick, borrowed from a decades-old technique for exactly this problem, is to
assign each *edge* of the tile grid (not each tile) a randomly chosen "color," and then choose
which glyph to draw in a given cell based on the four edge-colors surrounding it. Because two
neighboring cells share an edge, and that edge has one color regardless of which cell is asking
about it, adjacent tiles are guaranteed to agree — not by luck, by construction.

**Worley noise** (cellular / "cracked mud" noise) is layered on top to break the tiling up into
irregular panels — real cladding isn't one giant continuous woven pattern, it's discrete
panels with seams between them.

**Vector-field skins** pick a directional glyph (`-`, `/`, `|`, `\`) at each cell based on the
local direction of a smooth, analytically-defined flow field, so a wall reads as "brushed" or
"flowing" in a coherent direction rather than randomly.

**Cellular-automaton cladding** bakes a classic one-dimensional CA (Rule 30, the same rule that
produces the famous Sierpinski-like triangle pattern from a single seed cell) into a small
repeating texture at load time, then just samples it.

Finally, **QUARTER** doesn't add a fifth generator — it assigns *whichever* of the first four
each building uses based on which city district it sits in, rather than per-building, because a
prior experiment already showed that assigning something per-building (in that case, building
hue) produces something statistically indistinguishable from noise: neighboring buildings had
almost no more chance of sharing a hue family than pure chance would predict. Grouping by
district instead — a coarser, spatially coherent unit that already exists for other purposes —
is what makes a "material" or a "hue" read as a *place having a character* rather than as
per-building static.

### The mechanism

All façade material code lives around `ascii-city.html` 5493–5680 (`MATERIAL`, `MAT_NONE`
through `MAT_QUARTER`, `MAT_NAMES`).

**Wang tiling (`wangGlyph`, ~5524–5533).** Edge colors are hashed from the *edge's own lattice
coordinate*, never the cell's:

```js
const eN = hash3(cu, cz, seed) < 0.5 ? 1 : 0;
const eS = hash3(cu, cz + 1, seed) < 0.5 ? 1 : 0;
const eW = hash3(cu + 4096, cz, seed + 1) < 0.5 ? 1 : 0;
const eE = hash3(cu + 4097, cz, seed + 1) < 0.5 ? 1 : 0;
return WANG_TILES[(eN << 3) | (eE << 2) | (eS << 1) | eW];
```
Two edge colors per edge give `2^4 = 16` possible tile configurations, indexed directly by the
four-bit combination of neighboring edge colors — `WANG_TILES` (~5514–5520) is a 16-entry
lookup table mapping each configuration to one glyph. Because `eS` for cell `(cu, cz)` is
computed from `hash3(cu, cz+1, seed)` — the exact same call the cell *below* uses for its own
`eN` — the two cells necessarily agree.

**Worley seams (`worleySeam`, ~5557–5571).** One feature point per lattice cell; a queried
point's seam-ness is the *difference* between the distances to its two nearest feature points
(near zero difference = equidistant = a panel boundary). Originally computed live with 18
`hash3` calls per queried cell (two hashes × nine neighboring lattice cells), this was measured
to cost +34% of total cast wall-clock time relative to no material at all — despite costing
*zero* additional presenter runs, because the run counter cannot see sampler CPU (Chapter 6's
closing point). The fix bakes the feature points into a 64×64 torus (`WORLEY_X`/`WORLEY_Y`,
~5549–5556) once at load, turning the per-cell cost into two array reads; this alone recovered
41% of the material's own overhead for an identical output field.

**Vector-field skins (`flowGlyph`, ~5583–5592).** A smooth, curl-free analytic field —
`sin`/`cos` combinations of `u` and `z` — is evaluated at each cell to get a direction angle,
quantized to one of four 45°-spaced octant pairs, each mapped to one of `- / | \`. Four glyphs
is deliberately the finest resolution useful here: adjacent glyphs already differ by 45°, which
is as fine an angular distinction as a single monospace character can visually express.

**CA cladding (`buildCA`/`caGlyph`, ~5603–5626).** Rule 30 evolved from a single lit cell (not
a random row) into a 64×64 torus, baked once at load (`buildCA(CA_RULE)` runs at module init
time, not per frame or per cell). A single seed cell is used deliberately, not a randomized
starting row, because it reproduces the classic triangular Sierpinski-like landmark shape,
which later experience with the monolith's own face texture (§31b) found matters: a texture
with no recognizable landmark carries no sense of *identity* or orientation as the viewer moves
past it, reading as undifferentiated noise instead.

**QUARTER (`buildingMaterial`, ~5655–5667).** Reuses the *same* `districtSet` lookup the color
palette already uses to assign hue families per district — not a second, independent notion of
where district boundaries fall — specifically so a wall's cladding material and its base hue
can never disagree about which quarter of the city they're in:

```js
function buildingMaterial(b) {
  if (b.mat === undefined) {
    const dx = clamp(Math.floor(cx / DIST_PITCH), 0, DIST_N - 1);
    const dy = clamp(Math.floor(cy / DIST_PITCH), 0, DIST_N - 1);
    b.mat = MAT_HATCH + (districtSet[dy * DIST_N + dx] % 4);   // 6 districts over 4 materials
  }
  return b.mat;
}
```
Cached on the building object (`b.mat`) rather than recomputed per cell, because it is a
per-building constant and the Worley-baking lesson above already established that a per-cell
hash is measurably expensive compared to one lookup. The reuse of `districtSet` is the key
architectural point: it is the same mechanism §14.3 uses for hue-family assignment, chosen
specifically to avoid repeating the measured per-building-hash failure (7.0% same-family
neighbor rate against a 7.6% random baseline — i.e., no structure at all) in a second channel.

**The invariant every generator obeys ("mass only"):** none of these generators may overwrite a
cell that the underlying façade sampler (Chapter 4/10) has already marked as architecture —
mullions, floor slabs, pane divisions, lit windows keep their own glyphs. Each material
generator only fires on cells reported as plain dark wall mass, so a texture re-skins the wall
without erasing the structure drawn on top of it.

---

## 8. Traffic & the lane-loop parity trick

### The idea

Real-world traffic simulation usually needs explicit intersection logic: which car has right
of way, how to merge two lanes, how to prevent a car from being inserted where another already
is. This renderer sidesteps essentially all of that by never routing a car through an
intersection decision at all — every car instead drives forever around a single closed loop
that traces the perimeter of one city block (a "superblock," the area enclosed by four avenue
segments). A car on a loop never has to decide anything at a junction, because the loop simply
*passes through* the junction as one more segment of its own path — from the sidewalk, this
looks exactly like ordinary traffic crossing an intersection, but nothing in the simulation
ever modeled the intersection as a decision point.

The genuinely elegant part is how two-way traffic — cars on both sides of a street driving in
opposite directions — falls out of this scheme for free. Every loop is shrunk slightly inward
from its block's true boundary, offset toward the right (matching right-hand-drive
convention). Because two adjacent superblocks share an avenue as their common edge, and *each*
block's loop is offset toward its own right along that shared avenue, one block's loop ends up
running along the near side of the avenue while the neighboring block's loop runs along the
far side — automatically producing opposite-direction lanes on either side of the centerline,
without ever writing a single line of code that says "these two lanes go opposite ways."

### The mechanism

`buildLanes()` (~2547–2562) builds one closed rectangular loop per superblock: for every pair
of adjacent avenue centerlines on each axis, `addLane()` is given a rectangle inset from the
avenue centerlines by `LANE_OFF` (1.3 cells) on all four sides:

```js
const x0 = aveCentre(ks[a]) + LANE_OFF, x1 = aveCentre(ks[a + 1]) - LANE_OFF;
const y0 = aveCentre(ks[b]) + LANE_OFF, y1 = aveCentre(ks[b + 1]) - LANE_OFF;
addLane([x0, x1, x1, x0], [y0, y0, y1, y1]);
```
This single, symmetric inset is the entire mechanism: `x = aveCentre(k) + LANE_OFF` is the
*right* edge (in the +x sense) of one block's loop and simultaneously the *left* edge of the
next block's loop along the same avenue centerline — the two loops are on physically opposite
sides of the road's centerline purely because each was inset inward from its own rectangle,
and "inward, from the right" resolves to opposite absolute sides depending which block you're
inset from. One rule, correct handedness, zero special-case junction code.

Each loop is a closed polyline (`makeLoop`, ~2567) with its own cumulative arc length; a car's
position is just an arc-length offset (`s`) along its assigned lane's loop, advanced each
substep by its current speed. Because the population of cars is fixed at generation time — no
spawning or despawning — a following car's "car ahead" pointer (`ahead`) never has to be
recomputed; it is set once and stays valid for the simulation's entire life, since nothing is
ever inserted into or removed from a loop.

**Signals compose with, but do not implement, collision safety.** Because each of a junction's
four loops clips a different *quadrant* of the intersection, no two cars can ever physically
occupy the same point regardless of what the traffic lights show — the geometry alone prevents
collisions. What the signal system (derived, never stored — `signalPhase()`, a pure function of
sim clock and a per-junction `hash3` phase offset, so junctions don't all blink in unison) buys
is purely the *appearance* of organized, coordinated traffic — cars queuing and releasing
together — not safety, which the loop geometry already guarantees independently.

---

## 9. Pedestrians & signals

### The idea

Pedestrians follow essentially the same closed-loop trick as traffic (Chapter 8), one scale
down: instead of a loop around a superblock on the roadway, a pedestrian follows a loop around
a single city block on the sidewalk. This works for the same reason it works for cars — a
closed cycle never presents a decision point, so there's no route-planning logic needed at
all, just "keep advancing along the loop you were assigned."

The part worth understanding on its own is traffic signals, because the natural instinct is to
model a signal as an object with state — "this light is currently red" — that gets updated on
a timer and read by anything that needs to know. This renderer does the opposite: no signal
ever stores its current color anywhere. Every time anything (a car deciding whether to stop, a
pedestrian deciding whether to cross, the code drawing the signal's lit lens) needs to know
what color a given signal shows, it *recomputes* the answer fresh, as a pure function of the
current simulation clock and which junction is being asked about. This sounds wasteful but is
actually the safer design: there is no "signal state" object that could ever fall out of sync
with what's being drawn, because the only source of truth is the one formula every consumer
calls.

### The mechanism

**Signals as pure functions.** `signalPhase(kx, ky, axis)` computes which of the two axes
(east-west or north-south) is currently green/amber/red purely from `simClock` (advanced on
the fixed timestep, never from a wall clock — so pausing or slowing the simulation cannot
desync the lights from the cars they govern) and a `hash3`-derived per-junction phase offset,
so different junctions across the city aren't all synchronized to blink together. The two axes
are exactly half a cycle apart, meaning "both directions green simultaneously" is not a state
the formula can ever produce — it's unrepresentable by construction rather than a case that has
to be separately guarded against.

**A subtle but important sign-convention rule:** a pedestrian waiting to cross a road is
waiting on the signal governing traffic *on that road* — the axis *perpendicular* to the
direction they themselves are walking — which is the opposite mapping from a car traveling in
that same direction along the road. Getting this backwards is invisible to any test that only
checks internal self-consistency (a walker that waits for the wrong axis will still wait
*consistently*, just at the wrong times), which is why the actual verification derives the
correct axis independently from segment geometry rather than reading whichever array the
simulation itself uses to decide.

**Distance to a stop line, and why the modulo must be a true modulo.** A car's distance to the
next stop line is computed as a forward scan along its own lane loop's fixed vertex sequence,
wrapped *modularly*: a car that has just crossed a stop line measures its distance to the
*next* one around the loop, becoming free of the junction it just passed rather than remaining
somehow "in relation to" a line behind it. This must be implemented as a genuine `%` operation,
not a single conditional `+= totalLength` — because one lane vertex's arc-length position can
be *negative* relative to where a car currently sits, and a single conditional wrap leaves the
last stretch of the loop permanently reporting a negative distance, which downstream braking
logic misreads as "already past the line," causing a car to freeze just beyond a junction for
the length of a red light it had, in reality, already cleared.

**Amber commitment is provable, not distance-based.** A car only proceeds through an amber
light if, at its current legally-allowed speed, it can *actually* clear the stop line before
the phase ends — computed by comparing against the time amber has left, not against a fixed
"close enough" distance threshold. A fixed-distance threshold cannot make this promise for a
car that's boxed in behind a slower one, and measurably produced cars stuck straddling the
stop line into a red phase before this was corrected.

---

## 10. Procedural audio

### The idea

There are no sound files anywhere in this project — every sound (footsteps, tire roar, a
crowd murmur, a signage buzz, ambient room tone) is synthesized live from oscillators and
filtered noise, using the Web Audio API's node graph. Beyond keeping the file small, this is a
philosophical consistency: every other part of this city — its geometry, its colors, its
signage — is *derived* from world data rather than authored as a fixed asset, and audio
follows the same rule.

The one idea in this chapter worth understanding deeply, because it's a genuinely easy mistake
to make and the spec calls it out by name, concerns how the renderer decides how "open" or
"enclosed" a space feels for reverb purposes. The raycaster (Chapter 2) already knows, for
every column on screen, how far away the nearest wall is and how tall it is — so it's tempting
to derive a "how boxed-in does this space feel" metric directly from what's currently drawn on
screen, e.g. how much of the screen above/below the horizon is filled with wall. That would be
wrong, and specifically wrong in a way that's easy to overlook: this renderer implements
looking up and down not as a true camera rotation but as a vertical *shear* of the whole image
(the horizon line literally slides up or down the screen while everything else keeps its
shape) — which means simply tilting your head, with the world completely unchanged around you,
would change how much of the screen shows wall versus sky, and therefore would change how
reverberant the room sounds even though nothing about the actual room changed. The fix is to
derive the acoustic metrics from quantities that are geometrically invariant to that shear:
real-world depth and real-world height, never a screen row or anything read off the rendered
image.

### The mechanism

**Acoustic aggregates, computed inside the existing raycast loop** (`castColumns()`, the same
loop from Chapter 2, ~4227–4382) at essentially zero extra cost — two accumulators, updated
once per column as a byproduct of the DDA the wall-casting already performs:

```js
let acOpen = 1, acEnclose = 0;                 // (declared ~4223)
const AC_SUBTENSE = 0.35;   // (h - eye) / d above which a column reads "walled"
// ... inside castColumns(), per column:
//   near      = nearest-face depth, accumulated into nearSum across all columns
//   elevation = (heightAt(idx) - CAM_Z) / depth      <- the tangent, NOT a screen row
//   encCount += (elevation > AC_SUBTENSE) ? 1 : 0
acOpen    = clamp(nearSum / COLS / MAX_DEPTH, 0, 1);
acEnclose = clamp(encCount / COLS, 0, 1);
```
`openness` (`acOpen`) is the mean nearest-face depth across all columns, normalized by
`MAX_DEPTH` — how far away is the nearest wall, on average, all around you. `enclosure`
(`acEnclose`) is the fraction of columns whose nearest wall subtends more than a threshold
elevation angle — the *elevation tangent* `(height − CAM_Z) / depth`, a ratio of two purely
geometric, depth-and-height quantities with no screen-space term anywhere in it. Because pitch
is implemented as a shear of `HORIZON` (Chapter 2's §9.5), and this formula never references
`HORIZON`, `coverTop`, or any row value, it is invariant under pitch by construction — tilting
the camera up or down cannot change `acOpen` or `acEnclose`, only turning or walking can. This
constraint is enforced in the test harness specifically because it is judged the single
easiest invariant in the audio system to break silently — a future edit reaching for "what's
drawn near the top of the screen" as a proxy for enclosure would pass every ordinary check and
only fail a dedicated invariant test.

**The graph.** One ambient bed plus footsteps feed a dry path directly to master, and
separately a wet path through a pre-delay and a convolution reverb; `openness` drives the
reverb's damping cutoff and pre-delay time, `enclosure` drives its wet gain and dry
attenuation — a single procedurally-generated impulse response is reused throughout, with the
*feel* of different spaces produced entirely by ramping these send/return parameters rather
than swapping impulse responses.

**Standing constraints worth internalizing for reimplementation:** noise and impulse buffers
draw from the same seeded `mulberry32` discipline as Chapter 1 (a second, independently seeded
stream, never `Math.random()`); audio is strictly a read-only observer of simulation state —
it never writes back into it, and the audio clock (`AudioContext.currentTime`) never touches
the fixed simulation timestep, which is what keeps audio from threatening determinism; every
runtime parameter change is ramped via `setTargetAtTime` rather than assigned directly, because
a direct assignment produces an audible "zipper" click on every camera turn; and footsteps are
triggered by distance actually walked (measured *after* collision resolution, so walking into
a wall produces no footstep), never by elapsed time, so sprinting doesn't merely play the same
footstep sound faster.

---

## 11. Entities & picking

### The idea

Cars, pedestrians, and props are drawn as sprites projected into the same shared depth buffer
walls and floor use — meaning a sprite competes for visibility on exactly equal terms with
everything else already drawn that frame; a car standing partly behind a building corner has
its far end correctly hidden and its near end correctly visible, cell by cell, because both
are being tested against the identical depth buffer.

The crosshair "what am I looking at" readout has a specific, deliberately narrow design
principle behind it, worth understanding because it generalizes to a whole class of bugs this
project's own history ran into repeatedly. The tempting, obvious way to implement "what is the
player looking at" is: after the frame is fully rendered, read the depth value sitting in the
one cell right under the crosshair, then walk out from the camera by that many world units
along the view direction, and ask the world-generation data structures what's located at that
point. This *sounds* completely reasonable and would pass almost any casual test — but it is
subtly wrong in a specific, dangerous way: it does not ask "what mechanism actually drew this
pixel," it re-derives an *independent guess* at the answer using a second, different
calculation. If a car happens to be standing in front of a building at that exact point, this
approach will correctly compute a point in space, then look up the building at that location
and confidently report the building — silently naming the wrong thing, in a way that no
end-to-end visual check would catch, because a screenshot showing a car does still show
*something* recognizable near the crosshair. The fix is architectural: instead of
re-deriving the answer a second time, capture it at the exact place it was first decided.

### The mechanism

**Sprite projection** reuses the identical `FOCAL_COLS`/`FOCAL_ROWS` constants from Chapter 2,
so entities and walls can never disagree about scale:

```js
spriteRows = entityHeight * FOCAL_ROWS / depth;
spriteCols = entityWidth(headingRelativeToCamera) * FOCAL_COLS / depth;
```
Occlusion is per-cell and per-sprite-column (not one depth for the whole sprite): a long car's
depth is interpolated across its own screen footprint from its oriented heading, so a car
straddling a building corner is correctly split between visible and occluded columns. Every
write — wall, floor, or sprite — goes through the one shared function `fbPut` (~5288, Chapter
5), which performs the actual `depth >= fbDepth[i]` rejection before allowing the write.

**The pick, captured at the write site (`PICK`, ~7536; reset each frame by `pickReset()`,
called from `writeBackdrop()`).** `PICK` is a single record — kind, depth, id, side, world
coordinates — overwritten in place. A parallel helper, `pickSet(kind, depth, id, sub)` (~7544),
performs the identical `depth < PICK.depth` comparison `fbPut` performs against the framebuffer,
but against `PICK.depth` instead:

```js
function pickSet(kind, depth, id, sub) {
  if (!(depth < PICK.depth)) return false;
  PICK.kind = kind; PICK.depth = depth; PICK.id = id | 0; PICK.sub = sub | 0; PICK.door = 0;
  return true;
}
```
Every caster that could possibly win the crosshair's specific screen cell (`emitSpan` for
walls, `pickFloor`/`castFloorRow` for ground, the prop/car/pedestrian casters) calls `pickSet`
at its own write site with the *same* depth value it is about to pass to `fbPut` — not a
second, separately-computed depth. Sky is the implicit fallback state, because sky writes
`fbDepth = Infinity`, which loses to literally anything with a finite depth. Rain is
deliberately excluded from ever winning the pick — you cannot target a raindrop.

The oracle this design allows for is exact rather than approximate: with rain off,
`PICK.depth === fbDepth[PK_I]` must hold every single frame, where `PK_I` is the crosshair's
framebuffer index — an *equality*, made possible specifically because `PICK.depth` was captured
at the same write that produced `fbDepth[PK_I]`, not derived independently from it afterward.
(The comparison uses `Math.fround(PICK.depth)` purely to account for `fbDepth` being a
`Float32Array`'s narrower precision — a representational detail, not a tolerance band.) Tested
as a strict equality, this check caught real bugs on its first run; a looser, tolerance-based
version of the same check would have passed on a genuinely broken build, because "roughly the
same depth" is exactly what a wrong-but-plausible re-derivation produces.

**The invariant, generalized:** whenever a system needs to answer "what did the renderer just
draw here," the answer must be captured at the one place that decision was actually made, not
re-derived through a second, independent path — however innocuous that second path looks. This
same class of bug recurred in at least two other subsystems in this project (a signage mirror
defect, a signal-pole axis defect) before being named explicitly as a pattern to avoid.

---

## 12. Walking your own filesystem as a city

### The idea

Drag a folder onto the page and the entire city is rebuilt from that folder's contents: each
city block becomes a directory, each building inside that block becomes one entry of that
directory (a file, or a nested sub-directory), a building's height reflects how much data that
entry contains (recursively, for directories), and its color reflects what *kind* of data
dominates it (mostly source code, mostly images, mostly documents, and so on). The street grid
itself — the avenues, sidewalks, traffic, crossings — is completely untouched; only the
*buildings standing on the lots* change, because nothing in the road-generation logic ever
depended on what was going to be built on the land it laid out.

The genuinely hard design problem here is depth: a real filesystem's directory tree can be
enormously wider or deeper than the roughly 550 building lots this city actually has room for.
Two obvious, simpler approaches were tried and both measurably failed. Giving every single
*file* its own building overflows catastrophically for any real, sizeable directory — a
60,000-file home folder would need 60,000 buildings for ~550 lots, so almost the entire tree
goes completely undrawn, and worse, *which* files get drawn is essentially arbitrary (whichever
happened to be visited first in the directory walk), not a meaningful sample. Alternatively,
showing only one fixed level of the directory tree (the immediate children of the dropped
folder, and nothing deeper) usually leaves most of the city's lots sitting empty, because most
directory trees don't have enough top-level entries to fill 550 buildings — producing a
half-built ghost town instead of a full city.

The actual solution treats depth as a *budget to spend*, not a fixed number to pick in advance:
draw the current level of the tree in full, and if there is still room left over, take the
single *largest* remaining unexpanded directory and expand it into a full block of its own —
its children each becoming buildings — then repeat, always picking the currently-largest
unexpanded directory, until the lots run out. A directory that never gets its turn to be
expanded doesn't disappear from the city — it's still standing, as a single tower whose height
already reflects everything inside it. Nothing in the tree is ever silently dropped; anything
not shown at full resolution is simply *summarized* as one building instead.

This whole feature is also privacy-sensitive in an unusually literal way — it's reading the
contents of a folder on someone's actual computer — so the project treats "nothing about this
folder ever leaves the browser tab" not as a promise stated in the UI copy, but as a property
enforced by an automated check that scans the entire source file for any API capable of
transmitting data off the machine, with its own deliberately broken negative-control test
proving the scan actually catches an injected network call rather than passing regardless of
what it's given.

### The mechanism

**Ingestion (`fsWalk`/`fsReadDir`, ~3504–3538).** The browser's drag-and-drop
`webkitGetAsEntry`/`FileSystemEntry` API is used rather than the more modern File System Access
API's `showDirectoryPicker`, specifically because the latter is Chromium-only and requires a
secure browsing context — which a plain `file://`-opened HTML file (the normal way to use this
artifact) does not have. Two traps in the entry API are load-bearing: `readEntries()` returns
at most 100 entries per call and signals the end of a directory by returning an *empty* array,
so it must be called in a loop (`step()` recursing until an empty batch) or any directory with
more than 100 children silently truncates; and `preventDefault()` must be called on the
`dragover` event specifically because without it the browser's default behavior is to navigate
the entire tab away to the dropped folder, which looks exactly like a crash.

**Kind classification (`fsKind`/`FS_KIND_FAM`, ~3469–3488)** groups files by *purpose* (code,
doc, data, image, media, binary, config, other) via extension lookup, not by raw extension
string — the goal is that a source-heavy folder visually reads as "a source district," which a
literal per-extension palette would not achieve as cleanly. A directory's dominant kind
(`fsDomKind`, ~3489–3493) is decided by which kind holds the most cumulative *bytes* among its
contents, not by file count — a folder of 300 tiny config files sitting beside one 2 GB video
is unambiguously "a video folder" to a human glancing at it, and height is already keyed to
byte size, so keying color to count instead would put a visually tall building in the color of
its rarest content by volume.

**The block grid and Hilbert ordering (`fsBlockGrid`, ~3544–3584).** Available building lots
are grouped into their enclosing city blocks (the rectangular gaps between avenue/street
bands), then the blocks are visited in **Hilbert-curve order** rather than simple raster
(row-major) order, specifically so a directory subtree spilling across several blocks lands as
a compact spatial *patch* on the map rather than a thin ribbon wrapped arbitrarily across
distant parts of the city — measured to roughly halve the average bounding-box area of a
multi-block spill compared to raster order.

**The adaptive-depth pour (`fsPour`, ~3610–3692).** The root directory's immediate entries are
queued first; then a priority queue of not-yet-expanded subdirectories, sorted by recursive
size descending, is drained one at a time as long as there is still lot budget left:

```js
const queue = root.subs.slice().sort((a, b) => b.recSize - a.recSize);
while (queue.length && used < budget) {
  const n = queue.shift();                    // the LARGEST remaining unexpanded directory
  const es = fsEntries(n);
  if (used + es.length > budget && expanded > 0) { summarised++; continue; }
  groups.push(es); used += es.length; expanded++;
  for (const s of n.subs) queue.push(s);       // its children join the same competition
  queue.sort((a, b) => b.recSize - a.recSize);
}
```
Every directory that is expanded contributes its own children as new buildings; every
directory that never gets popped off the queue before the budget runs out remains, exactly as
it already was, a single unexpanded building — `nothing is missing, only summarised` is
architecturally true rather than aspirational, because an unexpanded directory was *never
removed* from the placement list, it simply never had its own children substituted in.

**Height (`heightOf`, ~3642–3643) is `log10` of recursive byte size**, normalized between the
15th and 98th percentile of the sizes actually being drawn (not the true min/max — a single
huge outlier file would otherwise crush every ordinary file to the floor of the height range),
then mapped linearly onto the engine's `[4, 24]` world-unit height range. A log scale is
mandatory rather than a stylistic choice: `stampMassing`'s height clamp only spans a 10× dynamic
range, while real file sizes routinely span six orders of magnitude — a linear map at any
scaling would put nearly everything on the floor except a single tallest tower.

**Privacy is a code property, not a UI claim.** `tools/test-fsworld.js` scans the entire
artifact source for any API surface capable of moving bytes off the machine (`fetch`, `XHR`,
`WebSocket`, form submission, etc.), and includes a control that deliberately injects a
`fetch(` call into a copy of the source specifically to confirm the scan actually turns red
when given something to catch — proving the check itself is capable of failing, not merely
that it happens to pass on this file today.

**Why it needs no renderer change at all.** `castColumns()` reads only `cellHeight` /
`cellType` / `cellBuilding`, and `sampleFacade()` reads only `buildings[id]` — neither cares,
or has ever cared, where those arrays' contents came from. Pouring a filesystem into those same
arrays *after* `generate()` has already laid out the street grid means the entire rest of the
renderer — raycasting, façade sampling, materials, traffic, pedestrians, the crosshair pick —
runs completely unmodified against filesystem-derived data. The street grid survives because
roads are a pure function of grid position (`bandOf(x)`/`bandOf(y)`) with no dependency on what
occupies the blocks between them — a city can have its entire building stock swapped out and
remain, mechanically, the same city.

---

## 13. What this can't do, and why

### The idea

Every renderer makes a tradeoff between what it can represent and how cheaply it can draw it,
and this one's tradeoff is unusually clean to state: the world is a **height field** — a flat
grid where every cell holds exactly one number, "how tall is whatever's here" — and nothing
more. A building, in this engine's terms, is not a 3D mesh; it's a column of solid space
starting at the ground and going up to that one height value, with nothing above it and no way
to describe "solid here, but only between these two heights, with open space above and below."

This single fact is the root cause of every genuine limitation the engine has, and it's worth
understanding *why* it's the root cause, because the reasoning generalizes: the entire
raycasting scheme in Chapter 2 depends on the fact that a farther, taller wall can only ever
become visible *above* whatever nearer geometry has already been drawn in that screen column —
which is only true because every wall starts at the same shared ground level. The moment
something needs "solid, then a gap, then solid again" in the same column — a bridge you can
walk both over and under, an overhang, a room stacked above another room — that foundational
assumption breaks, and the cheap one-scalar-per-column bookkeeping (`coverTop`) that makes this
whole engine fast stops being a valid shortcut. It isn't a matter of writing more code within
the existing data structure; the data structure itself (one height per cell) cannot represent
the shape being asked for, full stop.

The one deliberate, hand-built exception in the whole engine — a rotating cube rendered
standing on one vertex, purely as a visual centerpiece — is instructive precisely because of
*how* it's an exception. It is genuinely, geometrically three-dimensional: it's drawn with a
real ray-versus-box intersection test, one ray per character *cell* rather than one per column,
fired against the box in its own rotated coordinate frame. But its *lighting* is a stylistic
convention, not a simulated light source (a real world-space light was tried and measured to
fail, for a specific geometric reason: a cube balanced on one vertex has its faces arranged
symmetrically enough around the vertical axis that no fixed light position can ever tell two of
its three lit faces apart), and its *collision* is a plain circle, not the true rotating
hexagonal silhouette the object actually presents on screen — because the collision system
tests position on a 3×3 neighborhood of grid cells and was never built to sweep against an
arbitrarily rotated shape. The lesson: "this engine can do real 3D geometry" and "this engine
is a 3D engine" are different claims, and only the first one is true.

A separate, and genuinely counter-intuitive, non-limit is worth stating precisely because it's
so easy to assume the opposite: a building's footprint does **not** need to be a simple convex
rectangle, and the renderer required *zero* changes to support concave notches and carved
courtyards. This is not a case of "we got lucky" — it follows directly from an architectural
decision made for an unrelated reason (Chapter 4): the façade sampler was already built to
derive its texture coordinates purely from a wall's own absolute position in world space,
rather than from anything relative to the building's overall shape or bounding box. Once that
decision was made, a concave corner is simply one more wall segment with its own world
position — nothing about the sampler ever needed to know, or ask, whether the building it was
texturing was convex.

### The mechanism

**Environments — any height field, and nothing else.** `cellHeight` (Chapter 4, ~§7.1) holds a
single value per grid cell, written once at generation and touched at exactly one other
moment: `E`/room-entry carves an entered building's interior cells to 0 and restores them
byte-exactly on exit (the only way this engine can produce a walkable interior is to
temporarily remove the building while the player is inside it — evidence *for* the
one-height-per-cell constraint, not an exception to it). `blocked(px, py)` (~4121), the
collision test, takes only two arguments — there is no `z` parameter at all, so "solid at this
(x,y) only above/below a certain height" is not merely unimplemented, the function that would
need to answer it structurally cannot be asked the question.

Explicitly unrepresentable, with the shared mechanism named:

| Thing | Why the height field forbids it |
|---|---|
| A bridge walkable both over and under | needs one *interval* per cell (a floor height and a separate ceiling height); `blocked()` has no `z` to test against either |
| An overhang | the identical problem — two solid surfaces stacked in one column, and `coverTop`'s single-scalar coverage assumes exactly one |
| A tunnel visible from outside (not entered) | same interval problem, from the outside looking in |
| A room stacked directly above another room | one height value per cell; interiors exist only as temporary carves, never as permanently stacked geometry |

Two things that *look* like they should be on that list, and are explicitly not, because the
underlying mechanism doesn't actually require the missing piece:

- **Windows** (§30) are the interval problem *minus its collision half* — a window is a purely
  visual aperture (`emitSpan` draws the wall above and below the opening as two separate spans
  and simply declines to raise `coverTop` for that column), with no requirement that `blocked()`
  ever learn about height, because nobody needs to be *stopped* by a window the way they would
  by a wall.
- **Interiors reached through a portal** (§29) exist for the same reason: the building is
  temporarily hollowed (its interior cells zeroed) rather than needing a second height value
  layered over the first, and a flat interior ceiling is drawn by literally mirroring
  `rowTopAt`'s own formula (`depthAtCeilRow`, ~6291) with one sign flipped — no new interval
  data structure required, because there is still only ever one active height value per cell at
  any given moment.

**`cellHeight`'s own numeric representation caps how gentle a slope can be.** It is stored as a
`Uint8Array` in **quarter-units** — one storable step is 0.5 m of rise over a 2 m cell width —
so the *gentlest non-zero, non-flat slope this engine can ever express is a 25% grade (≈14°)*.
There is no representable value between perfectly flat and a one-in-four climb. (This is
`FEASIBILITY-terrain-navigation.md`'s finding, not something the renderer currently exploits —
walkable sloped terrain is presently unbuilt, and `isSolid` reports the entire terrain map as
impassable; the constraint is named here as a hard ceiling on what any future terrain-walking
feature could achieve without changing the underlying storage format.)

**Objects — any convex primitive with a hand-written ray intersection, at the cost of one ray
per covered character cell.** The monolith (`castMonolith`, ~8266) is the existence proof: a
real three-axis slab test against an oriented box, transformed into the box's own orthonormal
frame, run once per character *cell* the object could plausibly cover (nested column-and-row
loops, unlike every other caster in this engine, which is one ray per column only). Because
the transform is orthonormal, the intersection parameter `t` the slab test produces *is*
already perpendicular depth — the exact same quantity every other caster writes into `fbDepth`
— so the object occludes, and is occluded by, ordinary geometry through the same shared
per-cell depth test (`fbPut`, Chapter 5/11) with no special-case interaction code needed
anywhere else in the renderer.

**The deliberate non-limit, stated precisely:** non-convex, non-rectangular building footprints
require *no* renderer support whatsoever, because `uWorld` — the façade texture coordinate
every sampler reads (Chapter 4, Chapter 7) — is derived from the struck wall's absolute world
position, never from the building's footprint, bounding box, or any notion of convexity. A
concave corner produced by a courtyard or notch carve (Chapter 4) is, from the sampler's
perspective, indistinguishable from any other wall segment. This is the mirror image of the
genuine limitations above: those come from the *ray-traversal and collision* layer having no
concept of a height interval; this non-limit exists because the *texturing* layer was already
decoupled from footprint shape for an unrelated reason. A common mistaken assumption, worth
naming explicitly because it has been asserted incorrectly more than once in this project's own
history, is that concave footprints would need special renderer handling — they do not, and
`tools/test-footprints.js` verifies it by carving a building and confirming every framebuffer
cell whose geometry didn't move stays byte-identical.

---

## Cross-references

- Spec: `ascii-cyberpunk-city-renderer-spec-v3.md` — the authoritative decision record, organized
  by `§N`; code comments cite matching section numbers.
- `TREATMENT.md` — the portable kernel list (projection, height-span traversal, determinism,
  palette quantization, lattice snapping) this document expands into full derivations.
- `ENVELOPE.md` — the direct source for Chapter 13's limits, with every prohibition tied to its
  mechanism rather than stated as a bare conclusion.
- `FEASIBILITY-terrain-navigation.md` — the quarter-unit height-storage finding cited in
  Chapter 13.
- `tools/make-cityfs.js` — the offline instrument for §32's directory-to-city mapping; the
  renderer's own `fsApply()`/`fsPour()` (`ascii-city.html` ~3700, ~3610) is the sole
  implementation the tool measures rather than duplicates.
