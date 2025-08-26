--[[ 
    Life Sentence – Parts Assistant (Mobile/Arceus Friendly)
    Features:
      • Auto TP & Auto Pickup (Gear / Spring / Blade)
      • Draggable UI • Tween/Instant TP • Interval & Distance Controls
      • No-repeat TP (最近不重复) • Per-item cooldown • Max per-cycle
      • ESP with distance • Sound cue (可关) • Status & Logs
      • 手机按钮大、可拖动标题栏、可最小化
    Author: ChatGPT (GPT-5 Thinking) — 2025-08
]]

-- ===========================
-- 安全上下文 & 基础服务
-- ===========================
local Players        = game:GetService("Players")
local RunService     = game:GetService("RunService")
local TweenService   = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local SoundService   = game:GetService("SoundService")
local Workspace      = game:GetService("Workspace")

local LocalPlayer    = Players.LocalPlayer
local PlayerGui      = LocalPlayer:WaitForChild("PlayerGui")

local function getChar()
    local c = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
    return c
end
local function getHRP()
    local c = getChar()
    return c:WaitForChild("HumanoidRootPart", 5)
end
local function getHum()
    local c = getChar()
    return c:WaitForChild("Humanoid", 5)
end

-- 兼容执行器特性（有则用，无则安全降级）
local firetouchinterest = rawget(getfenv(), "firetouchinterest")
local fireproximityprompt = rawget(getfenv(), "fireproximityprompt")
local fireclickdetector = rawget(getfenv(), "fireclickdetector")

-- ===========================
-- 配置（可在 UI 调整）
-- ===========================
local AllowedNames = { Gear = true, Spring = true, Blade = true } -- 严格大小写
local ColorMap = {
    Gear   = Color3.fromRGB(80,180,255),
    Spring = Color3.fromRGB(255,220,80),
    Blade  = Color3.fromRGB(255,80,80),
}

local Settings = {
    Enabled          = false,     -- 自动拾取总开关
    ESPEnabled       = true,      -- ESP 开关
    SoundEnabled     = true,      -- 提示音开关
    UseTween         = true,      -- Tween 方式移动（更稳，不易检测）
    IntervalSec      = 3,         -- 每次传送间隔（秒）
    HoverOffsetY     = 3.5,       -- 传送到物体上方的高度
    MaxDistance      = 250,       -- 仅考虑这范围内的目标
    MaxPerCycle      = 3,         -- 单循环最多处理多少个目标
    PerInstanceCD    = 10,        -- 同一实例处理后的冷却(秒)
    ESPMaxShow       = 6,         -- 同屏最多显示多少个 ESP
    ESPMaxDist       = 250,       -- ESP 最远显示距离
    TweenTime        = 0.35,      -- Tween 时长
    TweenEasingStyle = Enum.EasingStyle.Quad,
    TweenEasingDir   = Enum.EasingDirection.Out,
}

-- 运行态
local State = {
    LastVisitedIds = {},    -- [inst] = tick() 最近处理时间
    LastTPPos      = nil,   -- 上一次传送到的位置（避免连跳同点）
    Busy           = false, -- 正在执行一次传送/拾取
    CycleIndex     = 0,     -- 循环计数
}

-- ===========================
-- 工具函数
-- ===========================
local function tprint(...)
    local msg = table.concat({...}, " ")
    print("[LS-Assist] " .. msg)
end

local function safeVector(v)
    if typeof(v) == "Vector3" and (v.X==v.X and v.Y==v.Y and v.Z==v.Z) then
        return true
    end
    return false
end

local function partOf(inst)
    if inst:IsA("BasePart") then return inst end
    return inst:FindFirstChildWhichIsA("BasePart", true)
end

local function getDist(a, b)
    if not (a and b) then return math.huge end
    return (a - b).Magnitude
end

local function now() return tick() end

local function recentlyVisited(inst)
    local t = State.LastVisitedIds[inst]
    if not t then return false end
    return (now() - t) < Settings.PerInstanceCD
end

local function markVisited(inst)
    State.LastVisitedIds[inst] = now()
