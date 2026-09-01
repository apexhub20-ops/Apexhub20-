if not game:IsLoaded() then game.Loaded:Wait() end

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local ts = game:GetService("TweenService")
local SoundService = game:GetService("SoundService")

getgenv().Config = getgenv().Config or {
    AimbotActive = false, CheckTeam = false, CheckWall = false, Radius = 150, FOVVisible = false,
    TargetPart = "Head", ESP = false, ESP_Box = false, ESP_Line = false, Health = false, Distance = false,
    AimSpeed = 50, AimInstant = false, AimNoShake = false, SairAimEnabled = false,
}

local espColor = Color3.new(1, 0, 0)
local fovColor = Color3.new(1, 0, 0)
local IsDraggingSlider = false
local stablePosition = nil
local LastTarget = nil

local SairAim = {
    Enabled = false, Attempts = 3, Count = 0, Processing = false,
    LastTarget = nil, LosingTarget = false, LossStart = 0, CooldownUntil = 0
}
local AimPermitido = true
local SairAimReleaseTime = 0.03
local SairAimLossConfirmTime = 0.02
local SairAimCooldown = 0.08

local ESP_Lines = {}
local ESP_Table = {}

-- ============================================================
-- TP STATE
-- ============================================================
local TP_SOUND_ID = "rbxassetid://5066021887"

local TPState = {
    Mode = "NONE",
    HasMark = false,
    IsSelecting = false,
    IsExecuting = false,
}

local markedPosition = nil
local tpMarker = nil
local TPConnections = {}

local previousTPPosition = nil
local hasPreviousTP = false
local isBackTP = false
local backTPBtn = nil

-- ============================================================
-- TP FORWARD DECLARATIONS
-- ============================================================
local ScreenGui
local tpStatusLabel
local tpCoordLabel
local tpOptionFrame
local CancelDedoSelection

-- ============================================================
-- TP CONNECTION MANAGEMENT
-- ============================================================

local function DisconnectTPConnections()
    for _, connection in pairs(TPConnections) do
        if connection then
            connection:Disconnect()
        end
    end
    table.clear(TPConnections)
end

-- ============================================================
-- TP CHARACTER HELPERS
-- ============================================================

local function GetCharacterRoot()
    local character = LocalPlayer.Character
    if not character then
        return nil, nil
    end

    local humanoid = character:FindFirstChildOfClass("Humanoid")
    local root = character:FindFirstChild("HumanoidRootPart")

    if not humanoid or not root then
        return nil, nil
    end

    if humanoid.Health <= 0 then
        return nil, nil
    end

    return character, root
end

-- ============================================================
-- TP UI STATE
-- ============================================================

local function UpdateTPUI()
    if not tpStatusLabel or not tpCoordLabel then
        return
    end

    if markedPosition and typeof(markedPosition) == "Vector3" then
        tpStatusLabel.Text = "✅ LOCAL MARCADO"
        tpStatusLabel.TextColor3 = Color3.fromRGB(80, 255, 120)

        tpCoordLabel.Text = string.format(
            "X: %.1f | Y: %.1f | Z: %.1f",
            markedPosition.X,
            markedPosition.Y,
            markedPosition.Z
        )
    else
        tpStatusLabel.Text = "❌ LOCAL NÃO MARCADO"
        tpStatusLabel.TextColor3 = Color3.fromRGB(255, 100, 100)
        tpCoordLabel.Text = "Nenhuma posição"
    end
end

-- ============================================================
-- TP MARKER
-- ============================================================

local function UpdateMarker()
    if tpMarker then
        tpMarker:Destroy()
        tpMarker = nil
    end

    if not markedPosition then
        return
    end

    tpMarker = Instance.new("Part")
    tpMarker.Name = "TP_Marker"
    tpMarker.Shape = Enum.PartType.Ball
    tpMarker.Size = Vector3.new(1, 1, 1)
    tpMarker.Position = markedPosition
    tpMarker.Anchored = true
    tpMarker.CanCollide = false
    tpMarker.CanTouch = false
    tpMarker.CanQuery = false
    tpMarker.Material = Enum.Material.Neon
    tpMarker.Color = Color3.fromRGB(255, 50, 50)
    tpMarker.Transparency = 0.2
    tpMarker.Parent = workspace

    local light = Instance.new("PointLight", tpMarker)
    light.Range = 6
    light.Color = Color3.fromRGB(255, 50, 50)
    light.Brightness = 1
end

-- ============================================================
-- TP SET/CLEAR MARK
-- ============================================================

local function SetMarkedPosition(position)
    if typeof(position) ~= "Vector3" then
        return false
    end

    if position ~= position then
        return false
    end

    if math.abs(position.X) > 100000 or math.abs(position.Y) > 100000 or math.abs(position.Z) > 100000 then
        return false
    end

    markedPosition = position
    TPState.HasMark = true
    TPState.Mode = "MARKED"

    UpdateMarker()
    UpdateTPUI()

    return true
end

local function ClearMarkedPosition()
    markedPosition = nil
    TPState.HasMark = false
    TPState.Mode = "NONE"

    if tpMarker then
        tpMarker:Destroy()
        tpMarker = nil
    end

    UpdateTPUI()
end

-- ============================================================
-- TP SAVE PREVIOUS POSITION
-- ============================================================

local function SavePreviousTPPosition(position)
    if typeof(position) ~= "Vector3" then
        return false
    end

    previousTPPosition = position
    hasPreviousTP = true

    if backTPBtn then
        backTPBtn.Visible = true
    end

    return true
end

-- ============================================================
-- TP RAYCAST
-- ============================================================

local function GetWorldPositionFromScreenPosition(screenPosition)
    if not Camera then return nil end

    local ray = Camera:ViewportPointToRay(screenPosition.X, screenPosition.Y)

    local raycastParams = RaycastParams.new()
    raycastParams.FilterType = Enum.RaycastFilterType.Blacklist
    raycastParams.FilterDescendantsInstances = {}

    local character = LocalPlayer.Character
    if character then
        table.insert(raycastParams.FilterDescendantsInstances, character)
    end

    if tpMarker then
        table.insert(raycastParams.FilterDescendantsInstances, tpMarker)
    end

    local result = workspace:Raycast(ray.Origin, ray.Direction * 1000, raycastParams)

    if result then
        return result.Position
    end

    return nil
end

-- ============================================================
-- TP INPUT HELPERS
-- ============================================================

local function GetInputScreenPosition(input)
    if input.UserInputType == Enum.UserInputType.Touch then
        return Vector2.new(input.Position.X, input.Position.Y)
    end

    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        local mousePosition = UserInputService:GetMouseLocation()
        return Vector2.new(mousePosition.X, mousePosition.Y)
    end

    return nil
end

local function IsInputOverOurUI(position)
    if not ScreenGui then return false end

    local objects = ScreenGui:GetGuiObjectsAtPosition(position.X, position.Y)

    for _, obj in ipairs(objects) do
        if obj:IsA("GuiObject") and obj.Visible then
            return true
        end
    end

    return false
end

-- ============================================================
-- TP MARKING
-- ============================================================

local function MarkCurrentPosition()
    local character, root = GetCharacterRoot()

    if not character or not root then
        ShowNotification("SEM PERSONAGEM", false)
        return false
    end

    return SetMarkedPosition(root.Position)
end

local function StartDedoSelection()
    if TPState.IsSelecting then return end

    TPState.IsSelecting = true
    TPState.Mode = "MARKING"

    DisconnectTPConnections()

    TPConnections.Input = UserInputService.InputBegan:Connect(function(input, gameProcessed)
        if not TPState.IsSelecting then return end

        if gameProcessed then return end

        if input.UserInputType ~= Enum.UserInputType.Touch and input.UserInputType ~= Enum.UserInputType.MouseButton1 then
            return
        end

        local screenPosition = GetInputScreenPosition(input)

        if not screenPosition then return end

        if IsInputOverOurUI(screenPosition) then
            return
        end

        local worldPosition = GetWorldPositionFromScreenPosition(screenPosition)

        if not worldPosition then
            ShowNotification("LOCAL INVÁLIDO", false)
            return
        end

        if SetMarkedPosition(worldPosition) then
            CancelDedoSelection()
            playPopSound()
            ShowNotification("MARCADO", true)
        end
    end)

    ShowNotification("TOQUE NA TELA", true)
