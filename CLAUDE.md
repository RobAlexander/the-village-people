# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project: Village Simulation

A browser-based village life simulation — a modern take on a 1990s screensaver. Delivered as a **single self-contained `index.html`** with all JavaScript embedded.

## Running

Open `index.html` in any modern browser. No build step, no server required (though it can be served from a static web server).

## Architecture

All code lives in `index.html` as one IIFE. Sections are marked with `// ── SECTION ──` banners. The ordering within the file matters — classes and functions must be declared before use.

---

## Coordinate System

- **nx** ∈ [0,1] — normalised horizontal position
- **ny** ∈ [0,1] — normalised depth (0 = far/top of ground, 1 = near/bottom)
- **Sky** occupies the top 33% of the canvas (`GROUND_TOP = 0.33`)
- **Ground** occupies the bottom 67%
- `worldY(ny)` converts ny to canvas pixel Y
- `sy` getter on each ground entity returns `worldY(this.ny)` — used for painter's-algorithm sort

---

## Time System

- `gameTime` — running milliseconds, persisted via localStorage
- `DAY_MS = 120000` — 2 minutes per full day cycle
- `tod()` — time-of-day fraction 0–1
- `dayNum()` — integer day count
- Four phases at t = 0 / 0.25 / 0.5 / 0.75: **night / dawn / day / dusk**
- `ambientLight(t)` → 0.08–1.0 (drives dimming of all colours)
- `skyColors(t)` — interpolated two-stop sky gradient colours
- `groundColor(t)` — interpolated ground base colour
- All colour interpolation uses `lerpHex(a, b, t)` / `dimHex(hex, f)`

---

## Draw Helpers

- `poly(pts, fill)` — filled polygon; `pts` is **always `[[x,y],[x,y],...]`** (array of pairs, never a flat array)
- `box(x, y, w, h, fill)` — filled rect
- `disk(x, y, r, fill)` — filled circle
- `cap(x, y, r, fill)` — filled semicircle (top half — used for hair)
- `noisyPoly(pts, seed, amp, fill)` — polygon with **seeded** per-vertex perturbation (no per-frame flicker); also uses array-of-pairs format
- `noisyBox(...)` — 8-point box with seeded noise
- `srand(seed, i)` / `hash(n)` — deterministic noise source

---

## Background Rendering

Drawn every frame in `drawBackground()`:

1. **Sky** — two-stop linear gradient, interpolated through night/dawn/day/dusk palettes
2. **Stars** — 90 static positions, twinkle via sin, visible only when `light < 0.55`
3. **Sun** — arc from t=0.22 to t=0.78, with radial glow and 8 rays
4. **Moon** — arc from t=0.72 to t=0.28 (wraps midnight), crescent effect via overlaid sky-coloured disk
5. **Clouds** — 7 seeded clouds (`CLOUDS` array); each drifts left-to-right at a unique speed via `gameTime * speed`; shape is 5 overlapping ellipses with seeded offsets (no per-frame flicker); fade in/out with `ambientLight` (invisible when `light < 0.28`, fully opaque by `light ≈ 0.50`)
6. **Horizon shadow** — 4px dark strip at sky/ground boundary
7. **Background trees** — 16 seeded conifers along the horizon, two-tone canopy
8. **Ground gradient** — `dimHex(gndCol, 0.82)` at top (horizon) → full `gndCol` at bottom; simple two-stop linear gradient, darker-far / lighter-near perspective
9. **Grass blades** — 2500 seeded triangular blades scattered across the ground; two colours (light/dark) switched by `ambientLight < 0.25`
10. **Ground tracks** — `drawTracks(light)` scales an 80×50 off-screen canvas onto the ground area; worn cells show as beaten earth, heavily-worn cells slowly pave to limestone cobblestone

---

## Entity Classes

