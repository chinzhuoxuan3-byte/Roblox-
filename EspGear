local Players = game:GetService("Players")
local Player = Players.LocalPlayer
local PlayerGui = Player:WaitForChild("PlayerGui")
local TweenService = game:GetService("TweenService")

-- 目标名称
local TARGET_NAMES = {["Gear"]=true, ["Blade"]=true, ["Spring"]=true}

-- ScreenGui
local Gui = Instance.new("ScreenGui")
Gui.Name = "LootESP_UI"
Gui.ResetOnSpawn = false
Gui.Parent = PlayerGui

-- 主面板
local Panel = Instance.new("Frame")
Panel.Size = UDim2.new(0,300,0,200)
Panel.Position = UDim2.new(0.5,-150,0.5,-100)
Panel.BackgroundColor3 = Color3.fromRGB(25,30,40)
Panel.BorderSizePixel = 0
Panel.Active = true
Panel.Draggable = true
Panel.Parent = Gui

local PanelCorner = Instance.new("UICorner")
PanelCorner.CornerRadius = UDim.new(0,14)
PanelCorner.Parent = Panel

local PanelStroke = Instance.new("UIStroke")
PanelStroke.Color = Color3.fromRGB(80,160,255)
PanelStroke.Thickness = 2
PanelStroke.Transparency = 0.3
PanelStroke.Parent = Panel

-- 背景渐变
local BgGrad = Instance.new("UIGradient")
BgGrad.Color = ColorSequence.new{
    ColorSequenceKeypoint.new(0, Color3.fromRGB(0,60,130)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(10,15,30))
}
BgGrad.Rotation = 65
BgGrad.Parent = Panel

-- 标题
local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1,0,0,30)
Title.Position = UDim2.new(0,0,0,0)
Title.BackgroundTransparency = 1
Title.Font = Enum.Font.Arcade
Title.TextSize = 22
Title.Text = "Loot ESP"
Title.TextColor3 = Color3.fromRGB(200,220,255)
Title.Parent = Panel

-- RGB 标题动效
task.spawn(function()
	local t = 0
	while task.wait(0.05) do
		t += 0.03
		local r = math.abs(math.sin(t))*255
		local g = math.abs(math.sin(t+2))*255
		local b = math.abs(math.sin(t+4))*255
		Title.TextColor3 = Color3.fromRGB(r,g,b)
	end
end)

-- 内容容器
local Body = Instance.new("Frame")
Body.Size = UDim2.new(1,-20,1,-40)
Body.Position = UDim2.new(0,10,0,35)
Body.BackgroundTransparency = 1
Body.Parent = Panel

-- 高亮管理
local highlights = {}
local espEnabled = {Gear=false, Blade=false, Spring=false}
local connAdded, connRemoved

local function addHighlight(obj)
	if highlights[obj] then return end
	local hl = Instance.new("Highlight")
	hl.Name = "LootESP_Highlight"
	hl.FillColor = Color3.fromRGB(255,200,0)
	hl.OutlineColor = Color3.fromRGB(255,255,255)
	hl.FillTransparency = 0.3
	hl.OutlineTransparency = 0
	hl.Adornee = obj
	hl.Parent = obj
	highlights[obj] = hl
end

local function removeHighlight(obj)
	if highlights[obj] then
		highlights[obj]:Destroy()
		highlights[obj] = nil
	end
end

local function clearAllHighlights()
	for obj, hl in pairs(highlights) do
		if hl and hl.Parent then hl:Destroy() end
	end
	highlights = {}
end

-- 扫描
local function scanAll()
	for _, obj in ipairs(workspace:GetDescendants()) do
		if TARGET_NAMES[obj.Name] and obj:IsA("BasePart") then
			if espEnabled[obj.Name] then
				addHighlight(obj)
			end
		end
	end
end

-- 监听
local function startListening()
	if connAdded then connAdded:Disconnect() end
	if connRemoved then connRemoved:Disconnect() end

	connAdded = workspace.DescendantAdded:Connect(function(obj)
		if TARGET_NAMES[obj.Name] and espEnabled[obj.Name] and obj:IsA("BasePart") then
			task.wait(0.05)
			addHighlight(obj)
		end
	end)
	connRemoved = workspace.DescendantRemoving:Connect(function(obj)
		if highlights[obj] then removeHighlight(obj) end
	end)
end

local function stopListening()
	if connAdded then connAdded:Disconnect() end
	if connRemoved then connRemoved:Disconnect() end
	connAdded, connRemoved = nil, nil
end

-- 创建 ESP 按钮
local function createESPButton(name, ypos)
	local btn = Instance.new("TextButton")
	btn.Size = UDim2.new(1,0,0,40)
	btn.Position = UDim2.new(0,0,0,ypos)
	btn.BackgroundColor3 = Color3.fromRGB(255,60,60)
	btn.TextColor3 = Color3.new(1,1,1)
	btn.Font = Enum.Font.SourceSansBold
	btn.TextSize = 18
	btn.Text = name.."：关闭"
	btn.Parent = Body

	btn.MouseButton1Click:Connect(function()
		espEnabled[name] = not espEnabled[name]
		if espEnabled[name] then
			btn.Text = name.."：开启"
			btn.BackgroundColor3 = Color3.fromRGB(60,255,100)
			scanAll()
			startListening()
		else
			btn.Text = name.."：关闭"
			btn.BackgroundColor3 = Color3.fromRGB(255,60,60)
			clearAllHighlights()
			stopListening()
		end
	end)
end

createESPButton("Gear",0)
createESPButton("Blade",50)
createESPButton("Spring",100)

-- ===============================
-- 可移动主面板开关按钮
-- ===============================
local ToggleButton = Instance.new("TextButton")
ToggleButton.Size = UDim2.new(0,100,0,40)
ToggleButton.Position = UDim2.new(0.5,-50,0.9,-20)
ToggleButton.BackgroundColor3 = Color3.fromRGB(50,120,255)
ToggleButton.TextColor3 = Color3.new(1,1,1)
ToggleButton.Font = Enum.Font.SourceSansBold
ToggleButton.TextSize = 16
ToggleButton.Text = "Loot ESP"
ToggleButton.Parent = Gui
ToggleButton.Active = true
ToggleButton.Draggable = true

local panelVisible = true
ToggleButton.MouseButton1Click:Connect(function()
	panelVisible = not panelVisible
	Panel.Visible = panelVisible
end)

print("[✅ 升级版 Loot ESP GUI 加载完成]")
