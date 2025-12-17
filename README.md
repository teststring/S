local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local player = Players.LocalPlayer

local speedEnabled = false
local antiStunEnabled = false

local speedValue = 700
local antiStunConn

local function applySpeed()
    local char = player.Character
    if char then
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then
            hum.WalkSpeed = speedEnabled and speedValue or 16
        end
    end
end

local function enableAntiStun()
    if antiStunConn then return end
    antiStunConn = RunService.RenderStepped:Connect(function()
        local char = player.Character
        if not char then return end

        local hum = char:FindFirstChildOfClass("Humanoid")
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if not hum or not hrp then return end

        local move = hum.MoveDirection
        if move.Magnitude > 0 then
            hrp.CFrame = hrp.CFrame + move.Unit * 1.2
        end
    end)
end

local function disableAntiStun()
    if antiStunConn then
        antiStunConn:Disconnect()
        antiStunConn = nil
    end
end

RunService.RenderStepped:Connect(function()
    if speedEnabled then
        applySpeed()
    end
end)

UserInputService.InputBegan:Connect(function(input, gpe)
    if gpe then return end

    if input.KeyCode == Enum.KeyCode.P then
        antiStunEnabled = not antiStunEnabled
        if antiStunEnabled then
            enableAntiStun()
        else
            disableAntiStun()
        end

    elseif input.KeyCode == Enum.KeyCode.K then
        local state = not (speedEnabled and antiStunEnabled)
        speedEnabled = state
        antiStunEnabled = state

        applySpeed()

        if antiStunEnabled then
            enableAntiStun()
        else
            disableAntiStun()
        end
    end
end)
