# STATIC — Full Implementation Plan

**Genre:** Co-op survival horror (1–4 players) with elemental ability-based puzzles and combat
**Platform:** Roblox
**V1 Scope:** 1 floor ("The Substation"), 2 elements (Fire, Water), 1 entity ("The Hum"), 5 rooms

---

## 0. Quick Reference

| Item | Value |
|---|---|
| Party size | 1–4 (loadout rules in §2.2) |
| Elements (V1) | Fire, Water |
| Elements (post-V1) | + Earth, Lightning |
| Entity (V1) | The Hum |
| Currency | Charge |
| Floor (V1) | The Substation (5 rooms) |
| Failure state | Downed → bleed-out 45s → Run Failed if whole team down |
| Target run length | 12–18 minutes |

---

## 1. Design Pillars & How This Plan Stays "Fun but Challenging"

This plan is built directly against the six failure modes identified earlier. Each one maps to a specific system below — if any of these systems get cut during development, revisit this table first.

| Risk (from earlier discussion) | System that addresses it |
|---|---|
| Puzzles become memorization | §2.5 Puzzle Pool randomization — each room draws from 2–3 possible element requirements per run |
| One element becomes "the" element | §2.3 Element specs — every ability has a Fire/Water tradeoff between puzzle utility, combat utility, and **noise** (which attracts the entity) |
| Entity feels unfair, not scary | §2.4 The Hum — all state transitions are telegraphed 2.5s before becoming dangerous |
| Escalation = bigger numbers | §6 Floor 2+ roadmap — new floors add new *behaviors* (Earth/Lightning, new entity), not just stat bumps |
| One mistake ends the run | §2.6 Downed/Revive — team-wide failure required, not individual death |
| No skill ceiling for repeat players | §9 Testing — efficiency/no-downs bonuses reward execution, not just completion |

---

## 2. Game Design Specification

### 2.1 Core Loop

1. Players spawn in **the Hub** (social lobby, capacity ~30–50).
2. Form a party of 1–4 via the Loadout Terminal.
3. Assign elements per the rules in §2.2; press **Ready**.
4. `PartyService` reserves a private server and teleports the party into **The Substation**.
5. Navigate Rooms 1→5 in sequence. Each room has one elemental puzzle gating its exit door, plus ambient threat from The Hum.
6. **The Hum** patrols, occasionally telegraphs investigation, and can transition to a Hunt state if it spots/hears a player.
7. Contact with The Hum → player enters **Downed** state (§2.6).
8. Reaching Room 5's extraction point and activating both stations within the sync window ends the run successfully.
9. Players return to the Hub with **Charge** earned, scaled by completion, speed, and no-downs bonus (§2.7, §9).

### 2.2 Party & Element Loadout Rules

V1 has two elements (Fire, Water). The loadout terminal enforces **coverage** — both elements must be represented in the party before "Ready" unlocks.

| Party size | Loadout rule |
|---|---|
| 1 | Player carries **both** Fire and Water, switchable via `1`/`2` hotkeys. Each element has its own cooldowns. |
| 2 | Each player picks exactly one element. Terminal blocks "Ready" until both Fire and Water are picked. |
| 3–4 | First two players must cover Fire and Water (as above). Remaining players pick freely (duplicates allowed — extra Fire/Water users add combat redundancy). |

This rule generalizes cleanly when Earth/Lightning are added post-V1: coverage requirement scales to "all unlocked elements represented for parties ≥ element count; smaller parties get multi-element loadouts."

### 2.3 Elements — V1 Detailed Specs

Every ability has three properties: **Cooldown**, **Effect**, and **Noise Radius** (how far The Hum can detect its use). This noise tradeoff is the core balancing lever — high-power abilities are loud.

#### Fire

| Ability | Type | Cooldown | Range | Effect | Noise Radius |
|---|---|---|---|---|---|
| **Ignite** | Puzzle | 0s (interaction-based, 1.5s hold) | 6 studs | Burns Fire Nodes (barricades, gas valves), provides 8-stud light radius for 10s | 15 studs |
| **Cinder Burst** | Combat | 8s | 30 studs | 15 dmg to entity weak points (staggers The Hum 3s, does not kill in V1) | 40 studs |
| **Flare** | Combat/Utility | 20s | Self (AOE) | Reveals The Hum's position for 2s if within 50 studs; staggers 1.5s | 60 studs |

