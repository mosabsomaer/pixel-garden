# Pixel Garden - Zen Mowing Animation

A top-down pixel art mowing animation built for Instagram Reels (9:16, 1080x1920). A riding lawnmower cuts tall grass in the shape of a Ramadan crescent moon (hilal), with looping mower sound effects.

## How to Run

Serve the project root with any static HTTP server and open `index.html`:

```sh
python3 -m http.server 8081
# then visit http://localhost:8081/index.html
```

Click the canvas or press any key to start audio (browser autoplay policy).

## Project Structure

```
pixel-garden/
  index.html                  # The entire animation (single-file, no build step)
  meme.mov                    # Lawnmower sound effect (looped during animation)
  plan.md                     # Original design notes
  art_entry/                  # Isometric pixel art assets (OCaml game project)
    vehicle_all.png           # Riding mower sprite sheet (used in animation)
    vehicle.ml / .mli         # OCaml source defining sprite layout
    tilesheet.ml / .mli       # Tilesheet helper (confirms 32px tile size)
    ...                       # Other game assets (not used in animation)
  main-characters-home-free-top-down-pixel-art-asset/
    Tiled_files/
      exterior.png            # Terrain sprite sheet (272x912, 17 cols, 16px tiles)
      ground_grass_details.png # Grass/stone detail sprites (336x288, 21 cols, 16px tiles)
      Exterior.tmx            # Tiled map file (reference for tile IDs)
      Interior1.tmx           # Interior map (not used)
      ...                     # Other animation sprites (birds, cat, smoke, trees)
    PNG/                      # Higher-res copies of the same sprite sheets
    PSD/                      # Photoshop source files
```

## Architecture (index.html)

Everything lives in a single self-contained HTML file. No dependencies, no build tools.

### Rendering Layers (drawn in order each frame)

1. **bgCvs** - Pre-rendered tall grass background (rendered once at init)
2. **cutCvs** - Accumulates cut ground tiles as the mower progresses
3. **Particles** - Green grass clipping particles spawned when tiles are cut
4. **Vehicle** - Animated mower sprite with CSS drop shadow
5. **vigCvs** - Pre-rendered radial vignette overlay

### Key Systems

- **Shape Builder** (`buildShape`) - Defines the crescent moon using two offset circles. A tile is "in the shape" if it's inside the outer circle but outside the inner circle.
- **Path Builder** (`buildPath`) - Generates an S-pattern mowing path that adapts to the shape's horizontal bounds per row band. The mower only cuts tiles within the crescent.
- **Tile Cutting** (`drawCutTile`) - When the mower reaches a tile, it draws the ground tile from exterior.png onto the cut canvas, with a subtle stripe tint and optional detail spots from ground_grass_details.png.

### Sprite Sheet Tile Mapping

All tile positions were derived from the TMX (Tiled Map Editor) files. The formula:

```
local_index = tile_gid - firstgid
row = floor(local_index / columns)
col = local_index % columns
source_x = col * 16
source_y = row * 16
```

**exterior.png** (firstgid=757 in TMX, 17 columns):
| Tile | TMX GID | Row | Col | Usage |
|------|---------|-----|-----|-------|
| Grass fill | 1065 | 18 | 2 | Base grass for the entire field (the ONLY solid fill tile) |
| Ground | 1116 | 21 | 2 | Cut/mowed ground |
| Tall grass decor | 1676-1701 | 52-55 | various | Individual grass blade sprites scattered randomly |

Important: Tiles adjacent to the grass fill (row 18 cols 1,3 / rows 17,19) are edge/border tiles with transparency. Using them as fill causes black spots.

**ground_grass_details.png** (firstgid=1 in TMX, 21 columns):
| Tile range | Rows | Usage |
|------------|------|-------|
| Spots layer | 1-4 | Small stone/flower decorations on ground |
| Grass details | 8-17 | Multi-tile grass/flower clusters (used in TMX detail layers) |

**vehicle_all.png** (32px tile units, from art_entry/):
- Each frame: 2x3 tiles = 64x96 pixels
- Layout: `srcX = (phase + 8 * ddir) * 64`, `srcY = dir * 96`
- 8 directions (0=NW, 1=W, 2=SW, 3=S, 4=SE, 5=E, 6=NE, 7=N)
- 8 animation phases per direction
- 3 turn states (ddir: 0=straight, 1=turn-left, 2=turn-right)
- Displayed at 2.2x scale with drop shadow

### Configuration Constants

| Constant | Value | Notes |
|----------|-------|-------|
| SRC | 16 | Source tile size in sprite sheets |
| TILE | 40 | Display tile size on canvas |
| COLS | 27 | Grid width (27 * 40 = 1080) |
| ROWS | 48 | Grid height (48 * 40 = 1920) |
| CUT_W | 2 | Rows cut per horizontal mowing pass |
| DURATION | 70 | Total animation duration in seconds |
| VEH_SCALE | 2.2 | Vehicle sprite display scale |

### Randomness

Two deterministic random systems ensure stable visuals across frames:
- `hash(a, b)` / `rand(a, b)` - Position-based hash for per-tile decorations (same tile always gets same decoration)
- `srand()` - Sequential seeded PRNG for runtime particle/decoration generation

## Important Notes for Claude

- Do NOT read PNG files into context - they waste tokens and provide no useful text data. Use the TMX files to understand tile positions instead.
- The TMX files are very large (300KB+). Use offset/limit parameters or Grep to find specific data rather than reading the whole file.
- The exterior.png sprite sheet has 57 rows of tiles. Only a handful are used: row 18 col 2 (grass), row 21 col 2 (ground), rows 52-55 (tall grass decorations).
- The grass fill tile (row 18 col 2) is the ONLY solid fill grass tile. Its neighbors are border/edge tiles with transparency.
- The vehicle sprite is isometric but used in a top-down context. Direction 5=East for "right", 1=West for "left", 3=South for "down", 7=North for "up".
