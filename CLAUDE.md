# CLAUDE.md — Grave Lantern

Gothic voxel dungeon crawler, playable solo (companions optional), with a
generated overworld around the dungeons: zones, gathering, crafting (`docs/12`).
Design docs: `/docs`. Playable prototype:
`/prototype/index.html` (three.js r128, single file, no build step).

## Coding rules (MANDATORY — apply in every session, never drift)

1. **Simple and concise.** Write the least code that works. No speculative
   abstractions, no class hierarchies, no frameworks.
2. **Few variables.** When values belong together, use one object or array
   instead of many loose variables (e.g. `S.fx.tele`, not `teleList`,
   `teleCount`, `teleMeshes`).
3. **One data registry.** All tunables and content (classes, skills, enemies,
   bosses, dungeons/biomes, palettes, loot, omens) live in `CFG` (numbers) and
   `DB` (content) at the top of the file. Logic reads from them. No magic
   numbers scattered through the code.
4. **One state object.** All mutable game state lives in `S`. Three.js handles
   live in `R`.
5. **Registries keyed by id.** Behavior is looked up by id:
   `SKILL[id]`, `PERK[id]`, `AI[role]`, `BOSS[id]`, `ARENA[id]`, `ACT[id]`, `MODEL[kind]`,
   `ANIM[kind]`, `HELD[id]`, `STAMP[id]`, `PLACE[mode]`, `SHAPE[id]`,
   `PATH[id]`, `FLOOR[style]`, `WALL[style]`, `ZONE[shape]`, `PANEL[id]`.
   Data-only tables looked up by id: `DB.ZONES`, `DB.NODES`, `DB.TRADES`,
   `DB.RECIPES`, `DB.PERKS`. To add content, add a `DB` entry and, if
   needed, one registry function. Do not add `if/else` chains.
6. **Ids are consistent.** The same short lowercase id is used in `DB`, the
   registries, save data and the UI (e.g. `golem`, `templar`, `undercroft`).
7. **Code map.** Every source file starts with a `CODE MAP` comment listing
   its sections as `§N NAME`, and every section begins with `// §N NAME`.
   Find code by grepping `§N`. Keep the map current when you add or move
   sections.
8. **Prefer tables and loops** over repeated code. Short functions.
9. **Extend, don't duplicate.** Before adding a new structure, check whether an
   existing registry or `DB` table already fits.
10. **Portable to GDScript.** Plain data (dictionaries/arrays) and plain
    functions only: no classes with inheritance, no closures holding hidden
    state, no DOM or three.js calls outside §14–§16 and the `R` handles. See
    `docs/11-optimization-seeding-porting.md` for the port map, the seeding
    rules and the optimization plan.

## Repo layout

- `docs/` — design plan (01–10). `docs/10-party-and-dungeon-runs.md` is the
  core game loop (loot → lobby → harder dungeon).
- `docs/12-overworld-and-skills.md` — solo play and the staged overworld
  plan: zones, gathering, crafting, trade skills.
- `docs/11-optimization-seeding-porting.md` — optimization plan, seeding
  rules, GDScript port map.
- `prototype/index.html` — three.js prototype. Open it in a browser.