### `House`
- Constructed when a new villager needs a home (`assignHome`)
- `completion` 0→1 driven by `buildMs` accumulated while `build`-state villagers are present; `buildMs / 55000`
- **Extensions** (0–3 completed, `extensions` field): triggered when a new adult is assigned to a full house (`capacity = 2 + extensions`). Type is seeded per slot: `'up'` (adds a floor) or `'out'` (widens by +0.30 per extension). Built by a resident in `build` state near the house; progress at `dt * workers / 40000`. `floors` and `widthScale` are derived dynamically from completed and in-progress extensions (not seeded/static).
- Draws incrementally: back wall → front wall → storey bands → roof → door → windows → chimney
- **Chimney smoke** (3 disks, semi-transparent) appears when `light < 0.45` and chimney is complete
- **Windows** lit amber (`WIN_LIT`) when `light < 0.35`, blue-grey (`WIN_DAY`) otherwise
- **Mature flora** — after `ageMs > 300 000 ms`, roses (62% chance) and window flower boxes (58% chance) fade in, seeded per house
- **Burning**: `burning` flag + `burnProgress` 0→1 (over 45 s real time). While burning, `update()` returns early after advancing progress. `fireSoundPlayed` flag ensures `playFireCrackle()` fires only once. Fire overlay: 5 animated flame tongues + 3 smoke puffs drawn on top of the normal house render

### `Ruin`
- Created by `checkBurning()` when a burning house reaches `burnProgress >= 1`
- Draws charred rubble: ash base polygon, 5 seeded timber chunks, chimney stump, soot overlay
- **Passable** — not included in `avoidBldgs` or `pathDetour`; ground-level only
- New houses cannot be assigned within ~0.13 nx / 0.06 ny of a ruin (`assignHome` skip check)
- Persisted under key `ru`; `widthScale` preserved so ruin matches former house footprint

### `Church`
- Spawned when non-priest adults ≥ `CHURCH_POP (10)`
- Positioned at nx=0.82, ny=0.35
- Draws incrementally: nave → roof → bell tower → spire → cross → arched door
- Slightly larger and slower to build than a house (`buildMs / 100000`)
- **Post-completion upgrades** (unlocked by `ageMs`):
  - `hasWindows` (ageMs > 30 000 ms) — rose window above door + lancet side windows
  - `hasBell` (ageMs > 120 000 ms) — bell silhouette in tower; rings `playChurchBell()` once per Sunday at dawn
  - `hasGargoyles` (ageMs > 300 000 ms) — gargoyle figures crouching at each corner of the nave roof

### `Mausoleum`
- Spawned by `checkMausoleum()` (called each frame) when `graveyard.crosses.length >= 100`
- Half the existing grave crosses are absorbed (removed) and tracked in `interredCount`
- Positioned near the graveyard; built by `build_mausoleum`-state villagers (priest prioritises it, adults have 30% chance); `buildMs / 500 000`
- `elaborateness` ∈ [0,1] driven by `graveyard.totalDead`; controls which of ~10 visual tiers are rendered (steps, body, wings, pilasters, cornice, pediment, dome, spire, iron fence, etc.)
- Persisted under key `ml` in the save state

### `KitchenGarden`
- Spawned by a resident adult once their house is complete, ≥6 houses exist, and the church exists (12% chance per `_pickTarget()` call)
- Placed beside the house: offset +0.09 nx if house is left of centre, −0.09 if right
- `built` flag: false until a `build_garden`-state villager accumulates `GARDEN_BUILD_MS = 20000` ms of work
- `growMs` advances in real time once built; `GARDEN_GROW_MS = 1200000` (20 real minutes) to fully grow
- 3 rows: `carrot` / `cabbage` / `turnip`; each row has a `stolen` flag
- `hasRipeVeg`: true when `grown` and at least one row is not stolen
- `harvest()`: resets all `stolen` flags and `growMs` to 0 (resident villager calls this)
- **Rabbits** can steal one row per visit when `hasRipeVeg` is true; rabbit carries `robGardenId` during the hop
- **Draw**: cream picket fence (`#deded0`) with gate gap in front rail; 3 soil rows; 4 vegetables per row scaled by `√growFrac` — carrot (orange triangle + green shoots), cabbage (flat green ellipse), turnip (white disk + purple cap + green tuft); stolen rows draw nothing
- Included in `avoidBldgs` and `pathDetour` as obstacles (dimensions: halfW = `24*s/W`, y1 = `ny−0.068`, y2 = `ny+0.008`)
- Persisted in localStorage under key `kg`

