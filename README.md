--[[
    skyaimbotv12.lua
    Place this LocalScript inside:
      StarterPlayer > StarterCharacterScripts

    ╔══════════════════════════════════════════════════════════╗
    ║  Press K (or click ☰) to open / close the menu         ║
    ║                                                          ║
    ║  BUTTON 1 — Sky Aim ON/OFF                              ║
    ║    • Counteracts gravity — character floats             ║
    ║    • MIN HEIGHT FLOOR — never falls below the map       ║
    ║    • HUNTING — teleports ABOVE AND BEHIND the nearest   ║
    ║      target every frame using the target's LookVector   ║
    ║    • LINE-OF-SIGHT raycast before engaging              ║
    ║    • LOCK-ON — tilts character & camera straight down   ║
    ║      at the target, auto-fires, draws blue bullet tracer║
    ║    • Fallback to random sky spot if no targets alive    ║
    ║                                                          ║
    ║  BUTTON 2 — Target: Enemies Only / All Players          ║
    ║    • Enemies Only — skips same-team players (default)   ║
    ║    • All Players  — targets everyone regardless         ║
    ║                                                          ║
    ║  BUTTON 3 — ESP: ON/OFF                                 ║
    ║    • Full blue Highlight on targets (through walls)     ║
    ║    • Name tag above head   — through walls              ║
    ║    • Health % beside torso — through walls              ║
    ║    • Current weapon below feet — through walls          ║
    ║    • Auto-cleans up when players leave                  ║
    ║                                                          ║
    ║  BUTTON 4 — Priority (cycles through 3 modes)          ║
    ║    • Closest       — nearest by 3D distance             ║
    ║    • Closest Mouse — nearest to the mouse cursor        ║
    ║    • Highest HP    — player with the most health        ║
    ║                                                          ║
    ║  BUTTON 5 — Hitbox Expander ON/OFF                      ║
    ║    • Scales every valid target's HumanoidRootPart 8×   ║
    ║    • Client-side only — only affects YOUR shots         ║
    ║    • Restores original sizes when turned OFF            ║
    ║    • Re-applies automatically after target respawns     ║
    ║    • VISUALIZATION: orange SelectionBox wireframe shows ║
    ║      the expanded hitbox boundary in the 3D world       ║
    ╚══════════════════════════════════════════════════════════╝

    Requirements:
      • Players assigned to Teams in Studio (Teams service).
      • Character has a Tool equipped that fires via Tool:Activate().
        Edit fireWeapon() for RemoteEvent-based weapons.
--]]

-- ─── Services ────────────────────────────────────────────────────────────────
local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService     = game:GetService("TweenService")

local LocalPlayer = Players.LocalPlayer
local Camera      = workspace.CurrentCamera

-- ─── Config ──────────────────────────────────────────────────────────────────
local HOVER_HEIGHT    = 20
local BEHIND_DISTANCE = 18
local MIN_HEIGHT      = 10
local FIRE_RATE       = 0.5
local AIM_SMOOTH      = 0.12

local TRACER_COLOR    = Color3.fromRGB(0, 140, 255)
local TRACER_WIDTH    = 0.08
local TRACER_LIFETIME = 0.35

local FALLBACK_HEIGHT = 200
local FALLBACK_RADIUS = 300

local ESP_FILL_COLOR    = Color3.fromRGB(0, 120, 255)
local ESP_OUTLINE_COLOR = Color3.fromRGB(180, 220, 255)
local ESP_NAME_COLOR    = Color3.fromRGB(120, 200, 255)
local ESP_HP_COLOR      = Color3.fromRGB(80, 255, 120)
local ESP_WEP_COLOR     = Color3.fromRGB(255, 200, 80)

local HITBOX_SCALE       = 8
-- Hitbox visualisation colours
local HITBOX_LINE_COLOR  = Color3.fromRGB(255, 130, 0)     -- orange wireframe
local HITBOX_FILL_COLOR  = Color3.fromRGB(255, 160, 0)     -- faint orange fill
local HITBOX_LINE_THICK  = 0.07
local HITBOX_FILL_ALPHA  = 0.82                            -- 0 = opaque, 1 = invisible

