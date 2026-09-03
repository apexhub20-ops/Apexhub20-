if not game:IsLoaded() then game.Loaded:Wait() end

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local ts = game:GetService("TweenService")
local SoundService = game:GetService("SoundService")
local ProximityPromptService = game:GetService("ProximityPromptService")

-- ============================================================
-- CONFIG
-- ============================================================
getgenv().Config = getgenv().Config or {}

if getgenv().Config.AimbotActive == nil then getgenv().Config.AimbotActive = false end
if getgenv().Config.CheckTeam == nil then getgenv().Config.CheckTeam = false end
if getgenv().Config.CheckWall == nil then getgenv().Config.CheckWall = false end
if getgenv().Config.Radius == nil then getgenv().Config.Radius = 150 end
if getgenv().Config.FOVVisible == nil then getgenv().Config.FOVVisible = false end
if getgenv().Config.TargetPart == nil then getgenv().Config.TargetPart = "Head" end
if getgenv().Config.ESP == nil then getgenv().Config.ESP = false end
if getgenv().Config.ESP_Box == nil then getgenv().Config.ESP_Box = false end
if getgenv().Config.ESP_Line == nil then getgenv().Config.ESP_Line = false end
if getgenv().Config.Health == nil then getgenv().Config.Health = false end
if getgenv().Config.Distance == nil then getgenv().Config.Distance = false end
if getgenv().Config.AimSpeed == nil then getgenv().Config.AimSpeed = 50 end
if getgenv().Config.AimInstant == nil then getgenv().Config.AimInstant = false end
if getgenv().Config.AimNoShake == nil then getgenv().Config.AimNoShake = false end
if getgenv().Config.SairAimEnabled == nil then getgenv().Config.SairAimEnabled = false end
if getgenv().Config.ButtonTP == nil then getgenv().Config.ButtonTP = false end
if getgenv().Config.ButtonBack == nil then getgenv().Config.ButtonBack = false end
if getgenv().Config.TPInteracao == nil then getgenv().Config.TPInteracao = false end
if getgenv().Config.TPInteracaoDelay == nil then getgenv().Config.TPInteracaoDelay = 0.50 end
if getgenv().Config.ButtonFreecam == nil then getgenv().Config.ButtonFreecam = false end

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
local backTPBtn = nil

-- ============================================================
-- ATALHO STATE
-- ============================================================
local ShortcutState = {
    EditMode = false
}

local ShortcutPositions = {
    TP = UDim2.fromOffset(60, 200),
    BACK = UDim2.fromOffset(60, 260),
    FREECAM = UDim2.fromOffset(60, 320)
}

local editStartTPPosition = nil
local editStartBackPosition = nil
local editStartFreecamPosition = nil

local shortcutTPButton = nil
local shortcutBackButton = nil
local shortcutFreecamGroup = nil
local shortcutFreecamButton = nil
local freecamMinusButton = nil
local freecamSpeedLabel = nil
local freecamPlusButton = nil
local shortcutConfigButton = nil
local shortcutEditBar = nil

-- ============================================================
-- TP INTERAÇÃO STATE
-- ============================================================
local TPInteracaoState = {
    Enabled = false,
    Waiting = false,
    InteractionId = 0
}

local TPInteracaoConnections = {}
local tpInteracaoPending = false

-- ============================================================
-- FREECAM STATE
-- ============================================================
_G.JoystickData = _G.JoystickData or {
    DraggingLevel = 0,
    Direction = Vector3.new(0, 0, 0)
}

local FreecamEnabled = false
local movePart = nil
local currentPos = nil
local speedMultiplier = 1.0
local minSpeed = 0.1
local maxSpeed = 5.0
local speedStep = 0.2
local FreecamRenderConnection = nil
local OriginalCameraType = nil
local OriginalCameraSubject = nil
local OriginalMinZoom = nil
local OriginalMaxZoom = nil
local freecamJoystick = nil
local freecamJoystickOuter = nil
local freecamJoystickInner = nil
local joystickDragging = false
local joystickActiveTouch = nil
local joystickCenter = nil
local sizeOuter = 120
local sizeInner = 50
local joystickRadius = sizeOuter / 2

-- ============================================================
-- TP FORWARD DECLARATIONS
-- ============================================================
local ScreenGui
local tpStatusLabel
local tpCoordLabel
local tpOptionFrame
local CancelDedoSelection

-- ============================================================
-- FUNÇÃO PARA ESCONDER BOTÃO DE PULO MOBILE
-- ============================================================
local function SetJumpButtonVisible(visible)
    pcall(function()
        local playerGui = LocalPlayer:FindFirstChild("PlayerGui")
        if not playerGui then return end
        
        local function processGui(gui)
            for _, obj in ipairs(gui:GetDescendants()) do
                if obj:IsA("GuiObject") then
                    local name = string.lower(obj.Name)
                    if name:find("jump") or name:find("pulo") then
                        obj.Visible = visible
                    end
                end
            end
        end
        
        processGui(playerGui)
    end)
end

-- ============================================================
-- TP CONNECTION MANAGEMENT
-- ============================================================
local function DisconnectTPConnections()
    for _, connection in pairs(TPConnections) do
        if connection then connection:Disconnect() end
    end
    table.clear(TPConnections)
end

local function DisconnectTPInteracaoConnections()
    for _, connection in ipairs(TPInteracaoConnections) do
        if connection then connection:Disconnect() end
    end
    table.clear(TPInteracaoConnections)
end

-- ============================================================
-- TP CHARACTER HELPERS
-- ============================================================
local function GetCharacterRoot()
    local character = LocalPlayer.Character
    if not character then return nil, nil end
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    local root = character:FindFirstChild("HumanoidRootPart")
    if not humanoid or not root then return nil, nil end
    if humanoid.Health <= 0 then return nil, nil end
    return character, root
end

-- ============================================================
-- TP UI STATE
-- ============================================================
local function UpdateTPUI()
    if not tpStatusLabel or not tpCoordLabel then return end
    if markedPosition and typeof(markedPosition) == "Vector3" then
        tpStatusLabel.Text = "✅ LOCAL MARCADO"
        tpStatusLabel.TextColor3 = Color3.fromRGB(80, 255, 120)
        tpCoordLabel.Text = string.format("X: %.1f | Y: %.1f | Z: %.1f", markedPosition.X, markedPosition.Y, markedPosition.Z)
    else
        tpStatusLabel.Text = "❌ LOCAL NÃO MARCADO"
        tpStatusLabel.TextColor3 = Color3.fromRGB(255, 100, 100)
        tpCoordLabel.Text = "Nenhuma posição"
    end
end

-- ============================================================
-- TP MARKER (PULSANTE)
-- ============================================================
local function UpdateMarker()
    if tpMarker then tpMarker:Destroy(); tpMarker = nil end
    if not markedPosition then return end

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
    tpMarker.Color = Color3.fromRGB(160, 50, 200)
    tpMarker.Transparency = 0.2
    tpMarker.Parent = workspace

    local light = Instance.new("PointLight", tpMarker)
    light.Range = 6
    light.Color = Color3.fromRGB(160, 50, 200)
    light.Brightness = 2

    spawn(function()
        while tpMarker and tpMarker.Parent do
            ts:Create(light, TweenInfo.new(1, Enum.EasingStyle.Sine, Enum.EasingDirection.Out), {Brightness = 4, Range = 8}):Play()
            ts:Create(tpMarker, TweenInfo.new(1, Enum.EasingStyle.Sine, Enum.EasingDirection.Out), {Transparency = 0.05, Size = Vector3.new(1.5, 1.5, 1.5)}):Play()
            task.wait(1)
            ts:Create(light, TweenInfo.new(1, Enum.EasingStyle.Sine, Enum.EasingDirection.Out), {Brightness = 1, Range = 4}):Play()
            ts:Create(tpMarker, TweenInfo.new(1, Enum.EasingStyle.Sine, Enum.EasingDirection.Out), {Transparency = 0.4, Size = Vector3.new(0.8, 0.8, 0.8)}):Play()
            task.wait(1)
        end
    end)
end

-- ============================================================
-- TP SET MARK
-- ============================================================
local function SetMarkedPosition(position)
    if typeof(position) ~= "Vector3" then return false end
    if position ~= position then return false end
    if math.abs(position.X) > 100000 or math.abs(position.Y) > 100000 or math.abs(position.Z) > 100000 then return false end
    markedPosition = position
    TPState.HasMark = true
    TPState.Mode = "MARKED"
    UpdateMarker()
    UpdateTPUI()
    return true
end

-- ============================================================
-- TP SAVE PREVIOUS POSITION
-- ============================================================
local function SavePreviousTPPosition(position)
    if typeof(position) ~= "Vector3" then return false end
    previousTPPosition = position
    hasPreviousTP = true
    if backTPBtn then backTPBtn.Visible = true end
    return true
end

-- ============================================================
-- TP RAYCAST (USA CÂMERA ATUAL)
-- ============================================================
local function GetWorldPositionFromScreenPosition(screenPosition)
    local camera = workspace.CurrentCamera
    if not camera then return nil end
    local ray = camera:ViewportPointToRay(screenPosition.X, screenPosition.Y)
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    local ignoreList = {}
    if LocalPlayer.Character then table.insert(ignoreList, LocalPlayer.Character) end
    if tpMarker then table.insert(ignoreList, tpMarker) end
    params.FilterDescendantsInstances = ignoreList
    local result = workspace:Raycast(ray.Origin, ray.Direction * 10000, params)
    if result then return result.Position end
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
        if obj:IsA("GuiObject") and obj.Visible then return true end
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
        if input.UserInputType ~= Enum.UserInputType.Touch and input.UserInputType ~= Enum.UserInputType.MouseButton1 then return end
        local screenPosition = GetInputScreenPosition(input)
        if not screenPosition then return end
        if IsInputOverOurUI(screenPosition) then return end
        local worldPosition = GetWorldPositionFromScreenPosition(screenPosition)
        if not worldPosition then ShowNotification("LOCAL INVÁLIDO", false); return end
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
    if TPState.Mode == "MARKING" then TPState.Mode = "NONE" end
    DisconnectTPConnections()
    UpdateTPUI()
end