### `Villager`
- Ages: `child` → `teen` → `adult` → `old` (durations: 200 s / 260 s / 600 s game-time)
- Old people then die after 4–9 min as old (stable per-id hash via `Math.sin(id*2.1)`)
- Genders: `m` / `f`; role: `isPriest` (one per simulation, male, appears with church)
- Scale by age: child = 0.5×, teen = 0.75×, adult/old = 1× (base `H/1200`)
- Hair colours: brown or blonde (random at spawn), priest has grey, old has white (`#e8e8e0`)
- **Wizard enchantment**: `skinColor` and `shirtColor` fields (normally null = defaults); set when the Wizard passes within 0.12 nx; persisted as `sc`/`stc`
- **Fertility**: each adult starts with 1–7 children (bell-curve random); each birth decrements both parents' fertility by 1. No more children when either partner reaches 0.
- **States**: `arrive` · `wander` · `going_home` · `build` · `build_church` · `build_garden` · `build_mausoleum` · `harvest` · `sleep` · `church` · `flee` · `dead` · `being_carried` · `at_grave` · `buried` · `funeral` · `fetch_body` · `carry_body` · `emigrate` · `mob`
- Old people walk slower (speed 0.000038 vs 0.000055); drawn with a walking stick
- **Evening routine**: at `tod() > 0.70`, wandering villagers with `handPartnerId === null` and a completed house (completion > 0.5) enter `going_home` and walk to the house door position at `going_home` speed (old=0.000042, non-old=0.000065 — faster than wander). `going_home` has a stateMs timeout (99999 ms) — falls back to `sleep` if unreachable. Villagers in `going_home` are immune to ghost flee and Sunday church redirect (but NOT wolf/bear flee).
- `going_home` is a transitory state — saved as `wander` in the cookie
- On Sundays (every 7th day, t=0.25–0.42), all non-sleeping/non-dead/non-going_home/non-mob villagers walk to church; stop moving when within 0.07 of church
- **Building avoidance**: `avoidBldgs(nx, ny, prevNy)` pushes villagers in ny only; applied in `_moveTo` when `avoid=true`. Includes houses, church, mausoleum, kitchen gardens, pond. Each house occupies `ny ∈ [ny − BLDG_BACK, ny + BLDG_FRONT]` where `BLDG_BACK = 0.044` and `BLDG_FRONT = 0.006`.
- **Pathfinding**: `pathDetour(nx, ny, tnx, tny)` — called each frame from `_moveTo` when `avoid=true`. Detects first obstacle whose ny range the straight-line path crosses. Returns a corner waypoint at the **near face** (south face + MARGIN if approaching from south, north face − MARGIN if from north) plus left/right offset, so the path to the waypoint never crosses the building. `avoidBldgs` then prevents ny penetration. States that intentionally target a building (`build`, `going_home`, `church`, `mob`) skip their own target building via an intentional-approach check. `funeral`, `fetch_body`, `carry_body` use `avoid=false` to approach bodies directly.
- **`_pickTarget()`**: clamps random wander targets out of building footprints to prevent oscillation
- **Walking animation**: sine-based leg sway (`gameTime * 0.0055 + id * 2.3`), per-villager phase offset; arms counter-swing at 0.5× amplitude. Active in all moving states including `build_garden`, `harvest`, and `build_mausoleum`. `build_garden` and `build_mausoleum` also use raised-arm building pose. Skirt and building-pose arms are static.
- **Mob state**: villager walks toward `mobTargetId` house at speed 0.000075 with `avoid=false`; carries a visible flaming torch; ignites house (sets `burning = true`) on arrival within 0.07 distance; times out after 120 s if unable to reach. Immune to Sunday church redirect.
- **Emigrate state**: when population exceeds 25 non-priest adults, `checkEmigration()` fires once per morning (tod 0.27–0.52). Excess adults can leave — childless/solo adults leave first. Leaver walks off-screen pushing a loaded handcart; removed from `villagers[]` on exit. Count tracked in `leaverCount`.
- Drawn with `noisyPoly`; seeded by `this.id` so shape is stable per villager