end

CancelDedoSelection = function()
    TPState.IsSelecting = false

    if TPState.Mode == "MARKING" then
        TPState.Mode = "NONE"
    end

    DisconnectTPConnections()
    UpdateTPUI()
end

-- ============================================================
-- TP TELEPORT CENTRAL
-- ============================================================

local function CreateTeleportEffect(position)
    if typeof(position) ~= "Vector3" then return end

    local effect = Instance.new("Part")
    effect.Name = "TP_Effect"
    effect.Shape = Enum.PartType.Ball
    effect.Size = Vector3.new(0.5, 0.5, 0.5)
    effect.Position = position
    effect.Anchored = true
    effect.CanCollide = false
    effect.CanTouch = false
    effect.CanQuery = false
    effect.Material = Enum.Material.Neon
    effect.Color = Color3.fromRGB(0, 200, 255)
    effect.Transparency = 0
    effect.Parent = workspace

    local light = Instance.new("PointLight", effect)
    light.Range = 8
    light.Color = Color3.fromRGB(0, 200, 255)
    light.Brightness = 2

    ts:Create(effect, TweenInfo.new(0.4, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
        Size = Vector3.new(3, 3, 3),
        Transparency = 1
    }):Play()

    task.delay(0.5, function()
        if effect and effect.Parent then
            effect:Destroy()
        end
    end)
end

local function PlayTeleportSound()
    local sound = Instance.new("Sound")
    sound.SoundId = TP_SOUND_ID
    sound.Volume = 0.5
    sound.Parent = SoundService
    sound:Play()

    sound.Ended:Connect(function()
        sound:Destroy()
    end)

    task.delay(2, function()
        if sound and sound.Parent then
            sound:Destroy()
        end
    end)
end

local function TeleportToPosition(targetPosition)
    if typeof(targetPosition) ~= "Vector3" then
        return false
    end

    if targetPosition ~= targetPosition then
        return false
    end

    if math.abs(targetPosition.X) > 100000 or math.abs(targetPosition.Y) > 100000 or math.abs(targetPosition.Z) > 100000 then
        return false
    end

    if TPState.IsExecuting then
        return false
    end

    local character, root = GetCharacterRoot()

    if not character or not root then
        return false
    end

    local originPosition = root.Position

    TPState.IsExecuting = true

    local success, err = pcall(function()
        root.AssemblyLinearVelocity = Vector3.zero
        root.AssemblyAngularVelocity = Vector3.zero
        character:PivotTo(CFrame.new(targetPosition))
    end)

    TPState.IsExecuting = false

    if not success then
        return false
    end

    task.wait(0.05)

    local currentCharacter, currentRoot = GetCharacterRoot()
    if not currentCharacter or not currentRoot then
        return false
    end

    local distance = (currentRoot.Position - targetPosition).Magnitude

    if distance <= 10 then
        CreateTeleportEffect(targetPosition)
        PlayTeleportSound()
        return true, originPosition
    end

    return false
end

local function TeleportToMarkedPosition()
    if not markedPosition then
        ShowNotification("NENHUM LOCAL MARCADO", false)
        return false
    end

    local success, originPosition = TeleportToPosition(markedPosition)

    if success then
        SavePreviousTPPosition(originPosition)
        ShowNotification("TELEPORTADO", true)
        return true
    else
        ShowNotification("TP FALHOU", false)
        return false
    end
end

local function TeleportBack()
    if not hasPreviousTP or not previousTPPosition then
        ShowNotification("SEM LOCAL ANTERIOR", false)
        return false
    end

    if TPState.IsExecuting then
        return false
    end

    isBackTP = true

    local success = TeleportToPosition(previousTPPosition)

    isBackTP = false

    if success then
        ShowNotification("VOLTADO", true)
        return true
    else
        ShowNotification("VOLTA FALHOU", false)
        return false
    end
end

-- ============================================================
-- NOTIFICATION SYSTEM
-- ============================================================

local NotifFrame, NotifLabel, NotifIcon
local NotifToken = 0

local function CreateNotification()
    if NotifFrame then return end
    NotifFrame = Instance.new("Frame", ScreenGui)
    NotifFrame.Size = UDim2.new(0, 240, 0, 50)
    NotifFrame.AnchorPoint = Vector2.new(0.5, 0)
    NotifFrame.Position = UDim2.new(0.5, 0, 0, -60)
    NotifFrame.BackgroundColor3 = Color3.fromRGB(35, 35, 40)
    NotifFrame.BackgroundTransparency = 0.05
    NotifFrame.BorderSizePixel = 0
    NotifFrame.Visible = false
    NotifFrame.ZIndex = 1000
    CreateCorner(NotifFrame, 12)
    local stroke = Instance.new("UIStroke", NotifFrame)
    stroke.Color = Color3.fromRGB(100, 100, 110)
    stroke.Thickness = 1
    stroke.Transparency = 0.3
    NotifIcon = Instance.new("TextLabel", NotifFrame)
    NotifIcon.Size = UDim2.new(0, 30, 1, 0)
    NotifIcon.Position = UDim2.new(0, 10, 0, 0)
    NotifIcon.BackgroundTransparency = 1
    NotifIcon.Text = "✓"
    NotifIcon.TextColor3 = Color3.fromRGB(80, 255, 120)
    NotifIcon.TextSize = 20
    NotifIcon.Font = Enum.Font.GothamBlack
    NotifIcon.ZIndex = 1001
    NotifLabel = Instance.new("TextLabel", NotifFrame)
    NotifLabel.Size = UDim2.new(1, -50, 1, 0)
    NotifLabel.Position = UDim2.new(0, 45, 0, 0)
    NotifLabel.BackgroundTransparency = 1
    NotifLabel.Text = ""
    NotifLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    NotifLabel.TextSize = 14
    NotifLabel.Font = Enum.Font.GothamBold
    NotifLabel.TextXAlignment = Enum.TextXAlignment.Left
    NotifLabel.TextYAlignment = Enum.TextYAlignment.Center
    NotifLabel.ZIndex = 1001
end

local function ShowNotification(text, enabled)
    CreateNotification()
    NotifToken = NotifToken + 1
    local token = NotifToken
    if enabled == false then
        NotifIcon.Text = "×"
        NotifIcon.TextColor3 = Color3.fromRGB(255, 100, 100)
    else
        NotifIcon.Text = "✓"
        NotifIcon.TextColor3 = Color3.fromRGB(80, 255, 120)
    end
    NotifLabel.Text = text
    NotifFrame.Visible = true
    NotifFrame.Position = UDim2.new(0.5, 0, 0, -60)
    ts:Create(NotifFrame, TweenInfo.new(0.25, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), {Position = UDim2.new(0.5, 0, 0, 10)}):Play()
    task.delay(1.5, function()
        if token ~= NotifToken then return end
        local hide = ts:Create(NotifFrame, TweenInfo.new(0.25, Enum.EasingStyle.Quint, Enum.EasingDirection.In), {Position = UDim2.new(0.5, 0, 0, -60)})
        hide:Play()
        hide.Completed:Connect(function()
            if token == NotifToken then NotifFrame.Visible = false end
        end)
    end)
end

local function playPopSound()
    local sound = Instance.new("Sound")
    sound.SoundId = "rbxassetid://12222203"
    sound.Volume = 0.5
    sound.Parent = SoundService
    sound:Play()
    task.delay(0.3, function() sound:Destroy() end)
end

-- ============================================================
-- UI HELPERS
-- ============================================================