#### Water

| Ability | Type | Cooldown | Range | Effect | Noise Radius |
|---|---|---|---|---|---|
| **Douse** | Puzzle | 0s (interaction-based, 1.5s hold) | 6 studs | Extinguishes Water Nodes (burning debris, overheated panels), fills hydraulic troughs | 10 studs |
| **Mist Veil** | Combat/Utility | 15s | Self (AOE, 10-stud radius) | Breaks The Hum's line-of-sight/tracking for 4s for everyone inside | 5 studs |

**Design intent:** Fire is loud-but-powerful (clears obstacles fast, hits hard, but draws The Hum). Water is quiet-and-defensive (low noise, breaks tracking, but doesn't damage). A team of all-Fire clears puzzles fast but gets hunted constantly; a team of all-Water is stealthy but can't clear half the puzzle pool. Coverage is mandatory specifically to prevent either extreme.

### 2.4 The Entity — "The Hum"

A state machine running server-side at ~10Hz.

```
Patrol → (hears/sees something) → Investigate → (telegraph, 2.5s) → Hunt → (loses target 6s) → Lost → Patrol
```

| State | Behavior | Player-facing telegraph |
|---|---|---|
| **Patrol** | Wanders between waypoints in rooms not currently occupied, via `PathfindingService` | Distant, low hum (ambient) |
| **Investigate** | Moves toward last known noise/sight source | Hum pitch rises, lights in the room begin to flicker — **this is the 2.5s warning window** |
| **Hunt** | Direct chase of the spotted player at 1.4x walk speed | Hum becomes a harsh tone, room lights turn red |
| **Lost** | Returns toward last-known position, then back to Patrol if nothing found within 6s | Hum fades back to ambient |

**Detection:**
- Sight: 60° cone, 25-stud range, requires raycast line-of-sight.
- Hearing: base 20-stud radius, modified by ability Noise Radius values (§2.3) when abilities are used nearby.

**Contact = Downed**, not instant death (§2.6).

### 2.5 Floor 1 — "The Substation" (5 Rooms, Randomized Puzzle Pools)

At run start, `RunService` rolls one configuration per room from its `PuzzlePool`. The room geometry stays the same between runs, but which element/station combination is "live" changes — preventing pure memorization while keeping the floor recognizable.

| Room | Theme | Puzzle Pool (one rolled per run) | Entity Presence |
|---|---|---|---|
| **1 — Entry Hall** | Tutorial-paced | A) Ignite a Barricade (Fire) — B) Douse Burning Debris blocking the same exit (Water) | Patrol only, low frequency |
| **2 — Flooded Maintenance** | Wider, water hazards | A) Douse overheated valve to drain a flooded trough, opening hydraulic door (Water) — B) Ignite a frozen valve wheel to free it (Fire) | First Investigate triggers possible |
| **3 — Control Room** | Combined puzzle | Both elements required: Ignite smoke-clogged vents AND Douse an overheating panel, in either order, within 30s of each other to unlock the door | Higher patrol density |
| **4 — The Catwalks** | Dark, narrow, avoidance-focused | Light source needed: Ignite 2 of 3 lamp stations (Fire) to see the path; Mist Veil recommended for crossing exposed sections | Hunt-state likely if noisy |
| **5 — Extraction Point** | Finale | Two stations (north/south), one needs Fire activation, one needs Water — both must be active within a 10s sync window to open the extraction door | Highest aggression; Hunt state probability increased |

### 2.6 Downed / Revive / Run Failure

- **On Hum contact:** player → **Downed**. Can crawl at 30% speed, cannot use abilities, has a **45-second bleed-out timer** (visible to teammates as a UI countdown).
- **Revive:** a teammate holds `E` for 3 seconds within 4 studs of a Downed player. **Interruptible** — if The Hum enters Hunt state targeting the reviver, the revive cancels. This is the primary "tension under pressure" moment of the game.
- **Run Failed:** triggers if (a) all players are simultaneously Downed, or (b) any player's bleed-out timer reaches 0 while no one else can reach them. Team is teleported back to the Hub.
- **Failure is not punishing toward meta-progression:** players keep 50% of Charge earned in-run-so-far even on a Run Failed. This keeps the loop "try again" rather than "that was wasted."