#### Hand-holding & Permanent Partners
- Each villager has `partnerId` (permanent bond id, or null) and `handPartnerId` (current hand-hold id, or null) and `holdHandMs` (remaining ms of hand-hold)
- **Couple morning emergence**: on waking from `sleep` at dawn, villager first tries to link with their permanent partner (`partnerId`). If none, 65% chance to form a new permanent bond with a housemate (same `homeId`, adult, also in `sleep`, unattached). Duration: 25–60 s game-time. Both partners record each other's id in `partnerId` permanently.
- **Re-linking**: whenever both permanent partners are wandering unattached and within 0.05 nx of each other, the lower-id partner re-initiates a hand-hold.
- **Child-parent**: on `spawnChild()`, 80% chance (if parent unattached) to link the newborn child with its parent. Duration: 18–40 s game-time. Because the child always gets a higher id, the parent is always the leader.
- **Leader / follower**: the lower-id partner is the leader and picks targets normally. The follower mirrors the leader's `tnx/tny` plus a fixed lateral offset of ±0.022 nx (sign determined by `this.id % 2`) so the pair walks visibly side-by-side. Neither partner picks building tasks while `holdHandMs > 0`.
- **Speed**: both partners move at `Math.min(leader_speed, follower_speed)` — the pair always walks at the pace of the slower member, preventing stretching.
- **Break conditions**: `_releaseHand()` is called (cleaning up both sides) when either partner enters `flee`, `build`, `build_church`, or `church` state, or when `holdHandMs` expires. `partnerId` is NOT cleared on hand-break — only on death.
- **Visual**: the leader draws a thin skin-coloured quad between the two inner hand positions (`x ± 11*s` at `y − 8*s`). Only rendered when partners are within normalised distance 0.07. Arm sway suppressed for both while linked.
- Hand-hold state (`handPartnerId`, `holdHandMs`) is **not persisted** to the cookie (transient). `partnerId` IS persisted.

### `Bird`
- 7 birds initialised at startup
- Cross the screen continuously from one edge; reset on exit
- Day only (`light > 0.28`); animated flapping wings via `Math.sin(flapT)`

### `GroundBird`
- 6 ground birds initialised at startup (seeds 100–105)
- Blue/yellow songbird that hops and occasionally makes short flights across the ground
- States: `perch` (idle bob) / `hop` (short ground move, arc height 8s) / `fly` (longer move, arc height 20s)
- Day only (`light < 0.28` → skip update and draw)
- Sorted by `sy` in painter's-algorithm layer

### `Dragon`
- Single instance; appears every 5–13 minutes of real time (300000 + random*500000 ms)
- Flies left-to-right across the sky at constant speed
- Small polygon silhouette; drawn above all ground entities (sky layer)

### `Wizard`
- Single instance; appears every 3–8 minutes of real time (180000 + random*300000 ms)
- Walks across the ground (either direction) at a slow constant pace
- Full robe, pointed hat with moon symbol, staff with glowing orb at night
- **Enchants** up to 3 nearby villagers per crossing (within 0.12 nx): sets `skinColor` and `shirtColor` to random fantasy colours
- Drawn in the painter's-algorithm ground layer (sorted by `sy`)

### `Wolf`
- Single instance; appears on roughly **1 night in 5** (20% chance rolled once at dusk via `nightRolled` flag)
- Night window: `tod() > 0.68 || tod() < 0.18`; `nightRolled` resets to false each day so the roll is exactly once per night
- Enters from a random screen edge; wanders the ground picking random targets
- At dawn walks toward the nearest edge and despawns
- **Villager flee**: any villager not in `sleep`, `flee`, `dead`, `being_carried`, or `at_grave` state within distance **0.16** enters `flee` state, running directly away at ~4.7× wander speed; recovers when `stateMs` expires or all threats (wolf, bear, ghost) are gone
- **Sound**: `playWolfGuitar()` fires every 7–16 s while active — 4 layered sine harmonics at E2/A2/D3, peak gain 0.030/harmonic, very quiet plucked-string timbre
- Drawn mirrored via `ctx.scale(this.facing, 1)` pivoted at its x position

### `Bear`
- Single instance; first appearance 3–8 minutes after start (180000 + random*300000 ms); reappears every 15–40 minutes (900000 + random*1500000 ms)
- Walks across the ground (either direction) at constant pace with waddling animation
- Large polygon silhouette with animated legs (`Math.sin(animT)`)
- **Villager flee**: any eligible villager within distance **0.11** flees (same immunity rules as wolf)
- Drawn in the painter's-algorithm ground layer

### `Ferret`
- Single instance; appears every 8–20 minutes of real time (480000 + random*720000 ms)
- Scurries across the ground at high speed (0.00019 nx/ms)
- Animated sinuous body wave (`Math.sin(animT)`)
- Does **not** cause villager flee
- Drawn in the painter's-algorithm ground layer

