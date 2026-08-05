-- ====================================================================
-- SERVIÇOS DO ROBLOX
-- ====================================================================
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local CoreGui = game:GetService("CoreGui")

local LocalPlayer = Players.LocalPlayer

-- ====================================================================
-- INICIALIZAÇÃO DA BIBLIOTECA RAYFIELD (Com Tratamento de Erro)
-- ====================================================================
local success, Rayfield = pcall(function()
    return loadstring(game:HttpGet('https://sirius.menu/rayfield'))()
end)

if not success or not Rayfield then
    warn("Falha crítica ao carregar a biblioteca Rayfield.")
    return
end

-- ====================================================================
-- ESTADOS E CONFIGURAÇÕES DO SCRIPT
-- ====================================================================
local KEY_CORRETA = "Anonymus777777"

local State = {
    Player = {
        WalkSpeed = 16,
        JumpPower = 50,
        Gravity = Workspace.Gravity,
        InfJump = false,
        Noclip = false,
        AutoSprint = false,
        Scale = 1,
    },
    Farm = {
        Enabled = false,
        MagnetOn = false,
        Radius = 20,
        Speed = 45,
    },
    Settings = {
        Notifications = true,
    },
    UI = {
        StatusLabel = nil,
        StatsLabel = nil,
    }
}

-- Estruturas internas para o Ímã
local MagnetData = {
    OverlapParams = nil,
    FixedObjects = {},
    Welds = {}
}

-- ====================================================================
-- GERENCIADOR DE CONEXÕES (PREVINE MEMORY LEAKS)
-- ====================================================================
local ConnectionManager = {}
local ActiveConnections = {}

function ConnectionManager:Add(name, connection)
    self:Remove(name)
    ActiveConnections[name] = connection
end

function ConnectionManager:Remove(name)
    if ActiveConnections[name] then
        ActiveConnections[name]:Disconnect()
        ActiveConnections[name] = nil
    end
end

function ConnectionManager:ClearAll()
    for name, connection in pairs(ActiveConnections) do
        if connection and typeof(connection) == "RBXScriptConnection" then
            connection:Disconnect()
        end
    end
    table.clear(ActiveConnections)
end

-- ====================================================================
-- FUNÇÕES AUXILIARES
-- ====================================================================
local function Notify(title, content, duration)
    if State.Settings.Notifications then
        pcall(function()
            Rayfield:Notify({
                Title = title,
                Content = content,
                Duration = duration or 3,
                Image = 4483362458,
            })
        end)
    end
end

local function GetCharacter()
    return LocalPlayer.Character
end

local function GetRoot()
    local char = GetCharacter()
    return char and char:FindFirstChild("HumanoidRootPart")
end

-- ====================================================================
-- MÓDULO: TELEPORTE
-- ====================================================================
local function ExecutarTeleporte(locationName, position)
    local tpSuccess, err = pcall(function()
        local hrp = GetRoot()
        if not hrp then
            error("HumanoidRootPart não encontrada. Personagem morto ou ausente.")
        end
        hrp.CFrame = CFrame.new(position + Vector3.new(0, 3, 0))
    end)

    if tpSuccess then
        Notify("Teleporte Concluído", "Você foi levado para: " .. locationName, 2)
    else
        warn("Erro no teleporte para " .. locationName .. ": " .. tostring(err))
        Notify("Erro de Teleporte", "Falha ao teleportar. Verifique se o personagem está vivo.", 3)
    end
end

-- ====================================================================
-- MÓDULO: PLAYER
-- ====================================================================
local function ApplyPlayerStats()
    pcall(function()
        local char = GetCharacter()
        if not char then return end
        
        local hum = char:FindFirstChildOfClass("Humanoid")
        local hrp = char:FindFirstChild("HumanoidRootPart")
        
        if hum then
            local targetSpeed = State.Player.AutoSprint and (State.Player.WalkSpeed * 1.5) or State.Player.WalkSpeed
            if hum.WalkSpeed ~= targetSpeed then hum.WalkSpeed = targetSpeed end
            
            hum.UseJumpPower = true
            if hum.JumpPower ~= State.Player.JumpPower then hum.JumpPower = State.Player.JumpPower end
        end

        if hrp then
            local targetSize = Vector3.new(2, 1, 1) * State.Player.Scale
            if hrp.Size ~= targetSize then hrp.Size = targetSize end
        end
        
        if Workspace.Gravity ~= State.Player.Gravity then
            Workspace.Gravity = State.Player.Gravity
        end
    end)
