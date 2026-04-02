--[[
   ██████╗ █████╗ ████████╗██╗  ██╗██╗   ██╗██████╗
  ██╔════╝██╔══██╗╚══██╔══╝██║  ██║██║   ██║██╔══██╗
  ██║     ███████║   ██║   ███████║██║   ██║██████╔╝
  ██║     ██╔══██║   ██║   ██╔══██║██║   ██║██╔══██╗
  ╚██████╗██║  ██║   ██║   ██║  ██║╚██████╔╝██████╔╝
   ╚═════╝╚═╝  ╚═╝   ╚═╝   ╚═╝  ╚═╝ ╚═════╝ ╚═════╝
  CatHub v3.0
  ⚠ DISCLAIMER:
    [LOCAL] = Only visible to YOU.  Other players cannot see it.
    [GAME]  = Only works in certain games / experiences.
    Some features may not work depending on the game's anti-cheat.
]]

-- ═══════════════════════════════════════════════
--  SERVICES
-- ═══════════════════════════════════════════════
local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService     = game:GetService("TweenService")
local CoreGui          = game:GetService("CoreGui")
local Lighting         = game:GetService("Lighting")
local StarterGui       = game:GetService("StarterGui")

local LP    = Players.LocalPlayer
local Mouse = LP:GetMouse()
local Cam   = workspace.CurrentCamera

local isMobile = UserInputService.TouchEnabled and not UserInputService.KeyboardEnabled

-- ═══════════════════════════════════════════════
--  HELPERS
-- ═══════════════════════════════════════════════
local function getChar()  return LP.Character end
local function getHRP()
    local c = getChar()
    return c and c:FindFirstChild("HumanoidRootPart")
end
local function getHuman()
    local c = getChar()
    return c and c:FindFirstChildOfClass("Humanoid")
end
local function disc(c)
    if typeof(c) == "RBXScriptConnection" then pcall(c.Disconnect, c) end
end
local function notify(t, m, d)
    pcall(StarterGui.SetCore, StarterGui, "SendNotification",
        { Title = t, Text = m, Duration = d or 4 })
end

-- ═══════════════════════════════════════════════
--  SCROLL-SAFE TAP DETECTION
--  Prevents scroll from accidentally toggling hacks
-- ═══════════════════════════════════════════════
local _tapStart   = Vector2.zero
local _tapMoved   = false
local TAP_THRESH  = 14  -- pixels of movement before we call it a scroll

UserInputService.InputBegan:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.Touch then
        _tapStart = Vector2.new(i.Position.X, i.Position.Y)
        _tapMoved = false
    end
end)
UserInputService.InputChanged:Connect(function(i)
    if i.UserInputType == Enum.UserInputType.Touch then
        if (Vector2.new(i.Position.X, i.Position.Y) - _tapStart).Magnitude > TAP_THRESH then
            _tapMoved = true
        end
    end
end)
-- Returns true only if the current touch is a genuine tap (not a scroll)
local function wasTap() return not _tapMoved end

-- ═══════════════════════════════════════════════
--  STATE
-- ═══════════════════════════════════════════════
local S = {
    -- Movement
    speedHack   = false,  walkSpeed  = 50,
    jumpHack    = false,  jumpPower  = 80,
    infJump     = false,
    fly         = false,  flySpeed   = 60,
    noclip      = false,
    lowGrav     = false,
    bhop        = false,
    spin        = false,  spinSpeed  = 5,
    safeFall    = false,
    loopTP      = false,
    -- Combat
    hitbox      = false,  hitboxSz   = 10,
    killAura    = false,  auraRange  = 15,
    antiRag     = false,
    faceNear    = false,
    btools      = false,
    silentAim   = false,
    -- Player
    godMode     = false,
    invis       = false,
    headless    = false,
    freeze      = false,
    charScale   = 10,  -- 10 = 1.0x, range 5–25
    noJumpAnim  = false,
    platformSt  = false,
    -- Visual
    nameESP     = false,
    hpESP       = false,
    boxESP      = false,
    fullbright  = false,
    noFog       = false,
    daytime     = false,
    fov         = 70,
    thirdPerson = false,  tpDist = 15,
    -- Utility
    antiAFK     = false,
    autoCol     = false,
    chatSpam    = false,  chatMsg   = "CatHub v3",  chatDelay = 3,
    -- Internal
    conns       = {},
    espObjs     = {},
    flyBV       = nil,
    flyBG       = nil,
    waypoints   = {},
}

