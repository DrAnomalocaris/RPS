# RPS World — Multiplayer Emoji Tag on a Shared Canvas

A tiny real‑time game built on Firebase Realtime Database. Players move emoji avatars on a 1000×1000 tile world, paint trails in their team color, and score by tagging weaker opponents via Rock–Paper–Scissors rules.

## How To Play
- Movement: `W`, `A`, `S`, `D` (accelerates velocity; you always drift until you change it)
- Camera: Follows your player. Use the mouse wheel to zoom.
- Objective: Touch another player you beat (Rock > Scissors, Scissors > Paper, Paper > Rock) to score +1.
- Respawn: If you are tagged by a stronger opponent, you respawn and switch to the killer’s team (their spawn zone), with a fresh color shade.
- Trails: Every tile you traverse is painted permanently on the shared world using your team’s color.

## Teams, Spawns, and Colors
- Rock (🪨): Spawns near the top‑center.
- Paper (📃): Spawns near the bottom‑right.
- Scissors (✂️): Spawns near the bottom‑left.
- Colors: Each team has a base color (Rock=red, Paper=green, Scissors=blue). Your personal trail color is a randomized shade near the team’s base.

## HUD
- Top center: Your Score and the live player count.
- Top right: List of players (you first), with an arrow and distance to each other player.
- Your name: Click your name (above the score) to set a display name (1–20 chars).

## Multiplayer Notes
- Open the game in two tabs (or share the page) to test multiplayer.
- Each tab gets a unique 5‑letter player id and writes only its own state.

## Running Locally
- Recommended: serve over `http://localhost` so ES modules and Firebase work reliably.
  - Python: `py -m http.server 5500` then visit `http://localhost:5500/`
- You can also open `index.html` directly (file://) — the app inlines Firebase config to avoid CORS for local testing.

## Firebase Setup
1) Enable Realtime Database in your Firebase project (note your instance URL).
2) Enable Anonymous Authentication: Console → Authentication → Sign‑in method → Anonymous → Enable.
3) Update Realtime Database rules (quick start):

```
{
  "rules": {
    "world": { ".read": true, ".write": "auth != null" },
    "world_meta": { ".read": true, ".write": "auth != null" },
    "players": { "$id": { ".read": true, ".write": "auth != null" } },
    "collisions": { ".read": "auth != null", ".write": "auth != null" }
  }
}
```

4) The app uses these RTDB paths:
- `world_meta` — world metadata/heartbeat
- `world/{xxxx_yyyy}` — tile colors (0xRRGGBB), keys are zero‑padded with `_`
- `players/{id}` — per‑player state (emoji, x, y, vx, vy, score, color, name, ts)
- `collisions/{loserId}/{collisionId}` — a winner posts a notice; the loser respawns when it sees the notice

5) Config is inlined in `index.html` and includes `databaseURL` so the app targets your RTDB.

## Collision Logic (Consistent Outcomes)
- Local detection: When you overlap an opponent, determine the RPS winner.
- Winner client:
  - Increments own `score`.
  - Posts `collisions/{loserId}/{randomId}` with `{ ts, from: winnerId, against: loserId }`.
- Loser client:
  - Listens at `collisions/{myId}`; when a note arrives, it respawns in the winner’s team area, resets its velocity, re‑jittters color, updates its own `/players/{myId}`, then deletes the note.

This ensures both players react to one source of truth (the collision note) without cross‑writing each other’s records.

## Troubleshooting
- Permission denied: Ensure RTDB rules above are in place and Anonymous auth is enabled.
- Players list shows only you:
  - Confirm rules allow `read` on `/players`.
  - Open DevTools; you should see logs like `[players] onValue snapshot received`.
- Trails paint but no players: `/world` may be readable while `/players` is not. Fix rules.
- No collisions: Slightly increase hit radius (`COLLIDE_MULT` in `index.html`) and check console for `[collisions]` logs.
- File:// warnings: Chrome may show “Permissions policy violation: unload is not allowed” from Firebase long‑poll — it’s benign.

## Notes
- The shared world can grow large; the client subscribes only to visible columns for live updates to stay light.
- For production, harden rules further (e.g., transactions, ownership checks, or a server function to arbitrate collisions).

Enjoy tagging!

