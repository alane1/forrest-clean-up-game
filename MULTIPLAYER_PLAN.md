# Multiplayer Plan — Forrest Clean Up

## Overview

This document covers the multiplayer architecture for a V1 "Forrest Friends" MMO mode,
V2 interaction features, security/performance concerns, and open questions.

The game is currently a single-player canvas game hosted on GitHub Pages (static only).
Multiplayer requires an external real-time backend since GitHub Pages cannot run server code.

---

## Backend Research Summary

Evaluated options for a free, static-hosting-compatible real-time backend with <10 concurrent users.

| Option | Free Tier | Latency | Complexity | Verdict |
|---|---|---|---|---|
| **Firebase Realtime DB** | 100 concurrent, 1GB, 10GB/mo | ~600ms | Low | ✅ Best fit for V1 |
| **PartyKit** (Cloudflare) | Unclear (check Workers pricing) | ~50ms | Low | ✅ Best if free tier works |
| **Supabase Realtime** | Unlimited connections but auto-pauses | ~50-100ms | Moderate | ⚠️ Not reliable on free tier |
| **Ably** | 3M msg/mo, 100 connections | ~50ms | Moderate | ⚠️ Message limits too tight |
| **Liveblocks** | 100-200 MAU | ~50ms | Low | ❌ Designed for collaboration, not games |
| **PeerJS / WebRTC** | Free but needs TURN server | ~40ms | High | ❌ Too complex, unstable behind NAT |

### Key Finding: Firebase for V1

Firebase Realtime Database is the recommended starting point:
- Genuinely free (no credit card, no auto-pause)
- 100 concurrent connections (plenty for <10 players)
- Simple SDK, loads from CDN — works with static GitHub Pages
- Uses WebSockets internally (not HTTP polling)
- ~600ms latency is acceptable for a cooperative cleanup game (not a twitch shooter)
- Shared game state = single JSON document all clients subscribe to

**If latency becomes a problem later**, migrate to PartyKit (purpose-built for games, Cloudflare-backed).

---

## V1 Architecture — "Forrest Friends" Mode

### Concept

A new game mode alongside the existing solo forest/winter modes.
All concurrent players share one live forest map and one live winter map.
Players see each other moving around and can clean up together.
Personal solo saves are untouched — localStorage continues to work as-is.

### What Gets Synced (Shared State)

```
shared-map/
  forest/
    tiles: { [tileId]: { type, litter, trees, ... } }   ← shared cleanup state
    players: { [playerId]: { x, y, name, avatar } }      ← player positions
    lastUpdated: timestamp
  winter/
    tiles: { ... }
    players: { ... }
    lastUpdated: timestamp
```

### What Stays Local (Not Synced)

- Solo forest/winter saves (localStorage, unchanged)
- Best score
- Settings (touch controls, etc.)
- Quest progress (V1 — can revisit)

### How It Works

1. Player enters "Forrest Friends" mode from the main menu
2. Game generates a short random player ID (e.g. `forest-abc123`) stored in sessionStorage
3. Game connects to Firebase and subscribes to the shared map
4. Player position is written to Firebase on each move (throttled to ~10 updates/sec)
5. Other players' positions are read from Firebase and rendered as colored dots with names
6. Tile cleanup actions (picking up litter, planting trees) are written to Firebase
7. All clients see the shared tile state — if Player A cleans a tile, Player B sees it cleaned
8. On disconnect, player's entry is removed from `players/` (Firebase `onDisconnect`)

### Rendering Other Players

Simple approach: draw colored circles with player names above them on the canvas,
layered on top of the existing tile render. No new animation system required for V1.

### Mode Entry UX

- New "Forrest Friends 🌲" button on the main menu
- Prompt for a display name (stored in sessionStorage)
- Show "X players online" counter
- Shared map is always the same map (not per-session) — it persists and evolves over time

### Map Initialization

The shared map is initialized once in Firebase with the same default tile layout as solo mode.
After that, it evolves as players clean it. No reset mechanism in V1 (can add later).

### Tech Stack (V1)

- Firebase Realtime Database (free Spark plan)
- Firebase JS SDK loaded from CDN (no build step)
- ~100-200 lines of new JS in `index.html`
- No new files, no build process, no server

---

## V2 — Interaction Features (Brainstorm)

Ideas for what players could do together beyond just seeing each other:

### Communication
- **Emoji reactions** — tap an emoji to broadcast a floating emoji over your character
- **Quick chat** — preset phrases ("Over here!", "Nice work!", "Help needed!")
- **Wave/emote** — press a key to play a wave animation visible to others

### Collaboration
- **Planting together** — two players on the same tile plant a tree faster
- **Heavy litter** — some litter requires 2+ players to remove
- **Campsite building** — campsites require contribution from multiple players
- **Supply drops** — one player finds seeds, another plants them

### Discovery & Exploration
- **Player trails** — faint path marks where other players have walked recently
- **Markers** — place a flag/marker at a point of interest visible to all
- **Fog of war** — only areas near a player are visible; team up to explore faster

### Progression & Social
- **Shared leaderboard** — top contributors to the shared map this week
- **Named player legacy** — planted trees show "Planted by [name]" on hover
- **Session recap** — "Today, 3 players cleaned 47 tiles and planted 12 trees"
- **Seasonal events** — timed community goals ("Plant 100 trees by Sunday")

