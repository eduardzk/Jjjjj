Players.LocalPlayer-- =====================================================================
-- TORNADO HUB v5 (OBSIDIAN UI) | Natural Disaster Survival
-- Lógica de replicação de física (Network Ownership) e Fly.
-- =====================================================================

local repo = "https://raw.githubusercontent.com/deividcomsono/Obsidian/main/"
local Library = loadstring(game:HttpGet(repo .. "Library.lua"))()

local Players    = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace  = game:GetService("Workspace")

local player = Players.LocalPlayer

local config = {
    tornado = false, radius = 15, height = 10, speed = 8, maxParts = 150,
    fly = false, flySpeed = 50, god = false,
}

local capturedParts = {}
local flyConn, godConn, angle = nil, nil, 0
local scanCooldown = 0
local flyHoldingUp, flyHoldingDown = false, false

-- =====================================================================
-- LÓGICA DO TORNADO
-- =====================================================================
local function isCharPart(part)
    for _, p in ipairs(Players:GetPlayers()) do
        if p.Character and part:IsDescendantOf(p.Character) then return true end
    end
    return false
end

local function attachPart(part)
    if part.Anchored or isCharPart(part) or capturedParts[part] then return false end
    
    local ok = pcall(function() part:SetNetworkOwner(player) end)
    if not ok then return false end

    part.CanCollide = true

    local att = Instance.new("Attachment")
    att.Name = "TornadoAtt"
    att.Parent = part

    local ap = Instance.new("AlignPosition")
    ap.Name = "TornadoAP"
    ap.Mode = Enum.PositionAlignmentMode.OneAttachment
    ap.Attachment0 = att
    ap.MaxForce = math.huge
    ap.MaxVelocity = math.huge
    ap.Responsiveness = 200
    ap.Parent = part

    local ao = Instance.new("AlignOrientation")
    ao.Name = "TornadoAO"
    ao.Mode = Enum.OrientationAlignmentMode.OneAttachment
    ao.Attachment0 = att
    ao.MaxTorque = math.huge
    ao.Responsiveness = 200
    ao.Parent = part

    capturedParts[part] = true
    return true
end

local function detachPart(part)
    if not part or not part.Parent then return end
    for _, name in ipairs({"TornadoAP","TornadoAO","TornadoAtt"}) do
        local obj = part:FindFirstChild(name)
        if obj then obj:Destroy() end
    end
    capturedParts[part] = nil
end

RunService.Heartbeat:Connect(function(dt)
    if not config.tornado then return end
    local char = player.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end

    angle = angle + dt * config.speed
    scanCooldown = scanCooldown + dt
    local count = 0
    for _ in pairs(capturedParts) do count = count + 1 end

    if count < config.maxParts and scanCooldown > 0.5 then
        scanCooldown = 0
        for _, obj in ipairs(Workspace:GetDescendants()) do
            if count >= config.maxParts then break end
            if obj:IsA("BasePart") and not obj.Anchored and not obj:IsDescendantOf(char) then
                if not obj:FindFirstChild("TornadoAP") then
                    if attachPart(obj) then count = count + 1 end
                end
            end
        end
    end

    local total = 0
    for _ in pairs(capturedParts) do total = total + 1 end
    if total == 0 then return end

    local i = 0
    for part in pairs(capturedParts) do
        if not part or not part.Parent then
            capturedParts[part] = nil
        else
            i = i + 1
            local a = angle + (i / total) * math.pi * 2
            local y = math.sin(a * 2) * config.height + config.height
            local targetPos = hrp.Position + Vector3.new(
                math.cos(a) * config.radius, 
                y, 
                math.sin(a) * config.radius
            )
            
            local ap = part:FindFirstChild("TornadoAP")
            local ao = part:FindFirstChild("TornadoAO")
            
            if ap then 
                ap.Position = targetPos 
                part.AssemblyLinearVelocity = Vector3.new(math.cos(a)*100, 50, math.sin(a)*100)
            end
            
            if ao then 
                ao.CFrame = CFrame.Angles(a, a * 2, 0) 
            end
        end
    end
end)

