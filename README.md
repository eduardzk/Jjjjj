-- =====================================================
-- TORNADO HUB v2 | Natural Disaster Survival
-- Menu: Obsidian UI | Mobile: Fly + God + Tornado
-- =====================================================

local repo = "https://raw.githubusercontent.com/deividcomsono/Obsidian/main/"
local Library = loadstring(game:HttpGet(repo .. "Library.lua"))()

local Players    = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace  = game:GetService("Workspace")
local UserInput  = game:GetService("UserInputService")

local player = Players.LocalPlayer

-- CONFIGURAÇÕES
local config = {
    tornado = false,
    radius  = 12,
    height  = 8,
    speed   = 5,
    maxParts= 200,
    fly     = false,
    flySpeed= 50,
    god     = false,
}

local capturedParts = {}
local flyConnection, godConnection

-- ============ MENU ============
local Window = Library:CreateWindow({
    Title = "Tornado + Fly + God | NDS",
    Footer = "Mobile Ready 📱",
    Icon = 95816097006870,
    NotifySide = "Right",
})

local TornadoTab = Window:AddTab("Tornado", "wind")
local PlayerTab  = Window:AddTab("Player", "user")

-- ============ ABA TORNADO ============
local TG = TornadoTab:AddLeftGroupbox("Controle do Tornado")

TG:AddToggle("ToggleTornado", {
    Text = "Ativar Tornado",
    Default = false,
    Callback = function(v) config.tornado = v end,
})

TG:AddSlider("Raio", {
    Text = "Raio",
    Default = 12, Min = 5, Max = 40, Rounding = 1, Suffix = " studs",
    Callback = function(v) config.radius = v end,
})

TG:AddSlider("Altura", {
    Text = "Altura",
    Default = 8, Min = 0, Max = 30, Rounding = 1, Suffix = " studs",
    Callback = function(v) config.height = v end,
})

TG:AddSlider("Velocidade", {
    Text = "Velocidade de Rotação",
    Default = 5, Min = 1, Max = 20, Rounding = 0.5, Suffix = "x",
    Callback = function(v) config.speed = v end,
})

TG:AddSlider("MaxPecas", {
    Text = "Máximo de Peças",
    Default = 200, Min = 20, Max = 500, Rounding = 10,
    Callback = function(v) config.maxParts = v end,
})

TG:AddButton({
    Text = "🧹 Soltar Todas as Peças",
    Func = function()
        for part in pairs(capturedParts) do
            if part and part.Parent then
                part.Velocity = Vector3.new(0, 50, 0)
            end
        end
        capturedParts = {}
        Library:Notify({ Title="Tornado", Description="Peças liberadas!", Time=2 })
    end,
})

-- ============ ABA PLAYER (MOBILE) ============
local PG = PlayerTab:AddLeftGroupbox("Movimento & Sobrevivência")

PG:AddToggle("ToggleFly", {
    Text = "Ativar Fly",
    Default = false,
    Callback = function(v)
        config.fly = v
        if v then
            startFly()
        else
            stopFly()
        end
    end,
})

PG:AddSlider("FlySpeed", {
    Text = "Velocidade do Fly",
    Default = 50, Min = 10, Max = 200, Rounding = 5,
    Callback = function(v) config.flySpeed = v end,
})

PG:AddToggle("ToggleGod", {
    Text = "Ativar God Mode (Invencibilidade)",
    Default = false,
    Callback = function(v)
        config.god = v
        if v then startGod() else stopGod() end
    end,
})

-- ============ LÓGICA DO TORNADO ============
local angle = 0

local function isCharacterPart(part)
    for _, p in ipairs(Players:GetPlayers()) do
        if p.Character and part:IsDescendantOf(p.Character) then
            return true
        end
    end
    return false
end

local function captureParts()
    local count = 0
    for _ in pairs(capturedParts) do count = count + 1 end
    if count >= config.maxParts then return end

    for _, obj in ipairs(Workspace:GetDescendants()) do
        if count >= config.maxParts then break end
        if obj:IsA("BasePart")
            and not obj.Anchored
            and not capturedParts[obj]
            and not isCharacterPart(obj) then
            capturedParts[obj] = true
            count = count + 1
        end
    end
end

RunService.Heartbeat:Connect(function(dt)
    if not config.tornado then return end

    local char = player.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end

    angle = angle + dt * config.speed

    captureParts()

    local total = 0
    for _ in pairs(capturedParts) do total = total + 1 end
    if total == 0 then return end

    local i = 0
    for part in pairs(capturedParts) do
        if not part or not part.Parent then
            capturedParts[part] = nil
        else
            i = i + 1
            local partAngle = angle + (i / total) * math.pi * 2
            local yOffset = math.sin(partAngle * 2) * config.height + config.height
            local targetPos = hrp.Position + Vector3.new(
                math.cos(partAngle) * config.radius,
                yOffset,
                math.sin(partAngle) * config.radius
            )
            part.CFrame   = CFrame.new(targetPos) * CFrame.Angles(partAngle, partAngle * 2, 0)
            part.Velocity = Vector3.new(0, 0, 0)
        end
    end
end)