end

local function roughlySamePos(a, b)
    if not (a and b) then return false end
    return (a - b).Magnitude < 5
end

-- ===========================
-- UI 构建（原生，无外部库，支持拖动/最小化）
-- ===========================
local Gui = Instance.new("ScreenGui")
Gui.Name = "LS_Assist_UI"
Gui.IgnoreGuiInset = true
Gui.ResetOnSpawn = false
Gui.ZIndexBehavior = Enum.ZIndexBehavior.Global
Gui.Parent = PlayerGui

-- 主窗口
local Main = Instance.new("Frame")
Main.Name = "Main"
Main.Size = UDim2.new(0, 320, 0, 420)
Main.Position = UDim2.new(0.06, 0, 0.2, 0)
Main.BackgroundColor3 = Color3.fromRGB(24, 24, 28)
Main.BorderSizePixel = 0
Main.Parent = Gui

local UICornerMain = Instance.new("UICorner", Main)
UICornerMain.CornerRadius = UDim.new(0, 14)

-- 顶部标题（可拖动）
local Title = Instance.new("TextButton")
Title.Size = UDim2.new(1, 0, 0, 44)
Title.Text = "Life Sentence – Parts Assistant"
Title.Font = Enum.Font.GothamBold
Title.TextSize = 16
Title.TextColor3 = Color3.fromRGB(255,255,255)
Title.BackgroundColor3 = Color3.fromRGB(36, 36, 44)
Title.AutoButtonColor = false
Title.Parent = Main

local UICornerTitle = Instance.new("UICorner", Title)
UICornerTitle.CornerRadius = UDim.new(0, 14)

-- 拖动逻辑
do
    local dragging, dragStart, startPos
    Title.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 
        or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = Main.Position
        end
    end)
    Title.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 
        or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement 
            or input.UserInputType == Enum.UserInputType.Touch) then
            local delta = input.Position - dragStart
            Main.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X,
                                      startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

-- 最小化按钮
local BtnMin = Instance.new("TextButton")
BtnMin.Size = UDim2.new(0, 80, 0, 32)
BtnMin.Position = UDim2.new(1, -88, 0, 6)
BtnMin.AnchorPoint = Vector2.new(0,0)
BtnMin.Text = "Minimize"
BtnMin.Font = Enum.Font.Gotham
BtnMin.TextSize = 14
BtnMin.TextColor3 = Color3.new(1,1,1)
BtnMin.BackgroundColor3 = Color3.fromRGB(70,70,90)
BtnMin.AutoButtonColor = true
BtnMin.Parent = Main
Instance.new("UICorner", BtnMin).CornerRadius = UDim.new(0, 10)

local Body = Instance.new("Frame")
Body.Name = "Body"
Body.Size = UDim2.new(1, -16, 1, -60)
Body.Position = UDim2.new(0, 8, 0, 52)
Body.BackgroundTransparency = 1
Body.Parent = Main

local minimized = false
BtnMin.MouseButton1Click:Connect(function()
    minimized = not minimized
    Body.Visible = not minimized
    BtnMin.Text = minimized and "Restore" or "Minimize"
    Main.Size = minimized and UDim2.new(0, 320, 0, 60) or UDim2.new(0, 320, 0, 420)
end)

-- 分栏容器
local ColumnL = Instance.new("Frame")
ColumnL.Size = UDim2.new(0.52, 0, 1, 0)
ColumnL.BackgroundTransparency = 1
ColumnL.Parent = Body

local ColumnR = Instance.new("Frame")
ColumnR.Size = UDim2.new(0.48, 0, 1, 0)
ColumnR.Position = UDim2.new(0.52, 0, 0, 0)
ColumnR.BackgroundTransparency = 1
ColumnR.Parent = Body

-- 工具：标题文本
local function makeHeader(parent, text)
    local h = Instance.new("TextLabel")
    h.Size = UDim2.new(1, 0, 0, 26)
    h.Text = text
    h.Font = Enum.Font.GothamBold
    h.TextSize = 14
    h.TextXAlignment = Enum.TextXAlignment.Left
    h.TextColor3 = Color3.fromRGB(200, 205, 220)
    h.BackgroundTransparency = 1
    h.Parent = parent
    return h
