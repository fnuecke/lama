--[[
    LAMA - Location Aware Movement API - 2013 Sangar

    This program is licensed under the MIT license.
    http://opensource.org/licenses/mit-license.php

    This script allows interacting with the API from the shell.
]]

os.loadAPI("apis/lama")

-------------------------------------------------------------------------------
-- Commands                                                                  --
-------------------------------------------------------------------------------

-- Forward declation for helper function namespace.
local private = {}

-- List of all commands of this program. Keep the variable declaration
-- separate, so that it can be referenced in the help callback.
local commands
commands = {
    help = {
        help = "Shows the help text for the specified command.",
        args = {{"command", "The command to get help on."}},
        call = function(name)
            local command = commands[name]
            if command == nil then
                print("No such command.")
            else
                print(command.help)
                if command.args and #command.args > 0 then
                    print("Parameters:")
                    for _, arg in ipairs(command.args) do
                        print(" " .. arg[1] .. " - " .. arg[2])
                    end
                end
            end
        end
    },
    get = {
        help = "Prints the current position and facing of the turtle.",
        call = function()
            write("Position = ")
            private.display(lama.get())
        end
    },
    set = {
        help = "Sets the absolute coordinates of the turtle.",
        args = {
            {"x", "The X coordinate."},
            {"y", "The Y coordinate."},
            {"z", "The Z coordinate."},
            {"facing", "The facing."}
        },
        call = function(x, y, z, f)
            x, y, z, f = private.validate(x, y, z, f)
            write("Position = ")
            private.display(lama.set(x, y, z, f))
        end
    },
    reset = {
        help = "Alias for 'set 0 0 0 0'.",
        call = function()
            write("Position = ")
            private.display(lama.set(0, 0, 0, 0))
        end
    },
    list = {
        help = "Prints a list of known waypoints.",
        call = function()
            for name, x, y, z, f in lama.waypoint.iter() do
                write(name .. " ")
                private.display(x, y, z, f)
            end
        end
    },
    add = {
        help = "Adds a new waypoint.",
        args = {
            {"name", "The name of the waypoint."},
            {"x", "The X coordinate."},
            {"y", "The Y coordinate."},
            {"z", "The Z coordinate."},
            {"facing", "The optional facing."}
        },
        call = function(name, x, y, z, f)
            x, y, z, f = private.validate(x, y, z, f, true)
            if lama.waypoint.add(name, x, y, z, f) then
                print("Updated waypoint.")
            else
                print("Added waypoint.")
            end
        end
    },
    remove = {
        help = "Removes a waypoint.",
        args = {{"name", "The name of the waypoint."}},
        call = function(name)
            if not lama.waypoint.remove(name) then
                print("No such waypoint.")
            else
                print("Removed waypoint.")
            end
        end
    }
}

-------------------------------------------------------------------------------
-- Helper functions                                                          --
-------------------------------------------------------------------------------

--[[
    Used to parse a coordinate from the arguments.

    @param start the index at which to start parsing.
    @param facingOptional whether the facing is optional.
]]
function private.validate(x,y , z, f, facingOptional)
    -- Make sure we have all we need.
    if x == nil or y == nil or z == nil or
       (f == nil and not facingOptional)
    then
        error("Missing argument.")
    end

    -- Format the input.
    x = math.floor(tonumber(x))
    y = math.floor(tonumber(y))
    z = math.floor(tonumber(z))
    if f ~= nil then
        f = string.lower(f)
        -- Rawget to avoid error on invalid indexing.
        assert(rawget(lama.side, f), "Invalid facing '" .. f ..
            "'. Must be one of {north, east, south, west}.")
        f = lama.side[f]
    end

    -- Return the processed coordinates.
    return x, y, z, f
end

--[[
    Utility function for printing a coordinate.

    @param x the X coordinate.
    @param y the Y coordinate.
    @param z the Z coordinate.
    @param f the optional facing.
]]
function private.display(x, y, z, f)
    if f ~= nil then
        print(string.format("(%d, %d, %d, %s)", x, y, z, lama.side[f]))
    else
        print(string.format("(%d, %d, %d)", x, y, z))
    end
end

-------------------------------------------------------------------------------
-- Command logic                                                             --
-------------------------------------------------------------------------------

--[[
    Automatically generates a usage description, based on the command list.
]]
local function usage()
    local programName = fs.getName(shell.getRunningProgram())
    print("Usage: " .. programName .. " <command> <args...>")
    local names = {}
    for name, _ in pairs(commands) do
        table.insert(names, name)
    end
    table.sort(names)
    print("Commands: " .. table.concat(names, ", "))
    print("Use '" .. programName ..
          " help <command>' to get more information on a specific command.")
end

--[[
    Handles command line arguments and executes the corresponding command.
]]
local function run(...)
    local args = {...}
    local command = commands[args[1]]
    if #args == 0 or command == nil then
        usage()
    else
        table.remove(args, 1)
        command.call(unpack(args))
    end
end

-------------------------------------------------------------------------------
-- Initialization                                                            --
-------------------------------------------------------------------------------

run(...)