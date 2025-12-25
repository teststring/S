-- Services
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

-- Player
local LocalPlayer = Players.LocalPlayer

-- Toggles
local ESPEnabled = false
local SilentAimEnabled = false
local SpeedEnabled = false
local AntiStunEnabled = false

-- Silent Aim config
local PredictionEnabled = true
local PredictionAmount = 0.1
local MaxRange = 400
local MaxRangeSq = MaxRange * MaxRange

-- Speed / AntiStun
local SpeedValue = 700
local AntiStunPower = 1.2

-- Vars
local TargetPosition = nil
local ESPs = {}

-- ESP Folder
local espFolder = game.CoreGui:FindFirstChild("PlayerESP")
if not espFolder then
    espFolder = Instance.new("Folder")
    espFolder.Name = "PlayerESP"
    espFolder.Parent = game.CoreGui
end

-- Utils
local function getHRP(char)
    return char and char:FindFirstChild("HumanoidRootPart")
end

local function isEnemy(plr)
    if not LocalPlayer.Team or not plr.Team then
        return true
    end
    return plr.Team ~= LocalPlayer.Team
end

local function getPredictedPosition(hrp)
    if not PredictionEnabled then
        return hrp.Position
    end
    return hrp.Position + (hrp.Velocity * PredictionAmount)
end

local function getMainColor(plr)
    if LocalPlayer.Team and plr.Team and plr.Team == LocalPlayer.Team then
        return Color3.fromRGB(0, 255, 0)
    end
    return Color3.fromRGB(255, 255, 0)
end

-- ESP
local function createESP(plr)
    if ESPs[plr] or not plr.Character then return end
    local head = plr.Character:FindFirstChild("Head")
    if not head then return end

    local gui = Instance.new("BillboardGui")
    gui.Name = plr.Name
    gui.Adornee = head
    gui.Size = UDim2.fromOffset(240, 50)
    gui.StudsOffset = Vector3.new(0, 3, 0)
    gui.AlwaysOnTop = true
    gui.Parent = espFolder

    local levelLabel = Instance.new("TextLabel")
    levelLabel.Name = "Level"
    levelLabel.Size = UDim2.new(1, 0, 0.45, 0)
    levelLabel.BackgroundTransparency = 1
    levelLabel.Font = Enum.Font.SourceSansBold
    levelLabel.TextSize = 13
    levelLabel.TextStrokeTransparency = 0.2
    levelLabel.TextColor3 = Color3.fromRGB(0, 170, 255)
    levelLabel.TextXAlignment = Enum.TextXAlignment.Center
    levelLabel.Parent = gui

    local mainLabel = Instance.new("TextLabel")
    mainLabel.Name = "Main"
    mainLabel.Size = UDim2.new(1, 0, 0.55, 0)
    mainLabel.Position = UDim2.new(0, 0, 0.45, 0)
    mainLabel.BackgroundTransparency = 1
    mainLabel.Font = Enum.Font.SourceSansBold
    mainLabel.TextSize = 14
    mainLabel.TextStrokeTransparency = 0.2
    mainLabel.TextXAlignment = Enum.TextXAlignment.Center
    mainLabel.Parent = gui

    ESPs[plr] = gui
end

-- 🔥 Closest player (MaxRange REAL)
local function getClosestPlayer(hrp)
    local closest = nil
    local closestDistSq = math.huge

    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and isEnemy(plr) and plr.Character then
            local hum = plr.Character:FindFirstChildOfClass("Humanoid")
            local thrp = getHRP(plr.Character)

            if hum and hum.Health > 0 and thrp then
                local diff = thrp.Position - hrp.Position
                local distSq = diff.X*diff.X + diff.Y*diff.Y + diff.Z*diff.Z

                if distSq <= MaxRangeSq and distSq < closestDistSq then
                    closestDistSq = distSq
                    closest = plr
                end
            end
        end
    end

    return closest
end

-- Silent Aim hook
task.spawn(function()
    local mt = getrawmetatable(game)
    setreadonly(mt, false)

    local old
    old = hookmetamethod(game, "__namecall", function(self, ...)
        local args = {...}

        if getnamecallmethod():lower() == "fireserver"
        and SilentAimEnabled
        and TargetPosition
        and typeof(args[1]) == "Vector3" then
            args[1] = TargetPosition
            return old(self, unpack(args))
        end

        return old(self, ...)
    end)

    setreadonly(mt, true)
end)

-- Main loop
RunService.RenderStepped:Connect(function()
    local char = LocalPlayer.Character
    local hrp = getHRP(char)
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if not hrp or not hum then return end

    -- Speed
    if SpeedEnabled then
        hum.WalkSpeed = SpeedValue
    end

    -- AntiStun
    if AntiStunEnabled then
        local move = hum.MoveDirection
        if move.Magnitude > 0 then
            hrp.CFrame = hrp.CFrame + move.Unit * AntiStunPower
        end
    end

    -- 🔑 CORREÇÃO PRINCIPAL
    -- sempre limpa antes de calcular
    TargetPosition = nil

    if SilentAimEnabled then
        local target = getClosestPlayer(hrp)
        if target and target.Character then
            local thrp = getHRP(target.Character)
            if thrp then
                TargetPosition = getPredictedPosition(thrp)
            end
        end
    end

    -- ESP update
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer then
            if not ESPs[plr] then
                createESP(plr)
            end

            local gui = ESPs[plr]
            local pChar = plr.Character
            local pHRP = getHRP(pChar)
            local pHum = pChar and pChar:FindFirstChildOfClass("Humanoid")

            if ESPEnabled and gui and pHRP and pHum then
                gui.Enabled = true
                gui.Adornee = pChar:FindFirstChild("Head")

                local dist = math.floor((hrp.Position - pHRP.Position).Magnitude)
                local level = "?"

                local data = plr:FindFirstChild("Data")
                if data and data:FindFirstChild("Level") then
                    level = data.Level.Value
                end

                gui.Level.Text = "Lv. " .. level
                gui.Main.Text =
                    "[" .. math.floor(pHum.Health) .. "] "
                    .. plr.DisplayName .. " (" .. dist .. "m)"

                gui.Main.TextColor3 = getMainColor(plr)
            elseif gui then
                gui.Enabled = false
            end
        end
    end
end)

-- Cleanup
Players.PlayerRemoving:Connect(function(plr)
    if ESPs[plr] then
        ESPs[plr]:Destroy()
        ESPs[plr] = nil
    end
end)

-- Binds
UserInputService.InputBegan:Connect(function(input, gp)
    if gp then return end

    if input.KeyCode == Enum.KeyCode.L then
        ESPEnabled = not ESPEnabled

    elseif input.KeyCode == Enum.KeyCode.B then
        SilentAimEnabled = not SilentAimEnabled

    elseif input.KeyCode == Enum.KeyCode.K then
        SpeedEnabled = not SpeedEnabled
        AntiStunEnabled = SpeedEnabled
        AntiStunPower = 1.2

    elseif input.KeyCode == Enum.KeyCode.P then
        if not SpeedEnabled then
            AntiStunEnabled = not AntiStunEnabled
            AntiStunPower = AntiStunEnabled and 0.4 or 1.2
        end
    end
end)
