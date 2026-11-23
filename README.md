--========================================================
-- BLACKHAT HUB | THANOS SIMULATOR
-- Author: blackhat
--========================================================

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local VirtualUser = game:GetService("VirtualUser")
local Workspace = game:GetService("Workspace")

local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")
local Camera = Workspace.CurrentCamera

local ICON_ID = "rbxthumb://type=Asset&id=103525855196551&w=150&h=150"

--========================================================
-- AYARLAR & DURUMLAR
--========================================================

local Config = {
    Fly = false,
    FlySpeed = 75,
    SpeedHack = false,
    WalkSpeed = 48,
    Noclip = false,
    PlayerESP = false,
    CrystalAssist = false,
    CrystalAutoTP = false,
    CrystalHitboxExpand = true,
    CrystalAutoClick = true,
    World5Assist = false,
    AntiAFK = true
}

local State = {
    BodyGyro = nil,
    BodyVelocity = nil,
    ESPObjects = {},
    CrystalIndex = 0,
    ExpandedParts = {},
    LastClick = 0
}

--========================================================
-- KARAKTER & YARDIMCI FONKSİYONLAR
--========================================================

local function GetChar()
    local char = LocalPlayer.Character
    if not char then return nil, nil, nil end
    local hrp = char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Torso")
    local hum = char:FindFirstChildOfClass("Humanoid")
    return char, hrp, hum
end

local function SafeClick()
    local now = os.clock()
    if now - State.LastClick < 0.08 then return end
    State.LastClick = now
    pcall(function()
        VirtualUser:CaptureController()
        VirtualUser:ClickButton1(Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2))
    end)
end

--========================================================
-- FLY SİSTEMİ
--========================================================

local function StopFly()
    if State.BodyGyro then pcall(function() State.BodyGyro:Destroy() end) State.BodyGyro = nil end
    if State.BodyVelocity then pcall(function() State.BodyVelocity:Destroy() end) State.BodyVelocity = nil end
    local _, _, hum = GetChar()
    if hum then pcall(function() hum.PlatformStand = false end) end
end

local function StartFly()
    StopFly()
    local _, hrp, hum = GetChar()
    if not hrp then return end

    if hum then pcall(function() hum.PlatformStand = true end) end

    local bg = Instance.new("BodyGyro")
    bg.Name = "BlackhatFlyGyro"
    bg.P = 9e4
    bg.maxTorque = Vector3.new(9e9, 9e9, 9e9)
    bg.cframe = hrp.CFrame
    bg.Parent = hrp

    local bv = Instance.new("BodyVelocity")
    bv.Name = "BlackhatFlyVel"
    bv.velocity = Vector3.zero
    bv.maxForce = Vector3.new(9e9, 9e9, 9e9)
    bv.Parent = hrp

    State.BodyGyro = bg
    State.BodyVelocity = bv
end

--========================================================
-- SPEED HACK & NOCLIP
--========================================================

RunService.Stepped:Connect(function()
    local char, _, hum = GetChar()
    if not char then return end

    if Config.Noclip then
        for _, v in ipairs(char:GetDescendants()) do
            if v:IsA("BasePart") then
                v.CanCollide = false
            end
        end
    end

    if Config.SpeedHack and hum and hum.Health > 0 then
        if hum.WalkSpeed ~= Config.WalkSpeed then
            hum.WalkSpeed = Config.WalkSpeed
        end
    end
end)

