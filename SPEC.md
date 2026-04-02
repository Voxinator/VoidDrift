# Void Drift -- Technical Specification

> **Last updated:** 2026-04-01
> **Source of truth:** `server.js` (single-file architecture)

---

## 1. Overview

Void Drift is a Canvas 2D space shooter with LAN multiplayer. Up to 4 players connect via Socket.io and fight cooperatively against alien enemies, asteroids, and a carrier boss -- while also being able to shoot each other. The background is a living space scene: a WebGL plasma nebula, a Canvas 2D starfield, and a full Canvas 2D game engine with mouse-following player ships, alien enemies, asteroids, bullets, lasers, and particles.

### Tech Stack

| Layer | Technology |
|-------|-----------|
| Server | Node.js (ESM), Express 5, Socket.io |
| Frontend | Vanilla JS, inline in a server.js template literal |
| Background L0 | WebGL (off-DOM canvas) -- plasma nebula |
| Background L1 | Canvas 2D (`#starfield`) -- stars + plasma composited via `drawImage` |
| Game L2 | Canvas 2D (`#game-canvas`) -- ships, enemies, asteroids, particles |

### Architecture

Everything is a single file: `server.js`. The Express server serves one HTML page via a template literal. All CSS, HTML, and JS for the frontend are inline inside that template literal. There is no build step and no bundler. The only npm dependencies are `express` and `socket.io`.

### Port

- Port **3800** (port 3737 is reserved for Project Home; all other projects use 3000-3736 or 3738+)

---

## 2. Multiplayer Architecture

### Host-Client Model

The first browser to connect becomes the **host**. The host runs the authoritative game loop -- all game state updates, physics, collision detection, spawning, and AI happen on the host. Subsequent browsers are **guests** that send input to the host and receive serialized state for rendering.

The server itself is a **relay**: it assigns player IDs, tracks which client is the host, relays messages between clients, and caches the latest game state for host migration.

### State Serialization

The host serializes the full game state each frame and broadcasts it to all guests via Socket.io. Guests deserialize and render directly -- they do not run their own game loop.

### Host Migration

If the host disconnects, the server promotes the next connected client to host. The cached game state is sent to the new host so the game continues without interruption.

### Guest Rendering

Guests receive the serialized game state and render it locally. Each guest runs its own rendering pipeline (WebGL plasma, Canvas starfield, Canvas game) but does not execute game logic. Mouse input is sent to the host for movement and aiming.

### Connection

- WebSocket only (no HTTP long-polling fallback) -- LAN latency assumed at 1-5ms
- Up to 4 simultaneous players

---

## 3. Players

### Multi-Player State

The game tracks a `players[]` array instead of a single player object. Each player has a unique ID assigned by the server via `nextId()`.

### Color Palette

| Player | Color |
|--------|-------|
| Player 1 | Cyan (`#00FFFF`) |
| Player 2 | Magenta (`#FF00FF`) |
| Player 3 | Yellow (`#FFFF00`) |
| Player 4 | Green (`#00FF00`) |

### Per-Player Death & Respawn

Each player has independent death and respawn timers. When one player dies, the others continue playing. Respawn and initial join use safe random spawn: random positions 150px+ from enemies, carriers, asteroids, and other players. Falls back to pure random if no safe spot found in 20 attempts.

### Entity IDs

All game entities (players, enemies, bullets, asteroids, etc.) use a `nextId()` system for unique identification. This is essential for multiplayer state sync and ownership tracking (e.g., which player fired which bullet).

### Player Ship Visual Design

- Arrow/chevron shape: tip at (12, 0), wings at (-10, -8) and (-10, 8), notch at (-6, 0)
- Fill: player color, Stroke: lighter variant (1px)
- Engine glow: white circle at (-7, 0), radius 2.5px
- Shadow glow: 12px blur in player color

### Movement

- Lerp factor scales with player level:
  - Level 1: `0.01` (nimble)
  - Level 2: `0.008` (slightly heavier)
  - Level 3+: `0.007` (floaty)
- Minimum speed floor: 1.8 px/frame. When the ship is >2px from mouse and lerp would produce movement slower than 1.8, it scales up to that floor. Prevents crawling near screen edges.
- Ship is clamped to screen bounds (margin = player radius)
- Idle detection: 5000ms since last mouse move
- When idle: drifts toward canvas center at lerp 0.01, target angle set to `-PI/2` (pointing up)
- When mouse leaves window: ship glides to a stop using stored momentum (`glideVx`/`glideVy`) with 0.995 friction per frame. Max glide speed capped at 4 px/frame. During glide, targetAngle is locked to current angle (ship holds heading). On mouse re-enter, normal tracking resumes.
- Glide velocity is captured as the ship's actual per-frame movement delta (`p.x - prevX`), no smoothing
- Mouse velocity spikes >50px/frame are ignored (prevents teleporting on re-entry)

### Rotation

- Target angle: `atan2(mouseY - playerY, mouseX - playerX)` -- always points toward mouse during drift; during normal movement, only when speed > 0.5 and not idle
- Smoothing: `angle += angleDiff * 0.12` per frame (angleDiff is wrapped to [-PI, PI])
- Stores `lastValidAngle` to hold orientation when mouse stops moving

### Auto-Fire

- Rate: every 700ms (`fireRate`)
- `fireTimer` starts at 500 (first shot within 200ms of spawn)
- Quick-draw: when a target newly enters range (`hadTarget` was false, now true), `fireTimer` jumps to at least `fireRate - 200`
- Target search range: 400px
- Target selection: `findTarget()` returns sorted candidates array (enemies priority 0, other players priority 0, asteroids priority 1, sorted by distance within priority). `autoFire()` iterates candidates and fires at first one passing cone check.
- Targeting cone: 20 degrees (`maxArc = 20 * PI / 180`). Cone check tests against both `p.angle` (current) AND `p.targetAngle` (intended) -- fires if either passes.
- Bullet spawns 14px ahead of player in aim direction

### Shield System

