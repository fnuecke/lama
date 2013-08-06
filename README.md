[LAMA - Location Aware Movement API][forum post]
====================================

This API provides *persistent* position and facing awareness for [ComputerCraft][] [turtles][]. You can query a turtle's position and facing at any time, and it will not 'desynchronize' if the turtle is unloaded (game quit / chunk unloaded) while it moves. To achieve this, it offers replacements for the default turtle movement functions, i.e. for `turtle.forward()`, `turtle.back()`, `turtle.up()`, `turtle.down()`, `turtle.turnLeft()` and `turtle.turnRight()`, as well as one for refueling, i.e. for `turtle.refuel()`. It also provides a few more high level movement functions to travel multiple blocks, as well as a waypoint system to store coordinates and refer to them by name.

With this API you can write *resumable* programs that make decisions based on the turtle's position. For example, saving the following program as the `startup` file will make the turtle run in circles until it runs out of fuel (precondition: turtle starts at `x = 0, y = 0`). It will not leave its track even if the turtle is forcibly shut down at any point.

```lua
os.loadAPI("lama")
while true do
    local x, y, z = lama.get()
    if x == 0 and y == 0 then
        lama.turn(lama.side.north)
    elseif x == 2 and y == 0 then
        lama.turn(lama.side.east)
    elseif x == 2 and y == 2 then
        lama.turn(lama.side.south)
    elseif x == 0 and y == 2 then
        lama.turn(lama.side.west)
    end
    lama.forward(math.huge) -- Keep trying.
end
```

Requirements
============

This API only works on turtles that have an ID and that use fuel. The ID is required for state persistence, the fuel is used for checking whether the turtle finished a move that was interrupted by a forced shut down (game quit / chunk unloaded).

**Important:** when using this API, you *must not* use any of the original movement related functions, since that would invalidate the API's internal state (coordinates would no longer match), neither the original refuel function, which would also invalidate the state. Always use the functions of this API to move and refuel the turtle. You can use `lama.hijackTurtleAPI()` to have the functions in the turtle API replaced with wrappers for this API's functions. The native functions won't be touched; don't use them either way.

Recommended
-----------

The API will create a startup file while performing a move, to allow completion of multi-try moves after reboot. If there's an original startup file, it will be backed up and after API initialization/move completion will be restored an executed. This obviously introduces additional file i/o overhead for each move. To avoid that, it is highly recommended to use some multi-startup-script solution, such as [Forairan's init-scripts][forairan] startup file or [my startup API][startup]. If present, LAMA will instead create a startup script for the multi-script-environment once, with no additional file i/o overhead when moving.

If you'd like to see compatibility with other multi-startup-script systems (I have not dabbled with custom OSs much, yet), let me know - or even better: implement it yourself and submit a pull request. It's pretty easy to add more, as long as there's a way to detect the used script/system. Should you wish to give it a go, search for `local startupHandlers`, which is the table containing the logic for different startup handlers.

Installation
============

To install the API, first run:

```> pastebin get q45K18dv lama-installer```

which will give you the [installer](installer). After it downloaded, run it:

```> lama-installer```

It will fetch the API and, if so desired the `lama-conf` program as well as an example program (it will interactively ask you). Note that the installer will *not* self-destruct, so you can use it to update your installation.

Alternatively you can install the API via cc-get:

```> cc-get install lama```

