---
ns: GRAPHICS
aliases: ["_SET_BLACKOUT"]
---
## SET_ARTIFICIAL_LIGHTS_STATE

```c
// 0x1268615ACE24D504 0xAA2A0EAF
void SET_ARTIFICIAL_LIGHTS_STATE(BOOL state);
```

Does not affect weapons, particles, fire/explosions, flashlights or the sun.

When set to true, all emissive textures (including ped components that have light effects), street lights, building lights, vehicle lights, etc will all be turned off.

Used in Humane Labs Heist for EMP.

## Parameters
* **state**: True turns off all artificial light sources in the map: buildings, street lights, car lights, etc. False turns them back on.

## Examples
```lua
-- Disable all lights in the map.
SetArtificialLightsState(true)

-- Enable all lights in the map.
SetArtificialLightsState(false)
```

```cs
// Disable all lights in the map.
SetArtificialLightsState(true);

// Enable all lights in the map.
SetArtificialLightsState(false);
```
local fireObjects = {
	"prop_beach_fire",
	-- "prop_hobo_stove_01", -- Doesn't work
	"prop_ld_rubble_01",
}

local inRange = false

closeObjects = {}
closeVehicles = {}

currentCount = 0
count = 0

RegisterNetEvent("SetLights")
AddEventHandler("SetLights", function(daylight)
	day = daylight
end)

local entityEnumerator = {
  __gc = function(enum)

    if enum.destructor and enum.handle then
	  enum.destructor(enum.handle)
	  
    end
    enum.destructor = nil
    enum.handle = nil
  end
}

local function EnumerateEntities(initFunc, moveFunc, disposeFunc)
  return coroutine.wrap(function()
    local iter, id = initFunc()
    if not id or id == 0 then
      disposeFunc(iter)
      return
    end
    
    local enum = {handle = iter, destructor = disposeFunc}
    setmetatable(enum, entityEnumerator)
    
    local next = true

    repeat
      coroutine.yield(id)
      next, id = moveFunc(iter)
    until not next

    enum.destructor, enum.handle = nil, nil
    disposeFunc(iter)
  end)
end

function EnumerateObjects()
	return EnumerateEntities(FindFirstObject, FindNextObject, EndFindObject)
end

function EnumerateVehicles()
	return EnumerateEntities(FindFirstVehicle, FindNextVehicle, EndFindVehicle)
end

-- Fire area lights
Citizen.CreateThread(function()
	while true do
		playerX, playerY, playerZ = table.unpack(GetEntityCoords(PlayerPedId(), true))
		for object in EnumerateObjects() do
			playerX, playerY, playerZ = table.unpack(GetEntityCoords(GetPlayerPed(-1), true))
			objectX, objectY, objectZ = table.unpack(GetEntityCoords(object, true))

			if day == false then
				for i, fireObject in ipairs(fireObjects) do
					if GetEntityModel(object) == GetHashKey(fireObject) then
						if(Vdist(playerX, playerY, playerZ, objectX, objectY, objectZ) <= 100.0)then
							table.insert(closeObjects, object)
						end

					end
				end
			end
		end

		if day == false then
			for i, object in pairs(closeObjects) do
				playerX, playerY, playerZ = table.unpack(GetEntityCoords(GetPlayerPed(-1), true))
				objectX, objectY, objectZ = table.unpack(GetEntityCoords(object, true))

				if DoesEntityExist(object) == false then
					table.remove(closeObjects, i)
				end

				if objectX < playerX - 100 or objectX > playerX + 100 or objectY < playerY - 100 or objectY > playerY + 100 then
					table.remove(closeObjects, i)
				end

			end
		end

		Citizen.Wait(500)
		count = #closeObjects
		for i=0, count do
			closeObjects[i] = nil
		end

	end
end)

Citizen.CreateThread(function()
	while true do
		Citizen.Wait(0)
		if day == false then
			for i, object in ipairs(closeObjects) do
			    playerX, playerY, playerZ = table.unpack(GetEntityCoords(PlayerPedId(), true))
				objectX, objectY, objectZ = table.unpack(GetEntityCoords(object, true))

				if(Vdist(playerX, playerY, playerZ, objectX, objectY, objectZ) <= 250.0)then
					DrawLightWithRangeAndShadow(objectX, objectY, objectZ, 255, 209, 100, 10.0, 0.1, 10.0)
				end

			end
		end
	end
end)



-- Vehicle headlights

