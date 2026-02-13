-- CONFIGURAÇÕES GLOBAIS
getgenv().HBE_Enabled = false
getgenv().ESP_Enabled = false
getgenv().WallCheck = true
getgenv().Noclip_Enabled = false
getgenv().Underground_Enabled = false 
getgenv().HitboxSize = 15
getgenv().GhostHitboxSize = 30 -- Tamanho quando estiver no Ghost Mode

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local player = Players.LocalPlayer
local camera = workspace.CurrentCamera
local fakeBody = nil

-- CARREGAR RAYFIELD LIBRARY
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

-- LIMPEZA DE GUIS ANTIGAS
if player.PlayerGui:FindFirstChild(" HRZ (SCRIPT VIP ") then 
    player.PlayerGui.HRZ_COMPATIBLE_V5:Destroy() 
end

-- PLATAFORMA INVISÍVEL
local plat = Instance.new("Part")
plat.Size = Vector3.new(20, 1, 20)
plat.Anchored = true
plat.Transparency = 1
plat.Name = "HRZ_SafePlate"
plat.Parent = workspace

-- INTERFACE PRINCIPAL (RAYFIELD)
local Window = Rayfield:CreateWindow({
   Name = " HRZ (VIP) ",
   LoadingTitle = "Carregando Script...",
   LoadingSubtitle = "By Hrz",
   ConfigurationSaving = {
      Enabled = true,
      FolderName = "HRZ_Configs",
      FileName = "MainConfig"
   },
   KeySystem = false
})

local MainTab = Window:CreateTab("Principal", 4483362458) -- Icone de casa

-- SCREEN GUI PARA O BOTÃO FLUTUANTE
local sg = Instance.new("ScreenGui", player.PlayerGui)
sg.Name = " HRZ (VIP SCRIPT)"
sg.ResetOnSpawn = false

-- TOGGLE FLUTUANTE EXCLUSIVO PARA GHOST
local GhostToggle = Instance.new("TextButton", sg)
GhostToggle.Size = UDim2.new(0, 50, 0, 50)
GhostToggle.Position = UDim2.new(0.85, 0, 0.8, 0)
GhostToggle.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
GhostToggle.Text = "GST\nOFF"
GhostToggle.TextColor3 = Color3.new(1,1,1)
GhostToggle.Font = Enum.Font.GothamBold
GhostToggle.TextSize = 12
GhostToggle.ZIndex = 10
Instance.new("UICorner", GhostToggle).CornerRadius = UDim.new(0, 12)

-- Tornar GhostToggle Arrastável
local draggingGhost, dragInputGhost, dragStartGhost, startPosGhost
GhostToggle.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        draggingGhost = true
        dragStartGhost = input.Position
        startPosGhost = GhostToggle.Position
    end
end)
GhostToggle.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
        dragInputGhost = input
    end
end)
UserInputService.InputChanged:Connect(function(input)
    if input == dragInputGhost and draggingGhost then
        local delta = input.Position - dragStartGhost
        GhostToggle.Position = UDim2.new(startPosGhost.X.Scale, startPosGhost.X.Offset + delta.X, startPosGhost.Y.Scale, startPosGhost.Y.Offset + delta.Y)
    end
end)
GhostToggle.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        draggingGhost = false
    end
end)

-- VARIÁVEIS PARA SINCRONIZAÇÃO
local GhostToggleRayfield

local function UpdateGhostVisual()
    local isEnabled = getgenv().Underground_Enabled
    local colorOn = Color3.fromRGB(0, 255, 150)
    local colorOff = Color3.fromRGB(40, 40, 40)
    
    GhostToggle.BackgroundColor3 = isEnabled and colorOn or colorOff
    GhostToggle.Text = "GST\n" .. (isEnabled and "ON" or "OFF")
    
    -- Sincronizar com Rayfield se o toggle existir
    if GhostToggleRayfield then
        GhostToggleRayfield:Set(isEnabled)
    end
end

local function ToggleGhostLogic()
    local char = player.Character
    if char and char:FindFirstChild("HumanoidRootPart") then
        local hrp = char.HumanoidRootPart
        if getgenv().Underground_Enabled then
            getgenv().WallCheck = false -- Desativa WallCheck para bater debaixo da terra
            char.Archivable = true
            if not fakeBody or not fakeBody.Parent then
                fakeBody = char:Clone()
                fakeBody.Parent = workspace
                for _, p in pairs(fakeBody:GetChildren()) do
                    if p:IsA("BasePart") then p.CanCollide = false p.Transparency = 0.5 end
                end
            end
            fakeBody:MoveTo(hrp.Position)
            hrp.CFrame = hrp.CFrame * CFrame.new(0, -15, 0)
        else
            getgenv().WallCheck = true
            if fakeBody then fakeBody:Destroy(); fakeBody = nil end
            hrp.CFrame = hrp.CFrame * CFrame.new(0, 16, 0)
        end
    end
    UpdateGhostVisual()
