# leak.

*Hide & seek — but unfair.* A lightweight, static web app for real-life hide-and-seek with chaotic, asymmetric mechanics and small-team play inside a fixed rectangular play zone.

This repository is a single-page prototype: the entire app lives in index.html (HTML, CSS, and JS). Read this README to learn how it works, how to run it locally, and how to contribute.

---

## Quick links

- Live app: open `index.html` (no server required) or serve the repo via GitHub Pages.
- App file: `index.html` — everything is embedded in this one file.

---

## What is leak?

leak is a phone-first web app for friends to play hide & seek in a real-world rectangle (the default is a zone in Root, Switzerland). Players choose or are assigned roles (hider or seeker). Hiders get a hiding window, then seekers hunt. The game includes "casino" rounds that grant role-specific buffs (e.g. scream orders, radar glimpses, decoys) and a simple hearts-based rule system.

Key goals:
- Single-file, zero-build, GitHub Pages friendly.
- Runs fully client-side; sync is best-effort and opt-in for cross-device play.
- Mobile-first UI with a satellite map and small, quiet notification sounds.

Status: TEST PROTOTYPE (UI-complete)

---

## Features

- Single HTML file: index.html contains the whole app (no bundler).
- Offline / sandbox-safe storage wrappers (works in frames that block localStorage).
- Same-browser multiplayer via BroadcastChannel + localStorage sync.
- Optional cross-device P2P sync (PeerJS transport) — fragile by NAT/TURN limits; opt-in.
- Bots for local testing (hider bots stand still; seeker bots wander).
- Casino mechanic: periodic buff windows to pick role-specific effects.
- Rule enforcement: zone exit detection and scream tasks are client-enforced; reporting is honor-based.

---

## Quick start (local)

1. Clone the repo or download the ZIP.
2. Open `index.html` in your browser — it runs without a server.

Optional: run a local static server to avoid some browser restrictions:

```bash
python3 -m http.server 8080
# then open http://localhost:8080
```

To deploy on GitHub Pages: push `index.html` to the repository root and enable Pages in your repo settings (branch: main, folder: /).

---

## Gameplay (brief)

1. Home: create or join a 4-character room code.
2. Lobby: pick roles (hider / seeker / random), edit the 4-corner play zone, add test bots, and set host options (hide time, game length, casino frequency).
3. Reveal: each player sees a one-time reveal overlay of their role.
4. Hide phase: hiders set their hide spot inside the zone ("hide here"). Seekers wait a countdown.
5. Seek phase: seekers hunt in real life and tap *catch* for a found hider in the app.
6. Casino windows: every N minutes the casino opens and players may spin for two buff offers (pick one).
7. Hearts & rules: players have 3 hearts. Leaving the zone, ignoring a scream, or being reported costs hearts; 0 hearts = eliminated.
8. End: seekers win by catching all hiders; hiders win by surviving until time runs out.

---

## Architecture overview

The app is intentionally simple and client-only. The central state is the `room` object (serialized to `localStorage` as `leak-room-<CODE>`). All mutations go through a single `commit()` funnel which saves the room, bumps `room.seq`, renders the UI, and broadcasts the update via the chosen transport.

Transport implementations:
- LocalTransport (default): BroadcastChannel + localStorage + `storage` event fallback (same-browser sync across tabs).
- PeerJSTransport (opt-in): WebRTC data channels using PeerJS Cloud for signaling (cross-device sync). This is fragile on symmetric NATs and has practical limits (~6 players).

Merging: incoming room snapshots are merged (union of players, fx, log) with per-element seqs to avoid accidental overwrites when multiple clients publish simultaneously.

Host & presence: the host sends a presence ping. If host disappears, the oldest non-host alive player promotes itself so the game continues.

---

## Data model (summary)

room: {
  code, hostId, phase (lobby|hide|seek|end), hideMin, totalMin, casinoMin,
  startedAt, hideEndsAt, gameEndsAt, winner, seq,
  players: [Player], fx: [Fx], log: [{id,text}]
}

Player includes: id, name, roleChoice, role, hearts, status, pos, hidePos, buffs (slow/ghost/etc.), pending scream tasks, seq, joinedAt.

Fx: map effects (radar circles, fake blips, etc.) with expirations.

---

## Buffs (casino)

Every spin draws two cards, each from a rarity tier. Tiers are weighted: common ~36% · uncommon ~30% · rare ~20% · mythic ~10% · legendary ~4% · secret ~0.2%. If a tier has no card for your role, the draw falls to the next tier.

Seeker cards: ring (common), scan (uncommon — reveal the chosen hider's live spot to you for 20 s), radar (uncommon — a live circle around a random hider for 90 s), stun (rare — freeze the chosen hider for 45 s), scream (rare), leak spot (secret — every seeker sees the chosen hider's live hide spot for 90 s), freeze hider (secret — target can't move for 2 min).
Hider cards: decoy (common), mirage (uncommon — plant 3 fake decoy blips for 45 s), shift (uncommon), slow-mo (rare), cloak (mythic — vanish for 45 s: no radar, ring, scan, leak or catch), ghost (legendary — invisible to radar, ring and scream), seeker vision (secret — see all live seeker positions for 120 s), freeze seeker (secret — target can't move for 5 min).

Frozen players can't move (tap-to-move, GPS, hide spot, shift) until the timer runs out — a 🧊 chip with a countdown shows on their HUD. Cloaked hiders also disappear from the seeker catch list and leak markers.

Add a buff by extending the `BUFFS` table in the code (giving it a `rarity`) and implementing its behavior in `casApply()`.

---

## Development notes

- The entire app is inside `index.html`. Use the browser debugger and editor to iterate quickly.
- Key functions to inspect:
  - geo & zone: CORNERS, Z, START, inZone(), randInZone(), dest()
  - storage: LS, SS (sandbox-safe wrappers)
  - sync: commit(), saveRoom(), chanFor(), mergeRoom()
  - lifecycle: createRoom(), joinRoom(), startGame(), resetRoom(), addBot()
  - game rules: loseHeart(), doCatch(), doReport(), applyBuff(), openCasino()
  - scream: ensureMic(), micRms(), openScream()
  - render: renderLobby(), renderGame(), renderSheet(), renderReveal(), renderEnd(), renderAll()
  - loop: tick() — runs timers, bot movement, fx pruning

UI IDs worth knowing: scr-home, scr-lobby, scr-game, ov-reveal, ov-casino, ov-scream, ov-shift, ov-confirm, ov-how, ov-end, sheet, dev, toasts.

---

## Testing

- Solo: create a room, add 2+ bots, start. Use the dev panel to jump phases, open casino, or end the game.
- Multi-tab: open the same room code in multiple tabs to simulate players.
- Real devices: enable GPS mode on phones; on desktop use the default `sim` (tap to move) mode.

---

## Known limitations

- No server: sync is best-effort. The honor system is required for reporting and many rule decisions.
- PeerJS P2P is fragile under some NATs and may require TURN to be reliable.
- Bots are simple and not competitive (seekers wander; hiders stand still).
- Some enforcement is client-side only (reports are honor-based). Zone exit and scream tasks are client-detected.

---

## Contributing

- Keep it single-file unless you add a build step. The project rule: no bundler, no frameworks, CDN-only dependencies.
- Keep animation and sound levels subtle and mobile-friendly.
- Add features as small, well-tested JS blocks that integrate into the `commit()` flow and persist through `saveRoom()`.

If you want a deeper walkthrough of the code, say which area you want documented (sync, map, casino, or scream) and I'll draft a short guide.

---

## License

MIT — feel free to reuse parts of the code.