-- ═══════════════════════════════════════════════
--  FEATURE IMPLEMENTATIONS
-- ═══════════════════════════════════════════════

-- Speed
local function applySpeed()
    local h = getHuman()
    if h then h.WalkSpeed = S.speedHack and S.walkSpeed or 16 end
end

-- Jump
local function applyJump()
    local h = getHuman()
    if h then h.JumpPower = S.jumpHack and S.jumpPower or 50 end
end

-- Gravity / FOV
local function applyGrav() workspace.Gravity = S.lowGrav and 25 or 196.2 end
local function applyFOV()  Cam.FieldOfView   = S.fov end

-- Infinite jump
UserInputService.JumpRequest:Connect(function()
    if S.infJump then
        local h = getHuman()
        if h then h:ChangeState(Enum.HumanoidStateType.Jumping) end
    end
end)

-- Fly
local function enableFly()
    local hrp = getHRP(); if not hrp then return end
    if S.flyBV then S.flyBV:Destroy() end
    if S.flyBG then S.flyBG:Destroy() end
    local bv = Instance.new("BodyVelocity", hrp)
    bv.MaxForce = Vector3.new(9e9, 9e9, 9e9); bv.Velocity = Vector3.zero
    local bg = Instance.new("BodyGyro", hrp)
    bg.MaxTorque = Vector3.new(9e9, 9e9, 9e9); bg.P = 9e4
    S.flyBV = bv; S.flyBG = bg
    disc(S.conns.fly)
    S.conns.fly = RunService.RenderStepped:Connect(function()
        if not S.fly then return end
        local spd = S.flySpeed; local v = Vector3.zero; local cf = Cam.CFrame
        if UserInputService:IsKeyDown(Enum.KeyCode.W)           then v += cf.LookVector  * spd end
        if UserInputService:IsKeyDown(Enum.KeyCode.S)           then v -= cf.LookVector  * spd end
        if UserInputService:IsKeyDown(Enum.KeyCode.A)           then v -= cf.RightVector * spd end
        if UserInputService:IsKeyDown(Enum.KeyCode.D)           then v += cf.RightVector * spd end
        if UserInputService:IsKeyDown(Enum.KeyCode.Space)       then v += Vector3.new(0, spd, 0) end
        if UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then v -= Vector3.new(0, spd, 0) end
        if isMobile and v == Vector3.zero then v = cf.LookVector * (spd * 0.4) end
        bv.Velocity = v; bg.CFrame = cf
    end)
end
local function disableFly()
    disc(S.conns.fly); S.conns.fly = nil
    if S.flyBV then S.flyBV:Destroy(); S.flyBV = nil end
    if S.flyBG then S.flyBG:Destroy(); S.flyBG = nil end
end

-- Noclip
local function applyNoclip()
    disc(S.conns.nc); S.conns.nc = nil
    if not S.noclip then return end
    S.conns.nc = RunService.Stepped:Connect(function()
        local c = getChar(); if not c then return end
        for _, p in ipairs(c:GetDescendants()) do
            if p:IsA("BasePart") then p.CanCollide = false end
        end
    end)
end

-- Bhop
local function applyBhop()
    disc(S.conns.bh); S.conns.bh = nil
    if not S.bhop then return end
    S.conns.bh = RunService.Stepped:Connect(function()
        local h = getHuman()
        if h and h.FloorMaterial ~= Enum.Material.Air then
            h:ChangeState(Enum.HumanoidStateType.Jumping)
        end
    end)
end

-- Spin
local function applySpin()
    disc(S.conns.sp); S.conns.sp = nil
    if not S.spin then return end
    S.conns.sp = RunService.Heartbeat:Connect(function()
        local hrp = getHRP()
        if hrp then hrp.CFrame *= CFrame.Angles(0, math.rad(S.spinSpeed), 0) end
    end)
end

-- Safe fall
local function applySafeFall()
    disc(S.conns.sf); S.conns.sf = nil
    if not S.safeFall then return end
    S.conns.sf = RunService.Stepped:Connect(function()
        local h = getHuman()
        if h then h:ChangeState(Enum.HumanoidStateType.Landed) end
    end)