end

-- 工具：切换按钮
local function makeToggle(parent, label, default, onToggle)
    local holder = Instance.new("Frame")
    holder.Size = UDim2.new(1, 0, 0, 40)
    holder.BackgroundTransparency = 1
    holder.Parent = parent

    local txt = Instance.new("TextLabel")
    txt.Size = UDim2.new(0.6, 0, 1, 0)
    txt.Text = label
    txt.Font = Enum.Font.Gotham
    txt.TextSize = 14
    txt.TextXAlignment = Enum.TextXAlignment.Left
    txt.TextColor3 = Color3.fromRGB(230,230,235)
    txt.BackgroundTransparency = 1
    txt.Parent = holder

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0.35, 0, 0.7, 0)
    btn.Position = UDim2.new(0.65, 0, 0.15, 0)
    btn.Text = default and "ON" or "OFF"
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 14
    btn.TextColor3 = Color3.new(1,1,1)
    btn.BackgroundColor3 = default and Color3.fromRGB(60,200,80) or Color3.fromRGB(200,60,60)
    btn.Parent = holder
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 10)

    btn.MouseButton1Click:Connect(function()
        default = not default
        btn.Text = default and "ON" or "OFF"
        btn.BackgroundColor3 = default and Color3.fromRGB(60,200,80) or Color3.fromRGB(200,60,60)
        onToggle(default)
    end)
    return holder
end

-- 工具：步进器(按钮+数值)
local function makeStepper(parent, label, min, max, step, default, onChange)
    local holder = Instance.new("Frame")
    holder.Size = UDim2.new(1, 0, 0, 40)
    holder.BackgroundTransparency = 1
    holder.Parent = parent

    local txt = Instance.new("TextLabel")
    txt.Size = UDim2.new(0.6, 0, 1, 0)
    txt.Text = label
    txt.Font = Enum.Font.Gotham
    txt.TextSize = 14
    txt.TextXAlignment = Enum.TextXAlignment.Left
    txt.TextColor3 = Color3.fromRGB(230,230,235)
    txt.BackgroundTransparency = 1
    txt.Parent = holder

    local val = Instance.new("TextLabel")
    val.Size = UDim2.new(0.25, 0, 0.7, 0)
    val.Position = UDim2.new(0.6, 0, 0.15, 0)
    val.Text = tostring(default)
    val.Font = Enum.Font.GothamBold
    val.TextSize = 14
    val.TextColor3 = Color3.fromRGB(255,255,255)
    val.BackgroundColor3 = Color3.fromRGB(50,50,65)
    val.Parent = holder
    Instance.new("UICorner", val).CornerRadius = UDim.new(0, 8)

    local minus = Instance.new("TextButton")
    minus.Size = UDim2.new(0.075, 0, 0.7, 0)
    minus.Position = UDim2.new(0.86, 0, 0.15, 0)
    minus.Text = "-"
    minus.Font = Enum.Font.GothamBold
    minus.TextSize = 16
    minus.TextColor3 = Color3.new(1,1,1)
    minus.BackgroundColor3 = Color3.fromRGB(100,60,60)
    minus.Parent = holder
    Instance.new("UICorner", minus).CornerRadius = UDim.new(0, 8)

    local plus = Instance.new("TextButton")
    plus.Size = UDim2.new(0.075, 0, 0.7, 0)
    plus.Position = UDim2.new(0.94, 0, 0.15, 0)
    plus.Text = "+"
    plus.Font = Enum.Font.GothamBold
    plus.TextSize = 16
    plus.TextColor3 = Color3.new(1,1,1)
    plus.BackgroundColor3 = Color3.fromRGB(60,100,60)
    plus.Parent = holder
    Instance.new("UICorner", plus).CornerRadius = UDim.new(0, 8)

    local cur = default
    local function apply(n)
        n = math.clamp(math.floor(n/step+0.5)*step, min, max)
        cur = n
        val.Text = tostring(cur)
        onChange(cur)
    end
    minus.MouseButton1Click:Connect(function() apply(cur - step) end)
    plus.MouseButton1Click:Connect(function() apply(cur + step) end)

    return holder