This includes only `lama-conf`, but *not* the example program. Note that you will have to use `os.loadAPI("/bin/lama-lib/lama")` in this case (in particular, you'll have to adjust that call in the example program).

In both cases you will get the [minified version](lama-min), which is roughly only 30% the size of the [original version](lama). The minified version was generated using [luamin][]. I just slightly modified it so that all variable names are kept, in particular so that config variables are left in a readable state.

API
===

This list is just meant to provide a quick overview of the available functionality. Please see the [actual code](lama) for full documentation. It is rather well commented. While you're at it, have a look at the `lama.reason` table to see how the movement functions may fail.

Constants
-------

* `lama.version`  
    A string representing the current API version.
* `lama.side`  
    A table of constants used to represent the direction a turtle is facing.
* `lama.reason`  
    A table of constants used to indicate failure reasons when trying to move.

State
-----

* `lama.get()  -> x, y, z, facing`  
    Get the current position and facing of the turtle.
* `lama.getX() -> number`  
    Get the current X position of the turtle.
* `lama.getZ() -> number`  
    Get the current Y position of the turtle.
* `lama.getY() -> number`  
    Get the current Z position of the turtle.
* `lama.getPosition() -> vector`  
    Get the current position of the turtle as a vector.
* `lama.getFacing()   -> lama.side`  
   Get the current facing of the turtle.
* `lama.set(x, y, z, facing) -> x, y, z, facing`  
   Set the current position and facing of the turtle.

Movement
--------

* `lama.forward(tries, aggressive) -> boolean, lama.reason`  
   Replacement for `turtle.forward()`.
* `lama.back(tries) -> boolean, lama.reason`  
    Replacement for `turtle.back()`.
* `lama.up(tries, aggressive) -> boolean, lama.reason`  
    Replacement for `turtle.up()`.
* `lama.down(tries, aggressive) -> boolean, lama.reason`  
    Replacement for `turtle.down()`.
* `lama.moveto(x, y, z, facing, tries, aggressive, longestFirst) -> boolean, lama.reason`  
    Makes the turtle move to the specified coordinates or waypoint. Will continue across reboots.
* `lama.navigate(path, tries, aggressive, longestFirst) -> boolean, lama.reason`  
    Makes the turtle move along the specified path of coordinates and/or waypoints. Will continue across reboots.

The parameters `tries` and `aggressive` for the movement functions do the following: if `tries` is specified, the turtle will try as many times to remove any obstruction it runs into before failing. If `aggressive` is true in addition to that, the turtle will not only try to dig out blocks but also attack entities that stand in its way.

Regarding `lama.navigate()`, note that the facing of all non-terminal path nodes will be ignored to avoid unnecessary turning along the way.

Rotation
--------

* `lama.turnRight() -> boolean`  
    Replacement for `turtle.turnRight()`.
* `lama.turnLeft() -> boolean`  
    Replacement for `turtle.turnRight()`.
* `lama.turnAround() -> boolean`  
    Turns the turtle around.
* `lama.turn(towards) -> boolean`  
    Turns the turtle to face in the specified direction.

Note that all turning functions behave like the original ones in that they return immediately if they fail (they can only fail if the VM's event queue is full). Otherwise they're guaranteed to complete the operation successfully. For `lama.turn()` and `lama.turnAround()` it is possible that one of two needed turn commands has already been issued, when failing. The internal state will represent that however, i.e. `lama.getFacing()` will still be correct.

Refueling
--------

* `lama.refuel(count) -> boolean`  
    Replacement for `turtle.refuel()`.

This has to be called instead of `turtle.refuel()` to ensure the API's state validity, since it uses the fuel level to check if it's in an unexpected state after rebooting. It is is otherwise functionally equivalent to `turtle.refuel()`.

Waypoints
---------

In some cases it may be easier (and more readable) to send your turtles to predefined, named locations, instead of some set of coordinates. That's what waypoints are for. They can be manipulated like so:

* `lama.waypoint.add(name, x, y, z, facing) -> boolean`  
    Adds a new waypoint or updates an existing one with the same name.
* `lama.waypoint.remove(name) -> boolean`  
    Removes a waypoint from the list of known waypoints.
* `lama.waypoint.exists(name) -> boolean`  
    Tests if a waypoint with the specified name exists.
* `lama.waypoint.get(name) -> x, y, z, facing`  
    Returns the coordinates of the specified waypoint.
* `lama.waypoint.iter() -> function`  
    Returns an iterator function over all known waypoints.
* `lama.waypoint.moveto(name, tries, aggressive, longestFirst) -> boolean, lama.reason`  
    This is like `lama.moveto()` except that it takes a waypoint as the target.

Note that waypoints *may* have a facing associated with them, but don't have to. This means that if a waypoint has a facing and the turtle is ordered to move to it, it will rotate into that direction, if it does not the turtle will remain with the orientation in which it arrived at the waypoint.

Utility
-------

* `lama.init()`  
    Can be used to ensure the API has been initialized without any side-effects. If this is the first call and the API has to be initialized because of that, this will block until any pending moves are completed.
* `lama.startupResult() -> boolean, lama.reason`  
    Can be used in resumable programs to check whether a move command that was issued just before the turtle was forcibly shut down completed successfully or not. You will need this if you make decisions based on the result of the movement's success (e.g. stop the program if the turtle fails to move).
* `lama.hijackTurtleAPI(restore)`  
    Replaces the original movement functions in the turtle API with wrappers for this API. Note that the wrappers have the same signature and behavior as the original movement functions. For example, calling the wrapper `turtle.forward(5)` will *not* make the turtle dig through obstructions! This is to make sure existing programs will not behave differently after injecting the wrappers. The original functions can be restored by passing `true` as this functions parameter. **Important:** the changes to the turtle API will be global.

Additional files
================

**lama-conf** can be used to query the current position and facing (`lama-conf get`), as well as set it (`lama-conf set x y z facing`) e.g. for calibrating newly placed turtles. It can also be used to manage waypoints (`lama-conf add name x y z [facing]`, `lama-conf remove name` and `lama-conf list`).

**lama-example** is a small example program, demonstrating how to use the API in a resumable program. It allows ordering the turtle to move along a path of waypoints.

Limitations
===========

If the turtle is moved by some external force (player pickaxing it and placing it somewhere else, RP2 frames, ...) the coordinates will obviously be wrong, because we have no way of tracking such influences (aside from GPS, perhaps - might look into that for roughly validating coordinates in some future version).

All multiblock movement will only perform straight moves, i.e. turtles will never try to evade obstacles, only break through them, if allowed. For now, I feel that pathfinding is beyond the scope of this API, so if you need it you'll have to build it on top of the API. The main reason I don't feel like this belongs in here is because it would quickly get out of hand, since you'd then have to keep track of fuel (because usage cannot be precomputed at the time the order is given) and possibly even map out the region as you go.

Closing remarks
===============

As far as I can tell, this is the first API using the fuel workaround. I'm quite sure this is the only truly working solution out there for properly keeping track of a turtle's position across forced shut downs (aside from GPS, which can be inaccurate). Two common methods I saw I can invalidate right away:

* Saving the state before and after `turtle.forward()` can fail because that call yields, and the turtle may be shut down during that yield, so you may never know whether it was successful or not.
* Saving after `turtle.native.forward()` and after successfully waiting for the result may fail if the turtle is shut down while waiting, since, again you will never know whether the move was successful, because the `turtle_response` event will not be sent after the reboot, see [my thread on the ComputerCraft forums on this issue][event post].

If you know of another method, I'd be intrigued to hear about it, though!

I have tested the robustness of the API using the following method:

- Have the turtle move randomly and continually in a 4x4x4 cube (it'd pick a new target when reaching one).
- Drop sand stacks into the turtle's path.
- Stand in the turtle's way.
- Have some bedrock in the cube.
- Have a mob in the cube.
- Quit the game over and over as well as resetting the turtle manually (Ctrl+R) to interrupt the program at different code locations, hopefully having hit each one at least once.
- Run a big [JAM quarry][jam] (160x160 blocks) and quit+resume the game repeatedly.

Still, this is software, and testing is a tricky business, so it's very possible for some interesting bugs to remain. If you come across any (and are sure it's not your fault) please let me know, thanks!

Changelog
=========

- Version 1.4
  - **Important:** from now on you must use `lama.refuel()` instead of `turtle.refuel()`.
  - Extended state validation to cases when the turtle isn't moving. This is achieved by also replacing `turtle.refuel()` and comparing the command ID we got from that to a newly generated one during initialization, where we'd expect a higher one. There's a slight chance for this to fail if a rollback only went back a couple of ticks. But if the game crashes so hard it can't save anymore it usually involves a lot more rollback.
  - When the API enters an invalid state (command ID indicating rollback or fuel level mismatch) it will now lock down and throw errors whenever any function other than `lama.set()` is called. This is to avoid turtles running amok when they don't know where they are. They'll just stay put until reinitialized by the player.
  - Added parameter to `lama.moveto()`, `lama.navigate()` and `lama.waypoint.moveto()` that can be used to tell the turtle to move along the short (default) or long axes first. For example: when going from (0, 0, 0) to (3, 1, 0) the first axis would be the long axis, the second the short one. This gives more control over which areas turtles will avoid when moving.
  - Went back to blocking when encountering invulnerable entities (sorry). This is to avoid having to use constructs like `repeat until lama.moveto(x, y, z, f, math.huge, true, true --[[ longest axis first ]])` to cover players blocking turtles, and turtles changing paths in such cases.

- Version 1.3
  - **Important:** Changed / introduced folder structure. The API is now in an `apis` folder and the programs (`lama-conf`) in a `programs` folder. This breaks the old installer on pastebin, get the new one if you plan on using it, please. You'll also have to either move the files or adjust your `os.loadAPI` paths. Sorry for the inconvenience.
  - Switched to lazy initialization. This means calling `os.loadAPI("apis/lama")` will no longer block due to the API finishing pending moves. Instead, the first call to any function in the API will trigger initialization and may block. A new function, `lama.init()` has been added specifically for this, but any function will trigger the same logic before doing its own thing. The intention is to allow other startup programs to run before continuing interruped movement.
  - Fixed broken `math.huge` serialization, which means resuming moves using an infinite number of tries didn't work. This was actually `textutils.serialize/unserialize`'s fault, it writes `inf` and then reads that back as `nil`, because it's interpreted as a variable.
  - Fixed waypoints being deleted if some other part of the state was invalid.
  - Changed how indestructible blocks and invulnerable entities are handled. When one is encountered, the move will now fail immediately. This is because moves with an infinite number of tries would otherwise never return when hitting bedrock, for example.
  - Tested some more for robustness using my own [mining program][jam]. It even survived two game crashes, so I feel this can safely be called stable now.

- Version 1.2
  - **Breaking Change:** Added a setting `useMinecraftCoordinates` and it defaults to `true`. *Set it back to `false` if you don't want to adjust your scripts*. I totally forgot to check what kind of coordinate system Minecraft uses internally (i.e. what you see when you enable the debug screen with F3). So now the script uses one that's different. If this setting is `true`, it will make all API functions return and accept coordinates of Minecraft's coordinate system.
  Internally the coordinate system used will always be the custom one, so you can upgrade and switch this setting on and off without breaking any state files.
  - Added setting `startupPriority` which allows setting the desired initialization priority of the API in multi-startup-script environments.
  - Added aliases `lama.goto()` for `lama.moveto()` and `lama.waypoint.goto()` for `lama.waypoint.moveto()`. Use at your own peril, since `goto` is a keyword in Lua 5.2, so in the case ComputerCraft ever updates its Lua implementation these will become unusable.
  - Added code to detect [my startup API][startup] as alternative to Forairan's init-scripts.
  - Tightened up value validation here and there to avoid state corruption/weirdness (mostly: checking whether coordinates are integers).

- Version 1.1
  - Added `lama.moveto()` function which allows issuing multiblock movement commands. The turtle will try to reach the specified coordinate by moving in straight lines; if you want smart navigation/pathfinding you'll have to use some other API on top of this one, or write it yourself, since I feel that's a bit beyond the scope of this API.
  - Added possibility to store waypoints (position plus facing). The related functions can be found in the lama.waypoint namespace.
  - Added `lama.navigate()` which allows issuing chained multiblock movement commands. It expects a table of coordinates or waypoints (can be mixed) and will try to move the turtle along the specified path. This will continue across turtle reboots.
  - Added functionality to `lama-conf` to manipulate waypoints.
  - Added more parameter type checking to avoid getting into an invalid state on bad input.
  - Changed the format of the state file by splitting it into several files, to reduce file i/o during moves. The API will automatically upgrade old state files it finds.
  - Fixed and improved validation of state files.
  - Fixed startup result being lost when the API was loaded a second time.
  - Lots of refactoring to make the code more readable and easier to maintain.
  - Switched to MIT license.

License
=======

This API is licensed under the [MIT License][license], which basically means you can do with it as you please.


[computercraft]: http://www.computercraft.info/
[event post]: http://www.computercraft.info/forums2/index.php?/topic/13855-events-across-world-reload/
[forairan]: http://www.computercraft.info/forums2/index.php?/topic/3018-init-scripts-utility-to-allow-for-multiple-independent-startup-scripts/
[forum post]: http://www.computercraft.info/forums2/index.php?/topic/13919-api-lama-location-aware-movement-api/
[jam]: https://github.com/fnuecke/ccapis/blob/master/programs/jam
[license]: http://opensource.org/licenses/mit-license.php
[luamin]: https://github.com/mathiasbynens/luamin
[startup]: https://github.com/fnuecke/ccapis
[turtles]: http://www.computercraft.info/wiki/Turtle