Citizen.CreateThread(function()
	while true do
		playerX, playerY, playerZ = table.unpack(GetEntityCoords(PlayerPedId(), true))

		for vehicle in EnumerateVehicles() do
			playerX, playerY, playerZ = table.unpack(GetEntityCoords(GetPlayerPed(-1), true))
			vehX, vehY, vehZ = table.unpack(GetEntityCoords(vehicle, true))

			if(Vdist(playerX, playerY, playerZ, vehX, vehY, vehZ) <= 250.0)then
				table.insert(closeVehicles, vehicle)
			end
		end

		for i, vehicle in pairs(closeVehicles) do

			playerX, playerY, playerZ = table.unpack(GetEntityCoords(GetPlayerPed(-1), true))
			vehX, vehY, vehZ = table.unpack(GetEntityCoords(vehicle, true))

			if DoesEntityExist(vehicle) == false then
				table.remove(closeVehicles, i)
			end

			if vehX < playerX - 250 or vehX > playerX + 250 or vehY < playerY - 250 or vehY > playerY + 250 then
				table.remove(closeVehicles, i)
			end
		end

		Citizen.Wait(500)
		count = #closeVehicles
		for i=0, count do
			closeVehicles[i] = nil
		end

	end
end)

Citizen.CreateThread(function()
	while true do
		Citizen.Wait(1)

		for i, vehicle in ipairs(closeVehicles) do
			carX, carY, carZ = table.unpack(GetEntityCoords(vehicle, false))
			dirX, dirY, dirZ = table.unpack(GetEntityForwardVector(vehicle))	

			if GetIsVehicleEngineRunning(vehicle) then
				-- FreezeEntityPosition(vehicle, false)
				DrawSpotLight(carX, carY, carZ, dirX, dirY, dirZ, 255, 255, 255, 50.0, 0.1, 0.9, 25.0, 0.95)
			else
				-- FreezeEntityPosition(vehicle, true)
			end

		end
	end
end)



-- Area lights
Citizen.CreateThread(function()
	while true do

		Citizen.Wait(1)
		x, y, z = table.unpack(GetEntityCoords(PlayerPedId(), false))
		dirX, dirY, dirZ = table.unpack(GetEntityForwardVector(PlayerPedId()))	

		for k,v in pairs(lightAreas) do
			DrawLightWithRangeAndShadow(v.x, v.y, v.z, 255, 255, 255, v.radius, 0.025, 10.0)
		end

	end
end) 
local blackout = true

-- Switches blackout on or off
Citizen.CreateThread(function()
	while true do
		Citizen.Wait(1)
		if blackout == true then
			SetBlackout(true)
		else
			SetBlackout(false)
		end
	end
end)

function DisplayHelpText(str)
    SetTextComponentFormat("STRING")
    AddTextComponentString(str)
    DisplayHelpTextFromStringLabel(0, 0, 1, -1)
end
resource_manifest_version "44febabe-d386-4d18-afbe-5e627f4af937"