local function CreateCorner(obj, radius)
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, radius or 8)
    c.Parent = obj
    return c
end

local function CreateStroke(obj, color, thickness, transparency)
    local s = Instance.new("UIStroke")
    s.Color = color or Color3.fromRGB(60, 60, 70)
    s.Thickness = thickness or 1
    s.Transparency = transparency or 0
    s.Parent = obj
    return s
end

local function AnimateButton(btn, props, time)
    ts:Create(btn, TweenInfo.new(time or 0.15), props):Play()
end

local function CreateSectionLabel(parent, text)
    local container = Instance.new("Frame", parent)
    container.Size = UDim2.new(0.9, 0, 0, 14)
    container.BackgroundTransparency = 1
    local indicator = Instance.new("Frame", container)
    indicator.Size = UDim2.new(0, 2, 0, 10)
    indicator.Position = UDim2.new(0, 0, 0.5, -5)
    indicator.BackgroundColor3 = Color3.fromRGB(180, 30, 30)
    indicator.BorderSizePixel = 0
    CreateCorner(indicator, 1)
    local label = Instance.new("TextLabel", container)
    label.Size = UDim2.new(1, -8, 1, 0)
    label.Position = UDim2.new(0, 6, 0, 0)
    label.BackgroundTransparency = 1
    label.Text = text
    label.TextColor3 = Color3.fromRGB(160, 160, 170)
    label.TextSize = 8
    label.Font = Enum.Font.GothamBold
    label.TextXAlignment = Enum.TextXAlignment.Left
    return container
end

-- ============================================================
-- UI CREATION
-- ============================================================

for _, v in pairs(game.CoreGui:GetChildren()) do if v.Name == "PRIDE_HUB" then v:Destroy() end end
ScreenGui = Instance.new("ScreenGui", game.CoreGui); ScreenGui.Name = "PRIDE_HUB"; ScreenGui.IgnoreGuiInset = true

local FOV_Ring = Instance.new("Frame", ScreenGui)
FOV_Ring.BackgroundTransparency = 1
FOV_Ring.AnchorPoint = Vector2.new(0.5, 0.5)
FOV_Ring.Visible = false
FOV_Ring.ZIndex = 5
FOV_Ring.BorderSizePixel = 0
Instance.new("UICorner", FOV_Ring).CornerRadius = UDim.new(1, 0)
local fovStroke = Instance.new("UIStroke", FOV_Ring)
fovStroke.Color = fovColor
fovStroke.Thickness = 2
fovStroke.Transparency = 0
fovStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border

local Main = Instance.new("Frame", ScreenGui)
Main.Size = UDim2.new(0, 200, 0, 290)
Main.Position = UDim2.new(0.5, -100, 0.5, -145)
Main.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
Main.BackgroundTransparency = 0.02
Main.Active = true
Main.Draggable = true
Main.ClipsDescendants = true
CreateCorner(Main, 10)
CreateStroke(Main, Color3.fromRGB(50, 50, 55), 1, 0.4)

local gr = Instance.new("UIGradient", Main)
gr.Color = ColorSequence.new({ColorSequenceKeypoint.new(0, Color3.fromRGB(24, 24, 30)), ColorSequenceKeypoint.new(0.5, Color3.fromRGB(20, 20, 25)), ColorSequenceKeypoint.new(1, Color3.fromRGB(24, 24, 30))})
gr.Rotation = 45

local hd = Instance.new("Frame", Main)
hd.Size = UDim2.new(1, 0, 0, 36)
hd.BackgroundColor3 = Color3.fromRGB(28, 28, 34)
hd.BackgroundTransparency = 0.05
hd.BorderSizePixel = 0
CreateCorner(hd, 10)

local hdGrad = Instance.new("UIGradient", hd)
hdGrad.Color = ColorSequence.new({ColorSequenceKeypoint.new(0, Color3.fromRGB(30, 30, 36)), ColorSequenceKeypoint.new(0.5, Color3.fromRGB(38, 26, 28)), ColorSequenceKeypoint.new(1, Color3.fromRGB(30, 30, 36))})
hdGrad.Rotation = 90

local hdLine = Instance.new("Frame", hd)
hdLine.Size = UDim2.new(1, 0, 0, 1)
hdLine.Position = UDim2.new(0, 0, 1, -1)
hdLine.BackgroundColor3 = Color3.fromRGB(160, 25, 25)
hdLine.BorderSizePixel = 0

local tt = Instance.new("TextLabel", hd)
tt.Size = UDim2.new(1, -60, 0, 20)
tt.Position = UDim2.new(0, 12, 0, 3)
tt.BackgroundTransparency = 1
tt.Text = "PRIDE HUB"
tt.TextColor3 = Color3.new(1, 1, 1)
tt.TextSize = 12
tt.Font = Enum.Font.GothamBlack
tt.TextXAlignment = Enum.TextXAlignment.Left

local st = Instance.new("TextLabel", hd)
st.Size = UDim2.new(1, -60, 0, 10)
st.Position = UDim2.new(0, 12, 0, 22)
st.BackgroundTransparency = 1
st.Text = "AIM • ESP • TP"
st.TextColor3 = Color3.fromRGB(130, 130, 140)
st.TextSize = 7
st.Font = Enum.Font.GothamMedium
st.TextXAlignment = Enum.TextXAlignment.Left

local btnMin = Instance.new("TextButton", hd)
btnMin.Size = UDim2.new(0, 20, 0, 20)
btnMin.Position = UDim2.new(1, -44, 0.5, -10)
btnMin.BackgroundColor3 = Color3.fromRGB(40, 40, 46)
btnMin.BackgroundTransparency = 0.15
btnMin.Text = "−"
btnMin.TextColor3 = Color3.new(1, 1, 1)
btnMin.TextSize = 13
btnMin.Font = Enum.Font.GothamBold
btnMin.AutoButtonColor = false
CreateCorner(btnMin, 6)
CreateStroke(btnMin, Color3.fromRGB(55, 55, 62), 1, 0.5)

local CloseBtn = Instance.new("TextButton", hd)
CloseBtn.Size = UDim2.new(0, 20, 0, 20)
CloseBtn.Position = UDim2.new(1, -22, 0.5, -10)
CloseBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 46)
CloseBtn.BackgroundTransparency = 0.15
CloseBtn.Text = "×"
CloseBtn.TextColor3 = Color3.new(1, 1, 1)
CloseBtn.TextSize = 13
CloseBtn.Font = Enum.Font.GothamBold
CloseBtn.AutoButtonColor = false
CreateCorner(CloseBtn, 6)
CreateStroke(CloseBtn, Color3.fromRGB(55, 55, 62), 1, 0.5)

local Icon = Instance.new("ImageButton", ScreenGui)
Icon.Size = UDim2.new(0, 38, 0, 38)
Icon.Position = UDim2.new(0, 10, 0, 100)
Icon.Image = "rbxassetid://73630975144333"
Icon.ImageTransparency = 0.05
Icon.BackgroundTransparency = 1
Icon.Visible = false
Icon.ZIndex = 10
CreateCorner(Icon, 8)
CreateStroke(Icon, Color3.fromRGB(160, 25, 25), 2, 0.4)

local minimizado = false

btnMin.MouseButton1Click:Connect(function()
    minimizado = not minimizado
    CancelDedoSelection()
    if minimizado then
        Main.Visible = false
        Icon.Visible = true
        Icon.Size = UDim2.new(0, 19, 0, 19)
        ts:Create(Icon, TweenInfo.new(0.2, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Size = UDim2.new(0, 38, 0, 38)}):Play()
    else
        Main.Visible = true
        Icon.Visible = false
        Main.Size = UDim2.new(0, 180, 0, 270)
        ts:Create(Main, TweenInfo.new(0.2, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), {Size = UDim2.new(0, 200, 0, 290)}):Play()
    end
end)

