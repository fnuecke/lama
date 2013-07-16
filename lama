--[[
    LAMA - Location Aware Movement API - 2013 Sangar

    This program is licensed under the Creative Commons BY-SA 3.0 license.
    http://creativecommons.org/licenses/by-sa/3.0/

    The API will keep track of the turtle's position and facing, even across
    multiple games: the state is persistent. In particular it is very robust,
    i.e. even if the turtle is currently moving when it is forced to shut down
    because its chunk is unloaded, the coordinates will always be correct after
    it resumes. In theory, anyways. It's the only actually working approach I'm
    aware of, using the turtle's fuel level to check if it moved in such cases.

    The API is relatively basic, in that it is purely focused on movement. It's
    essentially a rich drop-in replacement for the the original navigational
    functions in the turtle API:
        turtle.forward   -> lama.forward
        turtle.back      -> lama.back
        turtle.up        -> lama.up
        turtle.down      -> lama.down
        turtle.turnRight -> lama.turnRight
        turtle.turnLeft  -> lama.turnLeft

    When using this API, you must not use any other functions that alter
    the turtle's position or facing. In particular:

        DO NOT USE turtle.forward, turtle.back, turtle.up, turtle.down,
        turtle.turnRight or turtle.turnLeft NOR THEIR NATIVE EQUIVALENTS.

    Any other external force changing the turtle's position will also
    invalidate the coordinates, of course (such as the player pickaxing the
    turtle and placing it somewhere else or RP2 frames).

    The utility function lama.hijackTurtleAPI() can be used to override the
    original turtle API functions, to make it easier to integrate this API into
    existing programs.
    When starting a new program it is recommended using the functions directly
    though, to make full use of their capabilities (in particular automatically
    clearing the way). See the function's documentation for more information.
]]

-------------------------------------------------------------------------------
-- Constants / config                                                        --
-------------------------------------------------------------------------------

-- This is the name of the file in which we store our state, i.e. the position
-- and facing of the turtle, as well as whether it is currently moving or not.
-- You may want to change this if it collides with another program or API.
local stateFile = "/.lama-state"

-- The filename of the file to backup any original startup file to when
-- creating the startup file used to finish any running moves in case the
-- turtle is forced to shut down during the move. This is only used if no
-- multi-startup-script system is found on the computer.
local startupBackupFile = "/.lama-startup-backup"

-------------------------------------------------------------------------------
-- Environment checking                                                      --
-------------------------------------------------------------------------------

-- This API makes no sense when used on anything other than a turtle.
if not turtle then
    error("Can only run on turtles.")
end

-- Check if this computer has a label. Because if it hasn't this is pointless.
if not os.getComputerLabel() then
    error("Turtle has no label, required for state persistence.")
end

-- Check if turtles use fuel. If they don't, we cannot reliably check if the
-- turtle moved after it was forcibly shut down.
if turtle.getFuelLevel() == "unlimited" then
    error("Turtles must use fuel for this API to work correctly.")
end

-------------------------------------------------------------------------------
-- Internal state variable and forward declarations for utility functions    --
-------------------------------------------------------------------------------

-- We keep all of the state in a common table for easier serialization.
local state = {
    -- Coordinates and facing.
    position = {x = 0, y = 0, z = 0, f = 0},
    -- Whether we're currently moving.
    moving = false,
    -- Only relevant if moving is not false, in which case it's the fuel level
    -- from before triggering the move. This is used to check if we actually
    -- moved in case the turtle is forcibly shut down during the move (due to
    -- the server quitting or the chunk unloading, for example).
    preMoveFuel = 0,
    -- Number of tries left for the current move command.
    tries = 0,
    -- Whether we may attack obstacles when retrying.
    aggressive = false
}

-- Forward declaration of namespace for local utility methods.
local private = {}

-------------------------------------------------------------------------------
-- API                                                                       --
-------------------------------------------------------------------------------

-- The current version of the API.
version = "1.0"