### `Ghost`
- Single instance; appears only during **deep night** (`tod() > 0.78 || tod() < 0.12`)
- Requires at least one grave cross; chance scales from 4% (1 grave) to 13% (30+ graves); rolled once per deep-night period via `nightRolled` flag
- Spawns at a random grave position; drifts slowly around the graveyard area; vanishes at end of deep night
- Gentle vertical bob via `Math.sin(phase)`
- **Villager flee**: any eligible villager within distance **0.16** flees; `going_home` villagers are immune
- **Sound**: `playGhostMoan()` fires every 10–24 s while active — two detuned sines with LFO vibrato and descending/rising pitch envelope ("wooOOOooo")
- Drawn in the painter's-algorithm ground layer (semi-transparent)

### `Rabbit`
- 4 rabbits initialised at startup (seeds 200–203)
- States: `sit` / `freeze` / `hop`
- 15% chance per `sit` expiry to target a nearby ripe kitchen garden (within 0.15 nx distance); steals one row on arrival
- Drawn in the painter's-algorithm ground layer

### Farm Animals — `Chicken`, `Pig`, `Dog`
- Spawned per house by `spawnAnimalsForHouse()` when house is complete, has ≥3 adult/old residents, and has a built kitchen garden for that house
- `checkFarmAnimals()` checks each frame; `h.animalsGiven` flag prevents re-spawning
- **Per house**: 3 chickens, 2 pigs, 1 dog; all seeded from `h.id * 37`; placed near `h.ny + 0.04`
- Day only (`light < 0.25` → skip update and draw)
- **Chicken**: states `peck` (head-bob animation) / `walk`; roams ±0.07 nx, ±0.05 ny around house; speed 0.000055
- **Pig**: states `root` / `walk`; roams ±0.09 nx, ±0.06 ny around house; speed 0.000030; pink body, curly tail, snout with nostrils
- **Dog**: states `sit` / `walk` / `doorstep`; roams ±0.10 nx, ±0.07 ny; 20% chance to trot to doorstep; speed 0.000080; seeded coat colour (golden `#c88030` or brown `#7a4820`); wagging tail animation
- All sorted by `sy` in painter's-algorithm ground layer; `farmAnimals[]` array
- Persisted: `animalsGiven` flag saved per house (key `ag`); `farmAnimals` re-spawned from houses on load

### Pond & Ducks
- Single pond instance (`let pond`); spawned by `checkPond()` when ≥5 kitchen gardens are built
- `pickPondPos()`: grid search (0.04-step) for a spot ≥0.18 from any house/church/mausoleum; scores by clearance + jitter
- `pond` = `{ nx, ny, rx: 0.068, ry: 0.028 }` — centre + half-radii in nx/ny units
- **Draw** (`drawPond()`): dark outer rim ellipse, main water ellipse (`#4880a8`), off-centre shimmer patch, 5 seeded reeds along back edge with oval heads
- Pond added to `avoidBldgs` as a rectangle (`checkRect(pond.nx, pond.rx, pond.ny−pond.ry, pond.ny+pond.ry)`) — villagers walk around it
- **4 Ducks** (`Duck` class): male (white body `#f0ece0`, green head `#1a5828`) or female (brown body/head); seeded coat alternates by seed index
- States: `swim` (glides inside pond, speed 0.000018) / `preen` (still on bank) / `waddle` (walks near bank, speed 0.000045); 60% swim / 22% preen / 18% waddle transition weights
- Gentle body bob `Math.sin(animT * 0.0028)`; ripple ring drawn when swimming; legs drawn only when on bank
- Sorted in painter's algorithm at `worldY(pond.ny − pond.ry)` (pond draws as early as its back edge); ducks sorted by their own `sy`
- Persisted under key `po` (`nx, ny, rx, ry`); ducks rebuilt on load (not persisted)

### Ground Tracks
- `trackWear` and `trackPave`: two `Float32Array`s of size `TRACK_W × TRACK_H` (80×50 grid)
- `updateTracks(dt)`: every active (non-sleeping/dead) villager accumulates `dt * 0.001` wear at their grid cell (capped at `TRACK_MAX = 80`)
- Paving: once the village is mature (church complete + ≥4 complete houses), cells with `wear ≥ TRACK_STRONG (20)` slowly pave to cobblestone at `TRACK_PAVE_RATE = 1/480000` (0→1 over 8 real minutes)
- `drawTracks(light)`: renders wear/pave into the 80×50 off-screen canvas, then scales it up over the ground area with bilinear smoothing. Worn cells: beaten earth `#9a7040` at low opacity; paved cells: limestone `#c0b48a`. Fades at night.
- Grass blades affected by track wear (wornFrac shifts the blade colour)
- Persisted: `trkW` (sparse object of non-zero wear values) and `trkP` (sparse pave values ≥ 0.01) in the save state