end

-- EVENTO DO BOTÃO FLUTUANTE
GhostToggle.MouseButton1Click:Connect(function()
    getgenv().Underground_Enabled = not getgenv().Underground_Enabled
    ToggleGhostLogic()
end)

-- CRIAÇÃO DOS ELEMENTOS NO RAYFIELD
MainTab:CreateToggle({
   Name = "ESP (Highlight)",
   CurrentValue = false,
   Flag = "ESP_Enabled",
   Callback = function(Value)
      getgenv().ESP_Enabled = Value
   end,
})

MainTab:CreateToggle({
   Name = "Hitbox Expander (HBX)",
   CurrentValue = false,
   Flag = "HBE_Enabled",
   Callback = function(Value)
      getgenv().HBE_Enabled = Value
   end,
})

MainTab:CreateToggle({
   Name = "Wall Check",
   CurrentValue = true,
   Flag = "WallCheck",
   Callback = function(Value)
      getgenv().WallCheck = Value
   end,
})

MainTab:CreateToggle({
   Name = "Noclip",
   CurrentValue = false,
   Flag = "Noclip_Enabled",
   Callback = function(Value)
      getgenv().Noclip_Enabled = Value
   end,
})

GhostToggleRayfield = MainTab:CreateToggle({
   Name = "Ghost Mode (GST)",
   CurrentValue = false,
   Flag = "Underground_Enabled",
   Callback = function(Value)
      if getgenv().Underground_Enabled ~= Value then
          getgenv().Underground_Enabled = Value
          ToggleGhostLogic()
      end
   end,
})

MainTab:CreateSlider({
   Name = "Tamanho Hitbox Normal",
   Range = {1, 50},
   Increment = 1,
   Suffix = " Studs",
   CurrentValue = 15,
   Flag = "HitboxSize",
   Callback = function(Value)
      getgenv().HitboxSize = Value
   end,
})

MainTab:CreateSlider({
   Name = "Tamanho Hitbox Ghost",
   Range = {1, 100},
   Increment = 1,
   Suffix = " Studs",
   CurrentValue = 30,
   Flag = "GhostHitboxSize",
   Callback = function(Value)
      getgenv().GhostHitboxSize = Value
   end,
})

-- LÓGICA DE VISIBILIDADE (WALL CHECK)
local function IsVisible(targetPart)
    if not getgenv().WallCheck then return true end
    local origin = camera.CFrame.Position
    local direction = (targetPart.Position - origin)
    local raycastParams = RaycastParams.new()
    local ignoreList = {}
    for _, p in pairs(Players:GetPlayers()) do
        if p.Character then table.insert(ignoreList, p.Character) end
    end
    raycastParams.FilterDescendantsInstances = ignoreList
    raycastParams.FilterType = Enum.RaycastFilterType.Exclude
    local result = workspace:Raycast(origin, direction, raycastParams)
    return result == nil
end

-- LOOP PRINCIPAL
RunService.Stepped:Connect(function()
    local char = player.Character
    if char and char:FindFirstChild("HumanoidRootPart") then
        local hrp = char.HumanoidRootPart
        
        if (getgenv().Noclip_Enabled or getgenv().Underground_Enabled) then
            for _, part in pairs(char:GetDescendants()) do
                if part:IsA("BasePart") then part.CanCollide = false end
            end
        end

        if getgenv().Underground_Enabled then
            plat.CanCollide = true
            plat.CFrame = hrp.CFrame * CFrame.new(0, -3.2, 0)
        else
            plat.CanCollide = false
            plat.Position = Vector3.new(0, -5000, 0)
        end
    end

    -- Lógica de Hitbox Dinâmica
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            local root = p.Character.HumanoidRootPart
            local hum = p.Character:FindFirstChild("Humanoid")
            if hum and hum.Health > 0 then
                local visible = IsVisible(root)
                
                -- CHECAGEM DE HITBOX
                if getgenv().HBE_Enabled and visible then
                    local currentSize = getgenv().Underground_Enabled and getgenv().GhostHitboxSize or getgenv().HitboxSize
                    root.Size = Vector3.new(currentSize, currentSize, currentSize)
                    root.Transparency = 1
                else
                    root.Size = Vector3.new(2, 2, 1)
                end

                -- ESP
                local hl = p.Character:FindFirstChild("HRZ_Highlight")
                if getgenv().ESP_Enabled then
                    if not hl then hl = Instance.new("Highlight", p.Character); hl.Name = "HRZ_Highlight" end
                    hl.FillColor = visible and Color3.new(0, 1, 0) or Color3.new(1, 0, 0)
                elseif hl then hl:Destroy() end
            end
        end
    end
end)

Rayfield:Notify({
   Title = "Script Carregado",
   Content = "Rayfield UI + Ghost Toggle Flutuante ativos!",
   Duration = 5,
   Image = 4483362458,
})