RunService.Heartbeat:Connect(function()
    local char, hrp, hum = GetChar()
    if not char or not hrp then return end

    if Config.Fly then
        if not State.BodyGyro or not State.BodyVelocity or State.BodyGyro.Parent ~= hrp then
            StartFly()
        end
        if State.BodyGyro and State.BodyVelocity then
            local camCF = Camera.CFrame
            local moveDir = Vector3.zero
            if hum and hum.MoveDirection.Magnitude > 0 then
                moveDir = hum.MoveDirection
            end

            local flyDir = Vector3.zero
            if UserInputService:IsKeyDown(Enum.KeyCode.W) then flyDir += camCF.LookVector end
            if UserInputService:IsKeyDown(Enum.KeyCode.S) then flyDir -= camCF.LookVector end
            if UserInputService:IsKeyDown(Enum.KeyCode.D) then flyDir += camCF.RightVector end
            if UserInputService:IsKeyDown(Enum.KeyCode.A) then flyDir -= camCF.RightVector end
            if UserInputService:IsKeyDown(Enum.KeyCode.Space) then flyDir += Vector3.new(0, 1, 0) end
            if UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then flyDir -= Vector3.new(0, 1, 0) end

            if flyDir.Magnitude == 0 and moveDir.Magnitude > 0 then
                local flatLook = Vector3.new(camCF.LookVector.X, 0, camCF.LookVector.Z)
                if flatLook.Magnitude > 0 then
                    flatLook = flatLook.Unit
                    local rightVec = flatLook:Cross(Vector3.new(0, 1, 0))
                    local forwardAmt = moveDir:Dot(flatLook)
                    local rightAmt = moveDir:Dot(rightVec)
                    flyDir = (flatLook * forwardAmt) + (rightVec * rightAmt)
                end
            end

            if flyDir.Magnitude > 0 then
                flyDir = flyDir.Unit
            end

            State.BodyGyro.cframe = CFrame.new(hrp.Position, hrp.Position + camCF.LookVector)
            State.BodyVelocity.velocity = flyDir * Config.FlySpeed
        end
    elseif State.BodyGyro or State.BodyVelocity then
        StopFly()
    end
end)

--========================================================
-- PLAYER ESP
--========================================================

local function ClearPlayerESP()
    for plr, data in pairs(State.ESPObjects) do
        if data.Highlight then pcall(function() data.Highlight:Destroy() end) end
        if data.Billboard then pcall(function() data.Billboard:Destroy() end) end
        State.ESPObjects[plr] = nil
    end
end

local function UpdatePlayerESP()
    if not Config.PlayerESP then
        ClearPlayerESP()
        return
    end

    local _, myHRP = GetChar()

    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer and plr.Character then
            local char = plr.Character
            local hrp = char:FindFirstChild("HumanoidRootPart")
            local hum = char:FindFirstChildOfClass("Humanoid")

            if hrp and hum and hum.Health > 0 then
                local entry = State.ESPObjects[plr]
                if not entry then
                    entry = {}
                    State.ESPObjects[plr] = entry
                end

                if not entry.Highlight or entry.Highlight.Parent ~= char then
                    if entry.Highlight then pcall(function() entry.Highlight:Destroy() end) end
                    local hl = Instance.new("Highlight")
                    hl.Name = "BlackhatPlayerESP"
                    hl.FillColor = Color3.fromRGB(0, 220, 255)
                    hl.OutlineColor = Color3.fromRGB(255, 255, 255)
                    hl.FillTransparency = 0.45
                    hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
                    hl.Adornee = char
                    hl.Parent = char
                    entry.Highlight = hl
                end

                if not entry.Billboard or entry.Billboard.Parent ~= hrp then
                    if entry.Billboard then pcall(function() entry.Billboard:Destroy() end) end
                    local bb = Instance.new("BillboardGui")
                    bb.Name = "BlackhatPlayerTag"
                    bb.Adornee = hrp
                    bb.Size = UDim2.new(0, 170, 0, 38)
                    bb.StudsOffset = Vector3.new(0, 3.2, 0)
                    bb.AlwaysOnTop = true

                    local lbl = Instance.new("TextLabel")
                    lbl.Name = "TagLabel"
                    lbl.Size = UDim2.new(1, 0, 1, 0)
                    lbl.BackgroundTransparency = 1
                    lbl.TextColor3 = Color3.fromRGB(255, 255, 255)
                    lbl.TextStrokeTransparency = 0
                    lbl.TextSize = 11
                    lbl.Font = Enum.Font.GothamBold
                    lbl.RichText = true
                    lbl.Parent = bb
                    bb.Parent = hrp
                    entry.Billboard = bb
                end

                local lbl = entry.Billboard:FindFirstChild("TagLabel")
                if lbl then
                    local dist = myHRP and math.floor((hrp.Position - myHRP.Position).Magnitude) or 0
                    lbl.Text = string.format("%s <font color=\"#00FFAA\">[%dm]</font>\n<font color=\"#80E5FF\">HP: %d</font>", plr.DisplayName, dist, math.floor(hum.Health))
                end
            end
        end
    end

    for plr, data in pairs(State.ESPObjects) do
        if not plr.Parent or not plr.Character or not plr.Character:FindFirstChild("HumanoidRootPart") then
            if data.Highlight then pcall(function() data.Highlight:Destroy() end) end
            if data.Billboard then pcall(function() data.Billboard:Destroy() end) end
            State.ESPObjects[plr] = nil
        end
    end