### 2.7 Progression & Currency

**Currency: Charge**, earned per run:
- Base reward per room cleared.
- **Extraction bonus** (full floor completion).
- **Speed bonus** (run completed under a target time — see §9 for benchmarks).
- **No-Downs bonus** (no team member entered Downed state during the run).

**ProfileStore-backed unlocks (V1):**
- Cosmetic "auras" per element (visual flair on ability use).
- Hub decorations (purely social/cosmetic).
- (Post-V1) Earth and Lightning element unlocks, new floor access.

---

## 3. Technical Architecture

### 3.1 Place Structure

| Place | Purpose | MaxPlayers |
|---|---|---|
| **Hub** | Social lobby, party formation, loadout selection, shop | 50 |
| **The Substation (Run)** | Loaded per-party via `TeleportService:ReserveServer` | 4 |

Both places belong to the same Universe so `DataStoreService`/`ProfileStore` and `MessagingService` work consistently across them.

### 3.2 Rojo Project Layout

```
static/
├── default.project.json
├── wally.toml
├── aftman.toml
└── src/
    ├── client/
    │   ├── init.client.luau
    │   ├── Controllers/
    │   │   ├── AbilityController.luau
    │   │   └── EntityFeedbackController.luau   -- audio/lighting telegraphs
    │   └── UI/
    │       ├── Hud.luau                        -- Vide component
    │       ├── DownedOverlay.luau
    │       └── LoadoutTerminal.luau
    ├── server/
    │   ├── init.server.luau
    │   └── Services/
    │       ├── PartyService.luau               -- Hub party formation + teleport
    │       ├── AbilityService.luau              -- validates + applies ability effects
    │       ├── EntityService.luau               -- The Hum state machine
    │       ├── RoomService.luau                 -- puzzle state per room
    │       ├── DownedService.luau                -- downed/revive/run-failed
    │       └── DataService.luau                 -- ProfileStore wrapper
    └── shared/
        ├── Net/
        │   └── Remotes.luau                    -- typed remote definitions
        ├── Config/
        │   ├── Elements.luau
        │   ├── Entities.luau
        │   └── Rooms.luau
        └── Types.luau
```

### 3.3 default.project.json (excerpt)

```json
{
  "name": "static",
  "tree": {
    "$className": "DataModel",
    "ReplicatedStorage": {
      "Shared": { "$path": "src/shared" }
    },
    "ServerScriptService": {
      "Server": { "$path": "src/server" }
    },
    "StarterPlayer": {
      "StarterPlayerScripts": {
        "Client": { "$path": "src/client" }
      }
    }
  }
}
```

### 3.4 wally.toml (core dependencies)

```toml
[package]
name = "yourname/static"
version = "0.1.0"
registry = "https://github.com/UpliftGames/wally-index"
realm = "shared"

[dependencies]
Promise = "evaera/promise@^4"
Janitor = "howmanysmall/janitor@^1"
TestEZ = "roblox/testez@^0.4"
```

> ProfileStore is typically dropped in directly as a single ModuleScript (per its official repo) rather than pulled via Wally — place it under `src/server/Services/ProfileStore.luau`.

### 3.5 Networking Principles

- All client→server communication goes through typed wrappers in `Shared/Net/Remotes.luau` (see §4.5).
- **Server is authoritative for everything that matters:** cooldown timestamps, damage, puzzle state, entity state, downed/revive state. The client only predicts animations/VFX locally for responsiveness.
- Every ability-fire event is rate-limited server-side (max 1 per 0.1s per player) independent of cooldown checks, to block spam exploits.
- `EntityService` never runs on the client — clients only receive **state-change events** (Patrol/Investigate/Hunt/Lost) for audio/lighting telegraphs, never raw position data faster than needed for rendering.

### 3.6 Data Flow (Ability Use Example)

1. Client: player presses ability key → `AbilityController` plays local animation/VFX immediately (perceived responsiveness) and fires `Remotes.AbilityFire:FireServer(abilityId, targetPosition)`.
2. Server: `AbilityService` checks (a) does player have this element unlocked/equipped, (b) is the per-player cooldown table clear for this ability, (c) is target within `Range`.
3. If valid: applies the effect (e.g., `RoomService:ApplyNodeEffect(nodeId, "Ignite")` or `EntityService:ApplyStagger(3)`), updates the player's server-side cooldown timestamp, and fires `Remotes.AbilityResolved` to all clients in the run for synced VFX/state.
4. If invalid: server silently ignores (no error feedback to a potentially-exploiting client beyond the normal UI cooldown state already shown).

