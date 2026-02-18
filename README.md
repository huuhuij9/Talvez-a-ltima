--[[
    PAINEL JUMENTÃO - V4.4 (TEAMCHECK EDITION)
    Criado por: Manus AI
    Tema: Vermelho e Preto
    NOVIDADE: TeamCheck Automático por Proximidade (Lógica v15 integrada).
    Layout: Botão ⚡ fixado no TOPO do menu vertical.
]]

local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- CONFIGURAÇÕES GLOBAIS
getgenv().HBE_Enabled = false
getgenv().ESP_Enabled = false
getgenv().WallCheck = true
getgenv().TeamCheck = true 
getgenv().Pull_Enabled = false
getgenv().Underground_Enabled = false
getgenv().HitboxSize = 15
getgenv().GhostHitboxSize = 30
getgenv().ProximityRange = 50 -- Range para detectar aliados no spawn
getgenv().MaxAllies = 4 -- Limite de aliados

local Allies = {} -- Tabela de aliados detectados
local fakeBody = nil
local PULL_DISTANCE = 2
local LERP_SPEED = 0.3

-- PLATAFORMA INVISÍVEL PARA UNDER GHOST
local plat = Instance.new("Part")
plat.Size = Vector3.new(20, 1, 20)
plat.Anchored = true
plat.Transparency = 1
plat.Name = "Jumentao_SafePlate"
plat.Parent = workspace

-- Cores do Tema
local MainColor = Color3.fromRGB(200, 0, 0)
local BgColor = Color3.fromRGB(15, 15, 15)
local SidebarColor = Color3.fromRGB(25, 25, 25)
local TextColor = Color3.fromRGB(255, 255, 255)

-- ScreenGui Principal
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "PainelJumentaoV4_4"
ScreenGui.Parent = LocalPlayer:WaitForChild("PlayerGui")
ScreenGui.ResetOnSpawn = false
ScreenGui.DisplayOrder = 1000

-- Funções Utilitárias
local function CreateTween(obj, info, goal)
    return TweenService:Create(obj, info, goal)
end

local function Notify(title, text, duration)
    local NotificationFrame = Instance.new("Frame")
    NotificationFrame.Parent = ScreenGui
    NotificationFrame.BackgroundColor3 = BgColor
    NotificationFrame.Position = UDim2.new(1, 10, 0.8, 0)
    NotificationFrame.Size = UDim2.new(0, 220, 0, 70)
    NotificationFrame.ZIndex = 2000
    local UICorner = Instance.new("UICorner")
    UICorner.CornerRadius = UDim.new(0, 8)
    UICorner.Parent = NotificationFrame
    local UIStroke = Instance.new("UIStroke")
    UIStroke.Color = MainColor
    UIStroke.Thickness = 2
    UIStroke.Parent = NotificationFrame
    local TitleLabel = Instance.new("TextLabel")
    TitleLabel.Parent = NotificationFrame
    TitleLabel.BackgroundTransparency = 1
    TitleLabel.Position = UDim2.new(0, 10, 0, 5)
    TitleLabel.Size = UDim2.new(1, -20, 0, 20)
    TitleLabel.Font = Enum.Font.GothamBold
    TitleLabel.Text = title
    TitleLabel.TextColor3 = MainColor
    TitleLabel.TextSize = 14
    TitleLabel.TextXAlignment = Enum.TextXAlignment.Left
    TitleLabel.ZIndex = 2001
    local TextLabel = Instance.new("TextLabel")
    TextLabel.Parent = NotificationFrame
    TextLabel.BackgroundTransparency = 1
    TextLabel.Position = UDim2.new(0, 10, 0, 25)
    TextLabel.Size = UDim2.new(1, -20, 0, 40)
    TextLabel.Font = Enum.Font.Gotham
    TextLabel.Text = text
    TextLabel.TextColor3 = TextColor
    TextLabel.TextSize = 12
    TextLabel.TextWrapped = true
    TextLabel.TextXAlignment = Enum.TextXAlignment.Left
    TextLabel.ZIndex = 2001
    CreateTween(NotificationFrame, TweenInfo.new(0.5, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Position = UDim2.new(1, -230, 0.8, 0)}):Play()
    task.wait(duration or 3)
    local exit = CreateTween(NotificationFrame, TweenInfo.new(0.5, Enum.EasingStyle.Back, Enum.EasingDirection.In), {Position = UDim2.new(1, 10, 0.8, 0)})
    exit:Play()
    exit.Completed:Connect(function() NotificationFrame:Destroy() end)
