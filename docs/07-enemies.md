# 07 — Enemies

> **Party context:** HP and damage below are for a **5-person party at
> Depth 0**. Depth scales them by `1.08^depth`. All enemies use threat
> tables, and boss attacks are tagged with the shared mechanic vocabulary
> (`tank_buster`, `spread`, `stack`, `soak`, `interrupt_or_wipe`,
> `adds_phase`) from [10](10-party-and-dungeon-runs.md), so AI companions
> can respond to them.
>
> These three belong to the first two dungeons' families. Skeletons are
> filler in the **Restless Dead** family (Undercroft of St. Vael), the
> Charnel Golem is the Undercroft's final boss, and the Hollow Rider is the
> final boss of **Hollow Road Barrow** and also roams the optional Weeping
> Moor zone as a world boss.

The first three enemies cover the three size classes. Each is a data
resource (`data/enemies/*.tres`) with a rig, stats, a behavior tree and loot
references. All use **5 cm entity voxels** at true scale.

| Size class | Enemy | Height | Footprint | Collision radius | Voxel bounds (approx.) |
|---|---|---|---|---|---|
| **Medium** | Skeleton | 1.85 m | 0.6 × 0.4 m | 0.35 m | 12 × 37 × 8 |
| **Large** | Charnel Golem | 3.4 m (hunched) | 2.2 × 1.6 m | 1.1 m | 44 × 68 × 32 |
| **Extra Large** | The Hollow Rider | 4.3 m mounted | 4.6 × 1.6 m | 1.6 m (capsule along body) | 92 × 86 × 32 |

---

## 1. Skeleton (Medium)

**Look.** Reuses the humanoid rig and animations, with bones built from SDF
capsules. The ribcage is 7 rib arcs, each 1 voxel thick, with see-through
gaps, so you can see the torches behind it. Yellowed-bone palette with
darker joint caps, empty eye sockets with dim **ember-blue emissive pupils**,
and a rusted half-helm and tattered tabard as optional gear layers.

**Variants (same rig, different gear and brain):**
| Variant | Gear | Behavior |
|---|---|---|
| Skeleton Warrior | Rusted arming sword + round shield | Approach, 2-hit combo, raise shield when the player winds up |
| Skeleton Archer | Short bow | Keep 8–12 m away, retreat if the player comes within 4 m, volley of 3 |
| Bone Brute (elite) | 2H rusted greatsword | Slow overhead cleave (telegraphed by a red ground decal), knockback |

**Stats (Depth 0 baseline):** HP 120 · Damage 6–9 · Move 3.2 m/s ·
Armor 5 · Weak to *blunt* (+50%) and *holy* (+25%), resists *pierce* (–30%).

**Voxel-native touches**
- **Blunt hits knock off individual bones.** Losing an arm part removes its
  weapon. Losing a leg makes it crawl at 40% speed.
- **Death:** the parts become rigid bodies and collapse into a bone pile.
- **Reassembly:** if a *Necro Candle* prop is lit nearby, piles rebuild
  after 6 s (an animated reverse collapse). Snuff the candle to stop it.

**Animation:** shared humanoid set, plus `rise_from_ground` (a spawn
animation that pushes up through floor voxels with a dust burst) and
`crawl`.

---

## 2. Charnel Golem (Large): rotting flesh ogre-golem

**Concept.** A hulking ogre-shaped construct stitched together from the
corpses of the cathedral's dead and held up by an iron armature. Hunched,
with a massive forearm (left) fused to a **grave-slab cudgel**. A bloated,
split belly shows the iron frame and ribs. Iron bands, hooks and chains
hold flesh slabs in place, and wax candles burn on its shoulders: the
monks' last attempt to sanctify it.

**Look (layers on one rig, 18 parts):**
1. **Armature layer:** dark iron frame, rivets and a spine rod (always
   present).
2. **Bone layer:** oversized ribs and a jaw.
3. **Flesh layer:** mottled grey-green and bruise-purple palette with
   1-voxel stitch lines (darker), weeping sores (emissive dull yellow
   pustules), and exposed muscle red in the gaps.
4. **Props:** chains (a 2-voxel link chain mesh with spring physics),
   shoulder candles (emissive and light-casting) and hanging hooks.

**Stats (Undercroft final boss, Depth 0):** HP 14,000 · Damage 22–30 · Move 2.4 m/s
(lumbering, but its charge is 7 m/s) · Armor 15 · Weak to *fire* (+50%),
resists *poison* (immune) and *pierce* (–20%).

**Moveset**
| Attack | Telegraph | Effect |
|---|---|---|
| Slab Smash `tank_buster` | Raises its arm 0.8 s, red circle decal (r = 2.5 m) | Heavy hit on its threat target; the tank must use a defensive, others must not stand nearby. AoE + knockdown, cracks floor voxels (cosmetic) |
| Sweep | Torso twist 0.5 s, cone decal 120° | Knockback 4 m |
| Bile Retch `spread` | Belly swells (scales), 1.0 s | Aimed at a random non-tank; poison cone 6 m, leaves bile puddle for 8 s |
| Grab & Toss | Reaches out its right hand | If it connects: throw the player 6 m, damage on landing |
| Corpse Charge | Paws ground twice | Charges in a straight line and stops on wall impact (stunned 2 s, vulnerability window) |