end

task.spawn(function()
    while true do
        task.wait(0.2)
        ApplyPlayerStats()
    end
end)

ConnectionManager:Add("InfJump", UserInputService.JumpRequest:Connect(function()
    if State.Player.InfJump then
        pcall(function()
            local char = GetCharacter()
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            if hum then
                hum:ChangeState(Enum.HumanoidStateType.Jumping)
            end
        end)
    end
end))

ConnectionManager:Add("Noclip", RunService.Stepped:Connect(function()
    if not State.Player.Noclip then return end
    
    pcall(function()
        local char = GetCharacter()
        if char then
            for _, part in ipairs(char:GetDescendants()) do
                if part:IsA("BasePart") and part.CanCollide then
                    part.CanCollide = false
                end
            end
        end
    end)
end))

-- ====================================================================
-- MÓDULO: MAGNET (AUTO FARM)
-- ====================================================================
local function InitMagnetParams()
    if not MagnetData.OverlapParams then
        MagnetData.OverlapParams = OverlapParams.new()
        MagnetData.OverlapParams.FilterType = Enum.RaycastFilterType.Blacklist
    end
    MagnetData.OverlapParams.FilterDescendantsInstances = {GetCharacter() or {}}
end

local function ClearMagnet()
    for _, weld in pairs(MagnetData.Welds) do
        if weld and weld.Parent then weld:Destroy() end
    end
    table.clear(MagnetData.Welds)
    table.clear(MagnetData.FixedObjects)
end

local function ToggleMagnet(enabled)
    State.Farm.MagnetOn = enabled
    if not enabled then
        ClearMagnet()
        Notify("Ímã", "Desligado - Objetos Soltos", 2)
    else
        Notify("Ímã", "Ligado - Puxando Objetos", 2)
    end
    
    if State.UI.StatusLabel then
        pcall(function()
            State.UI.StatusLabel:Set("Status: " .. (enabled and "Running" or "Paused"))
        end)
    end
end

ConnectionManager:Add("MagnetLoop", RunService.Heartbeat:Connect(function()
    if not State.Farm.Enabled or not State.Farm.MagnetOn then return end

    pcall(function()
        local hrp = GetRoot()
        if not hrp then return end

        InitMagnetParams()
        
        local parts = Workspace:GetPartBoundsInRadius(hrp.Position, State.Farm.Radius, MagnetData.OverlapParams)

        for _, obj in ipairs(parts) do
            if obj.Name == "TranspBox" and not obj.Anchored and not MagnetData.FixedObjects[obj] then
                local dist = (obj.Position - hrp.Position).Magnitude

                if dist > 3 then
                    local direction = (hrp.Position - obj.Position).Unit
                    obj.AssemblyLinearVelocity = direction * State.Farm.Speed
                else
                    obj.AssemblyLinearVelocity = Vector3.zero
                    local weld = Instance.new("WeldConstraint")
                    weld.Part0 = hrp
                    weld.Part1 = obj
                    weld.Parent = obj

                    MagnetData.FixedObjects[obj] = true
                    MagnetData.Welds[obj] = weld
                end
            end
        end
    end)
end))

ConnectionManager:Add("MagnetHotkey", UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == Enum.KeyCode.T then
        if State.Farm.Enabled then
            ToggleMagnet(not State.Farm.MagnetOn)
        else
            Notify("Ímã", "Ative o Auto Farm no menu primeiro.", 2)
        end
    end
end))

-- ====================================================================
-- ATUALIZADOR DE UI (FPS, PING)
-- ====================================================================
task.spawn(function()
    while true do
        task.wait(1)
        if State.UI.StatsLabel then
            pcall(function()
                local ping = math.floor(LocalPlayer:GetNetworkPing() * 1000)
                local fps = math.floor(1 / RunService.RenderStepped:Wait())
                State.UI.StatsLabel:Set(string.format("FPS: %d | Ping: %d ms", fps, ping))
            end)
        end
    end
end)