end

local function MakeDraggable(obj, dragTarget)
    local dragging, dragInput, dragStart, startPos
    local target = dragTarget or obj
    obj.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = target.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then dragging = false end
            end)
        end
    end)
    obj.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then dragInput = input end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            local delta = input.Position - dragStart
            target.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

-- ===========================
-- LÓGICA DE TEAMCHECK INTELIGENTE
-- ===========================

local function AutoDetectAllies()
    local char = LocalPlayer.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return end
    
    local myPos = char.HumanoidRootPart.Position
    local potentialAllies = {}
    
    for _, p in pairs(Players:GetPlayers()) do
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
        local p = potentialAllies[i].player
        newAllies[p.UserId] = true
    end
    
    Allies = newAllies
    Notify("TeamCheck", "Aliados detectados e salvos!", 2)
end

LocalPlayer.CharacterAdded:Connect(function()
    task.wait(1.5)
    AutoDetectAllies()
end)

local function IsEnemy(otherPlayer)
    if not getgenv().TeamCheck then return true end
    return not Allies[otherPlayer.UserId]
end

-- ===========================
-- PAINEL PRINCIPAL
-- ===========================
local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Parent = ScreenGui
MainFrame.BackgroundColor3 = BgColor
MainFrame.Position = UDim2.new(0.5, -250, 0.5, -150)
MainFrame.Size = UDim2.new(0, 500, 0, 300)
MainFrame.ClipsDescendants = true
MainFrame.Visible = false
MainFrame.ZIndex = 100
local MainCorner = Instance.new("UICorner")
MainCorner.CornerRadius = UDim.new(0, 12)
MainCorner.Parent = MainFrame
local MainStroke = Instance.new("UIStroke")
MainStroke.Color = MainColor
MainStroke.Thickness = 1.5
MainStroke.Parent = MainFrame

local TopBar = Instance.new("Frame")
TopBar.Name = "TopBar"
TopBar.Parent = MainFrame
TopBar.BackgroundColor3 = SidebarColor
TopBar.Size = UDim2.new(1, 0, 0, 40)
TopBar.ZIndex = 101
local TopCorner = Instance.new("UICorner")
TopCorner.CornerRadius = UDim.new(0, 12)
TopCorner.Parent = TopBar
local Title = Instance.new("TextLabel")
Title.Parent = TopBar
Title.BackgroundTransparency = 1
Title.Position = UDim2.new(0, 15, 0, 0)
Title.Size = UDim2.new(0, 200, 1, 0)
Title.Font = Enum.Font.GothamBold
Title.Text = "PAINEL JUMENTÃO V4.4"
Title.TextColor3 = MainColor
Title.TextSize = 16
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.ZIndex = 102

MakeDraggable(TopBar, MainFrame)

local Sidebar = Instance.new("Frame")
Sidebar.Parent = MainFrame
Sidebar.BackgroundColor3 = SidebarColor
Sidebar.Position = UDim2.new(0, 0, 0, 40)
Sidebar.Size = UDim2.new(0, 140, 1, -40)
Sidebar.ZIndex = 101
local ContentArea = Instance.new("Frame")
ContentArea.Parent = MainFrame
ContentArea.BackgroundTransparency = 1
ContentArea.Position = UDim2.new(0, 150, 0, 50)
ContentArea.Size = UDim2.new(1, -160, 1, -60)
ContentArea.ZIndex = 101
local TabContainer = Instance.new("ScrollingFrame")
TabContainer.Parent = Sidebar
TabContainer.BackgroundTransparency = 1
TabContainer.Size = UDim2.new(1, 0, 1, 0)
TabContainer.ScrollBarThickness = 0
TabContainer.ZIndex = 102
local TabList = Instance.new("UIListLayout")
TabList.Parent = TabContainer
TabList.Padding = UDim.new(0, 5)

