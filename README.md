-- WindUI
local WindUI = loadstring(game:HttpGet(
    "https://raw.githubusercontent.com/Footagesus/WindUI/main/dist/main.lua"
))()

local Window = WindUI:CreateWindow({
    Title = "Quality x",
    Icon = "Star",
    Author = "Qualitu-Team"
})

Window:EditOpenButton({
    Title = "Open Quality",
    Icon = "monitor",
    CornerRadius = UDim.new(0, 16),
    StrokeThickness = 2,
    Color = ColorSequence.new(
        Color3.fromRGB(255,15,123),
        Color3.fromRGB(248,155,41)
    ),
    OnlyMobile = false,
    Enabled = true,
    Draggable = true
})

------------------------------------------------
-- Services
------------------------------------------------
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

if not LocalPlayer.Character then
    LocalPlayer.CharacterAdded:Wait()
end

------------------------------------------------
-- Tabs
------------------------------------------------
local CombatTab   = Window:Tab({Title = "Combat",   Icon = "crosshair"})
local PlayerTab   = Window:Tab({Title = "Player",   Icon = "user"})
local TeleportTab = Window:Tab({Title = "Teleport", Icon = "map-pin"})
local ESPTab      = Window:Tab({Title = "ESP",      Icon = "eye"})

------------------------------------------------
-- FOV
------------------------------------------------
local FOVRadius  = 150
local FOVEnabled = false
local sides = 32
local FOVLines = {}

for i = 1, sides do
    local line = Drawing.new("Line")
    line.Color = Color3.new(1,1,1)
    line.Thickness = 2
    line.Visible = false
    FOVLines[i] = line
end

local AimLine = Drawing.new("Line")
AimLine.Color = Color3.new(1,0,0)
AimLine.Thickness = 2
AimLine.Visible = false

CombatTab:Toggle({
    Title = "Show FOV",
    Default = false,
    Callback = function(state)
        FOVEnabled = state
        for _, l in pairs(FOVLines) do
            l.Visible = state
        end
        AimLine.Visible = state
    end
})

CombatTab:Slider({
    Title = "FOV Size",
    Step = 1,
    Value = {Min = 50, Max = 1000, Default = FOVRadius},
    Callback = function(v)
        FOVRadius = v
    end
})

------------------------------------------------
-- SilentAim Team (Multi Select)
------------------------------------------------
getgenv().SilentAimbot = true

local SilentAimTeams = {} -- เลือกได้หลายทีม

CombatTab:Dropdown({
    Title = "SilentAim Team",
    Description = "Select teams to lock",
    Values = {
        "Guards",
        "Criminals",
        "Inmates"
    },
    Multi = true,
    Default = {},
    Callback = function(selected)
        SilentAimTeams = {}
        for _, teamName in ipairs(selected) do
            SilentAimTeams[teamName] = true
        end
    end
})

-- GetTarget (รองรับหลายทีม + FOV)
local function GetTarget()
    if not next(SilentAimTeams) then return nil end

    local closest, bestDist = nil, math.huge
    local center = Camera.ViewportSize / 2

    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer
        and plr.Team
        and SilentAimTeams[plr.Team.Name]
        and plr.Character
        and plr.Character:FindFirstChild("Head") then

            local head = plr.Character.Head
            local pos, onScreen = Camera:WorldToViewportPoint(head.Position)

            if onScreen then
                local dist = (Vector2.new(pos.X, pos.Y) - center).Magnitude
                if dist <= FOVRadius and dist < bestDist then
                    bestDist = dist
                    closest = head
                end
            end
        end
    end

    return closest
end

------------------------------------------------
-- SilentAim Hook (ของเดิม)
------------------------------------------------
local castRay
for _, fn in ipairs(getgc(true)) do
    if typeof(fn) == "function" then
        for _, uv in ipairs(debug.getupvalues(fn)) do
            if typeof(uv) == "function"
            and debug.getinfo(uv).name == "castRay" then
                castRay = uv
                break
            end
        end
    end
    if castRay then break end
end

if castRay then
    local old
    old = hookfunction(castRay, function(a, b, ...)
        if getgenv().SilentAimbot then
            local target = GetTarget()
            if target then
                return target, target.Position
            end
        end
        return old(a, b, ...)
    end)
end

------------------------------------------------
-- Render (FOV + เส้นแดงจากกลางจอ)
------------------------------------------------
RunService.RenderStepped:Connect(function()
    if not FOVEnabled then
        AimLine.Visible = false
        return
    end

    local center = Camera.ViewportSize / 2
    local points = {}

    for i = 0, sides - 1 do
        local ang = math.rad(i * (360 / sides))
        points[i+1] = Vector2.new(
            center.X + FOVRadius * math.cos(ang),
            center.Y + FOVRadius * math.sin(ang)
        )
    end

    for i = 1, sides do
        local n = (i % sides) + 1
        FOVLines[i].From = points[i]
        FOVLines[i].To   = points[n]
    end

    local target = GetTarget()
    if target then
        local tPos = Camera:WorldToViewportPoint(target.Position)
        AimLine.From = Vector2.new(center.X, center.Y)
        AimLine.To   = Vector2.new(tPos.X, tPos.Y)
        AimLine.Visible = true
    else
        AimLine.Visible = false
    end
end)

