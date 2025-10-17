# Pick-up-trash-simulator-game
Roblox Trash Simulator System (Spawner, Leaderstats, TrashScript )



'Leaderstats'
-- Leaderstats (ServerScriptService -> Leaderstats)
game.Players.PlayerAdded:Connect(function(player)
	-- leaderstats folder
	local stats = Instance.new("Folder")
	stats.Name = "leaderstats"
	stats.Parent = player

	-- Coins
	local coins = Instance.new("IntValue")
	coins.Name = "Coins"
	coins.Value = 0
	coins.Parent = stats

	-- Upgrade level (starts at 1)
	local upgrade = Instance.new("IntValue")
	upgrade.Name = "UpgradeLevel"
	upgrade.Value = 1
	upgrade.Parent = player
end)


'HandleRemotes'
-- HandleRemotes (ServerScriptService)
local remFolder = game.ReplicatedStorage:WaitForChild("GameRemotes")
local addCoinsRemote = remFolder:WaitForChild("AddCoins")
local upgradeRemote = remFolder:WaitForChild("RequestUpgrade")

-- When client requests to add coins, validate then add on server
addCoinsRemote.OnServerEvent:Connect(function(player, amount)
	-- safety checks
	if typeof(amount) ~= "number" then return end
	if amount < 0 then return end

	local stats = player:FindFirstChild("leaderstats")
	if not stats then return end
	local coins = stats:FindFirstChild("Coins")
	if not coins then return end

	-- add coins
	coins.Value = coins.Value + math.floor(amount)
end)

-- Upgrade handler: cost = 50 * newLevel
upgradeRemote.OnServerEvent:Connect(function(player)
	local stats = player:FindFirstChild("leaderstats")
	if not stats then return end
	local coins = stats:FindFirstChild("Coins")
	local upgrade = player:FindFirstChild("UpgradeLevel")

	if not coins or not upgrade then return end

	local newLevel = upgrade.Value + 1
	local cost = 50 * newLevel

	if coins.Value >= cost then
		coins.Value = coins.Value - cost
		upgrade.Value = newLevel
	end
end)


'TrashSpawner'
local spawnDelay = 5      -- seconds between waves
local spawnCount = 5      -- how many trash per wave
local maxTrash = 50       -- max total trash

-- Trash template in Templates folder
local templatesFolder = workspace:FindFirstChild("Templates")
if not templatesFolder then
	warn("Templates folder missing!")
	return
end

local trashTemplate = templatesFolder:FindFirstChild("TrashTemplate")
if not trashTemplate then
	warn("TrashTemplate missing inside Templates folder!")
	return
end

-- Get random position on a sidewalk
local function getRandomSidewalkPosition()
	local sidewalks = {}
	for _, obj in ipairs(workspace:GetChildren()) do
		if obj:IsA("BasePart") and obj.Name == "Sidewalk" then
			table.insert(sidewalks, obj)
		end
	end
	if #sidewalks == 0 then return nil end

	local sidewalk = sidewalks[math.random(1, #sidewalks)]
	local size = sidewalk.Size
	local pos = sidewalk.Position
	local margin = 2 -- increased margin for spacing

	local x = pos.X + math.random(-size.X/2 + margin, size.X/2 - margin)
	local z = pos.Z + math.random(-size.Z/2 + margin, size.Z/2 - margin)
	local y = pos.Y + size.Y/2 + 1 + math.random() * 0.5 -- slight Y offset

	return Vector3.new(x, y, z)
end

-- Spawn one trash piece
local function spawnTrash()
	local pos = getRandomSidewalkPosition()
	if not pos then return end

	local clone = trashTemplate:Clone()
	clone.Position = pos
	clone.Parent = workspace
	clone.Transparency = 0
	clone.CanCollide = false
end

-- Continuous spawner loop
while true do
	task.wait(spawnDelay)

	-- Count current trash in workspace (ignore template folder)
	local currentTrash = 0
	for _, obj in ipairs(workspace:GetChildren()) do
		if obj:IsA("BasePart") and obj.Name == trashTemplate.Name and obj.Parent ~= templatesFolder then
			currentTrash += 1
		end
	end

	local toSpawn = math.min(spawnCount, maxTrash - currentTrash)
	for i = 1, toSpawn do
		spawnTrash()
	end
end

'trashScript'
local trash = script.Parent
local prompt = trash:WaitForChild("ProximityPrompt")
local collected = false

local function collect(player)
	if collected then return end
	collected = true

	-- Base coins + upgrade multiplier
	local base = 5
	local level = 1
	local upgrade = player:FindFirstChild("UpgradeLevel")
	if upgrade and upgrade:IsA("IntValue") then
		level = upgrade.Value
	end
	local coinsToGive = base * level

	-- Add coins to leaderboard
	local stats = player:FindFirstChild("leaderstats")
	if stats then
		local coins = stats:FindFirstChild("Coins")
		if coins then
			coins.Value += coinsToGive
		end
	end

	-- Optional client popup
	local remFolder = game.ReplicatedStorage:FindFirstChild("GameRemotes")
	if remFolder then
		local addCoinsEvent = remFolder:FindFirstChild("AddCoins")
		if addCoinsEvent then
			addCoinsEvent:FireClient(player, coinsToGive)
		end
	end

	-- Remove trash
	trash.CanCollide = false
	trash.Transparency = 1
	task.wait(0.2)
	trash:Destroy()
end

-- Triggered when player uses ProximityPrompt
prompt.Triggered:Connect(collect)

-- Optional auto-collection if player is very close
game:GetService("RunService").Heartbeat:Connect(function()
	if collected then return end
	for _, player in pairs(game.Players:GetPlayers()) do
		local char = player.Character
		if char and char:FindFirstChild("HumanoidRootPart") then
			if (char.HumanoidRootPart.Position - trash.Position).Magnitude < 3 then
				collect(player)
			end
		end
	end
end)