-- ===========================
-- TOGGLE FLUTUANTE (BOTÃO NO TOPO)
-- ===========================

local ToggleHolder = Instance.new("Frame")
ToggleHolder.Name = "ToggleHolder"
ToggleHolder.Parent = ScreenGui
ToggleHolder.BackgroundTransparency = 1
ToggleHolder.Position = UDim2.new(0.9, 0, 0.4, 0)
ToggleHolder.Size = UDim2.new(0, 60, 0, 60)
ToggleHolder.ZIndex = 500

local ToggleBtn = Instance.new("TextButton")
ToggleBtn.Name = "ToggleBtn"
ToggleBtn.Parent = ToggleHolder
ToggleBtn.BackgroundColor3 = MainColor
ToggleBtn.Size = UDim2.new(1, 0, 1, 0)
ToggleBtn.Text = "⚡"
ToggleBtn.TextColor3 = TextColor
ToggleBtn.Font = Enum.Font.GothamBold
ToggleBtn.TextSize = 28
ToggleBtn.ZIndex = 510
local ToggleCorner = Instance.new("UICorner")
ToggleCorner.CornerRadius = UDim.new(1, 0)
ToggleCorner.Parent = ToggleBtn
local ToggleStroke = Instance.new("UIStroke")
ToggleStroke.Color = TextColor
ToggleStroke.Thickness = 2
ToggleStroke.Parent = ToggleBtn

local QuickMenu = Instance.new("Frame")
QuickMenu.Name = "QuickMenu"
QuickMenu.Parent = ToggleHolder
QuickMenu.BackgroundColor3 = BgColor
QuickMenu.Position = UDim2.new(0.5, -100, 0, 70)
QuickMenu.Size = UDim2.new(0, 200, 0, 0)
QuickMenu.ClipsDescendants = true
QuickMenu.Visible = false
QuickMenu.ZIndex = 500
local QuickCorner = Instance.new("UICorner")
QuickCorner.CornerRadius = UDim.new(0, 10)
QuickCorner.Parent = QuickMenu
local QuickStroke = Instance.new("UIStroke")
QuickStroke.Color = MainColor
QuickStroke.Thickness = 2
QuickStroke.Parent = QuickMenu

local QuickList = Instance.new("UIListLayout")
QuickList.Parent = QuickMenu
QuickList.Padding = UDim.new(0, 5)
QuickList.HorizontalAlignment = Enum.HorizontalAlignment.Center
QuickList.SortOrder = Enum.SortOrder.LayoutOrder

local SyncTable = {}