-- ====================================================================
-- CRIAÇÃO DA INTERFACE RAYFIELD (MENU PRINCIPAL)
-- ====================================================================
local function BuildRayfieldMenu()
    local Window = Rayfield:CreateWindow({
        Name = "Torcidas 7",
        LoadingTitle = "Carregando...",
        LoadingSubtitle = "by Assistant",
        ConfigurationSaving = { Enabled = false },
        KeySystem = false,
    })

    if not Window then
        warn("Falha ao criar a janela do Rayfield.")
        return
    end

    -- ---------------------------------------------------------
    -- ABA 1: PLAYER
    -- ---------------------------------------------------------
    local PlayerTab = Window:CreateTab("Player")

    PlayerTab:CreateSlider({
        Name = "WalkSpeed", Range = {16, 300}, Increment = 1, CurrentValue = State.Player.WalkSpeed,
        Callback = function(v) State.Player.WalkSpeed = v end
    })
    
    PlayerTab:CreateSlider({
        Name = "JumpPower", Range = {50, 300}, Increment = 1, CurrentValue = State.Player.JumpPower,
        Callback = function(v) State.Player.JumpPower = v end
    })

    PlayerTab:CreateSlider({
        Name = "Gravidade", Range = {0, 500}, Increment = 5, CurrentValue = State.Player.Gravity,
        Callback = function(v) State.Player.Gravity = v end
    })

    PlayerTab:CreateToggle({
        Name = "Pulo Infinito (InfJump)", CurrentValue = false,
        Callback = function(v) State.Player.InfJump = v end
    })

    PlayerTab:CreateToggle({
        Name = "Atravessar Paredes (Noclip)", CurrentValue = false,
        Callback = function(v) State.Player.Noclip = v end
    })

    PlayerTab:CreateToggle({
        Name = "Auto Sprint", CurrentValue = false,
        Callback = function(v) State.Player.AutoSprint = v end
    })

    PlayerTab:CreateSlider({
        Name = "Tamanho do Personagem", Range = {0.5, 3}, Increment = 0.1, CurrentValue = 1,
        Callback = function(v) State.Player.Scale = v end
    })

    PlayerTab:CreateButton({
        Name = "Resetar Personagem",
        Callback = function() 
            pcall(function() GetCharacter():BreakJoints() end) 
        end
    })

    -- ---------------------------------------------------------
    -- ABA 2: AUTO FARM (MAGNET)
    -- ---------------------------------------------------------
    local FarmTab = Window:CreateTab("Auto Farm")

    FarmTab:CreateToggle({
        Name = "Ativar Auto Farm (Magnet)", CurrentValue = false,
        Callback = function(v)
            State.Farm.Enabled = v
            ToggleMagnet(v)
            if not v and State.UI.StatusLabel then 
                pcall(function() State.UI.StatusLabel:Set("Status: Stopped") end) 
            end
        end
    })

    FarmTab:CreateSlider({
        Name = "Raio do Ímã", Range = {5, 100}, Increment = 1, CurrentValue = State.Farm.Radius,
        Callback = function(v) State.Farm.Radius = v end
    })

    FarmTab:CreateSlider({
        Name = "Velocidade de Puxar", Range = {10, 200}, Increment = 1, CurrentValue = State.Farm.Speed,
        Callback = function(v) State.Farm.Speed = v end
    })

    State.UI.StatusLabel = FarmTab:CreateLabel("Status: Stopped")

    FarmTab:CreateButton({
        Name = "Pausar / Retomar Ímã (Atalho: T)",
        Callback = function()
            if State.Farm.Enabled then
                ToggleMagnet(not State.Farm.MagnetOn)
            else
                Notify("Aviso", "Ative o Auto Farm primeiro.", 2)
            end
        end
    })

    -- ---------------------------------------------------------
    -- ABA 3: TELEPORTS (Garantida e Visível)
    -- ---------------------------------------------------------
    local TeleportTab = Window:CreateTab("Teleports")

    TeleportTab:CreateButton({
        Name = "TP 1 — Pegar Caixas",
        Callback = function()
            ExecutarTeleporte("TP 1 — Pegar Caixas", Vector3.new(-25621, 32, -5925))
        end
    })

    TeleportTab:CreateButton({
        Name = "TP 2 — Construção",
        Callback = function()
            ExecutarTeleporte("TP 2 — Construção", Vector3.new(-3618, 65, -2508))
        end
    })

    TeleportTab:CreateButton({
        Name = "TP 3 — Auto Peças",
        Callback = function()
            ExecutarTeleporte("TP 3 — Auto Peças", Vector3.new(-3328, 65, -3422))
        end
    })

    TeleportTab:CreateButton({
        Name = "TP 4 — Posto Auto",
        Callback = function()
            ExecutarTeleporte("TP 4 — Posto Auto", Vector3.new(-3214, 66, -3697))
        end
    })

    TeleportTab:CreateButton({
        Name = "TP 5 — Concessionária",
        Callback = function()
            ExecutarTeleporte("TP 5 — Concessionária", Vector3.new(-3080, 66, -3689))
        end
    })

    TeleportTab:CreateButton({
        Name = "TP 6 — Ferro Velho",
        Callback = function()
            ExecutarTeleporte("TP 6 — Ferro Velho", Vector3.new(-3147, 64, -4246))
        end
    })

    -- ---------------------------------------------------------
    -- ABA 4: SETTINGS (Garantida e Visível)
    -- ---------------------------------------------------------
    local SettingsTab = Window:CreateTab("Settings")

    SettingsTab:CreateToggle({
        Name = "Notificações na Tela", CurrentValue = State.Settings.Notifications,
        Callback = function(v) State.Settings.Notifications = v end
    })

    SettingsTab:CreateButton({
        Name = "Destruir Interface (Descarregar)",
        Callback = function()
            ClearMagnet()
            ConnectionManager:ClearAll()
            Rayfield:Destroy()
        end
    })

    SettingsTab:CreateLabel("Versão: 2.4 (Abas Corrigidas)")
    State.UI.StatsLabel = SettingsTab:CreateLabel("Calculando FPS/Ping...")

    Notify("Sucesso", "Menu carregado! Todas as abas ativas.", 3)