---

## 4. Core Data Schemas & Module Code

### 4.1 Player Data Template (ProfileStore)

```lua
--!strict
-- src/server/Services/PlayerDataTemplate.luau

export type PlayerStats = {
    RunsCompleted: number,
    BestTimeSeconds: number?,
    TotalDowns: number,
}

export type PlayerData = {
    Charge: number,
    UnlockedElements: {[string]: boolean},
    Stats: PlayerStats,
}

local DEFAULT_DATA: PlayerData = {
    Charge = 0,
    UnlockedElements = {
        Fire = true,
        Water = true,
    },
    Stats = {
        RunsCompleted = 0,
        BestTimeSeconds = nil,
        TotalDowns = 0,
    },
}

return DEFAULT_DATA
```

### 4.2 Element/Ability Config

```lua
--!strict
-- src/shared/Config/Elements.luau

export type AbilityEffectType = "Damage" | "Stagger" | "Reveal" | "Conceal" | "Ignite" | "Douse"

export type AbilityEffect = {
    Type: AbilityEffectType,
    Value: number?,
    Duration: number?,
}

export type AbilityConfig = {
    Id: string,
    DisplayName: string,
    Element: "Fire" | "Water",
    Cooldown: number,      -- seconds
    Range: number,         -- studs, 0 = self/AOE
    Effects: {AbilityEffect},
    NoiseRadius: number,   -- studs The Hum can detect this use from
}

local Abilities: {[string]: AbilityConfig} = {
    Ignite = {
        Id = "Ignite", DisplayName = "Ignite", Element = "Fire",
        Cooldown = 0, Range = 6,
        Effects = { { Type = "Ignite" } },
        NoiseRadius = 15,
    },
    CinderBurst = {
        Id = "CinderBurst", DisplayName = "Cinder Burst", Element = "Fire",
        Cooldown = 8, Range = 30,
        Effects = {
            { Type = "Damage", Value = 15 },
            { Type = "Stagger", Duration = 3 },
        },
        NoiseRadius = 40,
    },
    Flare = {
        Id = "Flare", DisplayName = "Flare", Element = "Fire",
        Cooldown = 20, Range = 0,
        Effects = {
            { Type = "Reveal", Duration = 2 },
            { Type = "Stagger", Duration = 1.5 },
        },
        NoiseRadius = 60,
    },
    Douse = {
        Id = "Douse", DisplayName = "Douse", Element = "Water",
        Cooldown = 0, Range = 6,
        Effects = { { Type = "Douse" } },
        NoiseRadius = 10,
    },
    MistVeil = {
        Id = "MistVeil", DisplayName = "Mist Veil", Element = "Water",
        Cooldown = 15, Range = 0,
        Effects = { { Type = "Conceal", Duration = 4 } },
        NoiseRadius = 5,
    },
}

return Abilities
```

### 4.3 Entity Config — The Hum

```lua
--!strict
-- src/shared/Config/Entities.luau

export type EntityState = "Patrol" | "Investigate" | "Hunt" | "Lost"

export type EntityConfig = {
    Id: string,
    WalkSpeed: number,
    HuntSpeedMultiplier: number,
    SightAngleDegrees: number,
    SightRange: number,
    BaseHearingRange: number,
    InvestigateTelegraphSeconds: number,
    HuntLoseSeconds: number,
}

local TheHum: EntityConfig = {
    Id = "TheHum",
    WalkSpeed = 12,
    HuntSpeedMultiplier = 1.4,
    SightAngleDegrees = 60,
    SightRange = 25,
    BaseHearingRange = 20,
    InvestigateTelegraphSeconds = 2.5,
    HuntLoseSeconds = 6,
}

return TheHum
```

### 4.4 Entity State Machine (server skeleton)