-- ============================================================
-- TP EFEITO PORTAL
-- ============================================================
local function CriarEfeitoPortal(posicao, cor)
    cor = cor or Color3.fromRGB(160, 50, 200)
    local cores = {
        Color3.fromRGB(160, 50, 200),
        Color3.fromRGB(200, 100, 220),
        Color3.fromRGB(120, 30, 180),
        Color3.fromRGB(255, 200, 255)
    }
    for i, corAnel in ipairs(cores) do
        local anel = Instance.new("Part")
        anel.Shape = Enum.PartType.Cylinder
        anel.Size = Vector3.new(i * 2, 0.3, i * 2)
        anel.Position = posicao + Vector3.new(0, i * 0.5, 0)
        anel.Anchored = true
        anel.CanCollide = false
        anel.CanTouch = false
        anel.Material = Enum.Material.Neon
        anel.Color = corAnel
        anel.Transparency = 0.3 + (i * 0.1)
        anel.Parent = workspace
        ts:Create(anel, TweenInfo.new(1 + i * 0.2, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Size = Vector3.new(i * 5, 0.3, i * 5), Transparency = 1}):Play()
        task.delay(1.5 + i * 0.2, function() if anel and anel.Parent then anel:Destroy() end end)
    end
    local luz = Instance.new("Part")
    luz.Shape = Enum.PartType.Ball
    luz.Size = Vector3.new(2, 2, 2)
    luz.Position = posicao
    luz.Anchored = true
    luz.CanCollide = false
    luz.CanTouch = false
    luz.Material = Enum.Material.Neon
    luz.Color = Color3.fromRGB(255, 200, 255)
    luz.Transparency = 0.2
    luz.Parent = workspace
    local luzPoint = Instance.new("PointLight", luz)
    luzPoint.Range = 15
    luzPoint.Color = cor
    luzPoint.Brightness = 3
    ts:Create(luz, TweenInfo.new(1, Enum.EasingStyle.Sine, Enum.EasingDirection.Out), {Size = Vector3.new(0.5, 0.5, 0.5), Transparency = 1}):Play()
    task.delay(1.5, function() if luz and luz.Parent then luz:Destroy() end end)
end

-- ============================================================
-- TP TELEPORT (CORRIGIDO)
-- ============================================================
local function TeleportarSeguro(posicaoDestino)
    if typeof(posicaoDestino) ~= "Vector3" then ShowNotification("POSIÇÃO INVÁLIDA", false); return false end
    if posicaoDestino ~= posicaoDestino then ShowNotification("POSIÇÃO INVÁLIDA", false); return false end
    if math.abs(posicaoDestino.X) > 100000 or math.abs(posicaoDestino.Y) > 100000 or math.abs(posicaoDestino.Z) > 100000 then ShowNotification("POSIÇÃO MUITO LONGE", false); return false end
    if TPState.IsExecuting then return false end

    local character = LocalPlayer.Character
    if not character then ShowNotification("SEM PERSONAGEM", false); return false end
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if not humanoid then ShowNotification("SEM HUMANOID", false); return false end
    if humanoid.Health <= 0 then ShowNotification("PERSONAGEM MORTO", false); return false end
    local root = character:FindFirstChild("HumanoidRootPart")
    if not root then ShowNotification("SEM ROOT PART", false); return false end

    local posicaoOriginal = root.Position
    TPState.IsExecuting = true
    CriarEfeitoPortal(posicaoDestino, Color3.fromRGB(160, 50, 200))

    local sucesso = false
    local tentativas = 0
    local maxTentativas = 5

    while not sucesso and tentativas < maxTentativas do
        tentativas = tentativas + 1
        local colisaoOriginal = root.CanCollide
        root.CanCollide = false
        local success, err = pcall(function() character:PivotTo(CFrame.new(posicaoDestino)) end)
        if success then
            task.wait(0.05)
            local novaPosicao = root.Position
            local distancia = (novaPosicao - posicaoDestino).Magnitude
            if distancia < 5 then sucesso = true; break end
        end
        if not sucesso then
            success, err = pcall(function() root.CFrame = CFrame.new(posicaoDestino) end)
            if success then
                task.wait(0.05)
                local novaPosicao = root.Position
                local distancia = (novaPosicao - posicaoDestino).Magnitude
                if distancia < 5 then sucesso = true; break end
            end
        end
        if not sucesso then
            task.wait(0.1)
        end
        root.CanCollide = colisaoOriginal
    end

    root.CanCollide = true

    if not sucesso then
        root.AssemblyLinearVelocity = Vector3.zero
        root.AssemblyAngularVelocity = Vector3.zero
        pcall(function() root.CFrame = CFrame.new(posicaoDestino) end)
        task.wait(0.1)
        local novaPosicao = root.Position
        local distancia = (novaPosicao - posicaoDestino).Magnitude
        if distancia < 10 then sucesso = true end
        if not sucesso then
            pcall(function() character:PivotTo(CFrame.new(posicaoDestino)) end)
            task.wait(0.1)
            local novaPosicao2 = root.Position
            local distancia2 = (novaPosicao2 - posicaoDestino).Magnitude
            if distancia2 < 10 then sucesso = true end
        end
    end

    TPState.IsExecuting = false

    if sucesso then
        task.wait(0.05)
        local posicaoFinal = root.Position
        local distanciaFinal = (posicaoFinal - posicaoDestino).Magnitude
        if distanciaFinal < 10 then
            CriarEfeitoPortal(posicaoDestino, Color3.fromRGB(200, 100, 220))
            ShowNotification("✅ TELEPORTADO!", true)
            if posicaoOriginal and (posicaoOriginal - posicaoDestino).Magnitude > 10 then
                SavePreviousTPPosition(posicaoOriginal)
            end
            return true
        else
            ShowNotification("⚠️ TELEPORTE PARCIAL", false)
            return true
        end
    else
        ShowNotification("❌ FALHA NO TELEPORTE", false)
        return false
    end
end

local function PlayTeleportSound()
    local sound = Instance.new("Sound")
    sound.SoundId = TP_SOUND_ID
    sound.Volume = 0.5
    sound.Parent = SoundService
    sound:Play()
    sound.Ended:Connect(function() sound:Destroy() end)
    task.delay(2, function() if sound and sound.Parent then sound:Destroy() end end)
end

local function TeleportToMarkedPosition()
    if not markedPosition then ShowNotification("NENHUM LOCAL MARCADO", false); return false end
    local sucesso = TeleportarSeguro(markedPosition)
    if sucesso then
        PlayTeleportSound()
        ShowNotification("✅ TELEPORTADO!", true)
        return true
    else
        ShowNotification("❌ FALHA NO TP", false)
        return false
    end
end

local function TeleportBack()
    if not hasPreviousTP or not previousTPPosition then ShowNotification("SEM LOCAL ANTERIOR", false); return false end
    if TPState.IsExecuting then return false end
    local sucesso = TeleportarSeguro(previousTPPosition)
    if sucesso then ShowNotification("VOLTADO", true); return true else ShowNotification("VOLTA FALHOU", false); return false end
end

-- ============================================================
-- TP INTERAÇÃO (COM DELAY)
-- ============================================================
local function ExecuteTPInteracao()
    if tpInteracaoPending then return end
    if not getgenv().Config.TPInteracao then return end
    if not markedPosition then ShowNotification("NENHUM LOCAL MARCADO", false); return end
    tpInteracaoPending = true
    local delayTime = tonumber(getgenv().Config.TPInteracaoDelay) or 0
    delayTime = math.max(0, delayTime)
    task.wait(delayTime)
    if getgenv().Config.TPInteracao and markedPosition then
        TeleportToMarkedPosition()
    end
    tpInteracaoPending = false
end

local function StartTPInteracao()
    DisconnectTPInteracaoConnections()
    if ProximityPromptService then
        local conn = ProximityPromptService.PromptTriggered:Connect(function(prompt, player)
            if player ~= LocalPlayer then return end
            if getgenv().Config.TPInteracao then
                task.spawn(function() ExecuteTPInteracao() end)
            end
        end)
        table.insert(TPInteracaoConnections, conn)
    end
end

local function StopTPInteracao()
    DisconnectTPInteracaoConnections()
    TPInteracaoState.Waiting = false
    TPInteracaoState.InteractionId = TPInteracaoState.InteractionId + 1
    tpInteracaoPending = false
end

-- ============================================================
-- FREECAM SYSTEM
-- ============================================================
local function toV2(pos) return Vector2.new(pos.X, pos.Y) end

local function updateJoystickCenter()
    if freecamJoystickOuter then
        joystickCenter = Vector2.new(freecamJoystickOuter.AbsolutePosition.X + joystickRadius, freecamJoystickOuter.AbsolutePosition.Y + joystickRadius)
    end
end

local function moveJoystickInner(posV2)
    local dir = posV2 - joystickCenter
    local dist = math.min(dir.Magnitude, joystickRadius)
    local offset = dir.Magnitude > 0 and dir.Unit * dist or Vector2.new(0, 0)
    if freecamJoystickInner then
        freecamJoystickInner.Position = UDim2.new(0.5, offset.X - sizeInner / 2, 0.5, offset.Y - sizeInner / 2)
    end
    local dragLevel = math.floor((dist / joystickRadius) * 100)
    local dir3 = Vector3.new(offset.X / joystickRadius, 0, offset.Y / joystickRadius)
    _G.JoystickData.DraggingLevel = dragLevel
    _G.JoystickData.Direction = dir3
end

local function animateJoystickPress()
    if freecamJoystickOuter then
        ts:Create(freecamJoystickOuter, TweenInfo.new(0.15, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Size = UDim2.fromOffset(sizeOuter * 1.1, sizeOuter * 1.1), BackgroundTransparency = 0.3}):Play()
    end
    if freecamJoystickInner then
        ts:Create(freecamJoystickInner, TweenInfo.new(0.15, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Size = UDim2.fromOffset(sizeInner * 1.2, sizeInner * 1.2), BackgroundColor3 = Color3.fromRGB(200, 200, 200)}):Play()
    end
end

local function animateJoystickRelease()
    if freecamJoystickOuter then
        ts:Create(freecamJoystickOuter, TweenInfo.new(0.2, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Size = UDim2.fromOffset(sizeOuter, sizeOuter), BackgroundTransparency = 0.5}):Play()
    end
    if freecamJoystickInner then
        ts:Create(freecamJoystickInner, TweenInfo.new(0.2, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Size = UDim2.fromOffset(sizeInner, sizeInner), BackgroundColor3 = Color3.fromRGB(255, 255, 255)}):Play()
    end
end