------------------------------------------------
-- Player (NoClip)
------------------------------------------------

local SavedWalkSpeed = 16
local SavedJumpPower = 50

LocalPlayer.CharacterAdded:Connect(function(char)
    local hum = char:WaitForChild("Humanoid")

    task.wait(0.1) -- กันบางเกมรีเซ็ตทับ
    hum.WalkSpeed = SavedWalkSpeed
    hum.JumpPower = SavedJumpPower
end)

local NoClip = false

PlayerTab:Toggle({
    Title = "NoClip",
    Default = false,
    Callback = function(state)
        NoClip = state
    end
})

RunService.Stepped:Connect(function()
    if NoClip and LocalPlayer.Character then
        for _, part in ipairs(LocalPlayer.Character:GetDescendants()) do
            if part:IsA("BasePart") then
                part.CanCollide = false
            end
        end
    end
end)

PlayerTab:Slider({
    Title = "WalkSpeed",
    Step = 1,
    Value = {
        Min = 16,
        Max = 200,
        Default = 16
    },
    Callback = function(value)
        SavedWalkSpeed = value

        local char = LocalPlayer.Character
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if hum then
            hum.WalkSpeed = value
        end
    end
})

PlayerTab:Slider({
    Title = "JumpPower",
    Step = 1,
    Value = {
        Min = 50,
        Max = 300,
        Default = 50
    },
    Callback = function(value)
        SavedJumpPower = value

        local char = LocalPlayer.Character
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if hum then
            hum.JumpPower = value
        end
    end
})

------------------------------------------------
-- Teleport
------------------------------------------------
TeleportTab:Button({
    Title = "TP Tower",
    Callback = function()
        local hrp = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        if hrp then
            hrp.CFrame = CFrame.new(822.47509765625,139.43064880371094,2588.803466796875)
        end
    end
})

TeleportTab:Button({
    Title = "TP Thief",
    Callback = function()
        local hrp = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        if hrp then
            hrp.CFrame = CFrame.new(-977.0906982421875,109.65302276611328,2056.8154296875)
        end
    end
})

TeleportTab:Button({
    Title = "TP MP5 (Get & Back)",
    Callback = function()
        local hrp = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        if not hrp then return end

        local old = hrp.CFrame
        hrp.CFrame = CFrame.new(813.44921875,99.18183135986328,2229.074462890625)
        task.wait(0.3)

        local obj = workspace:GetChildren()[187]
        if obj and obj:FindFirstChild("TouchGiver") then
            firetouchinterest(hrp, obj.TouchGiver, 0)
            firetouchinterest(hrp, obj.TouchGiver, 1)
        end

        task.wait(0.3)
        hrp.CFrame = old
    end
})

TeleportTab:Button({
    Title = "TP Remington (Get & Back)",
    Callback = function()
        local hrp = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        if not hrp then return end

        local old = hrp.CFrame
        hrp.CFrame = CFrame.new(821.404541015625,98.10057830810547,2228.724853515625)
        task.wait(0.3)

        local obj = workspace:GetChildren()[183]
        if obj and obj:FindFirstChild("TouchGiver") then
            firetouchinterest(hrp, obj.TouchGiver, 0)
            firetouchinterest(hrp, obj.TouchGiver, 1)
        end

        task.wait(0.3)
        hrp.CFrame = old
    end
})

------------------------------------------------
-- Arrest Aura
------------------------------------------------
local ArrestRemote = ReplicatedStorage
    :WaitForChild("Remotes")
    :WaitForChild("ArrestPlayer")

local ArrestAuraEnabled = false
local DISTANCE = 10

CombatTab:Toggle({
    Title = "Arrest Aura",
    Description = "Only the police",
    Default = false,
    Callback = function(state)
        ArrestAuraEnabled = state
    end
})

task.spawn(function()
    while task.wait(0.2) do
        if not ArrestAuraEnabled then continue end

        local hrp = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        if not hrp then continue end

        for _, plr in pairs(Players:GetPlayers()) do
            if plr ~= LocalPlayer and plr.Character then
                local thrp = plr.Character:FindFirstChild("HumanoidRootPart")
                if thrp and (hrp.Position - thrp.Position).Magnitude <= DISTANCE then
                    pcall(function()
                        ArrestRemote:InvokeServer(plr)
                    end)
                end
            end
        end
    end
end)

------------------------------------------------
-- ESP : Name / Health / Distance (Separate Toggles)
------------------------------------------------

