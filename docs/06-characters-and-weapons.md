# 06 — Characters & Weapons

All entities use **5 cm voxels** (`ENTITY_VOXELS_PER_METER = 20`) and
**real-world metric proportions**, with mild heroic stylization (about 6 heads
tall instead of 7.5) so heads and hands read from the isometric camera.

## 1. Player characters

The vertical slice has three classes, one per party role (see
[10](10-party-and-dungeon-runs.md)). All three share the humanoid rig below,
and only gear and silhouette differ:
- **Templar** (tank): plate, kite shield, longsword or mace.
- **Sister of the Pale Flame** (healer): robes over mail, censer and tome,
  a hood with an emissive halo ring.
- **Reaver** (damage): leather and chain, dual war axes, a tattered half-cape.

AI companions use exactly the same models and gear, so what you equip on
them shows on them.

### Reference rig: Templar

Height **1.80 m = 36 voxels**. Rigid-part rig with 15 parts:

| Part | Size (voxels W×H×D) | Real size | Notes |
|---|---|---|---|
| Head | 6 × 6 × 6 | 30 cm | Room for eye sockets/visor slit (1 voxel), nose ridge, hair |
| Neck | 2 × 1 × 2 | — | Hidden by helm/gorget |
| Torso (upper) | 9 × 7 × 5 | 45 cm wide | Chest plate layer is +1 voxel shell |
| Pelvis | 8 × 3 × 5 | — | Tabard/fauld attaches here |
| Upper arm ×2 | 3 × 6 × 3 | 30 cm | Pauldron is a separate attachment |
| Forearm ×2 | 3 × 5 × 3 | 25 cm | Vambrace layer |
| Hand ×2 | 2 × 3 × 2 | 15 cm | `hand_r` / `hand_l` sockets at grip center |
| Thigh ×2 | 4 × 8 × 4 | 40 cm | |
| Shin ×2 | 3 × 8 × 3 | 40 cm | Greave layer |
| Foot ×2 | 3 × 2 × 5 | 25 cm long | Sabaton layer |

**Pivots** sit at real joint positions (shoulder, elbow, wrist, hip, knee,
ankle, neck, spine). Animation clips are authored once on this rig and
**retargeted by name** to any humanoid (skeletons reuse it).

### Modular gear (visible loot)
- Slots: **helm, chest, gloves, belt, legs, boots, cape, main hand, off hand**.
- Armor pieces are voxel shells authored **1 voxel outside the body part**
  and attached to the same pivot, so swapping is instant and needs no
  re-skinning.
- **Palette rows:** every armor or weapon mesh stores palette *indices*. Its
  material picks a row in the `item_palettes.png` texture, giving tiers
  (rusted iron → steel → blackened → gilded → runic) with one mesh and zero
  extra textures.
- **Affix visuals:** fire, cold or holy affixes toggle emissive voxel groups
  (runes along the fuller, glowing gem in a pommel) and attach a small
  particle emitter.
- **Cape and tabard:** a 2–3 segment voxel chain with spring physics for
  cheap secondary motion.

### Animation set (minimum)
`idle`, `walk`, `run`, `attack_1h_a/b/c` (combo), `attack_2h_a/b`,
`block`, `cast_forward`, `cast_ground`, `hit_light`, `hit_heavy`, `dodge`,
`death`, `interact`, `drink_potion`. Rigid parts make clips tiny; the full
set is around 60 KB.

## 2. Weapons (true-scale, 5 cm voxels)

| Weapon | Real length | Voxels (L × W × T) | Grip | Notes |
|---|---|---|---|---|
| Dagger | 35 cm | 7 × 1 × 1 blade + 3 grip | 1H | Crossguard 3 wide |
| Arming sword | 90 cm | 15 blade + 3 grip, 1–2 wide, 1 thick | 1H | Fuller as a 1-voxel darker line |
| Longsword | 1.10 m | 18 blade + 4 grip | 1H/2H | |
| Greatsword | 1.60 m | 26 blade + 6 grip, 2–3 wide | 2H | Ricasso + parrying hooks |
| Flanged mace | 75 cm | 15 total, head 4 × 4 × 4 | 1H | 6 flanges |
| War axe | 80 cm | 16 haft, head 5 × 6 × 1 | 1H | Bearded blade profile |
| Halberd | 2.20 m | 44 haft, head 6 × 8 × 1 | 2H | Spike, axe blade, back hook |
| Scythe (thematic) | 1.80 m | 36 haft, blade 18 × 3 × 1 curve | 2H | Hollow Rider boss drop |
| Longbow | 1.60 m | 32 tall, 1–2 thick | 2H | String = 1-voxel line mesh, animated |
| Crossbow | 70 cm | 14 × 12 | 2H | |
| Kite shield | 90 × 55 cm | 18 × 11 × 1 (+ boss and rim) | Off | Heraldry painted via mask |
| Holy censer / tome | — | 4 × 5 × 4 | Off | Caster off-hands |

### Procedural weapon generator
Weapons are generated like buildings: a small **part grammar** in meters
that is baked to 5 cm voxels at loot-drop time and cached by item seed:

```
Sword → Blade(profile: straight | leaf | tapered | flamberge, length, width, fuller?)
      + Guard(straight | curved | ring | quillon-block)
      + Grip(wrap pattern) + Pommel(disc | wheel | scent-stopper | skull)
Axe   → Haft(length) + Head(blade profile curve, beard, back spike?) + Bindings
```
- The **base type** fixes the dimensions, and **rarity and affixes** add
  ornament: a Rare gets an etched fuller, a Legendary gets a unique silhouette
  part (e.g. a skull pommel or a flame-wave blade).
- Bake time is under 1 ms per weapon (a few hundred voxels), so there is no
  pre-made weapon library to maintain.

## 3. Proportion rules (enforced by an import validator)
- Every entity resource declares `real_height_m`. The importer checks the
  voxel bounding box against `real_height_m × 20 ± 5%` and fails the import
  otherwise, so nothing drifts off-scale.
- Weapons declare `grip_point` and `real_length_m`, and are validated the
  same way.
- Socket positions are validated against the hand pivot so weapons never
  float.