local function CreateFreecamJoystick()
    if freecamJoystick then return end
    freecamJoystick = Instance.new("Frame")
    freecamJoystick.Size = UDim2.new(1, 0, 1, 0)
    freecamJoystick.BackgroundTransparency = 1
    freecamJoystick.Visible = false
    freecamJoystick.ZIndex = 550
    freecamJoystick.Parent = ScreenGui

    freecamJoystickOuter = Instance.new("Frame")
    freecamJoystickOuter.Size = UDim2.fromOffset(sizeOuter, sizeOuter)
    freecamJoystickOuter.Position = UDim2.new(0.15, 0, 0.75, 0)
    freecamJoystickOuter.AnchorPoint = Vector2.new(0.5, 0.5)
    freecamJoystickOuter.BackgroundColor3 = Color3.fromRGB(35, 20, 50)
    freecamJoystickOuter.BackgroundTransparency = 0.3
    freecamJoystickOuter.Parent = freecamJoystick
    freecamJoystickOuter.ZIndex = 551
    freecamJoystickOuter.Active = true
    Instance.new("UICorner", freecamJoystickOuter).CornerRadius = UDim.new(1, 0)
    local outerStroke = Instance.new("UIStroke", freecamJoystickOuter)
    outerStroke.Color = Color3.fromRGB(180, 80, 220)
    outerStroke.Transparency = 0.4
    outerStroke.Thickness = 2

    freecamJoystickInner = Instance.new("Frame")
    freecamJoystickInner.Size = UDim2.fromOffset(sizeInner, sizeInner)
    freecamJoystickInner.Position = UDim2.new(0.5, -sizeInner / 2, 0.5, -sizeInner / 2)
    freecamJoystickInner.BackgroundColor3 = Color3.fromRGB(200, 120, 240)
    freecamJoystickInner.BackgroundTransparency = 0.2
    freecamJoystickInner.Parent = freecamJoystickOuter
    freecamJoystickInner.ZIndex = 552
    Instance.new("UICorner", freecamJoystickInner).CornerRadius = UDim.new(1, 0)

    freecamJoystickOuter.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
            if not joystickDragging then
                updateJoystickCenter()
                joystickDragging = true
                joystickActiveTouch = input
                moveJoystickInner(toV2(input.Position))
                animateJoystickPress()
            end
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if joystickDragging and joystickActiveTouch and input == joystickActiveTouch then
            moveJoystickInner(toV2(input.Position))
        end
    end)

    UserInputService.InputEnded:Connect(function(input)
        if joystickDragging and joystickActiveTouch and input == joystickActiveTouch then
            joystickDragging = false
            joystickActiveTouch = nil
            if freecamJoystickInner then freecamJoystickInner.Position = UDim2.new(0.5, -sizeInner / 2, 0.5, -sizeInner / 2) end
            _G.JoystickData.DraggingLevel = 0
            _G.JoystickData.Direction = Vector3.new(0, 0, 0)
            animateJoystickRelease()
        end
    end)
end

local function StopFreecamRender()
    if FreecamRenderConnection then FreecamRenderConnection:Disconnect(); FreecamRenderConnection = nil end
end

local function StartFreecamRender()
    StopFreecamRender()
    FreecamRenderConnection = RunService.RenderStepped:Connect(function(dt)
        if not FreecamEnabled then return end
        local dir = _G.JoystickData.Direction
        local level = _G.JoystickData.DraggingLevel
        if level > 1 then
            local moveSpeed = (level * 0.02) * speedMultiplier
            local camCF = Camera.CFrame
            local forward = camCF.LookVector
            local right = camCF.RightVector
            local moveDir = (forward * -dir.Z) + (right * dir.X)
            moveDir += Vector3.new(0, dir.Y, 0)
            if moveDir.Magnitude > 0 then
                moveDir = moveDir.Unit
                currentPos = currentPos + moveDir * moveSpeed * dt * 60
            end
        end
        local camLook = Camera.CFrame.LookVector
        local yaw = math.atan2(camLook.X, camLook.Z)
        movePart.CFrame = CFrame.new(currentPos) * CFrame.Angles(0, yaw, 0)
    end)
end

local function UpdateFreecamUI()
    if not shortcutFreecamButton then return end
    if FreecamEnabled then
        shortcutFreecamButton.BackgroundColor3 = Color3.fromRGB(60, 30, 90)
        shortcutFreecamButton.BackgroundTransparency = 0.1
    else
        shortcutFreecamButton.BackgroundColor3 = Color3.fromRGB(30, 15, 50)
        shortcutFreecamButton.BackgroundTransparency = 0.05
    end
    if freecamMinusButton then freecamMinusButton.Visible = FreecamEnabled end
    if freecamSpeedLabel then freecamSpeedLabel.Visible = FreecamEnabled end
    if freecamPlusButton then freecamPlusButton.Visible = FreecamEnabled end
    if freecamJoystick then freecamJoystick.Visible = FreecamEnabled end
    if freecamSpeedLabel then freecamSpeedLabel.Text = string.format("%.1fx", speedMultiplier) end
end

local function EnableFreecam()
    if FreecamEnabled then return end
    FreecamEnabled = true
    OriginalCameraType = Camera.CameraType
    OriginalCameraSubject = Camera.CameraSubject
    OriginalMinZoom = LocalPlayer.CameraMinZoomDistance
    OriginalMaxZoom = LocalPlayer.CameraMaxZoomDistance
    currentPos = Camera.CFrame.Position
    if not movePart then
        movePart = Instance.new("Part")
        movePart.Size = Vector3.new(0, 0, 0)
        movePart.Anchored = true
        movePart.Transparency = 1
        movePart.CanCollide = false
        movePart.Name = "CameraPart"
        movePart.Parent = workspace
    end
    movePart.CFrame = Camera.CFrame
    Camera.CameraSubject = movePart
    Camera.CameraType = Enum.CameraType.Custom
    LocalPlayer.CameraMinZoomDistance = 0
    LocalPlayer.CameraMaxZoomDistance = 0
    if not freecamJoystick then CreateFreecamJoystick() end
    if freecamJoystick then freecamJoystick.Visible = true end
    SetJumpButtonVisible(false)
    _G.JoystickData.DraggingLevel = 0
    _G.JoystickData.Direction = Vector3.new(0, 0, 0)
    StartFreecamRender()
    UpdateFreecamUI()
    ShowNotification("FREECAM ATIVO", true)
end

local function DisableFreecam()
    if not FreecamEnabled then return end
    FreecamEnabled = false
    StopFreecamRender()
    if freecamJoystick then freecamJoystick.Visible = false end
    local humanoid = LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
    Camera.CameraType = OriginalCameraType or Enum.CameraType.Custom
    Camera.CameraSubject = humanoid or OriginalCameraSubject
    if OriginalMinZoom then LocalPlayer.CameraMinZoomDistance = OriginalMinZoom end
    if OriginalMaxZoom then LocalPlayer.CameraMaxZoomDistance = OriginalMaxZoom end
    SetJumpButtonVisible(true)
    _G.JoystickData.DraggingLevel = 0
    _G.JoystickData.Direction = Vector3.new(0, 0, 0)
    UpdateFreecamUI()
    ShowNotification("FREECAM DESATIVADO", false)
end

-- ============================================================
-- ATALHO SYSTEM
-- ============================================================
local function UpdateShortcutVisibility()
    if shortcutTPButton then shortcutTPButton.Visible = ShortcutState.EditMode or getgenv().Config.ButtonTP == true end
    if shortcutBackButton then shortcutBackButton.Visible = ShortcutState.EditMode or getgenv().Config.ButtonBack == true end
    if shortcutFreecamGroup then shortcutFreecamGroup.Visible = ShortcutState.EditMode or getgenv().Config.ButtonFreecam == true end
end

local function EnterShortcutEditMode()
    if ShortcutState.EditMode then return end
    if not shortcutTPButton or not shortcutBackButton or not shortcutFreecamGroup then return end
    editStartTPPosition = shortcutTPButton.Position
    editStartBackPosition = shortcutBackButton.Position
    editStartFreecamPosition = shortcutFreecamGroup.Position
    ShortcutState.EditMode = true
    if shortcutEditBar then shortcutEditBar.Visible = true end
    UpdateShortcutVisibility()
end

local function CancelShortcutEdit()
    if shortcutTPButton and editStartTPPosition then shortcutTPButton.Position = editStartTPPosition end
    if shortcutBackButton and editStartBackPosition then shortcutBackButton.Position = editStartBackPosition end
    if shortcutFreecamGroup and editStartFreecamPosition then shortcutFreecamGroup.Position = editStartFreecamPosition end
    ShortcutState.EditMode = false
    if shortcutEditBar then shortcutEditBar.Visible = false end
    UpdateShortcutVisibility()
    ShowNotification("ALTERAÇÕES CANCELADAS", false)
end

local function SaveShortcutEdit()
    if not shortcutTPButton or not shortcutBackButton or not shortcutFreecamGroup then return end
    ShortcutPositions.TP = shortcutTPButton.Position
    ShortcutPositions.BACK = shortcutBackButton.Position
    ShortcutPositions.FREECAM = shortcutFreecamGroup.Position
    ShortcutState.EditMode = false
    if shortcutEditBar then shortcutEditBar.Visible = false end
    UpdateShortcutVisibility()
    ShowNotification("ATALHOS SALVOS", true)
end

local function MakeShortcutDraggable(button)
    local dragging = false
    local dragStart = nil
    local startPos = nil
    button.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.Touch or input.UserInputType == Enum.UserInputType.MouseButton1 then
            if ShortcutState.EditMode then
                dragging = true
                dragStart = input.Position
                startPos = button.Position
            end
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if not dragging then return end
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            local viewport = Camera.ViewportSize
            local delta = input.Position - dragStart
            local maxX = math.max(0, viewport.X - button.AbsoluteSize.X)
            local maxY = math.max(0, viewport.Y - button.AbsoluteSize.Y)
            local newX = math.clamp(startPos.X.Offset + delta.X, 0, maxX)
            local newY = math.clamp(startPos.Y.Offset + delta.Y, 0, maxY)
            button.Position = UDim2.fromOffset(newX, newY)
        end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dragging = false end
    end)
end

-- ============================================================
-- NOTIFICATION SYSTEM (CORES VIVAS)
-- ============================================================
local NotifFrame, NotifLabel, NotifIcon
local NotifToken = 0

