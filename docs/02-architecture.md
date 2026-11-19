# 02 — Technical Architecture

## 1. Engine choice

**Recommendation: Godot 4.x (latest stable 4.x at project start).**

| Criterion | Godot 4 | Unity 6 |
|---|---|---|
| Editor/download size | ~120 MB, opens in seconds | Several GB, slower domain reloads |
| Iteration | GDScript hot-reload, no compile step | C# recompile + domain reload per change |
| Native speed path | GDExtension (C++) | Burst + Jobs (excellent) |
| Licensing | MIT, no runtime fees | Proprietary tiers |
| Voxel suitability | ArrayMesh + WorkerThreadPool is sufficient | Mesh API + Jobs is excellent |

Godot matches the "lightweight, quick to iterate" requirement. To keep the
Unity option open, the **voxel kernel is a plain C++17 library with no engine
headers**, wrapped by a thin GDExtension binding. A Unity port would need only
a native-plugin wrapper (P/Invoke) plus a rewrite of the gameplay layer.

### Language split
| Layer | Language | Why |
|---|---|---|
| Voxel kernel (storage, rasterizing stamps, meshing, ambient occlusion, raycasts) | C++ (GDExtension) | Hot loops over millions of voxels. Small (~3–4k lines) and changes rarely. |
| World generation logic (graphs, grammars, layouts) | GDScript (typed) | Changes daily, so it needs hot-reload. Emits *stamps*, never touches voxels in loops. |
| Gameplay, AI, UI, items | GDScript (typed) | Iteration speed |
| Shaders | Godot shading language | Palette lookup, dither fade, wind, wetness |

