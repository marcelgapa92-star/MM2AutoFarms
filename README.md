--[[
	MM2 Autofarm v5 — HALLOWEEN
	Based on stable v3 pathing + under-map default
	No forced Physics state · smoother tween · Halloween GUI
	RightShift = GUI
]]

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UIS = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local WS = game:GetService("Workspace")
local VirtualUser = game:GetService("VirtualUser")
local StarterGui = game:GetService("StarterGui")
local LP = Players.LocalPlayer

local S = {
	Enabled = false,
	UnderMap = true,
	UnderY = -18,
	Speed = 28,
	SoftTP = false,
	TPDelay = 0.12,
	Noclip = true,
	TouchHeight = 2.0,
	AntiAFK = true,
	CoinESP = true,
	CandyESP = true,
	AutoResetFull = true,
	FullAmount = 50,
	Humanize = true,
}

local collectedSession = 0
local lastCollectPos = nil
local busy = false
local nParts = {}

local function notify(t)
	pcall(function()
		StarterGui:SetCore("SendNotification", {
			Title = "🎃 MM2 Farm",
			Text = t,
			Duration = 2.4,
		})
	end)
end

local function getRoot()
	local c = LP.Character
	return c and c:FindFirstChild("HumanoidRootPart")
end

local function getHum()
	local c = LP.Character
	return c and c:FindFirstChildOfClass("Humanoid")
end

local function refNoclip()
	table.clear(nParts)
	local c = LP.Character
	if not c then return end
	for _, p in ipairs(c:GetChildren()) do
		if p:IsA("BasePart") then
			table.insert(nParts, p)
		end
	end
end

local function applyNoclip()
	for _, p in ipairs(nParts) do
		if p.Parent then
			p.CanCollide = false
		end
	end
end

LP.CharacterAdded:Connect(function()
	task.wait(0.5)
	refNoclip()
	lastCollectPos = nil
	busy = false
end)
task.defer(refNoclip)

local function getCoinContainer()
	for _, obj in ipairs(WS:GetChildren()) do
		if obj:GetAttribute("MapID") and obj:FindFirstChild("CoinContainer") then
			return obj.CoinContainer
		end
	end
	return WS:FindFirstChild("CoinContainer", true)
end

local function isCollected(coin)
	if not coin or not coin.Parent then return true end
	if coin:GetAttribute("Collected") then return true end
	local visual = coin:FindFirstChild("CoinVisual")
	if visual and visual:GetAttribute("Collected") then return true end
	if coin.Name == "CollectedCoin" then return true end
	return false
end

local function coinKind(coin)
	local ct = coin:FindFirstChild("CoinType")
	if ct and typeof(ct.Value) == "string" then
		return string.lower(ct.Value)
	end
	local n = string.lower(coin.Name)
	if n:find("candy") or n:find("treat") or n:find("halloween") then return "candy" end
	if n:find("heart") then return "heart" end
	return "coin"
end

local function coinPosition(coin)
	if coin:IsA("BasePart") then return coin.Position end
	if coin:IsA("Model") then
		if coin.PrimaryPart then return coin.PrimaryPart.Position end
		local p = coin:FindFirstChildWhichIsA("BasePart", true)
		if p then return p.Position end
		return coin:GetPivot().Position
	end
	local p = coin:FindFirstChildWhichIsA("BasePart", true)
	return p and p.Position or nil
end

local function listCoins()
	local out = {}
	local cc = getCoinContainer()
	if cc then
		for _, coin in ipairs(cc:GetChildren()) do
			if not isCollected(coin) and coinPosition(coin) then
				table.insert(out, coin)
			end
		end
		return out
	end
	for _, obj in ipairs(WS:GetDescendants()) do
		if obj:IsA("TouchTransmitter") and obj.Parent and obj.Parent:IsA("BasePart") then
			local p = obj.Parent
			local n = string.lower(p.Name)
			local pn = p.Parent and string.lower(p.Parent.Name) or ""
			if n == "coin" or n == "base" or pn:find("coin") or n:find("candy") then
				if not isCollected(p) then table.insert(out, p) end
			end
		end
	end
	return out
