# 03 — Procedural Generation: Overworld & Biomes

The overworld is split into **zones** (about 256 × 256 m each), connected by
road exits, like ARPG act areas. Each zone is generated deterministically from
`hash(world_seed, zone_id)` and follows a **zone recipe** (`data/zones/*.tres`)
that names the biome, the required points of interest (POIs) and the
difficulty.

## Pipeline (each stage writes to a shared `ZoneBlueprint`)

```
1. Recipe & Biome Field  →  2. Terrain (terraced heightfield)  →  3. POI placement
        →  4. Road network  →  5. Lots & Buildings (grammar)  →  6. Scatter (props, flora)
        →  7. Surface detail pass (voxel-level)  →  8. Encounters & nav  →  stamp buffer → kernel
```

### 1. Biome field
- The zone recipe gives a **primary biome**, with optional secondary biomes at
  the edges.
- Two low-frequency noise fields (*wetness* and *decay*) drive **sub-biome
  variation** inside a biome. On a moor, for example: bog ↔ heath ↔ blighted
  ground.
- Biome borders blend over 12–20 m by dithering material choices and mixing
  scatter tables (no hard seams).

### 2. Terrain: terraced 2.5D heightfield
ARPGs need readable, mostly walkable ground. Terrain is a heightfield
**quantized into tiers** (1.2 m steps by default) with cliffs between tiers:

```
h(x,z) = fbm(x,z) * amplitude
tier   = floor(h / TIER_HEIGHT)            # walkable plateaus
cliffs = tier changes, carved with biome "strata" materials (layered rock bands)
ramps  = placed where roads cross tiers (smoothstep slope over 4–6 m)
```
- **Cliff faces** get strata bands, overhang noise (±2 voxels) and root or
  icicle stamps, depending on the biome.
- Below the surface there are 3–6 m of soil or rock, then a "bedrock" uniform
  chunk. Uniform chunks are free.
- Water: a per-zone water level. Basins below it fill with a transparent
  water material, which becomes ice in snow biomes.

### 3. POI placement
- The recipe lists POIs with constraints, e.g.
  `{type: graveyard, count: 1, min_dist_from_entry: 60, tier: any, footprint: 30x24}`.
- Placement uses **Poisson-disk candidates**, scored by constraint
  satisfaction (flatness, distance, tier). The best score wins, with seeded
  tie-breaks.
- Typical POIs: dungeon entrance, waypoint shrine, ruined chapel, camp, bandit
  or cultist hideout, gallows crossroads, event arena, lore stone.

### 4. Road network
- Connect zone exits and POIs using a **minimum spanning tree plus 1–2 extra
  edges** for loops. ARPG players like at least one loop.
- Route each edge with **A\*** on a 2 m grid. Cost goes up with slope, water
  and tier changes, and down near existing roads, so roads merge naturally.
- Carve roads 3–5 m wide: cobbles in towns, packed dirt and ruts in the wild,
  sand-buried flagstones in the desert, packed snow with wheel tracks in snow
  biomes. Bridges are placed where a road crosses water or a tier gap
  (a stamp from the building grammar, `bridge_stone` or `bridge_wood`).

### 5. Lots & buildings
- Settlements split their area into **lots along roads** (frontage first, then
  recursive splitting of the remaining area).
- Each lot gets a building type from the settlement's style mix, built by the
  shape grammar ([05-building-generation.md](05-building-generation.md)).
- Lone structures (chapels, watchtowers, mausoleums) are placed as POIs.

### 6. Scatter
- **Poisson-disk scatter per layer**: large props (trees, menhirs), medium
  (gravestones, carts), small (bones, candles, mushrooms), and ground cover
  (grass tufts, pebbles).
- Each scatter entry has density, a slope limit, distance to road, a
  "near-POI" boost, and a cluster parameter (e.g. gravestones grow in rows
  inside graveyards).
- Big props are **procedural voxel stamps**: dead trees use space-colonization
  branching, rocks are noise-displaced SDF blobs, gravestones come from a small
  grammar (slab + cap shape + cross/angel + crack).

### 7. Surface detail pass (where tiny voxels pay off)
Runs inside the kernel as cheap per-voxel rules on the final surface:
- **Top faces:** snow cover (with 1–3 voxel drift heights), sand drifts, moss
  on stone in wet areas, leaf litter.
- **North-facing and overhanging faces:** moss and lichen, icicles in snow.
- **Cracks and wear:** noise-masked darkening, chipped edges on stone corners,
  where 30% of corner voxels are removed.
- **Story decals:** blood splatter, scorch marks and wax drips near candles,
  as 2D masks projected onto voxels.

### 8. Encounters & navigation
- A navmesh comes from the walkable top surface of tiers and ramps
  (Godot `NavigationRegion3D`, baked per zone from a simplified collision
  mesh, not the voxel mesh).
- Encounter packs are placed with a density budget per 1,000 m², more on
  roads and near POIs. They are drawn from the biome's monster table, with
  elite and champion rolls.

## Biome sheets (data: `data/biomes/*.tres`)

| Biome | Ground & strata | Flora / scatter | Architecture style | Light & weather | Monsters (example) |
|---|---|---|---|---|---|
| **Weeping Moor** (Act I) | Peat, heather, mossy granite | Dead oaks, yews, gravestones, gibbets, will-o-wisps | Gothic village, ruined chapels | Low moon, blue fog, drizzle | Skeletons, ghouls, crows |
| **Blighted Forest** | Black loam, fungal rot | Twisted pines, giant fungi, webs | Woodcutter huts, witch shacks | Green-tinted fog, spores | Rot-hounds, spiders, Charnel Golem |
| **Ashen Desert** | Sand, sandstone strata, salt flats | Dead palms, bones of colossi, cairns | Desert: domes, flat roofs, courtyard houses, sunken tombs | Harsh sun, heat haze, sandstorms | Mummified dead, sand wyrms |
| **Frostspine Mountains** | Snow, ice, slate | Frozen pines, ice spikes, frozen corpses | Nordic timber halls, mountain castles | Overcast, falling snow, blizzards | Frost skeletons, ice wraiths |
| **Drowned Fen** | Mud, reeds, rotten boardwalks | Mangrove-like roots, sunken coffins | Stilt houses, sunken abbey | Thick fog, rain, fireflies | Drowned dead, leeches |

Each sheet also sets: palette overrides, the fog color curve, ambient audio,
music stems, the snow/sand/moss rules for the detail pass, and the dungeon
themes it can spawn.

## Example zone recipe (sketch)
```gdscript
# data/zones/act1_weeping_moor.tres (shown as dict for readability)
{
  "biome": "weeping_moor",
  "size_m": Vector2(256, 256),
  "exits": [{"edge": "south", "to": "morrowgate"}, {"edge": "north", "to": "act1_blighted_wood"}],
  "pois": [
    {"type": "graveyard_old",    "count": 1, "min_dist_from": {"exit_south": 60}},
    {"type": "crossroads_gallows","count": 1, "on_road": true},
    {"type": "cathedral_ruin",   "count": 1, "contains": "dungeon:undercroft_st_vael", "max_dist_from": {"exit_north": 80}},
    {"type": "waypoint_shrine",  "count": 1, "on_road": true},
    {"type": "farmstead_burned", "count": [1, 3]}
  ],
  "monster_density": 1.0,
  "level_range": [1, 4]
}
```