### Death & Graveyard
- When an old villager dies, `_die()` is called: releases hand-hold, clears `partnerId` on surviving partner, sets state to `dead`
- `updateDeaths()` (called each frame after updates): assigns a free wandering adult as carrier (`fetch_body` → `carry_body`). **Dead priests are skipped** — their body is handled by the angel sequence instead.
- Carrier walks to body, picks it up (`being_carried`), carries it slowly to graveyard zone. `fetch_body` and `carry_body` use `avoid=false` so carriers can approach bodies near buildings.
- `depositBody()`: body transitions to `at_grave` for 15–25 s; up to 4 nearby wanderers become `funeral` mourners (also `avoid=false`)
- After `at_grave` timer expires, a grave cross is pushed to `graveyard.crosses` and state becomes `buried`; `updateDeaths()` removes `buried` from the array
- **Graveyard** (`let graveyard`): auto-created on first death at `{nx: church.nx-0.22, ny: church.ny+0.08}` or default `{0.12, 0.45}`; also tracks `totalDead` (cumulative, including absorbed by mausoleum)
- **Grave crosses**: white polygon cross (`#d8d0bc`), drawn in painter's-algorithm layer, sorted by `sy`
- Dead body drawn as horizontal rectangle with head; old person's hair remains white

### Priest Death — Angel Sequence
- Triggered from `_die()` when `isPriest` is true; calls `startPriestSequence(nx, ny)`
- `priestSequence` object drives a 4-phase state machine (~78 s total real time); `angels[]` holds 3 `Angel` instances
- **Phase `descend`** (~6 s): 3 angels spawn above the screen top and fly down to hover above the body
- **Phase `mourn`** (~60 s): angels bob gently; `playHeavenlyChorus()` fires every 15–23 s
- **Phase `build_tomb`** (~5 s): angels converge on the body; `priestTomb` is created at that position; dead priest is removed from `villagers[]`; `playHeavenlyChorus()` fires once more
- **Phase `ascend`** (~7 s): angels fly back off the top of the screen; `priestRespawnPending = true`; `priestSequence` and `angels[]` cleared
- **Priest tomb** (`let priestTomb`): red (`#c03030`) sarcophagus with gold (`#d4a020`) bands, gabled lid, and gold cross; sorted into painter's-algorithm layer; persisted under key `pt`
- **New priest**: `updateArrivals()` checks `priestRespawnPending` at dawn (tod 0.25–0.40); spawns a new priest walking in from off-screen left; `priestRespawnPending` persisted as `pr`

### `Angel`
- Transient class; instances live only during the priest death sequence in `angels[]`
- `update(dt)`: moves toward `tnx/tny` at speed 0.00014 (nx/ms), both axes
- `draw()`: white robe, two swept wing polygons with sine flap animation, raised prayer arms, skin-coloured head, gold halo ring; drawn **after** the sorted drawables loop (above all ground entities) but before birds/dragon

---

## Mob, Burning & Ruins

### Mob trigger
- `checkMob(dt)` fires on a 1–2 minute timer (global `nextMobCheckMs`)
- Uses `findBlockingHouse()`: BFS on a **40×25 coarse grid** (MARGIN=0.03) to detect if any house's approach point is unreachable from the bottom of the screen when another house is present. Coarse grid makes small inter-building gaps register as blocked.
- `buildReachGrid(GW, GH, excludeId)` marks all buildings (including kitchen gardens) as blocked; `bfsReach()` does 8-connected flood fill
- If a blocker is found, 2–5 wandering adults are put in `mob` state targeting `mobTargetId`
- **Dev console `mob` command**: immediately targets a house (blocking house or mid-list fallback) and recruits 3–6 mob members

### Mob state
- Mob villagers walk to the target house with `avoid=false` at speed 0.000075
- Draw a flaming torch (stick + orange/yellow flame polygons) above one arm
- Ignite house (`burning = true`) on arrival within 0.07 distance
- `playMobChant()` fires every 5–9 s while any mob villager is active: 5 detuned sawtooth oscillators (95–190 Hz) with 3.2 Hz rhythmic LFO