end

local function nearestCoin(fromPos)
	local best, bestD, bestPos = nil, 1e9, nil
	for _, coin in ipairs(listCoins()) do
		local pos = coinPosition(coin)
		if pos then
			local dx, dz = pos.X - fromPos.X, pos.Z - fromPos.Z
			local d = math.sqrt(dx * dx + dz * dz)
			if d < bestD then bestD, best, bestPos = d, coin, pos end
		end
	end
	return best, bestD, bestPos
end

-- v3-style soft tween (stable)
local function softMove(targetPos)
	local r = getRoot()
	if not r or busy then return end
	busy = true
	local h = getHum()
	if h then
		h.PlatformStand = true
		h.WalkSpeed = 0
	end
	if S.Noclip then applyNoclip() end

	local dist = (targetPos - r.Position).Magnitude
	local speed = math.clamp(S.Speed, 12, 55)
	local dur = math.clamp(dist / speed, 0.12, 2.5)
	if S.Humanize then
		dur = dur + math.random(0, 6) / 100
	end

	pcall(function()
		local tw = TweenService:Create(r, TweenInfo.new(dur, Enum.EasingStyle.Linear), {
			CFrame = CFrame.new(targetPos),
		})
		tw:Play()
		tw.Completed:Wait()
	end)

	if r.Parent then
		r.AssemblyLinearVelocity = Vector3.zero
		r.AssemblyAngularVelocity = Vector3.zero
	end
	busy = false
end

local function markCollect(pos)
	if lastCollectPos then
		if (pos - lastCollectPos).Magnitude > 3 then
			collectedSession += 1
			lastCollectPos = pos
		end
	else
		collectedSession += 1
		lastCollectPos = pos
	end
end

local function doReset()
	notify("Full " .. S.FullAmount .. " — resetting")
	collectedSession = 0
	lastCollectPos = nil
	busy = false
	pcall(function()
		local h = getHum()
		if h then h.Health = 0 end
	end)
	task.wait(2)
	refNoclip()
end

local function farmStep()
	if not S.Enabled or busy then return end
	local r = getRoot()
	local h = getHum()
	if not r or not h or h.Health <= 0 then return end
	if S.Noclip then applyNoclip() end

	local coin, dist, pos = nearestCoin(r.Position)
	if not coin or not pos then return end

	local under = S.UnderMap
	local travel = under and Vector3.new(pos.X, pos.Y + S.UnderY, pos.Z)
		or Vector3.new(pos.X, pos.Y + S.TouchHeight, pos.Z)
	local flat = Vector3.new(pos.X - r.Position.X, 0, pos.Z - r.Position.Z).Magnitude

	if S.SoftTP then
		local delay = math.clamp(S.TPDelay, 0.06, 0.5)
		if S.Humanize then delay = delay + math.random(0, 4) / 100 end
		if under then
			r.CFrame = CFrame.new(travel)
			r.AssemblyLinearVelocity = Vector3.zero
			task.wait(delay * 0.35)
			r.CFrame = CFrame.new(pos + Vector3.new(0, S.TouchHeight, 0))
			task.wait(delay * 0.3)
			markCollect(pos)
			r.CFrame = CFrame.new(travel)
		else
			r.CFrame = CFrame.new(pos + Vector3.new(0, S.TouchHeight, 0))
			task.wait(delay)
			markCollect(pos)
		end
		r.AssemblyLinearVelocity = Vector3.zero
		return
	end

	-- SAFE tween path (v3 proven)
	if under then
		if flat > 4.5 then
			softMove(travel)
		else
			softMove(Vector3.new(pos.X, pos.Y + S.TouchHeight, pos.Z))
			markCollect(pos)
			task.wait(0.05)
			softMove(travel)
		end
	else
		softMove(Vector3.new(pos.X, pos.Y + S.TouchHeight, pos.Z))
		markCollect(pos)
	end

	if S.AutoResetFull and collectedSession >= S.FullAmount then
		doReset()
	end
end