end

-- ====================================================================
-- SISTEMA DE KEY
-- ====================================================================
local function BuildKeySystem()
    local screen = Instance.new("ScreenGui")
    screen.Name = "KeySystemUI"
    screen.Parent = CoreGui
    screen.IgnoreGuiInset = true

    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(0, 350, 0, 150)
    frame.Position = UDim2.new(0.5, -175, 0.5, -75)
    frame.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
    frame.BorderSizePixel = 0
    frame.Parent = screen

    local corner = Instance.new("UICorner", frame)
    corner.CornerRadius = UDim.new(0, 10)

    local stroke = Instance.new("UIStroke", frame)
    stroke.Color = Color3.fromRGB(255, 50, 50)
    stroke.Thickness = 2

    local title = Instance.new("TextLabel", frame)
    title.Size = UDim2.new(1, 0, 0, 40)
    title.BackgroundTransparency = 1
    title.Text = "🔑 Torcidas 7 - Key System"
    title.TextColor3 = Color3.new(1, 1, 1)
    title.Font = Enum.Font.GothamBold
    title.TextSize = 18

    local textbox = Instance.new("TextBox", frame)
    textbox.Size = UDim2.new(1, -40, 0, 40)
    textbox.Position = UDim2.new(0, 20, 0, 45)
    textbox.PlaceholderText = "Digite sua Key aqui..."
    textbox.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
    textbox.TextColor3 = Color3.new(1, 1, 1)
    textbox.Font = Enum.Font.Gotham
    textbox.TextSize = 14
    Instance.new("UICorner", textbox).CornerRadius = UDim.new(0, 6)

    local btn = Instance.new("TextButton", frame)
    btn.Size = UDim2.new(1, -40, 0, 35)
    btn.Position = UDim2.new(0, 20, 0, 95)
    btn.BackgroundColor3 = Color3.fromRGB(255, 50, 50)
    btn.Text = "Verificar Key"
    btn.TextColor3 = Color3.new(1, 1, 1)
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 14
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)

    btn.MouseButton1Click:Connect(function()
        if textbox.Text == KEY_CORRETA then
            btn.Text = "Key Aprovada!"
            btn.BackgroundColor3 = Color3.fromRGB(50, 255, 50)
            task.wait(0.5)
            screen:Destroy()
            BuildRayfieldMenu()
        else
            btn.Text = "Key Incorreta!"
            textbox.Text = ""
            task.wait(1)
            btn.Text = "Verificar Key"
        end
    end)
end

-- ====================================================================
-- START DO SCRIPT
-- ====================================================================
BuildKeySystem()
