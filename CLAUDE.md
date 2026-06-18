# CLAUDE.md — Static (Roblox Project)

## What This Project Is

**Static** is a co-op survival horror game for Roblox (1–4 players). Players use elemental abilities (Fire, Water) to solve puzzles across a 5-room floor ("The Substation") while evading an AI entity called "The Hum." Full design spec lives in `STATIC Implementation Plan.md`. Ticket breakdown lives in `Static tickets guide.md`.

---

## Toolchain

| Tool | Version | Purpose |
|------|---------|---------|
| [Rojo](https://rojo.space/) | 7.4.4 | Syncs `src/` to Roblox Studio |
| [Wally](https://wally.run/) | 0.3.2 | Luau package manager |
| [Aftman](https://github.com/LPGhatguy/aftman) | — | Manages Rojo + Wally installs |

Install tools: `aftman install`  
Install packages: `wally install`  
Start sync: `rojo serve`

---

## Project Structure

```
static/
├── CLAUDE.md                        ← you are here
├── STATIC Implementation Plan.md   ← full game design + architecture spec
├── Static tickets guide.md          ← 49 ordered implementation tickets
├── default.project.json             ← Rojo mappings
├── aftman.toml                      ← toolchain pins
├── wally.toml                       ← Luau dependencies
└── src/
    ├── client/
    │   ├── init.client.luau         ← StarterPlayerScripts entry point
    │   ├── Controllers/
    │   │   ├── AbilityController.luau
    │   │   └── EntityFeedbackController.luau
    │   └── UI/
    │       ├── Hud.luau
    │       ├── DownedOverlay.luau
    │       └── LoadoutTerminal.luau
    ├── server/
    │   ├── init.server.luau         ← ServerScriptService entry point
    │   └── Services/
    │       ├── ProfileStore.luau    ← vendored directly (not via Wally)
    │       ├── PlayerDataTemplate.luau
    │       ├── DataService.luau
    │       ├── AbilityService.luau
    │       ├── EntityService.luau
    │       ├── RoomService.luau
    │       ├── DownedService.luau
    │       └── PartyService.luau
    └── shared/
        ├── Types.luau               ← all cross-module type exports
        ├── Net/
        │   └── Remotes.luau         ← ALL remotes defined here, nowhere else
        └── Config/
            ├── Elements.luau        ← ability balance data
            ├── Entities.luau        ← entity (The Hum) config
            ├── Rooms.luau           ← puzzle pool data per room
            ├── Economy.luau         ← charge/reward constants
            └── Places.luau          ← Hub + Substation place IDs
```

### Rojo Mappings (from `default.project.json`)

| Filesystem path | Roblox location |
|----------------|----------------|
| `src/client` | `StarterPlayer.StarterPlayerScripts.Client` |
| `src/server` | `ServerScriptService.Server` |
| `src/shared` | `ReplicatedStorage.Shared` |

---

## Code Conventions

### Language
- All Luau files use **`--!strict`** at the top.
- File extension is `.luau`, not `.lua`.

### Types
- All shared types live **only** in `src/shared/Types.luau`. Never redefine them locally.
- Import types with `local Types = require(game.ReplicatedStorage.Shared.Types)`.

### Remotes
- **All** `RemoteEvent` instances are created in `src/shared/Net/Remotes.luau`.
- No `Instance.new("RemoteEvent")` anywhere else in the codebase — ever.
- Clients listen; server fires to clients. Client→server via `Remotes.X:FireServer(...)`.

### Services (server-side)
- Each service is a plain module table: `local MyService = {}` ... `return MyService`.
- Services expose an `Init()` function called once from `init.server.luau` at startup.
- Services are **server-only** unless in `shared/`. Never `require` a server Service from a client script.

### Naming
- PascalCase for module names, types, and service tables.
- camelCase for local variables and function arguments.
- `SCREAMING_SNAKE_CASE` for top-level constants.

### Anti-patterns to avoid
- No `wait()` — use `task.wait()`.
- No `spawn()` — use `task.spawn()`.
- No `delay()` — use `task.delay()`.
- No hardcoded balance numbers in service files — all tunable values belong in `src/shared/Config/`.

---

## Architecture Rules

### Server is authoritative for everything that matters
- Cooldown timestamps, damage, puzzle state, entity state, downed/revive state.
- The client only predicts animations/VFX locally for responsiveness.
- Every `AbilityFire` is rate-limited server-side: max 1 per 0.1s per player, independent of cooldown checks.

### Entity (The Hum) is server-only
- `EntityService` state machine runs at ~10Hz on Heartbeat.
- Clients receive only `EntityStateChanged` events for feedback — never raw position data.
- No client-side copy of the state machine.

### Noise is the balancing lever
- **Every** successful `AbilityFire` calls `EntityService.NotifyNoise(noiseRadius, position, player)`.
- NoiseRadius values are defined in `Elements.luau` and must not be hardcoded elsewhere.

---

## Key Systems at a Glance

### Elements (V1)
| Ability | Element | Cooldown | NoiseRadius |
|---------|---------|----------|-------------|
| Ignite | Fire | 0s (hold) | 15 studs |
| Cinder Burst | Fire | 8s | 40 studs |
| Flare | Fire | 20s | 60 studs |
| Douse | Water | 0s (hold) | 10 studs |
| Mist Veil | Water | 15s | 5 studs |

### The Hum State Machine
```
Patrol → (hears/sees) → Investigate → (2.5s telegraph) → Hunt → (loses target 6s) → Lost → Patrol
```
- Sight: 60° cone, 25-stud range, requires raycast LOS.
- Hearing: BaseHearingRange (20) + ability's NoiseRadius.

### Downed / Revive
- Hum contact → Downed (45s bleed-out timer, crawl only, no abilities).
- Revive: teammate holds `E` for 3s within 4 studs — interruptible.
- Run Failed: all players downed simultaneously, or any bleed-out reaches 0.
- On Run Failed: players keep 50% of Charge earned so far.

### Puzzle Pools
- Each room has a `PuzzlePool` — one entry rolled per run at run start (`RoomService.RollPuzzles()`).
- Rolled selection is synced to all clients via `Remotes.PuzzlesRolled` so UI hints stay accurate.
- Room 3 (Control Room): both Fire + Water required within a 30s window.
- Room 5 (Extraction): both stations within a 10s sync window.

---

## Implementation Ticket Progress

Tickets are in `Static tickets guide.md`, ordered 1 → 49.

| # | Ticket | Status |
|---|--------|--------|
| 1 | Repository, Toolchain & Rojo Setup | ✅ Done |
| 2 | Place Structure & Universe Configuration | ⬜ |
| 3 | Shared Type Definitions (`Types.luau`) | ⬜ |
| 4 | Typed Remotes Module | ⬜ |
| 5 | ProfileStore Integration & Player Data Template | ⬜ |
| 6–49 | … | ⬜ |

Update the table above as tickets are completed.

---

## External References

- Full design spec: [`STATIC Implementation Plan.md`](./STATIC%20Implementation%20Plan.md)
- Ticket breakdown: [`Static tickets guide.md`](./Static%20tickets%20guide.md)
- ProfileStore repo: https://github.com/MadStudioRoblox/ProfileStore
- Rojo docs: https://rojo.space/docs/
- Wally docs: https://wally.run/


## Ticket Instructions

When doing the ticket please do not make unnecessary changes, do not rename existing variables or methods unless it is necessary to do so. Please provide all the necessary changes needed to fully and correctly solve the ticket, please do not go outside the scope of the ticket unless it is important to do so, and if it is then please let me know. Please do not overcomplicate things, be clean and simple. Please go through the ticket carefully and make sure to fully understand it and complete it and all its requirements. Please explain your changes to me and how they fully solve the ticket. After you provide your changes please answer the questions below:
1. Have we fully and correctly solved the ticket and ALL its requirements?
2. Are there any wrong or unnecessary or unneeded changes made?
3. Are there any changes we forgot to make for the ticket?
4. Are we 100% good?
5. Have we maintained structure and functionality with the changes provided for the ticket?
6. Are the changes simple, clean, correct, and effective?
7. Have we made the changes in the correct place or places?

PLEASE DO NOT OVERLOOK ANYTHING AND MAKE SURE TO UNDERSTAND EVERYTHING. Please make sure to uphold the structure and format of how we do code when you provide the changes. Also do NOT commit any changes, only I can commit and push code.

Give a short commit message of the changes made for the ticket. Then provide a step by step guide on how to test the changes made through our endpoints on postman and/or the mysql database if applicable.

## PR Review Instructions

Please answer the questions below:
1. Have we fully and correctly solved the ticket and ALL its requirements?
2. Are there any wrong or unnecessary or unneeded changes made?
3. Are there any changes we forgot to make for the ticket?
4. Are we 100% good?
5. Have we maintained structure and functionality with the changes provided for the ticket?
6. Are the changes simple, clean, correct, and effective?
7. Have we made the changes in the correct place or places?

PLEASE DO NOT OVERLOOK ANYTHING AND MAKE SURE TO UNDERSTAND EVERYTHING.

Please do a thorough review. Do not overlook or assume anything. If you have any questions or concerns then please ask me. Capture every detail, do not overlook anything. Please let me know of any wrong changes made, any wrong functionality, let me know of anything and everything no matter how small. Please make sure to also check the structure and format of the code added and confirm if it aligns with the existing structure and format we have in our codebase. Provide the comments needed. Please we have to make sure all is well and clean before shipping. If everything is perfectly done then please confirm to me so. Also make sure all is following the DRY principle. Make sure we have not gone out of ticket scope.

## Comments Instructions.

So we have some comments to work on for the pull request i made with the changes we made for the ticket, so please think and only work on a
comment if it is valid, correct, and important to implement. If the comment is NOT valid, incorrect, or unnecessary to implement, then do not do the comment and provide a response as to why the comment is NOT valid, incorrect, or unnecessary to implement. Otherwise if the comment is valid, correct, and important to implement, then implement it.