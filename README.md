```
 __      __   _     _   ____         _   __  _   
 \ \    / /  | |   (_) |  _ \       (_) / _|| |  
  \ \  / /__ |_| __ _  | | | | _ __  _ | |_ | |_ 
   \ \/ // _ \  / _` | | | | || '__|| ||  _|| __|
    \  /| (_) || (_| | | |_| || |   | || |  | |_ 
     \/  \___/  \__,_| |____/ |_|   |_||_|   \__|
```

**A multiplayer space shooter where momentum is your weapon and everything explodes.**

---

## What is this?

Void Drift is a browser-based space shooter built for LAN multiplayer. Up to 4 players fly ships through a WebGL plasma nebula, fighting alien fleets and each other. The signature mechanic is the drift -- hold left-click to decouple your movement from your aim, sliding through space like a car on ice while still firing in any direction.

## Features

- **LAN multiplayer** -- up to 4 players in the same browser game, no account needed
- **Drift mechanic** -- decouple movement from aiming for rear-wheel-drive space combat
- **PvP combat** -- auto-fire targets other players at the same priority as enemies
- **5 enemy types** -- saucers and fighters with lasers, photon torpedoes, shields, and adaptive AI
- **Carrier boss** -- a mothership that spawns enemies and fires bullet-hell torpedo barrages
- **Chain-reaction explosions** -- dying enemies detonate in AoE blasts that trigger other deaths
- **WebGL plasma background** -- full-screen procedural nebula with simplex noise, ship repulsion, and explosion displacement

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
| **Dependencies** | 2 (express, socket.io) |

---

Built by [Voxinator](https://github.com/Voxinator).