```lua
--!strict
-- src/server/Services/EntityService.luau

local RunService = game:GetService("RunService")
local TheHumConfig = require(game.ReplicatedStorage.Shared.Config.Entities)
local Remotes = require(game.ReplicatedStorage.Shared.Net.Remotes)

type EntityState = "Patrol" | "Investigate" | "Hunt" | "Lost"

local EntityService = {}

local currentState: EntityState = "Patrol"
local stateTimer: number = 0
local huntTarget: Player? = nil

local function setState(newState: EntityState)
    if newState ~= currentState then
        currentState = newState
        stateTimer = 0
        Remotes.EntityStateChanged:FireAllClients(newState)
    end
end

local function canSeePlayer(player: Player): boolean
    -- Raycast + cone-angle check against TheHumConfig.SightAngleDegrees / SightRange
    -- Returns true if line-of-sight and within cone.
    return false -- placeholder
end

local function heardNoise(noiseRadius: number, sourcePosition: Vector3): boolean
    -- Check distance from The Hum to sourcePosition against
    -- TheHumConfig.BaseHearingRange + noiseRadius
    return false -- placeholder
end

local function update(dt: number)
    stateTimer += dt

    if currentState == "Patrol" then
        -- Move along waypoints via PathfindingService
        for _, player in ipairs(game.Players:GetPlayers()) do
            if canSeePlayer(player) then
                huntTarget = player
                setState("Investigate")
                break
            end
        end

    elseif currentState == "Investigate" then
        -- Move toward last-known source; telegraph already fired on state entry
        if stateTimer >= TheHumConfig.InvestigateTelegraphSeconds then
            if huntTarget and canSeePlayer(huntTarget) then
                setState("Hunt")
            else
                setState("Lost")
            end
        end

    elseif currentState == "Hunt" then
        -- Direct chase at WalkSpeed * HuntSpeedMultiplier
        if huntTarget and not canSeePlayer(huntTarget) then
            setState("Lost")
        elseif huntTarget then
            -- DownedService:SetDowned(huntTarget) on proximity contact
        end

    elseif currentState == "Lost" then
        if stateTimer >= TheHumConfig.HuntLoseSeconds then
            huntTarget = nil
            setState("Patrol")
        end
    end
end

function EntityService.Init()
    RunService.Heartbeat:Connect(function(dt)
        -- Throttle to ~10Hz
        update(dt)
    end)
end

function EntityService.NotifyNoise(noiseRadius: number, position: Vector3, source: Player?)
    if currentState == "Patrol" and heardNoise(noiseRadius, position) then
        huntTarget = source
        setState("Investigate")
    end
end

function EntityService.ApplyStagger(duration: number)
    -- Freeze movement for `duration`, used by CinderBurst/Flare effects
end

return EntityService
```

### 4.5 Typed Remotes

```lua
--!strict
-- src/shared/Net/Remotes.luau

local ReplicatedStorage = game:GetService("ReplicatedStorage")

local folder = Instance.new("Folder")
folder.Name = "Remotes"
folder.Parent = ReplicatedStorage

local function event(name: string): RemoteEvent
    local e = Instance.new("RemoteEvent")
    e.Name = name
    e.Parent = folder
    return e
end

return {
    AbilityFire = event("AbilityFire"),           -- client -> server: (abilityId: string, target: Vector3?)
    AbilityResolved = event("AbilityResolved"),   -- server -> clients: (abilityId, casterUserId, resultData)
    EntityStateChanged = event("EntityStateChanged"), -- server -> clients: (state: EntityState)
    PuzzleSolved = event("PuzzleSolved"),         -- server -> clients: (roomId, stationId)
    PlayerDowned = event("PlayerDowned"),         -- server -> clients: (userId)
    PlayerRevived = event("PlayerRevived"),       -- server -> clients: (userId)
    RunEnded = event("RunEnded"),                 -- server -> clients: (result: "Extracted" | "Failed", chargeEarned)
}
```

### 4.6 Downed/Revive Service (server skeleton)