local PRIORITY_MODES  = { "Closest", "Closest Mouse", "Highest HP" }
local PRIORITY_COLORS = {
    Color3.fromRGB(50,  130,  50),
    Color3.fromRGB(160, 100,   0),
    Color3.fromRGB(160,  30,  30),
}

local TWEEN_TIME = 0.22
local PANEL_W    = 210
local PANEL_H    = 272

-- ─── State ───────────────────────────────────────────────────────────────────
local enabled         = false
local targetAll       = false
local espEnabled      = false
local hitboxEnabled   = false
local priorityIndex   = 1
local menuOpen        = false
local lastFireTime    = 0
local originalCamType
local espObjects      = {}   -- [player] = { highlight, nameLabel, hpLabel, weaponLabel }
local originalSizes   = {}   -- [player] = Vector3
local hitboxConns     = {}   -- [player] = RBXScriptConnection
local hitboxVisuals   = {}   -- [player] = SelectionBox

-- ─── Main ScreenGui ──────────────────────────────────────────────────────────
local screenGui = Instance.new("ScreenGui")
screenGui.Name           = "SkyAimbotV12Gui"
screenGui.ResetOnSpawn   = false
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
screenGui.IgnoreGuiInset = false
screenGui.Parent         = LocalPlayer:WaitForChild("PlayerGui")

-- ─── GUI Helpers ─────────────────────────────────────────────────────────────
local function corner(parent, radius)
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, radius or 8)
    c.Parent = parent
end

local function stroke(parent, thickness, color)
    local s = Instance.new("UIStroke")
    s.Thickness       = thickness or 1
    s.Color           = color or Color3.fromRGB(80, 80, 100)
    s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    s.Parent = parent
end

-- ─── Toggle Button (always visible) ──────────────────────────────────────────
local toggleBtn = Instance.new("TextButton")
toggleBtn.Name             = "MenuToggle"
toggleBtn.Size             = UDim2.new(0, 140, 0, 32)
toggleBtn.Position         = UDim2.new(0, 16, 0, 16)
toggleBtn.BackgroundColor3 = Color3.fromRGB(20, 20, 30)
toggleBtn.BorderSizePixel  = 0
toggleBtn.TextColor3       = Color3.fromRGB(200, 200, 255)
toggleBtn.Font             = Enum.Font.GothamBold
toggleBtn.TextSize         = 13
toggleBtn.Text             = "☰  Sky Aimbot V12"
toggleBtn.ZIndex           = 10
toggleBtn.Parent           = screenGui
corner(toggleBtn, 8)
stroke(toggleBtn, 1, Color3.fromRGB(70, 70, 110))

-- ─── Collapsible Panel ───────────────────────────────────────────────────────
local panel = Instance.new("Frame")
panel.Name             = "MenuPanel"
panel.Size             = UDim2.new(0, PANEL_W, 0, 0)
panel.Position         = UDim2.new(0, 16, 0, 54)
panel.BackgroundColor3 = Color3.fromRGB(15, 15, 22)
panel.BorderSizePixel  = 0
panel.ClipsDescendants = true
panel.ZIndex           = 9
panel.Parent           = screenGui
corner(panel, 10)
stroke(panel, 1, Color3.fromRGB(60, 60, 95))

local grad = Instance.new("UIGradient")
grad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(25, 25, 40)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(12, 12, 18)),
})
grad.Rotation = 90
grad.Parent   = panel

local layout = Instance.new("UIListLayout")
layout.FillDirection       = Enum.FillDirection.Vertical
layout.HorizontalAlignment = Enum.HorizontalAlignment.Center
layout.Padding             = UDim.new(0, 6)
layout.Parent              = panel

local padding = Instance.new("UIPadding")
padding.PaddingTop    = UDim.new(0, 10)
padding.PaddingBottom = UDim.new(0, 10)
padding.PaddingLeft   = UDim.new(0, 8)
padding.PaddingRight  = UDim.new(0, 8)
padding.Parent        = panel