> **Key idea: the Voxel Command Buffer.** GDScript decides *what* to build
> ("box of slate 4×0.3×6 m here", "pointed arch window here", "erode with noise
> 0.3"). It appends compact **stamp commands** to a buffer, and the C++ kernel
> rasterizes the whole buffer into chunks on worker threads. Generators stay
> readable and hot-reloadable, and the per-voxel work stays native.

## 2. Voxel scale & units

All gameplay and generation code works in **meters**. Voxel resolution is
applied only at bake or rasterize time.

```gdscript
# game/core/units.gd
const WORLD_VOXELS_PER_METER  := 10   # 10 cm world voxels
const ENTITY_VOXELS_PER_METER := 20   # 5 cm character/weapon voxels
const CHUNK_SIZE := 32                # voxels per chunk edge → 3.2 m chunks
```

| Thing | Real size | World voxels (10 cm) | Entity voxels (5 cm) |
|---|---|---|---|
| Chunk | 3.2 m | 32 | — |
| Dungeon cell | 4.0 m | 40 | — |
| Door | 1.2 × 2.4 m | 12 × 24 | — |
| Gothic window | 1.0 × 3.0 m | 10 × 30 (room for a pointed arch + mullion) | — |
| Human | 1.8 m | — | 36 tall |
| Longsword | 1.1 m | — | 22 long, 1 thick, 1–2 wide |

**Why two resolutions?** Entities are close to the camera and read best with
detail. Terrain covers huge areas, and halving its resolution means 8× less
data. Because content is authored as metric shapes, changing either constant
and re-baking re-scales the whole game. That is what "easily scaled" means
here. Hand-made `.vox` hero assets are the one exception: they are tied to
their authored resolution, and the importer resamples them (nearest-neighbor)
if the constant changes.

## 3. Voxel data model

```
Voxel      = uint16 material id (0 = air)
Chunk      = 32³ voxels, stored as:
               - palette (≤ 256 local materials) + 4- or 8-bit index array, or
               - "uniform" flag (all air / all stone) → 0 bytes of voxel data
Zone       = sparse hash map ChunkCoord → Chunk; generated lazily from seed
Persistence= only the *diff* against generation is saved (destroyed props, opened doors)
```

**Material table** (`data/materials/*.tres`): base color plus 2–4 variant
colors, roughness, emissive, `fragile`, `transparent`, footstep sound, and hit
particle color. Per-voxel color variation comes from a **hash of the world
position in the shader**, so it costs no memory. Stone looks mottled and wood
shows grain stripes along a material axis.

Memory estimate for a 256 × 256 m zone with 40 m of vertical range:
80 × 80 × 13 ≈ 83k chunks. Roughly 85% are uniform (air or solid), which
leaves ~12k mixed chunks × ~16 KB ≈ **200 MB worst case**. In practice only
chunks near the camera are kept in memory (see streaming), so the working set
is about 30–60 MB.

## 4. Meshing & rendering

- **Greedy meshing** per chunk, with a separate pass for transparent
  materials (stained glass, ice, water).
- **Per-vertex ambient occlusion** (the classic 3-neighbor rule). Faces only
  merge when their AO values match, which keeps the soft occlusion that makes
  tiny voxels look "detailed".
- Vertex format: position (packed), normal index, **material id** and AO byte.
  One opaque `ShaderMaterial` for the whole world, which looks material color
  up in a **palette texture**. Result: **1 draw call per chunk**, no texture
  atlases, and recoloring is just a palette change.
- **Camera-occlusion cutaway.** Chunks above the player's floor level, and
  wall voxels inside a capsule from the camera to the player, fade out with a
  screen-space dither driven by a shader uniform, so no remeshing is needed.
- **Lighting.** One directional light (moon/sun) with shadows, plus clustered
  omni lights for torches and candles (cap of ~32 visible). Emissive voxels
  (candle flames, runes, lava) glow through bloom. Volumetric fog is on in
  Gothic biomes. No global illumination bake. Vertex AO plus fog does the mood
  work.
- **LOD.** The fixed camera sees roughly 36 × 24 m, so there is no need for
  far LODs. A cheap 2× downsampled mesh is used only for the overworld
  minimap and the loading-screen flyover.

### Streaming
- Visible footprint ≈ 12 × 8 chunk columns. Keep a **ring of 2 chunks** around
  that in memory, meshed.
- Generation and meshing jobs go through `WorkerThreadPool`, prioritized by
  distance to the player along the movement direction.
- Dungeons are small enough (≤ 120 × 120 m per floor) to generate fully
  during the floor transition.

## 5. Performance budgets (target: integrated GPU laptop, 1080p, 60 fps)

| Budget | Target |
|---|---|
| Frame (CPU main thread) | ≤ 8 ms |
| Draw calls | ≤ 1,200 |
| Chunk rasterize + mesh (one 32³ chunk, one thread) | ≤ 1.5 ms |
| Full zone gen (layout + stamps, not meshing) | ≤ 1.5 s |
| Dungeon floor gen + mesh | ≤ 2.0 s |
| Active actors | 5 party + 60 enemies (AI ticks at 10 Hz, animation at frame rate) |
| Network | ≤ 64 KB/s per client at 20 Hz snapshots |
| RAM | ≤ 1.5 GB |
| Export size (vertical slice) | ≤ 150 MB |

## 6. Entities (characters, enemies, weapons)

- **Rigid-part voxel models.** Each bone is its own small voxel mesh (head,
  torso, upper arm…) parented to a `Node3D` pivot. Godot `AnimationPlayer`
  animates the pivots. No skinning: cheap, crisp, and it *looks* voxel.
- **Sources:**
  1. **SDF recipes** (`.tres` lists of capsules, boxes and ellipsoids plus
     material, in meters), baked to voxels at import. These re-scale for free.
  2. **MagicaVoxel `.vox`** for hero assets. A custom `EditorImportPlugin`
     reads the scene graph, builds one mesh per named model and uses the
     model's transform as its pivot.
- **Sockets** on the rig: `hand_r`, `hand_l`, `back`, `hip_l`, `head_top`,
  `vfx_core`. Weapons and armor attach to sockets.
- **Hit and death feedback.** A hit spawns voxel "chips" from the surface,
  using GPU particles colored by the struck material. On death, the parts
  become `RigidBody3D` pieces (skeletons fall apart), then fade after 8
  seconds.

## 7. Project layout

```
/addons/voxel_kernel/        # GDExtension binary + .gdextension
/native/voxel_kernel/        # C++ source (engine-agnostic core + godot binding)
    core/   chunk.hpp, palette.hpp, mesher_greedy.cpp, stamps.cpp, sdf.cpp
    godot/  register_types.cpp, voxel_world_node.cpp
/game/
  core/       units.gd, rng.gd (seeded PCG), events.gd (signal bus), service_locator.gd
  voxel/      voxel_world.gd, chunk_streamer.gd, stamp_buffer.gd
  worldgen/
    overworld/  zone_generator.gd, terrain.gd, roads.gd, poi_placer.gd, scatter.gd
    dungeon/    mission_graph.gd, layout_embedder.gd, kit_builder.gd, dresser.gd
    buildings/  grammar.gd, rules/*.gd, styles/ (resources)
  entities/   actor.gd, rig.gd, stats.gd, hitbox.gd
  ai/         brain.gd, behaviors/*.gd
  combat/     damage.gd, threat.gd, skills/*.gd, status_effects/*.gd, mechanics/*.gd
  party/      party.gd, companion_brain.gd, roles/{tank,healer,damage}_brain.gd, commands.gd
  runs/       sigil.gd, run_state.gd (timer, dread, deaths), omens/*.gd, leaderboard.gd
  net/        net_session.gd, snapshot.gd, online_service.gd
  lobby/      lobby.gd, reliquary_table.gd, party_board.gd
  items/      item.gd, affixes.gd, loot_tables.gd, weapon_generator.gd
  ui/
/data/        # ALL tunables, as Godot Resources (.tres): biomes, styles, enemies, items, materials
/assets/      # .vox hero models, audio, fonts
/tests/       # GdUnit4 suites + golden-seed snapshots
/tools/       # seed viewer scene, palette editor, profiling scenes
```

## 8. Code quality & maintainability rules

1. **Static typing everywhere** in GDScript (`var hp: int`, typed arrays),
   with warnings set to treat untyped declarations as errors.
2. **Formatting and lint:** `gdformat` and `gdlint` (gdtoolkit) run in a
   pre-commit hook and in CI. `clang-format` runs on C++ code.
3. **Data, not code, for content.** No magic numbers in scripts. Tunables
   live in `/data` Resources, so a designer can add a biome without writing
   code.
4. **Determinism.** Every generator takes a `SeededRng` derived from
   `hash(world_seed, zone_id, purpose)`. Never use the global `randi()`.
5. **Small composable nodes.** An actor is `Actor` plus components
   (`Health`, `Mover`, `Brain`, `Rig`, `Hurtbox`). Components talk through
   signals. Autoloads are limited to `Events`, `Services` and `Settings`.
6. **Tests.** GdUnit4 unit tests for graph and grammar logic, and
   **golden-seed snapshot tests**: generate seed 1234 and hash the stamp
   buffer. A changed hash makes CI fail until the snapshot is intentionally
   updated, which catches accidental generation changes.
7. **CI (GitHub Actions):** build the GDExtension for Linux, Windows and
   macOS, run headless tests (`godot --headless --script`), run lint, and
   export a Linux build artifact.
8. **Debug tooling is part of the build.** An F-key overlay shows chunk
   bounds, the mission graph, nav cells and the seed, and has a
   "regenerate zone" hotkey.

## 9. Networking (online co-op, up to 5)

- **Host-authoritative listen server** on Godot `MultiplayerAPI` + ENet. A
  dedicated server is the same project exported `--headless`.
- **Layouts are never sent.** Clients receive
  `(dungeon_id, seed, depth, omens)` and run the same deterministic
  generators. Only voxel *edit events* and actor state cross the wire.
- Actor snapshots at 20 Hz with interpolation, client-side prediction for
  the local player's movement and skill start, and server-side hit
  validation.
- AI companions and enemies run **only on the host**. A companion is just
  an actor whose input comes from a brain instead of a peer, so swapping a
  friend in for an AI mid-lobby is just a change of input source.
- `OnlineService` interface (lobbies, invites, friends) with an
  ENet/IP-code implementation first and a platform backend (e.g. Steam)
  later.
- **CI desync test:** two headless peers generate the same seed and run a
  scripted 60 s fight. Chunk hashes and final actor states must match.

## 10. Unity mapping (if switching later)

| Godot | Unity equivalent |
|---|---|
| GDExtension voxel kernel | Native plugin (same C++ core) or port the mesher to Burst |
| `WorkerThreadPool` | Job System |
| Resources (`.tres`) | ScriptableObjects |
| `AnimationPlayer` on pivots | Animator + rigid-part hierarchy |
| Signals / `Events` autoload | C# events / ScriptableObject event channels |
| GdUnit4 | Unity Test Framework |
