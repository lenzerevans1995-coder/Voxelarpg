# 05 — Procedural Buildings

Buildings use a **split / shape grammar** (CGA-style) that works on oriented
boxes in meters. Rules split volumes into floors, bays and parts, then emit
**stamps** (SDF or box primitives with materials) into the Voxel Command
Buffer. A **style sheet** picks rule variants, proportions, materials and
ornaments, so the same grammar builds a Gothic chapel, a High Fantasy tower,
a desert house or a snowbound hall.

## 1. Grammar structure

```
Lot ──► Footprint ──► Mass ──► { Floors ──► Facades ──► Bays ──► (Window | Door | Wall | Buttress) }
                         └──► Roof ──► (Gable | Hip | Spire | Dome | Flat+Crenel) ──► Ornament
                         └──► Towers / Annexes (recursive Mass)
```

Rules are small GDScript classes with one job each:

```gdscript
# game/worldgen/buildings/rules/split_floors.gd
class_name RuleSplitFloors extends GrammarRule

func apply(shape: Shape, ctx: GrammarContext) -> Array[Shape]:
    var h := ctx.style.floor_height            # e.g. 3.2 m Gothic, 2.8 m Nordic
    var ground := ctx.style.ground_floor_height
    return shape.split_y([ground] + ctx.fill(h, shape.size.y - ground), ["ground_floor", "floor"])
```

The engine (`grammar.gd`) is ~300 lines. It holds a symbol → rule map from
the style sheet, a seeded RNG, an LOD switch for preview, and a stamp output.
Rules never touch voxels.

## 2. Primitive stamps (kernel side)
Rules emit only these, and the C++ kernel rasterizes them:
`box`, `rounded_box`, `cylinder`, `cone`, `sphere/ellipsoid`, `wedge`
(roof slopes), `arch_pointed(width, height, depth, thickness)`,
`arch_round`, `arch_horseshoe`, `profile_extrude(2D polygon)` (moldings),
`lathe(2D profile)` (columns, finials, domes), `pattern_2d(mask)` (tracery,
rose window, glyph bands), and the modifiers `subtract`, `erode(noise, amt)`
and `paint(material, mask)`.

Because everything is in meters, a 1.0 m lancet window at 10 cm voxels has
10 voxels of width: enough for a pointed arch with a center mullion. A 3 m
rose window gets 30 voxels across, which is enough for 8-petal tracery.

## 3. Building types
| Type | Notes |
|---|---|
| **Townhouse** | Narrow, 2–3 floors, jettied upper floors (Gothic), steep roof |
| **Chapel** | Single nave, east apse, bell-cote, lancet windows, small graveyard lot |
| **Cathedral** | Cross footprint (nave, transepts, choir, apse), flying buttresses, twin west towers with spires, rose window over the west portal |
| **Tower** | Square or round mass, stair turret annex, conical or spire roof |
| **Mausoleum** | Small mass, heavy portal, pediment or pointed gable, statue niche |
| **Castle / Keep** | Keep mass, curtain wall around a polygon, round corner towers, gatehouse with portcullis, crenellations and machicolations |
| **Desert House** | Courtyard plan (hollow mass), flat roof with parapet, small domes, horseshoe arches, wind-catcher tower |
| **Nordic Hall** | Long rectangle, stone plinth, timber walls, very steep roof with carved ridge beams, snow load |
| **Ruin (modifier)** | Applies to any type (below) |

### Cathedral grammar (example, simplified)
```
Cathedral(lot)      → Footprint.cross(nave=lot.len*0.6, transept=lot.w*0.9, apse=semicircle)
Nave                → Mass(h=14m) + Aisles(h=8m, both sides) + Roof.gable(pitch=60°)
Aisle.facade        → Bays(width=4m) → each: Buttress | LancetWindow(1.4×5m)
Buttress            → Pier(1.2×1.2) + FlyingArch(to clerestory) + Pinnacle(finial)
WestFront           → TwinTowers(6×6, h=28m) + Portal(pointed, 3 orders, tympanum) + RoseWindow(r=3m)
Tower.top           → Spire(octagonal, h=12m) + Crockets(every 1m) + Cross
Apse                → Polygon(7 sides) + LancetWindow per side + Roof.cone_half
```

## 4. Style sheets (`data/styles/*.tres`)
Each style sets **proportions, rule choices, materials and ornament
probabilities**.

| Style | Walls / Roof | Openings | Signature ornament |
|---|---|---|---|
| **Gothic** | Dark granite ashlar, black slate (steep 55–65°) | Pointed lancets, tracery, rose windows | Flying buttresses, gargoyles, pinnacles, crockets |
| **High Fantasy** | Pale limestone, blue or teal tile, gold trim | Round arches, tall slender windows | Slender round towers, conical roofs, banners, crystal finials |
| **Desert** | Sandstone, mudbrick, whitewash | Horseshoe and pointed-horseshoe arches, lattice screens | Domes, parapets, wind-catchers, geometric tile bands |
| **Nordic / Snow** | Fieldstone base, dark timber, shingle roofs (very steep) | Small shuttered windows | Carved ridge beams, snow-laden eaves, icicles |
| **Castle** | Rough-cut grey stone | Arrow slits, murder holes | Crenellations, machicolations, portcullis, round towers |

Mixing styles is a style-sheet field (`fallback_style` plus per-rule weights),
e.g. "Gothic town, 20% Nordic outbuildings" at the edge of the snow biome.

## 5. Age & ruin pass
A building has `age ∈ [0,1]` and `ruin ∈ [0,1]` (from the POI or biome):
- **age** sets the amount of moss and lichen paint, soot near chimneys,
  discoloration bands and chipped corners.
- **ruin** sets subtractive noise erosion that is biased to upper floors and
  roofs. It removes roof spans and leaves rafters, adds rubble scatter at the
  base, and adds collapsed-wall cuts with debris cones. It also adds an
  overgrowth pass (ivy, roots) in wet biomes and sand fill in the desert.
- Ruins keep **gameplay readability**: erosion is masked so that doorways
  and the interior floor nav stay clear.

## 6. Interiors
- Ground-floor interiors are generated for enterable buildings (taverns,
  chapels, smithy, and the cathedral, which becomes the dungeon entrance).
  Partition walls come from BSP splits, and furniture comes from the same
  dressing rules as dungeons.
- Upper floors of non-enterable buildings are solid-hollow (shell only), and
  the cutaway shader never reveals them.

## 7. Performance notes
- A cathedral is about 600–1,200 stamps, and a townhouse about 80–150.
- Stamps are **bounded**: the kernel only rasterizes stamps into the chunks
  they overlap (AABB test), across worker threads.
- **Style preview tool** (`tools/building_preview.tscn`): pick a style, type
  and seed, and see the result in under 200 ms, for rapid art iteration.