end

--========================================================
-- PARLAYAN TAŞ / KRİSTAL / HEDEF VURMA SİSTEMİ
--========================================================

local CrystalKeywords = {
    "crystal", "stone", "gem", "shard", "core", "orb", "node",
    "rock", "relic", "frost", "ice", "heart", "ymir", "eye", "brain", "statue"
}

local function IsPotentialCrystalTarget(obj)
    if not obj or not obj.Parent then return false end
    local lower = string.lower(obj.Name)

    -- Oyuncu karakteri veya kendi aracımız değilse
    if Players:GetPlayerFromCharacter(obj) or Players:GetPlayerFromCharacter(obj.Parent) then
        return false
    end

    if obj:IsA("BasePart") then
        local matchesName = false
        for _, kw in ipairs(CrystalKeywords) do
            if string.find(lower, kw, 1, true) then
                matchesName = true
                break
            end
        end

        local isGlowingOrTranslucent = (obj.Material == Enum.Material.Neon or obj.Material == Enum.Material.Glass or obj.Material == Enum.Material.ForceField)
        local hasClickOrPrompt = obj:FindFirstChildOfClass("ClickDetector") or obj:FindFirstChildOfClass("ProximityPrompt")

        if matchesName or (isGlowingOrTranslucent and obj.Size.Magnitude >= 2.5) or hasClickOrPrompt then
            if obj.Size.Magnitude < 220 then
                return true
            end
        end
    elseif obj:IsA("Model") then
        for _, kw in ipairs(CrystalKeywords) do
            if string.find(lower, kw, 1, true) then
                return true
            end
        end
    end

    return false
end

local function GetTargetPart(obj)
    if obj:IsA("BasePart") then return obj end
    if obj:IsA("Model") then
        return obj.PrimaryPart or obj:FindFirstChildWhichIsA("BasePart", true)
    end
    return nil
end

local function RestoreExpandedParts()
    for part, orig in pairs(State.ExpandedParts) do
        if part and part.Parent then
            pcall(function()
                part.Size = orig.Size
                part.Transparency = orig.Transparency
            end)
        end
    end
    State.ExpandedParts = {}
end

local function ExpandPartHitbox(part)
    if not part or not part:IsA("BasePart") or State.ExpandedParts[part] then return end
    if part.Size.Magnitude > 50 then return end
    pcall(function()
        State.ExpandedParts[part] = {
            Size = part.Size,
            Transparency = part.Transparency
        }
        part.Size = Vector3.new(
            math.clamp(part.Size.X * 2.5, 8, 26),
            math.clamp(part.Size.Y * 2.5, 8, 26),
            math.clamp(part.Size.Z * 2.5, 8, 26)
        )
        part.Transparency = math.min(part.Transparency, 0.45)
    end)
end

