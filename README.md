LAMA - The Location Aware Movement API for ComputerCraft Turtles
================================================================

This API provides *persistent* position and facing awareness for [ComputerCraft](http://www.computercraft.info/) turtles. You can query a turtle's position and facing at any time, and it will not 'desynchronize' if the turtle is unloaded (game quit / chunk unloaded) while it moves. To achieve this, it offers replacements for the default turtle movement functions, i.e. for `turtle.forward()`, `turtle.back()`, `turtle.up()`, `turtle.down()`, `turtle.turnLeft()` and `turtle.turnRight()`.

This means you can write *resumable* programs that make decisions based on the turtle's position. For example, saving the following program as the `startup` file will make the turtle run in circles until it runs out of fuel (precondition: turtle starts at `x = 0, y = 0`). It will not leave its track even if the turtle is forcibly shut down at any point.

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

License
=======

This API is licensed under the [Creative Commons BY-SA 3.0](http://creativecommons.org/licenses/by-sa/3.0/) license.

Requirements
============

This API only works on turtles that have an ID and that use fuel. The ID is required for state persistence, the fuel is used for checking whether the turtle finished a move that was interrupted by a forced shut down (game quit / chunk unloaded).

**Important:** when using this API, you *must not* use any of the original movement related functions, since that would invalidate the API's internal state (coordinates would no longer match). Always use the functions of this API to move the turtle. You can use `lama.hijackTurtleAPI()` to have the functions in the turtle API replaced with wrappers for this API's functions. The native functions won't be touched; don't use them either way.

Recommended
===========

The API will create a startup file while performing a move, to allow completion of multi-try moves after reboot. If there's an original startup file, it will be backed up and after API initialization/move completion will be restored an executed. This obviously introduces additional file i/o overhead for each move. To avoid that, it is highly recommended to use [Forairan's init-scripts](http://www.computercraft.info/forums2/index.php?/topic/3018-init-scripts-utility-to-allow-for-multiple-independent-startup-scripts/) startup file. If present (or rather: if the `/init-scripts` folder exists) the API will instead create a startup script in the `/init-scripts` folder once, with no additional file i/o overhead when moving.

If you'd like to see compatibility with other multi-startup-script systems (I have not dabbled with custom OSs much, yet), let me know, or even better implement it yourself and submit a pull request. It's pretty easy to add more, as long as there's a way to detect the used script/system.

Limitations
===========

If the turtle is moved by some external force (player pickaxing it and placing it somewhere else, RP2 frames, ...) the coordinates will obviously be wrong, because we have no way of tracking such influences (aside from GPS, perhaps - might look into that for roughly validating coordinates in some future version).

Installation
============

To install the API, use
```pastebin get hCTqrKnP lama```
which will give you the [url="http://pastebin.com/hCTqrKnP"]minified version of the API[/url],
**OR**
{{{pastebin get dZnWFa6c lama}}}
for the [url="http://pastebin.com/dZnWFa6c"]fully commented version of the API[/url].

The minified version is roughly only 25% the size of the commented version, so unless you want to read the code ingame I'd recommend using the minified one.

Alternatively you can install the API via cc-get:
{{{cc-get install lama}}}
This includes lama-conf (see below). Note that you will have to use `os.loadAPI("/bin/lama-lib/lama")` in that case (in particular, you'll have to adjust that call in the lama-move example program).

API
===
Please see the code for full documentation. It is rather well commented. While you're at it, have a look at the `lama.reason` table to see how the movement functions may fail.
    
* `lama.version`
    A string representing the current API version.
* `lama.side`
    A table of constants used to represent the direction a turtle is facing.
* `lama.reason`
    A table of constants used to indicate failure reasons when trying to move.
.
* `lama.forward(tries, aggressive) -> boolean, lama.reason`
   Replacement for turtle.forward()
* `lama.back(tries) -> boolean, lama.reason`
    Replacement for turtle.back()
* `lama.up(tries, aggressive) -> boolean, lama.reason`
    Replacement for turtle.up()
* `lama.down(tries, aggressive) -> boolean, lama.reason`
    Replacement for turtle.down()
* `lama.move(x, y, z, facing) / lama.move(waypoint) -> boolean, lama.reason`
    Makes the turtle move to the specified coordinates or waypoint.

The parameters 'tries' and 'aggressive' for the movement functions do the following: if 'tries' is specified, the turtle will try as many times to remove any obstruction it runs into before failing. If 'aggressive' is true in addition to that, the turtle will not only try to dig out blocks but also attack entities that stand in its way (including the player).

* `lama.turnRight() -> boolean`
    Replacement for turtle.turnRight()
* `lama.turnLeft() -> boolean`
    Replacement for turtle.turnRight()
* `lama.turnAround() -> boolean`
    Turns the turtle around
* `lama.turn(towards) -> boolean`
    Turns the turtle to face in the specified direction

Note that all 'turning' functions behave like the original ones in that they return immediately if they fail (they can only fail if the VM's event queue is full). Otherwise they're guaranteed to complete the operation successfully. For `lama.turn()` and `lama.turnAround()` it is possible that one of two needed turn commands has already been issued, when failing. The internal state will represent that however, i.e. `lama.getFacing()` will still be correct.

* `lama.get()  -> x, y, z, facing`
    Get the current position and facing of the turtle
* `lama.getX() -> number`
    Get the current X position of the turtle
* `lama.getZ() -> number`
    Get the current Y position of the turtle
* `lama.getY() -> number`
    Get the current Z position of the turtle
* `lama.getPosition() -> vector`
    Get the current position of the turtle as a vector
* `lama.getFacing()   -> lama.side`
   Get the current facing of the turtle
* `lama.set(x, y, z, facing)`
   Set the current position and facing of the turtle
.
* `lama.startupResult() -> boolean, lama.reason`
    Can be used in resumable programs to check whether a move command that was issued just before the turtle was forcibly shut down completed successfully or not. You will need this if you make decisions based on the result of the movement's success (e.g. stop the program if the turtle fails to move). See the `lama-move` program below for a simple example.
* `lama.hijackTurtleAPI(restore)`
    Replaces the original movement functions in the turtle API with wrappers for this API. Note that the wrappers have the same signature and behavior as the original movement functions. For example, calling the wrapper `turtle.forward(5)` will *not* make the turtle dig through obstructions! This is to make sure existing programs will not behave differently after injecting the wrappers. The original functions can be restored by passing `true` as this functions parameter. **Important:** the changes to the turtle API will be global.


Waypoints
---------

In some cases it may be easier (and more readable) to send your turtles to predefined, named locations, instead of some set of coordinates. That's what waypoints are for. They can be manipulated like so:

* `waypoint.add(name, x, y, z, facing) -> boolean`
    Adds a new waypoint or updates an existing one with the same name.
* `waypoint.remove(name) -> boolean`
    Removes a waypoint from the list of known waypoints.
* `waypoint.exists(name) -> boolean`
    Tests if a waypoint with the specified name exists.
* `waypoint.get(name) -> x, y, z, facing`
    Returns the coordinates of the specified waypoint.
* `waypoint.iter() -> function`
    Returns an iterator function over all known waypoints.

http://www.computercraft.info/forums2/index.php?/topic/13919-api-lama-location-aware-movement-api/

Closing remarks
===============

As far as I can tell, this is the first API using the fuel workaround. I'm quite sure this is the only truly working solution out there for properly keeping track of a turtle's position across forced shut downs (aside from GPS, which can be inaccurate). Two common methods I saw I can invalidate right away:

* Saving the state before and after `turtle.forward()` can fail because that call yields, and the turtle may be shut down during that yield, so you may never know whether it was successful or not.
* Saving after `turtle.native.forward()` and after successfully waiting for the result may fail if the turtle is shut down while waiting, since, again you will never know whether the move was successful, because the `turtle_response` event will not be sent after the reboot, see [my thread on the ComputerCraft forums on this issue](http://www.computercraft.info/forums2/index.php?/topic/13855-events-across-world-reload/).

If you know of another method, I'd be intrigued to hear about it, though!

I have tested the robustness of the API by having the example program run continually (when done it'd pick a new random location inside a 4x4x4 cube) and quitting the game over and over (to interrupt it at different code locations, hopefully having hit each one at least once), dropping sand stacks into the turtle's path and standing in the turtle's way. Still, this is software, and testing is a tricky business, so it's very possible for some interesting bugs to remain. If you come across any (and are sure it's not your fault) please let me know, thanks!