- Spawn shields: 1 (base shield)
- Max shields: 5 (additional shields come from shield pickups only)
- Only the base shield (first slot) recharges at 10000ms (10 seconds); shields above 1 are pickup-only
- Drift modifier: while the player is drifting, base shield recharge rate doubles to 5000ms (see Section 4, Drift Mechanic)
- Visual: orbiting dots at 30px radius, spaced `2*PI / maxShields` apart (72 degrees for 5 max), rotating at 2 rad/s
- Active shields: filled circles in player color (3px radius, alpha 0.9)
- Recharging indicator: only shown when shields < 1 (slot 0 only) -- empty circle (alpha 0.3) with pie-fill showing progress (alpha 0.6)
- No dim outlines drawn for empty pickup-sourced slots

### Hit Behavior

- If shields > 0: lose 1 shield, `sparkShield` particles at impact point, 300ms invulnerability
- If shields = 0: death
- During invulnerability: `blinkTimer` increments

### Death

- `explodePlayer()` particle explosion (50-80 particles in player color/white/blue)
- 4-6 debris pieces
- 3-4 ring waves (player color 120px, white 90px, blue 60px, optional 50px)
- `addExplosionZone(x, y, 150)` -- largest explosion zone
- `screenFlashAlpha = 0.25` (white flash overlay)
- `screenShake.intensity = 18`
- 80px AoE damage: kills nearby enemies (via `killEnemy` -> dying state), breaks asteroids, deals 5 damage to carrier

### Respawn

- 5000ms delay after death
- Safe random spawn location (avoids enemies, other players, and hazards)
- 3-phase warp-in animation over 1500ms total:
  - Phase 1 (0-500ms, `warpPhase` 0-0.33): Horizontal line grows from 0 to 100px, white with 15px shadow glow
  - Phase 2 (500-1000ms, `warpPhase` 0.33-0.66): Ellipse portal, X-radius shrinks 50->30, Y-radius grows 2->25. 3 nested inner ellipses in alternating purple/white
  - Phase 3 (1000-1500ms, `warpPhase` 0.66-1.0): Ship materializes. Portal circle shrinks from 30px. Ship fades in (opacity 0->1), scales from 2x to 1x
- Post-respawn: 2000ms invulnerability, shields restored to 1 (base shield only)

### Engine Trail

- 1-2 particles per call (`budgetCount(1, 2)`) behind ship
- Multi-ship levels: iterates once per sub-ship; levels > 1 randomly skip 50% of sub-ship trails (`Math.random() > 0.5`)
- Sub-ship trails offset perpendicular to ship angle (8px spacing)
- Spread: +/-0.4 radians from opposite of ship angle
- Speed: 1-3 px/frame
- First 2 particles white, rest in player color
- Life: 10-20 frames, radius 1.5-3, friction 0.95
- Reduced when idle (30% chance of spawning)

---

## 4. Drift Mechanic

The drift mechanic decouples movement from aiming, creating a rear-wheel-drive slide feel.

### Activation

- **Hold left mouse button** to enter drift
- Ship must be moving > 0.5 speed to activate
- Not available during idle state
- Cancelled on player hit (shields or death)

### Physics

- **On enter:** captures current lerp velocity as `driftVx`/`driftVy`
- **During drift:** ship coasts on captured velocity; rotation tracks mouse independently (aim decoupled from travel direction)
- **Trajectory curve:** 3% per-frame steering pull toward mouse direction (rear-wheel drift feel)
- **Friction:** `0.998` per frame -- drift slowly decays
- **Wall bounce:** drift velocity reverses at 30% on screen edge hits
- **On release:** normal lerp-to-mouse movement resumes immediately

### Shield Recharge Bonus

- While drifting, base shield recharge rate is doubled (5000ms instead of 10000ms)
- This gives drifting a tactical purpose beyond movement: trade aiming precision for faster shield recovery
- The bonus applies only while actively drifting; recharge timer reverts to normal rate on drift end

### Visual

- Engine glow dims (radius 2.5 -> 1.5, alpha 0.3, cyan tint)
- Drift trail replaces engine trail: 1-2 dim particles emitted opposite to drift direction
- Occasional perpendicular "tire" sparks

---

## 5. Player-vs-Player Combat

### Auto-Fire Targeting

Player auto-fire targets other players at the **same priority as enemies** (priority 0). When another player is in range and within the targeting cone, bullets will fire at them just as they would at an enemy. Targets are sorted by distance within the same priority tier.

### Bullet Collision

All player bullets carry an `ownerId` matching the player who fired them. Bullet-player collision checks skip the owner -- a player's own bullets pass through them. Bullets from other players deal damage normally.

### Shield Interaction

Shields absorb PvP hits identically to enemy hits: each hit consumes 1 shield, triggers `sparkShield` particles, and grants 300ms invulnerability. If no shields remain, the hit kills the player.

---

## 6. Player Bullets

- **Shape**: ellipse, 4px x 1.5px
- **Color**: owner's player color with 15px shadow glow
- **White hot center**: 2px x 0.8px white ellipse
- **Trail**: up to 4 positions, drawn as fading ellipses (alphas: 0.1, 0.2, 0.4, 0.6)
- **Speed**: 8 px/frame
- **Lifetime**: 90 frames
- **Radius** (collision): 4px
- **Impact**: `bulletImpact()` -- 8-12 spark particles in player color/white, speed 1-4, life 10-15
- **Owner tracking**: each bullet stores `ownerId` for PvP collision filtering

---

## 7. Alien Ships

**Two visual types:**

- **Saucer**: horizontal ellipse (halfWidth = size/2, halfHeight = size/4), with dome (upper half-ellipse at 0.5x/0.8x scale), and small wing extensions (6x4px rectangles)
- **Fighter**: diamond body (0.7x halfWidth), with two swept-back wing triangles extending to 0.7-0.8x halfWidth

**Size**: 28-36px (random). Collision radius = `size * 0.45`.

**5 colors and weapon types:**

| Color | Hex | Weapon |
|-------|-----|--------|
| Red | `#FF3366` | Pulse laser |
| Magenta | `#FF33FF` | Pulse laser |
| Yellow | `#FFFF33` | Pulse laser |
| Green | `#33FF66` | Photon torpedoes |
| Orange | `#FF9933` | Photon torpedoes |

