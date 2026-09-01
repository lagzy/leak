leak

hide & seek — but unfair. a real-life hide-and-seek game for phones, played inside a fixed rectangle in Root (LU), Switzerland, with casino buffs, rule policing and hearts.

this file is the single source of truth for the project. read it fully before changing code.

what this project is

leak is a web app where friends meet in real life and play hide & seek with asymmetric, chaotic mechanics:

everyone starts at one meeting point.
players become hiders or seekers (chosen or random).
hiders get 10 minutes (configurable) to hide inside a fixed rectangle zone.
then seekers are released and hunt. catching a hider happens in real life, then the seeker taps catch in the app.
every 15 minutes a casino opens: spin → get 2 random buff offers → keep one. buffs sabotage the other side (scream orders, radar circles, ringing phones, slow-mo, decoys, …).
3 hearts per player. breaking rules (leaving the zone, ignoring a scream order, being reported) costs a heart. 0 hearts = automatic loss.
seekers win by catching all hiders before time runs out. hiders win by surviving.

current status: TEST PROTOTYPE (UI-complete)

there is no server and no budget. the app is 100% static and must stay deployable on GitHub Pages.
"multiplayer" is simulated/synced locally (see §5). the full game loop is playable solo with bots or across tabs of the same browser.
the map is real satellite imagery of the real zone.

file structure

leak/
├── index.html      ← THE app. single file: HTML + embedded CSS + embedded JS.
└── README.md       ← this file.

