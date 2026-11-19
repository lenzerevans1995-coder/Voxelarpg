# 10 — Party Play & Scaling Dungeon Runs

**Core concept.** This is a **party dungeon crawler**. You form a group of
up to **5** (yourself plus AI companions and/or online friends, in any mix),
pick a dungeon and a difficulty tier, and push through larger, harder runs
against the clock. The structure is modeled on high-end "mythic"-style
dungeon pushing in MMOs, with ARPG-style combat and loot.

The loop is **loot → lobby → harder dungeon**. The lobby town is the hub,
and dungeons, spread across every biome with their own enemy families, are
the game. Open overworld zones are optional side content (§7).

## 1. Roles (the trinity)

| Role | Job | Vertical-slice class |
|---|---|---|
| **Tank** | Holds threat, positions enemies, mitigates | **Templar**: shield and sword, taunts, holy wards |
| **Healer** | Keeps the party alive, cleanses | **Sister of the Pale Flame**: censer and tome, heal-over-time, burst heals, a resurrect on a long cooldown |
| **Damage (melee)** | Burst, interrupts | **Reaver**: dual axes, bleeds, mobility |
| **Damage (ranged)** | Sustained damage, crowd control | **Gravewarden**: crossbow, traps, slows (post-slice) |
| **Damage (caster)** | AoE, utility | **Occultist**: hexes, summoned wisps (post-slice) |

Party composition is free, but the recommended group is 1 tank, 1 healer
and 3 damage dealers. Solo players fill the empty slots with AI
companions.

Each class has around 8 active skills (4–5 on the bar), a talent tree with
role-flexible branches, and a gear set. Every class has **one interrupt and
one defensive skill**, which party mechanics rely on.

## 2. Dungeon runs: Sigils & Depth

- **Reliquary Sigils** are run items, one per dungeon. A sigil has a
  **Depth** (difficulty tier: 1, 2, 3 …, unbounded).
- Activating a sigil at the dungeon door starts the run:
  - A **timer** (e.g. 28 min for Undercroft of St. Vael).
  - **Dread**: a 0–100% meter filled by killing non-boss enemies.
    Completion needs **all bosses dead and Dread at 100%**. Groups plan
    *routes* (which packs to pull).
  - **Deaths** cost +5 s on the timer.
- **Result:**
  - In time: the sigil upgrades **+1 to +3 Depth** depending on time left,
    and grants a better end chest.
  - Over time: the run still completes with loot, but the sigil drops −1
    Depth.
- **Scaling per Depth:** enemy HP and damage are multiplied by
  `1.08^depth`. Gear item level scales with depth up to a cap; above the
  cap, rewards turn into cosmetics and leaderboard rank.

### Omens (rotating modifiers)
At Depth 3, 6 and 9, extra **Omens** stack on. They rotate weekly and are
data-driven (`data/omens/*.tres`). Original examples:

| Omen | Effect |
|---|---|
| **Bloodtithe** | Enemies that die leave a pool that heals other enemies standing in it; the tank must drag packs out |
| **Restless Dead** | 20% of slain enemies rise once as weaker skeletons |
| **Tolling Bell** | Every 60 s a bell tolls, and players must stand in candlelight within 5 s or get silenced |
| **Choir of Ash** | Elite packs gain a shield that only interrupts break |
| **Gilded Rot** | Bosses gain stacking damage the longer the fight runs (pressures damage output) |
| **Moonless** | Light radius is halved; enemies outside light are untargetable |

### Weekly seeds vs Uncharted
Procedural generation and learnable routes conflict, so there are two modes:

- **Charted runs (ranked).** Each dungeon has a **weekly seed**, the same
  layout for everyone that week. Groups learn the routes, and leaderboards
  are fair. Next week the layout regenerates and has to be learned again.
- **Uncharted runs.** A new seed per run, for players who want variety.
  Rewards are the same, but there is no leaderboard.

