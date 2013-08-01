--[[
    This is a small example program, demonstrating how to use LAMA.

    To use it you have to define a couple of waypoints first, then run this
    program with the names of the waypoints to travel to. For example:

		lama-conf add home 0 0 0 north
        lama-conf add tl 2 0 1
        lama-conf add tr 2 2 2 west
        lama-conf add br 0 2 1 east

        lama-example tl tr br tr home

    Note that what this program does could also be achieved by simply calling
    lama.navigate(), but I had no other ideas for (short) examples...
]]

-- Load the LAMA API.
os.loadAPI("apis/lama")

-------------------------------------------------------------------------------
-- Constants / config                                                        --
-------------------------------------------------------------------------------

-- This is the name of the file in which we store our state information.
local stateFile = "/.lama-example-state"

-------------------------------------------------------------------------------
-- Internal state                                                            --
-------------------------------------------------------------------------------

-- Our state. This is essentially just the list of waypoints to travel.
local state = {}

-------------------------------------------------------------------------------
-- State saving/loading                                                      --
-------------------------------------------------------------------------------

-- Saves the current state.
local function save()
    local f = fs.open(stateFile, "w")
    f.write(textutils.serialize(state))
    f.close()
end

-- Loads an existing state.
local function load()
    local f = fs.open(stateFile, "r")
    state = textutils.unserialize(f.readAll())
    f.close()
end

-------------------------------------------------------------------------------
-- Movement logic                                                            --
-------------------------------------------------------------------------------

local function travel()
    while #state > 0 do
    	local name = table.remove(state)
        save()
    	print("Moving to '" .. name .. "'...")
        local result, reason = lama.waypoint.moveto(name)
        if not result then
        	print("Failed moving to waypoint: " .. reason)
        	break
        end
    end
    fs.delete(stateFile)
end

-------------------------------------------------------------------------------
-- Resume command on startup stuff                                           --
-------------------------------------------------------------------------------

if fs.exists(stateFile) then
    local result, reason = lama.startupResult()
    if result then
        print("Resuming journey...")
        load()
        travel()
    else
        print("Failed moving to waypoint: " .. reason)
        fs.delete(stateFile)
    end
else
    print("Checking input...")
    local tArgs = {...}
    for _, name in ipairs(tArgs) do
        if lama.waypoint.exists(name) then
            table.insert(state, 1, name)
        else
            print("Skipping unknown waypoint '" .. name .. "'.")
        end
    end
    save()
    travel()
end