-- ESP
local espFolder
local function ensureEspFolder()
	if espFolder and espFolder.Parent then return espFolder end
	local pg = LP:FindFirstChild("PlayerGui") or LP:WaitForChild("PlayerGui")
	local old = pg:FindFirstChild("MM2FarmESP")
	if old then old:Destroy() end
	espFolder = Instance.new("Folder")
	espFolder.Name = "MM2FarmESP"
	espFolder.Parent = pg
	return espFolder
end

local function clearEsp()
	if espFolder then espFolder:ClearAllChildren() end
end

local function updateEsp()
	if not (S.CoinESP or S.CandyESP) then clearEsp() return end
	ensureEspFolder()
	local seen = {}
	for _, coin in ipairs(listCoins()) do
		local kind = coinKind(coin)
		local show = (kind == "candy" and S.CandyESP) or (kind ~= "candy" and S.CoinESP)
		if show then
			local pos = coinPosition(coin)
			if pos then
				local id = tostring(coin)
				seen[id] = true
				local bb = espFolder:FindFirstChild(id)
				if not bb then
					bb = Instance.new("BillboardGui")
					bb.Name = id
					bb.Size = UDim2.fromOffset(40, 40)
					bb.AlwaysOnTop = true
					bb.StudsOffset = Vector3.new(0, 1.5, 0)
					bb.Parent = espFolder
					local fr = Instance.new("Frame")
					fr.Name = "D"
					fr.Size = UDim2.fromScale(1, 1)
					fr.BackgroundTransparency = 0.3
					fr.BorderSizePixel = 0
					fr.Parent = bb
					Instance.new("UICorner", fr).CornerRadius = UDim.new(1, 0)
				end
				local part = coin:IsA("BasePart") and coin or coin:FindFirstChildWhichIsA("BasePart", true)
				if part then bb.Adornee = part end
				local fr = bb:FindFirstChild("D")
				if fr then
					fr.BackgroundColor3 = (kind == "candy") and Color3.fromRGB(255, 120, 40) or Color3.fromRGB(255, 220, 60)
				end
			end
		end
	end
	for _, ch in ipairs(espFolder:GetChildren()) do
		if not seen[ch.Name] then ch:Destroy() end
	end
end

-- Halloween GUI
pcall(function()
	local o = game:GetService("CoreGui"):FindFirstChild("MM2AutoFarm")
	if o then o:Destroy() end
end)
pcall(function()
	local o = LP.PlayerGui:FindFirstChild("MM2AutoFarm")
	if o then o:Destroy() end
end)

local parent = LP.PlayerGui
pcall(function()
	local t = Instance.new("ScreenGui")
	t.Parent = game:GetService("CoreGui")
	t:Destroy()
	parent = game:GetService("CoreGui")
end)

local gui = Instance.new("ScreenGui")
gui.Name = "MM2AutoFarm"
gui.ResetOnSpawn = false
gui.IgnoreGuiInset = true
gui.DisplayOrder = 260
gui.Parent = parent

local Main = Instance.new("Frame")
Main.Size = UDim2.fromOffset(318, 440)
Main.Position = UDim2.new(0.5, -159, 0.5, -220)
Main.BackgroundColor3 = Color3.fromRGB(14, 8, 12)
Main.BorderSizePixel = 0
Main.Active = true
Main.Parent = gui
Instance.new("UICorner", Main).CornerRadius = UDim.new(0, 12)
local stroke = Instance.new("UIStroke")
stroke.Color = Color3.fromRGB(255, 110, 25)
stroke.Thickness = 1.6
stroke.Parent = Main

local Top = Instance.new("TextButton")
Top.Size = UDim2.new(1, 0, 0, 40)
Top.BackgroundColor3 = Color3.fromRGB(42, 14, 16)
Top.Text = ""
Top.AutoButtonColor = false
Top.Parent = Main
Instance.new("UICorner", Top).CornerRadius = UDim.new(0, 12)
local tf = Instance.new("Frame")
tf.Size = UDim2.new(1, 0, 0, 14)
tf.Position = UDim2.new(0, 0, 1, -14)
tf.BackgroundColor3 = Color3.fromRGB(42, 14, 16)
tf.BorderSizePixel = 0
tf.Parent = Top