local function FindCrystalTargets()
    local list = {}
    local seen = {}

    for _, desc in ipairs(Workspace:GetDescendants()) do
        if not seen[desc] and IsPotentialCrystalTarget(desc) then
            local part = GetTargetPart(desc)
            if part and not seen[part] then
                seen[desc] = true
                seen[part] = true
                table.insert(list, {Object = desc, Part = part})
            end
        end
    end

    return list
end

local function TriggerTargetInteractions(obj, part)
    -- ClickDetector varsa tetikle
    local cd = (obj and obj:FindFirstChildOfClass("ClickDetector")) or (part and part:FindFirstChildOfClass("ClickDetector"))
    if cd and fireclickdetector then
        pcall(function() fireclickdetector(cd) end)
    end

    -- ProximityPrompt varsa tetikle
    local pp = (obj and obj:FindFirstChildOfClass("ProximityPrompt")) or (part and part:FindFirstChildOfClass("ProximityPrompt"))
    if pp and fireproximityprompt then
        pcall(function() fireproximityprompt(pp) end)
    end

    -- Tool aktifse tetikle
    local char = LocalPlayer.Character
    if char then
        local tool = char:FindFirstChildOfClass("Tool")
        if tool then
            pcall(function() tool:Activate() end)
        end
    end

    if Config.CrystalAutoClick then
        SafeClick()
    end
end

local function StepCrystalAssist()
    if not Config.CrystalAssist then
        RestoreExpandedParts()
        return
    end

    local targets = FindCrystalTargets()
    if #targets == 0 then return end

    State.CrystalIndex += 1
    if State.CrystalIndex > #targets then
        State.CrystalIndex = 1
    end

    local current = targets[State.CrystalIndex]
    local part = current and current.Part
    if not part or not part.Parent then return end

    if Config.CrystalHitboxExpand then
        ExpandPartHitbox(part)
    end

    local char, hrp = GetChar()
    if char and hrp then
        if Config.CrystalAutoTP then
            local offsetPos = part.Position - (part.CFrame.LookVector * 6) + Vector3.new(0, 1.5, 0)
            pcall(function()
                char:PivotTo(CFrame.new(offsetPos, part.Position))
                hrp.AssemblyLinearVelocity = Vector3.zero
            end)
        end
        pcall(function()
            Camera.CFrame = CFrame.new(Camera.CFrame.Position, part.Position)
        end)
    end

    TriggerTargetInteractions(current.Object, part)
end

--========================================================
-- WORLD 5 / KARLI BÖLGE YARDIMCISI
--========================================================

local function StepWorld5Assist()
    if not Config.World5Assist then return end
    local char, hrp = GetChar()
    if not char or not hrp then return end

    -- World 5 veya karlı bölge / Lazarus / Frost hedeflerine yakınlık ve otomatik saldırı
    local aiFolder = Workspace:FindFirstChild("AI") or Workspace
    local closestNPC = nil
    local bestDist = math.huge

    for _, model in ipairs(aiFolder:GetChildren()) do
        if model:IsA("Model") and not Players:GetPlayerFromCharacter(model) then
            local hum = model:FindFirstChildOfClass("Humanoid")
            local mHRP = model:FindFirstChild("HumanoidRootPart") or model.PrimaryPart
            if hum and hum.Health > 0 and mHRP then
                local d = (mHRP.Position - hrp.Position).Magnitude
                if d < bestDist then
                    bestDist = d
                    closestNPC = {Model = model, HRP = mHRP, Hum = hum}
                end
            end
        end
    end

    if closestNPC and closestNPC.HRP then
        pcall(function()
            local targetCF = CFrame.new(closestNPC.HRP.Position - Vector3.new(0, 0, 5), closestNPC.HRP.Position)
            char:PivotTo(targetCF)
            hrp.AssemblyLinearVelocity = Vector3.zero
        end)
        local tool = char:FindFirstChildOfClass("Tool")
        if tool then pcall(function() tool:Activate() end) end
        SafeClick()
    end