agent rule: keep it a single index.html. do not split into files unless we also add a build step (we won't). external libs only via CDN.

external dependencies (CDN only, no keys, no cost)

| lib | why |
|---|---|
| leaflet@1.9.4 (unpkg) | map rendering |
| Esri World Imagery tiles | free satellite tiles, no API key ("sky photo as on gmaps") |
| fontsource (jsdelivr) | Space Grotesk (UI) + JetBrains Mono (timers/codes) |

no frameworks, no build step, no bundler. vanilla JS only.

the play zone (hardcoded, geocoded from OSM Nominatim)

rectangle corners, all 6037 Root, Switzerland:

| # | address | lat | lng |
|---|---|---|---|
| 1 | Bahnhofstrasse 17 (default start point 🏁) | 47.1179626 | 8.3919364 |
| 2 | Luzernerstrasse 2 | 47.1161184 | 8.3929389 |
| 3 | Schulstrasse 4 | 47.1140680 | 8.3896612 |
| 4 | Oberwilstrasse 1 | 47.1161175 | 8.3959608 |

constants in code: CORNERS, bounding box Z (min/max lat/lng with ~25 m padding), START, helper inZone(p). the zone is drawn as a dashed white rectangle with a dark "outside" mask. players may not leave it — leaving triggers a banner and costs a heart after 20 s outside.

game flow / phase machine

home ── create/join ──▶ lobby ── host "lock roles & start" ──▶ hide ──▶ seek ──▶ end
                          ▲                                                        │
                          └────────────────── rematch (host) ◀─────────────────────┘

room.phase values: 'lobby' | 'hide' | 'seek' | 'end'

timestamps drive everything (any client can execute transitions; they're idempotent):

room.startedAt, room.hideEndsAt = startedAt + hideMin60k, room.gameEndsAt = startedAt + totalMin60k
hide → seek when now >= hideEndsAt
seek → end (seekers win) when no alive hiders remain
seek → end (hiders win) when now >= gameEndsAt

defaults: hide 10 min, total 45 min, casino every 15 min (lobby selects for hide/total).

round sequence details

lobby — room code (4 chars), role chips per player (hider / seeker / random, tap to cycle), host settings, "add test bot", zone preview map.
role reveal — full-screen overlay, shown once per game per tab (session flag leak-reveal-:).
hide phase — hiders set their hide spot ("hide here" → stores player.hidePos); seekers see a countdown.
seek phase — seekers see live seeker dots only (hiders invisible except via buffs); hiders see seeker dots live. catching = real-life find + tap catch in the players sheet.
end — winner overlay, per-player recap, rematch (host resets to lobby).

sync architecture (the unusual part — read carefully)

there is no server. state sync is best-effort and designed for the prototype:

authoritative-ish state object room is serialized to localStorage key leak-room- and broadcast via BroadcastChannel('leak-ch-').
every mutation goes through commit(fn) → mutate room → saveRoom() → renderAll(). saveRoom() bumps room.seq and calls transport.broadcast(room). commit() stays the only mutation funnel — that's what makes the transport swappable.

transport interface (Phase 3): the wire is pluggable. two ship:
  - LocalTransport (default) = BroadcastChannel('leak-ch-'+code) + localStorage 'leak-room-'+code + the 'storage' event fallback + the {ask: code} <-> {room} handshake. this is today's behavior, unchanged.
  - PeerJSTransport (opt-in, 'cross device (P2P)' toggle on the home screen) = free cross-device sync over WebRTC data channels via the public PeerJS Cloud signaling server (no account, no keys, no server cost). the room code is the peer id (leak-<CODE>); the host registers it, joiners connect to it; the host relays room snapshots to all peers, joiners push commits back to the host. PeerJS is lazy-loaded from CDN only when cross-device is selected, so same-browser play pays zero extra cost.
incoming rooms are merged, NOT wholesale-replaced: mergeRoom(incoming) unions players/fx/log by id (newer per-element seq wins) and applies envelope scalars only when the incoming room carries a higher seq. prevents two near-simultaneous commits from clobbering each other.
host presence is a heartbeat (transport.presence() every ~2 s); if no ping arrives in ~7 s, the oldest non-host alive player promotes itself (host migration) so bots keep moving and rematch still works.
foreground re-join: on hidden→visible (phone background/foreground), the transport rebuilds its channel from scratch — fresh websocket, resubscribe, re-track of room presence — so a host whose socket the OS suspended stays joinable instead of letting its room presence expire.
honest limits of PeerJSTransport (surfaced in-app, never a hard crash): symmetric-NAT peers can't connect without a paid TURN relay; ~6 players is the practical cap per room; PeerJS Cloud has no uptime guarantee. failures show a toast and fall back to same-browser mode.
window 'storage' event is a fallback sync path between tabs.
sandbox-proof storage: the page may run inside a sandboxed preview iframe where localStorage/sessionStorage throw SecurityError. ALL storage access goes through the wrappers LS / SS (makeStore()), which silently fall back to an in-memory Map. never use raw localStorage/sessionStorage anywhere.
if storage is blocked, joining a room uses a BroadcastChannel handshake: joiner posts {ask: code}, the tab holding the room replies with {room}.
host (room.hostId) is the only client that drives bot movement; phase transitions may be executed by any client.
cross-device play is opt-in via PeerJSTransport (see below). the default is same-browser.

identity

myId — persisted in localStorage (key leak.id) so a reopened tab reclaims its seat (Phase 1 #3: reconnection). each tab/browser = one player.
myName — persisted (leak.name).

state model

room
js
{
  code: 'ABCD',            // 4-char room code
  hostId: '…',             // player id of host
  phase: 'lobby|hide|seek|end',
  hideMin: 10, totalMin: 45, casinoMin: 15,
  startedAt, hideEndsAt, gameEndsAt,   // epoch ms
  winner: null | 'hiders' | 'seekers',
  forceCasinoId: 0,        // dev tool: non-zero opens casino for everyone once
  seq: 0,                  // room envelope version, bumped by saveRoom() (conflict-merge)
  players: [Player],
  fx: [Fx],                // map effects (radar circles, decoys)
  log: [{id, text}],       // toast feed, capped at 40
}

Player
js
{
  id, name, bot: bool,
  roleChoice: 'hider|seeker|random',  // lobby
  role: null | 'hider' | 'seeker',    // resolved at start
  hearts: 3, status: 'alive|caught|out|spectate',
  pos: {lat,lng}, speed,              // live position (GPS or sim)
  hidePos: {lat,lng} | null,          // hiders only
  casinoDone: n, casinoForce: n,      // casino window bookkeeping
  slowUntil, ghostUntil, ringUntil,   // epoch ms buffs/debuffs
  shiftReady: bool,
  pendingTask: null | {type:'scream', byId, deadline, holdMs:4000, dbMin:65},
  caughtBy: playerId | null,
  wander: {lat,lng} | null,           // bot nav helper
  seq: 0,                             // per-player version for merge
  joinedAt: epochMs                   // host-migration ordering
}

Fx (map effects)
js
{ id, kind: 'circle'|'fake', forId: playerId | 'seekers',
  center: {lat,lng}, radius?, until: epochMs, label? }

forId = only that player sees it (radar). 'seekers' = visible to all seekers (decoy).

buffs (casino payloads)

casino window: floor((now - startedAt) / 15min); player may spin once per window (casinoDone). dev tool can force a window (forceCasinoId). spin shows 2 distinct buffs from the player's role pool; pick one.

seeker buffs (BUFFS.seeker)

| id | effect |
|---|---|
| scream | target hider gets pendingTask: must hold-scream ≥ 65 dB for 4 s within 30 s (mic RMS via getUserMedia, holding-only fallback). fail/ignore → lose a heart. |
| radar | circle around a random hider on caster's map, 90 s. radius 80–420 m; 10% jackpot → ~10 m pinpoint. |
| ring | target's phone rings (soft beep loop + vibration) for 10 s. honor rule: can't silence. |

hider buffs (BUFFS.hider)

| id | effect |
|---|---|
| slow | a seeker is capped at 3 km/h for 90 s (slowUntil; GPS speed > ~3.3 km/h shows red banner). |
| decoy | fake pulsing hider blip on all seeker maps, 60 s. |
| shift | shiftReady=true → buff-bar "use" → slider modal → move hide spot ≤ 20 m (dest() polar math). |
| ghost | ghostUntil +120 s → immune to scream/radar/ring targeting. |

adding a buff: add entry to BUFFS[role] + a branch in applyBuff() (+ rendering if it needs map/HUD). keep effects honor-based or client-verifiable — no server exists.

hearts / rule system

3 hearts, UI in HUD (#hud-hearts) and players sheet.
loseHeart(pid, reason) → −1 heart, log event; at 0: hider → status:'caught', seeker → 'out'.
ways to lose hearts:
  1. leave the zone — auto-detected: banner + 20 s grace, then −1 heart (repeats with cooldown).
  2. fail/ignore a scream order — handled by the scream modal timer.
  3. reported by another player — players sheet → ⚠ report → confirm dialog. honor system, no validation.
⚙ dev panel has "lose a heart" for testing.

key code landmarks (index.html)

| area | symbols |
|---|---|
| constants/geo | CORNERS, Z, START, inZone, randInZone, distM, dest |
| storage-safe | LS, SS, makeStore, safeRef |
| sync | saveRoom, chanFor, commit, roomKey |
| lifecycle | createRoom, joinRoom, startGame, resetRoom, addBot |
| rules | loseHeart, doCatch, doReport, surrender, askConfirm |
| casino | casinoWinIdx, casinoOpenFor, openCasino, pickBuff, applyBuff |
| scream | ensureMic, micRms, openScream |
| render | showScreen, renderLobby, renderGame, renderSheet, renderReveal, renderEnd, renderAll, updateMap |
| loop | tick() (250 ms interval: phase transitions, fx prune, casino windows, ring loop, zone penalty, bot movement, HUD timers) |
| sound | tone() + presets (sNotif, sWarn, sGold, sCatch, sWin, sLose, sRing) — keep gain ≤ ~0.05, notifications must stay quiet; global muted toggle #btn-mute. |
| position | setPosMode(sim) — sim = tap map to move (default, for desktop testing), gps = watchPosition. |

DOM ids worth knowing

screens: scr-home, scr-lobby, scr-game · overlays: ov-reveal, ov-casino, ov-scream, ov-shift, ov-confirm, ov-how, ov-end · sheet (players bottom sheet) · dev (prototype tools) · toasts.

run & deploy
bash
local (any static server works; not even required)
python3 -m http.server 8080     # open http://localhost:8080
or just open index.html in the browser

GitHub Pages

push repo → index.html at root.
Settings → Pages → deploy from branch (main, /).
done. no env vars, no build, no keys.

testing matrix

solo: create game → add 2+ bots → start. use ⚙ dev panel: end hiding phase, open casino now, end game now.
multi-"player": open the page in 2–3 tabs of the same browser, join with the code.
on real phones: same device GPS via 🛰 gps toggle; on desktop use 📍 sim (tap map).

known limitations (by design, for now)

cross-device (PeerJSTransport) is free but fragile: symmetric-NAT users can't connect without a paid TURN relay, and ~6 players is the practical cap per room. failures fall back to same-browser mode.
rule enforcement is honor-based except: zone exit (GPS, with escalating grace on repeat exits) and scream task (mic/timer, with a retry button if the mic prompt was dismissed).
host migration and reconnection are implemented (see sync §5).
bots are simple: hider bots stand still, seeker bots wander toward hide spots and never auto-catch.
"seeker sees hider position" exists only via the radar buff (by design).
speed limit (slow) warns but doesn't punish automatically.

roadmap (in priority order)

free cross-device multiplayer is implemented (PeerJSTransport via PeerJS Cloud). the next upgrade path, if P2P proves too fragile in real play, is to swap the transport for a managed free-tier realtime DB (Supabase/Firebase) — a one-file change inside index.html since commit() is the only mutation funnel.
stricter enforcement: sustained overspeed → auto heart loss; scream dB calibration note.
hider "panic move" cooldown, seeker count-up pings, spectator cam.
i18n (EN/DE) — UI is currently English.

design rules (do not break)

ultra minimalistic & clean. off-white #f3f3f0 UI, ink #141518, hider mint #12c98f, seeker ember #ff4d2a, casino gold #f0b429. rounded pills, blurred dark HUD over the satellite map.
lowercase wordmark: leak (with pulsing orange drop on the home screen).
all transitions/animations smooth (cubic-bezier .22,.9,.3,1-family), no harsh flashes.
notification sounds must stay quiet (master gain ~0.04).
mobile-first, safe-area insets respected, no page scroll during game.
no external images/stock photos — map tiles and inline SVG/emoji only.