**Spawn:**
- Rate: 2500-5000ms (random interval, re-rolled each spawn)
- Max: 12 enemies simultaneously (doubles to 24 at player level 4+). Up to 2 carriers spawn simultaneously at level 4. Carriers launch elites at double rate when enemy count is below 75% of max capacity. On player death at L4, cap reduces back to 12 but existing enemies beyond the cap persist naturally.
- Spawn location: random edge of screen (20-40px outside)
- Player exclusion: 100px minimum distance from any player (up to 10 attempts to find valid position)
- Fade-in: 500ms

**AI:**
- Initial velocity: 0.5-1.5 px/frame, angled toward center with +/-0.5 rad jitter
- Direction changes: every 3000-6000ms, new random angle and speed 0.5-1.5
- Lazy pursuit: 30% chance (`pursuit` flag) -- applies 0.3 acceleration toward nearest player per second
- Hover oscillation: `sin(time/333 + offset) * 0.5` on Y axis
- Despawn: when 100px+ off any screen edge (300px for elite enemies)
- **Behavior system**: `behavior` property defaults to `'normal'`. On first non-lethal hit, 33% chance each of: `'retreat'` (0.6 px/s acceleration away from player), `'aggressive'` (0.8 px/s acceleration toward player), or no change.

**Laser charge-up:**
- Laser-type enemies (red, magenta, yellow) glow white in the 1 second before firing
- `charging` property ramps from 0 to 1 over the last 1000ms before `fireTimer` reaches `fireInterval`
- Draw color: lerps each RGB channel from base color toward 255 (white) by `charging` factor
- Shadow glow: increases from base 10px to `10 + charging * 20` (max 30px) during charge-up
- Resets to 0 when the laser fires or when player is out of range

**HP**: 2. Damage flash: white color for 150ms. Enemies have a `dying` property (see Dying State below). On destruction: `explodeEnemy()` + `addExplosionZone(x, y, 120)` + 100 score. `explodeEnemy` also sets `screenFlashAlpha = 0.08` and `screenShake.intensity = 6`.

**Dying State:**
- On lethal hit, enemies enter a dying state instead of instant death (via `killEnemy` function)
- `dyingDuration`: random 500-2000ms
- During dying: drift forward at half velocity, slowly rotate (0.03 rad/frame), flicker orange/white, not collidable or targetable
- Fire/smoke particles every 40-100ms (colors: `#FF4400`, `#FF8800`, `#FFCC00`, `#FFFFFF`)
- Occasional mini screen shakes (2% chance per frame, intensity 2)
- After timer: `finalExplodeEnemy` -- full `explodeEnemy` particles + ring waves + explosion zone + shield drop
- Final explosion AoE: radius = `enemy.radius * 3`, damages players, other enemies (via `killEnemy` -- chain reactions possible), asteroids, carrier (2 damage)

**Shields:**
- Enemies have a `shields` property (0-5, `maxShields: 5`)
- Shields absorb hits before HP damage (each hit consumes 1 shield)
- Spawn shield chance based on player level: 0% at L1, 25% at L2, 50% at L3, 75% at L4+
- Carrier-launched enemies use the same shield chance table
- Enemies steer toward nearby shield pickups within 200px (1.2 acceleration toward pickup)
- Enemies can pick up dropped shields on contact (`pickupRadius + enemy radius`), gaining +1 shield
- Visual: colored orbiting dots around enemy (same style as player shields) -- enemy color circles at `radius + 8` px orbit, 2.5px dot radius, rotating at 2 rad/s, alpha 0.8 (faded by `fadeIn`)
- Shield angle tracked per enemy (`shieldAngle`), dots spaced `2*PI / maxShields` apart

**Asteroid avoidance:**
- Weak avoidance force: 0.02 strength, 30px buffer beyond collision radii (`e.radius + ast.radius + 30`)
- Much weaker than carrier avoidance (carrier uses 0.06 strength, 80px buffer)
- Enemies still collide with asteroids regularly

**Weapons -- fire interval**: 2000-4000ms (re-rolled each shot), only fires when a player is within 500px.

---

## 8. Photon Torpedoes (Enemy Bullets)

Fired by green (`#33FF66`) and orange (`#FF9933`) enemies.

**Burst firing:**
- 2-5 shots per burst (`randInt(2, 5)`)
- 500ms between shots in a burst
- `burstTimer` starts at 500 (fires first shot immediately when burst starts)

**Owner tracking:** Each torpedo stores an `owner` reference to the enemy that fired it, used to prevent self-hits in friendly fire collisions.

**Visual (2-pass draw):**
1. **Trail**: up to 5 positions, circles with increasing size and opacity (alpha: `index/length * 0.4`, radius: `baseRadius * (0.3 + 0.5 * index/length)`). Drawn without shadowBlur.
2. **Glowing core**: colored circle at full radius with single shared 20px shadowBlur, pulsing (`0.8 + sin(pulsePhase) * 0.2`), pulsePhase increments by 0.15/frame
3. **White hot center**: white circle at `radius * 0.4 * pulse` (drawn in same pass as core)

Rendered with `globalCompositeOperation = 'lighter'` (additive blending).

**Speed**: 3 px/frame, aimed at nearest player position at time of firing.

**Radius** (collision): 5px.

**Range limit**: 300px from start position. On range expiration:
- `bulletImpact()` particles
- Ring wave: 40px maxRadius, expandSpeed 3, alpha 0.7
- AoE damage check: if any player within 40px + playerRadius, hits player

**Cap**: max 20 enemy bullets on screen (raised to 30 during carrier barrages).

---

## 9. Lasers

Fired by red (`#FF3366`), magenta (`#FF33FF`), and yellow (`#FFFF33`) enemies.

**Behavior:**
- Instant appearance -- line from enemy position toward nearest player
- Max range: 250px (or distance to player if closer)
- Hit detection: on creation, checks `pointToSegDist(player, laserStart, laserEnd)` against `playerRadius + laserWidth`
- If hit: records hit point on the laser for flash effect, calls `hitPlayer()`