local function CreateQuickToggle(text, icon, varName, callback)
    local btn = Instance.new("TextButton")
    btn.Parent = QuickMenu
    btn.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    btn.Size = UDim2.new(1, -10, 0, 35)
    btn.Text = ""
    btn.ZIndex = 501
    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 6)
    btnCorner.Parent = btn
    
    local iconLabel = Instance.new("TextLabel")
    iconLabel.Parent = btn
    iconLabel.BackgroundTransparency = 1
    iconLabel.Position = UDim2.new(0, 5, 0, 0)
    iconLabel.Size = UDim2.new(0, 25, 1, 0)
    iconLabel.Font = Enum.Font.GothamBold
    iconLabel.Text = icon
    iconLabel.TextColor3 = MainColor
    iconLabel.TextSize = 16
    iconLabel.ZIndex = 502
    
    local label = Instance.new("TextLabel")
    label.Parent = btn
    label.BackgroundTransparency = 1
    label.Position = UDim2.new(0, 35, 0, 0)
    label.Size = UDim2.new(1, -60, 1, 0)
    label.Font = Enum.Font.Gotham
    label.Text = text
    label.TextColor3 = TextColor
    label.TextSize = 11
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.ZIndex = 502
    
    local status = Instance.new("Frame")
    status.Parent = btn
    status.BackgroundColor3 = getgenv()[varName] and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 0, 0)
    status.Position = UDim2.new(1, -15, 0.5, -5)
    status.Size = UDim2.new(0, 10, 0, 10)
    status.ZIndex = 502
    local sCorner = Instance.new("UICorner")
    sCorner.CornerRadius = UDim.new(1, 0)
    sCorner.Parent = status
    
    SyncTable[varName] = function(val)
        status.BackgroundColor3 = val and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 0, 0)
    end
    
    btn.MouseButton1Click:Connect(function()
        getgenv()[varName] = not getgenv()[varName]
        local active = getgenv()[varName]
        status.BackgroundColor3 = active and Color3.fromRGB(0, 255, 0) or Color3.fromRGB(255, 0, 0)
        if callback then callback(active) end
        Notify(text, active and "Ativado" or "Desativado", 1)
    end)
end

local function ToggleGhost()
    local char = LocalPlayer.Character
    if char and char:FindFirstChild("HumanoidRootPart") then
        local hrp = char.HumanoidRootPart
        if getgenv().Underground_Enabled then
            getgenv().WallCheck = false
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

CreateQuickToggle("Hitbox", "🎯", "HBE_Enabled")
CreateQuickToggle("ESP", "👁️", "ESP_Enabled")
CreateQuickToggle("Pull", "🧲", "Pull_Enabled")
CreateQuickToggle("Under Ghost", "👻", "Underground_Enabled", ToggleGhost)
CreateQuickToggle("Team Check", "👥", "TeamCheck")

local OpenFullBtn = Instance.new("TextButton")
OpenFullBtn.Parent = QuickMenu
OpenFullBtn.BackgroundColor3 = MainColor
OpenFullBtn.Size = UDim2.new(1, -10, 0, 35)
OpenFullBtn.Font = Enum.Font.GothamBold
OpenFullBtn.Text = "ABRIR PAINEL"
OpenFullBtn.TextColor3 = TextColor
OpenFullBtn.TextSize = 12
OpenFullBtn.ZIndex = 501
local bCorner = Instance.new("UICorner")
bCorner.CornerRadius = UDim.new(0, 6)
bCorner.Parent = OpenFullBtn

OpenFullBtn.MouseButton1Click:Connect(function()
    MainFrame.Visible = not MainFrame.Visible
    if MainFrame.Visible then
        MainFrame:TweenSize(UDim2.new(0, 500, 0, 300), "Out", "Back", 0.4, true)
        for _, p in pairs(ContentArea:GetChildren()) do if p:IsA("ScrollingFrame") then p.Visible = false end end
        if ContentArea:FindFirstChild("CombatePage") then ContentArea.CombatePage.Visible = true end
    end
end)

local quickOpen = false
ToggleBtn.MouseButton1Click:Connect(function()
    quickOpen = not quickOpen
    if quickOpen then
        QuickMenu.Visible = true
        CreateTween(QuickMenu, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Size = UDim2.new(0, 200, 0, 250)}):Play()
        CreateTween(ToggleBtn, TweenInfo.new(0.3), {Rotation = 180}):Play()
    else
        local t = CreateTween(QuickMenu, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.In), {Size = UDim2.new(0, 200, 0, 0)})
        t:Play()
        CreateTween(ToggleBtn, TweenInfo.new(0.3), {Rotation = 0}):Play()
        t.Completed:Connect(function() if not quickOpen then QuickMenu.Visible = false end end)
    end
end)

MakeDraggable(ToggleBtn, ToggleHolder)

-- ===========================
-- CRIAÇÃO DE UI DO PAINEL
-- ===========================