-- Constants for a turtle's facing, used for turn() and get()/set().
side = {forward = 0, right = 1, back  = 2, left = 3,
        north   = 0, east  = 1, south = 2, west = 3,
        front   = 0}

-- Reasons for movement failure. One of these is returned as a second value by
-- the movement functions (forward, back, up, down) if they fail.
reason = {
    -- We could not determine what was blocking us. Mostly used when moving
    -- backwards, since we cannot detect anything in that direction.
    unknown = "unknown",

    -- The VM's event queue is full, meaning no further commands can be issued
    -- until some of them are processed first.
    queue_full = "queue_full",

    -- Another corouting is still waiting for a move to finish.
    coroutine = "coroutine",

    -- The fuel's empty so we cannot move at all.
    fuel = "fuel",

    -- Some block is in the way. If we had multiple tries, this means the block
    -- respawned/regenerated, so we either have a stack of sand/gravel that is
    -- higher than the number of tries was, or a cobblestone generator or
    -- something similar.
    block = "block",

    -- Another turtle got in our way. If we had multiple tries, this means the
    -- turtle did not move out of the way, so it's either not moving or wants
    -- to move to where this turtle is (direct opposite move direction).
    turtle = "turtle",

    -- Some unbreakable block is in the way. This can only be determined when
    -- we have multiple tries, in which case this is returned if the dig()
    -- command returns false.
    unbreakable_block = "unbreakable_block",

    -- Some entity is in our way. This is the case when we're blocked but no
    -- block is detected in the desired direction.
    entity = "entity",

    -- Some invulnerable entity is our way. This can only be determined when we
    -- have multiple tries and we're aggressive, in which case this is returned
    -- if the attack() command fails.
    invulnerable_entity = "invulnerable_entity"
}

-- Raise error when trying to access invalid key in the constant tables.
setmetatable(side, {
    __index = function(t,k)
        error("Trying to access invalid 'side' constant '" .. k .. "'.")
    end,
    __newindex = function()
        error("Trying to modify readonly table.")
        -- Yes, existing entries can be overwritten like this, but I prefer to
        -- keep the table enumerable via pairs.
    end
})
setmetatable(reason, {
    __index = function(t,k)
        error("Trying to access invalid 'reason' constant '" .. k .. "'.")
    end,
    __newindex = function()
        error("Trying to modify readonly table.")
        -- Yes, existing entries can be overwritten like this, but I prefer to
        -- keep the table enumerable via pairs.
    end
})

--------------------------------
-- Movement related functions --
--------------------------------

--[[
    Try to move the turtle forward.

    @param tries how often to try to move. If this larger than zero, the turtle
        will try to dig its way through any obstructions as many times (e.g. to
        get through stacks of sand or gravel).
    @param aggressive if set to true, will also try to attack to clear its way
        when obstructed - even if it's the player!
    @return true if the turtle moved successfully, (false, reason) if it failed
        and is still in the same position.
]]
function forward(tries, aggressive)
    if state.moving then
        return false, reason.coroutine
    end
    return private.move("forward", tries, aggressive)
end

--[[
    Try to move the turtle backward.

    Note that this does not have the 'tries' and 'aggressive' parameters, since
    the turtle would have to turn around first in order to dig or attack.

    @param tries how often to try to move. If this larger than zero, the turtle
        will try to wait for any obstructions it hits to go away as many times.
        As opposed to the other movement functions this will not dig nor attack
        and only wait for other turtle to pass. In the other cases it will
        immediately return false.
    @return true if the turtle moved successfully, (false, reason) if it failed
        and is still in the same position.
]]
function back(tries)
    if state.moving then
        return false, reason.coroutine
    end
    return private.move("back", tries)
end

--[[
    Try to move the turtle up.

    @param tries how often to try to move. If this larger than zero, the turtle
        will try to dig its way through any obstructions as many times (e.g. to
        get through stacks of sand or gravel).
    @param aggressive if set to true, will also try to attack to clear its way
        when obstructed - even if it's the player!
    @return true if the turtle moved successfully, (false, reason) if it failed
        and is still in the same position.
]]
function up(tries, aggressive)
    if state.moving then
        return false, reason.coroutine
    end
    return private.move("up", tries, aggressive)
