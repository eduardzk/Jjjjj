-- =====================================================
-- TORNADO HUB v4 | Natural Disaster Survival
-- Sem dependência externa | Mobile Ready
-- =====================================================

local Players    = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace  = game:GetService("Workspace")
local UserInput  = game:GetService("UserInputService")

local player = Players.LocalPlayer

local config = {
    tornado = false, radius = 12, height = 8, speed = 5, maxParts = 100,
    fly = false, flySpeed = 50, god = false,
}

local capturedParts = {}
local flyConn, godConn, angle = nil, nil, 0

-- ============ GUI SIMPLES (sem biblioteca) ============
local sg = Instance.new("ScreenGui")
sg.Name = "eu amo hitler"
sg.ResetOnSpawn = false
sg.Parent = player:WaitForChild("PlayerGui")

-- Painel principal
local frame = Instance.new("Frame")
frame.Size = UDim2.new(0, 260, 0, 360)
frame.Position = UDim2.new(0, 10, 0.5, -180)
frame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
frame.BorderSizePixel = 0
frame.Active = true
frame.Draggable = true
frame.Parent = sg

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 8)
corner.Parent = frame

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, 0, 0, 32)
title.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
title.TextColor3 = Color3.new(1, 1, 1)
title.Text = "🌪️ Tornado Hub v4"
title.Font = Enum.Font.GothamBold
title.TextSize = 15
title.Parent = frame

local titleCorner = Instance.new("UICorner")
titleCorner.CornerRadius = UDim.new(0, 8)
titleCorner.Parent = title

local function makeButton(text, yPos, color)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(0.9, 0, 0, 34)
    b.Position = UDim2.new(0.05, 0, 0, yPos)
    b.BackgroundColor3 = color or Color3.fromRGB(60, 60, 60)
    b.TextColor3 = Color3.new(1, 1, 1)
    b.Text = text
    b.Font = Enum.Font.GothamSemibold
    b.TextSize = 13
    b.Parent = frame
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, 6)
    c.Parent = b
    return b
end

local tornBtn = makeButton("Tornado: OFF", 40, Color3.fromRGB(150, 30, 30))
local flyBtn  = makeButton("Fly: OFF", 82, Color3.fromRGB(150, 30, 30))
local godBtn  = makeButton("God: OFF", 124, Color3.fromRGB(150, 30, 30))
local relBtn  = makeButton("🧹 Soltar Peças", 166, Color3.fromRGB(70, 70, 70))

-- Sliders simples (texto + botões +/-)
local function makeSlider(label, yPos, default, min, max, step, callback)
    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(0.9, 0, 0, 20)
    lbl.Position = UDim2.new(0.05, 0, 0, yPos)
    lbl.BackgroundTransparency = 1
    lbl.TextColor3 = Color3.new(1, 1, 1)
    lbl.Text = label .. ": " .. default
    lbl.Font = Enum.Font.Gotham
    lbl.TextSize = 12
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = frame

    local val = default
    local function update()
        lbl.Text = label .. ": " .. val
        callback(val)
    end
    local minus = Instance.new("TextButton")
    minus.Size = UDim2.new(0, 30, 0, 20)
    minus.Position = UDim2.new(0.72, 0, 0, yPos)
    minus.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
    minus.Text = "-"
    minus.TextColor3 = Color3.new(1, 1, 1)
    minus.Font = Enum.Font.GothamBold
    minus.Parent = frame

    local plus = Instance.new("TextButton")
    plus.Size = UDim2.new(0, 30, 0, 20)
    plus.Position = UDim2.new(0.86, 0, 0, yPos)
    plus.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
    plus.Text = "+"
    plus.TextColor3 = Color3.new(1, 1, 1)
    plus.Font = Enum.Font.GothamBold
    plus.Parent = frame

    minus.MouseButton1Click:Connect(function()
        val = math.max(min, val - step); update()
    end)
    plus.MouseButton1Click:Connect(function()
        val = math.min(max, val + step); update()
    end)
    update()
end