```lua
--!strict
-- src/server/Services/DownedService.luau

local Remotes = require(game.ReplicatedStorage.Shared.Net.Remotes)

local BLEED_OUT_SECONDS = 45
local REVIVE_HOLD_SECONDS = 3

local DownedService = {}

local downedPlayers: {[Player]: thread} = {}

local function checkRunFailed()
    local players = game.Players:GetPlayers()
    for _, player in ipairs(players) do
        if not downedPlayers[player] then
            return -- at least one player is up; run continues
        end
    end
    Remotes.RunEnded:FireAllClients("Failed", nil)
    -- Trigger teleport back to Hub with 50% of in-run Charge
end

function DownedService.SetDowned(player: Player)
    if downedPlayers[player] then return end

    Remotes.PlayerDowned:FireAllClients(player.UserId)

    downedPlayers[player] = task.delay(BLEED_OUT_SECONDS, function()
        downedPlayers[player] = nil
        checkRunFailed()
    end)

    checkRunFailed()
end

function DownedService.AttemptRevive(reviver: Player, target: Player): boolean
    if not downedPlayers[target] then return false end

    -- Caller is expected to have already validated the 3s hold
    -- and proximity (≤4 studs) client-and-server-side via a
    -- channel/progress remote not shown here.

    task.cancel(downedPlayers[target])
    downedPlayers[target] = nil
    Remotes.PlayerRevived:FireAllClients(target.UserId)
    return true
end

return DownedService
```

---

## 5. System-by-System Implementation Notes

**PartyService (Hub):** Listens for "Ready" from the Loadout Terminal UI. Validates element coverage (§2.2) server-side — never trust the client's claimed loadout. Calls `TeleportService:ReserveServer(substationPlaceId)`, stores the access code keyed to the party's player list, then `TeleportService:TeleportToPrivateServer(...)` for all party members simultaneously.

**AbilityService:** Holds a per-player `{[abilityId]: lastUsedTimestamp}` table server-side. On `AbilityFire`, checks `os.clock() - lastUsed >= Cooldown`. On success, dispatches to the relevant effect handler (`RoomService` for Ignite/Douse, `EntityService` for combat effects) and calls `EntityService.NotifyNoise(noiseRadius, position, player)` for **every** ability use — this is what makes the Fire/Water noise tradeoff real.

**RoomService:** Holds the randomized `PuzzlePool` selection per room (rolled once at run start, synced to all clients so UI hints are consistent). Exposes `ApplyNodeEffect(nodeId, effectType)`; when a node's required effect is applied, marks the room's puzzle solved and fires `Remotes.PuzzleSolved`, which triggers the door-open animation/CanCollide change.

**EntityService:** As in §4.4 — server-only state machine. Movement uses `PathfindingService:CreatePath` between waypoint `Attachment`s placed per room during level building.

**DataService:** Thin wrapper around ProfileStore. `GetProfile(player)` returns the session-locked profile; `player.PlayerRemoving` releases it. Autosave interval left at ProfileStore's default (~300s) — no need to override for this scale.

**Client UI (Vide):**
- `Hud.luau` — subscribes to a local "cooldown state" signal updated whenever `AbilityResolved` is received for abilities the local player owns; renders cooldown rings.
- `DownedOverlay.luau` — full-screen desaturation + bleed-out timer, shown on `PlayerDowned` for the local player; hidden on `PlayerRevived`.
- `EntityFeedbackController.luau` — purely cosmetic: on `EntityStateChanged`, adjusts ambient hum volume/pitch and room `Lighting` (flicker on Investigate, red tint on Hunt). This is the primary "fear delivery" mechanism and should get disproportionate polish time relative to its code complexity.

---

## 6. Post-V1 Roadmap (Floors 2+)

| Addition | What's new (not just bigger numbers) |
|---|---|
| **Earth** element | Puzzle: create temporary platforms/cover. Combat: a stationary shield that blocks The Hum's sight cone for 6s (high cooldown, zero noise — the "safe" element). |
| **Lightning** element | Puzzle: power dead electrical panels. Combat: chains between up to 3 players, sharing a single cooldown — forces positioning coordination, very loud. |
| **Floor 2 entity — "The Warden"** | Adds a telegraphed **charge attack** (long wind-up, short very-fast dash) — a new threat *type*, not a stat increase on The Hum. |
| **Branching room order** | Floor 2 introduces 2 valid paths through 6 rooms, adding route-choice as a new layer of replayability. |

---

## 7. Monetization Setup

- **Group:** Publish under a dedicated Group from day one (per the original playbook) — even solo, this keeps revenue-split options open later.
- **Game Passes:**
  - *Cosmetic Aura Pack* — visual variants for ability VFX. Zero gameplay impact (avoids "pay to win" criticism in a co-op game where fairness matters to the group).
  - *Hub Decoration Pack* — purely social/cosmetic.