end

--[[
    Try to move the turtle down.

    @param tries how often to try to move. If this larger than zero, the turtle
        will try to dig its way through any obstructions as many times (e.g. to
        get through stacks of sand or gravel).
    @param aggressive if set to true, will also try to attack to clear its way
        when obstructed - even if it's the player!
    @return true if the turtle moved successfully, (false, reason) if it failed
        and is still in the same position.
]]
function down(tries, aggressive)
    if state.moving then
        return false, reason.coroutine
    end
    return private.move("down", tries, aggressive)
end

-----------------------------------
-- Orientation related functions --
-----------------------------------

--[[
    Turn the turtle right.

    @return true if the turtle turned successfully, false otherwise.
]]
function turnRight()
    local id = turtle.native.turnRight()
    if id >= 0 then
        state.position.f = (state.position.f + 1) % 4
        private.save()
        return private.waitForResponse(id)
    end
    return false
end

--[[
    Turn the turtle left.

    @return true if the turtle turned successfully, false otherwise.
]]
function turnLeft()
    local id = turtle.native.turnLeft()
    if id >= 0 then
        state.position.f = (state.position.f - 1) % 4
        private.save()
        return private.waitForResponse(id)
    end
    return false
end

--[[
    Turn the turtle around.

    @return true if the turtle turned successfully, (false, reason) if it
        failed - in this case the turtle may also have turned around halfway.
        Only fails if the event queue is full.
]]
function turnAround()
    return turn(state.position.f + 2)
end

--[[
    Turn the turtle to face the specified direction.

    @param towards the direction in which the turtle should face.
    @return true if the turtle turned successfully, (false, reason) if it
        failed - in this case the turtle may already have turned partially
        towards the specified facing. Only fails if the event queue is full.
    @see lama.side
]]
function turn(towards)
    -- Ensure new facing is in bounds.
    towards = towards % 4
    -- Turn towards the target facing.
    local ids = {}
    while state.position.f ~= towards do
        -- We do not use the turnLeft() and turnRight() functions, because we
        -- want full control: we push all native events in one go and then wait
        -- for all of them to finish. This way we can stick to the pattern of
        -- immediately returning (non-yielding) if the turn fails due to a full
        -- event queue.
        local id
        if towards == (state.position.f + 1) % 4 then
            -- Special case for turning clockwise, to avoid turning three times
            -- when once is enough, in particular for the left -> forward case,
            -- where we wrap around (from 3 -> 0).
            id = turtle.native.turnRight()
            if id >= 0 then
                state.position.f = (state.position.f + 1) % 4
            else
                return false
            end
        else
            id = turtle.native.turnLeft()
            if id >= 0 then
                state.position.f = (state.position.f - 1) % 4
            else
                return false
            end
        end
        private.save()
        table.insert(ids, id)
    end
    return private.waitForResponse(ids)
end

-----------------------------
-- State related functions --
-----------------------------

--[[
Note: all coordinates are initially relative to the turtle's origin,
      i.e. to where it has been placed and this API was first used.
      If the turtle is moved in some way other than via the functions
      of this API (pickaxed by player and placed somewhere else, RP2
      frames, ...) the coordinates will refer to somewhere else in
      world space, since the origin has changed!
]]

--[[
    Get the position and facing of the turtle.

    @return a tuple (x, y, z, facing).
]]
function get()
    local position = state.position
    return position.x, position.y, position.z, position.f
end

--[[
    Get the current X coordinate of the turtle.

    @return the turtle's current X position.
]]
function getX()
    return state.position.x
end

--[[
    Get the current Y coordinate of the turtle.

    @return the turtle's current Y position.
]]
function getY()
    return state.position.y
end

--[[
    Get the current Z coordinate of the turtle.

    @return the turtle's current Z position.
]]
function getZ()
    return state.position.z
end

