# 11 — Optimization, seeding and the GDScript port

A plan, not a spec. The prototype (`prototype/index.html`) is the reference
implementation; everything here keeps its shape: `CFG` numbers, `DB` content,
`S` state, registries keyed by id.

## 1. Seeding

**Rule: one seed in, the same dungeon and the same fight out.**

| Stream | Seeded today | Plan |
|---|---|---|
| Layout, room shapes, corridors, props, packs (`genDungeon`, `startRun`) | Yes — `rng(seed)`, plus `hash3`/`noise2` which are pure functions of position | Keep. Never call `Math.random` in §6. |
| Charted mode | Yes — `hashStr(dungeon id) ^ week()` | Keep; show the seed in the Stats window (done). |
| Loot (`makeItem`) | No — `Math.random` | Give the run a second stream `S.run.rngLoot = rng(seed ^ 0x9E3779B9)` so loot does not shift when layout code changes. |
| Combat (boss target picks, arena mechanics, crits) | No | Third stream `S.run.rngFight`. Needed for replays, co-op lockstep and fair leaderboards. |
| Cosmetics (chips, embers, camera shake, idle sway) | No | Leave unseeded on purpose. They must never feed back into gameplay state. |

Steps: (1) add the two streams to `S.run`; (2) replace gameplay `Math.random`
calls in §8–§11 and §13 with them (grep `Math.random`, about a dozen sites);
(3) add a debug key that prints a state hash (party hp + foe hp + positions
rounded to cm) each second so two runs of one seed can be diffed.

Keep the stream split fixed forever: **layout / loot / fight**. Adding a draw to
one stream then never disturbs the others.

## 2. Optimization

Measured cost centres, in order, with the cheapest fix first.

1. **Dungeon build (one-off, ~1–2 s).** `genDungeon` fills a 520×52×520 grid
   and `meshVox` greedy-meshes it on the main thread.
   - Mesh only chunks that contain open floor or walls within `wallT` (skip the
     empty majority). Expected 3–5× faster.
   - Move gen + meshing to a Worker (web) / `WorkerThreadPool` (Godot); the
     functions are already pure: seed + `DB` entry in, arrays out.
   - Cache the last built dungeon by seed so a wipe/retry is instant.
2. **Model baking.** `bake()` voxelizes SDF primitives per unique key and caches
   in `BAKE`. Gear changes re-bake only the touched slot because every node has
   its own key. Plan: pre-bake at load behind the loading label, and in Godot
   save baked meshes as `.res` so shipping builds never bake.
3. **Draw calls.** One mesh per rig node (≈14 per humanoid). Fine for ~40
   actors. If packs grow: merge static nodes per limb, or switch to
   `MultiMesh`/instancing for trash that shares a bake key.
4. **Lights.** Fixed pool of `CFG.lights` point lights re-assigned to the
   nearest sources every 0.25 s; halos are sprites, not lights. Keep the pool;
   never add per-prop lights.
5. **Per-frame allocations.** `updateActors`/`animate` build `[...party,...foes]`
   each frame and telegraphs create a material each. Plan: keep one `S.all`
   list updated on spawn/kill; pool telegraph decals and ring meshes.
6. **Collision is O(n²)** over actors. Fine under ~60. Past that, bucket by nav
   cell (the grid already exists in `S.nav`).
7. **HUD.** The orb ray-cast is 2 × 120² pixels × ≤26 steps per frame. Halve
   the rate (update on odd frames) or move it to a shader (`orb()` is written
   as plain loops for exactly that). The minimap only repaints newly seen
   cells into its backing image.
8. **Pathfinding.** A* per actor with a 0.6 s refresh and an 8000-node cap.
   Plan: one flow field toward the player per nav change for followers.

Budget targets: 60 fps with 40 actors on integrated graphics; dungeon build
under 0.5 s; zero allocations per frame in §8–§12 once pooled.

## 3. GDScript port map

The code is written to translate line-for-line: dictionaries and arrays for
data, dictionaries of functions for behaviour, no inheritance.

| Prototype | Godot 4 |
|---|---|
| `CFG`, `DB` | `const` dictionaries in one autoload (`Data.gd`), or `.tres` resources per table later |
| `S` | autoload `State.gd` holding one dictionary |
| `R` | node references (`@onready`) on the main scene |
| Registries `SKILL`, `AI`, `BOSS`, `ARENA`, `ANIM`, `STAMP`, `PLACE`, `SHAPE`, `PATH`, `FLOOR`, `WALL`, `ZONE`, `HELD`, `PANEL` | `Dictionary` of `Callable`, same ids |
| `rng(seed)` | `RandomNumberGenerator` with `seed` set (wrap it so `range/int/pick/chance` keep their names) |
| `hash3`, `noise2`, `noise3` | port as-is (integer math); do **not** swap for `FastNoiseLite` or seeds stop matching the prototype |
| `Vox` + `meshVox` | `PackedByteArray` + `SurfaceTool`/`ArrayMesh` per chunk |
| `bake()` SDF primitives | same loops into an `ArrayMesh`; cache as resources |
| Rig nodes (`legL` → `shinL`, `armL` → `foreL` → `handL`) | `Node3D` tree with the same names; `POSE` easing is `rotation = rotation.lerp(target, k)` |
| `DB.SWINGS` keyframes | keep as arrays and `key()`; or import into `AnimationPlayer` tracks later |
| `tele()` + `GLSL` decal | `MeshInstance3D` quad with a `ShaderMaterial`; the fragment code ports almost verbatim |
| `chips()` | one `MultiMeshInstance3D` (same idea as the instanced mesh) |
| HUD (§14) | `Control` nodes; `--vx` border = `StyleBoxTexture` nine-patch; `orb()` = canvas-item shader |
| `updateActors/updateFx/updateRun/animate` | called in that order from one `_physics_process` |

Port order: Data → RNG/noise (verify with seed hashes against the prototype)
→ Vox/meshing → rigs/bake → nav → actors/AI → FX → HUD.

Rules that keep the port cheap (also in `CLAUDE.md`):

- No JS-only tricks in gameplay code: no getters, no prototype chains, no
  `setTimeout` for game logic (use timers in `S`), no closures that hide state.
- Engine calls stay in §14–§16 and behind `R`.
- Every tunable is a number in `CFG` or a row in `DB`.