local Title = Instance.new("TextLabel")
Title.BackgroundTransparency = 1
Title.Position = UDim2.fromOffset(12, 0)
Title.Size = UDim2.new(1, -50, 1, 0)
Title.Font = Enum.Font.GothamBold
Title.TextSize = 14
Title.TextColor3 = Color3.fromRGB(255, 145, 40)
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Text = "🎃  MM2 FARM  ·  HALLOWEEN v5"
Title.Parent = Top

local Close = Instance.new("TextButton")
Close.Size = UDim2.fromOffset(26, 26)
Close.Position = UDim2.new(1, -34, 0.5, -13)
Close.BackgroundColor3 = Color3.fromRGB(150, 40, 35)
Close.Text = "X"
Close.TextColor3 = Color3.new(1, 1, 1)
Close.Font = Enum.Font.GothamBold
Close.TextSize = 12
Close.Parent = Top
Instance.new("UICorner", Close).CornerRadius = UDim.new(0, 6)
Close.MouseButton1Click:Connect(function() Main.Visible = false end)

do
	local drag, d0, p0 = false, nil, nil
	Top.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			drag = true; d0 = i.Position; p0 = Main.Position
		end
	end)
	local function endD(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then drag = false end
	end
	Top.InputEnded:Connect(endD)
	UIS.InputEnded:Connect(endD)
	UIS.InputChanged:Connect(function(i)
		if drag and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
			local d = i.Position - d0
			Main.Position = UDim2.new(p0.X.Scale, p0.X.Offset + d.X, p0.Y.Scale, p0.Y.Offset + d.Y)
		end
	end)
end

local Body = Instance.new("ScrollingFrame")
Body.BackgroundTransparency = 1
Body.Position = UDim2.fromOffset(10, 48)
Body.Size = UDim2.new(1, -20, 1, -58)
Body.BorderSizePixel = 0
Body.ScrollBarThickness = 3
Body.ScrollBarImageColor3 = Color3.fromRGB(255, 120, 30)
Body.CanvasSize = UDim2.new(0, 0, 0, 540)
Body.Parent = Main
local lay = Instance.new("UIListLayout")
lay.Padding = UDim.new(0, 5)
lay.Parent = Body

local Status = Instance.new("TextLabel")
Status.Size = UDim2.new(1, 0, 0, 38)
Status.BackgroundColor3 = Color3.fromRGB(30, 14, 18)
Status.TextColor3 = Color3.fromRGB(230, 195, 170)
Status.Font = Enum.Font.Code
Status.TextSize = 11
Status.TextWrapped = true
Status.Text = "🎃 Session: 0 / 50\nWaiting..."
Status.Parent = Body
Instance.new("UICorner", Status).CornerRadius = UDim.new(0, 6)

local function addToggle(label, key)
	local b = Instance.new("TextButton")
	b.Size = UDim2.new(1, 0, 0, 28)
	b.BackgroundColor3 = Color3.fromRGB(34, 22, 28)
	b.TextColor3 = Color3.new(1, 1, 1)
	b.Font = Enum.Font.GothamBold
	b.TextSize = 12
	b.TextXAlignment = Enum.TextXAlignment.Left
	b.AutoButtonColor = false
	b.Parent = Body
	Instance.new("UICorner", b).CornerRadius = UDim.new(0, 6)
	local function paint()
		if S[key] then
			b.BackgroundColor3 = Color3.fromRGB(35, 105, 50)
			b.Text = "  ON   |  " .. label
		else
			b.BackgroundColor3 = Color3.fromRGB(34, 22, 28)
			b.Text = "  OFF  |  " .. label
		end
	end
	paint()
	b.MouseButton1Click:Connect(function()
		S[key] = not S[key]
		paint()
		if key == "Enabled" and S.Enabled then refNoclip() end
		if (key == "CoinESP" or key == "CandyESP") and not S.CoinESP and not S.CandyESP then clearEsp() end
		notify(label .. ": " .. (S[key] and "ON" or "OFF"))
	end)
end

local function addSlider(label, key, min, max)
	local hold = Instance.new("Frame")
	hold.Size = UDim2.new(1, 0, 0, 40)
	hold.BackgroundColor3 = Color3.fromRGB(28, 16, 20)
	hold.Parent = Body
	Instance.new("UICorner", hold).CornerRadius = UDim.new(0, 6)
	local lab = Instance.new("TextLabel")
	lab.BackgroundTransparency = 1
	lab.Position = UDim2.fromOffset(8, 2)
	lab.Size = UDim2.new(1, -16, 0, 14)
	lab.Font = Enum.Font.Gotham
	lab.TextSize = 11
	lab.TextColor3 = Color3.fromRGB(255, 175, 100)
	lab.TextXAlignment = Enum.TextXAlignment.Left
	lab.Text = label .. ": " .. tostring(S[key])
	lab.Parent = hold
	local bar = Instance.new("Frame")
	bar.Position = UDim2.fromOffset(8, 20)
	bar.Size = UDim2.new(1, -16, 0, 8)
	bar.BackgroundColor3 = Color3.fromRGB(48, 32, 36)
	bar.Parent = hold
	Instance.new("UICorner", bar).CornerRadius = UDim.new(0, 4)
	local fill = Instance.new("Frame")
	fill.Size = UDim2.fromScale(math.clamp((S[key] - min) / math.max(max - min, 1), 0, 1), 1)
	fill.BackgroundColor3 = Color3.fromRGB(255, 115, 30)
	fill.Parent = bar
	Instance.new("UICorner", fill).CornerRadius = UDim.new(0, 4)
	local dragging = false
	local function set(x)
		local a = math.clamp((x - bar.AbsolutePosition.X) / math.max(bar.AbsoluteSize.X, 1), 0, 1)
		local v = math.floor(min + (max - min) * a + 0.5)
		S[key] = v
		fill.Size = UDim2.fromScale(a, 1)
		lab.Text = label .. ": " .. tostring(v)
	end
	bar.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			dragging = true; set(i.Position.X)
		end
	end)
	UIS.InputChanged:Connect(function(i)
		if dragging and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
			set(i.Position.X)
		end
	end)
	UIS.InputEnded:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then dragging = false end
	end)
