# Void Drift — Testing Checklist

Start the server: `cd ~/Projects/VoidDrift && npm start`
Open in browser: `http://localhost:3800`

## A: Single Player (Host Alone)

- [ ] Ship follows mouse, auto-fires at enemies
- [ ] Shields absorb hits, recharge base shield after 10s
- [ ] Enemies spawn, pursue, fire (lasers + torpedoes)
- [ ] Asteroids spawn, break into smaller pieces, bounce off each other
- [ ] Shield pickups drop from enemies, collected on proximity
- [ ] Level-up progression (5 pickups per level, L1-L4)
- [ ] Carrier spawns at L3, fires torpedoes at L4
- [ ] Carrier multi-phase death (internal explosions, hull break, final boom)
- [ ] Player death: 5s wait, warp-in animation, 2s invulnerability
- [ ] Chain-reaction AoE explosions (enemy death damages nearby)
- [ ] Performance overlay (Shift+P), pause (P)
- [ ] Plasma background responds to ship position and explosions
- [ ] Score increments (+100 enemy, +50 asteroid)
- [ ] No console errors

## B: Two Players (Two Tabs on Same Machine)

- [ ] Second tab connects, shows second ship with different color
- [ ] Both ships visible in both tabs
- [ ] Moving mouse in tab 2 moves player 2 in both views
- [ ] Both ships auto-fire independently at nearest enemies
- [ ] Enemy AI targets nearest player
- [ ] Carrier torpedoes target nearest player
- [ ] Player 1 dies: player 2 continues, no disruption
- [ ] Both players dead: game continues (enemies still move)
- [ ] Shield pickup goes to nearest player
- [ ] Score is shared in both views
- [ ] Each player levels up independently
- [ ] Bullet colors match player colors
- [ ] No visible lag between tabs

## C: Host Migration

- [ ] Two tabs open (tab 1 = host, tab 2 = guest)
- [ ] Close tab 1: tab 2 becomes host, game continues
- [ ] Enemies/asteroids/carrier persist through migration
- [ ] Player 1's ship disappears
- [ ] Open tab 3: connects to tab 2 as guest, works normally
- [ ] Close tab 2: tab 3 becomes host (chain migration)

## D: Edge Cases

- [ ] All players dead simultaneously: game continues, respawn timers work
- [ ] Guest disconnects: host removes player cleanly
- [ ] Host disconnects during carrier death: migration preserves state
- [ ] Player joins mid-carrier fight: spawns with invulnerability
- [ ] 3-4 simultaneous players: all render with distinct colors
- [ ] Rapid connect/disconnect: no memory leaks or orphaned state

## E: Performance

- [ ] Host FPS stays near 60 during 2-player game (check Shift+P)
- [ ] WebSocket frame size < 5KB (check DevTools Network tab, WS frames)
- [ ] No serialization spikes in performance overlay

## F: Home Dashboard (Regression)

- [ ] `cd ~/Projects/Home && npm start` runs on port 3737
- [ ] Plasma/starfield background animates
- [ ] Project cards render (VoidDrift should appear as a card)
- [ ] Search, filter, sort work
- [ ] New project creation works
- [ ] No console errors