**Visual:**
- Line from (x1,y1) to (x2,y2) in enemy color
- Shadow glow: `12 * alpha` blur
- Composite mode: `lighter` (additive)
- On hit: white flash circle at hit point, radius 8-20px, fading over 200ms

**Fade:**
- Total life: 375ms
- Alpha: starts at 0.6 (`clamp(life/maxLife, 0, 1) * 0.6`), fades to 0

**Width animation:**
- First 100ms: expands from 3 to 6 (`3 + (elapsed/100) * 3`)
- After 100ms: shrinks from 6 toward 1 (`1 + 5 * alpha`)

**Cap**: max 8 lasers on screen.

---

## 10. Asteroids

**3 sizes:**

| Size | Base Radius | Color |
|------|------------|-------|
| Large | 50-60px | `#666666` |
| Medium | 30-35px | `#777777` |
| Small | 15-20px | `#888888` |

**Shape**: irregular polygon with 8-12 vertices. Each vertex at evenly-spaced angles, radius jittered by +/-30% (`rand(0.7, 1.3)`). Collision radius = `baseRadius * 0.9`. Stroke: `#999999`, 1px.

**Craters**: 3-4 per asteroid. Random position within 40% of baseRadius. Radius: 8-15% of baseRadius. Fill: `#444444` at 0.5 alpha.

**Spawn:**
- 1 large asteroid every 3000-6000ms
- Max 12 large asteroids
- Max 50 total asteroids
- Spawn outside screen edges (baseRadius + 10px off-screen)
- Drift toward center with +/-0.6 rad jitter, speed 0.5-1.0

**Breakup chain:**
- Large -> 2-3 medium (velocity = parent + random [-1.5, 1.5])
- Medium -> 2-3 small
- Small -> destroyed (no children)
- Large/medium destruction creates explosion zones (large: 100px, medium: 60px)
- Large destruction: `screenShake.intensity = 6`

**Rotation**: 0.008-0.026 rad/frame, random direction.

**Despawn**: when `baseRadius + 80`px off any screen edge.

---

## 11. Particle System

**Limits**: max 500 particles. When at cap, oldest particle (index 0) is removed via `shift()`.

**Particle properties:**
- `x, y` -- position
- `vx, vy` -- velocity
- `life, maxLife` -- countdown in frames
- `radius` -- size (shrinks by `*0.97` per frame if `shrink` is true)
- `color` -- hex string
- `alpha` -- computed as `life / maxLife`
- `glow` -- boolean, determines composite mode and shadowBlur
- `shrink` -- boolean
- `friction` -- velocity multiplier per frame (default 0.98)
- `gravity` -- added to vy per frame (default 0)
- `type` -- `'circle'`, `'spark'`, or `'debris'`
- `angle, angleVel` -- rotation (for debris)
- `vertices` -- array of {x,y} for debris polygon (optional; falls back to rectangle)

**Particle types:**
- **circle**: simple filled arc
- **spark**: motion-blurred line. Length = `sqrt(vx^2+vy^2) * 3` (min 2px). Line width = `max(0.5, radius * 0.5)`. Oriented along velocity direction.
- **debris**: rotating polygon. If `vertices` provided, draws polygon; otherwise draws rectangle `2*radius x radius`.

**Rendering:**
1. First pass: non-glow particles with `globalCompositeOperation = 'source-over'`
2. Second pass: glow particles with `globalCompositeOperation = 'lighter'` (additive blending)
3. **Performance LOD**: when particle count > 300, glow pass skips `shadowBlur` entirely

**Preset functions:**

| Function | Particle Count | Colors | Types | Usage |
|----------|---------------|--------|-------|-------|
| `engineTrail(x, y, angle)` | 1-2 | 2 white + rest player color | circle | Behind player ship each frame (per sub-ship) |
| `bulletImpact(x, y)` | 8-12 | 50% player color, 50% `#FFFFFF` | spark | When bullet hits something |
| `sparkShield(x, y)` | 15-25 | 60% player color, 40% `#FFFFFF` | spark | Shield absorbs hit |
| `explodeEnemy(x, y, color)` | 30-50 circles/sparks + 3-5 debris | 60% white + enemy color | circle/spark/debris | Enemy destroyed. Also spawns 2-3 ring waves. |
| `explodeAsteroid(x, y, size)` | 10-30 (varies by size) | 50% bright (`#CCBBAA`, `#FFDDBB`, `#FFFFFF` with glow:true), 50% muted (`#AA9977`, `#998866`, `#BB9955`) | 50% debris, 25% spark, 25% circle | Asteroid destroyed. |
| `explodePlayer(x, y)` | 50-80 circles/sparks + 4-6 debris | `#FFFFFF`, player color, `#0088FF`, `#00CCFF` | circle/spark/debris | Player death. Also spawns 3-4 ring waves. |
| `warpEffect(x, y, phase)` | 10-30 | `#FFFFFF`, `#CC88FF`, `#8844FF`, player color | circle | Respawn warp animation. |

---

## 12. Ring Waves

- **Max**: 12. When at cap, oldest is removed via `shift()`.
- **Properties**: `x, y, radius, maxRadius, expandSpeed, lineWidth, color, alpha`
- **Behavior**: radius grows by `expandSpeed` per frame. Alpha = `1 - radius/maxRadius`. LineWidth lerps from 2 to 0.5 as radius approaches maxRadius.
- **Rendering**: `globalCompositeOperation = 'lighter'`, 4px shadowBlur in ring color
- **Removed** when `radius >= maxRadius`

---

## 13. Explosion Zones (Plasma Bridge)

Created by `addExplosionZone(x, y, maxRadius)`:
- Default maxRadius: 120px
- `expandSpeed`: 6 px/frame
- `currentRadius` starts at `maxRadius * 0.3` (30% of max)
- Expands until reaching `maxRadius`, then starts fading
- `fadeDuration`: 2500ms
- `strength`: starts at 1.0, decays to 0 linearly over fade duration
- Passed to shader as `radius = (currentRadius * strength) / max(screenW, screenH)` (normalized)
- **Max 8** simultaneous zones (enforced by shader uniform array size)
- Removed when `strength <= 0`