Because the generators are deterministic, a **seed plus dungeon id** fully
describes a layout. Leaderboards, run replays and network sync all depend
on that.

## 3. Dungeons are larger and multi-boss

Compared to the solo-ARPG plan, party dungeons are:
- **3–4 bosses** per dungeon (one per wing plus a final boss), placed by
  the mission graph.
- **2–3 floors / wings**, 25–40 minutes at intended Depth.
- **Pack-based:** enemies come in **packs of 3–6 with roles** (a caster
  that must be interrupted, a frontliner, an archer), not single wanderers.
  Pack placement leaves **pull space**, so a tank can drag a pack back to
  a safe room.
- **Route choice:** the mission graph produces **alternative paths** with
  different Dread totals and risk (a long safe hall versus a short path past
  an elite). That is the strategic layer of routing.
- **Skips & shortcuts:** some side branches are optional. Skipping them is
  valid if the route still reaches 100% Dread.

The dungeon generator ([04](04-dungeon-generation.md)) gains:
`pack_budget` (instead of a per-room monster count), `dread_value` per
enemy, alternative-path rules, and boss slots per wing.

## 4. Threat & party mechanics (enemy side)
- Every enemy keeps a **threat table**. Damage generates threat, healing
  generates half as much spread across engaged enemies, and the tank has a
  threat multiplier (×3) plus **Taunt**.
- Boss mechanics are built for groups and reuse a small vocabulary:
  `tank_buster` (tank must use a defensive or swap), `spread` (players
  marked must separate), `stack` (shared damage), `interrupt_or_wipe`,
  `soak` (someone stands in it), `adds_phase`, `dispel`.
- The vocabulary is a set of **data resources**, so both human players and
  **AI companions** read the same mechanic definitions (see below).

The existing bosses fit naturally:
- **Charnel Golem:** *Slab Smash* is a `tank_buster`. *Bile Retch* targets
  a random non-tank, so players must `spread`. In phase 2, *Rib Barrage*
  must be `soak`ed by 2 players or it hits everyone.
- **Hollow Rider:** the unhorsed phase needs the tank on the rider while
  damage dealers handle the steed. *Lantern Head Hurl* is a `stack` on the
  target to split damage. *Call of the Gallows* is an `adds_phase`.

## 5. AI party members (companions)

AI companions are good enough to finish runs at moderate Depth and fill any
role.

**Architecture:** the same `brain.gd` utility AI as enemies, plus a party
layer:
- **Role controller:** `TankBrain`, `HealerBrain`, `DamageBrain`. Each
  scores actions using role-specific considerations:
  - *Tank:* keep threat lead on all engaged enemies, face bosses away from
    the party, use defensives on `tank_buster` telegraphs, pull packs to the
    party's chosen spot.
  - *Healer:* triage by `missing_hp × incoming_damage_rate`, pre-shield
    before known mechanics, manage mana, cleanse `dispel` debuffs,
    battle-res when it matters.
  - *Damage:* priority rotation from data, **interrupt rotation** (the
    party assigns interrupt order on cast-bar events), target the
    marked/priority enemy.
- **Mechanic awareness:** companions subscribe to telegraph and mechanic
  events (the same data the UI uses). `spread` moves the companion to a
  spot ≥ X m from others, `stack` moves it to the marked player,
  `soak` makes the nearest companion step in, and ground hazards make it
  path out of the decal shape.
- **Player commands (radial wheel / hotkeys):** *Follow*, *Hold here*,
  *Pull next pack*, *Focus my target*, *Use cooldowns now*, *Wait for mana*.
  When an AI is the tank, it leads the route: the player sets the route on
  the map (waypoints through packs) and the AI tank pulls along it.
- **Difficulty honesty:** companions use the same skills, cooldowns and
  reaction times as players, with an artificial reaction delay of
  150–350 ms by "skill tier". They never cheat. At very high Depth, a
  well-played human group should outperform them.