Icon.MouseButton1Click:Connect(function()
    minimizado = false
    Main.Visible = true
    Icon.Visible = false
    Main.Size = UDim2.new(0, 180, 0, 270)
    ts:Create(Main, TweenInfo.new(0.2, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), {Size = UDim2.new(0, 200, 0, 290)}):Play()
end)

CloseBtn.MouseButton1Click:Connect(function()
    CancelDedoSelection()
    Main.Visible = false
    Icon.Visible = false
end)

local dragging = false
local dragStart = nil
local startPos = nil

local function MakeDraggable(obj)
    local dragging, dragStart, startPos
    obj.InputBegan:Connect(function(input)
        if (input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch) and not IsDraggingSlider then
            dragging = true
            dragStart = input.Position
            startPos = obj.Position
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging then
            local delta = input.Position - dragStart
            obj.Position = UDim2.new(0, math.clamp(startPos.X.Offset + delta.X, 0, ScreenGui.AbsoluteSize.X - obj.AbsoluteSize.X), 0, math.clamp(startPos.Y.Offset + delta.Y, 0, ScreenGui.AbsoluteSize.Y - obj.AbsoluteSize.Y))
        end
    end)
    UserInputService.InputEnded:Connect(function() dragging = false end)
end
MakeDraggable(Main)

Icon.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = Icon.Position
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        local delta = input.Position - dragStart
        Icon.Position = UDim2.new(0, math.clamp(startPos.X.Offset + delta.X, 0, ScreenGui.AbsoluteSize.X - Icon.AbsoluteSize.X), 0, math.clamp(startPos.Y.Offset + delta.Y, 0, ScreenGui.AbsoluteSize.Y - Icon.AbsoluteSize.Y))
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = false
    end
end)

local barraAbas = Instance.new("Frame", Main)
barraAbas.Size = UDim2.new(1, 0, 0, 26)
barraAbas.Position = UDim2.new(0, 0, 0, 36)
barraAbas.BackgroundColor3 = Color3.fromRGB(14, 14, 18)
barraAbas.BackgroundTransparency = 0.2
barraAbas.BorderSizePixel = 0

local abaAIM = Instance.new("TextButton", barraAbas)
abaAIM.Size = UDim2.new(0.32, 0, 0, 22)
abaAIM.Position = UDim2.new(0.005, 0, 0, 2)
abaAIM.BackgroundColor3 = Color3.fromRGB(140, 20, 20)
abaAIM.BackgroundTransparency = 0.1
abaAIM.Text = "AIM"
abaAIM.TextColor3 = Color3.new(1, 1, 1)
abaAIM.TextSize = 9
abaAIM.Font = Enum.Font.GothamBlack
abaAIM.AutoButtonColor = false
CreateCorner(abaAIM, 6)
CreateStroke(abaAIM, Color3.fromRGB(160, 25, 25), 1, 0.3)

local abaESP = Instance.new("TextButton", barraAbas)
abaESP.Size = UDim2.new(0.32, 0, 0, 22)
abaESP.Position = UDim2.new(0.335, 0, 0, 2)
abaESP.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
abaESP.BackgroundTransparency = 0.15
abaESP.Text = "ESP"
abaESP.TextColor3 = Color3.fromRGB(140, 140, 150)
abaESP.TextSize = 9
abaESP.Font = Enum.Font.GothamBlack
abaESP.AutoButtonColor = false
CreateCorner(abaESP, 6)
CreateStroke(abaESP, Color3.fromRGB(40, 40, 46), 1, 0.5)

local abaTP = Instance.new("TextButton", barraAbas)
abaTP.Size = UDim2.new(0.32, 0, 0, 22)
abaTP.Position = UDim2.new(0.665, 0, 0, 2)
abaTP.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
abaTP.BackgroundTransparency = 0.15
abaTP.Text = "TP"
abaTP.TextColor3 = Color3.fromRGB(140, 140, 150)
abaTP.TextSize = 9
abaTP.Font = Enum.Font.GothamBlack
abaTP.AutoButtonColor = false
CreateCorner(abaTP, 6)
CreateStroke(abaTP, Color3.fromRGB(40, 40, 46), 1, 0.5)

local abaIndicator = Instance.new("Frame", barraAbas)
abaIndicator.Size = UDim2.new(0, 18, 0, 2)
abaIndicator.Position = UDim2.new(0.005, 0, 1, -1)
abaIndicator.BackgroundColor3 = Color3.fromRGB(180, 30, 30)
abaIndicator.BorderSizePixel = 0
CreateCorner(abaIndicator, 1)

local contentY = 62
local contentH = 228

local telaAIM = Instance.new("ScrollingFrame", Main)
telaAIM.Size = UDim2.new(1, 0, 0, contentH)
telaAIM.Position = UDim2.new(0, 0, 0, contentY)
telaAIM.BackgroundTransparency = 1
telaAIM.ScrollBarThickness = 2
telaAIM.ScrollBarImageColor3 = Color3.fromRGB(140, 20, 20)
telaAIM.BorderSizePixel = 0
telaAIM.Visible = true
telaAIM.CanvasSize = UDim2.new(0, 0, 0, 0)
telaAIM.AutomaticCanvasSize = Enum.AutomaticSize.Y

local layAIM = Instance.new("UIListLayout", telaAIM)
layAIM.Padding = UDim.new(0, 4)
layAIM.HorizontalAlignment = Enum.HorizontalAlignment.Center
layAIM.SortOrder = Enum.SortOrder.LayoutOrder

local padAIM = Instance.new("UIPadding", telaAIM)
padAIM.PaddingTop = UDim.new(0, 6)
padAIM.PaddingBottom = UDim.new(0, 6)
padAIM.PaddingLeft = UDim.new(0, 5)
padAIM.PaddingRight = UDim.new(0, 5)

local telaESP = Instance.new("ScrollingFrame", Main)
telaESP.Size = UDim2.new(1, 0, 0, contentH)
telaESP.Position = UDim2.new(0, 0, 0, contentY)
telaESP.BackgroundTransparency = 1
telaESP.ScrollBarThickness = 2
telaESP.ScrollBarImageColor3 = Color3.fromRGB(140, 20, 20)
telaESP.BorderSizePixel = 0
telaESP.Visible = false
telaESP.CanvasSize = UDim2.new(0, 0, 0, 0)
telaESP.AutomaticCanvasSize = Enum.AutomaticSize.Y

local layESP = Instance.new("UIListLayout", telaESP)
layESP.Padding = UDim.new(0, 4)
layESP.HorizontalAlignment = Enum.HorizontalAlignment.Center
layESP.SortOrder = Enum.SortOrder.LayoutOrder

local padESP = Instance.new("UIPadding", telaESP)
padESP.PaddingTop = UDim.new(0, 6)
padESP.PaddingBottom = UDim.new(0, 6)
padESP.PaddingLeft = UDim.new(0, 5)
padESP.PaddingRight = UDim.new(0, 5)

local telaTP = Instance.new("ScrollingFrame", Main)
telaTP.Size = UDim2.new(1, 0, 0, contentH)
telaTP.Position = UDim2.new(0, 0, 0, contentY)
telaTP.BackgroundTransparency = 1
telaTP.ScrollBarThickness = 2
telaTP.ScrollBarImageColor3 = Color3.fromRGB(140, 20, 20)
telaTP.BorderSizePixel = 0
telaTP.Visible = false
telaTP.CanvasSize = UDim2.new(0, 0, 0, 0)
telaTP.AutomaticCanvasSize = Enum.AutomaticSize.Y

local layTP = Instance.new("UIListLayout", telaTP)
layTP.Padding = UDim.new(0, 8)
layTP.HorizontalAlignment = Enum.HorizontalAlignment.Center
layTP.SortOrder = Enum.SortOrder.LayoutOrder

local padTP = Instance.new("UIPadding", telaTP)
padTP.PaddingTop = UDim.new(0, 10)
padTP.PaddingBottom = UDim.new(0, 10)
padTP.PaddingLeft = UDim.new(0, 8)
padTP.PaddingRight = UDim.new(0, 8)