end

-- Loop TP to cursor
local function applyLoopTP()
    disc(S.conns.lt); S.conns.lt = nil
    if not S.loopTP then return end
    S.conns.lt = RunService.Heartbeat:Connect(function()
        local hrp = getHRP()
        if hrp and Mouse.Target then
            hrp.CFrame = CFrame.new(Mouse.Hit.Position + Vector3.new(0, 3, 0))
        end
    end)
end

-- Third-person camera [LOCAL]
local function applyThirdPerson()
    disc(S.conns.tp); S.conns.tp = nil
    if not S.thirdPerson then
        Cam.CameraType = Enum.CameraType.Custom
        return
    end
    Cam.CameraType = Enum.CameraType.Scriptable
    S.conns.tp = RunService.RenderStepped:Connect(function()
        local hrp = getHRP(); if not hrp then return end
        Cam.CFrame = CFrame.new(
            hrp.Position + Vector3.new(0, 4, 0) - Cam.CFrame.LookVector * S.tpDist,
            hrp.Position + Vector3.new(0, 2, 0)
        )
    end)
end

-- God mode
local function applyGod()
    local h = getHuman(); if not h then return end
    if S.godMode then
        h.MaxHealth = math.huge
        h.Health    = math.huge
    else
        h.MaxHealth = 100
        h.Health    = 100
    end
end

-- Invisible [LOCAL]
local function applyInvis()
    local c = getChar(); if not c then return end
    for _, p in ipairs(c:GetDescendants()) do
        if p:IsA("BasePart") and p.Name ~= "HumanoidRootPart" then
            p.LocalTransparencyModifier = S.invis and 1 or 0
        end
        if p:IsA("Decal") then p.Transparency = S.invis and 1 or 0 end
    end
end

-- Headless [LOCAL]
local function applyHeadless()
    local c = getChar(); if not c then return end
    local head = c:FindFirstChild("Head"); if not head then return end
    if S.headless then
        head.Size = Vector3.new(0.001, 0.001, 0.001)
        for _, d in ipairs(head:GetChildren()) do
            if d:IsA("Decal") then d.Transparency = 1 end
            if d:IsA("SpecialMesh") then d.Scale = Vector3.new(0,0,0) end
        end
    else
        head.Size = Vector3.new(2, 1, 1)
        for _, d in ipairs(head:GetChildren()) do
            if d:IsA("Decal") then d.Transparency = 0 end
            if d:IsA("SpecialMesh") then d.Scale = Vector3.new(1,1,1) end
        end
    end
end

-- Freeze self
local function applyFreeze()
    local hrp = getHRP(); if hrp then hrp.Anchored = S.freeze end
end

-- Platform stand [LOCAL]
local function applyPlatformSt()
    local h = getHuman(); if not h then return end
    h.PlatformStand = S.platformSt
end

-- Character scale [LOCAL] — fixed implementation
local function applyScale()
    local c = getChar(); if not c then return end
    local h = c:FindFirstChildOfClass("Humanoid"); if not h then return end
    local sc = S.charScale / 10  -- convert: slider 10 → scale 1.0
    -- These NumberValues control Roblox R15 body scale
    local scaleProps = {
        BodyDepthScale  = sc,
        BodyHeightScale = sc,
        BodyWidthScale  = sc,
        HeadScale       = sc,
    }
    for name, val in pairs(scaleProps) do
        local nv = h:FindFirstChild(name)
        if not nv then
            nv = Instance.new("NumberValue", h)
            nv.Name = name
        end
        nv.Value = val
    end
    -- Also works on R6 by scaling the HRP / Torso directly
    local hrp = c:FindFirstChild("HumanoidRootPart")
    if hrp then
        -- R6 fallback: adjust size
        local torso = c:FindFirstChild("Torso") or c:FindFirstChild("UpperTorso")
        if torso then
            -- Only do R6 manual scaling if no BodyHeightScale responded
            -- (skip if R15 scale values exist)
        end
    end
end

-- Hitbox expander [LOCAL]
local function applyHitbox()
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LP and p.Character then
            local hrp = p.Character:FindFirstChild("HumanoidRootPart")
            if hrp then
                hrp.Size = S.hitbox
                    and Vector3.new(S.hitboxSz, S.hitboxSz, S.hitboxSz)
                    or  Vector3.new(2, 2, 1)
            end
        end
    end