- **Personalities (light):** named companions with their own bark lines and
  minor behavior quirks (for example a cautious healer who over-heals, or
  an aggressive Reaver who pulls early on low trust). This is flavor, not
  extra systems.

## 6. Online co-op (friends)

- **Up to 5 players**, freely mixed with AI (e.g. 2 friends + 3 AI).
- **Topology:** **host-authoritative listen server** using Godot's
  `MultiplayerAPI` over ENet. A dedicated headless server build comes later,
  using the same code with `--headless`.
- **What is synced:**
  - *Dungeon layout:* only `(dungeon_id, seed, depth, omens)`. Each client
    generates the identical dungeon locally from the deterministic
    generators, so no voxel data crosses the network.
  - *Voxel changes* (broken pillars, smashed props): small events with ids,
    applied deterministically.
  - *Actors:* the server simulates AI, combat and threat. Clients send
    input and skill casts, and receive snapshots at 20 Hz (position, facing,
    animation state, HP and resources, buffs) with interpolation. The local
    player uses **client-side prediction** for movement and skill animation
    start.
  - *Loot:* rolled per player on the server (personal loot, no
    loot-drama), plus a trade window inside the group for 2 hours.
- **Lobby:** a party finder via a lightweight matchmaking service
  (post-slice). The vertical slice supports invite by code/IP plus
  Steam-style lobbies through an abstraction layer (`OnlineService`
  interface) so platforms can be swapped.
- **Determinism discipline** (already a code rule) is what makes this cheap.
  CI runs a **desync test**: two headless instances generate the same seed,
  and their chunk hashes must match.

## 7. The core loop: Loot → Lobby → Harder Dungeon

```
          ┌──────────────────────────────────────────────┐
          ▼                                              │
   LOBBY (Morrowgate)  ── form party (AI / friends) ──►  DUNGEON RUN
   gear up, craft,                                      (timer, Dread,
   pick dungeon + Depth                                  bosses, Omens)
          ▲                                              │
          └──── end chest + boss loot + upgraded sigil ◄─┘
```

- **The lobby is the hub.** Morrowgate is a small, hand-dressed Gothic town
  square (procedurally decorated) that holds:
  - **The Reliquary Table:** a map of every dungeon. Pick dungeon, Depth and
    mode (Charted or Uncharted), then the party is teleported in.
  - **Party Board:** invite friends, browse public groups, hire or swap AI
    companions, and set companion roles.
  - **Vendors and crafting:** blacksmith (reroll an affix, upgrade item
    level), jeweler (gems and sockets), alchemist (flasks), and the
    transmog wardrobe for voxel cosmetics.
  - **Training yard:** a target dummy with a damage meter, plus a sandbox
    that replays boss mechanics.
  - **Hall of Records:** leaderboards per dungeon per week.
- Runs take 25–40 minutes. A play session is "one more run": you finish,
  land back in the lobby with loot, equip it, and queue a higher Depth.
- **No mandatory overworld.** Open zones (e.g. the Weeping Moor) are
  optional side content for world bosses like the Hollow Rider, a one-time
  story intro, and sigil farming. The lobby can launch any unlocked dungeon
  directly.

## 8. Dungeon roster: every dungeon is a biome with its own enemies

Dungeons are **not only crypts**. Each one picks a **biome** and a **theme
kit**. Indoor segments use the dungeon generator ([04](04-dungeon-generation.md)).
Outdoor segments (courtyards, canyons, mountain passes) use the overworld
terrain generator ([03](03-overworld-generation.md)) bounded to a corridor
of cells, so one dungeon can go *outside → inside → down*.

Each dungeon has a **unique enemy family** of 6–10 enemy types plus its own
bosses. Some generic enemies (skeletons) appear across dungeons as filler.