local function CreateNotification()
    if NotifFrame then return end
    NotifFrame = Instance.new("Frame")
    NotifFrame.Size = UDim2.new(0, 220, 0, 45)
    NotifFrame.AnchorPoint = Vector2.new(0.5, 0)
    NotifFrame.Position = UDim2.new(0.5, 0, 0, -60)
    NotifFrame.BackgroundColor3 = Color3.fromRGB(40, 20, 60)
    NotifFrame.BackgroundTransparency = 0
    NotifFrame.BorderSizePixel = 0
    NotifFrame.Visible = false
    NotifFrame.ZIndex = 1000
    NotifFrame.Parent = ScreenGui
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 10)
    corner.Parent = NotifFrame
    
    local stroke = Instance.new("UIStroke")
    stroke.Color = Color3.fromRGB(180, 80, 220)
    stroke.Thickness = 1.5
    stroke.Transparency = 0.2
    stroke.Parent = NotifFrame

    local grad = Instance.new("UIGradient", NotifFrame)
    grad.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(60, 25, 90)),
        ColorSequenceKeypoint.new(0.5, Color3.fromRGB(90, 35, 120)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(60, 25, 90))
    })
    grad.Rotation = 90

    NotifIcon = Instance.new("TextLabel")
    NotifIcon.Size = UDim2.new(0, 30, 1, 0)
    NotifIcon.Position = UDim2.new(0, 8, 0, 0)
    NotifIcon.BackgroundTransparency = 1
    NotifIcon.Text = ""
    NotifIcon.TextColor3 = Color3.fromRGB(200, 120, 240)
    NotifIcon.TextSize = 16
    NotifIcon.Font = Enum.Font.GothamBold
    NotifIcon.ZIndex = 1001
    NotifIcon.Parent = NotifFrame
    
    NotifLabel = Instance.new("TextLabel")
    NotifLabel.Size = UDim2.new(1, -45, 1, 0)
    NotifLabel.Position = UDim2.new(0, 40, 0, 0)
    NotifLabel.BackgroundTransparency = 1
    NotifLabel.Text = ""
    NotifLabel.TextColor3 = Color3.fromRGB(240, 235, 245)
    NotifLabel.TextSize = 12
    NotifLabel.Font = Enum.Font.GothamMedium
    NotifLabel.TextXAlignment = Enum.TextXAlignment.Left
    NotifLabel.TextYAlignment = Enum.TextYAlignment.Center
    NotifLabel.ZIndex = 1001
    NotifLabel.Parent = NotifFrame
    NotifLabel.TextWrapped = true
end

local function ShowNotification(text, enabled)
    CreateNotification()
    NotifToken = NotifToken + 1
    local token = NotifToken
    
    if enabled == false then
        NotifIcon.Text = "✕"
        NotifIcon.TextColor3 = Color3.fromRGB(255, 120, 120)
    else
        NotifIcon.Text = "✓"
        NotifIcon.TextColor3 = Color3.fromRGB(120, 255, 160)
    end
    
    NotifLabel.Text = text
    
    NotifFrame.Visible = true
    NotifFrame.Position = UDim2.new(0.5, 0, 0, -60)
    NotifFrame.BackgroundTransparency = 0.8
    
    ts:Create(NotifFrame, TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
        Position = UDim2.new(0.5, 0, 0, 10),
        BackgroundTransparency = 0
    }):Play()
    
    task.delay(1.8, function()
        if token ~= NotifToken then return end
        
        local hide = ts:Create(NotifFrame, TweenInfo.new(0.25, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
            Position = UDim2.new(0.5, 0, 0, -60),
            BackgroundTransparency = 0.8
        })
        hide:Play()
        
        hide.Completed:Connect(function()
            if token == NotifToken then 
                NotifFrame.Visible = false 
            end
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
    container.BackgroundColor3 = Color3.fromRGB(45, 25, 70)
    container.BackgroundTransparency = 0
    CreateCorner(container, 4)
    
    local grad = Instance.new("UIGradient", container)
    grad.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(60, 30, 90)),
        ColorSequenceKeypoint.new(0.5, Color3.fromRGB(90, 40, 120)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(60, 30, 90))
    })
    grad.Rotation = 90
    
    local stroke = Instance.new("UIStroke", container)
    stroke.Color = Color3.fromRGB(150, 70, 200)
    stroke.Thickness = 1
    stroke.Transparency = 0.3
    
    local indicator = Instance.new("Frame", container)
    indicator.Size = UDim2.new(0, 2, 0, 10)
    indicator.Position = UDim2.new(0, 4, 0.5, -5)
    indicator.BackgroundColor3 = Color3.fromRGB(200, 100, 240)
    indicator.BorderSizePixel = 0
    CreateCorner(indicator, 1)
    
    local label = Instance.new("TextLabel", container)
    label.Size = UDim2.new(1, -10, 1, 0)
    label.Position = UDim2.new(0, 8, 0, 0)
    label.BackgroundTransparency = 1
    label.Text = text
    label.TextColor3 = Color3.fromRGB(220, 180, 240)
    label.TextSize = 8
    label.Font = Enum.Font.GothamBold
    label.TextXAlignment = Enum.TextXAlignment.Left
    
    return container
end

-- ============================================================
-- UI CREATION (ROXO VIBRANTE)
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
Main.BackgroundColor3 = Color3.fromRGB(25, 12, 40)
Main.BackgroundTransparency = 0
Main.Active = true
Main.Draggable = true
Main.ClipsDescendants = true
CreateCorner(Main, 10)

local mainGrad = Instance.new("UIGradient", Main)
mainGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(30, 10, 50)),
    ColorSequenceKeypoint.new(0.25, Color3.fromRGB(40, 15, 65)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(50, 20, 75)),
    ColorSequenceKeypoint.new(0.75, Color3.fromRGB(40, 15, 65)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(30, 10, 50))
})
mainGrad.Rotation = 45

local mainStroke = Instance.new("UIStroke", Main)
mainStroke.Color = Color3.fromRGB(180, 80, 220)
mainStroke.Thickness = 2
mainStroke.Transparency = 0.2

local hd = Instance.new("Frame", Main)
hd.Size = UDim2.new(1, 0, 0, 36)
hd.BackgroundColor3 = Color3.fromRGB(50, 20, 75)
hd.BackgroundTransparency = 0
hd.BorderSizePixel = 0
CreateCorner(hd, 10)

local hdGrad = Instance.new("UIGradient", hd)
hdGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(70, 30, 100)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(120, 40, 140)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(70, 30, 100))
})
hdGrad.Rotation = 90

local hdLine = Instance.new("Frame", hd)
hdLine.Size = UDim2.new(1, 0, 0, 1)
hdLine.Position = UDim2.new(0, 0, 1, -1)
hdLine.BackgroundColor3 = Color3.fromRGB(200, 80, 240)
hdLine.BorderSizePixel = 0

local tt = Instance.new("TextLabel", hd)
tt.Size = UDim2.new(1, -60, 0, 20)
tt.Position = UDim2.new(0, 12, 0, 3)
tt.BackgroundTransparency = 1
tt.Text = "PRIDE HUB"
tt.TextColor3 = Color3.fromRGB(255, 255, 255)
tt.TextSize = 12
tt.Font = Enum.Font.GothamBold
tt.TextXAlignment = Enum.TextXAlignment.Left

local st = Instance.new("TextLabel", hd)
st.Size = UDim2.new(1, -60, 0, 10)
st.Position = UDim2.new(0, 12, 0, 22)
st.BackgroundTransparency = 1
st.Text = "AIM • ESP • TP • ATALHO"
st.TextColor3 = Color3.fromRGB(220, 180, 240)
st.TextSize = 7
st.Font = Enum.Font.GothamMedium
st.TextXAlignment = Enum.TextXAlignment.Left

local btnMin = Instance.new("TextButton", hd)
btnMin.Size = UDim2.new(0, 20, 0, 20)
btnMin.Position = UDim2.new(1, -44, 0.5, -10)
btnMin.BackgroundColor3 = Color3.fromRGB(80, 40, 110)
btnMin.BackgroundTransparency = 0.1
btnMin.Text = "−"
btnMin.TextColor3 = Color3.new(1, 1, 1)
btnMin.TextSize = 13
btnMin.Font = Enum.Font.GothamMedium
btnMin.AutoButtonColor = false
CreateCorner(btnMin, 6)
CreateStroke(btnMin, Color3.fromRGB(180, 80, 220), 1, 0.3)

local CloseBtn = Instance.new("TextButton", hd)
CloseBtn.Size = UDim2.new(0, 20, 0, 20)
CloseBtn.Position = UDim2.new(1, -22, 0.5, -10)
CloseBtn.BackgroundColor3 = Color3.fromRGB(80, 40, 110)
CloseBtn.BackgroundTransparency = 0.1
CloseBtn.Text = "×"
CloseBtn.TextColor3 = Color3.new(1, 1, 1)
CloseBtn.TextSize = 13
CloseBtn.Font = Enum.Font.GothamMedium
CloseBtn.AutoButtonColor = false
CreateCorner(CloseBtn, 6)
CreateStroke(CloseBtn, Color3.fromRGB(180, 80, 220), 1, 0.3)

local Icon = Instance.new("ImageButton", ScreenGui)
Icon.Size = UDim2.new(0, 38, 0, 38)
Icon.Position = UDim2.new(0, 10, 0, 100)
Icon.Image = "rbxassetid://73630975144333"
Icon.ImageTransparency = 0.05
Icon.BackgroundColor3 = Color3.fromRGB(80, 40, 110)
Icon.BackgroundTransparency = 0.1
Icon.Visible = false
Icon.ZIndex = 10
CreateCorner(Icon, 10)
CreateStroke(Icon, Color3.fromRGB(180, 80, 220), 2, 0.3)

local minimizado = true

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

Main.Visible = false
Icon.Visible = true

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
            local viewport = Camera.ViewportSize
            local delta = input.Position - dragStart
            obj.Position = UDim2.new(0, math.clamp(startPos.X.Offset + delta.X, 0, math.max(0, viewport.X - obj.AbsoluteSize.X)), 0, math.clamp(startPos.Y.Offset + delta.Y, 0, math.max(0, viewport.Y - obj.AbsoluteSize.Y)))
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
        local viewport = Camera.ViewportSize
        local delta = input.Position - dragStart
        Icon.Position = UDim2.new(0, math.clamp(startPos.X.Offset + delta.X, 0, math.max(0, viewport.X - Icon.AbsoluteSize.X)), 0, math.clamp(startPos.Y.Offset + delta.Y, 0, math.max(0, viewport.Y - Icon.AbsoluteSize.Y)))
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dragging = false end
end)