--[[
    Get the current X,Y,Z coordinates of the turtle as a vector.

    @return a vector instance representing the turtle's position.
]]
function getPosition()
    local position = state.position
    return vector.new(position.x, position.y, position.z)
end

--[[
    Get the direction the turtle is currently facing.

    @return the current orientation of the turtle.
    @see lama.side
]]
function getFacing()
    return state.position.f
end

--[[
    Sets the position and facing of the turtle.

    This can be useful to calibrate the turtle to a new origin, e.g. after
    placing it, to match the actual world coordinates. The facing must be one
    of the lama.side constants.

    @return the new position and facing of the turtle (like lama.get()).
]]
function set(x, y, z, f)
    local position = state.position
    position.x = tonumber(x or position.x)
    position.y = tonumber(y or position.x)
    position.z = tonumber(z or position.x)
    position.f = tonumber(f or position.f) % 4
    private.save()
    return get()
end

-----------------------
-- Utility functions --
-----------------------

--[[
    Gets the result of resuming a move on startup.

    This can be used to query whether a move that was interrupted and continued
    on startup finished successfully or not, and if not for what reason.

    For example: imagine you issue lama.forward() and the program stops. Your
    program restores its state and knows it last issued the forward command,
    but further execution depends on whether that move was successful or not.
    To check this after resuming across unloading you'd use this function.

    Note that this will return true in case the startup did not continue a move
    (program was not interrupted during a move).

    @return true or a tuple (result, reason) based on the startup move result.
]]
function startupResult()
    if not private.startupResult then
        return true
    end
    return private.startupResult.result, private.startupResult.reason
end

--[[
    Replaces the movement related functions in the turtle API.

    This makes it easier to integrate this API into existing programs.
    This does NOT touch the native methods.
    The injected functions will NOT be the same as calling the API function
    directly, to avoid changes in existing programs when this is dropped in.

    For example: a call to turtle.forward(1) will return false if the turtle is
    blocked, whereas lama.forward(1) would try to destroy the block, and then
    move. The function replacing turtle.forward() will behave the same as the
    old one, in that the parameter will be ignored. This follows the principle
    of least astonishment.

    @param restore whether to restore the original turtle API functions.
]]
function hijackTurtleAPI(restore)
    -- Wrap methods to avoid accidentally passing parameters along. This is
    -- done to make sure behavior is the same even if the functions are
    -- called with (unused/invalid) parameters.
    if restore then
        if not turtle.__lama then return end
        turtle.forward   = turtle.__lama.forward
        turtle.back      = turtle.__lama.back
        turtle.up        = turtle.__lama.up
        turtle.down      = turtle.__lama.down
        turtle.turnRight = turtle.__lama.turnRight
        turtle.turnLeft  = turtle.__lama.turnLeft
        turtle.__lama = nil
    else
        if turtle.__lama then return end
        turtle.__lama = {
            forward   = turtle.forward,
            back      = turtle.back,
            up        = turtle.up,
            down      = turtle.down,
            turnRight = turtle.turnRight,
            turnLeft  = turtle.turnLeft,
        }
        turtle.forward   = function() return forward() ~= false end
        turtle.back      = function() return back()    ~= false end
        turtle.up        = function() return up()      ~= false end
        turtle.down      = function() return down()    ~= false end
        turtle.turnRight = turnRight
        turtle.turnLeft  = turnLeft
    end
end

-------------------------------------------------------------------------------
-- State saving/loading                                                      --
-------------------------------------------------------------------------------

--[[
    Saves the current position, facing and movement state.

    @private
]]
function private.save()
    -- Serialize before opening the file, just in case.
    local t = textutils.serialize(state)
    local f = fs.open(stateFile, "w")
    f.write(t)
    f.close()
end