| # | Dungeon | Biome / setting | Segments | Unique enemy family | Bosses |
|---|---|---|---|---|---|
| 1 | **Undercroft of St. Vael** | Gothic cathedral and crypts | Nave → crypts → ossuary | Restless dead: skeleton warriors/archers, bone brutes, wax-faced acolytes (casters to interrupt), candle wraiths | Abbess Morwen (summoner), the Deacon of Wax, **Charnel Golem** |
| 2 | **Hollow Road Barrow** | Night moor and burial mounds | Outdoor moor road → barrow tunnels | Gallows-folk: hanged men, crow swarms, bog hounds, lantern-bearers | Mother of Crows, **The Hollow Rider** (mounted arena) |
| 3 | **Sunken Sepulchre of Ka-Zeth** | Ashen desert: sand tomb | Collapsed bazaar → sand dunes → tomb | Embalmed dead: wrapped warriors, scarab swarms, sand-wraiths, canopic priests (cleanse their curses) | The Twin Viziers (shared HP), Ka-Zeth the Dry King |
| 4 | **Frostspire Citadel** | Snowy mountain castle | Mountain pass → curtain walls → keep | The Frozen Host: frost-bitten knights, ice wraiths, yeti-like hulks, frost archers on battlements | Castellan Hrimgar, the Wyrm in the Glacier |
| 5 | **Drowned Abbey** | Fen and sunken abbey | Boardwalk swamp → flooded cloister | The Drowned: leech-priests, bloated drowners (explode on death), eel horrors, choir ghosts | The Drowned Choir (multi-target), Abbot Lurr |
| 6 | **Rotwood Hollow** | Blighted forest | Forest path → fungal caverns | Rot-hounds, giant spiders, spore-shamblers, witch-hags | The Weaver, the Heart-of-Rot (stationary, adds-phase) |
| 7 | **Ivory Spire** | High fantasy: corrupted mage tower | Gardens → tower floors (vertical) | Animated armors, arcane constructs, mirror-mages, gargoyles | The Archivist, the Unbound Familiar |

**Unique enemies are cheap to make**, which keeps the roster broad:
- Humanoid enemies **reuse the shared rig and animation set** and differ
  in SDF body recipe, gear and brain resource.
- New skeletons (quadruped, spider, serpent, flying swarm) are added once
  and then reused within families.
- Behaviors (`KiteAndShoot`, `ChargeLine`, `Summon`, `ExplodeOnDeath`…)
  are reusable resources, so most new enemies are **data, not code**.
- Every family needs a **role spread** for good pulls: frontliner, ranged,
  caster (must interrupt), support/healer, and one "elite" type.

## 9. Loot

- **Sources:** pack drops (low), boss drops (personal, guaranteed), and an
  **end chest** whose quality depends on Depth and timer result. A
  **weekly vault** offers one pick from your best runs.
- **Item tiers:** Common → Magic → Rare → Relic (set pieces per dungeon) →
  Legendary (build-changing powers, e.g. "Taunt also pulls enemies to you").
- **Dungeon-specific loot tables:** each dungeon drops its own themed sets
  and weapons. Frostspire drops frost-steel plate, Ka-Zeth drops golden
  khopesh-style blades. **The voxel look is visibly themed**, generated by
  the weapon and armor grammars with dungeon style tags, so players can show
  where gear came from.
- **Role-aware drops:** loot is rolled for your current role and class to
  avoid useless drops. AI companions also get gear, which you equip on them
  from the Party Board.
- **Item level** rises with Depth up to a soft cap. Past that, progression
  comes from affix rerolls, gems, Legendaries and cosmetics.

## 10. Progression summary
```
New character → short story intro (Weeping Moor, optional to replay)
  → Dungeons unlocked at Depth 0 (story mode, levels 1–20)
    → Level cap: Reliquary Sigils unlock for every dungeon
      → Loot → Lobby → Higher Depth (AI or friends) → better loot → ...
        → Weekly Omens + weekly seed reset keep routes fresh
          → Leaderboards, voxel transmogs, titles
```