local barraAbas = Instance.new("Frame", Main)
barraAbas.Size = UDim2.new(1, 0, 0, 26)
barraAbas.Position = UDim2.new(0, 0, 0, 36)
barraAbas.BackgroundColor3 = Color3.fromRGB(25, 12, 40)
barraAbas.BackgroundTransparency = 0
barraAbas.BorderSizePixel = 0

local abaAIM = Instance.new("TextButton", barraAbas)
abaAIM.Size = UDim2.new(0.24, 0, 0, 22)
abaAIM.Position = UDim2.new(0.005, 0, 0, 2)
abaAIM.BackgroundColor3 = Color3.fromRGB(90, 40, 130)
abaAIM.BackgroundTransparency = 0.1
abaAIM.Text = "AIM"
abaAIM.TextColor3 = Color3.fromRGB(255, 255, 255)
abaAIM.TextSize = 8
abaAIM.Font = Enum.Font.GothamBold
abaAIM.AutoButtonColor = false
CreateCorner(abaAIM, 6)
CreateStroke(abaAIM, Color3.fromRGB(200, 80, 240), 1, 0.3)

local abaESP = Instance.new("TextButton", barraAbas)
abaESP.Size = UDim2.new(0.24, 0, 0, 22)
abaESP.Position = UDim2.new(0.255, 0, 0, 2)
abaESP.BackgroundColor3 = Color3.fromRGB(50, 25, 75)
abaESP.BackgroundTransparency = 0.1
abaESP.Text = "ESP"
abaESP.TextColor3 = Color3.fromRGB(200, 190, 210)
abaESP.TextSize = 8
abaESP.Font = Enum.Font.GothamBold
abaESP.AutoButtonColor = false
CreateCorner(abaESP, 6)
CreateStroke(abaESP, Color3.fromRGB(130, 70, 180), 1, 0.4)

local abaTP = Instance.new("TextButton", barraAbas)
abaTP.Size = UDim2.new(0.24, 0, 0, 22)
abaTP.Position = UDim2.new(0.505, 0, 0, 2)
abaTP.BackgroundColor3 = Color3.fromRGB(50, 25, 75)
abaTP.BackgroundTransparency = 0.1
abaTP.Text = "TP"
abaTP.TextColor3 = Color3.fromRGB(200, 190, 210)
abaTP.TextSize = 8
abaTP.Font = Enum.Font.GothamBold
abaTP.AutoButtonColor = false
CreateCorner(abaTP, 6)
CreateStroke(abaTP, Color3.fromRGB(130, 70, 180), 1, 0.4)

local abaAtalho = Instance.new("TextButton", barraAbas)
abaAtalho.Size = UDim2.new(0.24, 0, 0, 22)
abaAtalho.Position = UDim2.new(0.755, 0, 0, 2)
abaAtalho.BackgroundColor3 = Color3.fromRGB(50, 25, 75)
abaAtalho.BackgroundTransparency = 0.1
abaAtalho.Text = "ATALHO"
abaAtalho.TextColor3 = Color3.fromRGB(200, 190, 210)
abaAtalho.TextSize = 7
abaAtalho.Font = Enum.Font.GothamBold
abaAtalho.AutoButtonColor = false
CreateCorner(abaAtalho, 6)
CreateStroke(abaAtalho, Color3.fromRGB(130, 70, 180), 1, 0.4)

local abaIndicator = Instance.new("Frame", barraAbas)
abaIndicator.Size = UDim2.new(0, 18, 0, 2)
abaIndicator.Position = UDim2.new(0.005, 0, 1, -1)
abaIndicator.BackgroundColor3 = Color3.fromRGB(200, 80, 240)
abaIndicator.BorderSizePixel = 0
CreateCorner(abaIndicator, 1)

local contentY = 62
local contentH = 228

local telaAIM = Instance.new("ScrollingFrame", Main)
telaAIM.Size = UDim2.new(1, 0, 0, contentH)
telaAIM.Position = UDim2.new(0, 0, 0, contentY)
telaAIM.BackgroundColor3 = Color3.fromRGB(25, 12, 40)
telaAIM.BackgroundTransparency = 0
telaAIM.ScrollBarThickness = 2
telaAIM.ScrollBarImageColor3 = Color3.fromRGB(180, 80, 220)
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
telaESP.BackgroundColor3 = Color3.fromRGB(25, 12, 40)
telaESP.BackgroundTransparency = 0
telaESP.ScrollBarThickness = 2
telaESP.ScrollBarImageColor3 = Color3.fromRGB(180, 80, 220)
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
telaTP.BackgroundColor3 = Color3.fromRGB(25, 12, 40)
telaTP.BackgroundTransparency = 0
telaTP.ScrollBarThickness = 2
telaTP.ScrollBarImageColor3 = Color3.fromRGB(180, 80, 220)
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

local telaAtalho = Instance.new("ScrollingFrame", Main)
telaAtalho.Size = UDim2.new(1, 0, 0, contentH)
telaAtalho.Position = UDim2.new(0, 0, 0, contentY)
telaAtalho.BackgroundColor3 = Color3.fromRGB(25, 12, 40)
telaAtalho.BackgroundTransparency = 0
telaAtalho.ScrollBarThickness = 2
telaAtalho.ScrollBarImageColor3 = Color3.fromRGB(180, 80, 220)
telaAtalho.BorderSizePixel = 0
telaAtalho.Visible = false
telaAtalho.CanvasSize = UDim2.new(0, 0, 0, 0)
telaAtalho.AutomaticCanvasSize = Enum.AutomaticSize.Y
local layAtalho = Instance.new("UIListLayout", telaAtalho)
layAtalho.Padding = UDim.new(0, 8)
layAtalho.HorizontalAlignment = Enum.HorizontalAlignment.Center
layAtalho.SortOrder = Enum.SortOrder.LayoutOrder
local padAtalho = Instance.new("UIPadding", telaAtalho)
padAtalho.PaddingTop = UDim.new(0, 10)
padAtalho.PaddingBottom = UDim.new(0, 10)
padAtalho.PaddingLeft = UDim.new(0, 8)
padAtalho.PaddingRight = UDim.new(0, 8)

-- ============================================================
-- COMPONENTES UI (ROXO VIBRANTE)
-- ============================================================
local selectedColorBtn = nil

local function criarSeletorCores(parent, callback)
    local frame = Instance.new("Frame", parent)
    frame.Size = UDim2.new(0.9, 0, 0, 42)
    frame.BackgroundColor3 = Color3.fromRGB(45, 25, 70)
    frame.BackgroundTransparency = 0
    frame.LayoutOrder = 99
    CreateCorner(frame, 7)
    CreateStroke(frame, Color3.fromRGB(130, 70, 180), 1, 0.3)
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
        CreateStroke(bt, Color3.fromRGB(180, 80, 220), 1, 0.3)
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
    btn.BackgroundColor3 = Color3.fromRGB(45, 25, 70)
    btn.BackgroundTransparency = 0
    btn.Text = ""
    btn.AutoButtonColor = false
    btn.BorderSizePixel = 0
    CreateCorner(btn, 7)
    CreateStroke(btn, Color3.fromRGB(130, 70, 180), 1, 0.3)
    
    local label = Instance.new("TextLabel", btn)
    label.Size = UDim2.new(0.55, 0, 1, 0)
    label.Position = UDim2.new(0, 9, 0, 0)
    label.BackgroundTransparency = 1
    label.Text = name
    label.TextColor3 = Color3.fromRGB(255, 255, 255)
    label.TextSize = 8
    label.Font = Enum.Font.GothamBold
    label.TextXAlignment = Enum.TextXAlignment.Left
    
    local toggleDot = Instance.new("Frame", btn)
    toggleDot.Size = UDim2.new(0, 30, 0, 16)
    toggleDot.Position = UDim2.new(1, -38, 0.5, -8)
    toggleDot.BackgroundColor3 = Color3.fromRGB(60, 35, 85)
    toggleDot.BorderSizePixel = 0
    CreateCorner(toggleDot, 8)
    CreateStroke(toggleDot, Color3.fromRGB(150, 70, 200), 1, 0.3)
    
    local dot = Instance.new("Frame", toggleDot)
    dot.Size = UDim2.new(0, 12, 0, 12)
    dot.Position = UDim2.new(0, 2, 0.5, -6)
    dot.BackgroundColor3 = Color3.fromRGB(160, 130, 180)
    dot.BorderSizePixel = 0
    CreateCorner(dot, 6)
    
    local function updateToggle(showNotif)
        local on = getgenv().Config[prop]
        ts:Create(dot, TweenInfo.new(0.15, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Position = UDim2.new(0, on and 16 or 2, 0.5, -6), BackgroundColor3 = on and Color3.fromRGB(220, 100, 255) or Color3.fromRGB(160, 130, 180)}):Play()
        ts:Create(toggleDot, TweenInfo.new(0.15), {BackgroundColor3 = on and Color3.fromRGB(80, 40, 110) or Color3.fromRGB(60, 35, 85)}):Play()
        ts:Create(btn, TweenInfo.new(0.15), {BackgroundColor3 = on and Color3.fromRGB(65, 35, 95) or Color3.fromRGB(45, 25, 70)}):Play()
        if showNotif then
            if prop == "ButtonTP" then
                ShowNotification(on and "ATALHO TP ATIVADO" or "ATALHO TP DESATIVADO", on)
            elseif prop == "ButtonBack" then
                ShowNotification(on and "ATALHO BACK ATIVADO" or "ATALHO BACK DESATIVADO", on)
            elseif prop == "TPInteracao" then
                ShowNotification(on and "TP INTERAÇÃO ATIVADO" or "TP INTERAÇÃO DESATIVADO", on)
            elseif prop == "ButtonFreecam" then
                ShowNotification(on and "BUTTON FREECAM ATIVO" or "BUTTON FREECAM DESATIVADO", on)
            else
                ShowNotification(name .. (on and " ATIVO" or " DESATIVADO"), on)
            end
        end
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
                if sairAimContadorLabel then sairAimContadorLabel.Text = "SAÍDAS: 0/" .. tostring(SairAim.Attempts); sairAimContadorLabel.Visible = true end
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
        if prop == "TPInteracao" then
            if getgenv().Config.TPInteracao then StartTPInteracao() else StopTPInteracao() end
        end
        if prop == "ButtonFreecam" then
            if not getgenv().Config.ButtonFreecam and FreecamEnabled then DisableFreecam() end
        end
        if prop == "ButtonTP" or prop == "ButtonBack" or prop == "ButtonFreecam" then UpdateShortcutVisibility() end
    end)
    return btn