---

## 14. Collision Detection

All collisions use circle-circle intersection: `gameDist(x1, y1, x2, y2) < radius1 + radius2`.

**Collision radii:**

| Entity | Radius |
|--------|--------|
| Player | 10px |
| Player bullet | 4px |
| Enemy | `size * 0.45` (typically 12.6-16.2px) |
| Carrier | 55px |
| Asteroid | `baseRadius * 0.9` (large: 45-54, medium: 27-31.5, small: 13.5-18) |
| Enemy bullet | 5px |
| Enemy bullet AoE | 40px (on range expiration) |
| Laser | point-to-segment distance check against `playerRadius + laserWidth` |
| Shield pickup | 20px (pickup radius for player collection) |
| Shield pickup AoE | 50px (mine explosion on expiration) |

**Collision matrix:**

| A | B | Outcome |
|---|---|---------|
| Player bullet | Enemy | Bullet destroyed, enemy takes 1 damage. If HP=0: enemy destroyed, +100 score, explosion zone. Bullet `ownerId` used for score attribution. |
| Player bullet | Other player | Bullet destroyed (skipped if `ownerId` matches target). Shield absorbs or kill. |
| Player bullet | Asteroid | Bullet destroyed, asteroid destroyed and breaks into children. Large/medium: explosion zone. Large: screen shake 6. +50 score. |
| Player | Enemy | Enemy destroyed (explosion zone), player hit. |
| Player | Asteroid | Asteroid destroyed and breaks, player hit. |
| Enemy | Asteroid | Both destroyed (enemy explosion zone, asteroid breaks). No score. |
| Enemy torpedo | Asteroid | Torpedo destroyed, asteroid destroyed and breaks. |
| Enemy torpedo | Other enemy | Friendly fire: torpedo destroyed, enemy takes 1 damage (2-HP system). Torpedoes skip their `owner` enemy. |
| Enemy torpedo | Player (direct) | Torpedo destroyed, player hit. |
| Enemy torpedo | Player (AoE) | On range expiration, if any player within 40 + playerRadius: player hit. |
| Laser | Player | Checked on creation only. If within range: player hit, flash at contact point. |
| Laser | Asteroid | Checked within 50ms of laser creation. Uses point-to-segment distance. If within `a.radius + l.width`: asteroid destroyed and breaks. |
| Player bullet | Carrier | Bullet destroyed; if carrier has shields, lose 1 shield + sparkShield; else `carrierTakeDamage(1)`. |
| Player | Carrier | Player hit (same as enemy body collision). |
| Asteroid | Carrier | Asteroid destroyed and breaks; `carrierTakeDamage(5)`. |
| Asteroid | Asteroid | Elastic bounce: push apart to resolve overlap, swap velocity along collision normal with 0.8 coefficient. |
| Shield pickup AoE | Player | On expiration (10s), 50px AoE: if player within 50 + playerRadius, player hit. |
| Shield pickup AoE | Enemy | On expiration, 50px AoE: enemies within range take 2 HP damage. Chain reactions possible. |
| Shield pickup AoE | Asteroid | On expiration, 50px AoE: asteroids within range destroyed and break into children. |
| Enemy explosion AoE | Player | If within `enemy.radius * 3`: player hit. |
| Enemy explosion AoE | Other enemy | If within range: `killEnemy` (dying state, chain reactions). |
| Enemy explosion AoE | Asteroid | If within range: asteroid destroyed and breaks. |
| Enemy explosion AoE | Carrier | If within range: 2 HP damage. |
| Player explosion AoE | Enemy | 80px radius: `killEnemy`. |
| Player explosion AoE | Asteroid | 80px radius: destroyed and breaks. |
| Player explosion AoE | Carrier | 80px radius: 5 HP damage. |
| Carrier chunk AoE | Player/Enemy/Asteroid | 60px radius each chunk. |
| Carrier final AoE | Player/Enemy/Asteroid | 150px radius. |

---

## 15. Shield Pickups

Shield pickups drop from destroyed enemies and can be collected for extra shields or left to expire as mines.

**Drop mechanics:**
- 30% chance to drop from any enemy death (called via `tryDropShield(x, y)` after every `explodeEnemy`)
- Tracked in `gameState.shieldPickups[]`

**Visual:**
- Glowing cyan pulsing diamond shape, rotating slowly (`pulsePhase * 0.3` rad)
- Outer glow: `#00FFFF` circle at `radius * 1.5 * pulse` with 15px shadowBlur, alpha `0.4 * pulse`
- Diamond body: 4-point shape, width = `radius * 0.6`, height = `radius` (12px), `#00FFFF`, alpha `0.9 * pulse`
- White center: 3px white circle, alpha `0.8 * pulse`
- Pulse: `0.7 + sin(pulsePhase) * 0.3` (pulsePhase increments by 0.08/frame)
- Rendered with `globalCompositeOperation = 'lighter'` (additive blending)

**Pickup behavior:**
- Pickup radius: 20px (collision = `pickupRadius + playerRadius`)
- Visual radius: 12px
- Fly over to collect: +1 shield, up to max 5
- Triggers `sparkShield` effect on collection

**Expiration (mine behavior):**
- Lifetime: 10000ms (10 seconds)
- Blinks in last 3 seconds (`sp.life < 3000`): visibility toggled by `sin(pulsePhase * 3) > -0.3`
- On expiration: `bulletImpact` particles + ring wave (50px maxRadius, expandSpeed 4, cyan)
- 50px AoE explosion damages players, enemies, and asteroids

---

## 16. Carrier

A large enemy mothership that spawns smaller enemies and fires torpedo barrages.

**Stats:**
- Radius: 55px, HP: 60, Color: `#FF0000`
- Shields: 8 (max 8), recharge rate: 8000ms
- Fade-in: 1000ms
- Rotation speed: 0.005 rad/frame

**Spawn:**
- Tracked in `gameState.carriers[]` array (converted from single `carrier` object for multi-carrier support)
- Up to 1 carrier at player levels 1-3; up to 2 carriers simultaneously at level 4+
- Spawns from random screen edge (70px outside), aimed toward center
- Drift speed: 0.3-0.5 px/frame