--[[
    Restores the position, facing and movement state.

    @private
]]
function private.load()
    -- If we don't have a state file, we have nothing to do here.
    if not fs.exists(stateFile) or fs.isDir(stateFile) then
        return
    end
    local f = fs.open(stateFile, "r")
    local t = f.readLine()
    f.close()
    -- Unserialize after closing the file, just in case.
    local s = textutils.unserialize(t)
    -- Validate the read state.
    if not type(s.position) == "table" or
       not type(s.position.x == "number") or
       not type(s.position.y == "number") or
       not type(s.position.z == "number") or
       not type(s.position.f == "number") or
       not (type(s.moving) == "boolean" or type(s.moving) == "string") or
       not type(s.preMoveFuel) == "number" or
       not type(s.tries) == "number"
    then
        print("LAMA: Invalid state file, deleting and ignoring it.")
        fs.delete(stateFile)
    else
        state = s
    end
end

-------------------------------------------------------------------------------
-- Resume command on startup stuff                                           --
-------------------------------------------------------------------------------

-- List of 'handlers'. This makes it easy to add support for different startup
-- script implementations (e.g. from different OSs or utility scripts).
local startupHandlers = {
    -- Default implementation creates a wrapper startup script and moves the
    -- original startup script, if any, to a backup location to be restored
    -- when the startup script is run. This has rather bad performance because
    -- it adds one file write, one deletion and two moves to each move command.
    default = {
        wrap = function()
            local haveStartup = fs.exists("/startup")
            if haveStartup then
                fs.move("/startup", startupBackupFile)
            end

            local f = fs.open("/startup", "w")
            f.writeLine("os.loadAPI('lama')")
            if haveStartup then
                f.writeLine("shell.run('/startup')")
            else
            end
            f.close()
        end,
        unwrap = function()
            fs.delete("/startup")
            if fs.exists(startupBackupFile) then
                fs.move(startupBackupFile, "/startup")
            end
        end
    },

    -- Implementation for using Forairan's init-script startup program. This
    -- will only create a startup script once, which has no side effects if
    -- there were no pending moves, so performance impact is minimal.
    forairan = {
        init = function()
            -- Remember whether we already have the startup file for our API.
            if not fs.exists("/init-scripts/00_lama") then
                local f = fs.open("/init-scripts/00_lama", "w")
                f.write("os.loadAPI('lama')")
                f.close()
            end
        end
    }
}

-- Detect which handler to use.
if fs.exists("/init-scripts") and fs.isDir("/init-scripts") then
    -- Found init-scripts folder, assume Forairan's startup script.
    private.startupHandler = startupHandlers.forairan
else
    -- Fall back to default implementation.
    private.startupHandler = startupHandlers.default
end

-- Run handler's init code.
if private.startupHandler.init then
    private.startupHandler.init()
end

--[[
    Creates a new startup file which initializes this API.

    Will backup any existing startup file and restore it when the generated
    startup file is run.

    @private
]]
function private.wrapStartup()
    if private.isStartupWrapped then
        return
    end
    private.isStartupWrapped = true
    if private.startupHandler.wrap then
        private.startupHandler.wrap()
    end
end

--[[
    Removes the startup file used for initializing the API.

    Restores the original startup file, if there was any.

    @param force whether to force the unwrapping, used in init.
    @private
]]
function private.unwrapStartup(force)
    if not private.isStartupWrapped and not force then
        return
    end
    private.isStartupWrapped = false
    if private.startupHandler.unwrap then
        private.startupHandler.unwrap()
    end
end

-------------------------------------------------------------------------------
-- Movement implementation                                                   --
-------------------------------------------------------------------------------

--[[
    Waits for queued commands to finish.

    @param ids the response ID or list of IDs of the commands to wait for.
    @return true if the commands were executed successfully, false otherwise.
    @private
]]
function private.waitForResponse(ids)
    if type(ids) ~= "table" then
        ids = {ids}
    elseif #ids == 0 then
        return true
    end
    local success = true
    repeat
        local event, responseID, result = os.pullEvent("turtle_response")
        if event == "turtle_response" then
            for i=1,#ids do
                if ids[i] == responseID then
                    success = success and result
                    table.remove(ids, i)
                    break
                end
            end
        end
    until #ids == 0
    return success