end

local function AddSliderVertical(name, prop, parent, max, min, suffix)
    min = min or 0
    suffix = suffix or ""
    local frame = Instance.new("Frame", parent)
    frame.Size = UDim2.new(0.9, 0, 0, 35)
    frame.BackgroundColor3 = Color3.fromRGB(40, 22, 65)
    frame.BackgroundTransparency = 0
    frame.BorderSizePixel = 0
    CreateCorner(frame, 7)
    CreateStroke(frame, Color3.fromRGB(130, 70, 180), 1, 0.3)
    
    local label = Instance.new("TextLabel", frame)
    label.Size = UDim2.new(0.5, 0, 0, 15)
    label.Position = UDim2.new(0, 7, 0, 2)
    label.BackgroundTransparency = 1
    label.Text = name
    label.TextColor3 = Color3.fromRGB(255, 255, 255)
    label.TextSize = 7
    label.Font = Enum.Font.GothamBold
    label.TextXAlignment = Enum.TextXAlignment.Left
    
    local valueLabel = Instance.new("TextLabel", frame)
    valueLabel.Size = UDim2.new(0.4, 0, 0, 15)
    valueLabel.Position = UDim2.new(0.6, 0, 0, 2)
    valueLabel.BackgroundTransparency = 1
    valueLabel.Text = getgenv().Config[prop] .. suffix
    valueLabel.TextColor3 = Color3.fromRGB(220, 140, 250)
    valueLabel.TextSize = 8
    valueLabel.Font = Enum.Font.GothamBold
    valueLabel.TextXAlignment = Enum.TextXAlignment.Right
    
    local barBg = Instance.new("Frame", frame)
    barBg.Size = UDim2.new(0.9, 0, 0, 6)
    barBg.Position = UDim2.new(0.05, 0, 0, 20)
    barBg.BackgroundColor3 = Color3.fromRGB(60, 35, 85)
    barBg.BorderSizePixel = 0
    CreateCorner(barBg, 3)
    
    local barFill = Instance.new("Frame", barBg)
    local percent = (getgenv().Config[prop] - min) / (max - min)
    barFill.Size = UDim2.new(percent, 0, 1, 0)
    barFill.BackgroundColor3 = Color3.fromRGB(200, 80, 240)
    barFill.BorderSizePixel = 0
    CreateCorner(barFill, 3)
    
    local fillGrad = Instance.new("UIGradient", barFill)
    fillGrad.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(160, 40, 200)),
        ColorSequenceKeypoint.new(0.5, Color3.fromRGB(230, 100, 255)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(160, 40, 200))
    })
    fillGrad.Rotation = 90
    
    local dot = Instance.new("TextButton", barBg)
    dot.Size = UDim2.new(0, 14, 0, 14)
    dot.Position = UDim2.new(percent, -7, 0.5, -7)
    dot.BackgroundColor3 = Color3.new(1, 1, 1)
    dot.Text = ""
    dot.AutoButtonColor = false
    CreateCorner(dot, 7)
    CreateStroke(dot, Color3.fromRGB(200, 80, 240), 2, 0)
    
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

-- ============================================================
-- SAIR AIM
-- ============================================================
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
    task.delay(SairAimReleaseTime, function() AimPermitido = true; SairAim.Processing = false end)
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

-- ============================================================
-- ABA AIM
-- ============================================================
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
PartBtn.BackgroundColor3 = Color3.fromRGB(45, 25, 70)
PartBtn.BackgroundTransparency = 0
PartBtn.Text = "ALVO: CABEÇA"
PartBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
PartBtn.TextSize = 8
PartBtn.Font = Enum.Font.GothamBold
PartBtn.AutoButtonColor = false
CreateCorner(PartBtn, 7)
CreateStroke(PartBtn, Color3.fromRGB(180, 80, 220), 1, 0.3)
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
sairAimInput.BackgroundColor3 = Color3.fromRGB(50, 28, 75)
sairAimInput.BackgroundTransparency = 0
sairAimInput.TextColor3 = Color3.fromRGB(255, 255, 255)
sairAimInput.PlaceholderColor3 = Color3.fromRGB(180, 150, 200)
sairAimInput.PlaceholderText = "QUANTAS TENTATIVAS"
sairAimInput.Text = "3"
sairAimInput.ClearTextOnFocus = false
sairAimInput.TextSize = 9
sairAimInput.Font = Enum.Font.GothamBold
sairAimInput.TextXAlignment = Enum.TextXAlignment.Center
sairAimInput.Visible = false
sairAimInput.LayoutOrder = 99
CreateCorner(sairAimInput, 7)
CreateStroke(sairAimInput, Color3.fromRGB(180, 80, 220), 1, 0.3)

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
sairAimContadorLabel.TextColor3 = Color3.fromRGB(220, 140, 250)
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

-- ============================================================
-- ABA ESP
-- ============================================================
CreateSectionLabel(telaESP, "ESP")
AddToggleVertical("ESP NOME", "ESP", telaESP)
AddToggleVertical("ESP BOX", "ESP_Box", telaESP)
AddToggleVertical("ESP LINHA", "ESP_Line", telaESP)
AddToggleVertical("ESP DIST", "Distance", telaESP)
AddToggleVertical("ESP VIDA", "Health", telaESP)

CreateSectionLabel(telaESP, "COR ESP")
criarSeletorCores(telaESP, function(cor)
    espColor = cor
    for p, line in pairs(ESP_Lines) do if line then line.Color = cor end end
    for p, obj in pairs(ESP_Table) do
        if obj.Box and obj.Box.UIStroke then obj.Box.UIStroke.Color = cor end
        if obj.Label then obj.Label.TextColor3 = cor end
    end
end)

-- ============================================================
-- ABA TP
-- ============================================================
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
tpCoordLabel.TextColor3 = Color3.fromRGB(220, 140, 250)
tpCoordLabel.TextSize = 9
tpCoordLabel.Font = Enum.Font.GothamMedium
tpCoordLabel.TextXAlignment = Enum.TextXAlignment.Center
tpCoordLabel.LayoutOrder = 2

UpdateTPUI()

local marcarBtn = Instance.new("TextButton", telaTP)
marcarBtn.Size = UDim2.new(0.9, 0, 0, 34)
marcarBtn.BackgroundColor3 = Color3.fromRGB(50, 28, 75)
marcarBtn.BackgroundTransparency = 0
marcarBtn.Text = "📍 MARCAR"
marcarBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
marcarBtn.TextSize = 11
marcarBtn.Font = Enum.Font.GothamBold
marcarBtn.AutoButtonColor = false
marcarBtn.LayoutOrder = 3
CreateCorner(marcarBtn, 7)
CreateStroke(marcarBtn, Color3.fromRGB(180, 80, 220), 1.5, 0.3)

local marcarGrad = Instance.new("UIGradient", marcarBtn)
marcarGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(100, 40, 140)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(160, 60, 200)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(100, 40, 140))
})
marcarGrad.Rotation = 90

marcarBtn.MouseButton1Click:Connect(function()
    if tpOptionFrame then tpOptionFrame:Destroy(); tpOptionFrame = nil end
    tpOptionFrame = Instance.new("Frame", telaTP)
    tpOptionFrame.Size = UDim2.new(0.9, 0, 0, 34)
    tpOptionFrame.BackgroundTransparency = 1
    tpOptionFrame.LayoutOrder = 4
    local dedoBtn = Instance.new("TextButton", tpOptionFrame)
    dedoBtn.Size = UDim2.new(0.48, 0, 0, 30)
    dedoBtn.Position = UDim2.new(0, 0, 0, 2)
    dedoBtn.BackgroundColor3 = Color3.fromRGB(55, 30, 80)
    dedoBtn.BackgroundTransparency = 0
    dedoBtn.Text = "👆 DEDO"
    dedoBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    dedoBtn.TextSize = 10
    dedoBtn.Font = Enum.Font.GothamBold
    dedoBtn.AutoButtonColor = false
    CreateCorner(dedoBtn, 7)
    CreateStroke(dedoBtn, Color3.fromRGB(180, 80, 220), 1, 0.3)
    local agoraBtn = Instance.new("TextButton", tpOptionFrame)
    agoraBtn.Size = UDim2.new(0.48, 0, 0, 30)
    agoraBtn.Position = UDim2.new(0.52, 0, 0, 2)
    agoraBtn.BackgroundColor3 = Color3.fromRGB(55, 30, 80)
    agoraBtn.BackgroundTransparency = 0
    agoraBtn.Text = "⚡ AGORA"
    agoraBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    agoraBtn.TextSize = 10
    agoraBtn.Font = Enum.Font.GothamBold
    agoraBtn.AutoButtonColor = false
    CreateCorner(agoraBtn, 7)
    CreateStroke(agoraBtn, Color3.fromRGB(180, 80, 220), 1, 0.3)
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

local tpButtonFrame = Instance.new("Frame", telaTP)
tpButtonFrame.Size = UDim2.new(0.9, 0, 0, 34)
tpButtonFrame.BackgroundTransparency = 1
tpButtonFrame.LayoutOrder = 5

local tpBtn = Instance.new("TextButton", tpButtonFrame)
tpBtn.Size = UDim2.new(0.78, 0, 0, 34)
tpBtn.Position = UDim2.new(0, 0, 0, 0)
tpBtn.BackgroundColor3 = Color3.fromRGB(50, 28, 75)
tpBtn.BackgroundTransparency = 0
tpBtn.Text = "🌀 TELEPORTAR"
tpBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
tpBtn.TextSize = 11
tpBtn.Font = Enum.Font.GothamBold
tpBtn.AutoButtonColor = false
CreateCorner(tpBtn, 7)
CreateStroke(tpBtn, Color3.fromRGB(180, 80, 220), 1.5, 0.3)

local tpBtnGrad = Instance.new("UIGradient", tpBtn)
tpBtnGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(100, 40, 140)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(160, 60, 200)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(100, 40, 140))
})
tpBtnGrad.Rotation = 90