**AI:**
- Orbits the nearest player at sweet spot range (200-350px): steers away if closer than 200px, drifts toward if farther than 350px
- **Flanking AI**: when 2 carriers are active, they intelligently approach the player from opposite sides
- **Carrier-carrier repulsion**: strong repulsion prevents overlap (120px + combined radii separation)
- Asteroid avoidance: 0.06 strength, 80px buffer beyond collision radii
- Random drift direction changes every 5000-8000ms, speed 0.3-0.5

**Enemy launches:**
- Every 3000ms, launches a new enemy from a random point around the carrier perimeter
- Double launch rate when enemy count is below 75% of max capacity
- Launched enemies get a speed boost outward from carrier
- Launched enemies are **elites** (see Elite Enemies below)

**Elite Enemies (carrier-spawned):**
- 5 shields, 3 HP (vs normal 0-5 shields, 2 HP)
- Visual: white body with pulsing glow ring and halo effect
- Slow launch velocity (0.5 px/frame) but strong pursuit AI (1.5x normal pursuit strength)
- Barrage-aware pathfinding: detects and flees the carrier torpedo cone
- Wider despawn boundary (300px vs normal 100px)
- Stronger asteroid and torpedo avoidance than normal enemies

**Torpedo barrage (bullet-hell style):**
- Only activates when a player is alive and at level 4+, and nearest player is within 500px
- Alternating left/right cannons (25px perpendicular offset from center), not simultaneous
- 45-degree spread arc toward nearest player (random angle within arc per shot)
- 200ms between shots
- 6-12 shots per burst (`randInt(6, 12)`)
- 2-3.5s between bursts (`rand(2000, 3500)`)
- Variable speed: 3.5-5 px/frame
- Torpedo color: `#FF4400`, radius 5px
- Enemy bullet cap raised to 30 for carrier barrages (vs 20 normal)

**Damage:**
- Shields absorb hits first (1 shield per hit, triggers `sparkShield`)
- Then HP damage, with 150ms damage flash

**Death (multi-stage destruction):**
- On HP reaching 0, carrier enters dying state (`dying=true`, `alive=false`, `dyingDuration=4000ms`)
- Enemies retreat immediately
- Initial flash 0.3, shake 15
- Carrier is not targetable or collidable during dying state
- **Phase 1 (0-2s):** Internal explosions at accelerating intervals (200-400ms). Each spawns 8-15 particles, ring wave, explosion zone, shake, flash. Hull drifts at 30% velocity, spins at 3x `rotationSpeed` (accelerating).
- **Phase 2 (at 50%):** Big flash 0.4, shake 25, explosion zone 180px, ring waves. Hull breaks into 5-7 debris chunks flying outward. Shield pickups drop. Score awarded (+500).
- **Phase 3 (50-100%):** Chunks drift with trailing fire particles. Mini-explosions every 200-500ms. Each chunk detonates individually (15-25 particles, ring wave, explosion zone, AoE damage).
- **Finale:** When last chunk explodes -- flash 0.6, shake 35, 200px ring wave, 250px explosion zone, 150px AoE damage.
- Carrier AoE damages: players, enemies (chain reactions), asteroids.

**Shield recharge:**
- Carrier steers toward nearby shield pickups within 200px
- Shields recharge over time at 8000ms interval

**Despawn:** removed if 300px+ off any screen edge.

**Known fix (carrier freeze):** A `var ci` shadowing bug previously caused game freezes during dual carrier battles. The loop variable was renamed to eliminate the shadowing conflict.

---

## 17. Rendering Layer Stack

| Layer | z-index | Technology | Canvas | In DOM? | Purpose |
|-------|---------|-----------|--------|---------|---------|
| L0: Plasma | n/a | WebGL | `_plasma.canvas` | No (off-DOM) | Nebula effect, drawn onto starfield via `drawImage` |
| L1: Starfield | 1 | Canvas 2D | `#starfield` | Yes | Stars + plasma background composite |
| L2: Game | 2 | Canvas 2D | `#game-canvas` | Yes | Ships, enemies, asteroids, bullets, particles |

**Compositing flow:** Each frame, the starfield's `requestAnimationFrame` callback:
1. Clears the starfield canvas
2. Calls `updatePlasma()` which renders the WebGL plasma to its off-DOM canvas
3. Calls `sfCtx.drawImage(_plasma.canvas, 0, 0, sfW, sfH)` to paint the plasma as background
4. Draws stars on top

The game canvas (`#game-canvas`) has its own RAF loop via `masterLoop` which calls `updateGame()` then `renderGame()`.

---

## 18. Layer 0: Plasma Nebula (WebGL)

### Off-DOM Canvas

The plasma canvas is created via `document.createElement('canvas')` and never appended to the DOM. It is rendered by WebGL each frame and drawn onto the starfield canvas via `drawImage`. Context options: `{ alpha: true, premultipliedAlpha: false, preserveDrawingBuffer: true }`.

Prefers WebGL2, falls back to WebGL1.

### Resolution

Half native resolution: `Math.floor(window.innerWidth * 0.5)` x `Math.floor(window.innerHeight * 0.5)` (0.5x). Upscaled to full viewport by the `drawImage` call.

### GLSL Shader

**Vertex shader:** Simple passthrough; positions a full-screen quad from the attribute `a_position`.

**Fragment shader structure:**

1. **Simplex noise** (`snoise`): 3D simplex noise implementation (Ashima Arts style). Returns values in approximately [-1, 1].

2. **fBM** (`fbm`): 4 octaves of simplex noise:
   - Octave 1: `snoise(p) * 0.5`
   - Octave 2: `snoise(p * 2.0) * 0.25`
   - Octave 3: `snoise(p * 4.0) * 0.125`
   - Octave 4: `snoise(p * 8.0) * 0.0625`

3. **HSL to RGB** (`hsl2rgb`): Standard conversion function.