-- ─── Panel Child Builders ─────────────────────────────────────────────────────
local function makeBtn(text, h, color)
    local b = Instance.new("TextButton")
    b.Size             = UDim2.new(1, 0, 0, h)
    b.BackgroundColor3 = color
    b.BorderSizePixel  = 0
    b.TextColor3       = Color3.fromRGB(255, 255, 255)
    b.Font             = Enum.Font.GothamBold
    b.TextSize         = 12
    b.Text             = text
    b.ZIndex           = 10
    b.Parent           = panel
    corner(b, 6)
    return b
end

local function makeLabel(textColor, bold)
    local l = Instance.new("TextLabel")
    l.Size                   = UDim2.new(1, 0, 0, 16)
    l.BackgroundTransparency = 1
    l.TextColor3             = textColor
    l.Font                   = bold and Enum.Font.GothamBold or Enum.Font.Gotham
    l.TextSize               = 11
    l.TextXAlignment         = Enum.TextXAlignment.Left
    l.Text                   = ""
    l.ZIndex                 = 10
    l.Parent                 = panel
    return l
end

-- ─── Panel Buttons ────────────────────────────────────────────────────────────
local mainBtn     = makeBtn("Sky Aim: OFF",          44, Color3.fromRGB(40,  40,  55))
local targetBtn   = makeBtn("Target: Enemies Only",  30, Color3.fromRGB(30,  90, 160))
local espBtn      = makeBtn("ESP: OFF",              30, Color3.fromRGB(40,  40,  55))
local priorityBtn = makeBtn("Priority: Closest",     30, PRIORITY_COLORS[1])
local hitboxBtn   = makeBtn("Hitbox Expander: OFF",  30, Color3.fromRGB(40,  40,  55))

local sep = Instance.new("Frame")
sep.Size             = UDim2.new(1, 0, 0, 1)
sep.BackgroundColor3 = Color3.fromRGB(55, 55, 80)
sep.BorderSizePixel  = 0
sep.Parent           = panel

local statusLabel = makeLabel(Color3.fromRGB(170, 170, 190), false)
local modeLabel   = makeLabel(Color3.fromRGB(255, 200,  80), true)
statusLabel.Text  = "Status: idle"

-- ─── Menu tween ──────────────────────────────────────────────────────────────
local tweenInfo = TweenInfo.new(TWEEN_TIME, Enum.EasingStyle.Quint, Enum.EasingDirection.Out)

local function setMenuOpen(state)
    menuOpen = state
    TweenService:Create(panel, tweenInfo, {
        Size = UDim2.new(0, PANEL_W, 0, state and PANEL_H or 0)
    }):Play()
    if state then
        toggleBtn.Text             = "✕  Sky Aimbot V12"
        toggleBtn.BackgroundColor3 = Color3.fromRGB(35, 20, 55)
    else
        toggleBtn.Text             = "☰  Sky Aimbot V12"
        toggleBtn.BackgroundColor3 = Color3.fromRGB(20, 20, 30)
    end
end

toggleBtn.MouseButton1Click:Connect(function() setMenuOpen(not menuOpen) end)

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == Enum.KeyCode.K then setMenuOpen(not menuOpen) end
end)

-- ─── ESP Overlay ScreenGui ───────────────────────────────────────────────────
local espGui = Instance.new("ScreenGui")
espGui.Name           = "SkyAimbotV12ESPGui"
espGui.ResetOnSpawn   = false
espGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
espGui.IgnoreGuiInset = true
espGui.Parent         = LocalPlayer:WaitForChild("PlayerGui")

local function makeESPLabel(color, size)
    local l = Instance.new("TextLabel")
    l.BackgroundTransparency = 1
    l.TextColor3             = color
    l.Font                   = Enum.Font.GothamBold
    l.TextSize               = size or 13
    l.TextStrokeTransparency = 0.35
    l.TextStrokeColor3       = Color3.fromRGB(0, 0, 0)
    l.Size                   = UDim2.new(0, 220, 0, 20)
    l.AnchorPoint            = Vector2.new(0.5, 0.5)
    l.Visible                = false
    l.Text                   = ""
    l.Parent                 = espGui
    return l
end

-- ─── Core Helpers ────────────────────────────────────────────────────────────
local function makeRayParams()
    local p = RaycastParams.new()
    p.FilterType = Enum.RaycastFilterType.Exclude
    local char = LocalPlayer.Character
    if char then p.FilterDescendantsInstances = { char } end
    return p