backTPBtn = Instance.new("TextButton", tpButtonFrame)
backTPBtn.Size = UDim2.new(0, 38, 0, 34)
backTPBtn.Position = UDim2.new(0.82, 4, 0, 0)
backTPBtn.BackgroundColor3 = Color3.fromRGB(50, 28, 75)
backTPBtn.BackgroundTransparency = 0
backTPBtn.Text = "⬅️"
backTPBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
backTPBtn.TextSize = 14
backTPBtn.Font = Enum.Font.GothamBold
backTPBtn.AutoButtonColor = false
backTPBtn.Visible = false
CreateCorner(backTPBtn, 7)
CreateStroke(backTPBtn, Color3.fromRGB(180, 80, 220), 1.5, 0.3)

tpBtn.MouseButton1Click:Connect(function() TeleportToMarkedPosition() end)
backTPBtn.MouseButton1Click:Connect(function()
    AnimateButton(backTPBtn, {Size = UDim2.new(0, 34, 0, 30)}, 0.08)
    TeleportBack()
    AnimateButton(backTPBtn, {Size = UDim2.new(0, 38, 0, 34)}, 0.08)
end)

-- Botão FORÇAR TP
local btnForcar = Instance.new("TextButton", telaTP)
btnForcar.Size = UDim2.new(0.9, 0, 0, 28)
btnForcar.BackgroundColor3 = Color3.fromRGB(55, 30, 80)
btnForcar.BackgroundTransparency = 0
btnForcar.Text = "⚡ FORÇAR TP (FALLBACK)"
btnForcar.TextColor3 = Color3.fromRGB(255, 210, 130)
btnForcar.TextSize = 9
btnForcar.Font = Enum.Font.GothamBold
btnForcar.AutoButtonColor = false
btnForcar.LayoutOrder = 6
CreateCorner(btnForcar, 7)
CreateStroke(btnForcar, Color3.fromRGB(255, 210, 130), 1, 0.3)
btnForcar.MouseButton1Click:Connect(function()
    if not markedPosition then ShowNotification("NENHUM LOCAL MARCADO", false); return end
    ShowNotification("🔄 FORÇANDO TP...", true)
    local character = LocalPlayer.Character
    if not character then ShowNotification("SEM PERSONAGEM", false); return end
    local root = character:FindFirstChild("HumanoidRootPart")
    if not root then ShowNotification("SEM ROOT", false); return end
    pcall(function() root.AssemblyLinearVelocity = Vector3.zero; root.AssemblyAngularVelocity = Vector3.zero; character:PivotTo(CFrame.new(markedPosition)) end)
    task.wait(0.05)
    pcall(function() root.CFrame = CFrame.new(markedPosition) end)
    task.wait(0.05)
    local distancia = (root.Position - markedPosition).Magnitude
    if distancia < 10 then ShowNotification("✅ FORÇADO COM SUCESSO!", true) else ShowNotification("⚠️ FALHA NO FORÇADO", false) end
end)

-- ============================================================
-- ABA ATALHO
-- ============================================================
CreateSectionLabel(telaAtalho, "ATALHO")
AddToggleVertical("Button TP", "ButtonTP", telaAtalho)
AddToggleVertical("Button BACK", "ButtonBack", telaAtalho)
AddToggleVertical("TP INTERAÇÃO", "TPInteracao", telaAtalho)

local tpDelayContainer = Instance.new("Frame", telaAtalho)
tpDelayContainer.Size = UDim2.new(0.9, 0, 0, 32)
tpDelayContainer.BackgroundColor3 = Color3.fromRGB(40, 22, 65)
tpDelayContainer.BackgroundTransparency = 0
tpDelayContainer.BorderSizePixel = 0
CreateCorner(tpDelayContainer, 7)
CreateStroke(tpDelayContainer, Color3.fromRGB(130, 70, 180), 1, 0.3)

local tpDelayLabel = Instance.new("TextLabel", tpDelayContainer)
tpDelayLabel.Size = UDim2.new(0.4, 0, 1, 0)
tpDelayLabel.Position = UDim2.new(0, 9, 0, 0)
tpDelayLabel.BackgroundTransparency = 1
tpDelayLabel.Text = "DELAY:"
tpDelayLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
tpDelayLabel.TextSize = 8
tpDelayLabel.Font = Enum.Font.GothamBold
tpDelayLabel.TextXAlignment = Enum.TextXAlignment.Left

local tpDelayBox = Instance.new("TextBox", tpDelayContainer)
tpDelayBox.Size = UDim2.new(0, 80, 0, 24)
tpDelayBox.Position = UDim2.new(0.5, 0, 0.5, -12)
tpDelayBox.BackgroundColor3 = Color3.fromRGB(50, 28, 75)
tpDelayBox.BackgroundTransparency = 0
tpDelayBox.TextColor3 = Color3.fromRGB(255, 255, 255)
tpDelayBox.PlaceholderColor3 = Color3.fromRGB(180, 150, 200)
tpDelayBox.PlaceholderText = "0.50"
tpDelayBox.Text = tostring(getgenv().Config.TPInteracaoDelay)
tpDelayBox.ClearTextOnFocus = false
tpDelayBox.TextSize = 11
tpDelayBox.Font = Enum.Font.GothamBold
tpDelayBox.TextXAlignment = Enum.TextXAlignment.Center
CreateCorner(tpDelayBox, 6)
CreateStroke(tpDelayBox, Color3.fromRGB(180, 80, 220), 1, 0.3)

tpDelayBox.FocusLost:Connect(function()
    local value = tonumber(tpDelayBox.Text)
    if value == nil then value = 0.50 end
    value = math.max(0, value)
    getgenv().Config.TPInteracaoDelay = value
    tpDelayBox.Text = tostring(value)
    ShowNotification("DELAY: " .. tostring(value) .. "s", true)
end)

AddToggleVertical("Button FREECAM", "ButtonFreecam", telaAtalho)

shortcutConfigButton = Instance.new("TextButton", telaAtalho)
shortcutConfigButton.Size = UDim2.new(0.5, 0, 0, 36)
shortcutConfigButton.BackgroundColor3 = Color3.fromRGB(50, 28, 75)
shortcutConfigButton.BackgroundTransparency = 0
shortcutConfigButton.Text = "⚙️"
shortcutConfigButton.TextColor3 = Color3.fromRGB(255, 255, 255)
shortcutConfigButton.TextSize = 16
shortcutConfigButton.Font = Enum.Font.GothamBold
shortcutConfigButton.AutoButtonColor = false
shortcutConfigButton.LayoutOrder = 10
CreateCorner(shortcutConfigButton, 7)
CreateStroke(shortcutConfigButton, Color3.fromRGB(180, 80, 220), 1.5, 0.3)
shortcutConfigButton.MouseButton1Click:Connect(function() EnterShortcutEditMode() end)

-- ============================================================
-- BOTÕES FLUTUANTES (COM ÍCONES)
-- ============================================================
shortcutTPButton = Instance.new("TextButton", ScreenGui)
shortcutTPButton.Size = UDim2.new(0, 50, 0, 50)
shortcutTPButton.Position = ShortcutPositions.TP
shortcutTPButton.Text = "🚪"
shortcutTPButton.TextSize = 22
shortcutTPButton.BackgroundColor3 = Color3.fromRGB(50, 28, 75)
shortcutTPButton.BackgroundTransparency = 0
shortcutTPButton.TextColor3 = Color3.new(1, 1, 1)
shortcutTPButton.Font = Enum.Font.GothamBold
shortcutTPButton.AutoButtonColor = false
shortcutTPButton.Visible = false
shortcutTPButton.ZIndex = 500
CreateCorner(shortcutTPButton, 25)
CreateStroke(shortcutTPButton, Color3.fromRGB(200, 80, 240), 2, 0.3)

local tpGrad = Instance.new("UIGradient", shortcutTPButton)
tpGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(100, 40, 140)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(180, 60, 220)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(100, 40, 140))
})
tpGrad.Rotation = 45

shortcutBackButton = Instance.new("TextButton", ScreenGui)
shortcutBackButton.Size = UDim2.new(0, 50, 0, 50)
shortcutBackButton.Position = ShortcutPositions.BACK
shortcutBackButton.Text = "↩️"
shortcutBackButton.TextSize = 22
shortcutBackButton.BackgroundColor3 = Color3.fromRGB(50, 28, 75)
shortcutBackButton.BackgroundTransparency = 0
shortcutBackButton.TextColor3 = Color3.new(1, 1, 1)
shortcutBackButton.Font = Enum.Font.GothamBold
shortcutBackButton.AutoButtonColor = false
shortcutBackButton.Visible = false
shortcutBackButton.ZIndex = 500
CreateCorner(shortcutBackButton, 25)
CreateStroke(shortcutBackButton, Color3.fromRGB(200, 80, 240), 2, 0.3)

local backGrad = Instance.new("UIGradient", shortcutBackButton)
backGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(100, 40, 140)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(180, 60, 220)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(100, 40, 140))
})
backGrad.Rotation = 45

shortcutFreecamGroup = Instance.new("Frame", ScreenGui)
shortcutFreecamGroup.Size = UDim2.new(0, 170, 0, 50)
shortcutFreecamGroup.Position = ShortcutPositions.FREECAM
shortcutFreecamGroup.BackgroundTransparency = 1
shortcutFreecamGroup.Visible = false
shortcutFreecamGroup.ZIndex = 500

shortcutFreecamButton = Instance.new("TextButton", shortcutFreecamGroup)
shortcutFreecamButton.Size = UDim2.new(0, 70, 0, 50)
shortcutFreecamButton.Position = UDim2.new(0, 0, 0, 0)
shortcutFreecamButton.Text = "👁️"
shortcutFreecamButton.TextSize = 22
shortcutFreecamButton.BackgroundColor3 = Color3.fromRGB(50, 28, 75)
shortcutFreecamButton.BackgroundTransparency = 0
shortcutFreecamButton.TextColor3 = Color3.new(1, 1, 1)
shortcutFreecamButton.Font = Enum.Font.GothamBold
shortcutFreecamButton.AutoButtonColor = false
CreateCorner(shortcutFreecamButton, 25)
CreateStroke(shortcutFreecamButton, Color3.fromRGB(200, 80, 240), 2, 0.3)

local freeGrad = Instance.new("UIGradient", shortcutFreecamButton)
freeGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(100, 40, 140)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(180, 60, 220)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(100, 40, 140))
})
freeGrad.Rotation = 45