end

--[[
    Figures out why a turtle cannot move in the specified direction.

    @param direction the direction to check for.
    @return one of the lama.reason constants.
    @private
]]
function private.tryGetReason(direction)
    local detect = ({
        forward = turtle.detect,
        up      = turtle.detectUp,
        down    = turtle.detectDown})[direction]
    local side = ({
        forward = "front",
        up      = "top",
        down    = "bottom"})[direction]
    if detect then
        if detect() then
            if peripheral.isPresent(side) and
               peripheral.getType(side) == "turtle"
            then
                -- A turtle is blocking us.
                return reason.turtle
            else
                -- Some other block is in our way.
                return reason.block
            end
        else
            -- Not a block, so we can assume it's some entity.
            return reason.entity
        end
    end

    -- Cannot determine what's blocking us.
    return reason.unknown
end

--[[
    Tries to move the turtle in the specified direction.

    If it doesn't work. checks whether we should try harder: if a number of
    tries is specified we'll dig (and if aggressive is set attack) as many
    times in the hope of getting somewhere.

    Use math.huge for infinite tries.

    @param direction the direction in which to move, i.e. forward, back, up or
        down. The appropriate functions are selected based on this value.
    @param tries if specified, the number of times to retry the move after
        trying to remove obstacles in our way. We may want to try more than
        once for stacks of sand/gravel or enemy entities.
    @param aggressive whether to allow attacking in addition to digging when
        trying to remove obstacles (only used if tries larger than zero).
    @param true if the move was successful, false otherwise.
    @private
]]
function private.move(direction, tries, aggressive)
    -- Check our fuel.
    if turtle.getFuelLevel() < 1 then
        return false, reason.fuel
    end

    -- Clean up arguments.
    tries = tonumber(tries or 0)

    -- Mapping for functions based on movement direction.
    local move = ({
        forward = turtle.native.forward,
        back    = turtle.native.back,
        up      = turtle.native.up,
        down    = turtle.native.down})[direction]
    local detect = ({
        forward = turtle.detect,
        up      = turtle.detectUp,
        down    = turtle.detectDown})[direction]
    local dig = ({
        forward = turtle.dig,
        up      = turtle.digUp,
        down    = turtle.digDown})[direction]
    local attack = ({
        forward = turtle.attack,
        up      = turtle.attackUp,
        down    = turtle.attackDown})[direction]
    local side = ({
        forward = "front",
        up      = "top",
        down    = "bottom"})[direction]

    -- Try to move until we're out of tries (or successful).
    local success
    while true do
        -- Initialize the move by calling the native function. Store the fuel
        -- level before the command to allow testing for a successful move in
        -- case we are interrupted by a server quit / chunk unload while
        -- waiting for the move to finish.
        state.preMoveFuel = turtle.getFuelLevel()
        local moveId = move()
        if moveId < 0 then
            -- Too many events queued already.
            state.preMoveFuel = 0
            private.unwrapStartup()
            return false, reason.queue_full
        end
        state.moving = direction
        state.tries = tries
        state.aggressive = aggressive
        private.save()

        -- Setup movement resumption via startup file.
        private.wrapStartup()

        -- Wait for movement to finish.
        success = private.waitForResponse(moveId)

        -- Update state and flush it to disk.
        private.updateState(success, success)

        -- If something went wrong check whether we should try again.
        if not success and tries > 0 then
            -- See what seems to be blocking us.
            if detect and detect() then
                -- Its a block, see if it's a turtle, otherwise try to hit it.
                if peripheral.isPresent(side) and
                   peripheral.getType(side) == "turtle"
                then
                    -- It's a turtle. Just wait for a bit to allow it to move
                    -- out of the way.
                    os.sleep(2)
                elseif dig and not dig() and tries == 1 then
                    -- Some block we can't dig, so there's nothing we can do.
                    private.unwrapStartup()
                    return false, reason.unbreakable_block
                end
            elseif aggressive and attack and not attack() and tries == 1 then
                -- Not a block but nothing we can/may attack. Abort.
                private.unwrapStartup()
                return false, reason.invulnerable_entity
            else
                -- We cannot do anything about the obstruction, just wait for a
                -- bit, maybe it'll pass.
                os.sleep(1)
            end
            -- We either destroyed a block, attacked something, or did nothing!
            -- Let's try again. Doin' it right. Dat bass.
            tries = tries - 1
            -- Wait a little to allow sand/gravel to drop down. I've had cases
            -- where the turtle would get into a weird state in which it moved
            -- below a dropping sand block, causing bad behavior.
            os.sleep(0.5)
        else
            -- We were either successful, or we're out of tries.
            break
        end
    end

    -- Restore startup file after we're done.
    private.unwrapStartup()

    -- Return if we're successful, and when not, for what reason.
    if success then
        return true
    else
        return false, private.tryGetReason(direction)
    end