local ESP_Name = false
local ESP_Health = false
local ESP_Distance = false

local ESPObjects = {}

-- สีตามทีม
local TeamColors = {
    Guards = Color3.fromRGB(0, 85, 255),       -- น้ำเงินเข้ม
    Inmates = Color3.fromRGB(255, 140, 0),     -- ส้ม
    Criminals = Color3.fromRGB(170, 0, 0)      -- แดงเข้ม
}

------------------------------------------------
-- Toggles
------------------------------------------------
ESPTab:Toggle({
    Title = "ESP Name",
    Default = false,
    Callback = function(v)
        ESP_Name = v
    end
})

ESPTab:Toggle({
    Title = "ESP Health",
    Default = false,
    Callback = function(v)
        ESP_Health = v
    end
})

ESPTab:Toggle({
    Title = "ESP Distance",
    Default = false,
    Callback = function(v)
        ESP_Distance = v
    end
})

------------------------------------------------
-- Create ESP
------------------------------------------------
local function CreateESP(plr)
    if ESPObjects[plr] then return end

    -- ===== Head GUI =====
    local HeadGui = Instance.new("BillboardGui")
    HeadGui.Size = UDim2.new(0, 200, 0, 40)
    HeadGui.StudsOffset = Vector3.new(0, 2.5, 0)
    HeadGui.AlwaysOnTop = true
    HeadGui.Parent = game:GetService("CoreGui")

    local NameLabel = Instance.new("TextLabel")
    NameLabel.Size = UDim2.new(1, 0, 0.5, 0)
    NameLabel.BackgroundTransparency = 1
    NameLabel.TextSize = 14
    NameLabel.Font = Enum.Font.SourceSansBold
    NameLabel.TextStrokeTransparency = 0
    NameLabel.Parent = HeadGui

    local HPLabel = Instance.new("TextLabel")
    HPLabel.Position = UDim2.new(0, 0, 0.5, 0)
    HPLabel.Size = UDim2.new(1, 0, 0.5, 0)
    HPLabel.BackgroundTransparency = 1
    HPLabel.TextSize = 14
    HPLabel.Font = Enum.Font.SourceSans
    HPLabel.TextStrokeTransparency = 0
    HPLabel.Parent = HeadGui

    -- ===== Foot GUI =====
    local FootGui = Instance.new("BillboardGui")
    FootGui.Size = UDim2.new(0, 200, 0, 20)
    FootGui.StudsOffset = Vector3.new(0, -3, 0)
    FootGui.AlwaysOnTop = true
    FootGui.Parent = game:GetService("CoreGui")

    local DistLabel = Instance.new("TextLabel")
    DistLabel.Size = UDim2.new(1, 0, 1, 0)
    DistLabel.BackgroundTransparency = 1
    DistLabel.TextSize = 14
    DistLabel.Font = Enum.Font.SourceSans
    DistLabel.TextStrokeTransparency = 0
    DistLabel.Parent = FootGui

    ESPObjects[plr] = {
        HeadGui = HeadGui,
        FootGui = FootGui,
        NameLabel = NameLabel,
        HPLabel = HPLabel,
        DistLabel = DistLabel
    }
end

------------------------------------------------
-- Remove ESP
------------------------------------------------
local function RemoveESP(plr)
    if ESPObjects[plr] then
        for _, v in pairs(ESPObjects[plr]) do
            if typeof(v) == "Instance" then
                v:Destroy()
            end
        end
        ESPObjects[plr] = nil
    end
end

------------------------------------------------
-- Update ESP
------------------------------------------------
RunService.RenderStepped:Connect(function()
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer then
            if not ESPObjects[plr] then
                CreateESP(plr)
            end

            local esp = ESPObjects[plr]
            local char = plr.Character
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            local head = char and char:FindFirstChild("Head")
            local hrp = char and char:FindFirstChild("HumanoidRootPart")

            if char and hum and head and hrp then
                local teamColor = TeamColors[plr.Team and plr.Team.Name] or Color3.new(1,1,1)

                esp.HeadGui.Adornee = head
                esp.FootGui.Adornee = hrp

                -- Name
                esp.NameLabel.Visible = ESP_Name
                esp.NameLabel.Text = plr.Name
                esp.NameLabel.TextColor3 = teamColor

                -- Health
                esp.HPLabel.Visible = ESP_Health
                esp.HPLabel.Text = "HP: " .. math.floor(hum.Health)
                esp.HPLabel.TextColor3 = Color3.fromRGB(0,255,0)

                -- Distance
                esp.FootGui.Enabled = ESP_Distance
                local dist = (LocalPlayer.Character.HumanoidRootPart.Position - hrp.Position).Magnitude
                esp.DistLabel.Text = string.format("[ %.0f m ]", dist)
                esp.DistLabel.TextColor3 = teamColor
            end
        end
    end
end)

Players.PlayerRemoving:Connect(RemoveESP)
