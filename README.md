    =====================================================================================
                                    Welcome to ARC!
    =====================================================================================
    
    Date: 2/9/2026
    Author: @Kr3ativeKrayon / @Slunderman2468
    Version: 1.0.0
    
    ------------------------------------Description--------------------------------------
    ARC is a high-performance, session-locked data persistence layer. It manages 
    the "Archive" of player state, utilizing JAR for compression and ProfileStore 
    for robust, dupe-proof storage.
    
    ------------------------------------How To Use---------------------------------------
    
    > Setup
        1. Place the module in ServerStorage (Server-Only).
        2. Configure Default.lua and Config.lua in the _ARC folder.
        3. Require and call ARC.StartPlayerSessionAsync(player) on join.
    <
    
    > Basic Example
        local ARC = require(path.to.ARC)
        
        -- Give a player gold safely (Live or Offline)
        ARC.ModifyArchiveValueAsync(player, "Gold", 500)
        
        -- Run complex logic on data
        ARC.ModifyArchiveAsync(player, function(data)
            data.Experience += 100
            if data.Experience >= 1000 then
                data.Level += 1
            end
        end)
    <
    
    > Documentation
        > Session Methods
            > ARC.StartPlayerSessionAsync(player : Player) -- Starts session & locks data
            > ARC.EndPlayerSessionAsync(player : Player)   -- Releases lock & saves
            > ARC.SetArchiveTemplateAsync(template : table) -- Updates reconciliation template
        <
        
        > Data Methods
            > ARC.FetchArchiveAsync(player : Player) -- Returns live .Data table
            > ARC.FetchArchiveValueAsync(id, key) -- Returns specific value (auto-decodes JAR)
            > ARC.ModifyArchiveAsync(id, callback) -- Safely modifies data via callback
            > ARC.ModifyArchiveValueAsync(id, key, value) -- Sets a specific key (auto-encodes JAR)
        <
        
        > Maintenance
            > ARC.NoahsAsync(id) -- The "Great Flood": Purges non-template keys & JAR-packs survivors
        <
        
    ------------------------------------Security & Safety-----------------------------------

     > IMPORTANT
        1. **Session Locking:** ARC uses strict session locking. If a player is "locked," 
           they cannot load data in other servers. Always ensure EndPlayerSessionAsync 
           is called on leave.
        2. **NoahsAsync:** This is a destructive cleanup tool. It will permanently 
           delete any data keys not found in your Default template. Use with caution.
        3. **JAR Integration:** ARC automatically detects JAR-encoded strings and 
           converts them back to tables. Do not manually modify JAR strings in the 
           DataStore or the "seal" may break.
    <