end

--========================================================
-- ARKA PLAN DÖNGÜLERİ
--========================================================

task.spawn(function()
    while task.wait(0.25) do
        if Config.PlayerESP then
            pcall(UpdatePlayerESP)
        elseif next(State.ESPObjects) ~= nil then
            pcall(ClearPlayerESP)
        end
    end
end)

task.spawn(function()
    while task.wait(0.18) do
        if Config.CrystalAssist then
            pcall(StepCrystalAssist)
        end
        if Config.World5Assist then
            pcall(StepWorld5Assist)
        end
    end
end)

task.spawn(function()
    while task.wait(15) do
        if Config.AntiAFK then
            pcall(function()
                VirtualUser:CaptureController()
                VirtualUser:ClickButton2(Vector2.new())
            end)
        end
    end
end)

LocalPlayer.CharacterAdded:Connect(function()
    StopFly()
    State.ExpandedParts = {}
end)

--========================================================
-- KULLANICI ARAYÜZÜ (BLACKHAT HUB)
--========================================================

local old = PlayerGui:FindFirstChild("BLACKHAT_HUB")
if old then old:Destroy() end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "BLACKHAT_HUB"
ScreenGui.ResetOnSpawn = false
ScreenGui.IgnoreGuiInset = true
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.Parent = PlayerGui

local Window = Instance.new("Frame")
Window.Name = "Window"
Window.Size = UDim2.fromOffset(330, 420)
Window.Position = UDim2.new(0.5, -165, 0.5, -210)
Window.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
Window.BorderSizePixel = 0
Window.ClipsDescendants = true
Window.Parent = ScreenGui

local WindowCorner = Instance.new("UICorner")
WindowCorner.CornerRadius = UDim.new(0, 10)
WindowCorner.Parent = Window

local WindowStroke = Instance.new("UIStroke")
WindowStroke.Color = Color3.fromRGB(0, 230, 150)
WindowStroke.Thickness = 1.2
WindowStroke.Transparency = 0.5
WindowStroke.Parent = Window

local Header = Instance.new("Frame")
Header.Size = UDim2.new(1, 0, 0, 44)
Header.BackgroundColor3 = Color3.fromRGB(35, 35, 42)
Header.BorderSizePixel = 0
Header.Parent = Window

local HeaderCorner = Instance.new("UICorner")
HeaderCorner.CornerRadius = UDim.new(0, 10)
HeaderCorner.Parent = Header

local HeaderLogo = Instance.new("ImageLabel")
HeaderLogo.Size = UDim2.fromOffset(26, 26)
HeaderLogo.Position = UDim2.fromOffset(10, 9)
HeaderLogo.BackgroundTransparency = 1
HeaderLogo.Image = ICON_ID
HeaderLogo.Parent = Header

local Title = Instance.new("TextLabel")
Title.BackgroundTransparency = 1
Title.Size = UDim2.new(0, 165, 0, 20)
Title.Position = UDim2.fromOffset(42, 6)
Title.Text = "Blackhat Hub <font color=\"#00FFAA\">| THANOS SIM</font>"
Title.RichText = true
Title.TextColor3 = Color3.new(1, 1, 1)
Title.TextSize = 13
Title.Font = Enum.Font.GothamBold
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.ZIndex = 3
Title.Parent = Header

local SubTitle = Instance.new("TextLabel")
SubTitle.BackgroundTransparency = 1
SubTitle.Size = UDim2.new(0, 165, 0, 12)
SubTitle.Position = UDim2.fromOffset(42, 24)
SubTitle.Text = "by blackhat"
SubTitle.TextColor3 = Color3.fromRGB(160, 160, 175)
SubTitle.TextSize = 9
SubTitle.Font = Enum.Font.GothamMedium
SubTitle.TextXAlignment = Enum.TextXAlignment.Left
SubTitle.ZIndex = 3
SubTitle.Parent = Header