end

--[[
    Finishes a movement.

    Based on whether it was successful or not it adjusts and then saves the new
    persistent state.

    @private
]]
function private.updateState(success, finish)
    -- Check if we actually moved.
    if success then
        -- Shortcuts.
        local position, direction = state.position, state.moving

        -- Yes, update our state. Build a table with the displacement we'd
        -- have to apply in our identity state.
        local delta = {
            forward = { 1,  0,  0},
            right   = { 0,  1,  0},
            back    = {-1,  0,  0},
            left    = { 0, -1,  0},
            up      = { 0,  0,  1},
            down    = { 0,  0, -1}
        }

        -- Apply our facing.
        for i=1,position.f do
            delta.forward, delta.right, delta.back, delta.left =
                delta.right, delta.back, delta.left, delta.forward
        end

        -- Finally, apply the actual displacement, based on the movement
        -- direction. This means we may do some extra work when moving
        -- up or down (where the facing doesn't matter), but it's not that
        -- bad, considering how long one move takes.
        position.x = position.x + delta[direction][1]
        position.y = position.y + delta[direction][2]
        position.z = position.z + delta[direction][3]
    end

    -- Switch back to non-moving state.
    if finish then
        state.moving = false
        state.preMoveFuel = 0
        state.tries = 0
        state.aggressive = false
    end

    private.save()
end

-------------------------------------------------------------------------------
-- Initialization                                                            --
-------------------------------------------------------------------------------

-- Load state, if any.
private.load()

-- Finish any active moves. Only do this on the first API load.
if not lama and state.moving then
    -- Make sure any active movement can complete.
    os.sleep(1) -- Wait a bit so that at least one slot in event queue is free.
    turtle.detect() -- Process event queue (blocks until processed).
    private.unwrapStartup(true)

    -- Here is the trick: if we did complete a move started directly before
    -- the turtle was shut down (server quit or chunk unload) then we expect
    -- to have exactly one fuel less than before, since only successful moves
    -- use up (exactly one) fuel.
    if turtle.getFuelLevel() == state.preMoveFuel then
        -- No fuel was used, so we didn't move. This can only fail if we
        -- refuel directly after moving, which cannot happen unless its pretty
        -- much forced using coroutines... so yeah, this will do.

        -- If we have some tries left, continue trying.
        if state.tries > 0 then
            local result, why =
                private.move(state.moving, state.tries, state.aggressive)
            private.startupResult = {
                result = result,
                reason = why
            }
        else
            private.startupResult = {
                result = false,
                reason = private.tryGetReason(state.moving)
            }
            private.updateState(false, true)
        end
    elseif turtle.getFuelLevel() == state.preMoveFuel - 1 then
        -- We used one fuel so we made our move! As with the case above, this
        -- can only be wrong if refueling is involved somewhere, which, again,
        -- can only happen using coroutines.
        private.updateState(true, true)
    else
        -- This should not be possible if the API is used correctly, i.e. it
        -- is not called from multiple coroutines and no other movement
        -- functions of the turtle are used.
        fs.delete(stateFile)
        error("Unexpected fuel state after reboot, probably invalid API use.")
    end
end