end

local function hasLineOfSight(origin, targetPos)
    local result = workspace:Raycast(origin, targetPos - origin, makeRayParams())
    if not result then return true end
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character then
            if result.Instance:IsDescendantOf(player.Character) then return true end
        end
    end
    return false
end

local function isValidTarget(player)
    if player == LocalPlayer then return false end
    if not targetAll then
        local myTeam = LocalPlayer.Team
        if myTeam and player.Team == myTeam then return false end
    end
    local char = player.Character
    local hum  = char and char:FindFirstChild("Humanoid")
    return char ~= nil and hum ~= nil and hum.Health > 0
end

local function getSortedTargets()
    local myChar = LocalPlayer.Character
    local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
    if not myRoot then return {} end

    local mousePos = UserInputService:GetMouseLocation()
    local list     = {}

    for _, player in ipairs(Players:GetPlayers()) do
        if not isValidTarget(player) then continue end
        local char = player.Character
        local root = char and char:FindFirstChild("HumanoidRootPart")
        local hum  = char and char:FindFirstChild("Humanoid")
        if not root or not hum then continue end

        local sp, inView = Camera:WorldToViewportPoint(root.Position)
        table.insert(list, {
            root      = root,
            player    = player,
            name      = player.Name,
            dist3D    = (root.Position - myRoot.Position).Magnitude,
            distMouse = inView and (Vector2.new(sp.X, sp.Y) - mousePos).Magnitude or math.huge,
            health    = hum.Health,
        })
    end

    if priorityIndex == 1 then
        table.sort(list, function(a, b) return a.dist3D    < b.dist3D    end)
    elseif priorityIndex == 2 then
        table.sort(list, function(a, b) return a.distMouse < b.distMouse end)
    elseif priorityIndex == 3 then
        table.sort(list, function(a, b) return a.health    > b.health    end)
    end

    return list
end

local function getNearestVisibleTarget()
    local myChar = LocalPlayer.Character
    local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
    if not myRoot then return nil end
    local origin = myRoot.Position + Vector3.new(0, 2, 0)
    for _, e in ipairs(getSortedTargets()) do
        if hasLineOfSight(origin, e.root.Position + Vector3.new(0, 1.5, 0)) then
            return e.root, e.dist3D, e.name
        end
    end
    return nil
end

local function trackAboveAndBehind()
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then return nil end

    local targets = getSortedTargets()
    if #targets > 0 then
        local t        = targets[1].root
        local lookFlat = t.CFrame.LookVector * Vector3.new(1, 0, 1)
        if lookFlat.Magnitude < 0.01 then lookFlat = Vector3.new(0, 0, 1) end
        lookFlat    = lookFlat.Unit
        local behind = -lookFlat * BEHIND_DISTANCE
        local ty     = math.max(t.Position.Y + HOVER_HEIGHT, MIN_HEIGHT)
        root.CFrame  = CFrame.new(
            t.Position.X + behind.X, ty, t.Position.Z + behind.Z
        )
        return targets[1].name
    else
        local angle = math.random() * 2 * math.pi
        local r     = math.random(50, FALLBACK_RADIUS)
        local y     = math.max(FALLBACK_HEIGHT + math.random(-30, 30), MIN_HEIGHT)
        root.CFrame = CFrame.new(math.cos(angle) * r, y, math.sin(angle) * r)
        return nil
    end
end

local function fireWeapon()
    local char = LocalPlayer.Character
    if not char then return end
    for _, item in ipairs(char:GetChildren()) do
        if item:IsA("Tool") then item:Activate(); break end
    end
end