local ShowButton = Instance.new("ImageButton")
ShowButton.Name = "ShowButton"
ShowButton.Size = UDim2.fromOffset(46, 46)
ShowButton.Position = UDim2.new(0, 12, 0.5, -23)
ShowButton.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
ShowButton.BorderSizePixel = 0
ShowButton.Visible = false
ShowButton.ZIndex = 100
ShowButton.Parent = ScreenGui

local ShowCorner = Instance.new("UICorner")
ShowCorner.CornerRadius = UDim.new(0, 12)
ShowCorner.Parent = ShowButton

local ShowStroke = Instance.new("UIStroke")
ShowStroke.Color = Color3.fromRGB(0, 230, 150)
ShowStroke.Thickness = 1.5
ShowStroke.Transparency = 0.3
ShowStroke.Parent = ShowButton

local ShowIcon = Instance.new("ImageLabel")
ShowIcon.Size = UDim2.fromOffset(28, 28)
ShowIcon.Position = UDim2.new(0.5, -14, 0.5, -14)
ShowIcon.BackgroundTransparency = 1
ShowIcon.Image = ICON_ID
ShowIcon.ZIndex = 101
ShowIcon.Parent = ShowButton

do
    local draggingShow = false
    local dragStartShow, startPosShow
    ShowButton.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            draggingShow = true
            dragStartShow = input.Position
            startPosShow = ShowButton.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then draggingShow = false end
            end)
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if draggingShow and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local delta = input.Position - dragStartShow
            ShowButton.Position = UDim2.new(startPosShow.X.Scale, startPosShow.X.Offset + delta.X, startPosShow.Y.Scale, startPosShow.Y.Offset + delta.Y)
        end
    end)
end

local HideButton = Instance.new("TextButton")
HideButton.Name = "HideButton"
HideButton.Size = UDim2.fromOffset(34, 30)
HideButton.Position = UDim2.new(1, -40, 0, 7)
HideButton.BackgroundColor3 = Color3.fromRGB(50, 50, 58)
HideButton.BorderSizePixel = 0
HideButton.Text = "—"
HideButton.TextColor3 = Color3.fromRGB(235, 235, 235)
HideButton.TextSize = 12
HideButton.Font = Enum.Font.GothamBold
HideButton.ZIndex = 10
HideButton.Parent = Header
Instance.new("UICorner", HideButton).CornerRadius = UDim.new(0, 6)

HideButton.MouseButton1Click:Connect(function()
    Window.Visible = false
    ShowButton.Visible = true
end)

ShowButton.MouseButton1Click:Connect(function()
    Window.Visible = true
    ShowButton.Visible = false
end)

do
    local dragging = false
    local dragStart, startPos, dragInput
    Header.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = Window.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then dragging = false end
            end)
        end
    end)
    Header.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then dragInput = input end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and input == dragInput then
            local delta = input.Position - dragStart
            Window.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)
end

local Content = Instance.new("ScrollingFrame")
Content.Size = UDim2.new(1, -12, 1, -52)
Content.Position = UDim2.fromOffset(6, 48)
Content.BackgroundTransparency = 1
Content.BorderSizePixel = 0
Content.ScrollBarThickness = 3
Content.ScrollBarImageColor3 = Color3.fromRGB(0, 230, 150)
Content.ScrollingDirection = Enum.ScrollingDirection.Y
Content.AutomaticCanvasSize = Enum.AutomaticSize.Y
Content.CanvasSize = UDim2.fromOffset(0, 0)
Content.Parent = Window

local Padding = Instance.new("UIPadding")
Padding.PaddingLeft = UDim.new(0, 5)
Padding.PaddingRight = UDim.new(0, 5)
Padding.PaddingTop = UDim.new(0, 4)
Padding.PaddingBottom = UDim.new(0, 8)
Padding.Parent = Content

