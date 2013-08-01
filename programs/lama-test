os.loadAPI("apis/lama")

-------------------------------------------------------------------------------
-- General constants.
-------------------------------------------------------------------------------

assert(lama)
assert(type(lama.version) == "string")
assert(type(lama.side) == "table")
assert(type(lama.reason) == "table")

-- lama.side entry presence and error on invalid indexing
assert(lama.side.forward == 0)
assert(lama.side.right == 1)
assert(lama.side.back == 2)
assert(lama.side.left == 3)
assert(lama.side.north == 0)
assert(lama.side.east == 1)
assert(lama.side.south == 2)
assert(lama.side.west == 3)
assert(lama.side.front == 0)
assert(lama.side["0"] == 0)
assert(lama.side["1"] == 1)
assert(lama.side["2"] == 2)
assert(lama.side["3"] == 3)
do
	local result = pcall(
		function()
			local x = lama.side.test_invalid_entry
			print(x)
		end)
	assert(result == false)
end

-- lama.reason entry presence and error on invalid indexing
assert(lama.reason.unknown)
assert(lama.reason.queue_full)
assert(lama.reason.coroutine)
assert(lama.reason.fuel)
assert(lama.reason.block)
assert(lama.reason.turtle)
assert(lama.reason.unbreakable_block)
assert(lama.reason.entity)
assert(lama.reason.invulnerable_entity)
do
	local result = pcall(
		function()
			local x = lama.reason.test_invalid_entry
		end)
	assert(result == false)
end

-------------------------------------------------------------------------------
-- Accessors.
-------------------------------------------------------------------------------

local x, y, z, f = lama.get()
assert(type(x) == "number")
assert(type(y) == "number")
assert(type(z) == "number")
assert(type(f) == "number")

x = lama.getX()
assert(type(x) == "number")

y = lama.getY()
assert(type(y) == "number")

z = lama.getZ()
assert(type(z) == "number")

do
	local v = lama.getPosition()
	assert(type(v) == "table")
	assert(type(v.x) == "number")
	assert(type(v.y) == "number")
	assert(type(v.z) == "number")
	assert(v.x == x)
	assert(v.y == y)
	assert(v.z == z)
end

do
	local f2 = lama.getFacing()
	assert(type(f2) == "number")
	assert(f == f2)
end

do
	local x2, y2, z2, f2 = lama.set(x, y, z, f)
	assert(type(x) == "number")
	assert(type(y) == "number")
	assert(type(z) == "number")
	assert(type(f) == "number")
	assert(x == x2)
	assert(y == y2)
	assert(z == z2)
	assert(f == f2)
end

-------------------------------------------------------------------------------
-- Waypoints
-------------------------------------------------------------------------------

do
	local home, somewhere = "lama-test-home", "lama-test-somewhere"
	lama.waypoint.add(home)
	assert(lama.waypoint.exists(home))
	local wx, wy, wz, wf = lama.waypoint.get(home)
	assert(wx == x)
	assert(wy == y)
	assert(wz == z)
	assert(wf == f)

	assert(lama.waypoint.add(home))

	lama.waypoint.add(somewhere, 1, 2, 3, lama.side.west)
	assert(lama.waypoint.exists(somewhere))
	local wx2, wy2, wz2, wf2 = lama.waypoint.get(somewhere)
	assert(wx2 == 1, tostring(wx2) .. " != 1")
	assert(wy2 == 2, tostring(wy2) .. " != 2")
	assert(wz2 == 3, tostring(wz2) .. " != 3")
	assert(wf2 == lama.side.west)

	local expected = {[home] = true, [somewhere] = true}
	for name, x, y, z, facing in lama.waypoint.iter() do
		expected[name] = nil
	end
	local count = 0
	for _,_ in pairs(expected) do count = count + 1 end
	assert(count == 0)

	assert(lama.waypoint.remove(somewhere))
	assert(lama.waypoint.exists(somewhere) == false)

	assert(lama.waypoint.moveto(home))
	assert(lama.navigate({home, home, home}))

	assert(lama.waypoint.remove(home))