-- ============ LÓGICA DO FLY (MOBILE) ============
function startFly()
    local char = player.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end

    local bodyGyro = Instance.new("BodyGyro")
    bodyGyro.P = 9e4
    bodyGyro.MaxTorque = Vector3.new(9e9, 9e9, 9e9)
    bodyGyro.CFrame = hrp.CFrame
    bodyGyro.Parent = hrp

    local bodyVel = Instance.new("BodyVelocity")
    bodyVel.Velocity = Vector3.new(0, 0, 0)
    bodyVel.MaxForce = Vector3.new(9e9, 9e9, 9e9)
    bodyVel.Parent = hrp

    flyConnection = RunService.RenderStepped:Connect(function()
        if not config.fly then return end

        local moveDir = Vector3.new(0, 0, 0)
        local cam = Workspace.CurrentCamera

        -- Mobile: joystick virtual
        local thumbstick = UserInput:GetFocusedTextBox()
        local leftStick = UserInput:GetDeviceRotation()

        -- Usa o movimento do Humanoid para direção (funciona no mobile)
        local humanoid = char:FindFirstChildOfClass("Humanoid")
        if humanoid then
            local moveVector = humanoid.MoveDirection
            if moveVector.Magnitude > 0 then
                moveDir = moveVector * config.flySpeed
            end
        end

        -- Botões de subir/descer na tela
        -- (criados abaixo)

        bodyVel.Velocity = moveDir
        bodyGyro.CFrame = cam.CFrame
    end)

    Library:Notify({ Title="Fly", Description="Fly ativado! Use o joystick para voar.", Time=3 })
end

function stopFly()
    if flyConnection then
        flyConnection:Disconnect()
        flyConnection = nil
    end
    local char = player.Character
    if char then
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if hrp then
            local bg = hrp:FindFirstChildOfClass("BodyGyro")
            local bv = hrp:FindFirstChildOfClass("BodyVelocity")
            if bg then bg:Destroy() end
            if bv then bv:Destroy() end
        end
    end
end

-- ============ BOTÕES MOBILE PARA FLY ============
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "MobileFlyButtons"
screenGui.ResetOnSpawn = false
screenGui.Parent = player:WaitForChild("PlayerGui")

local upBtn = Instance.new("TextButton")
upBtn.Size = UDim2.new(0, 60, 0, 60)
upBtn.Position = UDim2.new(1, -150, 0.5, -70)
upBtn.BackgroundColor3 = Color3.new(0, 0.6, 0)
upBtn.TextColor3 = Color3.new(1, 1, 1)
upBtn.Text = "⬆️"
upBtn.Font = Enum.Font.GothamBold
upBtn.TextSize = 24
upBtn.Parent = screenGui

local downBtn = Instance.new("TextButton")
downBtn.Size = UDim2.new(0, 60, 0, 60)
downBtn.Position = UDim2.new(1, -150, 0.5, 0)
downBtn.BackgroundColor3 = Color3.new(0.6, 0, 0)
downBtn.TextColor3 = Color3.new(1, 1, 1)
downBtn.Text = "⬇️"
downBtn.Font = Enum.Font.GothamBold
downBtn.TextSize = 24
downBtn.Parent = screenGui

-- Lógica dos botões (pressionar e segurar)
local holdingUp, holdingDown = false, false

upBtn.MouseButton1Down:Connect(function() holdingUp = true end)
upBtn.MouseButton1Up:Connect(function() holdingUp = false end)
upBtn.MouseLeave:Connect(function() holdingUp = false end)

downBtn.MouseButton1Down:Connect(function() holdingDown = true end)
downBtn.MouseButton1Up:Connect(function() holdingDown = false end)
downBtn.MouseLeave:Connect(function() holdingDown = false end)

-- Atualiza a velocidade vertical no Fly
RunService.RenderStepped:Connect(function()
    if not config.fly then return end
    local char = player.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    local bv = hrp:FindFirstChildOfClass("BodyVelocity")
    if not bv then return end

    local vertical = 0
    if holdingUp then vertical = config.flySpeed end
    if holdingDown then vertical = -config.flySpeed end

    -- Soma a velocidade vertical à direção horizontal do joystick
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    local horizontal = Vector3.new(0, 0, 0)
    if humanoid and humanoid.MoveDirection.Magnitude > 0 then
        horizontal = humanoid.MoveDirection * config.flySpeed
    end

    bv.Velocity = Vector3.new(horizontal.X, vertical, horizontal.Z)
end)

-- ============ LÓGICA DO GOD MODE ============
function startGod()
    local char = player.Character
    if not char then return end
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if not humanoid then return end

    humanoid.MaxHealth = math.huge
    humanoid.Health = math.huge

    godConnection = humanoid.HealthChanged:Connect(function()
        if config.god then
            humanoid.Health = math.huge
        end
    end)

    Library:Notify({ Title="God Mode", Description="Invencibilidade ativada!", Time=2 })
end

function stopGod()
    if godConnection then
        godConnection:Disconnect()
        godConnection = nil
    end
    local char = player.Character
    if char then
        local humanoid = char:FindFirstChildOfClass("Humanoid")
        if humanoid then
            humanoid.MaxHealth = 100
            humanoid.Health = 100
        end
    end
end

-- ============ NOTIFICAÇÃO FINAL ============
Library:Notify({
    Title = "hitler hub carregado w",
    Description = "Menu carregado! Fly e God prontos para mobile.",
    Time = 4,
})