-- =====================================================================
-- LÓGICA DO FLY E GOD MODE
-- =====================================================================
local function startFly()
    local char = player.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end

    local bg = Instance.new("BodyGyro")
    bg.P = 9e4
    bg.MaxTorque = Vector3.new(9e9, 9e9, 9e9)
    bg.CFrame = hrp.CFrame
    bg.Parent = hrp

    local bv = Instance.new("BodyVelocity")
    bv.MaxForce = Vector3.new(9e9, 9e9, 9e9)
    bv.Velocity = Vector3.zero
    bv.Parent = hrp

    flyConn = RunService.RenderStepped:Connect(function()
        if not config.fly then return end
        local cam = Workspace.CurrentCamera
        bg.CFrame = cam.CFrame
        local h = char:FindFirstChildOfClass("Humanoid")
        local move = h and h.MoveDirection * config.flySpeed or Vector3.zero
        local vy = 0
        if flyHoldingUp then vy = config.flySpeed end
        if flyHoldingDown then vy = -config.flySpeed end
        bv.Velocity = Vector3.new(move.X, vy, move.Z)
    end)
end

local function stopFly()
    if flyConn then flyConn:Disconnect(); flyConn = nil end
    local char = player.Character
    if char then
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if hrp then
            local bg = hrp:FindFirstChildOfClass("BodyGyro"); if bg then bg:Destroy() end
            local bv = hrp:FindFirstChildOfClass("BodyVelocity"); if bv then bv:Destroy() end
        end
    end
end

local function startGod()
    local char = player.Character
    if not char then return end
    local h = char:FindFirstChildOfClass("Humanoid")
    if not h then return end
    h.MaxHealth = math.huge
    h.Health = math.huge
    godConn = h.HealthChanged:Connect(function()
        if config.god then h.Health = math.huge end
    end)
end

local function stopGod()
    if godConn then godConn:Disconnect(); godConn = nil end
    local char = player.Character
    if char then
        local h = char:FindFirstChildOfClass("Humanoid")
        if h then h.MaxHealth = 100; h.Health = 100 end
    end
end

-- =====================================================================
-- INTERFACE GRÁFICA (OBSIDIAN)
-- =====================================================================
local Window = Library:CreateWindow({
    Title = "Tornado Hub v5",
    Footer = "Natural Disaster Survival",
    Icon = 95816097006870,
    NotifySide = "Right",
    ShowCustomCursor = true,
})

local Tabs = {
    Main = Window:AddTab("Main", "tornado"),
}

-- Aba do Tornado
local TornadoGroup = Tabs.Main:AddLeftGroupbox("Tornado")

TornadoGroup:AddToggle("TornadoToggle", {
    Text = "Ativar Tornado",
    Default = false,
    Tooltip = "Puxa peças soltas e arremessa jogadores.",
    Callback = function(state) config.tornado = state end
})

TornadoGroup:AddSlider("RadiusSlider", {
    Text = "Raio",
    Default = 15,
    Min = 5,
    Max = 50,
    Rounding = 0,
    Callback = function(val) config.radius = val end
})

TornadoGroup:AddSlider("HeightSlider", {
    Text = "Altura",
    Default = 10,
    Min = 0,
    Max = 40,
    Rounding = 0,
    Callback = function(val) config.height = val end
})

TornadoGroup:AddSlider("SpeedSlider", {
    Text = "Velocidade",
    Default = 8,
    Min = 1,
    Max = 25,
    Rounding = 0,
    Callback = function(val) config.speed = val end
})

TornadoGroup:AddSlider("MaxPartsSlider", {
    Text = "Máximo de Peças",
    Default = 150,
    Min = 20,
    Max = 500,
    Rounding = 0,
    Callback = function(val) config.maxParts = val end
})

TornadoGroup:AddButton({
    Text = "Soltar Peças",
    Func = function()
        for part in pairs(capturedParts) do detachPart(part) end
        capturedParts = {}
    end
})

-- Aba do Jogador
local PlayerGroup = Tabs.Main:AddRightGroupbox("Jogador")