freecamMinusButton = Instance.new("TextButton", shortcutFreecamGroup)
freecamMinusButton.Size = UDim2.new(0, 30, 0, 50)
freecamMinusButton.Position = UDim2.new(0, 75, 0, 0)
freecamMinusButton.BackgroundColor3 = Color3.fromRGB(50, 28, 75)
freecamMinusButton.BackgroundTransparency = 0
freecamMinusButton.Text = "-"
freecamMinusButton.TextColor3 = Color3.new(1, 1, 1)
freecamMinusButton.TextSize = 16
freecamMinusButton.Font = Enum.Font.GothamBold
freecamMinusButton.AutoButtonColor = false
freecamMinusButton.Visible = false
CreateCorner(freecamMinusButton, 25)
CreateStroke(freecamMinusButton, Color3.fromRGB(200, 80, 240), 1, 0.3)

freecamSpeedLabel = Instance.new("TextLabel", shortcutFreecamGroup)
freecamSpeedLabel.Size = UDim2.new(0, 40, 0, 50)
freecamSpeedLabel.Position = UDim2.new(0, 105, 0, 0)
freecamSpeedLabel.BackgroundTransparency = 1
freecamSpeedLabel.Text = "1.0x"
freecamSpeedLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
freecamSpeedLabel.TextSize = 10
freecamSpeedLabel.Font = Enum.Font.GothamBold
freecamSpeedLabel.TextXAlignment = Enum.TextXAlignment.Center
freecamSpeedLabel.Visible = false

freecamPlusButton = Instance.new("TextButton", shortcutFreecamGroup)
freecamPlusButton.Size = UDim2.new(0, 30, 0, 50)
freecamPlusButton.Position = UDim2.new(0, 140, 0, 0)
freecamPlusButton.BackgroundColor3 = Color3.fromRGB(50, 28, 75)
freecamPlusButton.BackgroundTransparency = 0
freecamPlusButton.Text = "+"
freecamPlusButton.TextColor3 = Color3.new(1, 1, 1)
freecamPlusButton.TextSize = 16
freecamPlusButton.Font = Enum.Font.GothamBold
freecamPlusButton.AutoButtonColor = false
freecamPlusButton.Visible = false
CreateCorner(freecamPlusButton, 25)
CreateStroke(freecamPlusButton, Color3.fromRGB(200, 80, 240), 1, 0.3)

shortcutEditBar = Instance.new("Frame", ScreenGui)
shortcutEditBar.Size = UDim2.new(0, 200, 0, 40)
shortcutEditBar.Position = UDim2.new(0.5, -100, 0.02, 0)
shortcutEditBar.BackgroundColor3 = Color3.fromRGB(45, 25, 70)
shortcutEditBar.BorderSizePixel = 0
shortcutEditBar.Visible = false
shortcutEditBar.ZIndex = 600
CreateCorner(shortcutEditBar, 10)
CreateStroke(shortcutEditBar, Color3.fromRGB(180, 80, 220), 1.5, 0.3)

local cancelEditBtn = Instance.new("TextButton", shortcutEditBar)
cancelEditBtn.Size = UDim2.new(0.45, 0, 0, 30)
cancelEditBtn.Position = UDim2.new(0.03, 0, 0.5, -15)
cancelEditBtn.BackgroundColor3 = Color3.fromRGB(80, 40, 110)
cancelEditBtn.Text = "Cancelar"
cancelEditBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
cancelEditBtn.TextSize = 10
cancelEditBtn.Font = Enum.Font.GothamBold
cancelEditBtn.AutoButtonColor = false
cancelEditBtn.ZIndex = 601
CreateCorner(cancelEditBtn, 7)
CreateStroke(cancelEditBtn, Color3.fromRGB(180, 80, 220), 1, 0.3)

local saveEditBtn = Instance.new("TextButton", shortcutEditBar)
saveEditBtn.Size = UDim2.new(0.45, 0, 0, 30)
saveEditBtn.Position = UDim2.new(0.52, 0, 0.5, -15)
saveEditBtn.BackgroundColor3 = Color3.fromRGB(80, 40, 110)
saveEditBtn.Text = "Salvar"
saveEditBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
saveEditBtn.TextSize = 10
saveEditBtn.Font = Enum.Font.GothamBold
saveEditBtn.AutoButtonColor = false
saveEditBtn.ZIndex = 601
CreateCorner(saveEditBtn, 7)
CreateStroke(saveEditBtn, Color3.fromRGB(180, 80, 220), 1, 0.3)

cancelEditBtn.MouseButton1Click:Connect(function() CancelShortcutEdit() end)
saveEditBtn.MouseButton1Click:Connect(function() SaveShortcutEdit() end)

MakeShortcutDraggable(shortcutTPButton)
MakeShortcutDraggable(shortcutBackButton)
MakeShortcutDraggable(shortcutFreecamGroup)
UpdateShortcutVisibility()

shortcutTPButton.MouseButton1Click:Connect(function()
    if not ShortcutState.EditMode then TeleportToMarkedPosition() end
end)

shortcutBackButton.MouseButton1Click:Connect(function()
    if not ShortcutState.EditMode then TeleportBack() end
end)

shortcutFreecamButton.MouseButton1Click:Connect(function()
    if not ShortcutState.EditMode then
        if FreecamEnabled then DisableFreecam() else EnableFreecam() end
    end
end)

freecamMinusButton.MouseButton1Click:Connect(function()
    if ShortcutState.EditMode then return end
    speedMultiplier = math.clamp(speedMultiplier - speedStep, minSpeed, maxSpeed)
    UpdateFreecamUI()
end)

freecamPlusButton.MouseButton1Click:Connect(function()
    if ShortcutState.EditMode then return end
    speedMultiplier = math.clamp(speedMultiplier + speedStep, minSpeed, maxSpeed)
    UpdateFreecamUI()
end)

-- ============================================================
-- SELEÇÃO DE ABA
-- ============================================================
local function selecionarAba(aba)
    telaAIM.Visible = (aba == "aim")
    telaESP.Visible = (aba == "esp")
    telaTP.Visible = (aba == "tp")
    telaAtalho.Visible = (aba == "atalho")
    if aba ~= "tp" then
        CancelDedoSelection()
        if tpOptionFrame then tpOptionFrame:Destroy(); tpOptionFrame = nil end
    end
    if aba == "aim" then
        ts:Create(abaAIM, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(90, 40, 130), BackgroundTransparency = 0.1, TextColor3 = Color3.fromRGB(255, 255, 255)}):Play()
        ts:Create(abaESP, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(50, 25, 75), BackgroundTransparency = 0.1, TextColor3 = Color3.fromRGB(200, 190, 210)}):Play()
        ts:Create(abaTP, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(50, 25, 75), BackgroundTransparency = 0.1, TextColor3 = Color3.fromRGB(200, 190, 210)}):Play()
        ts:Create(abaAtalho, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(50, 25, 75), BackgroundTransparency = 0.1, TextColor3 = Color3.fromRGB(200, 190, 210)}):Play()
        ts:Create(abaIndicator, TweenInfo.new(0.15), {Position = UDim2.new(0.005, 0, 1, -1)}):Play()
    elseif aba == "esp" then
        ts:Create(abaESP, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(90, 40, 130), BackgroundTransparency = 0.1, TextColor3 = Color3.fromRGB(255, 255, 255)}):Play()
        ts:Create(abaAIM, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(50, 25, 75), BackgroundTransparency = 0.1, TextColor3 = Color3.fromRGB(200, 190, 210)}):Play()
        ts:Create(abaTP, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(50, 25, 75), BackgroundTransparency = 0.1, TextColor3 = Color3.fromRGB(200, 190, 210)}):Play()
        ts:Create(abaAtalho, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(50, 25, 75), BackgroundTransparency = 0.1, TextColor3 = Color3.fromRGB(200, 190, 210)}):Play()
        ts:Create(abaIndicator, TweenInfo.new(0.15), {Position = UDim2.new(0.255, 0, 1, -1)}):Play()
    elseif aba == "tp" then
        ts:Create(abaTP, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(90, 40, 130), BackgroundTransparency = 0.1, TextColor3 = Color3.fromRGB(255, 255, 255)}):Play()
        ts:Create(abaAIM, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(50, 25, 75), BackgroundTransparency = 0.1, TextColor3 = Color3.fromRGB(200, 190, 210)}):Play()
        ts:Create(abaESP, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(50, 25, 75), BackgroundTransparency = 0.1, TextColor3 = Color3.fromRGB(200, 190, 210)}):Play()
        ts:Create(abaAtalho, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(50, 25, 75), BackgroundTransparency = 0.1, TextColor3 = Color3.fromRGB(200, 190, 210)}):Play()
        ts:Create(abaIndicator, TweenInfo.new(0.15), {Position = UDim2.new(0.505, 0, 1, -1)}):Play()
    else
        ts:Create(abaAtalho, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(90, 40, 130), BackgroundTransparency = 0.1, TextColor3 = Color3.fromRGB(255, 255, 255)}):Play()
        ts:Create(abaAIM, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(50, 25, 75), BackgroundTransparency = 0.1, TextColor3 = Color3.fromRGB(200, 190, 210)}):Play()
        ts:Create(abaESP, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(50, 25, 75), BackgroundTransparency = 0.1, TextColor3 = Color3.fromRGB(200, 190, 210)}):Play()
        ts:Create(abaTP, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(50, 25, 75), BackgroundTransparency = 0.1, TextColor3 = Color3.fromRGB(200, 190, 210)}):Play()
        ts:Create(abaIndicator, TweenInfo.new(0.15), {Position = UDim2.new(0.755, 0, 1, -1)}):Play()
    end
end

abaAIM.MouseButton1Click:Connect(function() selecionarAba("aim") end)
abaESP.MouseButton1Click:Connect(function() selecionarAba("esp") end)
abaTP.MouseButton1Click:Connect(function() selecionarAba("tp") end)
abaAtalho.MouseButton1Click:Connect(function() selecionarAba("atalho") end)
selecionarAba("aim")

-- ============================================================
-- RESPAWN HANDLER
-- ============================================================
LocalPlayer.CharacterAdded:Connect(function(character)
    if FreecamEnabled then
        task.wait(0.5)
        if FreecamEnabled then
            Camera.CameraType = Enum.CameraType.Custom
            Camera.CameraSubject = movePart
        end
    end
end)

-- ============================================================
-- ESP SYSTEM
-- ============================================================
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

-- ============================================================
-- RENDER LOOP (AIM + ESP)
-- ============================================================
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

-- Iniciar TP Interação se já estiver ativo
if getgenv().Config.TPInteracao then
    StartTPInteracao()
end