4. **Main function:**
   - Computes UV coordinates, applies aspect ratio correction
   - **Ship repulsion**: computed in unrotated screen space, scaled by `u_ship_alpha`. Ship position with drift offset computed in shader.
   - **Explosion cloud displacement**: up to 8 zones, each with noise-warped boundary, cutout via smoothstep
   - **Rotation**: applied to noise sampling coordinates only. Rotation speed: `0.00873` rad/sec (~0.5 deg/sec). Rotates around center of screen.
   - **Color computation**: Two fBM samples combined. Hue rotates fully every 30 seconds, spatially varied. Full saturation. Lightness varies with cloud density. Alpha modulated by cloud density, ship hole, and explosion holes.

### Uniforms

| Uniform | Type | Purpose |
|---------|------|---------|
| `u_time` | `float` | `performance.now() / 1000` |
| `u_resolution` | `vec2` | Drawing buffer dimensions (half-res) |
| `u_mouse` | `vec2` | Ship normalized position (X: [0,1], Y: [0,1] bottom-up) |
| `u_explosions[8]` | `vec3[8]` | Each: `(normalizedX, 1-normalizedY, radius)` |
| `u_explosion_count` | `int` | Number of active explosion zones (max 8) |
| `u_ship_alpha` | `float` | Ship presence factor (0-1). Scales repulsion radius, strength, and hole cutout. |

---

## 19. Layer 1: Starfield (Canvas 2D)

### Stars

- **Count**: 150
- **Properties per star**: `x`, `y`, `r` (1.0-2.5px), `opacity` (0.2-0.8), `speed` (0.1-0.4), `depth` (0-1), `twinkle` (random phase)

### Twinkling

Uses `sin(time * 0.001 + star.twinkle)`. When sine > 0.6 (~20% of the time), opacity is modulated for occasional brightening.

### Depth-Based Speed

Actual vertical speed: `star.speed * (0.5 + star.depth * 0.5)`. Stars wrap around vertically.

### Parallax

Stars offset by mouse position: `px = star.x + mouseX * 20 * star.depth`. Maximum parallax offset is 20px for depth=1 stars.

---

## 20. Game Loop

**Master loop:** `masterLoop(timestamp)` called via `requestAnimationFrame`.

**`updateGame(timestamp, mouseX, mouseY)`:**
1. Compute delta time; cap at 16ms if > 100ms
2. If paused, return early
3. Update mouse state (prevX/prevY, lastMoveTime)
4. `updateParticleBudget()`
5. `updatePlayer(dt)` (for each player)
6. `updateBullets(dt)`
7. `updateCarrier(dt)` (includes spawn, AI, enemy launches, torpedo barrage)
8. `updateEnemies(dt)` (includes spawn logic and firing)
9. `updateAsteroids(dt)` (includes spawn logic)
10. `updateEnemyBullets(dt)`
11. `updateLasers(dt)`
12. `updateShieldPickups(dt)`
13. `checkCollisions()`
14. `updateParticles(dt)`
15. `updateExplosionZones(dt)`
16. `updateRingWaves(dt)`
17. `renderGame()`

**`renderGame()` draw order:**
1. Apply screen shake translation
2. `drawAsteroids()`
3. `drawCarrier()` (if alive)
4. `drawEnemies()`
5. `drawBullets()`
6. `drawEnemyBullets()`
7. `drawLasers()`
8. `drawShieldPickups()`
9. `drawPlayer()` (for each player)
10. `drawParticles()`
11. `drawRingWaves()`
12. `drawHUD()`
13. `drawScreenFlash()`
14. Restore from shake
15. Draw performance overlay (outside shake transform)

**Pause:** `gameState.paused` is set by `P` key or `visibilitychange` (tab hidden).

---

## 21. HUD & Performance Overlay

- **Score**: bottom-right corner, `10px monospace`, `#00FFFF`, alpha 0.3. Text: `"SCORE " + gameState.score`.
- **Screen flash**: white full-screen rect, alpha decays by 0.05 per frame. Ticks even when player is dead.

**Keyboard controls:**

| Key | Action |
|-----|--------|
| `P` | Pause / unpause game |
| `Shift+P` | Toggle performance panel (expanded/collapsed) |
| `` ` `` (backtick) | Toggle debug mode (shown as "DEBUG" in perf overlay) |
| `1`-`4` | Set player level instantly (debug mode only) |
| `5` | Toggle god mode -- infinite health (shown as "DEBUG [GOD]" in perf overlay, debug mode only) |

All key handlers ignore keypresses when focused on INPUT, SELECT, or TEXTAREA elements.

**Debug mode:**
- Toggled with the backtick key
- When active, "DEBUG" is displayed in the performance overlay
- Keys 1-4 instantly set the player's level to the corresponding value
- Key 5 toggles god mode (infinite health); perf overlay shows "DEBUG [GOD]" when god mode is active

**Performance audit overlay** (press `Shift+P` to toggle):

- **Collapsed** (default): single line showing FPS, particle budget percentage, and `[P]` hint. Shows "PAUSED" suffix when game is paused.
- **Expanded**: full breakdown showing:
  - FPS + budget percentage
  - Per-subsystem **update timings** (ms/frame)
  - Per-draw-function **render timings** (ms/frame)
  - **Entity counts**: enemies, asteroids, bullets, eBullets, particles, lasers, rings, expZones, pickups, carrier
- **Color coding** for timing values: green (`#00FF88`) < 0.5ms, orange (`#FF9933`) 0.5-2ms, red (`#FF3366`) > 2ms
- **FPS color**: green > 50, orange 30-50, red < 30

---

## 22. Input Handling

### Mousemove

A single `mousemove` listener on `document` updates:
- `sharedMouseX`, `sharedMouseY` -- raw pixel coords (fed to game engine)
- `sharedMouseNX`, `sharedMouseNY` -- normalized [0,1] coords, Y-flipped (fed to plasma shader)
- `starfieldMouseX`, `starfieldMouseY` -- [-1, 1] range (fed to star parallax)

### Mouse State Flow

```
sharedMouseX/Y  --[masterLoop]--> updateGame(timestamp, mouseX, mouseY)
                                     --> gameState.mouse.x/y (with prevX/prevY tracking)
                                     --> updatePlayer reads gameState.mouse
```