### Burning
- Burning house: `burnProgress` advances at `dt / 45000` (45 s to burn down)
- Fire overlay: 5 animated flame tongues (orange + yellow, Date.now()-driven wobble) + 3 smoke puffs
- `playFireCrackle()` fires once on first burning frame: 20 s of white noise split into bandpass crackle (900 Hz) + lowpass roar (180 Hz)

### Collapse & ruin
- `checkBurning()`: when `burnProgress >= 1`, calls `playCollapseSound()` then creates a `Ruin`
- `playCollapseSound()`: white noise swept 4000→60 Hz over 2.2 s + sine boom 100→28 Hz
- Ruin: charred rubble drawn in painter's-algorithm layer; passable (not in avoidBldgs/pathDetour)
- House residents displaced (`homeId = null`, re-assigned via `assignHome`)
- Associated kitchen garden removed; `mobTargetId` cleared
- New houses blocked from placing within 0.13 nx / 0.06 ny of any ruin

---

## Spawning & Population

- **First batch**: 4 adults arrive quickly (every 12–27 s), alternating m/f
- **Subsequent arrivals**: one adult every 70–180 s of game-time
- **Child production**: any two cohabiting non-priest adults (any gender combination) with fertility > 0 produce a child; the lower-id adult owns the timer. Initial timer: 60–150 s game-time; subsequent births: 60–180 s. The higher-id adult's presence enables it. Each birth decrements both partners' `fertility` by 1.
- `assignHome`: villager takes a free slot in an existing house (max 2 non-child adults), or founds a new one and enters `build` state. New houses skip positions within 0.13 nx / 0.06 ny of any ruin.
- At population 10 non-priest adults: church is spawned and a priest villager is added
- When the priest dies, the angel sequence runs (~78 s); a replacement priest walks in the following morning (see Priest Death — Angel Sequence)
- **Emigration**: checked once per morning (tod 0.27–0.52). If non-priest adults > 25, excess adults may leave (`emigrate` state), pushing a handcart off-screen. Childless/solo adults leave preferentially. Tracked in `leaverCount`.

---

## Sound

All sound via Web Audio API, lazy-initialised (`audioCtx`). Sound is **on by default**; toggled via Esc menu. All sounds are quiet.

- `playGrow()` — rising sine sweep (440→660 Hz, 0.55 s); fires on teen→adult transition and on child spawn
- `playChurchBell()` — 5 bell partials (fundamental + tierce + quint + octave + 3rd), 3 rings spaced 1.4 s apart; fires once per Sunday service at dawn when `hasBell` is true
- `playWolfGuitar()` — 4 sine harmonics decaying at `1.4/√h` s; fires periodically while wolf is active
- `playGhostMoan()` — two detuned sines with LFO vibrato, pitch descends then rises over 4 s; fires periodically while ghost is active
- `playHeavenlyChorus()` — 5-voice C major 9th chord (C4 E4 G4 C5 E5), each with LFO vibrato, 1 s fade-in, sustain to 3.5 s, fade-out by 5.5 s; fires at sequence start, every 15–23 s during mourning, and once at tomb materialisation
- `playMobChant()` — 5 detuned sawtooth oscillators (95–190 Hz) with 3.2 Hz rhythmic AM LFO; 2.8 s duration; fires every 5–9 s while mob is active
- `playFireCrackle()` — 20 s white noise through bandpass (900 Hz crackle) + lowpass (180 Hz roar), slow fade-in/fade-out; fires once when a house starts burning
- `playCollapseSound()` — white noise swept 4000→60 Hz + sine boom 100→28 Hz, ~3.8 s; fires when a burning house collapses

---

## UI Panel

- **Esc key** toggles the panel overlay
- Panel shows: day number, clock time + phase name, population breakdown (children/teens/adults/elders/priest), house count, church status, grave count, mausoleum status (if exists), leavers count
- Sound toggle button (Sound: OFF/ON)
- Reset simulation button (with confirmation step)

## Dev Console