**Phases**
- **Phase 1 (100–50%):** the moveset above.
- **Phase 2 (< 50%), "Sloughing":** the flesh layer detaches in chunks. The
  flesh parts become physics gibs that leave rotting puddles. It becomes
  faster (+30% move, shorter telegraphs) with lower armor (–40%), and
  **Bile Retch** is replaced with **Rib Barrage** `soak` (5 rib-spike
  projectiles land at marked spots, and each must be soaked by a player or
  it explodes party-wide). This is voxel-native feedback: the model visibly changes
  from bloated hulk to skeletal iron frame.
- **Damage chipping:** every heavy hit removes a small cluster of flesh
  voxels at the impact point (capped at 30% of the flesh layer to protect
  the silhouette before phase 2).

**Arena:** an octagonal chapter house (6×6 cells) with pillars that break
when it charges into them (fragile material). Destroying 2 pillars collapses
part of the ceiling for bonus damage: an environmental mechanic.

---

## 3. The Hollow Rider (Extra Large): headless horseman

**Concept.** A cursed knight who once rode the moor road to warn the
cathedral and was beheaded at the crossroads gallows. He rides a gaunt,
oversized **nightmare destrier** (2.4 m at the withers, far larger than a
real horse) whose ribs show through tattered caparison barding. Where his
head should be, a **caged lantern** burns with spectral green-white fire.
He wields a **great scythe** (true-scale 2.6 m haft, enlarged to match his
size) and carries a **chain-flail** at his hip.

**Look**
- **Rider:** 2.2 m tall if standing (an XL humanoid rig at a 1.25 scale
  factor; the voxel size stays 5 cm, he just has more voxels). Fluted black
  plate armor, a torn cloak with 3-segment spring cloth, and a lantern cage
  at the neck as its own part with an emissive core plus a point light that
  flickers.
- **Steed:** quadruped rig with 22 parts (head, neck ×2, torso, pelvis, 4
  legs × 3 segments, tail ×3). Exposed rib voxels, burning hooves (emissive
  plus ember particles), mane as spring-chain voxel strips, and eyes that
  match the lantern color.
- **Total silhouette:** 4.3 m tall, 4.6 m long, the largest thing on screen
  and readable from far away.

**Stats (Hollow Road Barrow final boss, Depth 0):** HP 32,000 (rider and steed share HP in phase 1, then split 60/40) ·
Damage 25–40 · Mounted move 9 m/s (charges at 14 m/s) · Armor 20 · Weak to
*holy* (+50%), resists *cold* (–40%), immune to *fear*.

**Spawn & arena.** He is fought at **Hanged Man's Crossroads**, the outdoor
final arena of Hollow Road Barrow. In the optional Weeping Moor zone he
also appears at night as a world boss (an event timer, or triggered by
ringing the gallows bell). The
arena is the open crossroads (~40 m radius) with gibbets and broken carts
as cover. Carts are fragile props that he smashes through.

**Moveset**
| Phase | Attack | Telegraph | Effect |
|---|---|---|---|
| 1 (mounted) | **Ride-down** | Rears up and the lantern flares, then a long line decal | Charges through the arena, trample damage and knockdown, leaves a fire trail for 4 s |
| 1 | **Reaping Arc** | Scythe drawn back 0.7 s, wide arc decal | Big 200° sweep around him while circling |
| 1 | **Lantern Bolts** | Lantern brightens 3 times | 3 homing spectral-fire projectiles |
| 1 | **Call of the Gallows** | Bell tolls | Summons 4–6 Skeletons from the ground (reuses the Medium enemy) |
| 2 (< 60%, "Unhorsed") | — | — | Steed and rider **split into two targets**: |
| 2 – Steed | **Wild Trample** | Paws, snorts fire | Erratic charges, kick cone behind it |
| 2 – Rider (on foot) | **Severing Sweep / Head Hurl** | Lantern pulled from the neck cage | Throws the burning lantern head, which explodes in a 4 m AoE and leaves a fire patch. He is briefly "blind", so his attacks are inaccurate until it returns. |
| 3 (one target < 20%) | **Remount Enrage** | Survivor calls the other | If the steed survives, the rider's ghost remounts (+speed). If the rider survives, he gains the steed's burning-hoof trail. |

**Rewards:** guaranteed rare, a chance at the **Hollow Scythe** (legendary
2H) or the **Lantern of the Headless** (legendary off-hand light source,
increased light radius in dungeons).

---

## AI architecture (shared)
- Lightweight **utility-scored behavior selection** (`brain.gd`) ticked at
  10 Hz: each behavior scores itself on distance, cooldowns, player state and
  health phase, and the highest score runs. It is simpler than a full
  behavior tree and easy to tune in data.
- Behaviors are reusable resources: `ApproachAndMelee`, `KiteAndShoot`,
  `ChargeLine`, `AoEAroundSelf`, `SummonMinions`, `PhaseSwitch`.
- Every attack has a **telegraph resource** (decal shape, duration and
  color) so the readability rules are consistent across all enemies.