local selectedColorBtn = nil

local function criarSeletorCores(parent, callback)
    local frame = Instance.new("Frame", parent)
    frame.Size = UDim2.new(0.9, 0, 0, 42)
    frame.BackgroundTransparency = 1
    frame.LayoutOrder = 99
    local grid = Instance.new("UIGridLayout", frame)
    grid.CellSize = UDim2.new(0, 26, 0, 18)
    grid.CellPadding = UDim2.new(0, 3, 0, 3)
    grid.HorizontalAlignment = Enum.HorizontalAlignment.Center
    local cores = {Color3.new(1, 0, 0), Color3.new(1, 0.5, 0), Color3.new(1, 1, 0), Color3.new(0, 1, 0), Color3.new(0, 1, 1), Color3.new(0, 0, 1), Color3.new(0.5, 0, 1), Color3.new(1, 0, 1), Color3.new(1, 1, 1), Color3.new(0, 0, 0)}
    for _, c in ipairs(cores) do
        local bt = Instance.new("TextButton", frame)
        bt.Size = UDim2.new(0, 26, 0, 18)
        bt.BackgroundColor3 = c
        bt.Text = ""
        bt.AutoButtonColor = false
        CreateCorner(bt, 4)
        CreateStroke(bt, Color3.fromRGB(50, 50, 55), 1, 0.3)
        bt.MouseButton1Click:Connect(function()
            if selectedColorBtn then
                if selectedColorBtn:FindFirstChild("UIStroke2") then selectedColorBtn:FindFirstChild("UIStroke2"):Destroy() end
            end
            selectedColorBtn = bt
            local sel = Instance.new("UIStroke", bt)
            sel.Name = "UIStroke2"
            sel.Color = Color3.new(1, 1, 1)
            sel.Thickness = 2
            sel.Transparency = 0
            callback(c)
        end)
    end
    return frame
end

local function AddToggleVertical(name, prop, parent)
    local btn = Instance.new("TextButton", parent)
    btn.Size = UDim2.new(0.9, 0, 0, 32)
    btn.BackgroundColor3 = Color3.fromRGB(30, 30, 36)
    btn.BackgroundTransparency = 0.05
    btn.Text = ""
    btn.AutoButtonColor = false
    btn.BorderSizePixel = 0
    CreateCorner(btn, 7)
    CreateStroke(btn, Color3.fromRGB(50, 50, 56), 1, 0.5)
    local label = Instance.new("TextLabel", btn)
    label.Size = UDim2.new(0.55, 0, 1, 0)
    label.Position = UDim2.new(0, 9, 0, 0)
    label.BackgroundTransparency = 1
    label.Text = name
    label.TextColor3 = Color3.new(1, 1, 1)
    label.TextSize = 8
    label.Font = Enum.Font.GothamBold
    label.TextXAlignment = Enum.TextXAlignment.Left
    local toggleDot = Instance.new("Frame", btn)
    toggleDot.Size = UDim2.new(0, 30, 0, 16)
    toggleDot.Position = UDim2.new(1, -38, 0.5, -8)
    toggleDot.BackgroundColor3 = Color3.fromRGB(45, 45, 52)
    toggleDot.BorderSizePixel = 0
    CreateCorner(toggleDot, 8)
    local dot = Instance.new("Frame", toggleDot)
    dot.Size = UDim2.new(0, 12, 0, 12)
    dot.Position = UDim2.new(0, 2, 0.5, -6)
    dot.BackgroundColor3 = Color3.fromRGB(100, 100, 110)
    dot.BorderSizePixel = 0
    CreateCorner(dot, 6)
    local function updateToggle(showNotif)
        local on = getgenv().Config[prop]
        ts:Create(dot, TweenInfo.new(0.12), {Position = UDim2.new(0, on and 16 or 2, 0.5, -6), BackgroundColor3 = on and Color3.fromRGB(220, 50, 50) or Color3.fromRGB(100, 100, 110)}):Play()
        ts:Create(toggleDot, TweenInfo.new(0.12), {BackgroundColor3 = on and Color3.fromRGB(60, 20, 20) or Color3.fromRGB(45, 45, 52)}):Play()
        ts:Create(btn, TweenInfo.new(0.12), {BackgroundColor3 = on and Color3.fromRGB(40, 20, 20) or Color3.fromRGB(30, 30, 36)}):Play()
        if showNotif then ShowNotification(name .. (on and " ATIVO" or " DESATIVADO"), on) end
    end
    updateToggle(false)
    btn.MouseButton1Click:Connect(function()
        getgenv().Config[prop] = not getgenv().Config[prop]
        updateToggle(true)
        if prop == "SairAimEnabled" then
            if getgenv().Config[prop] then
                SairAim.Enabled = true
                SairAim.Count = 0
                SairAim.Processing = false
                AimPermitido = true
                if sairAimInput then sairAimInput.Visible = true end
                if sairAimContadorLabel then
                    sairAimContadorLabel.Text = "SAÍDAS: 0/" .. tostring(SairAim.Attempts)
                    sairAimContadorLabel.Visible = true
                end
            else
                SairAim.Enabled = false
                if sairAimInput then sairAimInput.Visible = false end
                if sairAimContadorLabel then sairAimContadorLabel.Visible = false end
                SairAim.Count = 0
                SairAim.Processing = false
                SairAim.LastTarget = nil
                SairAim.LosingTarget = false
                SairAim.LossStart = 0
                SairAim.CooldownUntil = 0
                AimPermitido = true
            end
        end
    end)
    return btn
end

local function AddSliderVertical(name, prop, parent, max, min, suffix)
    min = min or 0
    suffix = suffix or ""
    local frame = Instance.new("Frame", parent)
    frame.Size = UDim2.new(0.9, 0, 0, 35)
    frame.BackgroundColor3 = Color3.fromRGB(26, 26, 32)
    frame.BackgroundTransparency = 0.05
    frame.BorderSizePixel = 0
    CreateCorner(frame, 7)
    CreateStroke(frame, Color3.fromRGB(50, 50, 56), 1, 0.5)
    local label = Instance.new("TextLabel", frame)
    label.Size = UDim2.new(0.5, 0, 0, 15)
    label.Position = UDim2.new(0, 7, 0, 2)
    label.BackgroundTransparency = 1
    label.Text = name
    label.TextColor3 = Color3.new(1, 1, 1)
    label.TextSize = 7
    label.Font = Enum.Font.GothamBold
    label.TextXAlignment = Enum.TextXAlignment.Left
    local valueLabel = Instance.new("TextLabel", frame)
    valueLabel.Size = UDim2.new(0.4, 0, 0, 15)
    valueLabel.Position = UDim2.new(0.6, 0, 0, 2)
    valueLabel.BackgroundTransparency = 1
    valueLabel.Text = getgenv().Config[prop] .. suffix
    valueLabel.TextColor3 = Color3.fromRGB(220, 60, 60)
    valueLabel.TextSize = 8
    valueLabel.Font = Enum.Font.GothamBlack
    valueLabel.TextXAlignment = Enum.TextXAlignment.Right
    local barBg = Instance.new("Frame", frame)
    barBg.Size = UDim2.new(0.9, 0, 0, 6)
    barBg.Position = UDim2.new(0.05, 0, 0, 20)
    barBg.BackgroundColor3 = Color3.fromRGB(45, 45, 52)
    barBg.BorderSizePixel = 0
    CreateCorner(barBg, 3)
    local barFill = Instance.new("Frame", barBg)
    local percent = (getgenv().Config[prop] - min) / (max - min)
    barFill.Size = UDim2.new(percent, 0, 1, 0)
    barFill.BackgroundColor3 = Color3.fromRGB(160, 25, 25)
    barFill.BorderSizePixel = 0
    CreateCorner(barFill, 3)
    local dot = Instance.new("TextButton", barBg)
    dot.Size = UDim2.new(0, 14, 0, 14)
    dot.Position = UDim2.new(percent, -7, 0.5, -7)
    dot.BackgroundColor3 = Color3.new(1, 1, 1)
    dot.Text = ""
    dot.AutoButtonColor = false
    CreateCorner(dot, 7)
    CreateStroke(dot, Color3.fromRGB(160, 25, 25), 2, 0)
    local sliderConn = nil
    local function updateSlider(input)
        local p = math.clamp((input.Position.X - barBg.AbsolutePosition.X) / barBg.AbsoluteSize.X, 0, 1)
        local val = math.floor(min + p * (max - min))
        getgenv().Config[prop] = val
        valueLabel.Text = val .. suffix
        barFill.Size = UDim2.new(p, 0, 1, 0)
        dot.Position = UDim2.new(p, -7, 0.5, -7)
    end
    barBg.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            IsDraggingSlider = true
            sliderConn = UserInputService.InputChanged:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then updateSlider(input) end
            end)
            updateSlider(input)
        end
    end)
    dot.MouseButton1Down:Connect(function()
        IsDraggingSlider = true
        sliderConn = UserInputService.InputChanged:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then updateSlider(input) end
        end)
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            if sliderConn then sliderConn:Disconnect(); sliderConn = nil; ShowNotification(name .. ": " .. getgenv().Config[prop] .. suffix, true) end
            IsDraggingSlider = false
        end
    end)
    return frame