In multiplayer, guest mouse input is sent to the host via Socket.io. The host applies it to the corresponding player in the `players[]` array.

### mouseleave / mouseenter

- `mouseleave`: sets `mouseInWindow = false`. Ship enters glide-to-stop mode.
- `mouseenter`: sets `mouseInWindow = true`. Normal mouse-tracking resumes.

### Idle Detection

Player is considered idle when `gameState.time - mouse.lastMoveTime > 5000`. When idle:
- Ship drifts toward canvas center
- Target angle resets to `-PI/2` (pointing up)
- Engine trail reduced (70% chance of skipping)
- Auto-fire still works if targets are in range and cone

---

## 23. Sound Effects (Web Audio API)

### Architecture

Pre-decoded `AudioBuffer` pool via Web Audio API. All sound files are decoded at load time into buffers. Playback creates a new `AudioBufferSourceNode` per instance -- unlimited polyphony, no channel contention.

### Spatial Audio Helper

`sfxAt(buffer, x, y)` plays a sound with distance-based volume and stereo panning:
- Volume scales inversely with distance from the listener (local player position)
- Stereo pan computed from horizontal offset relative to the player

### Sound Catalog

| Sound | Trigger |
|-------|---------|
| Blaster | Player bullet fired |
| Laser | Enemy laser fired |
| Photon torpedo | Enemy torpedo fired |
| Asteroid explosion | Asteroid destroyed |
| Ship explosion (6 variants) | Player or enemy death (randomized selection) |
| Shield pickup | Shield collected |

---

## 24. Background Music

- Auto-discovers Nebula `*.mp3` tracks from the `/sounds/` directory at startup
- Shuffled playlist per session -- no repeat until all tracks played
- Tracks play back-to-back with no gap, loops indefinitely
- Playback starts on splash screen Play button click (satisfies browser autoplay policy)

---

## 25. Splash Screen

Liquid glass splash screen displayed before game starts:
- "Void Drift" title with rotating color gradient text
- Frosted glass Play button
- Click starts the game loop and initiates music playback

---

## 26. CRT Post-Processing

All effects are pure CSS applied to the game canvas layers -- zero JavaScript performance cost.

| Effect | Implementation |
|--------|---------------|
| Scanlines | CSS overlay with repeating horizontal lines |
| Vignette | Radial gradient darkening screen edges |
| Neon color pop | CSS `filter: saturate() contrast() brightness()` |
| Phosphor glow | CSS glow effect on canvas elements |

---

## 27. Browser Resize

Game field, starfield canvas, and plasma WebGL canvas all resize correctly on `window.resize`. Canvas dimensions and viewport uniforms are recalculated to match the new window size.

---

## 28. Performance Optimizations

| Optimization | Detail |
|-------------|--------|
| **Glow sprite system** | Pre-rendered radial gradient sprites replace Canvas 2D `shadowBlur` (CPU Gaussian blur). Sprites are drawn via `drawImage` instead. FPS improved from ~38 to 120+. |
| **dt-normalized particles** | Particle life, friction, shrink, and ring wave expansion all normalized to a 60fps baseline (`dt60 = dt / 16.667`). Frame-rate independent at any FPS. |
| **Batched draw passes** | Enemy drawing split into 3 passes (bodies, charging indicators, shields). Shield drawing split into 2 passes (active, recharging). Lasers split into 2 passes. Minimizes GPU state changes from `globalCompositeOperation` and `shadowBlur` toggling. |
| **Eliminated JSON deep clone** | Guest rendering loop replaced `JSON.parse(JSON.stringify(gameState))` with a direct reference swap. Removes per-frame serialization overhead. |
| Plasma half-resolution | Rendered at 0.5x native, upscaled via `drawImage` |
| Off-screen culling | All draw functions skip entities more than 50px off-screen |
| Particle glow LOD | When particle count > 300, glow particles rendered without shadowBlur |
| Torpedo range limit | 300px max travel caps on-screen count; AoE detonation cleans up |
| Trail length limits | Player bullets: 4 positions, torpedoes: 5 positions |
| Entity count caps | Particles: 500, ring waves: 12, enemies: 12/24 (level-dependent), large asteroids: 12, total asteroids: 50, enemy bullets: 20 (30 during carrier barrages), lasers: 8, explosion zones: 8 |
| Delta time capping | If dt > 100ms (tab was hidden), cap to 16ms to prevent physics explosion |
| Visibility API pause | Game state frozen when tab is hidden |
| Oldest-recycle for particles/rings | `shift()` removes oldest when at cap |
| Dynamic particle budget | `_particleBudget` (0.03-1.0) scales all particle emission counts based on count pressure and FPS pressure |

---

## 29. File Structure

```
~/Projects/VoidDrift/
├── server.js          -- Express + Socket.io server + entire frontend (inline HTML/CSS/JS)
├── package.json       -- ESM, express + socket.io, scripts: start
├── package-lock.json  -- Lockfile
├── project.json       -- Project Home registration
├── CLAUDE.md          -- Claude Code instructions
├── SPEC.md            -- This file
├── README.md          -- Quick start guide
├── sounds/            -- Nebula *.mp3 background music tracks (auto-discovered)
└── node_modules/      -- Dependencies (express, socket.io)
```

---

## 30. Conventions & Constraints

- **Two npm dependencies.** Express and Socket.io only.
- **No build step.** No bundler, transpiler, or preprocessor.
- **Port 3800.** Port 3737 is reserved for Project Home.
- **Template literal escaping:** No backticks in inline JS. No unintended `${}`. Use `[].join('\\n')` for multi-line strings (e.g., GLSL shaders). Use string concatenation for dynamic HTML.
- **Code style in game code:** `var` (not `let`/`const`), regular functions (not arrow functions), `gameCtx` for game canvas 2D context, `sfCtx` for starfield context.
- **Utility functions:** `lerp`, `gameDist`, `rand`, `randInt`, `randChoice`, `clamp`, `angleWrap`, `pointToSegDist` -- all global, used throughout game code.
- **Entity IDs:** All entities use `nextId()` for unique identification across the multiplayer state.
