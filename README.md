# ASCII Cyberpunk City

**A walkable first-person cyberpunk city rendered entirely in characters.
One HTML file. No build, no dependencies, no network requests.**

### → [Walk it](https://skinnerboxentertainment.github.io/super-secret-ascii-city-visual-clone-thingie/)

Click the page to capture the mouse, then walk.

---

## Controls

| | |
|---|---|
| `W A S D` | walk |
| mouse | look |
| `E` | open the door you are facing — or, on a **billboard**, name the loop and credit whoever made the footage |
| `B` | back / leave |
| `V` | cycle the look: RESTRAINED / BALANCED / MAXIMAL |
| `X` | cycle the wall material |
| `T` | tuner — live sliders over every feel constant |
| `M` | debug overlay: frame/sim/ray/present in ms, runs, DDA steps, minimap |
| `P` | presentation benchmark |
| `R` | reseed the city |
| `G` | wireframe overlay |
| `N` | mute |
| `Esc` | release the mouse |

**Drag a folder onto the page** and the city becomes that folder — buildings are
files and directories, sized by what they contain. `E` on a folder descends into
it, `B` comes back up. Nothing leaves your machine; the page makes no network
requests of any kind.

---

## What it is

Every wall, car, pedestrian, sign and raindrop is a character on an 192 × 72
grid, drawn with one ray per column and a per-cell depth buffer. The palette is
128 indexed colours, so a whole frame batches into roughly 1,400 draw calls.

The city is a **pure function of a seed** — nothing about it is stored. Change
the number and you get a different city, deterministically, every time.

The audio is synthesised in the browser: no samples, no files. The layout of the
city is also its score — a closed bank of partials over a drone, where the shape
of the buildings around you decides the harmony.

**217 animated billboard loops** play on the walls. Seven are drawn by the
project; 210 are transformed from third-party footage and every one is credited
in [ATTRIBUTION.md](ATTRIBUTION.md) — or in the city itself, by pressing `E`
while looking at one.

---

## A warning about load time

This is a **5.4 MB single file**, about 3 MB over the wire, and roughly 88% of
that is billboard data. The first load takes a moment while the browser parses
it. Once it starts, it runs.

If that matters more to you than the animation does, the development repository
generates a build with no third-party loops at all, which is **700 KB**.

---

## Licence

MIT — see [LICENSE](LICENSE). [NOTICE](NOTICE) carries the third-party and
provenance position, including what was done to the billboard footage and how
to have a loop removed.

The font is **unscii-8** by Viznut and contributors, public domain, subsetted to
the 82 glyphs this renderer can draw and embedded as a data URI.
