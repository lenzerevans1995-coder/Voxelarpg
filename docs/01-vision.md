# 01 — Vision & Pillars

## Elevator pitch
A dark, candle-lit Gothic **party dungeon crawler** with classic isometric
ARPG combat. Form a group of up to 5 (AI companions, friends, or a mix),
take a Reliquary Sigil into a procedurally generated dungeon, beat the
timer, get loot, return to the lobby, and push a harder Depth. Dungeons
span every biome, from cathedral crypts and sand tombs to frozen citadels
and drowned abbeys, and each has its own enemies.

Every cathedral spire, crypt, gravestone and blade is made of tiny
voxels, so the world can be generated, dressed, scorched and smashed at a
fine level of detail, while staying light enough to build and iterate in
seconds.

## Pillars

1. **Readable darkness.** Gothic mood (fog, candlelight, moonlight, decay), but
   the combat silhouette is always readable from the isometric camera. Enemies
   get rim light and team-colored emissive accents. Walls between the camera
   and the player fade out with a dither.
2. **Tiny voxels, big detail.** At 10 cm world voxels you can see tracery in a
   rose window, mortar lines, cracked flagstones and moss on north faces.
   Characters use 5 cm voxels, which is enough for visor slits, ribs, rivets
   and runes.
3. **Everything is generated, nothing is generic.** Data-driven grammars and
   biome sheets produce places that feel authored: a graveyard always has a
   lych-gate and a yew, and a cathedral always faces east.
4. **Fast to build, fast to change.** Edit a style sheet, press a key, and the
   zone regenerates in under 2 seconds. No long bakes and no huge asset
   pipeline.
5. **Physical, voxel-native combat.** Hits chip voxels off enemies, skeletons
   come apart bone by bone, the Charnel Golem sloughs off flesh, and barrels,
   crates and tombstones shatter.
6. **Party first.** Every system assumes a group of up to 5: tank, healer,
   damage. AI companions are first-class, so solo players get the full
   experience, and friends can drop in to replace them.

## Core loop: Loot → Lobby → Harder Dungeon
```
Lobby (Morrowgate: vendors, crafting, party board, dungeon map)
  → Pick dungeon + Depth (difficulty) + party (AI / friends)
    → Timed run: packs, 3–4 bosses, rotating Omens (modifiers)
      → Loot (personal drops + end chest, themed per dungeon, visible on the voxel model)
        → Back to lobby → gear up → higher Depth
```
See [10-party-and-dungeon-runs.md](10-party-and-dungeon-runs.md) for the full
system.

## Scope guard (what we are *not* building yet)
- No free camera rotation (fixed isometric, with optional 90° snaps later).
- No fully destructible terrain. Destruction is limited to props and a
  "fragile" material flag so performance stays predictable.
- Online co-op is **in scope** (up to 5, host-authoritative). There is no
  MMO-scale persistence and no open-world PvP.
- Vertical slice: **3 classes**, one per role (Templar tank, Sister of the
  Pale Flame healer, Reaver damage). The class system is data-driven so
  more can be added.
- The overworld is optional side content, not the main loop.
