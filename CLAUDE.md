# Void Drift

## Overview
Canvas 2D space shooter with LAN multiplayer. Single-file architecture: server.js serves inline HTML/CSS/JS.

## Architecture
- Express server on port 3800
- Single GET / route serves the full game as inline HTML
- Socket.io for LAN multiplayer (WebSocket only, no polling)
- Three rendering layers: WebGL plasma (off-DOM), Canvas starfield (#starfield), Canvas game (#game-canvas)

## Multiplayer Model
- Host-client: first browser = host (runs authoritative game loop), subsequent = guests (send input, receive state, render only)
- Server is a relay — assigns player IDs, tracks host, relays messages, caches state for host migration
- LAN only (1-5ms latency assumed)

## Key Commands
- `npm start` — run the server
- Mouse controls ship movement (auto-fire)
- P — pause, Shift+P — performance overlay

## Port
Uses port 3800. Port 3737 is reserved for Project Home.