local Layout = Instance.new("UIListLayout")
Layout.Padding = UDim.new(0, 5)
Layout.SortOrder = Enum.SortOrder.LayoutOrder
Layout.Parent = Content

local function AddButton(text)
    local button = Instance.new("TextButton")
    button.Size = UDim2.new(1, 0, 0, 34)
    button.BackgroundColor3 = Color3.fromRGB(42, 44, 54)
    button.BorderSizePixel = 0
    button.TextColor3 = Color3.new(1, 1, 1)
    button.TextSize = 11
    button.Font = Enum.Font.GothamMedium
    button.Text = text
    button.ZIndex = 5
    button.Parent = Content
    Instance.new("UICorner", button).CornerRadius = UDim.new(0, 7)
    return button
end

local function AddStepper(labelText, getValue, stepDelta, minVal, maxVal, unitText, onChanged)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, 0, 0, 34)
    frame.BackgroundColor3 = Color3.fromRGB(35, 37, 47)
    frame.BorderSizePixel = 0
    frame.Parent = Content
    Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 6)

    local lbl = Instance.new("TextLabel")
    lbl.Size = UDim2.new(1, -100, 1, 0)
    lbl.Position = UDim2.fromOffset(10, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text = labelText .. ": " .. tostring(getValue()) .. (unitText or "")
    lbl.TextColor3 = Color3.fromRGB(220, 220, 220)
    lbl.TextSize = 10
    lbl.Font = Enum.Font.GothamBold
    lbl.TextXAlignment = Enum.TextXAlignment.Left
    lbl.Parent = frame

    local plusBtn = Instance.new("TextButton")
    plusBtn.Size = UDim2.fromOffset(36, 22)
    plusBtn.Position = UDim2.new(1, -86, 0.5, -11)
    plusBtn.BackgroundColor3 = Color3.fromRGB(50, 150, 100)
    plusBtn.Text = "+"
    plusBtn.TextColor3 = Color3.new(1, 1, 1)
    plusBtn.TextSize = 11
    plusBtn.Font = Enum.Font.GothamBold
    plusBtn.Parent = frame
    Instance.new("UICorner", plusBtn).CornerRadius = UDim.new(0, 5)

    local minusBtn = Instance.new("TextButton")
    minusBtn.Size = UDim2.fromOffset(36, 22)
    minusBtn.Position = UDim2.new(1, -44, 0.5, -11)
    minusBtn.BackgroundColor3 = Color3.fromRGB(150, 50, 50)
    minusBtn.Text = "-"
    minusBtn.TextColor3 = Color3.new(1, 1, 1)
    minusBtn.TextSize = 11
    minusBtn.Font = Enum.Font.GothamBold
    minusBtn.Parent = frame
    Instance.new("UICorner", minusBtn).CornerRadius = UDim.new(0, 5)

    local function refresh()
        lbl.Text = labelText .. ": " .. tostring(getValue()) .. (unitText or "")
        if onChanged then onChanged(getValue()) end
    end

    plusBtn.MouseButton1Click:Connect(function()
        local nextVal = math.clamp(getValue() + stepDelta, minVal, maxVal)
        onChanged(nextVal)
        lbl.Text = labelText .. ": " .. tostring(nextVal) .. (unitText or "")
    end)

    minusBtn.MouseButton1Click:Connect(function()
        local nextVal = math.clamp(getValue() - stepDelta, minVal, maxVal)
        onChanged(nextVal)
        lbl.Text = labelText .. ": " .. tostring(nextVal) .. (unitText or "")
    end)

    return frame, refresh
end

local function SetToggle(button, name, enabled)
    button.Text = name .. ": " .. (enabled and "ON" or "OFF")
    button.BackgroundColor3 = enabled and Color3.fromRGB(0, 160, 100) or Color3.fromRGB(42, 44, 54)
end

--========================================================
-- BUTONLAR VE BAĞLANTILAR
--========================================================

local FlyBtn = AddButton("Fly: OFF")
AddStepper("Fly Speed", function() return Config.FlySpeed end, 15, 15, 250, "", function(v)
    Config.FlySpeed = v
end)

local SpeedBtn = AddButton("Speed Hack: OFF")
AddStepper("WalkSpeed", function() return Config.WalkSpeed end, 8, 16, 200, "", function(v)
    Config.WalkSpeed = v
end)

local NoclipBtn = AddButton("Noclip: OFF")
local PlayerESPBtn = AddButton("Player ESP: OFF")

local CrystalAssistBtn = AddButton("Crystal / Stone Hit Assist: OFF")
local CrystalTPBtn = AddButton("Crystal Auto TP: OFF")
local CrystalHitboxBtn = AddButton("Crystal Hitbox Expand: ON")
local CrystalClickBtn = AddButton("Crystal Auto Click/Hit: ON")
local World5Btn = AddButton("Snow / World 5 Auto Assist: OFF")
local AntiAFKBtn = AddButton("Anti AFK: ON")

FlyBtn.MouseButton1Click:Connect(function()
    Config.Fly = not Config.Fly
    SetToggle(FlyBtn, "Fly", Config.Fly)
    if Config.Fly then StartFly() else StopFly() end
end)

SpeedBtn.MouseButton1Click:Connect(function()
    Config.SpeedHack = not Config.SpeedHack
    SetToggle(SpeedBtn, "Speed Hack", Config.SpeedHack)
    local _, _, hum = GetChar()
    if hum then
        hum.WalkSpeed = Config.SpeedHack and Config.WalkSpeed or 16
    end
end)

NoclipBtn.MouseButton1Click:Connect(function()
    Config.Noclip = not Config.Noclip
    SetToggle(NoclipBtn, "Noclip", Config.Noclip)
end)

PlayerESPBtn.MouseButton1Click:Connect(function()
    Config.PlayerESP = not Config.PlayerESP
    SetToggle(PlayerESPBtn, "Player ESP", Config.PlayerESP)
    if Config.PlayerESP then UpdatePlayerESP() else ClearPlayerESP() end
end)

CrystalAssistBtn.MouseButton1Click:Connect(function()
    Config.CrystalAssist = not Config.CrystalAssist
    SetToggle(CrystalAssistBtn, "Crystal / Stone Hit Assist", Config.CrystalAssist)
    if not Config.CrystalAssist then RestoreExpandedParts() end
end)

CrystalTPBtn.MouseButton1Click:Connect(function()
    Config.CrystalAutoTP = not Config.CrystalAutoTP
    SetToggle(CrystalTPBtn, "Crystal Auto TP", Config.CrystalAutoTP)
end)

CrystalHitboxBtn.MouseButton1Click:Connect(function()
    Config.CrystalHitboxExpand = not Config.CrystalHitboxExpand
    SetToggle(CrystalHitboxBtn, "Crystal Hitbox Expand", Config.CrystalHitboxExpand)
    if not Config.CrystalHitboxExpand then RestoreExpandedParts() end
end)

CrystalClickBtn.MouseButton1Click:Connect(function()
    Config.CrystalAutoClick = not Config.CrystalAutoClick
    SetToggle(CrystalClickBtn, "Crystal Auto Click/Hit", Config.CrystalAutoClick)
end)

World5Btn.MouseButton1Click:Connect(function()
    Config.World5Assist = not Config.World5Assist
    SetToggle(World5Btn, "Snow / World 5 Auto Assist", Config.World5Assist)
end)

AntiAFKBtn.MouseButton1Click:Connect(function()
    Config.AntiAFK = not Config.AntiAFK
    SetToggle(AntiAFKBtn, "Anti AFK", Config.AntiAFK)
end)

print("[Blackhat Hub] Thanos Simulator Script Loaded!")
