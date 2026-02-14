-- CONFIGURAÇÕES GLOBAIS
getgenv().HBE_Enabled = false
getgenv().ESP_Enabled = false
getgenv().WallCheck = true
getgenv().Underground_Enabled = false 
getgenv().Pull_Enabled = false -- Nova configuração para Pull
getgenv().HitboxSize = 15
getgenv().GhostHitboxSize = 30 -- Tamanho quando estiver no Ghost Mode

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local player = Players.LocalPlayer
local camera = workspace.CurrentCamera
local fakeBody = nil

-- CONFIGURAÇÕES PULL
local PULL_DISTANCE = 2
local LERP_SPEED = 0.3

-- LIMPEZA DE GUIS ANTIGAS
if player.PlayerGui:FindFirstChild("HRZ_COMPATIBLE_V5") then 
    player.PlayerGui.HRZ_COMPATIBLE_V5:Destroy() 
end

-- PLATAFORMA INVISÍVEL
local plat = Instance.new("Part")
plat.Size = Vector3.new(20, 1, 20)
plat.Anchored = true
plat.Transparency = 1
plat.Name = "HRZ_SafePlate"
plat.Parent = workspace

-- INTERFACE
local sg = Instance.new("ScreenGui", player.PlayerGui)
sg.Name = "HRZ_COMPATIBLE_V5"
sg.ResetOnSpawn = false

local SideFrame = Instance.new("Frame", sg)
SideFrame.Size = UDim2.new(0, 55, 0, 310)
SideFrame.Position = UDim2.new(0, 10, 0.3, 0)
SideFrame.BackgroundTransparency = 0.8
SideFrame.BackgroundColor3 = Color3.new(0,0,0)
SideFrame.Active = true
SideFrame.Draggable = true 

Instance.new("UICorner", SideFrame).CornerRadius = UDim.new(0, 10)

local layout = Instance.new("UIListLayout", SideFrame)
layout.Padding = UDim.new(0, 8)
layout.HorizontalAlignment = "Center"
layout.VerticalAlignment = "Center"

local function CreateToggle(name, varName, colorOn, specialCallback)
    local btn = Instance.new("TextButton", SideFrame)
    btn.Size = UDim2.new(0, 45, 0, 45)
    btn.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    btn.Text = name
    btn.TextColor3 = Color3.new(1,1,1)
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 10
    
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 10)

    local function UpdateVisual()
        btn.BackgroundColor3 = getgenv()[varName] and colorOn or Color3.fromRGB(40, 40, 40)
        btn.Text = name .. "\n" .. (getgenv()[varName] and "ON" or "OFF")
    end
    
    UpdateVisual()

    btn.MouseButton1Click:Connect(function()
        getgenv()[varName] = not getgenv()[varName]
        UpdateVisual()
        if specialCallback then specialCallback() end
    end)
end

local function ToggleGhost()
    local char = player.Character
    if char and char:FindFirstChild("HumanoidRootPart") then
        local hrp = char.HumanoidRootPart
        if getgenv().Underground_Enabled then
            getgenv().WallCheck = false -- Desativa WallCheck para bater debaixo da terra
            char.Archivable = true
            fakeBody = char:Clone()
            fakeBody.Parent = workspace
            fakeBody:MoveTo(hrp.Position)
            for _, p in pairs(fakeBody:GetChildren()) do
                if p:IsA("BasePart") then p.CanCollide = false p.Transparency = 0.5 end
            end
            hrp.CFrame = hrp.CFrame * CFrame.new(0, -15, 0)
        else
            getgenv().WallCheck = true
            if fakeBody then fakeBody:Destroy() end
            hrp.CFrame = hrp.CFrame * CFrame.new(0, 16, 0)
        end
    end
end

CreateToggle("ESP", "ESP_Enabled", Color3.fromRGB(0, 170, 255))
CreateToggle("HBX", "HBE_Enabled", Color3.fromRGB(255, 170, 0))
CreateToggle("WALL", "WallCheck", Color3.fromRGB(170, 0, 255))
CreateToggle("PULL", "Pull_Enabled", Color3.fromRGB(255, 100, 100)) -- Novo Toggle Pull
CreateToggle("GST", "Underground_Enabled", Color3.fromRGB(0, 255, 150), ToggleGhost)

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

-- LÓGICA PULL (GET TARGET)
local function getPullTarget()
    local closest = nil
    local shortestDist = 250
    local mouse = UserInputService:GetMouseLocation()

    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= player and p.Character then
            local root = p.Character:FindFirstChild("HumanoidRootPart")
            local hum = p.Character:FindFirstChild("Humanoid")
            if root and hum and hum.Health > 0 then
                local pos, onScreen = camera:WorldToViewportPoint(root.Position)
                if onScreen then
                    local mag = (Vector2.new(pos.X, pos.Y) - mouse).Magnitude
                    if mag < shortestDist then
                        closest = root
                        shortestDist = mag
                    end
                end
            end
        end
    end
    return closest
end

-- LOOP PRINCIPAL
RunService.Stepped:Connect(function()
    local char = player.Character
    if char and char:FindFirstChild("HumanoidRootPart") then
        local hrp = char.HumanoidRootPart
        
        -- Noclip removido, mantendo apenas para Underground
        if getgenv().Underground_Enabled then
            for _, part in pairs(char:GetDescendants()) do
                if part:IsA("BasePart") then part.CanCollide = false end
            end
            plat.CanCollide = true
            plat.CFrame = hrp.CFrame * CFrame.new(0, -3.2, 0)
        else
            plat.CanCollide = false
            plat.Position = Vector3.new(0, -5000, 0)
        end
    end

    -- Lógica de Pull
    if getgenv().Pull_Enabled then
        local myChar = player.Character
        local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
        if myRoot then
            local targetRoot = getPullTarget()
            if targetRoot then
                local targetPos = myRoot.CFrame.Position + (myRoot.CFrame.LookVector * PULL_DISTANCE)
                targetRoot.CFrame = targetRoot.CFrame:Lerp(CFrame.new(targetPos, myRoot.Position), LERP_SPEED)
            end
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