- **Backtick (`` ` ``) key** toggles a green dev overlay (top-left)
- Shows live stats: tod, gameTime, villager count, per-state counts, wolf/bear/ghost/wizard status, church ageMs/upgrades, grave count, mausoleum elaborateness
- Commands: `wolf` `bear` `wizard` `ghost` `ferret` `bell` `mausoleum` `kill` `enchant` `day N` `tod N` `reach` `mob` `help`
- `reach` — runs BFS reachability check, reports any blocked house approach points
- `mob` — immediately recruits a mob targeting a blocking house (or mid-list house as fallback)

---

## Persistence

- `saveState()` / `loadState()` — serialise/restore full game state to `localStorage` key `vs` every 14 s
- JSON encoded; gracefully ignores missing/corrupt data
- Persisted: `gameTime`, `arrivalCount`, `nextArrivalMs`, `nextId`, `leaverCount`, all villagers (dead/buried excluded, transitory states reverted to wander, `mob` saved as `wander`), all houses (including `ageMs`, `extensions`, `extProgress`, `extPending`, `animalsGiven` as key `ag`, burning state as `bn`/`bp`), church (including `ageMs`, `lastBellDay`), graveyard (position + crosses + `totalDead` under key `gy`), mausoleum (key `ml`), priest tomb (key `pt`), priest respawn pending flag (key `pr`), kitchen gardens (`kg`), ground track wear (`trkW`: sparse index→value) and pave (`trkP`: sparse index→value), ruins (`ru`: array of `{nx, ny, ws}`)
- Villager fields persisted include `partnerId`, `fertility`, `skinColor`, `shirtColor`
- Farm animals are **not** stored directly — `farmAnimals[]` is rebuilt from houses where `animalsGiven === true` on load
- Pond persisted under key `po` (`nx, ny, rx, ry`); ducks are **not** persisted — rebuilt from pond data on load
- Hand-hold state (`handPartnerId`, `holdHandMs`) is NOT persisted
- Angel sequence (`priestSequence`, `angels[]`) is NOT persisted — if a reload occurs mid-sequence, the dead priest body is excluded from the save, `priestRespawnPending` is saved, and the new priest arrives next morning

---

## Main Loop

`requestAnimationFrame` loop; `dt` capped at 100 ms to prevent time jumps.

Order per frame:
1. Accumulate `gameTime`
2. Auto-save check (every 14 s)
3. `updateArrivals()` — spawn checks, church/priest trigger
4. `checkEmigration()` — overpopulation emigration check (once per morning)
5. Update all villagers, houses, church, birds, dragon, wizard, wolf, ferret, bear, ghost, groundBirds, rabbits, kitchenGardens; then `checkFarmAnimals()` + update all `farmAnimals`; then `checkPond()` + update all `ducks`; then `updateTracks(dt)`
6. `checkMausoleum()` — triggers mausoleum if graves ≥ 100; then update mausoleum if it exists
7. `updatePriestSequence(dt)` — advances the angel sequence state machine; updates angel positions
8. `updateDeaths()` — assign carriers to dead bodies (skipping dead priests); remove `buried` and off-screen `emigrate` villagers
9. `checkMob(dt)` — timer-based mob recruitment; `checkBurning()` — convert burned-out houses to ruins; mob sound timer tick
10. `drawBackground()` — sky, stars, sun/moon, ground, trees, grass
11. Build `drawables` array — ruins + houses + kitchenGardens + church + mausoleum + priestTomb + villagers + wizard + wolf + ferret + bear + ghost + groundBirds + rabbits + farmAnimals + pond + ducks + grave crosses — sort by `sy` (painter's algorithm)
12. Draw sorted ground entities — each drawable wrapped in its own try/catch; a failing draw logs to console, resets `globalAlpha`, and does **not** abort the remaining drawables
13. Draw angels (above ground entities, below birds)
14. Draw sky entities on top: birds, dragon
15. `updatePanel()` / `updateDevCon()` — refresh overlays if open

---

## Key Constraints

- Single HTML file — no external dependencies, no frameworks, no build step
- Canvas-based 2D polygon rendering only (no images, no sprites, no CSS animations)
- ~256 distinct colours in the palette (defined in `const C = { ... }`)
- Day/night cycle: 2 minutes per full cycle (`DAY_MS = 120000`)
- All sounds quiet (max gain ≈ 0.07); Web Audio only, no audio files
- Responsive: all sizing derived from `W` and `H`, recomputed on `resize` event
- State persistence via `localStorage` across page reloads
- `noisyPoly` perturbation must always use a **fixed seed** — never `Math.random()` — to prevent per-frame flicker
- `ctx.ellipse()` radii must be guarded against zero (e.g. `if (gs > 0)` before cabbage draw) — zero-radius ellipse throws a `DOMException` in most browsers
- `poly(pts, fill)` expects `[[x,y],...]` — **never pass a flat array** `[x,y,x,y,...]`; this throws silently in the drawable try/catch, making the entity invisible