end

-- 工具：按钮
local function makeButton(parent, label, cb, color)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(1, 0, 0, 40)
    b.Text = label
    b.Font = Enum.Font.GothamBold
    b.TextSize = 14
    b.TextColor3 = Color3.new(1,1,1)
    b.BackgroundColor3 = color or Color3.fromRGB(70,90,130)
    b.Parent = parent
    Instance.new("UICorner", b).CornerRadius = UDim.new(0, 10)
    b.MouseButton1Click:Connect(function() cb(b) end)
    return b
end

-- 左列：主开关 + 参数
makeHeader(ColumnL, "Main")
makeToggle(ColumnL, "Auto Collect", Settings.Enabled, function(on)
    Settings.Enabled = on
end)
makeToggle(ColumnL, "ESP Overlay", Settings.ESPEnabled, function(on)
    Settings.ESPEnabled = on
end)
makeToggle(ColumnL, "Sound Cue", Settings.SoundEnabled, function(on)
    Settings.SoundEnabled = on
end)
makeToggle(ColumnL, "Tween Move (smooth)", Settings.UseTween, function(on)
    Settings.UseTween = on
end)

makeHeader(ColumnL, "Timing / Limits")
makeStepper(ColumnL, "Interval (sec)", 1, 10, 1, Settings.IntervalSec, function(v)
    Settings.IntervalSec = v
end)
makeStepper(ColumnL, "Max Distance", 100, 800, 25, Settings.MaxDistance, function(v)
    Settings.MaxDistance = v
end)
makeStepper(ColumnL, "Max Per Cycle", 1, 10, 1, Settings.MaxPerCycle, function(v)
    Settings.MaxPerCycle = v
end)
makeStepper(ColumnL, "Per-Item Cooldown(s)", 3, 60, 1, Settings.PerInstanceCD, function(v)
    Settings.PerInstanceCD = v
end)

makeHeader(ColumnL, "Positioning")
makeStepper(ColumnL, "Hover Offset Y", 1, 10, 0.5, Settings.HoverOffsetY, function(v)
    Settings.HoverOffsetY = v
end)
makeStepper(ColumnL, "ESP Max Distance", 100, 800, 25, Settings.ESPMaxDist, function(v)
    Settings.ESPMaxDist = v
end)
makeStepper(ColumnL, "ESP Max Show", 3, 12, 1, Settings.ESPMaxShow, function(v)
    Settings.ESPMaxShow = v
end)

-- 右列：状态 & 日志 & 控制
makeHeader(ColumnR, "Status")
local Status = Instance.new("TextLabel")
Status.Size = UDim2.new(1, 0, 0, 46)
Status.Text = "Idle"
Status.Font = Enum.Font.GothamMedium
Status.TextSize = 14
Status.TextColor3 = Color3.fromRGB(235,235,240)
Status.BackgroundColor3 = Color3.fromRGB(38,40,60)
Status.Parent = ColumnR
Instance.new("UICorner", Status).CornerRadius = UDim.new(0, 10)

local function setStatus(s)
    Status.Text = s
end

makeHeader(ColumnR, "Controls")
makeButton(ColumnR, "Start", function()
    Settings.Enabled = true
    setStatus("Running…")
end, Color3.fromRGB(60,160,80))

makeButton(ColumnR, "Stop", function()
    Settings.Enabled = false
    setStatus("Stopped.")
end, Color3.fromRGB(160,80,80))