client_scripts {
	"interiors.lua",

	"list.lua",
	"lights.lua",
}
lightAreas = {
	{x=178.87762451172, y=2236.8432617188, z=89.945236206055, radius=10.0}, 
	{x=1598.0017089844, y=3581.4372558594, z=38.770069122314, radius=10.0}, 
	{x=1591.8985595703, y=3591.4787597656, z=38.875568389893, radius=10.0}, 
	{x=1578.3811035156, y=3603.5209960938, z=38.731395721436, radius=10.0}, 
	{x=1578.9027099609, y=3615.7609863281, z=38.775196075439, radius=10.0}, 
	{x=1561.4851074219, y=3523.3645019531, z=35.776844024658, radius=10.0}, 
	{x=1549.8492431641, y=3517.3044433594, z=35.993041992188, radius=10.0}, 
	{x=1730.7401123047, y=3309.4423828125, z=41.223476409912, radius=10.0}, 
	{x=1737.8201904297, y=3325.849609375, z=41.223476409912, radius=10.0}, 
	{x=2334.8896484375, y=3138.6457519531, z=48.186500549316, radius=10.0}, 
	{x=2341.720703125, y=3128.5322265625, z=48.208751678467, radius=10.0}, 
	{x=2354.7048339844, y=3132.658203125, z=48.208751678467, radius=10.0}, 
	{x=2350.6259765625, y=3118.2622070313, z=48.20874786377, radius=10.0}, 
	{x=2392.2983398438, y=3122.6999511719, z=48.153160095215, radius=10.0}, 
	{x=2429.6904296875, y=3112.8576660156, z=48.167869567871, radius=10.0}, 
	{x=2422.6242675781, y=3134.7534179688, z=48.146949768066, radius=10.0},
	{x=1859.8074951172, y=3852.0942382813, z=33.013431549072, radius=10.0}, 
	{x=1854.7263183594, y=3834.4167480469, z=32.608955383301, radius=10.0}, 
	{x=1862.6306152344, y=3841.1945800781, z=32.611175537109, radius=10.0},
	{x=-2051.470703125, y=3237.1552734375, z=31.501232147217, radius=10.0},
	{x=3137.16, y=2178.87, z=4.24, radius=25.0},
	{x=3137.16, y=2188.73, z=4.24, radius=25.0},
	{x=3137.16, y=2198.73, z=4.24, radius=25.0},
	{x=3128.47, y=2221.84, z=4.19, radius=40.0},
	{x=3146.19, y=2181.04, z=4.19, radius=5.0},
	{x=-3946.61, y=5589.11, z=16.61, radius=15.0},
	{x=-3925.4, y=5589.11, z=16.61, radius=15.0},
	{x=-3966.34, y=5589.11, z=16.61, radius=15.0},
	{x=-4018.94, y=5586.32, z=14.14, radius=5.0},
	{x=-4021.05, y=5593.49, z=14.22, radius=15.0},
	{x=-411.35, y=1229.15, z=330.74, radius=15.0},
	{x=974.44, y=-99.43, z=76.28, radius=15.0},
	{x=981.23, y=-97.81, z=76.42, radius=5.0},
	{x=978.54, y=-92.9, z=76.36, radius=5.0},
	{x=988.11, y=-99.37, z=75.92, radius=5.0},
	{x=980.07, y=-101.88, z=76.23, radius=5.0},
	{x=986.72, y=-95.18, z=76.86, radius=5.0},
	{x=983.18, y=-98.42, z=76.55, radius=5.0},
	{x=-3139.71, y=-1562.78, z=19.02, radius=10.0},
	{x=-3133.87, y=-1562.66, z=19.02, radius=10.0},
	{x=-3130.9, y=-1608.05, z=19.02, radius=10.0},
	{x=-3141.46, y=-1610.59, z=19.58, radius=10.0},
	{x=-3141.41, y=-1604.74, z=19.58, radius=10.0},
	{x=114.22, y=-1286.69, z=31.09, radius=25.0},
	{x=2578.5, y=6158.99, z=167.08, radius=10.0},
	{x=2557.54, y=6175.7, z=165.62, radius=15.0},
	{x=2558.38, y=6181.31, z=166.42, radius=5.0},
	{x=110.01, y=6627.11, z=34.52, radius=15.0},
	{x=103.29, y=6627.11, z=34.52, radius=15.0},
	{x=36.13, y=-2698.85, z=14.5, radius=15.0},
	{x=28.36, y=-2698.48, z=14.5, radius=15.0},
	{x=-112.88, y=-2662.01, z=6.03, radius=15.0},
	{x=-130.65, y=-2654.77, z=6.03, radius=15.0},
	{x=32.58, y=-2739.41, z=30.01, radius=25.0},
	{x=5853.71, y=154.69, z=357.74, radius=20.0}, 
	{x=5867.03, y=151.13, z=357.74, radius=5.0},
	{x=5851.29, y=154.02, z=353.86, radius=20.0},
	{x=5927.39, y=127.90, z=352.24, radius=100.0},
	{x=5951.00, y=150.39, z=346.02, radius=5.0},
	{x=5837.11, y=115.82, z=351.95, radius=20.0}, 
	{x=5840.55, y=84.40, z=350.93, radius=20.0}, 
	{x=5843.67, y=34.15, z=350.33, radius=20.0}, 
	{x=5824.57, y=126.17, z=351.99, radius=20.0}, 
	{x=5834.78, y=145.32, z=352.24, radius=20.0},
	{x=2939.13, y=2795.5, z=43.32, radius=15.0},
	{x=2970.95, y=2769.95, z=49.15, radius=10.0},
	{x=2953.31, y=2782.07, z=43.6, radius=15.0},
	{x=2933.94, y=2804.47, z=43.6, radius=15.0},
	{x=2952.73, y=2808.81, z=43.6, radius=15.0},
	{x=-2059.26, y=5624.84, z=5.48, radius=10.0},
	{x=-2070.72, y=5617.74, z=5.48, radius=10.0},
	{x=-2069.92, y=5624.34, z=5.48, radius=10.0},
	{x=-2075.69, y=5616.05, z=9.67, radius=10.0},
	{x=-2076.24, y=5626.96, z=9.67, radius=10.0},
	{x=929.51, y=-3182.62, z=27.11, radius=100.0},
	{x=913.57, y=-3185.97, z=12.19, radius=15.0},
	{x=908.53, y=-3185.59, z=12.32, radius=15.0},
	{x=908.19, y=-3180.21, z=12.65, radius=15.0},
	{x=912.04, y=-3185.25, z=8.31, radius=15.0},
	{x=924.97, y=-3184.88, z=12.08, radius=20.0},
	{x=940.32, y=-3185.62, z=12.63, radius=25.0},
	{x=953.87, y=-3155.52, z=10.44, radius=10.0},
	{x=944.18, y=-3155.52, z=10.44, radius=10.0},
	{x=933.06, y=-3155.52, z=10.44, radius=10.0},
	{x=921.41, y=-3155.52, z=10.44, radius=10.0},
	{x=913.61, y=-3155.52, z=10.44, radius=10.0},
	{x=903.86, y=-3155.52, z=10.44, radius=10.0},
	{x=-774.13, y=319.75, z=88.93, radius=13.0},
}