local function CreateTab(name, isDefault)
    local btn = Instance.new("TextButton")
    btn.Parent = TabContainer
    btn.BackgroundColor3 = isDefault and MainColor or Color3.fromRGB(35, 35, 35)
    btn.Size = UDim2.new(1, -10, 0, 35)
    btn.Font = Enum.Font.GothamSemibold
    btn.Text = name
    btn.TextColor3 = TextColor
    btn.TextSize = 13
    btn.ZIndex = 103
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = btn
    
    local page = Instance.new("ScrollingFrame")
    page.Name = name .. "Page"
    page.Parent = ContentArea
    page.BackgroundTransparency = 1
    page.Size = UDim2.new(1, 0, 1, 0)
    page.Visible = isDefault
    page.ScrollBarThickness = 2
    page.ZIndex = 103
    local layout = Instance.new("UIListLayout")
    layout.Parent = page
    layout.Padding = UDim.new(0, 10)
    
    btn.MouseButton1Click:Connect(function()
        for _, p in pairs(ContentArea:GetChildren()) do if p:IsA("ScrollingFrame") then p.Visible = false end end
        for _, b in pairs(TabContainer:GetChildren()) do if b:IsA("TextButton") then b.BackgroundColor3 = Color3.fromRGB(35, 35, 35) end end
        page.Visible = true
        btn.BackgroundColor3 = MainColor
    end)
    return page
end

local function CreateToggle(parent, text, varName, callback)
    local frame = Instance.new("Frame")
    frame.Parent = parent
    frame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    frame.Size = UDim2.new(1, 0, 0, 40)
    frame.ZIndex = 104
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = frame
    local label = Instance.new("TextLabel")
    label.Parent = frame
    label.BackgroundTransparency = 1
    label.Position = UDim2.new(0, 10, 0, 0)
    label.Size = UDim2.new(0.7, 0, 1, 0)
    label.Font = Enum.Font.Gotham
    label.Text = text
    label.TextColor3 = TextColor
    label.TextSize = 14
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.ZIndex = 105
    local btn = Instance.new("TextButton")
    btn.Parent = frame
    btn.BackgroundColor3 = getgenv()[varName] and MainColor or Color3.fromRGB(50, 50, 50)
    btn.Position = UDim2.new(1, -50, 0.5, -10)
    btn.Size = UDim2.new(0, 40, 0, 20)
    btn.Text = ""
    btn.ZIndex = 105
    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(1, 0)
    btnCorner.Parent = btn
    local indicator = Instance.new("Frame")
    indicator.Parent = btn
    indicator.BackgroundColor3 = TextColor
    indicator.Position = getgenv()[varName] and UDim2.new(1, -18, 0.5, -8) or UDim2.new(0, 2, 0.5, -8)
    indicator.Size = UDim2.new(0, 16, 0, 16)
    indicator.ZIndex = 106
    local indCorner = Instance.new("UICorner")
    indCorner.CornerRadius = UDim.new(1, 0)
    indCorner.Parent = indicator
    
    btn.MouseButton1Click:Connect(function()
        getgenv()[varName] = not getgenv()[varName]
        local active = getgenv()[varName]
        local goal = active and UDim2.new(1, -18, 0.5, -8) or UDim2.new(0, 2, 0.5, -8)
        local color = active and MainColor or Color3.fromRGB(50, 50, 50)
        CreateTween(indicator, TweenInfo.new(0.3), {Position = goal}):Play()
        CreateTween(btn, TweenInfo.new(0.3), {BackgroundColor3 = color}):Play()
        if callback then callback(active) end
        if SyncTable[varName] then SyncTable[varName](active) end
    end)
end