-- 日志框
makeHeader(ColumnR, "Logs")
local LogBox = Instance.new("TextBox")
LogBox.Size = UDim2.new(1, 0, 1, -210)
LogBox.Position = UDim2.new(0, 0, 0, 210)
LogBox.ClearTextOnFocus = false
LogBox.MultiLine = true
LogBox.TextWrapped = false
LogBox.TextXAlignment = Enum.TextXAlignment.Left
LogBox.TextYAlignment = Enum.TextYAlignment.Top
LogBox.TextEditable = false
LogBox.Font = Enum.Font.Code
LogBox.TextSize = 14
LogBox.Text = ""
LogBox.TextColor3 = Color3.fromRGB(220,230,240)
LogBox.BackgroundColor3 = Color3.fromRGB(28,28,36)
LogBox.Parent = ColumnR
Instance.new("UICorner", LogBox).CornerRadius = UDim.new(0, 8)

local function log(s)
    LogBox.Text = (os.date("[%H:%M:%S] ") .. s .. "\n") .. LogBox.Text
end

-- ===========================
-- 声音
-- ===========================
local SoundReady = Instance.new("Sound")
SoundReady.SoundId = "rbxassetid://138186576"
SoundReady.Volume = 1.6
SoundReady.Parent = SoundService

-- ===========================
-- ESP 系统
-- ===========================
local ESPGui = Instance.new("Folder")
ESPGui.Name = "ESP_Folder"
ESPGui.Parent = Gui

local Tracked = {} -- [inst] = {bb=..., lbl=..., name=..., adornee=...}
local function attachESP(inst)
    if Tracked[inst] or not AllowedNames[inst.Name] then return end
    local adorn = partOf(inst)
    if not adorn then return end

    local bb = Instance.new("BillboardGui")
    bb.Name = "ESP_" .. inst.Name
    bb.Adornee = adorn
    bb.Size = UDim2.new(0, 140, 0, 38)
    bb.StudsOffset = Vector3.new(0, 2.2, 0)
    bb.AlwaysOnTop = true
    bb.Enabled = true
    bb.Parent = ESPGui

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1,0,1,0)
    lbl.BackgroundTransparency = 1
    lbl.TextColor3 = ColorMap[inst.Name] or Color3.new(1,1,1)
    lbl.Font = Enum.Font.GothamBold
    lbl.TextScaled = true
    lbl.Text = inst.Name
    lbl.Parent = bb

    Tracked[inst] = { bb = bb, lbl = lbl, name = inst.Name, adornee = adorn }
end

local function detachESP(inst)
    local t = Tracked[inst]
    if not t then return end
    if t.bb then t.bb:Destroy() end
    Tracked[inst] = nil
end

-- 初次扫描 + 动态监听
for _, d in ipairs(Workspace:GetDescendants()) do
    if AllowedNames[d.Name] then
        attachESP(d)
    end
end
Workspace.DescendantAdded:Connect(function(d)
    if AllowedNames[d.Name] then
        attachESP(d)
    end
end)
Workspace.DescendantRemoving:Connect(function(d)
    if Tracked[d] then
        detachESP(d)
    end
end)

-- ESP 刷新（每 1s）
task.spawn(function()
    while task.wait(1) do
        if not Settings.ESPEnabled then
            for _, t in pairs(Tracked) do if t.bb then t.bb.Enabled = false end end
        else
            local hrp = getHRP()
            if not hrp then continue end

            -- 收集候选（有距离的和已存在的）
            local cand = {}
            for inst, t in pairs(Tracked) do
                if inst and inst.Parent and t.adornee then
                    local dist = getDist(hrp.Position, t.adornee.Position)
                    table.insert(cand, {inst = inst, t = t, dist = dist})
                end
            end
            table.sort(cand, function(a,b) return a.dist < b.dist end)

            local shown = 0
            for i, c in ipairs(cand) do
                local t = c.t
                if c.dist <= Settings.ESPMaxDist and shown < Settings.ESPMaxShow then
                    t.bb.Enabled = true
                    t.lbl.Text = string.format("%s [%dm]", t.name, math.floor(c.dist))
                    shown += 1
                else
                    t.bb.Enabled = false
                end
            end
        end
    end
end)

