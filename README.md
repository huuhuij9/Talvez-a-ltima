--[[
    HRZ VIP V11 - SUPER-LITE ON-SCREEN EDITION
    - Foco: Performance Extrema e Segurança Total.
    - Regra de Ouro: HBE e Magnet só ativam se o inimigo estiver VISÍVEL NA TELA.
    - Correção: WallCheck Rigoroso + WorldToViewportPoint.
    - Funções: Hitbox Control, Prediction, FPS Unlocker.
]]

-- ===========================
-- OTIMIZAÇÕES ANTI-FREEZE
-- ===========================
local function AntiFreezeOptimize()
    local workspace = game:GetService("Workspace")
    if workspace:FindFirstChildOfClass("Terrain") then
        workspace.Terrain.WaterWaveSize = 0
    end
    game:GetService("LogService"):ClearOutput()
    task.spawn(function()
        while task.wait(25) do
            collectgarbage("collect")
        end
    end)
end
task.spawn(AntiFreezeOptimize)

-- ============================================================
-- CONFIGURAÇÕES GLOBAIS (getgenv)
-- ============================================================
getgenv().HBE_Enabled = true
getgenv().ESP_Enabled = false
getgenv().WallCheck = true
getgenv().TeamCheck = true 
getgenv().HitboxSize = 10
getgenv().ProximityRange = 55
getgenv().MaxAllies = 4
getgenv().TPDistance = 3 
getgenv().Magnet_Enabled = false

-- Configurações de Precisão (Otimizadas)
getgenv().PredictionAmount = 0.165
getgenv().WallCheckMargin = 0.8

-- Persistência do FPS
getgenv().FPS_Value = 60
getgenv().FPS_Advanced = false
getgenv().FPS_Active = false

-- ===========================
-- VARIÁVEIS DE SERVIÇO & CACHE
-- ===========================
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local Mouse = LocalPlayer:GetMouse()

local Allies = {} 

-- ===========================
-- FUNÇÃO DE VISIBILIDADE TOTAL
-- ===========================
local function IsFullyVisible(targetPart)
    -- 1. Verificar se está na tela (OnScreen)
    local screenPos, onScreen = Camera:WorldToViewportPoint(targetPart.Position)
    if not onScreen then return false end
    
    -- 2. Verificar se há paredes (WallCheck)
    if getgenv().WallCheck then
        local origin = Camera.CFrame.Position
        local targetPos = targetPart.Position
        local direction = (targetPos - origin)
        
        local rayParams = RaycastParams.new()
        rayParams.FilterType = Enum.RaycastFilterType.Exclude
        local ignoreList = {LocalPlayer.Character}
        for _, p in ipairs(Players:GetPlayers()) do
            if p.Character then table.insert(ignoreList, p.Character) end
        end
        rayParams.FilterDescendantsInstances = ignoreList
        
        local result = workspace:Raycast(origin, direction, rayParams)
        if result and result.Instance then
            return false
        end
        
        -- Verificação de bordas (Margem de Segurança)
        local margin = getgenv().WallCheckMargin
        if workspace:Raycast(origin, (targetPos + Vector3.new(margin, 0, 0) - origin), rayParams) then
            return false
        end
    end
    
    return true
end

-- ===========================
-- MAGNETIC BULLET (COM PREDIÇÃO)
-- ===========================
local function GetMagneticTarget()
    if not getgenv().Magnet_Enabled then return nil end
    local char = LocalPlayer.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return nil end
    
    local closestDist = math.huge
    local target = nil
    local myPos = char.HumanoidRootPart.Position
    
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character and not (getgenv().TeamCheck and Allies[p.UserId]) then
            local hum = p.Character:FindFirstChild("Humanoid")
            if hum and hum.Health > 0 then
                local root = p.Character:FindFirstChild("HumanoidRootPart")
                if root then
                    local dist = (root.Position - myPos).Magnitude
                    if dist < closestDist then
                        -- Só seleciona se estiver TOTALMENTE visível (Tela + Parede)
                        if IsFullyVisible(root) then
                            closestDist = dist
                            target = root
                        end
                    end
                end
            end
        end
    end
    
    if target then
        local velocity = target.Velocity
        local predictedCFrame = target.CFrame + (velocity * getgenv().PredictionAmount)
        return {CFrame = predictedCFrame, Instance = target}
    end
    return nil
end

