# Static — Implementation Guide (Ticket Breakdown)

This guide decomposes **STATIC_Implementation_Plan.md** into a sequential set of
implementation tickets, **Ticket 1 → Ticket 49**, ordered so that each ticket's
dependencies are always lower-numbered tickets. Working through them in order
produces a fully functional V1 of *Static*, followed by QA/launch tickets and a
Phase 2 (post-V1) roadmap.

**How to read each ticket:**
- **Epic** — grouping for sprint planning
- **Depends on** — tickets that must be complete first
- **Description** — what this ticket delivers and why
- **Acceptance Criteria** — testable conditions for "done"
- **Implementation Notes** — concrete technical detail (configs, formulas, file paths)
- **Files** — primary files created/modified, per the Rojo layout in
  STATIC_Implementation_Plan.md §3.2

A suggested sprint grouping is provided at the end.

---

## EPIC 1 — Project Foundations

### Ticket 1: Repository, Toolchain & Rojo Setup
**Depends on:** none
**Description:** Establish the base project skeleton so all subsequent tickets have a working sync/build pipeline.

**Acceptance Criteria**
- Git repository initialized with `.gitignore` excluding `*.rbxl`, build artifacts.
- `aftman.toml` pins Rojo and Wally versions; `wally.toml` declares Promise, Janitor, TestEZ per §3.4.
- `default.project.json` maps `src/client → StarterPlayerScripts`, `src/server → ServerScriptService`, `src/shared → ReplicatedStorage` per §3.2–3.3.
- `rojo serve` connects successfully to a Studio session via the Rojo plugin; an empty placeholder script in each of `src/client`, `src/server`, `src/shared` confirms all three sync correctly.

**Implementation Notes**
- Use the exact folder layout from §3.2 — later tickets assume this structure exists.
- ProfileStore is added as a raw ModuleScript (not via Wally) in Ticket 5.

**Files:** `default.project.json`, `aftman.toml`, `wally.toml`, `src/client/init.client.luau`, `src/server/init.server.luau`, `src/shared/` (empty)

---

### Ticket 2: Place Structure & Universe Configuration
**Depends on:** 1
**Description:** Create the two-place Universe (Hub + The Substation) that all teleport/party logic depends on.

**Acceptance Criteria**
- Universe contains two places: **Hub** (`MaxPlayers = 50`) and **The Substation** (`MaxPlayers = 4`).
- Both places are published and reachable; both sync from the same Rojo project via separate `default.project.json` build targets or a documented build script.
- Place IDs for both are recorded in a config file (e.g., `src/shared/Config/Places.luau`) for use by `TeleportService` later (Ticket 25).

**Implementation Notes**
- `MaxPlayers = 4` on The Substation is a hard backstop — actual party-size enforcement happens in Ticket 24, but this property prevents any possibility of a 5th player landing in a run server even via direct join.

**Files:** `src/shared/Config/Places.luau`, Studio place configuration (documented in `README.md`)

---

### Ticket 3: Shared Type Definitions (`Types.luau`)
**Depends on:** 1
**Description:** Centralize all cross-module Luau types so every later module imports from one source of truth.

**Acceptance Criteria**
- `Types.luau` exports: `PlayerData`, `PlayerStats`, `AbilityEffect`, `AbilityEffectType`, `AbilityConfig`, `EntityState`, `EntityConfig`, `RoomConfig`, `PuzzleRequirement`.
- File compiles under `--!strict` with zero type errors.
- All type definitions match the field names/types used in STATIC_Implementation_Plan.md §4.1–§4.3 exactly (no renamed fields that would desync the plan from the code).

**Implementation Notes**
- `RoomConfig`/`PuzzleRequirement` are **not** fully specified in the original plan (only described in prose in §2.5) — this ticket is the first place they get a concrete type. See Ticket 19 for the populated config using these types.

```lua
export type PuzzleRequirement = {
    Element: "Fire" | "Water",
    AbilityId: string,    -- e.g. "Ignite", "Douse"
    StationName: string,  -- name of the target Instance in the room
}

export type RoomConfig = {
    Id: string,
    Name: string,
    PuzzlePool: {{PuzzleRequirement}}, -- one entry chosen per run
}
```

**Files:** `src/shared/Types.luau`

---

### Ticket 4: Typed Remotes Module
**Depends on:** 1, 3
**Description:** Single source of truth for all client↔server events, per §4.5.

**Acceptance Criteria**
- `Remotes.luau` creates a `Remotes` Folder under `ReplicatedStorage` containing: `AbilityFire`, `AbilityResolved`, `EntityStateChanged`, `PuzzleSolved`, `PlayerDowned`, `PlayerRevived`, `RunEnded`.
- Every later ticket that fires or listens to a remote does so **only** through this module — no ad-hoc `Instance.new("RemoteEvent")` elsewhere in the codebase.
- A code-review checklist item is added (see Ticket 41) to enforce this.

**Implementation Notes**
- Use the exact code from §4.5 as the starting point.

**Files:** `src/shared/Net/Remotes.luau`

---

### Ticket 5: ProfileStore Integration & Player Data Template
**Depends on:** 1, 3
**Description:** Stand up session-locked persistence before any system that needs to read/write player progression.

**Acceptance Criteria**
- `PlayerDataTemplate.luau` matches §4.1 exactly: `Charge: number`, `UnlockedElements: {Fire: true, Water: true}`, `Stats: {RunsCompleted, BestTimeSeconds, TotalDowns}`.
- `DataService.luau` loads a profile on `PlayerAdded`, exposes `DataService.Get(player): PlayerData?`, and releases the profile on `PlayerRemoving`.
- **Session-locking verified manually**: join a test server, disconnect abruptly, immediately join a second server — the second server does not get write access until the lock is released (no duplicate-write window). Document this test in `README.md` under "Verification Steps."

**Implementation Notes**
- Place ProfileStore's module file directly under `src/server/Services/ProfileStore.luau` per the official repo (not Wally).
- Autosave interval left at ProfileStore's default (~300s) — do not override.

**Files:** `src/server/Services/ProfileStore.luau`, `src/server/Services/PlayerDataTemplate.luau`, `src/server/Services/DataService.luau`

---

## EPIC 2 — Elemental Ability System

### Ticket 6: Element/Ability Config Data (Fire & Water)
**Depends on:** 3
**Description:** Encode the V1 ability data table — the single source of truth for balance numbers.

**Acceptance Criteria**
- `Elements.luau` defines `Ignite`, `CinderBurst`, `Flare`, `Douse`, `MistVeil` exactly matching §2.3/§4.2:
  - Ignite: Fire, Cooldown 0, Range 6, Effect `Ignite`, NoiseRadius 15
  - CinderBurst: Fire, Cooldown 8, Range 30, Effects `Damage 15` + `Stagger 3s`, NoiseRadius 40
  - Flare: Fire, Cooldown 20, Range 0, Effects `Reveal 2s` + `Stagger 1.5s`, NoiseRadius 60
  - Douse: Water, Cooldown 0, Range 6, Effect `Douse`, NoiseRadius 10
  - MistVeil: Water, Cooldown 15, Range 0, Effect `Conceal 4s`, NoiseRadius 5
- Table is keyed by ability ID and typed as `{[string]: AbilityConfig}`.