end

-- Kill aura [GAME]
local function applyKillAura()
    disc(S.conns.ka); S.conns.ka = nil
    if not S.killAura then return end
    S.conns.ka = RunService.Heartbeat:Connect(function()
        local hrp = getHRP(); if not hrp then return end
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= LP and p.Character then
                local ep = p.Character:FindFirstChild("HumanoidRootPart")
                if ep and (ep.Position - hrp.Position).Magnitude <= S.auraRange then
                    local h = p.Character:FindFirstChildOfClass("Humanoid")
                    if h then h.Health = 0 end
                end
            end
        end
    end)
end

-- Face nearest
local function applyFace()
    disc(S.conns.fn); S.conns.fn = nil
    if not S.faceNear then return end
    S.conns.fn = RunService.Heartbeat:Connect(function()
        local hrp = getHRP(); if not hrp then return end
        local near, bd = nil, math.huge
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= LP and p.Character then
                local ep = p.Character:FindFirstChild("HumanoidRootPart")
                if ep then
                    local d = (ep.Position - hrp.Position).Magnitude
                    if d < bd then near, bd = ep, d end
                end
            end
        end
        if near then
            hrp.CFrame = CFrame.new(hrp.Position,
                Vector3.new(near.Position.X, hrp.Position.Y, near.Position.Z))
        end
    end)
end

-- Silent aim [LOCAL] — redirects bullets toward nearest player
local function applySilentAim()
    disc(S.conns.sa); S.conns.sa = nil
    if not S.silentAim then return end
    S.conns.sa = RunService.RenderStepped:Connect(function()
        local hrp = getHRP(); if not hrp then return end
        local near, bd = nil, math.huge
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= LP and p.Character then
                local ep = p.Character:FindFirstChild("HumanoidRootPart")
                if ep then
                    local sp, onScreen = Cam:WorldToViewportPoint(ep.Position)
                    if onScreen then
                        local d = (ep.Position - hrp.Position).Magnitude
                        if d < bd then near, bd = ep, d end
                    end
                end
            end
        end
        if near then
            local sp = Cam:WorldToViewportPoint(near.Position)
            Mouse.Icon = ""
            -- move roblox mouse to target (executor-dependent)
            pcall(function()
                mousemoverel(
                    sp.X - Mouse.X,
                    sp.Y - Mouse.Y
                )
            end)
        end
    end)
end

-- BTools
local function applyBtools()
    disc(S.conns.bt); S.conns.bt = nil
    if not S.btools then return end
    S.conns.bt = Mouse.Button1Down:Connect(function()
        local t = Mouse.Target
        if t and not t:IsDescendantOf(getChar() or Instance.new("Folder")) then
            t:Destroy()
        end
    end)
end

-- Fullbright [LOCAL]
local function applyFullbright()
    Lighting.Ambient        = S.fullbright and Color3.fromRGB(255,255,255) or Color3.fromRGB(70,70,70)
    Lighting.Brightness     = S.fullbright and 2 or 1
    Lighting.OutdoorAmbient = S.fullbright and Color3.fromRGB(255,255,255) or Color3.fromRGB(127,127,127)
end

-- Remove fog [LOCAL]
local function applyFog()
    Lighting.FogEnd   = S.noFog and 1e9 or 100000
    Lighting.FogStart = S.noFog and 1e9 or 0
end

-- Daytime [LOCAL]
local function applyDay()
    if S.daytime then Lighting.ClockTime = 12 end
end

-- Anti-AFK
local function applyAFK()
    disc(S.conns.afk); S.conns.afk = nil
    if not S.antiAFK then return end
    S.conns.afk = RunService.Heartbeat:Connect(function()
        pcall(function() game:GetService("VirtualUser"):ClickButton2(Vector2.new()) end)
    end)
end

-- Auto-collect [GAME]
local function applyCollect()
    disc(S.conns.ac); S.conns.ac = nil
    if not S.autoCol then return end
    S.conns.ac = RunService.Heartbeat:Connect(function()
        local hrp = getHRP(); if not hrp then return end
        for _, d in ipairs(workspace:GetDescendants()) do
            if d:IsA("BasePart") then
                local n = d.Name:lower()
                if n:find("coin") or n:find("gem") or n:find("cash")
                    or n:find("orb") or n:find("token") or n:find("drop") then
                    if (d.Position - hrp.Position).Magnitude < 60 then
                        hrp.CFrame = CFrame.new(d.Position)
                        task.wait()
                    end
                end
            end
        end
    end)