end

addToggle("Autofarm", "Enabled")
addToggle("Under map", "UnderMap")
addToggle("Noclip", "Noclip")
addToggle("Soft TP (faster)", "SoftTP")
addToggle("Humanize delays", "Humanize")
addToggle("Anti AFK", "AntiAFK")
addToggle("Coin ESP", "CoinESP")
addToggle("Candy ESP", "CandyESP")
addToggle("Auto reset on full", "AutoResetFull")
addSlider("Full amount", "FullAmount", 10, 50)
addSlider("Safe speed", "Speed", 12, 55)
addSlider("Under Y", "UnderY", -45, -5)
addSlider("Touch height", "TouchHeight", 1, 4)
addSlider("TP delay (Soft)", "TPDelay", 50, 400)

task.spawn(function()
	while true do
		if S.Enabled then
			pcall(farmStep)
			Status.Text = string.format(
				"🎃 Session: %d / %d\nLeft: %d | %s",
				collectedSession,
				S.FullAmount,
				#listCoins(),
				S.SoftTP and "SOFT-TP" or "SAFE-TWEEN"
			)
			task.wait(S.SoftTP and 0.04 or 0.03)
		else
			local h = getHum()
			if h then h.PlatformStand = false end
			task.wait(0.2)
		end
	end
end)

task.spawn(function()
	while true do
		pcall(updateEsp)
		task.wait(0.35)
	end
end)

RunService.Stepped:Connect(function()
	if S.Enabled and S.Noclip then applyNoclip() end
end)

LP.Idled:Connect(function()
	if S.AntiAFK then
		VirtualUser:CaptureController()
		VirtualUser:ClickButton2(Vector2.zero)
	end
end)

UIS.InputBegan:Connect(function(input, gp)
	if gp then return end
	if input.KeyCode == Enum.KeyCode.RightShift then
		Main.Visible = not Main.Visible
	end
end)

notify("v5 ready — v3 pathing + Halloween GUI")
print("[MM2 Autofarm v5 Halloween] loaded")
