```
 __      __     _      _   ____         _   __  _   
 \ \    / /    (_)    | | |  _ \       (_) / _|| |  
  \ \  / /__    _   __| | | | | | _ __  _ | |_ | |_ 
   \ \/ // _ \ | | / _` | | | | || '__|| ||  _|| __|
    \  /| (_) || || (_| | | |_| || |   | || |  | |_ 
     \/  \___/ |_| \__,_| |____/ |_|   |_||_|   \__|
```

**A multiplayer space shooter where momentum is your weapon and everything explodes.**

---

## What is this?

Void Drift is a browser-based space shooter built for LAN multiplayer. Up to 4 players fly ships through a WebGL plasma nebula, fighting alien fleets and each other. The signature mechanic is the drift -- hold left-click to decouple your movement from your aim, sliding through space like a car on ice while still firing in any direction.

## Features

- **LAN multiplayer** -- up to 4 players in the same browser game, no account needed
- **Drift mechanic** -- hold left-click to decouple movement from aiming. Ship captures velocity and coasts with wall bounce, 3% steering, and near-zero friction. Cancelled on hit.
- **PvP combat** -- auto-fire targets other players at the same priority as enemies. Shields absorb PvP hits.
- **5 enemy types** -- saucers and fighters with lasers, photon torpedoes, shields, and adaptive AI
- **Elite enemies** -- carrier-spawned elites with 5 shields, 3 HP, white pulsing glow, barrage-aware pathfinding, and strong pursuit AI
- **Carrier boss** -- a mothership that spawns enemies and fires bullet-hell torpedo barrages. Up to 2 carriers at level 4 with intelligent flanking AI and carrier-carrier repulsion.
- **Level 4 scaling** -- max enemies doubles to 24, dual carriers spawn simultaneously with double launch rate when below 75% capacity. On death, cap drops back to 12 but existing enemies persist.
- **Chain-reaction explosions** -- dying enemies detonate in AoE blasts that trigger other deaths
- **Web Audio SFX** -- spatial sound with distance-based volume and stereo panning. Blasters, lasers, torpedoes, explosions, shield pickups, 6 randomized ship death sounds.
- **Background music** -- auto-discovers Nebula MP3 tracks from `/sounds/`, shuffled playlist, loops back-to-back
- **Liquid glass splash screen** -- rotating gradient title, frosted glass Play button. Starts music on click.
- **CRT post-processing** -- CSS scanline overlay, vignette, neon color pop, phosphor glow. All CSS, zero perf cost.
- **WebGL plasma background** -- full-screen procedural nebula with simplex noise, ship repulsion, and explosion displacement
- **Glow sprite system** -- pre-rendered radial gradient sprites replace Canvas 2D shadowBlur. FPS went from ~38 to 120+.
- **Frame-rate independent particles** -- all particle life, friction, shrink, and ring wave expansion dt-normalized to 60fps baseline

## Quick Start

```
git clone https://github.com/Voxinator/VoidDrift.git
cd VoidDrift
npm install
npm start
```

Open [http://localhost:3800](http://localhost:3800) in your browser. Open a second tab (or a browser on another machine on your LAN) for multiplayer.

## Controls

| Input | Action |
|-------|--------|
| Mouse | Move ship |
| Auto-fire | Fires automatically when enemies or players are in range and within the targeting cone |
| Left-click + drag | Drift -- decouple movement from aiming, coast on captured velocity |
| `P` | Pause |
| `Shift+P` | Performance overlay |
| `` ` `` (backtick) | Toggle debug mode |
| `1`-`4` | Set player level (debug mode) |
| `5` | Toggle god mode (debug mode) |

## Multiplayer

Void Drift uses a host-client model. The first browser to connect becomes the host and runs the authoritative game loop. Additional browsers are guests -- they send mouse input to the host and render the game state they receive back. The server is just a relay.

If the host disconnects, the next player is promoted and the game continues.

LAN only. Designed for 1-5ms latency. Up to 4 players.

## Tech

Single-file architecture. The entire game -- server, HTML, CSS, JavaScript, GLSL shaders -- lives in one `server.js` file. No build step. No bundler. No framework.

| | |
|---|---|
| **Runtime** | Node.js |
| **Server** | Express 5 |
| **Multiplayer** | Socket.io (WebSocket only) |
| **Rendering** | WebGL (plasma nebula) + Canvas 2D (starfield, game) |
| **Audio** | Web Audio API (pre-decoded AudioBuffers, spatial stereo panning) |
| **Dependencies** | 2 (express, socket.io) |

## Origin Story

This game was never supposed to exist. It started as [Project Home](https://github.com/Voxinator) -- a simple localhost dashboard to display all my project folders as cards. Clean, minimal, done in an afternoon.

But the background felt empty. So I added a WebGL plasma nebula -- procedural simplex noise with color cycling. Looked cool. Then twinkling stars on a parallax canvas layer. Even better. Then I thought it would be neat if a little ship followed your mouse cursor around while you browsed your projects.

Then the ship needed something to shoot at. So I added asteroids that broke into smaller pieces. Then enemies -- five color-coded types with different weapons and AI behaviors. Then shields, then a leveling system, then a carrier mothership boss that launches fighters and fires torpedo barrages. Then chain-reaction explosions. Then a drift mechanic because the movement felt too simple.

Two days later, the "background decoration" had a full game loop, a boss fight, and a particle system with dynamic LOD budgeting. The dashboard still worked fine -- the game just played silently behind it while you clicked around your projects.

At that point it deserved its own home. So I pulled it out, gave it a name, and added LAN multiplayer. Void Drift is what happens when scope creep wins and you just lean into it.

---

Built by [Voxinator](https://github.com/Voxinator).