local oldIndex;
oldIndex = hookmetamethod(game, "__index", function(self, key)
    if not checkcaller() and getgenv().Magnet_Enabled and self == Mouse then
        if key == "Target" or key == "Hit" then
            local tData = GetMagneticTarget()
            if tData then 
                return (key == "Hit" and tData.CFrame or tData.Instance) 
            end
        end
    end
    return oldIndex(self, key)
end)

-- ===========================
-- TEAMCHECK & TELEPORTE & LOOP
-- ===========================
local function AutoDetectAllies()
    local char = LocalPlayer.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return end
    local myPos = char.HumanoidRootPart.Position
    local potentialAllies = {}
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            local dist = (p.Character.HumanoidRootPart.Position - myPos).Magnitude
            if dist <= getgenv().ProximityRange then
                table.insert(potentialAllies, {player = p, distance = dist})
            end
        end
    end
    table.sort(potentialAllies, function(a, b) return a.distance < b.distance end)
    local newAllies = {}
    for i = 1, math.min(#potentialAllies, getgenv().MaxAllies) do
        newAllies[potentialAllies[i].player.UserId] = true
    end
    Allies = newAllies
end

LocalPlayer.CharacterAdded:Connect(function()
    task.wait(1.5)
    AutoDetectAllies()
end)

local function TeleportToEnemy()
    local char = LocalPlayer.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return end
    local closestEnemy = nil
    local shortestDist = math.huge
    local myPos = char.HumanoidRootPart.Position
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character and not (getgenv().TeamCheck and Allies[p.UserId]) then
            local hum = p.Character:FindFirstChild("Humanoid")
            if hum and hum.Health > 0 and p.Character:FindFirstChild("HumanoidRootPart") then
                local dist = (p.Character.HumanoidRootPart.Position - myPos).Magnitude
                if dist < shortestDist then
                    closestEnemy = p.Character.HumanoidRootPart
                    shortestDist = dist
                end
            end
        end
    end
    if closestEnemy then
        local targetPos = closestEnemy.CFrame * CFrame.new(0, 0, getgenv().TPDistance)
        char.HumanoidRootPart.CFrame = CFrame.new(targetPos.Position, closestEnemy.Position)
    end
end

RunService.Heartbeat:Connect(function()
    local hbe = getgenv().HBE_Enabled
    local esp = getgenv().ESP_Enabled
    local hSize = getgenv().HitboxSize
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character then
            local root = p.Character:FindFirstChild("HumanoidRootPart")
            local hum = p.Character:FindFirstChild("Humanoid")
            if root and hum and hum.Health > 0 then
                local isEnemy = not (getgenv().TeamCheck and Allies[p.UserId])
                
                -- REGRA DE OURO: HBE só ativa se estiver visível na tela e sem paredes
                if hbe and isEnemy and IsFullyVisible(root) then
                    root.Size = Vector3.new(hSize, hSize, hSize)
                    root.Transparency = 1
                    root.CanCollide = false
                else
                    root.Size = Vector3.new(2, 2, 2)
                    root.CanCollide = true
                end
                
                local hl = p.Character:FindFirstChild("JUMENTAO_HL")
                if esp then
                    if not hl then 
                        hl = Instance.new("Highlight", p.Character)
                        hl.Name = "JUMENTAO_HL" 
                    end
                    hl.FillColor = isEnemy and Color3.new(1, 0, 0) or Color3.new(0, 0.5, 1)
                    hl.Enabled = true
                elseif hl then 
                    hl:Destroy() 
                end
            end
        end
    end
end)

-- ===========================
-- INTERFACE SUPER-LITE (GUI)
-- ===========================
if LocalPlayer.PlayerGui:FindFirstChild("HRZ_SUPER_LITE") then LocalPlayer.PlayerGui.HRZ_SUPER_LITE:Destroy() end
local sg = Instance.new("ScreenGui", LocalPlayer.PlayerGui)
sg.Name = "HRZ_SUPER_LITE"
sg.ResetOnSpawn = false

local function makeDraggable(obj)
    local dragging, dragInput, dragStart, startPos
    obj.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = obj.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then dragging = false end
            end)
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local delta = input.Position - dragStart
            obj.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

local Main = Instance.new("Frame", sg)
Main.Size = UDim2.new(0, 180, 0, 320)
Main.Position = UDim2.new(0.5, -90, 0.5, -160)
Main.BackgroundColor3 = Color3.new(0, 0, 0)
Main.BorderSizePixel = 1
Main.BorderColor3 = Color3.new(0, 1, 1)
Main.Visible = false
makeDraggable(Main)

