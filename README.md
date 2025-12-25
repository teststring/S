local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer

local ESPEnabled = false
local SilentAimEnabled = false
local SpeedEnabled = false
local AntiStunEnabled = false

local PredictionEnabled = true
local PredictionAmount = 0.1
local MaxRange = 400
local SpeedValue = 700

local AntiStunPower = 1.2

local TargetPosition = nil
local CurrentTargetHRP = nil
local ESPs = {}

-- =========================
-- ESP FOLDER
-- =========================
local espFolder = game.CoreGui:FindFirstChild("PlayerESP")
if not espFolder then
    espFolder = Instance.new("Folder")
    espFolder.Name = "PlayerESP"
    espFolder.Parent = game.CoreGui
end

-- =========================
-- UTILS
-- =========================
local function getMainColor(plr)
    if LocalPlayer.Team and plr.Team and plr.Team == LocalPlayer.Team then
        return Color3.fromRGB(0, 255, 0)
    end
    return Color3.fromRGB(255, 255, 0)
end

local function getHRP(char)
    return char and char:FindFirstChild("HumanoidRootPart")
end

local function isEnemy(plr)
    if not LocalPlayer.Team or not plr.Team then return true end
    return plr.Team ~= LocalPlayer.Team
end

-- =========================
-- PREDICTION (SEGURA)
-- =========================
local function getPredictedPosition(hrp)
    if not hrp then return nil end
    local hum = hrp.Parent:FindFirstChildOfClass("Humanoid")

    if not PredictionEnabled or not hum or hum.WalkSpeed < 5 then
        return hrp.Position
    end

    return hrp.Position + (hrp.Velocity * PredictionAmount)
end

-- =========================
-- TARGET ACQUISITION (MAX RANGE REAL)
-- =========================
local function getClosestPlayer(lpHRP)
    local closest, closestDist = nil, math.huge

    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and isEnemy(plr) and plr.Character then
            local hum = plr.Character:FindFirstChildOfClass("Humanoid")
            local hrp = getHRP(plr.Character)

            if hum and hum.Health > 0 and hrp then
                local dist = (hrp.Position - lpHRP.Position).Magnitude
                if dist <= MaxRange and dist < closestDist then
                    closestDist = dist
                    closest = hrp
                end
            end
        end
    end

    return closest
end

-- =========================
-- METAMETHOD HOOK (REVALIDA RANGE)
-- =========================
task.spawn(function()
    local mt = getrawmetatable(game)
    setreadonly(mt, false)

    local old
    old = hookmetamethod(game, "__namecall", function(self, ...)
        local args = {...}
        local method = getnamecallmethod():lower()

        if method == "fireserver"
        and SilentAimEnabled
        and TargetPosition
        and typeof(args[1]) == "Vector3" then

            local char = LocalPlayer.Character
            local hrp = getHRP(char)

            if not hrp then
                return old(self, ...)
            end

            -- 🔴 REVALIDAÇÃO FINAL DE DISTÂNCIA
            if (TargetPosition - hrp.Position).Magnitude > MaxRange then
                TargetPosition = nil
                CurrentTargetHRP = nil
                return old(self, ...)
            end

            args[1] = TargetPosition
            return old(self, unpack(args))
        end

        return old(self, ...)
    end)

    setreadonly(mt, true)
end)

-- =========================
-- MAIN LOOP
-- =========================
RunService.RenderStepped:Connect(function()
    local char = LocalPlayer.Character
    local hrp = getHRP(char)
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if not hrp or not hum then return end

    -- SPEED
    if SpeedEnabled then
        hum.WalkSpeed = SpeedValue
    end

    -- ANTISTUN
    if AntiStunEnabled then
        local move = hum.MoveDirection
        if move.Magnitude > 0 then
            hrp.CFrame = hrp.CFrame + move.Unit * AntiStunPower
        end
    end

    -- =========================
    -- SILENT AIM LOGIC (DEFENSIVA)
    -- =========================
    if SilentAimEnabled then
        local targetHRP = getClosestPlayer(hrp)

        if targetHRP then
            CurrentTargetHRP = targetHRP
            TargetPosition = getPredictedPosition(targetHRP)
        else
            -- 🔴 LIMPA IMEDIATAMENTE
            CurrentTargetHRP = nil
            TargetPosition = nil
        end
    else
        CurrentTargetHRP = nil
        TargetPosition = nil
    end

    -- =========================
    -- ESP
    -- =========================
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer then
            if not ESPs[plr] then
                local head = plr.Character and plr.Character:FindFirstChild("Head")
                if head then
                    local gui = Instance.new("BillboardGui")
                    gui.Name = plr.Name
                    gui.Adornee = head
                    gui.Size = UDim2.fromOffset(240, 50)
                    gui.StudsOffset = Vector3.new(0, 3, 0)
                    gui.AlwaysOnTop = true
                    gui.Parent = espFolder

                    local lvl = Instance.new("TextLabel", gui)
                    lvl.Name = "Level"
                    lvl.Size = UDim2.new(1, 0, 0.45, 0)
                    lvl.BackgroundTransparency = 1
                    lvl.Font = Enum.Font.SourceSansBold
                    lvl.TextSize = 13
                    lvl.TextStrokeTransparency = 0.2
                    lvl.TextColor3 = Color3.fromRGB(0, 170, 255)

                    local main = Instance.new("TextLabel", gui)
                    main.Name = "Main"
                    main.Size = UDim2.new(1, 0, 0.55, 0)
                    main.Position = UDim2.new(0, 0, 0.45, 0)
                    main.BackgroundTransparency = 1
                    main.Font = Enum.Font.SourceSansBold
                    main.TextSize = 14
                    main.TextStrokeTransparency = 0.2

                    ESPs[plr] = gui
                end
            end

            local gui = ESPs[plr]
            local pHRP = plr.Character and getHRP(plr.Character)
            local pHum = plr.Character and plr.Character:FindFirstChildOfClass("Humanoid")

            if ESPEnabled and gui and pHRP and pHum then
                gui.Enabled = true
                local dist = math.floor((hrp.Position - pHRP.Position).Magnitude)
                gui.Main.Text = "[" .. math.floor(pHum.Health) .. "] " .. plr.DisplayName .. " (" .. dist .. "m)"
                gui.Main.TextColor3 = getMainColor(plr)
            elseif gui then
                gui.Enabled = false
            end
        end
    end
end)

-- =========================
-- INPUTS
-- =========================
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