**Implementation Notes**
- This file is read by Ticket 7 (server validation), Ticket 8 (client UI), Ticket 13 (entity integration), and Ticket 45 (LiveOps overrides) — any future balance change happens here, not in scattered hardcoded values.

**Files:** `src/shared/Config/Elements.luau`

---

### Ticket 7: AbilityService — Cooldown Validation & Effect Dispatch
**Depends on:** 4, 5, 6
**Description:** Server-authoritative core of the ability system — every ability use passes through this.

**Acceptance Criteria**
- Server maintains a per-player `{[abilityId]: number}` table of last-used timestamps (`os.clock()`).
- On `AbilityFire(abilityId, target)`:
  - Reject if `player.UnlockedElements[Elements[abilityId].Element]` is falsy.
  - Reject if `os.clock() - lastUsed[abilityId] < Elements[abilityId].Cooldown`.
  - Reject if `target` distance exceeds `Elements[abilityId].Range` (when `Range > 0`).
  - Reject (silently) if more than 1 request per 0.1s from the same player, **independent of the above checks** (anti-spam floor).
- Valid requests update the cooldown timestamp and fire `AbilityResolved` to all clients in the run server.
- Effect dispatch is a stub in this ticket — actual effects wired in Tickets 13 (entity) and 22 (room nodes).

**Implementation Notes**
- Rejections must be silent (no error remote) — the client's own predicted cooldown UI already prevents most invalid presses; server rejection of an already-invalid press is a no-op, not an error case to surface.

**Files:** `src/server/Services/AbilityService.luau`

---

### Ticket 8: Client AbilityController — Input & Local Prediction
**Depends on:** 4, 6
**Description:** Responsive client-side input layer that fires server requests and predicts feedback.

**Acceptance Criteria**
- Hotkeys `1` (Fire ability) / `2` (Water ability) trigger immediate local animation/VFX placeholder on press (before server response).
- `Remotes.AbilityFire:FireServer(abilityId, target)` is called on press; `target` is computed from camera raycast for ranged abilities, `nil` for self/AOE abilities.
- Local cooldown UI state starts counting down **immediately on press** (optimistic), but is corrected if `AbilityResolved` is never received within 0.5s (server silently rejected) — in that case the cooldown UI resets to ready.
- For party-shared element loadouts (≥3 players, duplicate elements possible per §2.2), `AbilityResolved` events for *other* players' abilities do not affect the local player's own cooldown UI.

**Implementation Notes**
- This ticket only covers input plumbing — final VFX/SFX come in Ticket 37, final HUD in Ticket 34.

**Files:** `src/client/Controllers/AbilityController.luau`

---

## EPIC 3 — Entity AI: The Hum

### Ticket 9: Entity Config Data
**Depends on:** 3
**Description:** Encode The Hum's tunable parameters per §4.3.

**Acceptance Criteria**
- `Entities.luau` exports `TheHum: EntityConfig` with exactly: `WalkSpeed = 12`, `HuntSpeedMultiplier = 1.4`, `SightAngleDegrees = 60`, `SightRange = 25`, `BaseHearingRange = 20`, `InvestigateTelegraphSeconds = 2.5`, `HuntLoseSeconds = 6`.

**Implementation Notes**
- Floor 2's "The Warden" (Ticket 48) will add a second `EntityConfig` entry to this same file — design the export as `{[string]: EntityConfig}` even though V1 only populates `TheHum`, to avoid a breaking refactor later.

**Files:** `src/shared/Config/Entities.luau`

---

### Ticket 10: EntityService State Machine Skeleton
**Depends on:** 4, 9
**Description:** Server-only state machine driving The Hum's behavior, per §2.4/§4.4.

**Acceptance Criteria**
- States `Patrol | Investigate | Hunt | Lost` implemented as the exact cycle: `Patrol → Investigate → Hunt → Lost → Patrol` (with `Investigate → Lost` if target lost before telegraph completes, and `Hunt → Lost` if line-of-sight lost).
- `setState()` fires `EntityStateChanged` to all clients **only on actual state changes** (not every tick).
- Update loop runs on `Heartbeat`, throttled so AI logic executes at ~10Hz regardless of frame rate (accumulate `dt`, run logic when accumulator ≥ 0.1s).
- No client-side copy of this state machine exists — verified by code review (Ticket 41 checklist).

**Implementation Notes**
- `canSeePlayer()` and `heardNoise()` are stubs in this ticket, returning `false` — implemented in Ticket 11.
- Movement logic is a stub in this ticket — implemented in Ticket 12.

**Files:** `src/server/Services/EntityService.luau`

---

### Ticket 11: Perception — Sight Cone & Hearing Radius
**Depends on:** 10
**Description:** Implement the two detection functions that drive `Patrol → Investigate` transitions.

**Acceptance Criteria**
- `canSeePlayer(player)`: true only if (a) player is within `SightRange` (25 studs), (b) the angle between The Hum's forward vector and the direction to the player is ≤ `SightAngleDegrees / 2` (30°), AND (c) a raycast from The Hum to the player hits nothing solid first (true line-of-sight).
- `heardNoise(noiseRadius, position)`: true if distance from The Hum to `position` ≤ `BaseHearingRange + noiseRadius`.
- **Unit tests (TestEZ)** cover at least 5 geometric cases per function: directly in front/in cone, behind/out of cone, in cone but occluded by a wall, at exact range boundary, beyond hearing radius vs. within it after a loud (CinderBurst, radius 40) vs. quiet (MistVeil, radius 5) ability.

**Implementation Notes**
- Occlusion raycast must use a `RaycastParams` filter that ignores the players' own characters and The Hum's own model (avoid self-occlusion false negatives).

**Files:** `src/server/Services/EntityService.luau`, `src/server/Services/__tests__/EntityService.spec.luau`

---

### Ticket 12: Entity Movement & Pathfinding
**Depends on:** 10
**Description:** Implement physical movement for all four states.

**Acceptance Criteria**
- **Patrol:** moves between waypoint `Attachment`s (placed per room in Studio) via `PathfindingService:CreatePath`, looping continuously, at `WalkSpeed`.
- **Investigate:** during the 2.5s telegraph window, begins moving toward the heard/seen source's last known position at `WalkSpeed` (movement starts immediately; only the *state transition* to Hunt is delayed by the telegraph).
- **Hunt:** moves directly toward `huntTarget`'s current position at `WalkSpeed * HuntSpeedMultiplier` (16.8 studs/s).
- **Lost:** continues toward last-known position, then resumes Patrol waypoints once `HuntLoseSeconds` (6s) elapses with no re-acquisition.
- If `PathfindingService:CreatePath` fails (no valid path), The Hum holds position and re-attempts pathing every 1s rather than erroring.

**Implementation Notes**
- Waypoint `Attachment`s are placed during room-building tickets (26–31) — this ticket implements the *consumer* of those waypoints; an empty waypoint list is valid (Hum holds position) for early testing before rooms exist.

**Files:** `src/server/Services/EntityService.luau`

---

### Ticket 13: Ability-to-Entity Integration (Noise & Combat Effects)
**Depends on:** 7, 11, 12
**Description:** Wire AbilityService effect dispatch into EntityService — this is what makes the Fire/Water noise tradeoff (§2.3) real.