end

local sairAimInput = nil
local sairAimContadorLabel = nil

local function ExecutarSaidaAim()
    if not SairAim.Enabled then return end
    if SairAim.Processing then return end
    if SairAim.Count >= SairAim.Attempts then return end
    if os.clock() < SairAim.CooldownUntil then return end
    SairAim.Processing = true
    SairAim.Count = SairAim.Count + 1
    SairAim.CooldownUntil = os.clock() + SairAimCooldown
    if sairAimContadorLabel then sairAimContadorLabel.Text = "SAÍDAS: " .. tostring(SairAim.Count) .. "/" .. tostring(SairAim.Attempts) end
    AimPermitido = false
    task.delay(SairAimReleaseTime, function()
        AimPermitido = true
        SairAim.Processing = false
    end)
end

local function ProcessarSairAim(Target)
    if not SairAim.Enabled then return end
    if SairAim.Count >= SairAim.Attempts then return end
    if Target then
        SairAim.LastTarget = Target
        SairAim.LosingTarget = false
        SairAim.LossStart = 0
        return
    end
    if not SairAim.LastTarget then return end
    if not SairAim.LosingTarget then
        SairAim.LosingTarget = true
        SairAim.LossStart = os.clock()
        return
    end
    local tempoPerdido = os.clock() - SairAim.LossStart
    if tempoPerdido >= SairAimLossConfirmTime then
        ExecutarSaidaAim()
        SairAim.LosingTarget = false
        SairAim.LossStart = 0
    end
end

CreateSectionLabel(telaAIM, "AIMBOT")
AddToggleVertical("ATIVAR AIM", "AimbotActive", telaAIM)
AddToggleVertical("TIME", "CheckTeam", telaAIM)
AddToggleVertical("PAREDE", "CheckWall", telaAIM)
AddToggleVertical("MOSTRAR FOV", "FOVVisible", telaAIM)

CreateSectionLabel(telaAIM, "FOV")
AddSliderVertical("RAIO FOV", "Radius", telaAIM, 600)

CreateSectionLabel(telaAIM, "ALVO")
local PartBtn = Instance.new("TextButton", telaAIM)
PartBtn.Size = UDim2.new(0.9, 0, 0, 28)
PartBtn.BackgroundColor3 = Color3.fromRGB(30, 24, 24)
PartBtn.BackgroundTransparency = 0.05
PartBtn.Text = "ALVO: CABEÇA"
PartBtn.TextColor3 = Color3.new(1, 1, 1)
PartBtn.TextSize = 8
PartBtn.Font = Enum.Font.GothamBold
PartBtn.AutoButtonColor = false
CreateCorner(PartBtn, 7)
CreateStroke(PartBtn, Color3.fromRGB(160, 25, 25), 1, 0.4)
PartBtn.MouseButton1Click:Connect(function()
    getgenv().Config.TargetPart = (getgenv().Config.TargetPart == "Head" and "HumanoidRootPart" or "Head")
    PartBtn.Text = "ALVO: " .. (getgenv().Config.TargetPart == "Head" and "CABEÇA" or "TRONCO")
    ShowNotification("ALVO: " .. (getgenv().Config.TargetPart == "Head" and "CABEÇA" or "TRONCO"), true)
end)

CreateSectionLabel(telaAIM, "VELOCIDADE")
AddSliderVertical("VELOC AIMBOT", "AimSpeed", telaAIM, 100, 1, "%")

CreateSectionLabel(telaAIM, "MODOS")
AddToggleVertical("AIM FERA", "AimInstant", telaAIM)
AddToggleVertical("AIM STABLE", "AimNoShake", telaAIM)

CreateSectionLabel(telaAIM, "SAIR AIM")
AddToggleVertical("SAIR AIM", "SairAimEnabled", telaAIM)

sairAimInput = Instance.new("TextBox", telaAIM)
sairAimInput.Size = UDim2.new(0.9, 0, 0, 28)
sairAimInput.BackgroundColor3 = Color3.fromRGB(35, 35, 40)
sairAimInput.BackgroundTransparency = 0.05
sairAimInput.TextColor3 = Color3.fromRGB(220, 220, 220)
sairAimInput.PlaceholderColor3 = Color3.fromRGB(120, 120, 125)
sairAimInput.PlaceholderText = "QUANTAS TENTATIVAS"
sairAimInput.Text = "3"
sairAimInput.ClearTextOnFocus = false
sairAimInput.TextSize = 9
sairAimInput.Font = Enum.Font.GothamMedium
sairAimInput.TextXAlignment = Enum.TextXAlignment.Center
sairAimInput.Visible = false
sairAimInput.LayoutOrder = 99
CreateCorner(sairAimInput, 7)
CreateStroke(sairAimInput, Color3.fromRGB(160, 25, 25), 1, 0.4)

sairAimInput.FocusLost:Connect(function()
    local numero = tonumber(sairAimInput.Text)
    if numero then
        numero = math.floor(numero)
        numero = math.clamp(numero, 1, 100)
        SairAim.Attempts = numero
        sairAimInput.Text = tostring(numero)
    else
        sairAimInput.Text = tostring(SairAim.Attempts)
    end
    if sairAimContadorLabel then sairAimContadorLabel.Text = "SAÍDAS: " .. tostring(SairAim.Count) .. "/" .. tostring(SairAim.Attempts) end
end)

sairAimContadorLabel = Instance.new("TextLabel", telaAIM)
sairAimContadorLabel.Size = UDim2.new(0.9, 0, 0, 16)
sairAimContadorLabel.BackgroundTransparency = 1
sairAimContadorLabel.Text = "SAÍDAS: 0/3"
sairAimContadorLabel.TextColor3 = Color3.fromRGB(220, 60, 60)
sairAimContadorLabel.TextSize = 9
sairAimContadorLabel.Font = Enum.Font.GothamBold
sairAimContadorLabel.TextXAlignment = Enum.TextXAlignment.Center
sairAimContadorLabel.Visible = false
sairAimContadorLabel.LayoutOrder = 100

CreateSectionLabel(telaAIM, "COR FOV")
criarSeletorCores(telaAIM, function(cor)
    fovColor = cor
    fovStroke.Color = cor
end)

CreateSectionLabel(telaESP, "ESP")
AddToggleVertical("ESP NOME", "ESP", telaESP)
AddToggleVertical("ESP BOX", "ESP_Box", telaESP)
AddToggleVertical("ESP LINHA", "ESP_Line", telaESP)
AddToggleVertical("ESP DIST", "Distance", telaESP)
AddToggleVertical("ESP VIDA", "Health", telaESP)