end

-------------------------------------------------------------------------------
-- Simple movement
-------------------------------------------------------------------------------

if turtle.getFuelLevel() > 1 then
	local result, reason = lama.forward()
	local x2, y2, z2, f2 = lama.get()
	if result then
		if f == lama.side.north then
			assert(x2 == x + 1)
			assert(y2 == y)
			assert(z2 == z)
			assert(f2 == f)
		elseif f == lama.side.east then
			assert(x2 == x)
			assert(y2 == y + 1)
			assert(z2 == z)
			assert(f2 == f)
		elseif f == lama.side.south then
			assert(x2 == x - 1)
			assert(y2 == y)
			assert(z2 == z)
			assert(f2 == f)
		elseif f == lama.side.west then
			assert(x2 == x)
			assert(y2 == y - 1)
			assert(z2 == z)
			assert(f2 == f)
		end
	else
		assert(x2 == x)
		assert(y2 == y)
		assert(z2 == z)
		assert(f2 == f)
	end
	result, reason = lama.back()
	local x3, y3, z3, f3 = lama.get()
	if result then
		if f == lama.side.north then
			assert(x3 == x2 - 1)
			assert(y3 == y2)
			assert(z3 == z2)
			assert(f3 == f2)
		elseif f == lama.side.east then
			assert(x3 == x2)
			assert(y3 == y2 - 1)
			assert(z3 == z2)
			assert(f3 == f2)
		elseif f == lama.side.south then
			assert(x3 == x2 + 1)
			assert(y3 == y2)
			assert(z3 == z2)
			assert(f3 == f2)
		elseif f == lama.side.west then
			assert(x3 == x2)
			assert(y3 == y2 + 1)
			assert(z3 == z2)
			assert(f3 == f2)
		end
	else
		assert(x2 == x)
		assert(y2 == y)
		assert(z2 == z)
		assert(f2 == f)
	end
else
	print("Not enough fuel to test movement functions.")
end

if turtle.getFuelLevel() > 1 then
	local result, reason = lama.up()
	local x2, y2, z2, f2 = lama.get()
	if result then
		assert(x2 == x)
		assert(y2 == y)
		assert(z2 == z + 1)
		assert(f2 == f)
	else
		assert(x2 == x)
		assert(y2 == y)
		assert(z2 == z)
		assert(f2 == f)
	end
	result, reason = lama.down()
	local x3, y3, z3, f3 = lama.get()
	if result then
		assert(x3 == x2)
		assert(y3 == y2)
		assert(z3 == z2 - 1)
		assert(f3 == f2)
	else
		assert(x3 == x2)
		assert(y3 == y2)
		assert(z3 == z2)
		assert(f3 == f2)
	end
else
	print("Not enough fuel to test movement functions.")
end

do
	local result = lama.turnRight()
	local f2 = lama.getFacing()
	if result then
		assert(f2 == (f + 1) % 4)
	else
		assert(f2 == f)
	end
	result = lama.turnLeft()
	local f3 = lama.getFacing()
	if result then
		assert(f3 == (f2 - 1) % 4)
	else
		assert(f3 == f2)
	end
	result = lama.turnAround()
	local f4 = lama.getFacing()
	if result then
		assert(f4 == (f3 + 2) % 4)
	end
	reason = lama.turn(f)
	local f5 = lama.getFacing()
	if reason then
		assert(f5 == f)
	end
end

-------------------------------------------------------------------------------
-- Complex movement
-------------------------------------------------------------------------------

assert(lama.moveto(x, y, z))
assert(lama.moveto(x, y, z, f))

do
	local p = {x = x, y = y, z = z, facing = f}
	assert(lama.navigate({p, p, p}))
end

-- TODO make lama.moveto and lama.navigate actually move the turtle

-- TODO interrupt movement somehow? perhaps via coroutine and unloading then
--      reloading the API to trigger "resume" logic?

print("All tests passed!")