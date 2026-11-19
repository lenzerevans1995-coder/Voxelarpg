# 12 — Solo play, overworld, gathering and skills

**Not a change of direction.** Grave Lantern is still a dungeon crawler; this
doc extends it outward: the country around the dungeons is generated too, you
can play it alone, and you gather and craft between descents. Companions are
optional rather than assumed.

Built in stages. Each stage is playable on its own.

## Stage 1 — combat foundation (done in the prototype)

- Solo by default; five classes (`templar`, `sister`, `reaver`, `ranger`,
  `warlock`). Alone: foes are weaker (`CFG.solo`), slain foes drop health orbs,
  bosses shed orbs as they lose life, you mend out of combat, and a boss keeps
  its wounds when you die.
- Controls: WASD or hold left click to move; hold left click on a foe to attack
  it; you also strike whatever comes into reach; Space attacks toward the
  cursor; 1–4 skills. No right click. Hovered foes glow and get a bracket.
- Swings chain through `DB.CLASSES[cls].atk.sw` (e.g. slash → backhand → chop)
  and fill the whole attack cycle. Bosses chain their own lists.
- Level-ups offer one boon of three (`DB.PERKS`, behaviour in `PERK[id]`).
  Looted weapons carry a boon affix (`DB.WPERKS`), so weapons change how you
  fight (piercing shards, chain sparks, orbiting blades, death bursts…).

## Stage 2a — overworld zones, gathering, crafting (first version done)

- `DB.ZONES`: one generated outdoor zone per biome (`Vael Marsh`, `Hollow Moor`,
  `Ka-Zeth Dunes`, `Frostspire Tundra`), built by the same `genDungeon()` with the
  shared outdoor layout in `DB.ZONE`, fixed by the save's world seed. Roaming
  packs are weaker than dungeon packs. A rune gate (`MODEL.gate`) in the largest
  clearing leads down into that biome's dungeon.
- `DB.NODES` (`ore`, `fish`, `wood`): hold left click on a node to walk up and
  work it (`work()`); it yields on a timer, sometimes a rare (gem, pearl, amber),
  empties, and respawns. `DB.TRADES` levels (mining, fishing, woodcutting,
  smithing, cooking) speed work up and improve rare odds.
- `DB.RECIPES`, crafted in town: rations from fish (Q in the field heals), and
  forged weapons, armour and charms whose rarity follows Smithing level.
- Still to do from the list below: one continuous map instead of four separate
  zones, generated towns inside the world, overworld-only enemy families, rune
  stones and shrines, the world-map panel, the fishing bite window.

## Stage 2b — one continuous overworld (next)

One seeded world per save (`S.save.world = {seed}`), generated once, streamed
in chunks around the player with the existing `Vox` + `meshVox`.

| Table | Contents |
|---|---|
| `DB.BIOMES` | `crypt-marsh`, `moor`, `desert`, `frost`: palette, `FLOOR`/`WALL` style, props, enemy list, node list. Reuses the four dungeon palettes. |
| `DB.SITES` | `town`, `dungeon`, `shrine`, `camp`, `ruin`: how many per biome, min spacing, what stands there |
| `DB.ENEMIES` (+) | overworld roamers per biome: wolves, bog wisps, scarab swarms, ice wraiths, bandits. Each needs a `MODEL` and reuses `AI`/`foeAI` |

- **Map**: a biome is picked per region by two low-frequency `noise2` fields
  (heat, damp), so the four biomes meet along winding borders. Roads are `PATH`
  polylines between sites.
- **Towns** are generated like `genTown()` already does (houses, lamps,
  graveyard) but laid along a road, with a bank chest, a forge, a cooking fire
  and vendors. The current lobby becomes the first town.
- **Dungeons** are sites: walking into the entrance calls `startRun()` with
  `seed = hash(world seed, site id)`, so each entrance always leads to the same
  dungeon until its depth sigil rises.
- **Flair**: rune stones with glowing glyphs (`DB.SIGILS`), relic pedestals,
  standing circles, will-o-wisps, shrines that grant a timed boon.
- **World map** panel (`PANEL.map`): the same fog-of-war image as the minimap,
  whole world, with discovered sites.

## Stage 3 — gathering, crafting, skills

RuneScape-style: you get better at what you do.

| Table | Contents |
|---|---|
| `DB.TRADES` | `mining`, `fishing`, `woodcutting`, `smithing`, `cooking`, plus combat `attack`, `defence`, `magic`, `archery`. Each: xp curve, what it unlocks per level |
| `DB.NODES` | `copper`, `iron`, `silver`, `runestone`, `pond`, `river`, `oak`, `deadwood`: biome, trade, level needed, time per haul, yield table, respawn |
| `DB.ITEMS` | ores, bars, fish, logs, cooked food, crafted gear bases |
| `DB.RECIPES` | station (`forge`, `fire`, `bench`), inputs, output, trade + level, xp |

- **Gathering**: hold left click on a node, exactly like attacking a foe. The
  character walks up and works it on a timer (`NODE[kind]` drives the animation
  and effect: pick swings and sparks, a cast line and a bobber). Trade xp per
  haul; higher levels are faster and unlock richer nodes.
- **Fishing** is the one with a little play in it: a bite window — release and
  re-press when the bobber dips for a better catch, otherwise you still get the
  common fish.
- **Crafting**: at a station, a recipe list filtered by what you carry. Smithed
  weapons roll a boon affix from `DB.WPERKS`; cooked food is the solo healing
  item (replaces most orb drops in the overworld).
- **Combat skills** train from use: `attack`/`archery`/`magic` from damage
  dealt with that weapon type, `defence` from damage taken. They add small,
  permanent stat growth beside the level-up boons.

## Stage 4 — depth

Boss remakes still owed (a leaner Charnel Golem, more boss attack animations),
ability trees per class (boons arranged in three branches with prerequisites),
more enemy families per biome, armour sets with visible detail.

## Rules that stay

Everything above follows `CLAUDE.md`: content in `DB`, numbers in `CFG`, state
in `S`, behaviour in registries keyed by id, portable to GDScript. Seeds follow
doc 11 (layout / loot / fight streams; the world seed feeds the layout stream).