local function spawnTracer(fromPos, toPos)
    local dir = toPos - fromPos
    local len = dir.Magnitude
    if len < 0.1 then return end
    local t        = Instance.new("Part")
    t.Name         = "BulletTracer"
    t.Anchored     = true
    t.CanCollide   = false
    t.CastShadow   = false
    t.Material     = Enum.Material.Neon
    t.Color        = TRACER_COLOR
    t.Transparency = 0
    t.Size         = Vector3.new(TRACER_WIDTH, TRACER_WIDTH, len)
    t.CFrame       = CFrame.new(fromPos + dir * 0.5, toPos)
    t.Parent       = workspace
    task.spawn(function()
        local steps = 20
        local iv    = TRACER_LIFETIME / steps
        for i = 1, steps do
            task.wait(iv)
            if not t or not t.Parent then break end
            t.Transparency = i / steps
        end
        if t and t.Parent then t:Destroy() end
    end)
end

-- ─── Hitbox Expander ─────────────────────────────────────────────────────────

-- Creates (or recycles) a SelectionBox wireframe for the given player's root.
-- The SelectionBox Adornee is the HumanoidRootPart, so it automatically
-- tracks the part's position & orientation and sizes itself to match.
local function createHitboxVisual(player)
    if hitboxVisuals[player] then return end

    local char = player.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then return end

    local sb = Instance.new("SelectionBox")
    sb.Name                = "HitboxVisual"
    sb.Color3              = HITBOX_LINE_COLOR
    sb.LineThickness       = HITBOX_LINE_THICK
    sb.SurfaceColor3       = HITBOX_FILL_COLOR
    sb.SurfaceTransparency = HITBOX_FILL_ALPHA
    sb.Adornee             = root
    sb.Parent              = workspace     -- must be in workspace, not a ScreenGui

    hitboxVisuals[player] = sb
end

local function removeHitboxVisual(player)
    local sb = hitboxVisuals[player]
    if sb then
        pcall(function() sb:Destroy() end)
        hitboxVisuals[player] = nil
    end
end

local function clearAllHitboxVisuals()
    for player in pairs(hitboxVisuals) do removeHitboxVisual(player) end
end

-- Applies or removes the 8× scale and matching visual
local function applyHitbox(player, expand)
    local char = player and player.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then return end

    if expand then
        if not originalSizes[player] then
            originalSizes[player] = root.Size
        end
        root.Size = originalSizes[player] * HITBOX_SCALE

        -- Attach or refresh the SelectionBox
        createHitboxVisual(player)
        -- Keep adornee current (character may have respawned)
        if hitboxVisuals[player] then
            hitboxVisuals[player].Adornee = root
        end
    else
        if originalSizes[player] then
            root.Size            = originalSizes[player]
            originalSizes[player] = nil
        end
        removeHitboxVisual(player)
    end
end

local function hookHitboxPlayer(player)
    if hitboxConns[player] then return end
    hitboxConns[player] = player.CharacterAdded:Connect(function()
        task.wait(0.1)
        if hitboxEnabled then applyHitbox(player, true) end
    end)
end

local function setHitbox(state)
    hitboxEnabled = state
    if state then
        hitboxBtn.Text             = "Hitbox Expander: ON"
        hitboxBtn.BackgroundColor3 = Color3.fromRGB(200, 80, 0)
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then
                hookHitboxPlayer(player)
                applyHitbox(player, true)
            end
        end
    else
        hitboxBtn.Text             = "Hitbox Expander: OFF"
        hitboxBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 55)
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then applyHitbox(player, false) end
        end
        clearAllHitboxVisuals()
    end
end

-- ─── ESP Logic ───────────────────────────────────────────────────────────────
local function createESP(player)
    if espObjects[player] then return end
    local hl = Instance.new("Highlight")
    hl.Name                = "V12ESP"
    hl.FillColor           = ESP_FILL_COLOR
    hl.OutlineColor        = ESP_OUTLINE_COLOR
    hl.FillTransparency    = 0.45
    hl.OutlineTransparency = 0
    hl.DepthMode           = Enum.HighlightDepthMode.AlwaysOnTop
    if player.Character then
        hl.Adornee = player.Character
        hl.Parent  = player.Character
    end
    player.CharacterAdded:Connect(function(char)
        if espObjects[player] then
            espObjects[player].highlight.Adornee = char
            espObjects[player].highlight.Parent  = char
        end
    end)
    espObjects[player] = {
        highlight   = hl,
        nameLabel   = makeESPLabel(ESP_NAME_COLOR, 14),
        hpLabel     = makeESPLabel(ESP_HP_COLOR,   12),
        weaponLabel = makeESPLabel(ESP_WEP_COLOR,  11),
    }