**Acceptance Criteria**
- **Every** successful `AbilityFire` (regardless of ability) calls `EntityService.NotifyNoise(noiseRadius, position, player)`.
- `NotifyNoise`, when called while The Hum is in `Patrol`, checks `heardNoise()` — if true, sets `huntTarget = player` and transitions to `Investigate`.
- `CinderBurst` resolution calls `EntityService.ApplyStagger(3)` **and** applies 15 damage to The Hum's tracked health (health pool added in this ticket: starts at an arbitrarily high value for V1 since The Hum is not killable — damage only triggers the stagger; the health number exists for forward-compatibility with a future killable variant).
- `Flare` resolution calls `EntityService.ApplyStagger(1.5)` and, if The Hum is within 50 studs, fires a client-only `EntityStateChanged`-adjacent "reveal ping" (does not change the actual state — purely a 2s visual reveal handled in Ticket 14).
- `MistVeil` resolution, if The Hum is in `Hunt` with `huntTarget` inside the 10-stud Mist radius, forces an immediate `Hunt → Lost` transition (early loss, not waiting for `HuntLoseSeconds`) for **all** players inside the radius simultaneously — i.e., Mist Veil breaks tracking for the whole group, not just the caster.
- `ApplyStagger(duration)` freezes EntityService's movement/state-transition logic for `duration` seconds (state remains visually whatever it was, just frozen).

**Implementation Notes**
- The "every ability use is heard" rule is the core balancing mechanism — do not special-case any ability to be silent beyond its configured `NoiseRadius`.

**Files:** `src/server/Services/AbilityService.luau`, `src/server/Services/EntityService.luau`

---

### Ticket 14: Client Entity Feedback Controller
**Depends on:** 10
**Description:** The primary "fear delivery" system — purely cosmetic, driven by `EntityStateChanged`.

**Acceptance Criteria**
- `Patrol`: ambient hum sound at baseline volume/pitch, normal `Lighting`.
- `Investigate`: over the 2.5s telegraph window, hum pitch rises linearly from baseline to peak, and room lights flicker (randomized brief `Lighting.Brightness` dips) — this window must be **visually/audibly distinguishable from Hunt** so players have a genuine warning, not just a shorter version of the scary state.
- `Hunt`: hum becomes a sustained harsh tone, `Lighting` shifts toward red tint (`ColorShift_Top`/`ColorShift_Bottom` or equivalent).
- `Lost`: fades back to `Patrol` ambience over ~1s (not instant — instant cuts read as a bug, not relief).
- Flare's "reveal ping" (from Ticket 13) renders a brief silhouette/outline of The Hum through walls for 2s when within 50 studs, independent of the state-driven effects above.

**Implementation Notes**
- This controller has no gameplay logic — it only listens to remotes and adjusts `Lighting`/`SoundService` properties. It can run entirely independent of game state and is safe to iterate on without touching server code.

**Files:** `src/client/Controllers/EntityFeedbackController.luau`

---

## EPIC 4 — Downed / Revive / Run Failure