local Title = Instance.new("TextLabel", Main)
Title.Size = UDim2.new(1, 0, 0, 25)
Title.Text = "HRZ SUPER-LITE"
Title.TextColor3 = Color3.new(0, 1, 1)
Title.TextSize = 14
Title.Font = Enum.Font.SourceSansBold
Title.BackgroundColor3 = Color3.new(0.1, 0.1, 0.1)
Title.BorderSizePixel = 0

local function AddBtn(text, y, var, callback)
    local b = Instance.new("TextButton", Main)
    b.Size = UDim2.new(0.9, 0, 0, 25)
    b.Position = UDim2.new(0.05, 0, 0, y)
    b.BackgroundColor3 = Color3.new(0.1, 0.1, 0.1)
    b.BorderSizePixel = 0
    b.Text = text .. (getgenv()[var] and " [ON]" or " [OFF]")
    b.TextColor3 = getgenv()[var] and Color3.new(0, 1, 0) or Color3.new(1, 0, 0)
    b.Font = Enum.Font.SourceSansBold
    b.TextSize = 11
    b.MouseButton1Click:Connect(function()
        getgenv()[var] = not getgenv()[var]
        b.Text = text .. (getgenv()[var] and " [ON]" or " [OFF]")
        b.TextColor3 = getgenv()[var] and Color3.new(0, 1, 0) or Color3.new(1, 0, 0)
        if callback then callback() end
    end)
end

AddBtn("HITBOX", 35, "HBE_Enabled")
AddBtn("ESP", 65, "ESP_Enabled")
AddBtn("WALLCHECK", 95, "WallCheck")
AddBtn("MAGNET", 125, "Magnet_Enabled")
AddBtn("TEAMCHECK", 155, "TeamCheck", AutoDetectAllies)

local HBELabel = Instance.new("TextLabel", Main)
HBELabel.Size = UDim2.new(1, 0, 0, 20)
HBELabel.Position = UDim2.new(0, 0, 0, 185)
HBELabel.Text = "HITBOX: " .. getgenv().HitboxSize
HBELabel.TextColor3 = Color3.new(1, 1, 1)
HBELabel.Font = Enum.Font.SourceSansBold
HBELabel.TextSize = 11
HBELabel.BackgroundTransparency = 1

local HBE_Minus = Instance.new("TextButton", Main)
HBE_Minus.Size = UDim2.new(0.4, 0, 0, 20)
HBE_Minus.Position = UDim2.new(0.05, 0, 0, 205)
HBE_Minus.BackgroundColor3 = Color3.new(0.1, 0.1, 0.1)
HBE_Minus.Text = "-"
HBE_Minus.TextColor3 = Color3.new(1, 0, 0)
HBE_Minus.Font = Enum.Font.SourceSansBold
HBE_Minus.TextSize = 16
HBE_Minus.BorderSizePixel = 0
HBE_Minus.MouseButton1Click:Connect(function()
    if getgenv().HitboxSize > 1 then
        getgenv().HitboxSize = getgenv().HitboxSize - 1
        HBELabel.Text = "HITBOX: " .. getgenv().HitboxSize
    end
end)

local HBE_Plus = Instance.new("TextButton", Main)
HBE_Plus.Size = UDim2.new(0.4, 0, 0, 20)
HBE_Plus.Position = UDim2.new(0.55, 0, 0, 205)
HBE_Plus.BackgroundColor3 = Color3.new(0.1, 0.1, 0.1)
HBE_Plus.Text = "+"
HBE_Plus.TextColor3 = Color3.new(0, 1, 0)
HBE_Plus.Font = Enum.Font.SourceSansBold
HBE_Plus.TextSize = 16
HBE_Plus.BorderSizePixel = 0
HBE_Plus.MouseButton1Click:Connect(function()
    if getgenv().HitboxSize < 50 then
        getgenv().HitboxSize = getgenv().HitboxSize + 1
        HBELabel.Text = "HITBOX: " .. getgenv().HitboxSize
    end
end)

-- FPS UNLOCKER
task.spawn(function()
    while true do
        if getgenv().FPS_Active then
            local startTime = tick()
            local frameTime = getgenv().FPS_Advanced and (60 / getgenv().FPS_Value) or (1 / getgenv().FPS_Value)
            RunService.Heartbeat:Wait()
            while (tick() - startTime) < frameTime do end
        else
            task.wait(0.5)
        end
    end
end)