end

local function removeESP(player)
    local obj = espObjects[player]
    if not obj then return end
    pcall(function() obj.highlight:Destroy()   end)
    pcall(function() obj.nameLabel:Destroy()   end)
    pcall(function() obj.hpLabel:Destroy()     end)
    pcall(function() obj.weaponLabel:Destroy() end)
    espObjects[player] = nil
end

local function clearAllESP()
    for player in pairs(espObjects) do removeESP(player) end
end

local function updateESP()
    for _, player in ipairs(Players:GetPlayers()) do
        if player == LocalPlayer then continue end
        local shouldShow = espEnabled and isValidTarget(player)
        local obj        = espObjects[player]

        if shouldShow then
            if not obj then createESP(player); obj = espObjects[player] end
            if not obj then continue end

            obj.highlight.FillTransparency    = 0.45
            obj.highlight.OutlineTransparency = 0

            local char = player.Character
            local root = char and char:FindFirstChild("HumanoidRootPart")
            local head = char and char:FindFirstChild("Head")
            local hum  = char and char:FindFirstChild("Humanoid")

            if not root or not head or not hum then
                obj.nameLabel.Visible   = false
                obj.hpLabel.Visible     = false
                obj.weaponLabel.Visible = false
                continue
            end

            local function ws(pos)
                local sp, iv = Camera:WorldToViewportPoint(pos)
                return Vector2.new(sp.X, sp.Y), iv
            end

            local namePos, nameOn = ws(head.Position + Vector3.new(0, 2.5, 0))
            obj.nameLabel.Visible  = nameOn
            obj.nameLabel.Position = UDim2.fromOffset(namePos.X, namePos.Y)
            obj.nameLabel.Text     = player.Name

            local hpPos, hpOn = ws(root.Position + Vector3.new(4, 0, 0))
            local hpPct       = math.floor((hum.Health / hum.MaxHealth) * 100)
            obj.hpLabel.Visible  = hpOn
            obj.hpLabel.Position = UDim2.fromOffset(hpPos.X, hpPos.Y)
            obj.hpLabel.Text     = string.format("HP  %d%%", hpPct)

            local wepPos, wepOn = ws(root.Position - Vector3.new(0, 4, 0))
            local tool          = char:FindFirstChildOfClass("Tool")
            obj.weaponLabel.Visible  = wepOn
            obj.weaponLabel.Position = UDim2.fromOffset(wepPos.X, wepPos.Y)
            obj.weaponLabel.Text     = tool and tool.Name or "(no weapon)"
        elseif obj then
            obj.nameLabel.Visible          = false
            obj.hpLabel.Visible            = false
            obj.weaponLabel.Visible        = false
            obj.highlight.FillTransparency    = 1
            obj.highlight.OutlineTransparency = 1
        end
    end
end

-- ─── Toggle Logic ─────────────────────────────────────────────────────────────
local function setEnabled(state)
    enabled = state
    if enabled then
        mainBtn.Text             = "Sky Aim: ON"
        mainBtn.BackgroundColor3 = Color3.fromRGB(140, 0, 200)
        originalCamType          = Camera.CameraType
        Camera.CameraType        = Enum.CameraType.Scriptable
        statusLabel.Text         = "Status: active"
        modeLabel.Text           = "Searching…"
    else
        mainBtn.Text             = "Sky Aim: OFF"
        mainBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 55)
        statusLabel.Text         = "Status: idle"
        modeLabel.Text           = ""
        Camera.CameraType        = originalCamType or Enum.CameraType.Custom
    end
end

local function setESP(state)
    espEnabled = state
    if state then
        espBtn.Text             = "ESP: ON"
        espBtn.BackgroundColor3 = Color3.fromRGB(0, 130, 220)
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= LocalPlayer then createESP(p) end
        end
    else
        espBtn.Text             = "ESP: OFF"
        espBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 55)
        clearAllESP()
    end
end