### Ticket 15: DownedService — Core Logic
**Depends on:** 4, 5
**Description:** Implement the §2.6 failure-state core, independent of what *causes* a down (that's Ticket 16).

**Acceptance Criteria**
- `SetDowned(player)`: no-op if already downed; otherwise fires `PlayerDowned(player.UserId)`, starts a 45-second bleed-out `task.delay`, and disables the player's movement (`WalkSpeed = 0` or character state flag) and ability use (checked in `AbilityService`, Ticket 7 — add the downed check there).
- Bleed-out expiry: if not revived within 45s, the player remains downed indefinitely (no auto-respawn) until the run ends.
- `checkRunFailed()`: if **all** players in the run server are simultaneously downed, fires `RunEnded("Failed", chargeEarnedSoFar * 0.5)` and triggers the Hub-return teleport (stubbed in this ticket, implemented fully in Ticket 25's teleport infrastructure — for this ticket, a same-place "return to spawn" stub is acceptable).
- `checkRunFailed()` is called both when a player becomes downed and when a bleed-out timer expires (covers the case where the last standing player is the one who times out).

**Implementation Notes**
- `chargeEarnedSoFar` is a running total maintained by Ticket 32 — this ticket can reference a placeholder `0` until Ticket 32 lands, but the function signature must already accept the value.

**Files:** `src/server/Services/DownedService.luau`

---

### Ticket 16: Entity-to-Downed Contact Integration
**Depends on:** 12, 15
**Description:** Connect The Hum's `Hunt` state to actually downing players.

**Acceptance Criteria**
- While in `Hunt`, EntityService checks proximity to `huntTarget` every AI tick (10Hz); on contact (≤ ~4 studs), calls `DownedService.SetDowned(huntTarget)`.
- After downing a player, The Hum transitions to `Lost` (it doesn't "hold" a downed player — this allows the rest of the team to maneuver around it).
- A downed player cannot become a new `huntTarget` (EntityService perception functions skip players for whom `DownedService` reports `IsDowned = true`).

**Implementation Notes**
- This is intentionally a tight loop between two services — keep the proximity check in `EntityService` (it already runs the 10Hz tick) and call into `DownedService` rather than duplicating timer logic.

**Files:** `src/server/Services/EntityService.luau`, `src/server/Services/DownedService.luau`

---

### Ticket 17: Revive Interaction
**Depends on:** 15
**Description:** The core "tension under pressure" interaction per §2.6.

**Acceptance Criteria**
- A non-downed player holding `E` for 3 continuous seconds within 4 studs of a downed teammate triggers `DownedService.AttemptRevive(reviver, target)`.
- The hold is **interruptible**: if the reviver moves beyond 4 studs, releases `E`, or **becomes a `huntTarget` for The Hum** (i.e., Hum transitions toward `Hunt` with this player as target) at any point during the hold, the progress resets to 0 — it does not pause and resume.
- On success: `task.cancel` the bleed-out timer, fire `PlayerRevived(target.UserId)`, restore `WalkSpeed` and ability access for `target`.
- Revive progress is replicated to all nearby clients (for the UI in Ticket 18) via a lightweight progress remote, separate from the discrete `PlayerRevived` event.

**Implementation Notes**
- The interrupt-on-Hunt-target condition requires `EntityService` to expose a small read-only "who is currently a Hunt target" query — add this as a minimal getter, not a new remote (server-to-server internal call).

**Files:** `src/server/Services/DownedService.luau`, `src/server/Services/EntityService.luau` (getter), `src/client/Controllers/` (revive input)

---

### Ticket 18: Client Downed Overlay UI
**Depends on:** 15, 17
**Description:** Visual feedback for downed/reviving states.

**Acceptance Criteria**
- On `PlayerDowned` (local player): full-screen desaturation effect + visible bleed-out countdown (45s → 0).
- On `PlayerRevived` (local player): overlay clears immediately, countdown removed.
- For nearby players reviving a teammate: a progress ring/bar reflecting the replicated revive progress from Ticket 17, visible to **both** the reviver and the downed player.
- If a revive is interrupted (per Ticket 17 conditions), the progress UI resets to 0 with a brief visual "interrupted" flash rather than silently disappearing — players need to understand *why* it failed.

**Implementation Notes**
- Built with Vide (see Ticket 34 for Vide setup) — if Ticket 34 hasn't landed yet, this ticket may use a placeholder `ScreenGui`/`Frame` and be refactored to Vide components in Ticket 34. Note this explicitly in the PR description so it isn't missed.

**Files:** `src/client/UI/DownedOverlay.luau`

---

## EPIC 5 — Room & Puzzle System

### Ticket 19: Room/Puzzle Config Schema & Floor 1 Data
**Depends on:** 3
**Description:** Define the data structure for rooms/puzzle pools (using the types from Ticket 3) and populate it for all 5 Floor 1 rooms per §2.5.

**Acceptance Criteria**
- `Rooms.luau` exports `SubstationFloor: {RoomConfig}` with 5 entries, `PuzzlePool` populated per §2.5:
  - Room 1 (Entry Hall): pool = `[{Fire, Ignite, "Barricade"}]` OR `[{Water, Douse, "BurningDebris"}]` (one of two single-requirement options).
  - Room 2 (Flooded Maintenance): pool = `[{Water, Douse, "OverheatedValve"}]` OR `[{Fire, Ignite, "FrozenValveWheel"}]`.
  - Room 3 (Control Room): pool has one entry containing **both** `{Fire, Ignite, "SmokeVents"}` and `{Water, Douse, "OverheatingPanel"}` (combined puzzle — both required, order-independent, within a 30s window — window constant defined here as `Room3_SyncWindowSeconds = 30`).
  - Room 4 (The Catwalks): pool = `[{Fire, Ignite, "LampStation1"}, {Fire, Ignite, "LampStation2"}]` (2-of-3 lamp stations; a 3rd lamp station exists in geometry but is never in the active pool for V1 — reserved for post-V1 variation).
  - Room 5 (Extraction Point): pool has one entry containing `{Fire, ?, "NorthStation"}` and `{Water, ?, "SouthStation"}` with a `Room5_SyncWindowSeconds = 10` constant.
- A top-level `Floor1_PuzzlePoolSeedPolicy` constant/comment documents that pool selection is **per-run** (Ticket 21), not per-server-lifetime.

**Implementation Notes**
- This ticket only defines *data*. Geometry/stations referenced by `StationName` (e.g., `"Barricade"`, `"OverheatedValve"`) are built in Tickets 26–31 — `RoomService` (Ticket 20) must tolerate a missing `StationName` instance gracefully (warn, don't error) so this ticket can land before room geometry exists.

**Files:** `src/shared/Config/Rooms.luau`

---

### Ticket 20: RoomService — Node Effects & Puzzle State
**Depends on:** 4, 19
**Description:** Server-authoritative tracking of per-room puzzle completion.

**Acceptance Criteria**
- For the **active** room (the one the party is currently in — tracked via a simple "current room index" per run server, advanced on door-open), `RoomService` exposes `ApplyNodeEffect(stationName: string, effectType: "Ignite" | "Douse")`.
- `ApplyNodeEffect` checks the room's **rolled** `PuzzleRequirement`(s) (from Ticket 21) — if `stationName` + `effectType` matches a requirement, marks it satisfied.
- When **all** requirements for the active room's rolled puzzle are satisfied (for Room 3 and Room 5, both of two; for others, the single requirement), `RoomService` fires `PuzzleSolved(roomId, ...)`, opens the room's exit door (toggle `CanCollide`/play door animation), and advances the "current room index."
- Room 3's combined requirement additionally enforces the `Room3_SyncWindowSeconds = 30` window: if requirement A is satisfied at time T and requirement B is satisfied after `T + 30`, requirement A is reset to unsatisfied (players must redo it). Same pattern, with `Room5_SyncWindowSeconds = 10`, for Room 5.
- Missing `StationName` Instances (per Ticket 19's tolerance note) produce a single `warn()` at room-load time, not a runtime error on `ApplyNodeEffect`.

**Implementation Notes**
- "Current room index" state lives in `RoomService`, scoped per run server (this is a 1–4 player private server, so a single module-level variable is sufficient — no per-party keying needed within a run server).

**Files:** `src/server/Services/RoomService.luau`

---

### Ticket 21: Puzzle Pool Randomization at Run Start
**Depends on:** 20
**Description:** Roll one `PuzzleRequirement` (or set, for Rooms 3/5) per room at the start of each run.

**Acceptance Criteria**
- On run server initialization (before players are allowed to leave Room 1), `RoomService.RollPuzzles()` selects, for each room in `SubstationFloor`, one entry from its `PuzzlePool` via `math.random`.
- The rolled selection is **synced to all clients** in the run (via a remote or included in an existing one — extend `RunEnded`'s sibling or add a small `Remotes.PuzzlesRolled` event) so client-side UI hints (Ticket 35) are consistent with server-enforced requirements. **Do not** let clients infer requirements only from server rejection/acceptance — that would make hints impossible without trial-and-error.
- Re-running `RollPuzzles()` mid-run is not possible (idempotent — only the first call per server has effect); verified by a unit test calling it twice and asserting the second call is a no-op.
- For Rooms with only 1 entry in `PuzzlePool` (none in V1, but the system must support it for post-V1 content), `RollPuzzles` selects that single entry deterministically.

**Implementation Notes**
- `math.random` without an explicit seed is acceptable — runs don't need to be reproducible/shareable in V1.

**Files:** `src/server/Services/RoomService.luau`, `src/shared/Net/Remotes.luau` (add `PuzzlesRolled`)

---

### Ticket 22: Ability-to-Room Integration (Ignite/Douse Node Effects)
**Depends on:** 7, 20
**Description:** Connect `AbilityService`'s effect dispatch for `Ignite`/`Douse` to `RoomService.ApplyNodeEffect`.

**Acceptance Criteria**
- `AbilityFire("Ignite", target)` and `AbilityFire("Douse", target)`, after passing all AbilityService checks (Ticket 7), resolve `target` to the nearest interactable node `Instance` within `Range` (6 studs) tagged with a matching `StationName`.
- If a matching node is found, call `RoomService.ApplyNodeEffect(stationName, "Ignite" | "Douse")`.
- If **no** matching node is within range, the ability still resolves successfully (cooldown consumed, `AbilityResolved` fires, `NotifyNoise` still called per Ticket 13) — Ignite/Douse used "into nothing" is a valid (if wasted) action, not an error.
- Both Ignite and Douse remain subject to the Ticket 13 noise-notification, even when they affect a room node rather than The Hum directly — this is what makes "loud puzzle-solving" a real risk per §2.3.

**Implementation Notes**
- Node Instances are tagged via `CollectionService` with tag `"Node"` and an attribute `StationName` for fast lookup — define this convention here for use by room-building tickets (26–31).

**Files:** `src/server/Services/AbilityService.luau`, `src/server/Services/RoomService.luau`

---

## EPIC 6 — Hub & Party Flow

### Ticket 23: Hub Place Layout & Social Space
**Depends on:** 2
**Description:** Build the minimal Hub environment needed to host party formation.

**Acceptance Criteria**
- Hub place loads with a spawn area, a "Loadout Terminal" interactable (geometry placeholder acceptable — final art in Ticket 38), and a "Looking for Group" board (geometry placeholder).
- Players can walk, see each other, and the place is stable at `MaxPlayers = 50` with no errors in the output.

**Implementation Notes**
- This ticket is intentionally lightweight — its only purpose is to provide a place for Ticket 24's UI to attach to. Do not invest art time here yet.

**Files:** Hub place geometry (Studio), `src/shared/Config/Places.luau` (confirm Hub ID wired)

---

### Ticket 24: Loadout Terminal UI & Element Coverage Validation
**Depends on:** 5, 23
**Description:** The interaction that forms a party and enforces §2.2's coverage rules before allowing "Ready."

**Acceptance Criteria**
- UI lets each player in a party select an element loadout: 1 element if party size ≥ 2 (per-player), or both Fire+Water if party size = 1.
- **Server-side** validation (not just UI graying-out) before accepting "Ready": for party size ≥ 2, at least one player must have Fire and at least one must have Water selected; for party size = 1, the single player is automatically assigned both.
- A player can only select elements present in their `ProfileStore.UnlockedElements` (Ticket 5) — for V1 this is always Fire+Water (both unlocked by default), but the check must exist for post-V1 (Tickets 46–47).
- If coverage is not met, "Ready" remains disabled and the UI explains why (e.g., "Someone needs Water").

**Implementation Notes**
- "Party" at this stage is a client-side grouping (players standing near the terminal together / using an invite flow) — formal reserved-server assignment happens in Ticket 25. This ticket's server validation should operate on whatever player-list the party-formation UI produces, via a `ValidateLoadout(players, loadouts): boolean` server function callable from Ticket 25.

**Files:** `src/client/UI/LoadoutTerminal.luau`, `src/server/Services/PartyService.luau` (validation function only — teleport logic in Ticket 25)

---

### Ticket 25: PartyService — Reserved Server Teleportation
**Depends on:** 2, 24
**Description:** Move a validated party from the Hub into a private Substation run server.

**Acceptance Criteria**
- On "Ready" (post-validation from Ticket 24), `PartyService` calls `TeleportService:ReserveServer(SubstationPlaceId)`, obtaining an access code.
- All party members are teleported together via `TeleportService:TeleportToPrivateServer(SubstationPlaceId, accessCode, partyPlayerList)`.
- If any party member fails to load into the reserved server within a timeout (e.g., 30s), the run is aborted for the whole party and everyone returns to Hub (no partial-party runs).
- The reserved server, on load, immediately calls `RoomService.RollPuzzles()` (Ticket 21) before any player can act.

**Implementation Notes**
- This completes the §2.6 "return to Hub" loop referenced as a stub in Ticket 15 — `RunEnded` (Failed or Extracted) now triggers a real `TeleportService:Teleport` back to the Hub place ID from `Places.luau`.

**Files:** `src/server/Services/PartyService.luau`, `src/server/Services/DownedService.luau` (wire real teleport), `src/server/Services/RoomService.luau` (wire RollPuzzles on server start)

---

## EPIC 7 — Floor 1 Content: The Substation

### Ticket 26: Room 1 — Entry Hall
**Depends on:** 19–22, 9–14
**Description:** Build the first room's geometry and wire it to the puzzle/entity systems.

**Acceptance Criteria**
- Geometry (gray-box acceptable) for Entry Hall, with both possible nodes built and tagged: `"Barricade"` (Fire/Ignite) and `"BurningDebris"` (Water/Douse) — only the rolled one is ever "live" per Ticket 21, but both must exist as Instances so either roll works.
- At least 2 Hum patrol waypoint `Attachment`s placed in this room (low-density per §2.5's "Patrol only, low frequency").
- Exit door instance wired to `RoomService`'s door-open behavior (Ticket 20).
- Manual test: entering with either puzzle roll, the correct node responds to its ability and opens the door; the incorrect ability on a node does nothing (no crash, no false-positive solve).

**Files:** Room 1 geometry (Studio), tagged node Instances

---

### Ticket 27: Room 2 — Flooded Maintenance
**Depends on:** 19–22, 9–14
**Description:** Build Room 2 per §2.5.

**Acceptance Criteria**
- Geometry includes both possible nodes: `"OverheatedValve"` (Water/Douse) and `"FrozenValveWheel"` (Fire/Ignite), tagged per Ticket 22's convention.
- Hum patrol waypoints placed at higher density than Room 1 — this is the first room where an `Investigate` transition is plausible during normal play (per §2.5).
- Exit door wired as in Ticket 26.

**Files:** Room 2 geometry (Studio), tagged node Instances

---

### Ticket 28: Milestone Playtest 1 — Core Loop Validation (Rooms 1–2)
**Depends on:** 26, 27
**Description:** First real playtest gate, per the original plan's Phase 2 milestone. **This ticket is a checkpoint, not a code deliverable** — its output is a go/no-go decision and a list of follow-up tickets if "no-go."

**Acceptance Criteria**
- 2–4 testers (who have not seen the puzzle pools) play Rooms 1–2 back-to-back, no art, multiple times (different puzzle rolls).
- Observed outcomes recorded against early indicators of the §9 targets (full targets apply to the complete floor in Ticket 40, but directional signals should already be visible here):
  - Does the `Investigate` telegraph give players a genuine, usable warning (i.e., do they ever successfully react to it)?
  - Does at least one tester experience a `Downed` state across multiple sessions (Hum is not toothless)?
  - Does the puzzle-pool randomization produce a noticeably different experience between runs (not "felt the same")?
- **Go:** proceed to Ticket 29. **No-go:** file follow-up tickets against the relevant Epic 3/4/5 tickets with specific tuning changes (e.g., adjust `InvestigateTelegraphSeconds` or a `NoiseRadius` value in Ticket 6/9's config files) before proceeding.

**Files:** none (process ticket) — any tuning changes land as edits to `Elements.luau`/`Entities.luau` (Tickets 6/9)

---

### Ticket 29: Room 3 — Control Room
**Depends on:** 19–22, 28
**Description:** Build the first room requiring **both** elements, per §2.5.

**Acceptance Criteria**
- Geometry includes `"SmokeVents"` (Fire/Ignite) and `"OverheatingPanel"` (Water/Douse), both **always live** (Room 3's pool has one entry containing both requirements — Ticket 19).
- The `Room3_SyncWindowSeconds = 30` behavior (Ticket 20) is verified: solving both within 30s opens the door; solving one, waiting >30s, then solving the other resets the first.
- Higher-density Hum patrol waypoints per §2.5 ("higher patrol density").

**Files:** Room 3 geometry (Studio), tagged node Instances

---

### Ticket 30: Room 4 — The Catwalks
**Depends on:** 19–22, 28
**Description:** Build the avoidance-focused, low-light room per §2.5.

**Acceptance Criteria**
- Geometry is narrow/exposed per the "Catwalks" theme; ambient `Lighting` for this room is darker than Rooms 1–3 by default (before any Ignite).
- `"LampStation1"` and `"LampStation2"` nodes built and tagged (Fire/Ignite); a 3rd lamp station exists in geometry but is **not** tagged/active for V1 (reserved per Ticket 19's note).
- Igniting both active lamp stations is required to open the exit (RoomService treats this as a 2-of-2 requirement from the single rolled pool entry).
- Hum aggression in this room should make `Hunt` transitions plausible if players are noisy (verified qualitatively during Ticket 40).

**Files:** Room 4 geometry (Studio), tagged node Instances

---

### Ticket 31: Room 5 — Extraction Point & Run Completion
**Depends on:** 19–22, 28
**Description:** Build the finale room and wire the successful-run path.

**Acceptance Criteria**
- Geometry includes `"NorthStation"` (Fire) and `"SouthStation"` (Water), positioned far enough apart that a single player cannot easily activate both within the sync window (encouraging the "split the team" design intent).
- `Room5_SyncWindowSeconds = 10` enforced per Ticket 20.
- On both stations activated within the window: extraction door opens; **on all surviving players reaching the extraction zone** (a trigger volume), `RunEnded("Extracted", chargeEarned)` fires (`chargeEarned` wired in Ticket 32 — placeholder `0` acceptable until then) and `PartyService` teleports the party back to the Hub.
- Highest Hum aggression of any room per §2.5 — patrol waypoint density and/or a slightly reduced `BaseHearingRange` override **for this room only** (implemented as a per-room override read by `EntityService`, not a global config change) may be used to achieve this; document whatever value is chosen.

**Files:** Room 5 geometry (Studio), tagged node Instances, `src/server/Services/EntityService.luau` (per-room override hook), `src/server/Services/RoomService.luau` (RunEnded wiring)

---

## EPIC 8 — Progression & Currency

### Ticket 32: Charge Currency — Earning Logic
**Depends on:** 31, 4
**Description:** Implement the §2.7 reward formula.

**Acceptance Criteria**
- A per-run-server `chargeEarned` accumulator increments by a fixed **base reward per room cleared** (constant, e.g., defined alongside `Rooms.luau`) on each `PuzzleSolved`.
- On `RunEnded("Extracted", ...)`:
  - Add an **extraction bonus** (flat constant).
  - Add a **speed bonus** if total run time < a configurable threshold (constant `Run.SpeedBonusThresholdSeconds`, suggested default consistent with the "12–18 min target run length" — e.g., bonus if under 12 minutes).
  - Add a **no-downs bonus** (multiplier, constant `Run.NoDownsBonusMultiplier`) if `DownedService` reports zero downs occurred this run.
- On `RunEnded("Failed", ...)`, `chargeEarned` is the accumulator value **× 0.5** (per §2.6), with no extraction/speed/no-downs bonuses.
- Final `chargeEarned` value is passed to Ticket 33 for persistence.

**Implementation Notes**
- All constants (`base reward`, extraction bonus, thresholds, multiplier) belong in a single `src/shared/Config/Economy.luau` file — this is the file Ticket 45 exposes via Experiments API Configs.

**Files:** `src/shared/Config/Economy.luau`, `src/server/Services/RoomService.luau` (room-clear increments), `src/server/Services/DownedService.luau` / `PartyService.luau` (final calculation on RunEnded)

---

### Ticket 33: ProfileStore Persistence — Charge, Unlocks & Stats
**Depends on:** 5, 32
**Description:** Persist run results back into each player's profile.

**Acceptance Criteria**
- On `RunEnded` (either result), for each player: `Profile.Data.Charge += chargeEarned` (split evenly if computed per-party — define and document the split policy, e.g., equal share regardless of individual contribution, consistent with co-op design intent).
- `Stats.RunsCompleted += 1` on `"Extracted"` only (a Failed run does not count as completed).
- `Stats.BestTimeSeconds` updated if this run's time is lower than the existing value (or if `nil`).
- `Stats.TotalDowns` incremented by `DownedService`'s per-player down count for this run.
- All writes happen via the existing `DataService.Get(player)` profile reference (Ticket 5) — no new DataStore calls introduced.

**Files:** `src/server/Services/DataService.luau`, `src/server/Services/PartyService.luau` (call site on RunEnded)

---

## EPIC 9 — UI / HUD

### Ticket 34: Vide Setup & HUD Cooldown Display
**Depends on:** 7, 8
**Description:** Establish Vide as the UI framework and build the primary in-run HUD.

**Acceptance Criteria**
- Vide added as a dependency (Wally or vendored per its repo); a minimal "hello world" Vide component renders correctly in both Hub and Substation places.
- HUD displays, for each ability the local player has equipped: an icon + radial/numeric cooldown derived from the same prediction logic established in Ticket 8 (optimistic countdown, corrected on rejection).
- HUD is visually present but minimal — no final art (final visual pass is part of Ticket 37/38's scope for VFX, not HUD chrome).
- If Ticket 18 (Downed Overlay) was built with a placeholder `ScreenGui`, refactor it to a Vide component in this ticket.

**Files:** `src/client/UI/Hud.luau`, refactor `src/client/UI/DownedOverlay.luau`

---

### Ticket 35: Objective Tracker UI
**Depends on:** 20, 34
**Description:** Show players what the current room requires, using the synced puzzle-roll data from Ticket 21.

**Acceptance Criteria**
- HUD element shows the current room's requirement(s) in player-readable form (e.g., "Ignite the Barricade" / "Douse the Overheating Panel **and** Ignite the Smoke Vents").
- For Rooms 3 and 5 (dual requirements with sync windows), the tracker shows **both** requirements and visually indicates when one has been satisfied and the sync-window countdown is active.
- Updates live as `PuzzleSolved` events arrive — no manual refresh needed.

**Files:** `src/client/UI/Hud.luau` (or a dedicated `ObjectiveTracker.luau` Vide component)

---

## EPIC 10 — Audio & Visual Polish

### Ticket 36: Ambient Audio System (Hum States)
**Depends on:** 14
**Description:** Replace placeholder audio cues from Ticket 14 with final sound assets.

**Acceptance Criteria**
- Distinct `Sound` assets (or layered `SoundGroup`s) for: Patrol ambience, Investigate rising-pitch loop, Hunt harsh tone, Lost fade-out — matching the timing/behavior already implemented in Ticket 14.
- Audio is **spatial** (3D `Sound` parented to The Hum's model, or `SoundService` regional zones) so direction/distance to The Hum is perceivable — this is a core fairness mechanic (players can sometimes tell where The Hum is by sound alone, mitigating "unfair" surprise contact).
- Jump-scare stinger plays once on `Hunt → contact (Downed)` transition — sourced from `EntityService`'s contact event (Ticket 16), routed through a client remote or inferred from `PlayerDowned`.

**Files:** Audio assets (uploaded), `src/client/Controllers/EntityFeedbackController.luau` (asset wiring)

---

### Ticket 37: Ability VFX/SFX
**Depends on:** 8
**Description:** Final visual/audio feedback for each ability, replacing Ticket 8's placeholders.

**Acceptance Criteria**
- Each of `Ignite`, `CinderBurst`, `Flare`, `Douse`, `MistVeil` has distinct particle/beam VFX and a sound effect, triggered on `AbilityResolved` (so all players in the run see/hear the effect, not just the caster).
- VFX durations are visually consistent with their `Effects` durations from `Elements.luau` (e.g., `MistVeil`'s fog visual persists for the full 4s `Conceal` duration).
- `Flare`'s VFX is bright/loud enough to read as "high noise" (`NoiseRadius = 60`) relative to `MistVeil`'s subtlety (`NoiseRadius = 5`) — this is a usability cue, not just decoration: players should be able to *feel* the noise tradeoff from §2.3 through audio/visual intensity alone.

**Files:** VFX/SFX assets (uploaded), `src/client/Controllers/AbilityController.luau` (trigger wiring)

---

### Ticket 38: Environment Dressing via AI Generation (Cube 3D / 4D)
**Depends on:** 26–31
**Description:** Fill out Rooms 1–5 with environmental props using Roblox's Cube 3D/4D generation tools.

**Acceptance Criteria**
- Game Settings → Security: "Allow EditableImage/EditableMesh APIs" and "Allow Mesh & Image APIs" enabled (prerequisite, verified first).
- Each room has at least 5–8 generated props (crates, pipes, debris, furniture appropriate to an industrial substation theme) placed via Cube 3D/4D prompts.
- Generated assets are **uploaded as persistent Roblox assets** (not left in-memory-only) — verified by reloading Studio and confirming props remain.
- No generated prop occludes a tagged node Instance (Ticket 22) or a Hum waypoint `Attachment` (Ticket 12) — a pass is done to check this after placement.

**Implementation Notes**
- Per the original research: Cube 3D is rate-limited (~5 generations/min/experience) and weak at in-model text/logos — avoid prompting for anything with readable signage; use `SurfaceGui`/decals for any in-game text instead.

**Files:** Room geometry (Studio), generated mesh assets

---

### Ticket 39: Hero Asset — The Hum Character Model
**Depends on:** 10
**Description:** Replace The Hum's placeholder body (used since Ticket 10) with a custom-modeled character.

**Acceptance Criteria**
- Model built in Blender, UV-unwrapped, exported as `.fbx`/`.gltf`, imported via Roblox's 3D Importer.
- Model rig is compatible with whatever animation approach `EntityService`'s movement (Ticket 12) drives (walk/chase animation states for Patrol/Investigate/Hunt at minimum).
- Model silhouette is legible at the Flare-reveal distance (50 studs, per Ticket 13) and in the red-tinted `Hunt` lighting (Ticket 14) — verified visually, not just in isolation under default lighting.
- Textured via Substance 3D if available; otherwise Studio materials — either is acceptable, but PBR maps (diffuse/normal/roughness) should be present if Substance is used.

**Files:** Model assets (uploaded), `src/server/Services/EntityService.luau` (model reference swap)

---

## EPIC 11 — QA & Security

### Ticket 40: Full V1 Playtest & Balance Pass
**Depends on:** 1–39 (full V1 functional + polish)
**Description:** The complete-floor equivalent of Ticket 28's milestone, evaluated against the full §9 targets.

**Acceptance Criteria** — across multiple full-floor runs with fresh testers:
- Run completion rate (new groups): **60–70%**. If outside this band, identify and file follow-up tuning tickets (likely targets: `Entities.luau` detection ranges, `Elements.luau` noise radii, or `Room5_SyncWindowSeconds`).
- Average run time: **12–18 minutes**.
- Average downs per run: **1–3**.
- % of Hunt-state transitions resulting in a Down: **40–60%**.
- Players react to the `Investigate` telegraph (per Ticket 14) within its window at least 50% of the time (observed, not self-reported).
- For each metric outside its band, the specific config value(s) to adjust and the direction of adjustment are documented before closing this ticket.

**Files:** none directly — produces follow-up tuning tickets against Tickets 6/9/19/32 config files

---

### Ticket 41: RemoteEvent Security Audit
**Depends on:** 7, 13, 17, 20, 22
**Description:** Verify the server-authority principles from §3.5 hold across the full implementation before launch.

**Acceptance Criteria** — checklist, each item verified by code review and/or a targeted exploit test using a modified client:
- All 7 remotes from Ticket 4 are fired/consumed **only** via `Remotes.luau` (no stray `Instance.new("RemoteEvent")`).
- `AbilityService` (Ticket 7) rejects: unlocked-element bypass, cooldown bypass (rapid-fire test), out-of-range targets, >1 request/0.1s spam — each tested with a client that sends malformed/forged requests.
- `RoomService.ApplyNodeEffect` (Ticket 20/22) cannot be called with an arbitrary `stationName` to solve puzzles remotely — confirm range/proximity is enforced server-side, not just client-side targeting.
- `DownedService.AttemptRevive` (Ticket 17) cannot be triggered without the 4-stud proximity and 3s hold being independently verifiable server-side (not solely a client-reported "I held it").
- `EntityService` state and position are never sent to clients at higher fidelity than `EntityStateChanged` requires (no raw per-tick position stream that would let a client predict/avoid The Hum via packet sniffing beyond intended visual cues).

**Files:** none directly — produces remediation tickets if any check fails

---

## EPIC 12 — Monetization & Publishing

### Ticket 42: Group Setup & Revenue Configuration
**Depends on:** none (can run in parallel with any prior ticket)
**Description:** Publish *Static* under a Group per §7.

**Acceptance Criteria**
- A Roblox Group is created; both places (Hub, Substation) are owned by the Group.
- Group roles configured per the original playbook's recommendations (Owner + at minimum a "Developer" role with Edit access).
- Payouts page accessible (may require meeting Roblox's group-age/funds prerequisites — document current status if not yet unlocked).

**Files:** none (Roblox dashboard configuration)

---

### Ticket 43: Game Passes & Developer Products
**Depends on:** 32, 33, 42
**Description:** Implement the §7 monetization items, explicitly **excluding** anything that affects in-run survival.

**Acceptance Criteria**
- **Game Pass — Cosmetic Aura Pack:** purchasing grants a cosmetic-only visual variant on ability VFX (Ticket 37 hooks check `UserOwnsGamePassAsync` and swap particle color/style only — no stat change).
- **Game Pass — Hub Decoration Pack:** purchasing unlocks cosmetic Hub decorations only.
- **Developer Product — Charge Boost:** grants a fixed `Charge` amount to `ProfileStore` on successful purchase receipt (via `ProcessReceipt`), with standard idempotent-receipt handling (Roblox may retry `ProcessReceipt` — ensure double-grants can't occur).
- **Explicitly not implemented:** any "Revive Token" or other in-run-survival purchase — this is a deliberate exclusion per §7's design rationale, not an oversight. If anyone proposes one later, it should be a new design discussion, not assumed in-scope here.

**Files:** `src/server/Services/MonetizationService.luau` (new), `src/server/Services/DataService.luau` (Charge Boost grant)

---

### Ticket 44: Store Page, Content Maturity Rating & Soft Launch
**Depends on:** 38, 39, 40, 41, 43
**Description:** Publish-readiness — the gate before any real traffic.

**Acceptance Criteria**
- Store page configured: icon, thumbnails/short trailer (can leverage Ticket 37's VFX and Ticket 39's Hum model for trailer footage), description.
- Content maturity rating set honestly — **Moderate**, given horror tone/entity-contact content per the original playbook's guidance.
- Soft launch: place set to a small-audience access mode (e.g., shared with a small Discord/friend group) rather than fully public.
- Basic Acquisition Analytics dashboard reviewed at least once post-soft-launch to confirm data is flowing (even if volumes are tiny) — this validates the pipeline before scaling traffic.

**Files:** none directly (Roblox dashboard configuration + uploaded media)

---

### Ticket 45: Experiments API Config Wiring
**Depends on:** 6, 9, 19 (Room5/Room3 sync constants), 32 (Economy constants)
**Description:** Expose the tunable values identified throughout this guide as Creator Hub Configs, per §10.

**Acceptance Criteria**
- The following values are readable from Configs at runtime (with the hardcoded values from their respective tickets as defaults/fallbacks if Configs are unset):
  - `Hum.BaseHearingRange`, `Hum.InvestigateTelegraphSeconds`, `Hum.HuntLoseSeconds` (Ticket 9)
  - Per-ability `Cooldown` and `NoiseRadius` for all 5 V1 abilities (Ticket 6)
  - `Run.SpeedBonusThresholdSeconds`, `Run.NoDownsBonusMultiplier` (Ticket 32)
  - `Room5_SyncWindowSeconds`, `Room3_SyncWindowSeconds` (Ticket 19)
- A 50/50 Control/Variant test can be configured for at least one value (e.g., `Hum.InvestigateTelegraphSeconds`) and produces visibly different server behavior between the two groups — verified with a manual two-server test before relying on it for real experiments.

**Files:** `src/server/Services/ConfigService.luau` (new — Configs fetch + fallback layer), updates to read sites in `EntityService.luau`, `AbilityService.luau`, `RoomService.luau`

---

# Phase 2 — Post-V1 Roadmap

The following tickets extend *Static* per §6 of the implementation plan. They assume Tickets 1–45 are complete and live.

### Ticket 46: Earth Element Implementation
**Depends on:** 6, 7, 24, 33
**Description:** Add the third element per §6.

**Acceptance Criteria**
- New `AbilityConfig` entries added to `Elements.luau` for Earth's puzzle ability ("create temporary platform/cover") and combat ability ("stationary shield blocking Hum's sight cone for 6s, high cooldown, zero `NoiseRadius`").
- `ProfileStore.UnlockedElements.Earth` defaults to `false`; an unlock path (e.g., `Stats.RunsCompleted` threshold, or a Charge cost) is implemented and persisted.
- Ticket 24's coverage validation logic generalizes correctly to 3 elements for parties of various sizes (re-verify the 1/2/3/4-player rules from §2.2 still produce valid coverage with 3 elements in the pool — document the updated per-size loadout rules).
- AbilityService (Ticket 7) and EntityService (Ticket 13's `NotifyNoise`/`ApplyStagger` pattern) require **no structural changes** — Earth's abilities flow through the existing data-driven dispatch, confirming the architecture's extensibility.

**Files:** `src/shared/Config/Elements.luau`, `src/server/Services/DataService.luau` (unlock logic), `src/client/UI/LoadoutTerminal.luau`

---

### Ticket 47: Lightning Element Implementation
**Depends on:** 46
**Description:** Add the fourth element per §6, including its unique multi-player mechanic.

**Acceptance Criteria**
- New `AbilityConfig` entries for Lightning's puzzle ability ("power dead electrical panels") and its combat ability: a **shared-cooldown chain effect linking up to 3 players**, very high `NoiseRadius`.
- The shared-cooldown mechanic requires a new pattern beyond per-player cooldown tables (Ticket 7) — implement as a party-scoped cooldown keyed by ability ID, consumed by whichever player activates it, blocking all linked players until it resets. Document this as an intentional, narrow extension to AbilityService.
- Coverage validation (Ticket 24, extended in Ticket 46) updated for 4-element pools.
- `ProfileStore.UnlockedElements.Lightning` unlock path implemented, consistent with Ticket 46's pattern.

**Files:** `src/shared/Config/Elements.luau`, `src/server/Services/AbilityService.luau` (party-scoped cooldown), `src/client/UI/LoadoutTerminal.luau`

---

### Ticket 48: Floor 2 Entity — The Warden
**Depends on:** 9–14, 47
**Description:** Add a second entity type with a genuinely new behavior (charge attack), not a stat reskin of The Hum.

**Acceptance Criteria**
- `Entities.luau`'s `{[string]: EntityConfig}` table (Ticket 9's forward-compatible structure) gains a `TheWarden` entry with its own tuning values.
- `EntityService`'s state machine gains a new state or sub-state for the **charge attack**: long telegraphed wind-up (visually/audibly distinct from the Investigate telegraph — must not be confused with it), followed by a short, very-fast, short-range dash that downs any player in its path on contact.
- The charge attack is dodgeable by any player who reacts during the wind-up (movement out of the dash's line) — verified in playtesting, consistent with the "telegraphed, not unfair" pillar from §1.
- The Warden otherwise reuses Patrol/Investigate/Hunt/Lost from the existing state machine — only the charge attack is new logic.

**Files:** `src/shared/Config/Entities.luau`, `src/server/Services/EntityService.luau` (charge-attack state), new Warden model assets

---

### Ticket 49: Floor 2 — Branching Room Layout
**Depends on:** 19–22, 46–48
**Description:** Introduce route choice per §6 — 2 valid paths through 6 rooms.

**Acceptance Criteria**
- `Rooms.luau` (or a new `Floor2Rooms.luau` following the same `RoomConfig`/`PuzzlePool` pattern from Ticket 19) defines 6 rooms with a branch point after Room 1: Path A (rooms 2A–4A) and Path B (rooms 2B–4B), reconverging at Room 5/6.
- `RoomService`'s "current room index" model (Ticket 20) is generalized to support branching (e.g., a room graph rather than a flat array) — this is the one structural change to `RoomService` in Phase 2; document it clearly as such.
- Both paths use Earth/Lightning (Tickets 46–47) puzzles in addition to Fire/Water, and both paths encounter The Warden (Ticket 48) at least once.
- Puzzle pool randomization (Ticket 21) operates per-room as before, independent of which path was chosen.

**Files:** `src/shared/Config/Rooms.luau` (or new `Floor2Rooms.luau`), `src/server/Services/RoomService.luau` (room-graph generalization), Floor 2 geometry (Studio)

---

# Suggested Sprint Grouping

Mapping back to the 5 phases in STATIC_Implementation_Plan.md §8, for teams that plan in sprints rather than strict ticket-by-ticket sequence:

| Sprint | Tickets | Maps to plan phase |
|---|---|---|
| 1 | 1–5 | Phase 1 — Foundations |
| 2 | 6–14 | Phase 2 — Core Loop (Abilities + Entity AI) |
| 3 | 15–22 | Phase 2 — Core Loop (Downed/Revive + Rooms system) |
| 4 | 23–28 | Phase 2/3 — Hub/Party + Rooms 1–2 + Milestone Playtest |
| 5 | 29–35 | Phase 3 — Rooms 3–5 + Progression + Core UI |
| 6 | 36–39 | Phase 4 — Polish (Audio/VFX/Art) |
| 7 | 40–45 | Phase 5 — QA, Security, Monetization, Launch |
| 8+ | 46–49 | Phase 2 (post-V1 roadmap) |

Tickets within a sprint that share an Epic can generally be parallelized across team members once their listed dependencies are satisfied; cross-Epic dependencies (e.g., Ticket 13 depending on both Epic 2 and Epic 3 work) are the main serialization points to watch when assigning work.