local FPSTextBox = Instance.new("TextBox", Main)
FPSTextBox.Size = UDim2.new(0.5, 0, 0, 20)
FPSTextBox.Position = UDim2.new(0.05, 0, 0, 235)
FPSTextBox.PlaceholderText = "FPS"
FPSTextBox.Text = tostring(getgenv().FPS_Value)
FPSTextBox.BackgroundColor3 = Color3.new(0.1, 0.1, 0.1)
FPSTextBox.TextColor3 = Color3.new(1, 1, 1)
FPSTextBox.Font = Enum.Font.SourceSans
FPSTextBox.TextSize = 11
FPSTextBox.BorderSizePixel = 0

local FPSSetBtn = Instance.new("TextButton", Main)
FPSSetBtn.Size = UDim2.new(0.35, 0, 0, 20)
FPSSetBtn.Position = UDim2.new(0.6, 0, 0, 235)
FPSSetBtn.BackgroundColor3 = Color3.new(0.1, 0.1, 0.1)
FPSSetBtn.Text = getgenv().FPS_Active and "OFF" or "SET"
FPSSetBtn.TextColor3 = getgenv().FPS_Active and Color3.new(1, 0, 0) or Color3.new(0, 1, 0)
FPSSetBtn.Font = Enum.Font.SourceSansBold
FPSSetBtn.TextSize = 11
FPSSetBtn.BorderSizePixel = 0
FPSSetBtn.MouseButton1Click:Connect(function()
    if getgenv().FPS_Active then
        getgenv().FPS_Active = false
        FPSSetBtn.Text = "SET"
        FPSSetBtn.TextColor3 = Color3.new(0, 1, 0)
    else
        local val = tonumber(FPSTextBox.Text)
        if val and val > 0 then
            getgenv().FPS_Value = val
            getgenv().FPS_Active = true
            FPSSetBtn.Text = "OFF"
            FPSSetBtn.TextColor3 = Color3.new(1, 0, 0)
        end
    end
end)

local FPSAdvBtn = Instance.new("TextButton", Main)
FPSAdvBtn.Size = UDim2.new(0.9, 0, 0, 20)
FPSAdvBtn.Position = UDim2.new(0.05, 0, 0, 260)
FPSAdvBtn.BackgroundColor3 = Color3.new(0.1, 0.1, 0.1)
FPSAdvBtn.Text = getgenv().FPS_Advanced and "MODE: FPM" or "MODE: FPS"
FPSAdvBtn.TextColor3 = Color3.new(0, 1, 1)
FPSAdvBtn.Font = Enum.Font.SourceSans
FPSAdvBtn.TextSize = 10
FPSAdvBtn.BorderSizePixel = 0
FPSAdvBtn.MouseButton1Click:Connect(function()
    getgenv().FPS_Advanced = not getgenv().FPS_Advanced
    FPSAdvBtn.Text = getgenv().FPS_Advanced and "MODE: FPM" or "MODE: FPS"
end)

local TBtn = Instance.new("TextButton", sg)
TBtn.Size = UDim2.new(0, 40, 0, 40)
TBtn.Position = UDim2.new(0.1, 0, 0.1, 0)
TBtn.BackgroundColor3 = Color3.new(0, 0, 0)
TBtn.BorderSizePixel = 1
TBtn.BorderColor3 = Color3.new(0, 1, 1)
TBtn.Text = "MENU"
TBtn.TextColor3 = Color3.new(0, 1, 1)
TBtn.Font = Enum.Font.SourceSansBold
TBtn.TextSize = 11
makeDraggable(TBtn)
TBtn.MouseButton1Click:Connect(function() Main.Visible = not Main.Visible end)

local TPBtn = Instance.new("TextButton", sg)
TPBtn.Size = UDim2.new(0, 40, 0, 40)
TPBtn.Position = UDim2.new(0.9, 0, 0.25, 0)
TPBtn.BackgroundColor3 = Color3.new(0, 0, 0)
TPBtn.BorderSizePixel = 1
TPBtn.BorderColor3 = Color3.new(0, 1, 1)
TPBtn.Text = "TP"
TPBtn.TextColor3 = Color3.new(0, 1, 1)
TPBtn.Font = Enum.Font.SourceSansBold
TPBtn.TextSize = 11
makeDraggable(TPBtn)
TPBtn.MouseButton1Click:Connect(TeleportToEnemy)

task.spawn(AutoDetectAllies)