CreateSectionLabel(telaESP, "COR ESP")
criarSeletorCores(telaESP, function(cor)
    espColor = cor
    for p, line in pairs(ESP_Lines) do
        if line then line.Color = cor end
    end
    for p, obj in pairs(ESP_Table) do
        if obj.Box and obj.Box.UIStroke then obj.Box.UIStroke.Color = cor end
        if obj.Label then obj.Label.TextColor3 = cor end
    end
end)

CreateSectionLabel(telaTP, "TELEPORTE")

tpStatusLabel = Instance.new("TextLabel", telaTP)
tpStatusLabel.Size = UDim2.new(0.9, 0, 0, 18)
tpStatusLabel.BackgroundTransparency = 1
tpStatusLabel.Text = "❌ LOCAL NÃO MARCADO"
tpStatusLabel.TextColor3 = Color3.fromRGB(255, 100, 100)
tpStatusLabel.TextSize = 10
tpStatusLabel.Font = Enum.Font.GothamBold
tpStatusLabel.TextXAlignment = Enum.TextXAlignment.Center
tpStatusLabel.LayoutOrder = 1

tpCoordLabel = Instance.new("TextLabel", telaTP)
tpCoordLabel.Size = UDim2.new(0.9, 0, 0, 18)
tpCoordLabel.BackgroundTransparency = 1
tpCoordLabel.Text = "Nenhuma posição"
tpCoordLabel.TextColor3 = Color3.fromRGB(220, 60, 60)
tpCoordLabel.TextSize = 9
tpCoordLabel.Font = Enum.Font.GothamBold
tpCoordLabel.TextXAlignment = Enum.TextXAlignment.Center
tpCoordLabel.LayoutOrder = 2

UpdateTPUI()

local marcarBtn = Instance.new("TextButton", telaTP)
marcarBtn.Size = UDim2.new(0.9, 0, 0, 34)
marcarBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 36)
marcarBtn.BackgroundTransparency = 0.05
marcarBtn.Text = "📍 MARCAR"
marcarBtn.TextColor3 = Color3.new(1, 1, 1)
marcarBtn.TextSize = 11
marcarBtn.Font = Enum.Font.GothamBold
marcarBtn.AutoButtonColor = false
marcarBtn.LayoutOrder = 3
CreateCorner(marcarBtn, 7)
CreateStroke(marcarBtn, Color3.fromRGB(160, 25, 25), 1.5, 0.3)

marcarBtn.MouseButton1Click:Connect(function()
    if tpOptionFrame then
        tpOptionFrame:Destroy()
        tpOptionFrame = nil
    end

    tpOptionFrame = Instance.new("Frame", telaTP)
    tpOptionFrame.Size = UDim2.new(0.9, 0, 0, 34)
    tpOptionFrame.BackgroundTransparency = 1
    tpOptionFrame.LayoutOrder = 4

    local dedoBtn = Instance.new("TextButton", tpOptionFrame)
    dedoBtn.Size = UDim2.new(0.48, 0, 0, 30)
    dedoBtn.Position = UDim2.new(0, 0, 0, 2)
    dedoBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 46)
    dedoBtn.BackgroundTransparency = 0.05
    dedoBtn.Text = "👆 DEDO"
    dedoBtn.TextColor3 = Color3.new(1, 1, 1)
    dedoBtn.TextSize = 10
    dedoBtn.Font = Enum.Font.GothamBold
    dedoBtn.AutoButtonColor = false
    CreateCorner(dedoBtn, 7)
    CreateStroke(dedoBtn, Color3.fromRGB(160, 25, 25), 1, 0.4)

    local agoraBtn = Instance.new("TextButton", tpOptionFrame)
    agoraBtn.Size = UDim2.new(0.48, 0, 0, 30)
    agoraBtn.Position = UDim2.new(0.52, 0, 0, 2)
    agoraBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 46)
    agoraBtn.BackgroundTransparency = 0.05
    agoraBtn.Text = "⚡ AGORA"
    agoraBtn.TextColor3 = Color3.new(1, 1, 1)
    agoraBtn.TextSize = 10
    agoraBtn.Font = Enum.Font.GothamBold
    agoraBtn.AutoButtonColor = false
    CreateCorner(agoraBtn, 7)
    CreateStroke(agoraBtn, Color3.fromRGB(160, 25, 25), 1, 0.4)

    dedoBtn.MouseButton1Click:Connect(function()
        tpOptionFrame:Destroy()
        tpOptionFrame = nil
        StartDedoSelection()
    end)

    agoraBtn.MouseButton1Click:Connect(function()
        tpOptionFrame:Destroy()
        tpOptionFrame = nil
        CancelDedoSelection()
        DisconnectTPConnections()
        if MarkCurrentPosition() then
            playPopSound()
            ShowNotification("MARCADO", true)
        end
    end)
end)

-- Frame para botões TP e VOLTAR
local tpButtonFrame = Instance.new("Frame", telaTP)
tpButtonFrame.Size = UDim2.new(0.9, 0, 0, 34)
tpButtonFrame.BackgroundTransparency = 1
tpButtonFrame.LayoutOrder = 5

local tpBtn = Instance.new("TextButton", tpButtonFrame)
tpBtn.Size = UDim2.new(0.78, 0, 0, 34)
tpBtn.Position = UDim2.new(0, 0, 0, 0)
tpBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 36)
tpBtn.BackgroundTransparency = 0.05
tpBtn.Text = "🌀 TELEPORTAR"
tpBtn.TextColor3 = Color3.new(1, 1, 1)
tpBtn.TextSize = 11
tpBtn.Font = Enum.Font.GothamBold
tpBtn.AutoButtonColor = false
CreateCorner(tpBtn, 7)
CreateStroke(tpBtn, Color3.fromRGB(160, 25, 25), 1.5, 0.3)

backTPBtn = Instance.new("TextButton", tpButtonFrame)
backTPBtn.Size = UDim2.new(0, 38, 0, 34)
backTPBtn.Position = UDim2.new(0.82, 4, 0, 0)
backTPBtn.BackgroundColor3 = Color3.fromRGB(30, 30, 36)
backTPBtn.BackgroundTransparency = 0.05
backTPBtn.Text = "⬅️"
backTPBtn.TextColor3 = Color3.new(1, 1, 1)
backTPBtn.TextSize = 14
backTPBtn.Font = Enum.Font.GothamBold
backTPBtn.AutoButtonColor = false
backTPBtn.Visible = false
CreateCorner(backTPBtn, 7)
CreateStroke(backTPBtn, Color3.fromRGB(160, 25, 25), 1.5, 0.3)

tpBtn.MouseButton1Click:Connect(function()
    TeleportToMarkedPosition()
end)

backTPBtn.MouseButton1Click:Connect(function()
    AnimateButton(backTPBtn, {Size = UDim2.new(0, 34, 0, 30)}, 0.08)
    TeleportBack()
    AnimateButton(backTPBtn, {Size = UDim2.new(0, 38, 0, 34)}, 0.08)
end)