end

-- Chat spam [GAME]
local function applyChatSpam()
    disc(S.conns.cs); S.conns.cs = nil
    if not S.chatSpam then return end
    local token = {}; S.conns.cs = token
    coroutine.wrap(function()
        while S.conns.cs == token do
            pcall(function()
                game:GetService("ReplicatedStorage")
                    .DefaultChatSystemChatEvents
                    .SayMessageRequest:FireServer(S.chatMsg, "All")
            end)
            task.wait(S.chatDelay)
        end
    end)()
end

-- ═══════════════════════════════════════════════
--  ESP — BillboardGui + SelectionBox
--  Works on ALL executors (no Drawing lib needed)
-- ═══════════════════════════════════════════════
local espFolder = Instance.new("Folder")
espFolder.Name   = "CatHubESP"
espFolder.Parent = workspace

local function clearAllESP()
    for _, o in ipairs(S.espObjs) do pcall(function() o:Destroy() end) end
    S.espObjs = {}
end

local function buildESPFor(player)
    if player == LP then return end
    local char  = player.Character;                        if not char  then return end
    local hrp   = char:FindFirstChild("HumanoidRootPart"); if not hrp   then return end
    local human = char:FindFirstChildOfClass("Humanoid");  if not human then return end

    -- Billboard
    local bb = Instance.new("BillboardGui")
    bb.Name         = "ESP_" .. player.Name
    bb.Adornee      = hrp
    bb.Size         = UDim2.new(0, 120, 0, 52)
    bb.StudsOffset  = Vector3.new(0, 3.5, 0)
    bb.AlwaysOnTop  = true
    bb.ResetOnSpawn = false
    bb.Parent       = espFolder
    table.insert(S.espObjs, bb)

    if S.nameESP then
        local nl = Instance.new("TextLabel", bb)
        nl.Size               = UDim2.new(1, 0, 0.55, 0)
        nl.BackgroundTransparency = 1
        nl.Text               = player.Name
        nl.TextColor3         = Color3.fromRGB(255, 220, 50)
        nl.TextSize           = 14
        nl.Font               = Enum.Font.GothamBold
        nl.TextStrokeTransparency = 0.3
        nl.TextStrokeColor3   = Color3.new(0,0,0)
        nl.TextScaled         = false
        nl.ZIndex             = 2
    end

    if S.hpESP then
        local bg = Instance.new("Frame", bb)
        bg.Size             = UDim2.new(1, -8, 0, 7)
        bg.Position         = UDim2.new(0, 4, 1, -9)
        bg.BackgroundColor3 = Color3.fromRGB(35, 8, 8)
        bg.BorderSizePixel  = 0; bg.ZIndex = 2
        local bgc = Instance.new("UICorner", bg); bgc.CornerRadius = UDim.new(0, 3)

        local fill = Instance.new("Frame", bg)
        fill.BorderSizePixel = 0; fill.ZIndex = 3
        local pct0 = math.clamp(human.Health / math.max(human.MaxHealth, 1), 0, 1)
        fill.Size             = UDim2.new(pct0, 0, 1, 0)
        fill.BackgroundColor3 = Color3.fromRGB(55 + math.floor(200*(1-pct0)), 55 + math.floor(150*pct0), 40)
        local fc = Instance.new("UICorner", fill); fc.CornerRadius = UDim.new(0, 3)

        local hc
        hc = RunService.Heartbeat:Connect(function()
            if not bb.Parent or not human.Parent then disc(hc); return end
            local pct = math.clamp(human.Health / math.max(human.MaxHealth, 1), 0, 1)
            fill.Size             = UDim2.new(pct, 0, 1, 0)
            fill.BackgroundColor3 = Color3.fromRGB(
                55 + math.floor(200*(1-pct)),
                55 + math.floor(150*pct),
                40)
        end)
    end

    if S.boxESP then
        local sb = Instance.new("SelectionBox")
        sb.Adornee           = char
        sb.Color3            = Color3.fromRGB(255, 50, 50)
        sb.LineThick