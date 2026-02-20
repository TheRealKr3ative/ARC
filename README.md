--[[
    =====================================================================================
                                    Welcome to ARC!
    =====================================================================================

	╔══════════════════════════════════════════════════════╗
	║                    ARC  v2.1                         ║
	║         Archive — ProfileStore Wrapper               ║
	║                                                      ║
	║  Handles:                                            ║
	║    • Session lifecycle (start, end, cancel)          ║
	║    • Live & offline read / write, consistent codec   ║
	║    • Batch reads and multi-key writes                ║
	║    • Atomic transform-based updates                  ║
	║    • Schema wipe (reset a player to defaults)        ║
	║    • Global hooks: OnProfileLoaded, OnProfileEnded   ║
	║    • Per-player session-end listeners                ║
	║    • Active session introspection                    ║
	║    • Strict type guards on every public call         ║
	╚══════════════════════════════════════════════════════╝

    Author  : @Kr3ativeKrayon / @Slunderman2468
    Version : 2.1.0
    Updated : 2/18/2026
    
    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    DISCLAIMER!!!
    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    
    ARC Instance Support has gone into beta if you have any problems please let me know
    -> Discord : TheRealKr3ative
    
    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    TABLE OF CONTENTS
    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
      1.  What is ARC?
      2.  Architecture Overview
      3.  Setup
      4.  Quick Start
      5.  API Reference
            5.1  Configuration
            5.2  Global Hooks
            5.3  Session Management
            5.4  Reading Data
            5.5  Writing Data
            5.6  Schema Management
            5.7  Encode / Decode
            5.8  Debug
      6.  Error Reference
      7.  Security & Safety
      8.  Changelog


    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    1. WHAT IS ARC?
    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

    ARC is a high-performance, session-locked data persistence layer built on
    top of ProfileStore. It manages the "Archive" of every player's state,
    using JAR for binary serialisation and Utility for Base64 transport so
    that any Roblox data type — including nested tables, Vector3s, CFrames,
    and Enums — can be stored in a standard Datastore string safely.

    ARC adds a clean, type-safe API on top of ProfileStore so the rest of
    your codebase never has to think about session locking, offline access,
    encode/decode cycles, or schema migrations. All of that is handled here.


    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    2. ARCHITECTURE OVERVIEW
    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

    ┌──────────────┐       read / write       ┌─────────────┐
    │  Your Code   │ ─────────────────────── ▶│     ARC     │
    └──────────────┘                          └──────┬──────┘
                                                     │  encode / decode
                                               ┌─────▼──────┐
                                               │   Utility   │  (Base64 I/O)
                                               └─────┬───────┘
                                                     │  pack / unpack
                                               ┌─────▼──────┐
                                               │     JAR     │  (Binary buffer)
                                               └─────┬───────┘
                                                     │  session lock / persist
                                               ┌─────▼──────┐
                                               │ ProfileStore│  (Roblox Datastore)
                                               └────────────┘

    Data flow on WRITE:
        value → JAR.write → JAR.carbonate → Base64 → Datastore string

    Data flow on READ:
        Datastore string → Base64 decode → JAR.rehydrate → JAR.read → value

    This means values are always stored as compact binary strings in the
    Datastore. You should never manually inspect or edit these strings.


    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    3. SETUP
    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

    1. Place the ARC folder in ServerStorage (server-only — never replicate).

    2. Configure _ARC/Default.lua — this is your data schema:

           return {
               Coins   = 0,
               Level   = 1,
               XP      = 0,
               Banned  = false,
               Inventory = {},
           }

       Every key you want to persist must exist here. ARC uses this table
       to type-check writes, fill in missing keys on load (Reconcile), and
       reset data via WipeArchiveAsync.

    3. Configure _ARC/Config.lua:

           return {
               LiveProfileName  = "PlayerData_v1",
               StudioProfileName = "PlayerData_Studio",
               ARCPrefix        = "ARC_",
               Errors = {
                   InvalidProfile = "Your data could not be loaded. Please rejoin.",
               },
           }

       Increment LiveProfileName whenever you make a breaking schema change
       to start all players on a fresh default without touching existing data.

    4. Hook ARC into your player lifecycle:

           local Players = game:GetService("Players")
           local ARC     = require(ServerStorage.ARC)

           Players.PlayerAdded:Connect(function(player)
               ARC.StartPlayerSessionAsync(player)
           end)

           Players.PlayerRemoving:Connect(function(player)
               ARC.EndPlayerSessionAsync(player)
           end)

           -- Handle players who joined before the script loaded.
           for _, player in ipairs(Players:GetPlayers()) do
               task.spawn(ARC.StartPlayerSessionAsync, player)
           end


    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    4. QUICK START
    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

    local ARC = require(ServerStorage.ARC)


    -- ── Read a single value ───────────────────────────────
    local coins = ARC.FetchArchiveValueAsync(player, "Coins")
    print(coins)   -- always a number; defaults to Default.Coins if missing


    -- ── Write a single value ──────────────────────────────
    ARC.ModifyArchiveValueAsync(player, "Coins", 500)


    -- ── Atomic increment (read → transform → write) ───────
    ARC.UpdateArchiveValueAsync(player, "Coins", function(current)
        return current + 50
    end)


    -- ── Read multiple keys in one call ────────────────────
    local stats = ARC.FetchArchiveValuesAsync(player, { "Coins", "Level", "XP" })
    print(stats.Coins, stats.Level, stats.XP)


    -- ── Write multiple keys in one call ───────────────────
    ARC.ModifyArchiveValuesAsync(player, {
        Coins = 1000,
        Level = 5,
        XP    = 250,
    })


    -- ── Work with offline players ─────────────────────────
    -- All methods accept a UserId or string in place of a Player instance.
    -- ARC opens a temporary session, applies the change, then closes it.
    ARC.ModifyArchiveValueAsync(123456789, "Banned", true)

    local offlineCoins = ARC.FetchArchiveValueAsync("123456789", "Coins")


    -- ── Listen for profile events ─────────────────────────
    ARC.OnProfileLoaded(function(player, data)
        print(player.Name .. " loaded. Coins: " .. tostring(data.Coins))
    end)

    ARC.OnProfileEnded(function(player)
        print(player.Name .. " session ended.")
    end)


    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    5. API REFERENCE
    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

    The `id` parameter accepted by data methods can be any of:
        Player instance  — live player on this server
        number           — UserId  (live or offline)
        string           — UserId as a string (live or offline)

    For offline players, ARC opens a temporary ProfileStore session,
    applies the operation, then closes it automatically. Never pass an
    offline id to session management functions (Start/End).


    ────────────────────────────────────────────────────
    5.1  CONFIGURATION
    ────────────────────────────────────────────────────

    ARC.SetArchiveTemplateAsync(template : ModuleScript | table)
        Hot-swaps the default data template and rebuilds the ProfileStore.
        Pass either a ModuleScript (will be required) or a plain table.
        Call this before any sessions start — it is not safe to call while
        sessions are active.

        Example:
            ARC.SetArchiveTemplateAsync(ServerStorage.NewDefault)


    ARC.GetDefaultTemplate() → table
        Returns a shallow copy of the current default template.
        Safe to read at any time; modifying the returned table has no effect.

        Example:
            local defaults = ARC.GetDefaultTemplate()
            print(defaults.Coins)   -- 0


    ────────────────────────────────────────────────────
    5.2  GLOBAL HOOKS
    ────────────────────────────────────────────────────

    ARC.OnProfileLoaded(callback : (player, decodedData) -> ())
        Registers a function that fires each time any player's profile loads
        successfully. Receives the Player and a fully decoded copy of their
        data table. Runs in its own task so it cannot block session startup.

        Register as many callbacks as you like; all will be called.

        Example:
            ARC.OnProfileLoaded(function(player, data)
                -- Give new players a welcome reward.
                if data.TotalLogins == 0 then
                    ARC.ModifyArchiveValueAsync(player, "Coins", 100)
                end
            end)


    ARC.OnProfileEnded(callback : (player) -> ())
        Registers a function that fires each time any player's session ends
        (both normal leaves and forced session-end kicks). Runs in its own task.

        Example:
            ARC.OnProfileEnded(function(player)
                -- Clean up any server-side state keyed to this player.
                Leaderboard:Remove(player)
            end)


    ────────────────────────────────────────────────────
    5.3  SESSION MANAGEMENT
    ────────────────────────────────────────────────────

    ARC.StartPlayerSessionAsync(player : Player)
        Opens a ProfileStore session for the player, reconciles missing keys
        against the default template, and registers the session internally.
        Kicks the player with Config.Errors.InvalidProfile if the session
        cannot be obtained (e.g. the player's data is locked elsewhere).
        Safe to call if a session is already active — logs a warning and exits.

        Call this in Players.PlayerAdded.


    ARC.EndPlayerSessionAsync(player : Player)
        Ends the player's session, saving and releasing the lock.
        Safe to call if no session exists — silently returns.

        Call this in Players.PlayerRemoving.


    ARC.IsSessionActive(player : Player) → boolean
        Returns true if the player has an active, valid session.
        Synchronous — no yielding.

        Example:
            if not ARC.IsSessionActive(player) then return end


    ARC.GetActiveSessions() → { Player }
        Returns an array of all players with currently active sessions.

        Example:
            for _, player in ipairs(ARC.GetActiveSessions()) do
                print(player.Name .. " is active.")
            end


    ARC.ListenToSessionEnd(player : Player, callback : () -> ()) → RBXScriptConnection?
        Connects a one-time callback to this specific player's session end event.
        Returns the RBXScriptConnection so you can disconnect it early if needed.
        Returns nil if the player has no active session.

        Example:
            ARC.ListenToSessionEnd(player, function()
                print(player.Name .. " just left.")
            end)


    ────────────────────────────────────────────────────
    5.4  READING DATA
    ────────────────────────────────────────────────────

    ARC.FetchArchiveAsync(player : Player) → table
        Returns a fully decoded copy of the player's data table.
        Falls back to a copy of the default template if the player has
        no active session.

        ⚠  Returns a COPY — mutating the result does not affect stored data.
           Use ModifyArchiveAsync / ModifyArchiveValueAsync to write.

        Example:
            local data = ARC.FetchArchiveAsync(player)
            print(data.Level)


    ARC.FetchArchiveValueAsync(id, key : string) → any
        Returns a single decoded value for any player (live or offline).
        Falls back to Default[key] if the key is missing or nil.
        Never returns nil for a key that exists in the default template.

        Example:
            local level = ARC.FetchArchiveValueAsync(player, "Level")
            local coins = ARC.FetchArchiveValueAsync(123456789, "Coins")


    ARC.FetchArchiveValuesAsync(id, keys : { string }) → { [string]: any }
        Reads multiple keys in a single call.
        For offline players this opens exactly one session for all keys,
        rather than one session per key. Always prefer this over multiple
        FetchArchiveValueAsync calls when you need several values at once.

        Example:
            local stats = ARC.FetchArchiveValuesAsync(player, {
                "Coins", "Level", "XP", "Inventory"
            })
            print(stats.Coins, stats.Level)


    ────────────────────────────────────────────────────
    5.5  WRITING DATA
    ────────────────────────────────────────────────────

    ARC.ModifyArchiveAsync(id, callback : (data : table) -> ())
        Low-level write. Runs `callback` with the raw internal data table.
        Use this when you need to modify several keys in one atomic operation
        and the higher-level functions are not flexible enough.

        For live players the callback runs immediately against the live table.
        For offline players a temporary session is opened and closed around it.

        ⚠  The data table contains encoded (packed) values. Decode values
           yourself before reading them inside the callback, or use the
           higher-level functions which handle this automatically.

        Example:
            ARC.ModifyArchiveAsync(player, function(data)
                data["Coins"]  = ARC.Encode(500)
                data["Level"]  = ARC.Encode(10)
            end)


    ARC.ModifyArchiveValueAsync(id, key : string, value : any)
        Write a single value. Performs a type-check against the default
        template and encodes the value before storage.

        Example:
            ARC.ModifyArchiveValueAsync(player, "Coins", 750)
            ARC.ModifyArchiveValueAsync(player, "Inventory", { "Sword", "Shield" })


    ARC.ModifyArchiveValuesAsync(id, values : { [string]: any })
        Write multiple key-value pairs in a single session open.
        For offline players this is significantly faster than calling
        ModifyArchiveValueAsync once per key.
        Each value is type-checked and encoded individually.

        Example:
            ARC.ModifyArchiveValuesAsync(player, {
                Coins = 1000,
                Level = 5,
                XP    = 250,
            })


    ARC.UpdateArchiveValueAsync(id, key : string, transform : (current) → any)
        Atomic read-modify-write. The transform function receives the current
        decoded value (or the default if missing) and should return the new
        value to store. Type-checks the returned value before writing.

        For live players this is fully atomic — no other code can read an
        intermediate state. For offline players it opens one session.

        Example — safe increment:
            ARC.UpdateArchiveValueAsync(player, "Coins", function(c)
                return c + 50
            end)

        Example — append to a table:
            ARC.UpdateArchiveValueAsync(player, "Inventory", function(inv)
                table.insert(inv, "Potion")
                return inv
            end)


    ────────────────────────────────────────────────────
    5.6  SCHEMA MANAGEMENT
    ────────────────────────────────────────────────────

    ARC.ArchivePlayerAsync(id)
        Cleans and normalises a player's data against the current template:
          • Removes keys that no longer exist in the default template.
          • Adds missing keys using the default value.
          • Re-encodes all values to normalise storage format.
        Use this after a schema migration or as part of a scheduled cleanup job.

        Example:
            -- Migrate all offline saves after updating Default.lua.
            for _, userId in ipairs(legacyUserIds) do
                ARC.ArchivePlayerAsync(userId)
            end


    ARC.ArchivePlayersAsync(ids : { id })
        Runs ArchivePlayerAsync on an array of ids sequentially.

        Example:
            ARC.ArchivePlayersAsync({ player1, player2, 123456789 })


    ARC.WipeArchiveAsync(id)
        Resets a player's archive entirely to the default template values.
        All existing data is permanently lost.

        ⚠  This cannot be undone. Use with caution — typically only for
           admin commands or anti-cheat resets.

        Example:
            ARC.WipeArchiveAsync(bannedUserId)


    ────────────────────────────────────────────────────
    5.7  ENCODE / DECODE
    ────────────────────────────────────────────────────

    These are exposed so other systems can pack values the same way ARC does,
    without going through a full write cycle.

    ARC.Encode(value : any) → string?
        Encodes a value to a Base64 JAR string, exactly as ARC would before
        writing to the Datastore. Returns nil on failure.

        Example:
            local packed = ARC.Encode({ kills = 10, deaths = 2 })


    ARC.Decode(value : string | buffer) → any?
        Decodes a Base64 JAR string or raw buffer back to its original value.
        Returns nil on failure or if the input is nil.

        Example:
            local data = ARC.Decode(packed)
            print(data.kills)   -- 10


    ────────────────────────────────────────────────────
    5.8  DEBUG
    ────────────────────────────────────────────────────

    ARC.Inspect()
        Prints a summary of all currently active sessions to the output:

            ── ARC  active sessions: 3 ──
               [12345678] Player1
               [87654321] Player2
               [11223344] Player3

        Call from the server console or a BindableFunction during development.


    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    6. ERROR REFERENCE
    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

    All ARC errors are prefixed with [ARC ERROR] and printed via warn()
    so they appear in the Developer Console without crashing the server.

    [ARC ERROR] Session already active for <Name> — ignoring duplicate start.
        → StartPlayerSessionAsync was called twice for the same player.
          Check that PlayerAdded is not firing multiple times or that you
          are not calling Start from two separate scripts.

    [ARC ERROR] Could not open offline session for <userId> — session may be
    active elsewhere.
        → ARC tried to open a temporary offline session but ProfileStore
          could not acquire the lock. This usually means another server or
          a live session already holds the lock. Wait and retry.

    [ARC ERROR] Offline session callback failed for <userId>: <reason>
        → The callback passed to ModifyArchiveAsync threw an error while
          running against offline data. The session is still closed cleanly.
          Check the reason string for the root cause.

    [ARC ERROR] Live modify failed: <reason>
        → The callback passed to ModifyArchiveAsync threw an error while
          running against a live player's data. The error is caught so the
          session remains valid. Check the reason string.

    [ARC ERROR] Type mismatch for "<key>": expected <type>, got <type>
        → A value of the wrong type was passed to ModifyArchiveValueAsync,
          ModifyArchiveValuesAsync, or UpdateArchiveValueAsync. The write is
          skipped. Check that you are not accidentally passing a string where
          a number is expected (or vice versa).

    [ARC] Invalid ID: <value>
        → An id that is not a Player, number, or numeric string was passed
          to a data method. Check the caller.

    [ARC] Key must be a non-empty string, got: <value>
        → A nil or non-string key was passed. Likely a typo in the key name.

    [ARC] Expected a Player instance.
        → A session management function received a non-Player argument.
          Session functions (Start, End, IsActive, etc.) only accept Player
          instances — they do not accept UserIds.

    [ARC] Expected a callback function.
        → A non-function was passed where a callback was required.


    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    7. SECURITY & SAFETY
    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

    NEVER RETURNS NIL FOR TEMPLATE KEYS
        FetchArchiveValueAsync and FetchArchiveValuesAsync always fall back
        to the default template value if a key is absent or nil. Your code
        will never receive nil for a key that exists in Default.lua.

    TYPE SAFETY ON EVERY WRITE
        ModifyArchiveValueAsync, ModifyArchiveValuesAsync, and
        UpdateArchiveValueAsync all compare the new value's type against the
        type of the same key in the default template. Mismatches emit an error
        and skip the write, preventing type corruption in the Datastore.

    SESSION GUARD
        StartPlayerSessionAsync checks for an existing session before opening
        a new one. Duplicate calls are safe and log a warning rather than
        creating a conflicting second session.

    OFFLINE ACCESS IS ALWAYS LOCKED
        When ARC opens a temporary session for an offline player it uses
        ProfileStore's own session locking, so the same dupe-prevention
        guarantees apply to offline writes as they do to live ones.
        A failed lock emits an error and skips the operation rather than
        writing to an unlocked key.

    BINARY STORAGE — DO NOT HAND-EDIT
        Values are stored as Base64-encoded JAR binary strings in the
        Datastore. Reading them directly in the Datastore editor will show
        unreadable strings. Editing them by hand will corrupt the data.
        Always use ARC.Encode / ARC.Decode or the standard write methods.

    SERVER-ONLY
        ARC must live in ServerStorage and must never be required by a
        LocalScript or ModuleScript in ReplicatedStorage. All data access
        is server-authoritative. Expose data to clients only through
        RemoteEvents or Attributes, never by replicating ARC itself.


    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
    8. CHANGELOG
    ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

    v2.1.0  (2/19/2026)
    ───────────────────
    New methods:
      • FetchArchiveValuesAsync     — batch read, one session for all keys
      • ModifyArchiveValuesAsync    — batch write, one session for all keys
      • UpdateArchiveValueAsync     — atomic transform-based read-modify-write
      • WipeArchiveAsync            — reset a player to defaults
      • GetDefaultTemplate          — safe copy of the current template
      • GetActiveSessions           — array of all players with live sessions
      • OnProfileLoaded             — global hook: fires on every session start
      • OnProfileEnded              — global hook: fires on every session end
      • ListenToSessionEnd          — per-player session-end connection
      • IsSessionActive             — replaces SessionActiveAsync (now sync)
      • Inspect                     — prints active session summary
      • Encode / Decode             — replaces EncodeValueAsync/DecodeValueAsync
                                      (removed unused id parameter)

    Bug fixes:
      • FetchArchiveValueAsync offline path no longer opens a write session
        to perform a read — now uses a dedicated read-only session open.
      • FetchArchiveAsync now returns decoded values consistently; previously
        it returned the raw encoded data table for live players.
      • FetchArchiveValueAsync on the live path now decodes the value before
        returning it; previously it returned the raw encoded string.
      • ArchivePlayerAsync no longer double-encodes already-encoded values;
        it now decodes first, then re-encodes.
      • SetArchiveTemplateAsync now rebuilds the ProfileStore after updating
        the template; previously the old store was kept in memory.
      • StartPlayerSessionAsync now guards against duplicate calls.
      • ArchivePlayersAsync changed from pairs to ipairs and fixed the type
        annotation (was incorrectly typed as Players service).

    Renamed / removed:
      • SessionActiveAsync       → IsSessionActive  (synchronous bool, not async)
      • SessionEndedAsync        → ListenToSessionEnd  (clearer name & returns connection)
      • EncodeValueAsync         → Encode  (id parameter removed, it was unused)
      • DecodeValueAsync         → Decode  (id parameter removed, it was unused)

    v1.3.0  (2/15/2026)
    ───────────────────
    • Added automatic JAR packing for table values in ArchivePlayerAsync.
    • Added type-mismatch error on ModifyArchiveValueAsync.
    • Added Studio/Live profile name separation in Config.

    v1.0.0
    ───────
    • Initial release.
    • Session start / end, FetchArchiveAsync, ModifyArchiveAsync,
      ModifyArchiveValueAsync, ArchivePlayerAsync, basic encode/decode.
]]