PlayerGroup:AddToggle("FlyToggle", {
    Text = "Voo (Fly)",
    Default = false,
    Callback = function(state)
        config.fly = state
        if state then startFly() else stopFly() end
    end
})

PlayerGroup:AddSlider("FlySpeedSlider", {
    Text = "Velocidade do Voo",
    Default = 50,
    Min = 10,
    Max = 150,
    Rounding = 0,
    Callback = function(val) config.flySpeed = val end
})

PlayerGroup:AddToggle("GodToggle", {
    Text = "God Mode",
    Default = false,
    Callback = function(state)
        config.god = state
        if state then startGod() else stopGod() end
    end
})

-- =====================================================================
-- BOTÕES MOBILE DE VOO (SUBIR/DESCER)
-- =====================================================================
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "TornadoMobileFly"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = player:WaitForChild("PlayerGui")

local function makeFlyBtn(text, yPos, color)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 60, 0, 60)
    btn.Position = UDim2.new(1, -140, 0.5, yPos)
    btn.BackgroundColor3 = color
    btn.TextColor3 = Color3.new(1, 1, 1)
    btn.Text = text
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 24
    btn.Parent = ScreenGui
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 8)
    return btn
end

local upBtn = makeFlyBtn("⬆️", -70, Color3.fromRGB(0, 150, 0))
local downBtn = makeFlyBtn("⬇️", 0, Color3.fromRGB(150, 0, 0))

upBtn.MouseButton1Down:Connect(function() flyHoldingUp = true end)
upBtn.MouseButton1Up:Connect(function() flyHoldingUp = false end)
upBtn.MouseLeave:Connect(function() flyHoldingUp = false end)

downBtn.MouseButton1Down:Connect(function() flyHoldingDown = true end)
downBtn.MouseButton1Up:Connect(function() flyHoldingDown = false end)
downBtn.MouseLeave:Connect(function() flyHoldingDown = false end)

-- Respawn handling
player.CharacterAdded:Connect(function(char)
    task.wait(1)
    if config.god then startGod() end
    if config.fly then stopFly(); startFly() end
end)then h.Health = math.huge end
    end)
end

local function stopGod()
    if godConn then godConn:Disconnect(); godConn = nil end
    local char = player.Character
    if char then
        local h = char:FindFirstChildOfClass("Humanoid")
        if h then h.MaxHealth = 100; h.Health = 100 end
    end
end

-- ============ BOTÕES ============
tornBtn.MouseButton1Click:Connect(function()
    config.tornado = not config.tornado
    tornBtn.Text = "Tornado: " .. (config.tornado and "ON" or "OFF")
    tornBtn.BackgroundColor3 = config.tornado and Color3.fromRGB(0, 150, 0) or Color3.fromRGB(150, 30, 30)
end)

flyBtn.MouseButton1Click:Connect(function()
    config.fly = not config.fly
    flyBtn.Text = "Fly: " .. (config.fly and "ON" or "OFF")
    flyBtn.BackgroundColor3 = config.fly and Color3.fromRGB(0, 150, 0) or Color3.fromRGB(150, 30, 30)
    if config.fly then startFly() else stopFly() end
end)

godBtn.MouseButton1Click:Connect(function()
    config.god = not config.god
    godBtn.Text = "God: " .. (config.god and "ON" or "OFF")
    godBtn.BackgroundColor3 = config.god and Color3.fromRGB(0, 150, 0) or Color3.fromRGB(150, 30, 30)
    if config.god then startGod() else stopGod() end
end)

relBtn.MouseButton1Click:Connect(function()
    for part in pairs(capturedParts) do detachPart(part) end
    capturedParts = {}
end)

-- Reaplica God se o personagem respawnar
player.CharacterAdded:Connect(function(char)
    task.wait(1)
    if config.god then startGod() end
    if config.fly then stopFly(); startFly() end
end)

print("[TornadoHub v4] Carregado com sucesso!")ass("Humanoid")
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
})hildOfClass("Humanoid")
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