local function CreateSlider(parent, text, min, max, default, varName, callback)
    local frame = Instance.new("Frame")
    frame.Parent = parent
    frame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    frame.Size = UDim2.new(1, 0, 0, 50)
    frame.ZIndex = 104
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = frame
    local label = Instance.new("TextLabel")
    label.Parent = frame
    label.BackgroundTransparency = 1
    label.Position = UDim2.new(0, 10, 0, 5)
    label.Size = UDim2.new(1, -20, 0, 20)
    label.Font = Enum.Font.Gotham
    label.Text = text .. ": " .. default
    label.TextColor3 = TextColor
    label.TextSize = 12
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.ZIndex = 105
    local sliderBg = Instance.new("Frame")
    sliderBg.Parent = frame
    sliderBg.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    sliderBg.Position = UDim2.new(0, 10, 0, 30)
    sliderBg.Size = UDim2.new(1, -20, 0, 6)
    sliderBg.ZIndex = 105
    local sCorner = Instance.new("UICorner")
    sCorner.CornerRadius = UDim.new(1, 0)
    sCorner.Parent = sliderBg
    local sliderFill = Instance.new("Frame")
    sliderFill.Parent = sliderBg
    sliderFill.BackgroundColor3 = MainColor
    sliderFill.Size = UDim2.new((default - min) / (max - min), 0, 1, 0)
    sliderFill.ZIndex = 106
    local fCorner = Instance.new("UICorner")
    fCorner.CornerRadius = UDim.new(1, 0)
    fCorner.Parent = sliderFill
    
    local dragging = false
    local function UpdateSlider(input)
        local pos = math.clamp((input.Position.X - sliderBg.AbsolutePosition.X) / sliderBg.AbsoluteSize.X, 0, 1)
        sliderFill.Size = UDim2.new(pos, 0, 1, 0)
        local val = math.floor(min + (max - min) * pos)
        label.Text = text .. ": " .. val
        getgenv()[varName] = val
        if callback then callback(val) end
    end
    sliderBg.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            UpdateSlider(input)
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            UpdateSlider(input)
        end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)
end

local CombatTab = CreateTab("Combate", true)
local VisualTab = CreateTab("Visual", false)
local MiscTab = CreateTab("Outros", false)

CreateToggle(CombatTab, "Ativar Hitbox (HBX)", "HBE_Enabled")
CreateSlider(CombatTab, "Tamanho da Hitbox", 2, 50, 15, "HitboxSize")
CreateToggle(CombatTab, "Under Ghost", "Underground_Enabled", ToggleGhost)
CreateToggle(CombatTab, "Pull Inimigos", "Pull_Enabled")
CreateToggle(CombatTab, "Wall Check", "WallCheck")
CreateToggle(CombatTab, "Team Check", "TeamCheck")
CreateToggle(VisualTab, "Ativar ESP", "ESP_Enabled")

local function CreateButton(parent, text, callback)
    local btn = Instance.new("TextButton")
    btn.Parent = parent
    btn.BackgroundColor3 = MainColor
    btn.Size = UDim2.new(1, 0, 0, 40)
    btn.Font = Enum.Font.GothamBold
    btn.Text = text
    btn.TextColor3 = TextColor
    btn.TextSize = 14
    btn.ZIndex = 105
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = btn
    btn.MouseButton1Click:Connect(callback)
end

CreateButton(CombatTab, "Resetar Aliados", function()
    Allies = {}
    AutoDetectAllies()
    Notify("Aliados", "Resetados e Redetectados!", 2)
end)

CreateButton(MiscTab, "Rejoin Game", function()
    game:GetService("TeleportService"):Teleport(game.PlaceId, LocalPlayer)
end)

local function CreateTopBtn(text, pos, color, callback)
    local btn = Instance.new("TextButton")
    btn.Parent = TopBar
    btn.BackgroundColor3 = color
    btn.Size = UDim2.new(0, 30, 0, 30)
    btn.AnchorPoint = Vector2.new(0, 0.5)
    btn.Position = UDim2.new(1, pos.X.Offset, 0.5, 0)
    btn.Font = Enum.Font.GothamBold
    btn.Text = text
    btn.TextColor3 = TextColor
    btn.TextSize = 14
    btn.ZIndex = 105
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = btn
    btn.MouseButton1Click:Connect(callback)
    return btn
end

