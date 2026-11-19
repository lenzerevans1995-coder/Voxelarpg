# CLAUDE.md — Grave Lantern

Gothic voxel party dungeon crawler. Design docs: `/docs`. Playable prototype:
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
   `SKILL[id]`, `AI[role]`, `BOSS[id]`, `MODEL[kind]`, `ANIM[kind]`,
   `STAMP[id]`, `FLOOR[style]`. To add content, add a `DB` entry and, if
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

## Repo layout

- `docs/` — design plan (01–10). `docs/10-party-and-dungeon-runs.md` is the
  core game loop (loot → lobby → harder dungeon).
- `prototype/index.html` — three.js prototype. Open it in a browser.