makeSlider("Raio", 210, 12, 5, 40, 1, function(v) config.radius = v end)
makeSlider("Altura", 235, 8, 0, 30, 1, function(v) config.height = v end)
makeSlider("Velocidade", 260, 5, 1, 20, 0.5, function(v) config.speed = v end)
makeSlider("Max Peças", 285, 100, 20, 300, 10, function(v) config.maxParts = v end)
makeSlider("Fly Speed", 310, 50, 10, 150, 5, function(v) config.flySpeed = v end)

-- ============ TORNADO ============
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

    local att = Instance.new("Attachment")
    att.Name = "TornadoAtt"
    att.Parent = part

    local ap = Instance.new("AlignPosition")
    ap.Name = "TornadoAP"
    ap.Mode = Enum.PositionAlignmentMode.OneAttachment
    ap.Attachment0 = att
    ap.MaxForce = math.huge
    ap.MaxVelocity = math.huge
    ap.Responsiveness = 50
    ap.Parent = part

    local ao = Instance.new("AlignOrientation")
    ao.Name = "TornadoAO"
    ao.Mode = Enum.OrientationAlignmentMode.OneAttachment
    ao.Attachment0 = att
    ao.MaxTorque = math.huge
    ao.Responsiveness = 50
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

    local count = 0
    for _ in pairs(capturedParts) do count = count + 1 end
    if count < config.maxParts then
        for _, obj in ipairs(Workspace:GetDescendants()) do
            if count >= config.maxParts then break end
            if obj:IsA("BasePart") and not obj.Anchored then
                if attachPart(obj) then count = count + 1 end
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
                math.cos(a) * config.radius, y, math.sin(a) * config.radius
            )
            local ap = part:FindFirstChild("TornadoAP")
            local ao = part:FindFirstChild("TornadoAO")
            if ap then ap.Position = targetPos end
            if ao then ao.CFrame = CFrame.Angles(a, a * 2, 0) end
        end
    end
end)

-- ============ FLY ============
local flyHoldingUp, flyHoldingDown = false, false

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
    bv.Velocity = Vector3.new(0, 0, 0)
    bv.Parent = hrp

    flyConn = RunService.RenderStepped:Connect(function()
        if not config.fly then return end
        local cam = Workspace.CurrentCamera
        bg.CFrame = cam.CFrame
        local h = char:FindFirstChildOfClass("Humanoid")
        local move = h and h.MoveDirection * config.flySpeed or Vector3.new(0, 0, 0)
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

-- Botões de subir/descer para mobile
local upBtn = Instance.new("TextButton")
upBtn.Size = UDim2.new(0, 60, 0, 60)
upBtn.Position = UDim2.new(1, -140, 0.5, -70)
upBtn.BackgroundColor3 = Color3.fromRGB(0, 150, 0)
upBtn.TextColor3 = Color3.new(1, 1, 1)
upBtn.Text = "⬆️"
upBtn.Font = Enum.Font.GothamBold
upBtn.TextSize = 24
upBtn.Parent = sg

local downBtn = Instance.new("TextButton")
downBtn.Size = UDim2.new(0, 60, 0, 60)
downBtn.Position = UDim2.new(1, -140, 0.5, 0)
downBtn.BackgroundColor3 = Color3.fromRGB(150, 0, 0)
downBtn.TextColor3 = Color3.new(1, 1, 1)
downBtn.Text = "⬇️"
downBtn.Font = Enum.Font.GothamBold
downBtn.TextSize = 24
downBtn.Parent = sg

upBtn.MouseButton1Down:Connect(function() flyHoldingUp = true end)
upBtn.MouseButton1Up:Connect(function() flyHoldingUp = false end)
upBtn.MouseLeave:Connect(function() flyHoldingUp = false end)
downBtn.MouseButton1Down:Connect(function() flyHoldingDown = true end)
downBtn.MouseButton1Up:Connect(function() flyHoldingDown = false end)
downBtn.MouseLeave:Connect(function() flyHoldingDown = false end)

-- ============ GOD ============
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