CreateTopBtn("X", UDim2.new(0, -40, 0, 0), Color3.fromRGB(150, 0, 0), function() ScreenGui:Destroy() end)
CreateTopBtn("-", UDim2.new(0, -75, 0, 0), Color3.fromRGB(40, 40, 40), function()
    MainFrame:TweenSize(UDim2.new(0, 0, 0, 0), "Out", "Quad", 0.3, true)
    task.wait(0.3)
    MainFrame.Visible = false
end)

-- ===========================
-- LÓGICA DE JOGO (LOOP v4.4)
-- ===========================

local function getPullTarget()
    local closest = nil
    local shortestDist = 250
    local mouse = UserInputService:GetMouseLocation()
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character and IsEnemy(p) then -- Apenas inimigos
            local root = p.Character:FindFirstChild("HumanoidRootPart")
            local hum = p.Character:FindFirstChild("Humanoid")
            if root and hum and hum.Health > 0 then
                local pos, onScreen = Camera:WorldToViewportPoint(root.Position)
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

local function IsVisible(targetPart, targetChar)
    if not getgenv().WallCheck then return true end
    local origin = Camera.CFrame.Position
    local direction = (targetPart.Position - origin)
    local distance = direction.Magnitude
    local raycastParams = RaycastParams.new()
    raycastParams.FilterDescendantsInstances = {LocalPlayer.Character}
    raycastParams.FilterType = Enum.RaycastFilterType.Exclude
    local result = workspace:Raycast(origin, direction.Unit * distance, raycastParams)
    if result and result.Instance then
        return result.Instance:IsDescendantOf(targetChar)
    end
    return false
end

RunService.Heartbeat:Connect(function()
    -- Lógica de Ghost e Plataforma
    local char = LocalPlayer.Character
    if char and char:FindFirstChild("HumanoidRootPart") then
        local hrp = char.HumanoidRootPart
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
        local myChar = LocalPlayer.Character
        local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
        if myRoot then
            local targetRoot = getPullTarget()
            if targetRoot then
                local targetPos = myRoot.CFrame.Position + (myRoot.CFrame.LookVector * PULL_DISTANCE)
                targetRoot.CFrame = targetRoot.CFrame:Lerp(CFrame.new(targetPos, myRoot.Position), LERP_SPEED)
            end
        end
    end

    -- Lógica de Hitbox e ESP (TeamCheck Inteligente)
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            local root = p.Character.HumanoidRootPart
            local hum = p.Character:FindFirstChild("Humanoid")
            if hum and hum.Health > 0 then
                local isEnemy = IsEnemy(p)
                local visible = IsVisible(root, p.Character)
                
                -- HITBOX DINÂMICA (Apenas Inimigos Visíveis)
                if getgenv().HBE_Enabled and isEnemy and visible then
                    local currentSize = getgenv().Underground_Enabled and getgenv().GhostHitboxSize or getgenv().HitboxSize
                    root.Size = Vector3.new(currentSize, currentSize, currentSize)
                    root.Transparency = 1
                else
                    root.Size = Vector3.new(2, 2, 1)
                    root.Transparency = 1
                end

                -- ESP (Aliados Azul / Inimigos Verde-Vermelho)
                local hl = p.Character:FindFirstChild("JUMENTAO_HL")
                if getgenv().ESP_Enabled then
                    if not hl then hl = Instance.new("Highlight", p.Character); hl.Name = "JUMENTAO_HL" end
                    if not isEnemy then
                        hl.FillColor = Color3.new(0, 0.5, 1) -- Azul para aliados
                        hl.FillTransparency = 0.5
                    else
                        hl.FillColor = visible and Color3.new(0, 1, 0) or Color3.new(1, 0, 0)
                        hl.FillTransparency = 0.3
                    end
                elseif hl then hl:Destroy() end
            end
        end
    end
end)

task.spawn(AutoDetectAllies)
Notify("PAINEL JUMENTÃO v4.4", "TeamCheck Automático Ativo!", 4)
print("Painel Jumentão V4.4 TeamCheck Loaded!")