local function cyclePriority()
    priorityIndex = (priorityIndex % #PRIORITY_MODES) + 1
    priorityBtn.Text             = "Priority: " .. PRIORITY_MODES[priorityIndex]
    priorityBtn.BackgroundColor3 = PRIORITY_COLORS[priorityIndex]
end

-- ─── Button Connections ───────────────────────────────────────────────────────
mainBtn.MouseButton1Click:Connect(function()     setEnabled(not enabled)      end)
targetBtn.MouseButton1Click:Connect(function()
    targetAll = not targetAll
    if targetAll then
        targetBtn.Text             = "Target: All Players"
        targetBtn.BackgroundColor3 = Color3.fromRGB(180, 40, 40)
    else
        targetBtn.Text             = "Target: Enemies Only"
        targetBtn.BackgroundColor3 = Color3.fromRGB(30, 90, 160)
    end
end)
espBtn.MouseButton1Click:Connect(function()      setESP(not espEnabled)       end)
priorityBtn.MouseButton1Click:Connect(function() cyclePriority()              end)
hitboxBtn.MouseButton1Click:Connect(function()   setHitbox(not hitboxEnabled) end)

-- New players joining / leaving
Players.PlayerAdded:Connect(function(player)
    if hitboxEnabled then
        hookHitboxPlayer(player)
        player.CharacterAdded:Wait()
        task.wait(0.1)
        applyHitbox(player, true)
    end
end)

Players.PlayerRemoving:Connect(function(player)
    removeESP(player)
    removeHitboxVisual(player)
    originalSizes[player] = nil
    if hitboxConns[player] then
        hitboxConns[player]:Disconnect()
        hitboxConns[player] = nil
    end
end)

-- ─── Main Loop ───────────────────────────────────────────────────────────────
RunService.Heartbeat:Connect(function()

    -- ESP (always runs)
    updateESP()

    -- Hitbox — re-apply every frame so any server overwrite is undone
    if hitboxEnabled then
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then
                applyHitbox(player, true)
            end
        end
    end

    if not enabled then return end

    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then return end

    -- Height floor
    if root.Position.Y < MIN_HEIGHT then
        root.CFrame = CFrame.new(root.Position.X, MIN_HEIGHT, root.Position.Z)
    end

    -- Counter gravity
    if root:IsA("BasePart") then
        root.AssemblyLinearVelocity = Vector3.new(
            root.AssemblyLinearVelocity.X, 0, root.AssemblyLinearVelocity.Z
        )
    end

    local now = tick()
    local target, dist, targetName = getNearestVisibleTarget()

    if target then
        modeLabel.Text       = string.format("LOCKED · %s", PRIORITY_MODES[priorityIndex])
        modeLabel.TextColor3 = Color3.fromRGB(255, 80, 80)
        statusLabel.Text     = string.format("%s  %.0f st", targetName, dist)

        local aimPos = target.Position + Vector3.new(0, 1.5, 0)
        local camPos = root.Position   + Vector3.new(0, 2,   0)

        local downDir = aimPos - root.Position
        if downDir.Magnitude > 0.01 then
            root.CFrame = CFrame.new(root.Position, root.Position + downDir)
        end

        Camera.CFrame = Camera.CFrame:Lerp(CFrame.new(camPos, aimPos), 1 - AIM_SMOOTH)

        if now - lastFireTime >= FIRE_RATE then
            lastFireTime = now
            fireWeapon()
            spawnTracer(camPos, aimPos)
        end
    else
        modeLabel.TextColor3 = Color3.fromRGB(180, 100, 255)
        local name = trackAboveAndBehind()
        if name then
            modeLabel.Text   = string.format("Hunting · %s", PRIORITY_MODES[priorityIndex])
            statusLabel.Text = string.format("Behind: %s", name)
        else
            modeLabel.Text   = "No targets — roaming"
            statusLabel.Text = "Searching…"
        end
    end
end)

-- Reset on respawn
LocalPlayer.CharacterRemoving:Connect(function()
    if enabled then
        Camera.CameraType        = originalCamType or Enum.CameraType.Custom
        enabled                  = false
        mainBtn.Text             = "Sky Aim: OFF"
        mainBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 55)
        statusLabel.Text         = "Status: idle"
        modeLabel.Text           = ""
    end
end)
