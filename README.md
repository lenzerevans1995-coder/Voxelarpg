# Grave Lantern — Gothic Voxel Party Dungeon Crawler (Design Plan)

> Working title. A Gothic **party dungeon crawler** with Diablo-style
> isometric combat and mythic-dungeon-style scaling runs. Form a party of up
> to 5 (AI companions and/or friends), beat timed, procedurally generated
> dungeons across many biomes, each with its own enemies, get loot, return to
> the lobby, and push harder. Everything is made of **tiny, detailed voxels**
> and **generated from data-driven rules**.

**Core loop:** Loot → Lobby → Harder Dungeon.

This repo currently holds the **design and technical plan**. The code
scaffold follows the roadmap in [`docs/08-roadmap.md`](docs/08-roadmap.md).

## Plan at a glance

| Topic | Decision |
|---|---|
| Engine | **Godot 4.x** (GDScript for game logic, a small C++ GDExtension for the voxel kernel). Unity mapping in the architecture doc. |
| Voxel scale | **World 10 cm/voxel**, **characters, enemies and weapons 5 cm/voxel**. Both are config constants. |
| Scaling | Content is authored **in meters** (shapes, grammars, SDFs) and *baked* into voxels, so resolution and world size can change without re-authoring. |
| Game structure | Lobby hub (Morrowgate) → pick dungeon + **Depth** (difficulty tier) → party run with timer, **Dread** meter (enemy-kill progress) and weekly **Omens** (modifiers) → loot → repeat |
| Party | Up to 5. Tank / healer / damage roles. **AI companions** fill any slot. **Online co-op** (host-authoritative). |
| Camera | Fixed 3/4 isometric perspective, like classic ARPGs. That keeps chunk streaming and level-of-detail (LOD) work small. |
| World | **Dungeons** (built graph-first, then dressed with kit pieces) across 7 biome themes. They mix indoor and outdoor segments using the overworld terrain generator. Open overworld zones are optional side content. |
| Buildings | A **shape grammar** with style sheets: Gothic, High Fantasy, Desert, Nordic/Snow, Castle, plus a ruin/weathering pass. |
| Biomes / dungeons | Gothic cathedral, Night moor, Ashen desert tomb, Snowy mountain citadel, Drowned fen abbey, Blighted forest, High-fantasy mage tower. Each has a **unique enemy family** and its own bosses. |
| Enemies (first three) | **M:** Skeleton · **L:** Charnel Golem (rotting flesh ogre) · **XL:** The Hollow Rider (headless horseman) |
| First level | Morrowgate lobby → **Undercroft of St. Vael** (first party dungeon: 3 wings, 3 bosses). Optional: Weeping Moor zone with the Hollow Rider world boss. |

## Documents

1. [Vision & Pillars](docs/01-vision.md)
2. [Technical Architecture](docs/02-architecture.md): engine, voxel kernel, scale, performance budgets, code standards
3. [Procedural Generation: Overworld & Biomes](docs/03-overworld-generation.md)
4. [Procedural Generation: Dungeons](docs/04-dungeon-generation.md)
5. [Procedural Buildings (Gothic / High Fantasy / Desert / Snow / Castles)](docs/05-building-generation.md)
6. [Characters & Weapons](docs/06-characters-and-weapons.md)
7. [Enemies: Skeleton, Charnel Golem, Hollow Rider](docs/07-enemies.md)
8. [Roadmap & Milestones](docs/08-roadmap.md)
9. [First Level Layout](docs/09-first-level.md)
10. [Party Play, Scaling Dungeon Runs, Lobby, Dungeon Roster & Loot](docs/10-party-and-dungeon-runs.md) ← **core game structure**
