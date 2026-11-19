# 08 — Roadmap & Milestones

Each milestone ends with something playable. Durations assume a small team
(1–3 developers). Each milestone has an **exit check** that must pass before
moving on.

| # | Milestone | Scope | Exit check |
|---|---|---|---|
| **M0** | Skeleton project (1–2 wk) | Godot project, folder layout, CI (lint, tests, GDExtension build on 3 OSes), `units.gd`, seeded RNG, debug overlay | CI green; `godot --headless` test run < 30 s |
| **M1** | Voxel kernel (3–4 wk) | Chunk storage (palette + uniform), stamp rasterizer (box/cylinder/arch/lathe/SDF/erode), greedy mesher + AO, palette shader, chunk streaming, cutaway dither | 32³ chunk rasterize + mesh ≤ 1.5 ms; a 60 × 60 m test room at 60 fps on an integrated GPU |
| **M2** | Dungeon generator v1 (3–4 wk) | Mission graph, cell embedding, Gothic crypt kit, dressing, lights, validation, golden-seed tests | 100 random seeds all validate; floor generation ≤ 2 s |
| **M3** | Combat core (3–4 wk) | Templar playable: movement, 4 skills, hit chips, damage/threat, Skeleton enemy (warrior/archer), death collapse | Fight a skeleton pack in a generated crypt and have it feel good |
| **M4** | Party & AI companions (4–5 wk) | Healer + Reaver classes, party framework, Tank/Healer/Damage brains, mechanic vocabulary, commands wheel | A full AI party clears a 3-pack room with no player input other than "follow" |
| **M5** | Run system + lobby (3 wk) | Sigils, Depth scaling, timer, Dread, deaths penalty, end chest, Morrowgate lobby (Reliquary Table, Party Board, vendors) | Loot → lobby → harder run loop works end to end |
| **M6** | Online co-op (4–5 wk) | ENet host-authoritative, snapshots, prediction, seed-sync, AI swap-in/out, invite codes | 3 humans + 2 AI clear a run with no desync; CI desync test green |
| **M7** | Vertical slice: Undercroft of St. Vael (5–6 wk) | Cathedral nave segment (building grammar), Crypts, Ossuary, 3 bosses incl. **Charnel Golem**, Restless Dead family, loot tables, 2 Omens | Playtest: 5 runs Depth 0→5, median clear 25 ± 4 min |
| **M8** | Biome expansion (ongoing) | Hollow Road Barrow (+ **Hollow Rider**, outdoor segments), then Sepulchre of Ka-Zeth (desert), Frostspire Citadel (snow), etc. Each = kit + enemy family + bosses + style sheet | Each new dungeon ships with no new engine code (data + behaviors only) |
| **M9** | Weekly systems | Weekly seeds (Charted), Omen rotation, leaderboards, weekly vault | Season dry-run over 4 simulated weeks |

## Risks & mitigations

| Risk | Mitigation |
|---|---|
| Voxel meshing too slow in volume | Native kernel from day one; uniform-chunk shortcut; profile scene in `/tools` checked in CI |
| Procedural dungeons feel samey | Authored *structure* (recipes, boss slots, vignettes) + procedural *layout*; per-dungeon kits and enemy families |
| AI companions feel dumb | Shared mechanic vocabulary means one data source; "AI-only clear" automated test at fixed Depth per dungeon |
| Netcode complexity | Determinism + seed sync removes world-state sync; only actors are replicated; desync test in CI |
| Content treadmill | Every new dungeon must be data + behavior resources only (M8 exit check) |
| Scope creep | Overworld stays optional; new classes only after the 3-role slice is fun |
