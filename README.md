--[[ 
  Script: ESP + Aimbot com Stealth
  Teclas:
  [P] - Ativar/Desativar ESP
  [L] - Ativar/Desativar Aimbot
--]]

-- CONFIG
local teamCheck = true
local aimPart = "Head"
local aimbotFOV = 150
local aimSmoothness = 0.18 -- quanto menor, mais rápido o aimbot
local espColor = Color3.new(1, 0, 0)

-- VARIÁVEIS
local players = game:GetService("Players")
local localPlayer = players.LocalPlayer
local userInput = game:GetService("UserInputService")
local runService = game:GetService("RunService")
local camera = workspace.CurrentCamera

local espEnabled = true
local aimbotEnabled = false

-- TOGGLES
userInput.InputBegan:Connect(function(input)
    if input.KeyCode == Enum.KeyCode.P then
        espEnabled = not espEnabled
    elseif input.KeyCode == Enum.KeyCode.L then
        aimbotEnabled = not aimbotEnabled
    end
end)

-- DRAWING DE FORMA SEGURA
local function createSafeDrawing(type)
    local obj = Drawing.new(type)
    obj.ZIndex = 2
    return obj
end

-- ESP
local function createESP(player)
    local box = createSafeDrawing("Square")
    box.Color = espColor
    box.Thickness = 1.5
    box.Transparency = 1
    box.Filled = false

    local connection
    connection = runService.RenderStepped:Connect(function()
        if not espEnabled or not player.Character or not player.Character:FindFirstChild("HumanoidRootPart") then
            box.Visible = false
            return
        end

        if teamCheck and player.Team == localPlayer.Team then
            box.Visible = false
            return
        end

        local root = player.Character.HumanoidRootPart
        local pos, onScreen = camera:WorldToViewportPoint(root.Position)

        if onScreen then
            local size = Vector3.new(2, 3, 1.5) * 4
            local topLeft = camera:WorldToViewportPoint((root.CFrame * CFrame.new(-size.X / 2, size.Y / 2, 0)).Position)
            local bottomRight = camera:WorldToViewportPoint((root.CFrame * CFrame.new(size.X / 2, -size.Y / 2, 0)).Position)

            box.Size = Vector2.new(math.abs(topLeft.X - bottomRight.X), math.abs(topLeft.Y - bottomRight.Y))
            box.Position = Vector2.new(math.min(topLeft.X, bottomRight.X), math.min(topLeft.Y, bottomRight.Y))
            box.Visible = true
        else
            box.Visible = false
        end
    end)

    player.AncestryChanged:Connect(function(_, parent)
        if not parent then
            connection:Disconnect()
            box:Remove()
        end
    end)
end

-- AIMBOT - ENCONTRA O INIMIGO MAIS PRÓXIMO DENTRO DO FOV
local function getClosestTarget()
    local closestPlayer = nil
    local shortestDistance = aimbotFOV

    for _, player in pairs(players:GetPlayers()) do
        if player ~= localPlayer and player.Character and player.Character:FindFirstChild(aimPart) then
            if teamCheck and player.Team == localPlayer.Team then continue end

            local partPos, onScreen = camera:WorldToViewportPoint(player.Character[aimPart].Position)
            if onScreen then
                local distance = (Vector2.new(partPos.X, partPos.Y) - userInput:GetMouseLocation()).Magnitude
                if distance < shortestDistance then
                    shortestDistance = distance
                    closestPlayer = player
                end
            end
        end
    end

    return closestPlayer
end

-- MOVIMENTO SUAVE DO MOUSE (simula humano)
local function smoothMouseMove(delta)
    delta = delta + Vector2.new(math.random(-1,1), math.random(-1,1)) -- ruído leve
    mousemoverel(delta.X * aimSmoothness, delta.Y * aimSmoothness)
end

-- LOOP DE AIMBOT
runService.RenderStepped:Connect(function()
    if not aimbotEnabled then return end

    local target = getClosestTarget()
    if target and target.Character and target.Character:FindFirstChild(aimPart) then
        local aimPos = camera:WorldToScreenPoint(target.Character[aimPart].Position)
        local mouse = userInput:GetMouseLocation()
        local delta = Vector2.new(aimPos.X, aimPos.Y) - mouse

        if delta.Magnitude < aimbotFOV then
            smoothMouseMove(delta)
        end
    end
end)

-- MONITORA JOGADORES NOVOS
for _, player in pairs(players:GetPlayers()) do
    if player ~= localPlayer then
        createESP(player)
    end
end

players.PlayerAdded:Connect(function(player)
    if player ~= localPlayer then
        createESP(player)
    end
end)