local function selecionarAba(aba)
    telaAIM.Visible = (aba == "aim")
    telaESP.Visible = (aba == "esp")
    telaTP.Visible = (aba == "tp")

    if aba ~= "tp" then
        CancelDedoSelection()
        if tpOptionFrame then
            tpOptionFrame:Destroy()
            tpOptionFrame = nil
        end
    end

    if aba == "aim" then
        ts:Create(abaAIM, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(140, 20, 20), BackgroundTransparency = 0.1, TextColor3 = Color3.new(1, 1, 1)}):Play()
        ts:Create(abaESP, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(24, 24, 30), BackgroundTransparency = 0.15, TextColor3 = Color3.fromRGB(140, 140, 150)}):Play()
        ts:Create(abaTP, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(24, 24, 30), BackgroundTransparency = 0.15, TextColor3 = Color3.fromRGB(140, 140, 150)}):Play()
        ts:Create(abaIndicator, TweenInfo.new(0.15), {Position = UDim2.new(0.005, 0, 1, -1)}):Play()
    elseif aba == "esp" then
        ts:Create(abaESP, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(140, 20, 20), BackgroundTransparency = 0.1, TextColor3 = Color3.new(1, 1, 1)}):Play()
        ts:Create(abaAIM, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(24, 24, 30), BackgroundTransparency = 0.15, TextColor3 = Color3.fromRGB(140, 140, 150)}):Play()
        ts:Create(abaTP, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(24, 24, 30), BackgroundTransparency = 0.15, TextColor3 = Color3.fromRGB(140, 140, 150)}):Play()
        ts:Create(abaIndicator, TweenInfo.new(0.15), {Position = UDim2.new(0.335, 0, 1, -1)}):Play()
    else
        ts:Create(abaTP, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(140, 20, 20), BackgroundTransparency = 0.1, TextColor3 = Color3.new(1, 1, 1)}):Play()
        ts:Create(abaAIM, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(24, 24, 30), BackgroundTransparency = 0.15, TextColor3 = Color3.fromRGB(140, 140, 150)}):Play()
        ts:Create(abaESP, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(24, 24, 30), BackgroundTransparency = 0.15, TextColor3 = Color3.fromRGB(140, 140, 150)}):Play()
        ts:Create(abaIndicator, TweenInfo.new(0.15), {Position = UDim2.new(0.665, 0, 1, -1)}):Play()
    end
end

abaAIM.MouseButton1Click:Connect(function() selecionarAba("aim") end)
abaESP.MouseButton1Click:Connect(function() selecionarAba("esp") end)
abaTP.MouseButton1Click:Connect(function() selecionarAba("tp") end)
selecionarAba("aim")

local function MakeESP(p)
    if p == LocalPlayer then return end
    local l = Drawing.new("Line")
    l.Color = espColor
    l.Thickness = 1
    l.Visible = false
    ESP_Lines[p] = l
    local box = Instance.new("Frame", ScreenGui)
    box.BackgroundTransparency = 1
    local stroke = Instance.new("UIStroke", box)
    stroke.Color = espColor
    stroke.Thickness = 1
    local label = Instance.new("TextLabel", ScreenGui)
    label.BackgroundTransparency = 1
    label.TextColor3 = espColor
    label.TextSize = 7
    label.RichText = true
    local hb = Instance.new("Frame", ScreenGui)
    hb.BackgroundColor3 = Color3.new(0,0,0)
    hb.BorderSizePixel = 0
    local hm = Instance.new("Frame", hb)
    hm.BackgroundColor3 = Color3.new(0,1,0)
    hm.BorderSizePixel = 0
    ESP_Table[p] = {Box = box, Label = label, HBack = hb, HMain = hm}
end

for _, p in pairs(Players:GetPlayers()) do MakeESP(p) end
Players.PlayerAdded:Connect(MakeESP)
Players.PlayerRemoving:Connect(function(p)
    if ESP_Lines[p] then ESP_Lines[p]:Remove() end
    if ESP_Table[p] then
        ESP_Table[p].Box:Destroy()
        ESP_Table[p].Label:Destroy()
        ESP_Table[p].HBack:Destroy()
    end
end)

RunService.RenderStepped:Connect(function(dt)
    local cfg = getgenv().Config
    FOV_Ring.Visible = cfg.FOVVisible
    FOV_Ring.Size = UDim2.new(0, cfg.Radius * 2, 0, cfg.Radius * 2)
    FOV_Ring.Position = UDim2.new(0, Camera.ViewportSize.X / 2, 0, Camera.ViewportSize.Y / 2)

    local Target = nil
    local MaxDist = cfg.Radius

    for p, obj in pairs(ESP_Table) do
        local line = ESP_Lines[p]
        if p.Character and p.Character:FindFirstChild("HumanoidRootPart") and p.Character:FindFirstChild("Humanoid") and p.Character.Humanoid.Health > 0 then
            local root = p.Character.HumanoidRootPart
            local pos, vis = Camera:WorldToViewportPoint(root.Position)
            if vis and not (cfg.CheckTeam and p.Team == LocalPlayer.Team) then
                local s = 1000/pos.Z
                if line then
                    line.From = Vector2.new(Camera.ViewportSize.X/2, 0)
                    line.To = Vector2.new(pos.X, pos.Y)
                    line.Visible = cfg.ESP_Line
                    line.Color = espColor
                end
                obj.Box.Visible = cfg.ESP_Box
                obj.Box.Position = UDim2.new(0, pos.X-s/2, 0, pos.Y-s/1.5)
                obj.Box.Size = UDim2.new(0, s, 0, s*1.5)
                if obj.Box.UIStroke then obj.Box.UIStroke.Color = espColor end
                obj.Label.Visible = (cfg.ESP or cfg.Distance)
                obj.Label.Position = UDim2.new(0, pos.X, 0, pos.Y-s/1.5-12)
                obj.Label.Text = (cfg.ESP and p.Name or "") .. (cfg.Distance and " ["..math.floor((root.Position-Camera.CFrame.Position).Magnitude).."m]" or "")
                obj.Label.TextColor3 = espColor
                obj.HBack.Visible = cfg.Health
                obj.HBack.Position = UDim2.new(0, pos.X-s/2-4, 0, pos.Y-s/1.5)
                obj.HBack.Size = UDim2.new(0, 2, 0, s*1.5)
                local hp = p.Character.Humanoid.Health / p.Character.Humanoid.MaxHealth
                obj.HMain.Size = UDim2.new(1, 0, hp, 0)
                obj.HMain.Position = UDim2.new(0, 0, 1-hp, 0)

                if cfg.AimbotActive then
                    local dist = (Vector2.new(pos.X, pos.Y) - Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)).Magnitude
                    if dist < MaxDist then
                        local part = p.Character:FindFirstChild(cfg.TargetPart)
                        if part then
                            local wallCheck = true
                            if cfg.CheckWall then
                                local parts = Camera:GetPartsObscuringTarget({part.Position}, {LocalPlayer.Character, p.Character})
                                wallCheck = (#parts == 0)
                            end
                            if wallCheck then
                                Target = part
                                MaxDist = dist
                            end
                        end
                    end
                end
            else
                if line then line.Visible = false end
                obj.Box.Visible = false
                obj.Label.Visible = false
                obj.HBack.Visible = false
            end
        else
            if line then line.Visible = false end
            obj.Box.Visible = false
            obj.Label.Visible = false
            obj.HBack.Visible = false
        end
    end

    ProcessarSairAim(Target)

    if Target then
        if Target ~= LastTarget then
            stablePosition = Target.Position
            LastTarget = Target
        end
        local aimPosition = Target.Position
        if cfg.AimNoShake then
            local velocity = Target.AssemblyLinearVelocity or Vector3.zero
            local predicted = Target.Position + velocity * 0.02
            if stablePosition then
                local difference = predicted - stablePosition
                local distance = difference.Magnitude
                local DEADZONE = 0.05
                if distance > DEADZONE then
                    local alpha = 1 - math.exp(-12 * dt)
                    stablePosition = stablePosition:Lerp(predicted, math.clamp(alpha, 0, 1))
                end
            else
                stablePosition = predicted
            end
            aimPosition = stablePosition
        else
            stablePosition = Target.Position
            aimPosition = Target.Position
        end

        if cfg.AimbotActive and AimPermitido then
            local targetCF = CFrame.lookAt(Camera.CFrame.Position, aimPosition)
            if cfg.AimInstant then
                Camera.CFrame = targetCF
            else
                local aimSpeed = cfg.AimSpeed / 100
                Camera.CFrame = Camera.CFrame:Lerp(targetCF, math.clamp(aimSpeed * 0.5, 0, 1))
            end
        end
    else
        stablePosition = nil
        LastTarget = nil
    end
end)