- **Developer Products:**
  - *Charge Boost* — currency pack, standard simulator-style monetization.
  - **Avoid** a "Revive Token" or anything that affects in-run survival — letting paying players bypass the Downed/Revive tension undermines the exact co-op mechanic that makes the game interesting, and creates resentment among non-paying teammates.
- **Rewarded Video (post-launch, once eligible):** "Watch an ad for a small Charge bonus after a Run Failed" — fits the genre's natural pacing (a brief pause after a failed run) without touching in-run balance.

---

## 8. Build Plan / Milestones

### Phase 1 — Foundations (1–2 weeks)
- [ ] Rojo + Git + Wally set up per §3.2–3.4
- [ ] ProfileStore wired with the §4.1 template; verify session locking with a two-server test
- [ ] Typed Remotes (§4.5) scaffolded, empty handlers on server

### Phase 2 — Core Loop, Gray-Box (3–4 weeks)
- [ ] Gray-box Rooms 1–2 with placeholder geometry
- [ ] Fire (Ignite, Cinder Burst) and Water (Douse, Mist Veil) abilities functional end-to-end, including cooldown validation and noise-radius hooks
- [ ] The Hum state machine (§4.4) functional with Patrol/Investigate/Hunt/Lost and telegraphs
- [ ] Downed/Revive (§4.6) functional
- [ ] **Milestone playtest:** 2–4 people, Rooms 1–2 only, no art. Validate per §9 benchmarks before continuing.

### Phase 3 — Full V1 Floor (3–4 weeks)
- [ ] Rooms 3–5 built, including Room 5's dual-station sync mechanic
- [ ] Puzzle pool randomization (§2.5) implemented and verified across multiple runs
- [ ] Flare ability + full noise-radius balancing pass
- [ ] Hub: Loadout Terminal with coverage enforcement, party formation, reserved-server teleport

### Phase 4 — Polish & Content (2–3 weeks)
- [ ] Vide HUD, Downed overlay, entity feedback (lighting/audio telegraphs) — prioritize this over additional content
- [ ] Audio pass: ambient hum layers per state, jump-scare stingers, ability SFX
- [ ] Cube 3D / 4D generation for room dressing; hand-modeled Hum character in Blender

### Phase 5 — Publish & Soft Launch
- [ ] Group, Game Passes, Developer Products configured (§7)
- [ ] Content maturity rating set honestly (Moderate, per horror content)
- [ ] Soft launch to a small group; begin tracking §9 metrics

---

## 9. Testing & Balance Protocol

Run every milestone playtest with people who have **not** seen the puzzle pools, and record:

| Metric | V1 Target | What it tells you |
|---|---|---|
| Run completion rate (new groups) | 60–70% | Below this = too hard/unfair; above ~85% = not challenging enough |
| Average run time | 12–18 min | Too short = floor is too easy/small; too long = pacing drags |
| Downs per run (average) | 1–3 | Zero downs across many groups = Hum isn't threatening enough |
| % of Hunt-state transitions that result in a Down | ~40–60% | If near 100%, Hunt is unescapable (unfair); if near 0%, Hunt has no stakes |
| Time-to-react after telegraph (observed) | Players should act within the 2.5s window at least half the time | If players consistently can't react in time, lengthen the telegraph; if they're never threatened, shorten it |

**Process:** watch sessions live, don't just collect post-run surveys. The moment a group goes quiet and frustrated (vs. tense-and-engaged) during a telegraph or puzzle is the signal to adjust — usually noise radius, telegraph timing, or puzzle pool variety, in that order.

---

## 10. LiveOps Hooks (Experiments API)

Expose these as Configs from day one so post-launch tuning doesn't require redeploys:

- `Hum.BaseHearingRange`, `Hum.InvestigateTelegraphSeconds`, `Hum.HuntLoseSeconds`
- `Abilities.<Id>.Cooldown`, `Abilities.<Id>.NoiseRadius`
- `Run.SpeedBonusThresholdSeconds`, `Run.NoDownsBonusMultiplier`
- `Room5.SyncWindowSeconds`

Use these for the rolling A/B tests described in the original playbook — e.g., test whether a slightly longer telegraph improves the completion-rate metric in §9 without dropping the "tense" feeling reported by playtesters.