### Meta / Fun
- **Player hats/colors** — choose your avatar color or hat at session start
- **Pet companions** — a small animal follows you around (cosmetic only)

---

## Security Concerns

### Firebase Rules (Critical)
- By default Firebase is open to anyone with the project config
- Must write Firebase Security Rules to limit what clients can write:
  - Players can only write to their own `players/{playerId}` entry
  - Tile state writes must be validated (valid tile ID, valid state transitions)
  - Rate limiting via rules (Firebase supports this with timestamps)
- Without rules, any user could overwrite the entire shared map

### Player Identity
- No auth in V1 = no real identity
- Player IDs are random and stored in sessionStorage
- A bad actor could impersonate any player ID if they know it
- Acceptable for V1 with trusted family/friend users
- V2 option: anonymous Firebase Auth (no signup, but gives a stable UID per device)

### Data Validation
- Tile writes should be validated: only valid tile types, positions within bounds
- Player position should be bounded to map dimensions
- Name input should be sanitized and length-limited

### Abuse / Griefing
- A user could spam-clean or spam-dirty tiles
- Rate limiting via Firebase rules (max 1 write per tile per second)
- For now with <10 known users, this is low risk

### API Key Exposure
- Firebase config (API key, project ID) will be visible in source
- This is normal and expected for Firebase — security comes from Security Rules, not key secrecy
- Restrict the API key in Google Cloud Console to only the GitHub Pages domain

---

## Performance Concerns

### Write Frequency
- Player position at 10 updates/sec × 10 players = 100 writes/sec to Firebase
- Firebase free tier has no explicit write rate limit, but recommend throttling to 5 updates/sec
- Tile updates are infrequent — not a concern

### Bandwidth
- Firebase charges for download bandwidth (10GB/mo free)
- Each connected client downloads all state changes from all other players
- At 10 players × 10 updates/sec × ~100 bytes per update = ~10KB/sec per client
- 10 clients × 10KB/sec × 3600sec/hr = ~360MB/hr
- Well within 10GB/mo free tier even with heavy use

### Firebase Latency (~600ms)
- Player movement will feel ~600ms behind for other players
- Acceptable for a cooperative cleanup game — not a competitive twitch game
- Local player movement is still instant (render locally, sync to Firebase)
- If this feels bad in testing, switch backend to PartyKit

### Canvas Rendering
- Other players are drawn as an overlay on the existing canvas render loop
- Should be minimal performance impact — just a few extra circles per frame
- No new render loop needed

### Disconnection Handling
- Firebase `onDisconnect()` removes stale player entries automatically
- Handles browser close, tab close, network drop
- Test this explicitly — ghost players are a bad experience

---

## Simplification Review & Proposals

### Current V1 Plan Simplifications Already Made
- ✅ No auth — just random player IDs
- ✅ No server — Firebase only
- ✅ No build step — CDN SDK
- ✅ No new files — all JS in `index.html`
- ✅ No custom signaling — Firebase handles all real-time routing

### Further Simplification Proposals

**Proposal A: Don't sync tile state in V1**
Instead of syncing the full shared map tile-by-tile, V1 could *only* sync player positions.
Players see each other moving around but each has their own map state.
Much simpler implementation. Validate the social presence feature before adding shared state.

**Proposal B: Poll instead of subscribe (even simpler)**
Instead of Firebase real-time subscriptions, poll the shared state every 2-3 seconds.
Feels less live but requires almost no Firebase SDK knowledge.
Can be upgraded to subscriptions later without changing the data model.

**Proposal C: Skip winter map in V1**
Only implement the shared forest map. Winter adds complexity with mode-switching.
Prove it works on one map first.

**Recommendation:** Start with Proposal A (positions only) + Proposal C (forest only).
If player presence works and feels fun, add shared tile state as V1.5.

---

## Open Questions

### Product
1. Should the shared map reset periodically (weekly, monthly) or evolve indefinitely?
2. What happens if the shared map is fully cleaned — is there a "win" state, or does it regenerate?
3. Should players be able to opt out of being visible to others (invisible mode)?
4. What player name/avatar options should be available at session start?
5. Should there be any persistence of contribution history (who cleaned what)?

### Technical
6. Which Firebase project should host this — a new one, or an existing one?
7. Should the shared map be seeded from the default solo map layout, or a custom curated layout?
8. What's the target frame rate for other-player position updates (smoothness vs. bandwidth)?
9. Should we support mobile (touch) players in Forrest Friends V1?
10. Is anonymous Firebase Auth worth adding in V1 for more stable player IDs, or skip?

### Scope
11. Should V1 target the lanelabs.github.io deployment or a separate test URL first?
12. What's the definition of "done" for V1 — what must work before considering it shipped?

---

## Recommended V1 Implementation Order

1. Set up Firebase project + Realtime Database + Security Rules
2. Add Firebase SDK to `index.html` via CDN
3. Add "Forrest Friends" mode button to main menu
4. Implement player name prompt + random player ID
5. Write player position to Firebase on move (throttled)
6. Read + render other players as colored dots on canvas
7. Implement `onDisconnect` cleanup
8. Test with 2+ browser tabs
9. If presence works and feels good → add shared tile state
10. Deploy to lanelabs.github.io

---

*Created: 2026-03-04*
*Status: Draft — pending review and question responses*