-- ===========================
-- 目标查找 & 排序
-- ===========================
local function collectTargets()
    local hrp = getHRP()
    if not hrp then return {} end

    local list = {}
    for inst, _ in pairs(Tracked) do
        if inst and inst.Parent then
            local p = partOf(inst)
            if p then
                local d = getDist(hrp.Position, p.Position)
                if d <= Settings.MaxDistance and not recentlyVisited(inst) then
                    table.insert(list, {inst=inst, part=p, dist=d})
                end
            end
        end
    end
    table.sort(list, function(a,b) return a.dist < b.dist end)
    return list
end

-- ===========================
-- 拾取实现（多策略并行尝试）
-- ===========================
local function tryPickup(inst, part)
    local ok = false
    -- ProximityPrompt
    local prompt = inst:FindFirstChildWhichIsA("ProximityPrompt", true)
    if prompt and fireproximityprompt then
        pcall(function()
            fireproximityprompt(prompt, 1)
            ok = true
        end)
    end

    -- ClickDetector
    if not ok then
        local click = inst:FindFirstChildWhichIsA("ClickDetector", true)
        if click and fireclickdetector then
            pcall(function()
                fireclickdetector(click)
                ok = true
            end)
        end
    end

    -- Touch 触碰
    if not ok and part and firetouchinterest then
        local hrp = getHRP()
        if hrp then
            pcall(function()
                firetouchinterest(hrp, part, 0)
                task.wait(0.1)
                firetouchinterest(hrp, part, 1)
                ok = true
            end)
        end
    end

    -- 作为兜底：贴近后轻微 MoveTo（触发触碰盒）
    if not ok then
        local hum = getHum()
        if hum and part then
            pcall(function()
                hum:MoveTo(part.Position + Vector3.new(0,0.5,0))
            end)
        end
    end
end

-- ===========================
-- 传送（Tween/瞬移）
-- ===========================
local function tpTo(pos)
    local hrp = getHRP()
    if not hrp then return end
    if not safeVector(pos) then return end

    -- 避免连续同点
    if State.LastTPPos and roughlySamePos(State.LastTPPos, pos) then
        pos = pos + Vector3.new(0, 0.01, 0)
    end

    if Settings.UseTween then
        local tween = TweenService:Create(hrp, TweenInfo.new(
            Settings.TweenTime, Settings.TweenEasingStyle, Settings.TweenEasingDir
        ), {CFrame = CFrame.new(pos)})
        tween:Play()
        tween.Completed:Wait()
    else
        hrp.CFrame = CFrame.new(pos)
        RunService.Heartbeat:Wait()
    end

    State.LastTPPos = pos
end

-- ===========================
-- 主循环
-- ===========================
task.spawn(function()
    while task.wait(0.1) do
        if not Settings.Enabled then
            task.wait(0.2)
            continue
        end
        if State.Busy then
            task.wait(0.05)
            continue
        end

        State.Busy = true
        State.CycleIndex += 1

        local targets = collectTargets()
        local processed = 0

        if #targets == 0 then
            setStatus(("No target (<=%dm)."):format(Settings.MaxDistance))
            State.Busy = false
            task.wait(Settings.IntervalSec)
            continue
        end

        for _, item in ipairs(targets) do
            if processed >= Settings.MaxPerCycle then break end
            local inst, part = item.inst, item.part
            if not (inst and inst.Parent and part) then continue end

            -- 传送到上方
            local dest = part.Position + Vector3.new(0, Settings.HoverOffsetY, 0)
            setStatus(("TP -> %s [%dm]"):format(inst.Name, math.floor(item.dist)))
            log(("Go %s  d=%dm"):format(inst.Name, math.floor(item.dist)))
            tpTo(dest)

            -- 拾取
            tryPickup(inst, part)
            markVisited(inst)
            processed += 1

            -- 提示音（可关）
            if Settings.SoundEnabled then
                pcall(function() SoundReady:Play() end)
            end

            -- 间隔
            task.wait(Settings.IntervalSec)
        end

        setStatus(("Cycle #%d done (%d/%d)."):format(State.CycleIndex, processed, Settings.MaxPerCycle))
        State.Busy = false
    end
end)

-- 初始日志
log("Ready. Tap Start to run. Targets: Gear/Spring/Blade (case-sensitive).")
setStatus("Idle")
```0
