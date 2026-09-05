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
if getgenv().Config.TPInteracao == nil then getgenv().Config.TPInteracao = false end
if getgenv().Config.TPInteracaoDelay == nil then getgenv().Config.TPInteracaoDelay = 0.50 end
if getgenv().Config.ButtonFreecam == nil then getgenv().Config.ButtonFreecam = false end
if getgenv().Config.ProxESP == nil then getgenv().Config.ProxESP = false end
if getgenv().Config.ProxInstant == nil then getgenv().Config.ProxInstant = false end
if getgenv().Config.ProxAutoFarm == nil then getgenv().Config.ProxAutoFarm = false end
if getgenv().Config.ProxHideFiltered == nil then getgenv().Config.ProxHideFiltered = false end
if getgenv().Config.ProxShowOnlyFiltered == nil then getgenv().Config.ProxShowOnlyFiltered = false end
if getgenv().Config.ProxHideFiltered and getgenv().Config.ProxShowOnlyFiltered then getgenv().Config.ProxShowOnlyFiltered = false end

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
-- TEMA CLARO
-- ============================================================
local THEME = {
    bg        = Color3.fromRGB(240, 235, 250),
    bgSoft    = Color3.fromRGB(250, 247, 255),
    card      = Color3.fromRGB(228, 220, 245),
    cardHover = Color3.fromRGB(216, 204, 240),
    accent    = Color3.fromRGB(150, 90, 235),
    accent2   = Color3.fromRGB(190, 140, 250),
    stroke    = Color3.fromRGB(180, 140, 240),
    text      = Color3.fromRGB(60, 40, 95),
    textSoft  = Color3.fromRGB(110, 90, 145),
    white     = Color3.fromRGB(255, 255, 255),
    ok        = Color3.fromRGB(70, 190, 130),
    warn      = Color3.fromRGB(235, 170, 60),
    bad       = Color3.fromRGB(235, 95, 110),
}

-- ============================================================
-- TP STATE
-- ============================================================
local TPState = {
    Mode = "NONE",
    HasMark = false,
    IsSelecting = false,
    IsExecuting = false,
}

local markedPosition = nil
local tpMarker = nil
local TPConnections = {}

-- ============================================================
-- ATALHO STATE
-- ============================================================
local ShortcutState = { EditMode = false }

local ShortcutPositions = {
    TP = UDim2.fromOffset(60, 200),
    FREECAM = UDim2.fromOffset(60, 260)
}

local editStartTPPosition = nil
local editStartFreecamPosition = nil

local shortcutTPButton = nil
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
local TPInteracaoState = { Enabled = false, Waiting = false, InteractionId = 0 }
local TPInteracaoConnections = {}
local tpInteracaoPending = false

-- ============================================================
-- FREECAM STATE
-- ============================================================
_G.JoystickData = _G.JoystickData or { DraggingLevel = 0, Direction = Vector3.new(0, 0, 0) }

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
-- FORWARD DECLARATIONS
-- ============================================================
local ScreenGui
local tpStatusLabel
local tpCoordLabel
local tpOptionFrame
local CancelDedoSelection
local ShowNotification
local playPopSound

-- ============================================================
-- PROXIMY STATE
-- ============================================================
local ProxFolder = nil
local ProxSelectedItems = {}
local ProxItemCounts = {}
local ProxItemButtons = {}
local ProxInteracted = {}
local ProxModified = {}
local ProxTPHistory = {}
local ProxMaxHistory = 10
local ProxBusy = false
local ProxListExpanded = false
local proxSelectedLabel = nil
local proxListTitle = nil
local proxItemScroll = nil
local proxSearchBox = nil
local proxManualBox = nil

-- ============================================================
-- JUMP BUTTON MOBILE
-- ============================================================
local function SetJumpButtonVisible(visible)
    pcall(function()
        local playerGui = LocalPlayer:FindFirstChild("PlayerGui")
        if not playerGui then return end
        for _, obj in ipairs(playerGui:GetDescendants()) do
            if obj:IsA("GuiObject") then
                local name = string.lower(obj.Name)
                if name:find("jump") or name:find("pulo") then
                    obj.Visible = visible
                end
            end
        end
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
        tpStatusLabel.Text = "LOCAL MARCADO"
        tpStatusLabel.TextColor3 = THEME.ok
        tpCoordLabel.Text = string.format("X: %.1f | Y: %.1f | Z: %.1f", markedPosition.X, markedPosition.Y, markedPosition.Z)
    else
        tpStatusLabel.Text = "LOCAL NAO MARCADO"
        tpStatusLabel.TextColor3 = THEME.bad
        tpCoordLabel.Text = "Nenhuma posicao"
    end
end

-- ============================================================
-- TP MARKER
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
    tpMarker.Color = THEME.accent
    tpMarker.Transparency = 0.2
    tpMarker.Parent = workspace
end

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
-- TP RAYCAST
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
        if not worldPosition then ShowNotification("LOCAL INVALIDO", false); return end
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
-- TP TELEPORT
-- ============================================================
local function TeleportarSeguro(posicaoDestino)
    if typeof(posicaoDestino) ~= "Vector3" then ShowNotification("POSICAO INVALIDA", false); return false end
    if posicaoDestino ~= posicaoDestino then ShowNotification("POSICAO INVALIDA", false); return false end
    if math.abs(posicaoDestino.X) > 100000 or math.abs(posicaoDestino.Y) > 100000 or math.abs(posicaoDestino.Z) > 100000 then ShowNotification("POSICAO MUITO LONGE", false); return false end
    if TPState.IsExecuting then return false end

    local character = LocalPlayer.Character
    if not character then ShowNotification("SEM PERSONAGEM", false); return false end
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if not humanoid then ShowNotification("SEM HUMANOID", false); return false end
    if humanoid.Health <= 0 then ShowNotification("PERSONAGEM MORTO", false); return false end
    local root = character:FindFirstChild("HumanoidRootPart")
    if not root then ShowNotification("SEM ROOT PART", false); return false end

    TPState.IsExecuting = true

    local sucesso = false
    local tentativas = 0
    local maxTentativas = 5

    while not sucesso and tentativas < maxTentativas do
        tentativas = tentativas + 1
        local colisaoOriginal = root.CanCollide
        root.CanCollide = false
        local success = pcall(function() character:PivotTo(CFrame.new(posicaoDestino)) end)
        if success then
            task.wait(0.05)
            if (root.Position - posicaoDestino).Magnitude < 5 then sucesso = true; break end
        end
        if not sucesso then
            success = pcall(function() root.CFrame = CFrame.new(posicaoDestino) end)
            if success then
                task.wait(0.05)
                if (root.Position - posicaoDestino).Magnitude < 5 then sucesso = true; break end
            end
        end
        if not sucesso then task.wait(0.1) end
        root.CanCollide = colisaoOriginal
    end

    root.CanCollide = true

    if not sucesso then
        root.AssemblyLinearVelocity = Vector3.zero
        root.AssemblyAngularVelocity = Vector3.zero
        pcall(function() root.CFrame = CFrame.new(posicaoDestino) end)
        task.wait(0.1)
        if (root.Position - posicaoDestino).Magnitude < 10 then sucesso = true end
        if not sucesso then
            pcall(function() character:PivotTo(CFrame.new(posicaoDestino)) end)
            task.wait(0.1)
            if (root.Position - posicaoDestino).Magnitude < 10 then sucesso = true end
        end
    end

    TPState.IsExecuting = false

    if sucesso then
        task.wait(0.05)
        if (root.Position - posicaoDestino).Magnitude < 10 then
            ShowNotification("TELEPORTADO", true)
        else
            ShowNotification("TELEPORTE PARCIAL", false)
        end
        return true
    else
        ShowNotification("FALHA NO TELEPORTE", false)
        return false
    end
end

local function TeleportToMarkedPosition()
    if not markedPosition then ShowNotification("NENHUM LOCAL MARCADO", false); return false end
    return TeleportarSeguro(markedPosition)
end

-- ============================================================
-- TP INTERAÇÃO
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
-- PROXIMY CORE
-- ============================================================
local function ProxFindPart(obj)
    if not obj then return nil end
    if obj:IsA("BasePart") then return obj end
    for _, v in ipairs(obj:GetDescendants()) do
        if v:IsA("BasePart") then return v end
    end
    return nil
end

local function ProxValidPrompt(p)
    if not p.Enabled then return nil end
    local part = ProxFindPart(p.Parent)
    if not part then return nil end
    if not part:IsDescendantOf(workspace) then return nil end
    return part
end

local function ProxGetName(p)
    if p.ObjectText and p.ObjectText ~= "" then
        return p.ObjectText
    elseif p.ActionText and p.ActionText ~= "" then
        return p.ActionText
    elseif p.Parent then
        return p.Parent.Name
    end
    return "ProximityPrompt"
end

local function ProxIsAutoPickupPart(part)
    if not part or not part:IsA("BasePart") or not part.CanTouch then return false end
    return part:FindFirstChild("TouchInterest") ~= nil
        or part:FindFirstChild("TouchTransmitter") ~= nil
        or part:FindFirstChildOfClass("TouchTransmitter") ~= nil
end

local function ProxGetAutoPickupRoot(part)
    local model = part:FindFirstAncestorOfClass("Model")
    if model and model ~= LocalPlayer.Character then return model end
    return part
end

local function ProxGetAutoPickupName(root, part)
    local name = root and root.Name or (part and part.Name) or "Auto Pickup"
    return "AUTO PICKUP: " .. name
end

local function ProxMatchesSelectedName(itemName)
    if #ProxSelectedItems == 0 then return false end
    local lowerName = string.lower(tostring(itemName or ""))
    for _, selected in ipairs(ProxSelectedItems) do
        if lowerName == string.lower(tostring(selected)) then return true end
    end
    return false
end

local function ProxIsFilteredName(itemName)
    return ProxMatchesSelectedName(itemName)
end

local function ProxMatch(p)
    return ProxIsFilteredName(ProxGetName(p))
end

local function ProxInstant(p)
    local promptId = tostring(p:GetDebugId())
    if not ProxModified[promptId] then
        ProxModified[promptId] = {
            prompt = p,
            originalHoldDuration = p.HoldDuration,
            originalRequiresLineOfSight = p.RequiresLineOfSight,
            originalMaxActivationDistance = p.MaxActivationDistance
        }
    end
    p.HoldDuration = 0
    p.RequiresLineOfSight = false
    p.MaxActivationDistance = 999999
end

local function ProxRestoreAll()
    for promptId, data in pairs(ProxModified) do
        local p = data.prompt
        if p and p.Parent then
            pcall(function()
                p.HoldDuration = data.originalHoldDuration
                p.RequiresLineOfSight = data.originalRequiresLineOfSight
                p.MaxActivationDistance = data.originalMaxActivationDistance
            end)
        end
    end
    ProxModified = {}
end

local function ProxGetDistance(part)
    local char = LocalPlayer.Character
    if not char then return math.huge end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return math.huge end
    return (hrp.Position - part.Position).Magnitude
end

local function ProxGetAutoPickupTargets()
    local targets = {}
    local seen = {}
    for _, obj in ipairs(workspace:GetDescendants()) do
        if ProxIsAutoPickupPart(obj) then
            local character = LocalPlayer.Character
            if not character or not obj:IsDescendantOf(character) then
                local root = ProxGetAutoPickupRoot(obj)
                local key = tostring(root:GetDebugId())
                if not seen[key] then
                    seen[key] = true
                    table.insert(targets, {
                        part = obj,
                        name = ProxGetAutoPickupName(root, obj),
                        id = "autopickup_" .. key,
                        kind = "auto",
                        distance = ProxGetDistance(obj)
                    })
                end
            end
        end
    end
    return targets
end

local function ProxCollectAutoPickup(part)
    local character, root = GetCharacterRoot()
    if not character or not root or not part or not part.Parent then return false end
    if type(firetouchinterest) == "function" then
        local ok = pcall(function()
            firetouchinterest(root, part, 0)
            task.wait()
            firetouchinterest(root, part, 1)
        end)
        if ok then return true end
    end
    pcall(function() character:PivotTo(part.CFrame * CFrame.new(0, 2, 0)) end)
    return true
end

local function ProxInHistory(id)
    for _, h in ipairs(ProxTPHistory) do
        if h == id then return true end
    end
    return false
end

local function ProxAddHistory(id)
    table.insert(ProxTPHistory, 1, id)
    if #ProxTPHistory > ProxMaxHistory then
        table.remove(ProxTPHistory, #ProxTPHistory)
    end
end

local function ProxUpdateSelectedLabel()
    if not proxSelectedLabel then return end
    if #ProxSelectedItems == 0 then
        proxSelectedLabel.Text = "Nenhum item selecionado"
        proxSelectedLabel.TextColor3 = THEME.textSoft
    else
        proxSelectedLabel.Text = table.concat(ProxSelectedItems, " • ")
        proxSelectedLabel.TextColor3 = THEME.accent
    end
end

local function ProxTagMatches(itemName, tag)
    local rawName = string.lower(tostring(itemName or ""))
    local cleanTag = string.lower(tostring(tag or "")):gsub("^%s*(.-)%s*$", "%1")
    cleanTag = cleanTag:gsub("%s+", " ")
    if cleanTag == "" then return false end

    local isAutoPickup = rawName:find("^auto pickup:") ~= nil
    local nameWithoutAutoPrefix = rawName:gsub("^auto pickup:%s*", "")
    local compactTag = cleanTag:gsub("[%s_:%-]+", "")

    -- "pickup" e "auto pickup" sao tags globais dos objetos de toque.
    if compactTag == "pickup" or compactTag == "autopickup" then
        return isAutoPickup
    end

    return rawName:sub(1, #cleanTag) == cleanTag
        or nameWithoutAutoPrefix:sub(1, #cleanTag) == cleanTag
end

local function ProxSelectByTag(tag)
    local added = 0
    for itemName in pairs(ProxItemCounts) do
        if ProxTagMatches(itemName, tag) then
            local exists = false
            for _, selected in ipairs(ProxSelectedItems) do
                if selected == itemName then exists = true break end
            end
            if not exists then
                table.insert(ProxSelectedItems, itemName)
                added = added + 1
            end
        end
    end
    ProxUpdateSelectedLabel()
    return added
end

local function ProxToggleItem(itemName)
    local found = nil
    for i, s in ipairs(ProxSelectedItems) do
        if s == itemName then found = i break end
    end
    if found then
        table.remove(ProxSelectedItems, found)
    else
        table.insert(ProxSelectedItems, itemName)
    end
    ProxUpdateSelectedLabel()
    local data = ProxItemButtons[itemName]
    if data then
        data.button.BackgroundColor3 = found and THEME.card or THEME.accent
        data.nameLabel.TextColor3 = found and THEME.text or THEME.white
    end
    ShowNotification(found and ("REMOVIDO: " .. itemName) or ("SELECIONADO: " .. itemName), not found)
end

local function ProxCreateItemButton(itemName, count)
    local btn = Instance.new("TextButton", proxItemScroll)
    btn.Name = itemName
    btn.Size = UDim2.new(1, -6, 0, 26)
    btn.BackgroundColor3 = THEME.card
    btn.BorderSizePixel = 0
    btn.Text = ""
    btn.AutoButtonColor = false
    btn.ZIndex = 106

    local corner = Instance.new("UICorner", btn)
    corner.CornerRadius = UDim.new(0, 6)

    local nameLabel = Instance.new("TextLabel", btn)
    nameLabel.Size = UDim2.new(1, -50, 1, 0)
    nameLabel.Position = UDim2.new(0, 8, 0, 0)
    nameLabel.BackgroundTransparency = 1
    nameLabel.Text = itemName
    nameLabel.TextColor3 = THEME.text
    nameLabel.TextStrokeTransparency = 1
    nameLabel.Font = Enum.Font.GothamBold
    nameLabel.TextSize = 9
    nameLabel.TextXAlignment = Enum.TextXAlignment.Left
    nameLabel.TextTruncate = Enum.TextTruncate.AtEnd
    nameLabel.ZIndex = 107

    local countLabel = Instance.new("TextLabel", btn)
    countLabel.Size = UDim2.new(0, 36, 0, 16)
    countLabel.Position = UDim2.new(1, -42, 0.5, -8)
    countLabel.BackgroundColor3 = THEME.accent2
    countLabel.Text = tostring(count)
    countLabel.TextColor3 = THEME.white
    countLabel.TextStrokeTransparency = 1
    countLabel.Font = Enum.Font.GothamBold
    countLabel.TextSize = 9
    countLabel.ZIndex = 107
    Instance.new("UICorner", countLabel).CornerRadius = UDim.new(0, 5)

    ProxItemButtons[itemName] = { button = btn, nameLabel = nameLabel, countLabel = countLabel }

    for _, s in ipairs(ProxSelectedItems) do
        if s == itemName then
            btn.BackgroundColor3 = THEME.accent
            nameLabel.TextColor3 = THEME.white
            break
        end
    end

    local lastClick = 0
    btn.MouseButton1Click:Connect(function()
        local now = tick()
        if now - lastClick < 0.25 then return end
        lastClick = now
        ProxToggleItem(itemName)
    end)
end

local function ProxUpdateItemList(searchTerm)
    if not proxItemScroll then return end
    for _, child in ipairs(proxItemScroll:GetChildren()) do
        if child:IsA("TextButton") then child:Destroy() end
    end
    ProxItemButtons = {}

    local sorted = {}
    for name, count in pairs(ProxItemCounts) do
        table.insert(sorted, { name = name, count = count })
    end
    table.sort(sorted, function(a, b) return a.name < b.name end)

    local shown = 0
    for _, item in ipairs(sorted) do
        local pass = true
        if searchTerm and searchTerm ~= "" then
            pass = string.find(string.lower(item.name), string.lower(searchTerm), 1, true) ~= nil
        end
        if pass then
            ProxCreateItemButton(item.name, item.count)
            shown = shown + 1
        end
    end

    proxItemScroll.CanvasSize = UDim2.new(0, 0, 0, shown * 28)
    if proxListTitle then
        proxListTitle.Text = string.format("ITENS DETECTADOS (%d)", #sorted)
    end
end

-- TP ROTATIVO (evita últimos 10)
local function ProxTeleportRotativo()
    if #ProxSelectedItems == 0 then
        ShowNotification("SELECIONE UM ITEM", false)
        return
    end

    local targets = {}
    for _, p in ipairs(workspace:GetDescendants()) do
        if p:IsA("ProximityPrompt") then
            local part = ProxValidPrompt(p)
            if part and ProxMatch(p) then
                table.insert(targets, {
                    part = part,
                    distance = ProxGetDistance(part),
                    id = tostring(part:GetDebugId()),
                    name = ProxGetName(p)
                })
            end
        end
    end

    for _, autoTarget in ipairs(ProxGetAutoPickupTargets()) do
        if ProxMatchesSelectedName(autoTarget.name) then
            table.insert(targets, autoTarget)
        end
    end

    if #targets == 0 then
        ShowNotification("NENHUM ITEM NO MAPA", false)
        return
    end

    table.sort(targets, function(a, b) return a.distance < b.distance end)

    local target = nil
    for _, t in ipairs(targets) do
        if not ProxInHistory(t.id) then target = t break end
    end

    if not target then
        ProxTPHistory = {}
        target = targets[1]
        ShowNotification("HISTORICO RESETADO", true)
    end

    local char, root = GetCharacterRoot()
    if not root then ShowNotification("SEM PERSONAGEM", false); return end
    root.CFrame = target.part.CFrame * CFrame.new(0, 5, 0)
    ProxAddHistory(target.id)
    ShowNotification("TP: " .. tostring(target.name or "ITEM"), true)
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
    _G.JoystickData.DraggingLevel = math.floor((dist / joystickRadius) * 100)
    _G.JoystickData.Direction = Vector3.new(offset.X / joystickRadius, 0, offset.Y / joystickRadius)
end

local function animateJoystickPress()
    if freecamJoystickOuter then
        ts:Create(freecamJoystickOuter, TweenInfo.new(0.15, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Size = UDim2.fromOffset(sizeOuter * 1.1, sizeOuter * 1.1), BackgroundTransparency = 0.3}):Play()
    end
    if freecamJoystickInner then
        ts:Create(freecamJoystickInner, TweenInfo.new(0.15, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Size = UDim2.fromOffset(sizeInner * 1.2, sizeInner * 1.2), BackgroundColor3 = THEME.accent2}):Play()
    end
end

local function animateJoystickRelease()
    if freecamJoystickOuter then
        ts:Create(freecamJoystickOuter, TweenInfo.new(0.2, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Size = UDim2.fromOffset(sizeOuter, sizeOuter), BackgroundTransparency = 0.5}):Play()
    end
    if freecamJoystickInner then
        ts:Create(freecamJoystickInner, TweenInfo.new(0.2, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Size = UDim2.fromOffset(sizeInner, sizeInner), BackgroundColor3 = THEME.white}):Play()
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
    freecamJoystickOuter.BackgroundColor3 = THEME.card
    freecamJoystickOuter.BackgroundTransparency = 0.3
    freecamJoystickOuter.Parent = freecamJoystick
    freecamJoystickOuter.ZIndex = 551
    freecamJoystickOuter.Active = true
    Instance.new("UICorner", freecamJoystickOuter).CornerRadius = UDim.new(1, 0)
    local outerStroke = Instance.new("UIStroke", freecamJoystickOuter)
    outerStroke.Color = THEME.accent
    outerStroke.Transparency = 0.4
    outerStroke.Thickness = 2

    freecamJoystickInner = Instance.new("Frame")
    freecamJoystickInner.Size = UDim2.fromOffset(sizeInner, sizeInner)
    freecamJoystickInner.Position = UDim2.new(0.5, -sizeInner / 2, 0.5, -sizeInner / 2)
    freecamJoystickInner.BackgroundColor3 = THEME.accent2
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
            local moveDir = (camCF.LookVector * -dir.Z) + (camCF.RightVector * dir.X)
            moveDir += Vector3.new(0, dir.Y, 0)
            if moveDir.Magnitude > 0 then
                currentPos = currentPos + moveDir.Unit * moveSpeed * dt * 60
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
        shortcutFreecamButton.BackgroundColor3 = THEME.accent
        shortcutFreecamButton.BackgroundTransparency = 0.1
    else
        shortcutFreecamButton.BackgroundColor3 = THEME.card
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
    if shortcutFreecamGroup then shortcutFreecamGroup.Visible = ShortcutState.EditMode or getgenv().Config.ButtonFreecam == true end
end

local function EnterShortcutEditMode()
    if ShortcutState.EditMode then return end
    if not shortcutTPButton or not shortcutFreecamGroup then return end
    editStartTPPosition = shortcutTPButton.Position
    editStartFreecamPosition = shortcutFreecamGroup.Position
    ShortcutState.EditMode = true
    if shortcutEditBar then shortcutEditBar.Visible = true end
    UpdateShortcutVisibility()
end

local function CancelShortcutEdit()
    if shortcutTPButton and editStartTPPosition then shortcutTPButton.Position = editStartTPPosition end
    if shortcutFreecamGroup and editStartFreecamPosition then shortcutFreecamGroup.Position = editStartFreecamPosition end
    ShortcutState.EditMode = false
    if shortcutEditBar then shortcutEditBar.Visible = false end
    UpdateShortcutVisibility()
    ShowNotification("ALTERACOES CANCELADAS", false)
end

local function SaveShortcutEdit()
    if not shortcutTPButton or not shortcutFreecamGroup then return end
    ShortcutPositions.TP = shortcutTPButton.Position
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
            button.Position = UDim2.fromOffset(math.clamp(startPos.X.Offset + delta.X, 0, maxX), math.clamp(startPos.Y.Offset + delta.Y, 0, maxY))
        end
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then dragging = false end
    end)
end

-- ============================================================
-- NOTIFICATION SYSTEM
-- ============================================================
local NotifFrame, NotifLabel, NotifIcon
local NotifToken = 0

local function CreateNotification()
    if NotifFrame then return end
    NotifFrame = Instance.new("Frame")
    NotifFrame.Size = UDim2.new(0, 220, 0, 45)
    NotifFrame.AnchorPoint = Vector2.new(0.5, 0)
    NotifFrame.Position = UDim2.new(0.5, 0, 0, -60)
    NotifFrame.BackgroundColor3 = THEME.bgSoft
    NotifFrame.BackgroundTransparency = 0
    NotifFrame.BorderSizePixel = 0
    NotifFrame.Visible = false
    NotifFrame.ZIndex = 1000
    NotifFrame.Parent = ScreenGui

    Instance.new("UICorner", NotifFrame).CornerRadius = UDim.new(0, 10)
    local stroke = Instance.new("UIStroke", NotifFrame)
    stroke.Color = THEME.accent
    stroke.Thickness = 1.5
    stroke.Transparency = 0.2

    NotifIcon = Instance.new("TextLabel", NotifFrame)
    NotifIcon.Size = UDim2.new(0, 30, 1, 0)
    NotifIcon.Position = UDim2.new(0, 8, 0, 0)
    NotifIcon.BackgroundTransparency = 1
    NotifIcon.Text = ""
    NotifIcon.TextColor3 = THEME.accent
    NotifIcon.TextStrokeTransparency = 1
    NotifIcon.TextSize = 14
    NotifIcon.Font = Enum.Font.GothamBold
    NotifIcon.ZIndex = 1001

    NotifLabel = Instance.new("TextLabel", NotifFrame)
    NotifLabel.Size = UDim2.new(1, -45, 1, 0)
    NotifLabel.Position = UDim2.new(0, 40, 0, 0)
    NotifLabel.BackgroundTransparency = 1
    NotifLabel.Text = ""
    NotifLabel.TextColor3 = THEME.text
    NotifLabel.TextStrokeTransparency = 1
    NotifLabel.TextSize = 12
    NotifLabel.Font = Enum.Font.GothamMedium
    NotifLabel.TextXAlignment = Enum.TextXAlignment.Left
    NotifLabel.TextYAlignment = Enum.TextYAlignment.Center
    NotifLabel.TextWrapped = true
    NotifLabel.ZIndex = 1001
end

ShowNotification = function(text, enabled)
    CreateNotification()
    NotifToken = NotifToken + 1
    local token = NotifToken

    if enabled == false then
        NotifIcon.Text = "X"
        NotifIcon.TextColor3 = THEME.bad
    else
        NotifIcon.Text = "OK"
        NotifIcon.TextColor3 = THEME.ok
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
            if token == NotifToken then NotifFrame.Visible = false end
        end)
    end)
end

playPopSound = function()
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
    s.Color = color or THEME.stroke
    s.Thickness = thickness or 1
    s.Transparency = transparency or 0
    s.Parent = obj
    return s
end

local function CreateSectionLabel(parent, text)
    local container = Instance.new("Frame", parent)
    container.Size = UDim2.new(0.9, 0, 0, 14)
    container.BackgroundColor3 = THEME.card
    container.BorderSizePixel = 0
    CreateCorner(container, 4)

    local indicator = Instance.new("Frame", container)
    indicator.Size = UDim2.new(0, 2, 0, 10)
    indicator.Position = UDim2.new(0, 4, 0.5, -5)
    indicator.BackgroundColor3 = THEME.accent
    indicator.BorderSizePixel = 0
    CreateCorner(indicator, 1)

    local label = Instance.new("TextLabel", container)
    label.Size = UDim2.new(1, -10, 1, 0)
    label.Position = UDim2.new(0, 8, 0, 0)
    label.BackgroundTransparency = 1
    label.Text = text
    label.TextColor3 = THEME.text
    label.TextStrokeTransparency = 1
    label.TextSize = 8
    label.Font = Enum.Font.GothamBold
    label.TextXAlignment = Enum.TextXAlignment.Left

    return container
end

-- ============================================================
-- UI CREATION
-- ============================================================
for _, v in pairs(game.CoreGui:GetChildren()) do if v.Name == "PRIDE_HUB" then v:Destroy() end end
ScreenGui = Instance.new("ScreenGui", game.CoreGui)
ScreenGui.Name = "PRIDE_HUB"
ScreenGui.IgnoreGuiInset = true

ProxFolder = Instance.new("Folder", ScreenGui)
ProxFolder.Name = "PROX_ESP"

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
Main.BackgroundColor3 = THEME.bg
Main.BorderSizePixel = 0
Main.Active = true
Main.Draggable = true
Main.ClipsDescendants = true
CreateCorner(Main, 10)
CreateStroke(Main, THEME.accent, 2, 0.2)

local hd = Instance.new("Frame", Main)
hd.Size = UDim2.new(1, 0, 0, 36)
hd.BackgroundColor3 = THEME.accent
hd.BorderSizePixel = 0
CreateCorner(hd, 10)

local hdGrad = Instance.new("UIGradient", hd)
hdGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(170, 110, 245)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(140, 80, 225)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(170, 110, 245))
})
hdGrad.Rotation = 90

local tt = Instance.new("TextLabel", hd)
tt.Size = UDim2.new(1, -60, 0, 20)
tt.Position = UDim2.new(0, 12, 0, 3)
tt.BackgroundTransparency = 1
tt.Text = "PRIDE HUB"
tt.TextColor3 = THEME.white
tt.TextStrokeTransparency = 1
tt.TextSize = 12
tt.Font = Enum.Font.GothamBold
tt.TextXAlignment = Enum.TextXAlignment.Left

local st = Instance.new("TextLabel", hd)
st.Size = UDim2.new(1, -60, 0, 10)
st.Position = UDim2.new(0, 12, 0, 22)
st.BackgroundTransparency = 1
st.Text = "AIM | ESP | TP | ATALHO | PROX"
st.TextColor3 = Color3.fromRGB(235, 220, 255)
st.TextStrokeTransparency = 1
st.TextSize = 7
st.Font = Enum.Font.GothamMedium
st.TextXAlignment = Enum.TextXAlignment.Left

local btnMin = Instance.new("TextButton", hd)
btnMin.Size = UDim2.new(0, 20, 0, 20)
btnMin.Position = UDim2.new(1, -44, 0.5, -10)
btnMin.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
btnMin.BackgroundTransparency = 0.75
btnMin.Text = "-"
btnMin.TextColor3 = THEME.white
btnMin.TextStrokeTransparency = 1
btnMin.TextSize = 13
btnMin.Font = Enum.Font.GothamMedium
btnMin.AutoButtonColor = false
CreateCorner(btnMin, 6)

local CloseBtn = Instance.new("TextButton", hd)
CloseBtn.Size = UDim2.new(0, 20, 0, 20)
CloseBtn.Position = UDim2.new(1, -22, 0.5, -10)
CloseBtn.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
CloseBtn.BackgroundTransparency = 0.75
CloseBtn.Text = "x"
CloseBtn.TextColor3 = THEME.white
CloseBtn.TextStrokeTransparency = 1
CloseBtn.TextSize = 13
CloseBtn.Font = Enum.Font.GothamMedium
CloseBtn.AutoButtonColor = false
CreateCorner(CloseBtn, 6)

local Icon = Instance.new("ImageButton", ScreenGui)
Icon.Size = UDim2.new(0, 38, 0, 38)
Icon.Position = UDim2.new(0, 10, 0, 100)
Icon.Image = "rbxassetid://73630975144333"
Icon.ImageTransparency = 0.05
Icon.BackgroundColor3 = THEME.accent
Icon.BackgroundTransparency = 0.1
Icon.Visible = false
Icon.ZIndex = 10
CreateCorner(Icon, 10)
CreateStroke(Icon, THEME.accent, 2, 0.3)

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
    ProxRestoreAll()
    if ProxFolder then ProxFolder:ClearAllChildren() end
    Main.Visible = false
    Icon.Visible = false
end)

Main.Visible = false
Icon.Visible = true

local dragging = false
local dragStart = nil
local startPos = nil

local function MakeDraggable(obj)
    local d, ds, sp
    obj.InputBegan:Connect(function(input)
        if (input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch) and not IsDraggingSlider then
            d = true
            ds = input.Position
            sp = obj.Position
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if d then
            local viewport = Camera.ViewportSize
            local delta = input.Position - ds
            obj.Position = UDim2.new(0, math.clamp(sp.X.Offset + delta.X, 0, math.max(0, viewport.X - obj.AbsoluteSize.X)), 0, math.clamp(sp.Y.Offset + delta.Y, 0, math.max(0, viewport.Y - obj.AbsoluteSize.Y)))
        end
    end)
    UserInputService.InputEnded:Connect(function() d = false end)
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

-- ============================================================
-- BARRA DE ABAS (5 ABAS)
-- ============================================================
local barraAbas = Instance.new("Frame", Main)
barraAbas.Size = UDim2.new(1, 0, 0, 26)
barraAbas.Position = UDim2.new(0, 0, 0, 36)
barraAbas.BackgroundColor3 = THEME.bg
barraAbas.BorderSizePixel = 0

local function CriarAba(texto, posX, ordem)
    local aba = Instance.new("TextButton", barraAbas)
    aba.Size = UDim2.new(0.19, 0, 0, 22)
    aba.Position = UDim2.new(posX, 0, 0, 2)
    aba.BackgroundColor3 = THEME.card
    aba.Text = texto
    aba.TextColor3 = THEME.textSoft
    aba.TextStrokeTransparency = 1
    aba.TextSize = 7
    aba.Font = Enum.Font.GothamBold
    aba.AutoButtonColor = false
    aba.LayoutOrder = ordem
    CreateCorner(aba, 6)
    return aba
end

local abaAIM    = CriarAba("AIM",     0.005, 1)
local abaESP    = CriarAba("ESP",     0.205, 2)
local abaTP     = CriarAba("TP",      0.405, 3)
local abaAtalho = CriarAba("ATALHO",  0.605, 4)
local abaProx   = CriarAba("PROX",    0.805, 5)

local abaIndicator = Instance.new("Frame", barraAbas)
abaIndicator.Size = UDim2.new(0, 18, 0, 2)
abaIndicator.Position = UDim2.new(0.005, 0, 1, -1)
abaIndicator.BackgroundColor3 = THEME.accent
abaIndicator.BorderSizePixel = 0
CreateCorner(abaIndicator, 1)

local contentY = 62
local contentH = 228

local function CriarTela()
    local tela = Instance.new("ScrollingFrame", Main)
    tela.Size = UDim2.new(1, 0, 0, contentH)
    tela.Position = UDim2.new(0, 0, 0, contentY)
    tela.BackgroundColor3 = THEME.bg
    tela.BorderSizePixel = 0
    tela.ScrollBarThickness = 2
    tela.ScrollBarImageColor3 = THEME.accent
    tela.Visible = false
    tela.CanvasSize = UDim2.new(0, 0, 0, 0)
    tela.AutomaticCanvasSize = Enum.AutomaticSize.Y
    tela.ElasticBehavior = Enum.ElasticBehavior.Always
    local lay = Instance.new("UIListLayout", tela)
    lay.Padding = UDim.new(0, 4)
    lay.HorizontalAlignment = Enum.HorizontalAlignment.Center
    lay.SortOrder = Enum.SortOrder.LayoutOrder
    local pad = Instance.new("UIPadding", tela)
    pad.PaddingTop = UDim.new(0, 6)
    pad.PaddingBottom = UDim.new(0, 6)
    pad.PaddingLeft = UDim.new(0, 5)
    pad.PaddingRight = UDim.new(0, 5)
    return tela
end

local telaAIM = CriarTela()
local telaESP = CriarTela()
local telaTP = CriarTela()
local telaAtalho = CriarTela()
local telaProx = CriarTela()
telaAIM.Visible = true

-- ============================================================
-- COMPONENTES UI
-- ============================================================
local selectedColorBtn = nil
local ToggleRefreshers = {}

local function criarSeletorCores(parent, callback)
    local frame = Instance.new("Frame", parent)
    frame.Size = UDim2.new(0.9, 0, 0, 42)
    frame.BackgroundColor3 = THEME.card
    frame.LayoutOrder = 99
    frame.BorderSizePixel = 0
    CreateCorner(frame, 7)
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
        CreateStroke(bt, THEME.stroke, 1, 0.3)
        bt.MouseButton1Click:Connect(function()
            if selectedColorBtn and selectedColorBtn:FindFirstChild("UIStroke2") then
                selectedColorBtn:FindFirstChild("UIStroke2"):Destroy()
            end
            selectedColorBtn = bt
            local sel = Instance.new("UIStroke", bt)
            sel.Name = "UIStroke2"
            sel.Color = Color3.new(1, 1, 1)
            sel.Thickness = 2
            callback(c)
        end)
    end
    return frame
end

local sairAimInput = nil
local sairAimContadorLabel = nil

local function AddToggleVertical(name, prop, parent)
    local btn = Instance.new("TextButton", parent)
    btn.Size = UDim2.new(0.9, 0, 0, 32)
    btn.BackgroundColor3 = THEME.card
    btn.Text = ""
    btn.AutoButtonColor = false
    btn.BorderSizePixel = 0
    CreateCorner(btn, 7)

    local label = Instance.new("TextLabel", btn)
    label.Size = UDim2.new(0.55, 0, 1, 0)
    label.Position = UDim2.new(0, 9, 0, 0)
    label.BackgroundTransparency = 1
    label.Text = name
    label.TextColor3 = THEME.text
    label.TextStrokeTransparency = 1
    label.TextSize = 8
    label.Font = Enum.Font.GothamBold
    label.TextXAlignment = Enum.TextXAlignment.Left

    local toggleStroke = CreateStroke(btn, THEME.stroke, 1, 0.55)

    local toggleDot = Instance.new("Frame", btn)
    toggleDot.Size = UDim2.new(0, 30, 0, 16)
    toggleDot.Position = UDim2.new(1, -38, 0.5, -8)
    toggleDot.BackgroundColor3 = Color3.fromRGB(210, 200, 230)
    toggleDot.BorderSizePixel = 0
    CreateCorner(toggleDot, 8)

    local dot = Instance.new("Frame", toggleDot)
    dot.Size = UDim2.new(0, 12, 0, 12)
    dot.Position = UDim2.new(0, 2, 0.5, -6)
    dot.BackgroundColor3 = Color3.fromRGB(170, 155, 195)
    dot.BorderSizePixel = 0
    CreateCorner(dot, 6)

    local function updateToggle(showNotif)
        local on = getgenv().Config[prop]
        ts:Create(dot, TweenInfo.new(0.15, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Position = UDim2.new(0, on and 16 or 2, 0.5, -6), BackgroundColor3 = on and THEME.white or Color3.fromRGB(170, 155, 195)}):Play()
        ts:Create(toggleDot, TweenInfo.new(0.15), {BackgroundColor3 = on and THEME.accent or Color3.fromRGB(210, 200, 230)}):Play()
        ts:Create(btn, TweenInfo.new(0.15), {BackgroundColor3 = on and THEME.cardHover or THEME.card}):Play()
        ts:Create(toggleStroke, TweenInfo.new(0.15), {Transparency = on and 0.15 or 0.55}):Play()
        if showNotif then
            if prop == "ButtonTP" then
                ShowNotification(on and "ATALHO TP ATIVADO" or "ATALHO TP DESATIVADO", on)
            elseif prop == "TPInteracao" then
                ShowNotification(on and "TP INTERACAO ATIVADO" or "TP INTERACAO DESATIVADO", on)
            elseif prop == "ButtonFreecam" then
                ShowNotification(on and "BUTTON FREECAM ATIVO" or "BUTTON FREECAM DESATIVADO", on)
            else
                ShowNotification(name .. (on and " ATIVO" or " DESATIVADO"), on)
            end
        end
    end
    ToggleRefreshers[prop] = updateToggle
    btn.MouseEnter:Connect(function()
        ts:Create(btn, TweenInfo.new(0.12), {BackgroundColor3 = THEME.cardHover}):Play()
        ts:Create(toggleStroke, TweenInfo.new(0.12), {Transparency = 0.1}):Play()
    end)
    btn.MouseLeave:Connect(function()
        local on = getgenv().Config[prop]
        ts:Create(btn, TweenInfo.new(0.16), {BackgroundColor3 = on and THEME.cardHover or THEME.card}):Play()
        ts:Create(toggleStroke, TweenInfo.new(0.16), {Transparency = on and 0.15 or 0.55}):Play()
    end)
    updateToggle(false)
    btn.MouseButton1Click:Connect(function()
        getgenv().Config[prop] = not getgenv().Config[prop]
        if prop == "ProxHideFiltered" and getgenv().Config[prop] then
            getgenv().Config.ProxShowOnlyFiltered = false
            if ToggleRefreshers.ProxShowOnlyFiltered then ToggleRefreshers.ProxShowOnlyFiltered(false) end
        elseif prop == "ProxShowOnlyFiltered" and getgenv().Config[prop] then
            getgenv().Config.ProxHideFiltered = false
            if ToggleRefreshers.ProxHideFiltered then ToggleRefreshers.ProxHideFiltered(false) end
        end
        updateToggle(true)

        if prop == "SairAimEnabled" then
            if getgenv().Config[prop] then
                SairAim.Enabled = true
                SairAim.Count = 0
                SairAim.Processing = false
                AimPermitido = true
                if sairAimInput then sairAimInput.Visible = true end
                if sairAimContadorLabel then sairAimContadorLabel.Text = "SAIDAS: 0/" .. tostring(SairAim.Attempts); sairAimContadorLabel.Visible = true end
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
        if prop == "ButtonTP" or prop == "ButtonFreecam" then UpdateShortcutVisibility() end
        if prop == "ProxInstant" then
            if getgenv().Config.ProxInstant then
                for _, p in ipairs(workspace:GetDescendants()) do
                    if p:IsA("ProximityPrompt") then ProxInstant(p) end
                end
            else
                ProxRestoreAll()
            end
        end
        if prop == "ProxAutoFarm" then
            if getgenv().Config.ProxAutoFarm then ProxInteracted = {} end
        end
    end)
    return btn
end

local function AddSliderVertical(name, prop, parent, max, min, suffix)
    min = min or 0
    suffix = suffix or ""
    local frame = Instance.new("Frame", parent)
    frame.Size = UDim2.new(0.9, 0, 0, 35)
    frame.BackgroundColor3 = THEME.card
    frame.BorderSizePixel = 0
    CreateCorner(frame, 7)

    local label = Instance.new("TextLabel", frame)
    label.Size = UDim2.new(0.5, 0, 0, 15)
    label.Position = UDim2.new(0, 7, 0, 2)
    label.BackgroundTransparency = 1
    label.Text = name
    label.TextColor3 = THEME.text
    label.TextStrokeTransparency = 1
    label.TextSize = 7
    label.Font = Enum.Font.GothamBold
    label.TextXAlignment = Enum.TextXAlignment.Left

    local valueLabel = Instance.new("TextLabel", frame)
    valueLabel.Size = UDim2.new(0.4, 0, 0, 15)
    valueLabel.Position = UDim2.new(0.6, 0, 0, 2)
    valueLabel.BackgroundTransparency = 1
    valueLabel.Text = getgenv().Config[prop] .. suffix
    valueLabel.TextColor3 = THEME.accent
    valueLabel.TextStrokeTransparency = 1
    valueLabel.TextSize = 8
    valueLabel.Font = Enum.Font.GothamBold
    valueLabel.TextXAlignment = Enum.TextXAlignment.Right

    local barBg = Instance.new("Frame", frame)
    barBg.Size = UDim2.new(0.9, 0, 0, 6)
    barBg.Position = UDim2.new(0.05, 0, 0, 20)
    barBg.BackgroundColor3 = Color3.fromRGB(210, 200, 230)
    barBg.BorderSizePixel = 0
    CreateCorner(barBg, 3)

    local barFill = Instance.new("Frame", barBg)
    local percent = (getgenv().Config[prop] - min) / (max - min)
    barFill.Size = UDim2.new(percent, 0, 1, 0)
    barFill.BackgroundColor3 = THEME.accent
    barFill.BorderSizePixel = 0
    CreateCorner(barFill, 3)

    local dot = Instance.new("TextButton", barBg)
    dot.Size = UDim2.new(0, 14, 0, 14)
    dot.Position = UDim2.new(percent, -7, 0.5, -7)
    dot.BackgroundColor3 = Color3.new(1, 1, 1)
    dot.Text = ""
    dot.AutoButtonColor = false
    CreateCorner(dot, 7)
    CreateStroke(dot, THEME.accent, 2, 0)

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
            sliderConn = UserInputService.InputChanged:Connect(function(inp)
                if inp.UserInputType == Enum.UserInputType.MouseMovement or inp.UserInputType == Enum.UserInputType.Touch then updateSlider(inp) end
            end)
            updateSlider(input)
        end
    end)
    dot.MouseButton1Down:Connect(function()
        IsDraggingSlider = true
        sliderConn = UserInputService.InputChanged:Connect(function(inp)
            if inp.UserInputType == Enum.UserInputType.MouseMovement or inp.UserInputType == Enum.UserInputType.Touch then updateSlider(inp) end
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
local function ExecutarSaidaAim()
    if not SairAim.Enabled then return end
    if SairAim.Processing then return end
    if SairAim.Count >= SairAim.Attempts then return end
    if os.clock() < SairAim.CooldownUntil then return end
    SairAim.Processing = true
    SairAim.Count = SairAim.Count + 1
    SairAim.CooldownUntil = os.clock() + SairAimCooldown
    if sairAimContadorLabel then sairAimContadorLabel.Text = "SAIDAS: " .. tostring(SairAim.Count) .. "/" .. tostring(SairAim.Attempts) end
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
    if (os.clock() - SairAim.LossStart) >= SairAimLossConfirmTime then
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
PartBtn.BackgroundColor3 = THEME.card
PartBtn.Text = "ALVO: CABECA"
PartBtn.TextColor3 = THEME.text
PartBtn.TextStrokeTransparency = 1
PartBtn.TextSize = 8
PartBtn.Font = Enum.Font.GothamBold
PartBtn.AutoButtonColor = false
PartBtn.BorderSizePixel = 0
CreateCorner(PartBtn, 7)
PartBtn.MouseButton1Click:Connect(function()
    getgenv().Config.TargetPart = (getgenv().Config.TargetPart == "Head" and "HumanoidRootPart" or "Head")
    PartBtn.Text = "ALVO: " .. (getgenv().Config.TargetPart == "Head" and "CABECA" or "TRONCO")
    ShowNotification("ALVO: " .. (getgenv().Config.TargetPart == "Head" and "CABECA" or "TRONCO"), true)
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
sairAimInput.BackgroundColor3 = THEME.bgSoft
sairAimInput.TextColor3 = THEME.text
sairAimInput.PlaceholderColor3 = THEME.textSoft
sairAimInput.PlaceholderText = "QUANTAS TENTATIVAS"
sairAimInput.Text = "3"
sairAimInput.ClearTextOnFocus = false
sairAimInput.TextSize = 9
sairAimInput.Font = Enum.Font.GothamBold
sairAimInput.TextXAlignment = Enum.TextXAlignment.Center
sairAimInput.Visible = false
sairAimInput.LayoutOrder = 99
sairAimInput.BorderSizePixel = 0
CreateCorner(sairAimInput, 7)

sairAimInput.FocusLost:Connect(function()
    local numero = tonumber(sairAimInput.Text)
    if numero then
        numero = math.clamp(math.floor(numero), 1, 100)
        SairAim.Attempts = numero
        sairAimInput.Text = tostring(numero)
    else
        sairAimInput.Text = tostring(SairAim.Attempts)
    end
    if sairAimContadorLabel then sairAimContadorLabel.Text = "SAIDAS: " .. tostring(SairAim.Count) .. "/" .. tostring(SairAim.Attempts) end
end)

sairAimContadorLabel = Instance.new("TextLabel", telaAIM)
sairAimContadorLabel.Size = UDim2.new(0.9, 0, 0, 16)
sairAimContadorLabel.BackgroundTransparency = 1
sairAimContadorLabel.Text = "SAIDAS: 0/3"
sairAimContadorLabel.TextColor3 = THEME.accent
sairAimContadorLabel.TextStrokeTransparency = 1
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
tpStatusLabel.Text = "LOCAL NAO MARCADO"
tpStatusLabel.TextColor3 = THEME.bad
tpStatusLabel.TextStrokeTransparency = 1
tpStatusLabel.TextSize = 10
tpStatusLabel.Font = Enum.Font.GothamBold
tpStatusLabel.TextXAlignment = Enum.TextXAlignment.Center
tpStatusLabel.LayoutOrder = 1

tpCoordLabel = Instance.new("TextLabel", telaTP)
tpCoordLabel.Size = UDim2.new(0.9, 0, 0, 18)
tpCoordLabel.BackgroundTransparency = 1
tpCoordLabel.Text = "Nenhuma posicao"
tpCoordLabel.TextColor3 = THEME.accent
tpCoordLabel.TextStrokeTransparency = 1
tpCoordLabel.TextSize = 9
tpCoordLabel.Font = Enum.Font.GothamMedium
tpCoordLabel.TextXAlignment = Enum.TextXAlignment.Center
tpCoordLabel.LayoutOrder = 2

UpdateTPUI()

local marcarBtn = Instance.new("TextButton", telaTP)
marcarBtn.Size = UDim2.new(0.9, 0, 0, 34)
marcarBtn.BackgroundColor3 = THEME.cardHover
marcarBtn.Text = "MARCAR"
marcarBtn.TextColor3 = THEME.text
marcarBtn.TextStrokeTransparency = 1
marcarBtn.TextSize = 11
marcarBtn.Font = Enum.Font.GothamBold
marcarBtn.AutoButtonColor = false
marcarBtn.LayoutOrder = 3
marcarBtn.BorderSizePixel = 0
CreateCorner(marcarBtn, 7)

marcarBtn.MouseButton1Click:Connect(function()
    if tpOptionFrame then tpOptionFrame:Destroy(); tpOptionFrame = nil end
    tpOptionFrame = Instance.new("Frame", telaTP)
    tpOptionFrame.Size = UDim2.new(0.9, 0, 0, 34)
    tpOptionFrame.BackgroundTransparency = 1
    tpOptionFrame.LayoutOrder = 4

    local dedoBtn = Instance.new("TextButton", tpOptionFrame)
    dedoBtn.Size = UDim2.new(0.48, 0, 0, 30)
    dedoBtn.Position = UDim2.new(0, 0, 0, 2)
    dedoBtn.BackgroundColor3 = THEME.card
    dedoBtn.Text = "DEDO"
    dedoBtn.TextColor3 = THEME.text
    dedoBtn.TextStrokeTransparency = 1
    dedoBtn.TextSize = 10
    dedoBtn.Font = Enum.Font.GothamBold
    dedoBtn.AutoButtonColor = false
    dedoBtn.BorderSizePixel = 0
    CreateCorner(dedoBtn, 7)

    local agoraBtn = Instance.new("TextButton", tpOptionFrame)
    agoraBtn.Size = UDim2.new(0.48, 0, 0, 30)
    agoraBtn.Position = UDim2.new(0.52, 0, 0, 2)
    agoraBtn.BackgroundColor3 = THEME.card
    agoraBtn.Text = "AGORA"
    agoraBtn.TextColor3 = THEME.text
    agoraBtn.TextStrokeTransparency = 1
    agoraBtn.TextSize = 10
    agoraBtn.Font = Enum.Font.GothamBold
    agoraBtn.AutoButtonColor = false
    agoraBtn.BorderSizePixel = 0
    CreateCorner(agoraBtn, 7)

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

local tpBtn = Instance.new("TextButton", telaTP)
tpBtn.Size = UDim2.new(0.9, 0, 0, 34)
tpBtn.BackgroundColor3 = THEME.accent
tpBtn.Text = "TELEPORTAR"
tpBtn.TextColor3 = THEME.white
tpBtn.TextStrokeTransparency = 1
tpBtn.TextSize = 11
tpBtn.Font = Enum.Font.GothamBold
tpBtn.AutoButtonColor = false
tpBtn.LayoutOrder = 5
tpBtn.BorderSizePixel = 0
CreateCorner(tpBtn, 7)
tpBtn.MouseButton1Click:Connect(function() TeleportToMarkedPosition() end)

local btnForcar = Instance.new("TextButton", telaTP)
btnForcar.Size = UDim2.new(0.9, 0, 0, 28)
btnForcar.BackgroundColor3 = THEME.card
btnForcar.Text = "FORCAR TP"
btnForcar.TextColor3 = THEME.warn
btnForcar.TextStrokeTransparency = 1
btnForcar.TextSize = 9
btnForcar.Font = Enum.Font.GothamBold
btnForcar.AutoButtonColor = false
btnForcar.LayoutOrder = 6
btnForcar.BorderSizePixel = 0
CreateCorner(btnForcar, 7)
btnForcar.MouseButton1Click:Connect(function()
    if not markedPosition then ShowNotification("NENHUM LOCAL MARCADO", false); return end
    ShowNotification("FORCANDO TP", true)
    local character = LocalPlayer.Character
    if not character then ShowNotification("SEM PERSONAGEM", false); return end
    local root = character:FindFirstChild("HumanoidRootPart")
    if not root then ShowNotification("SEM ROOT", false); return end
    pcall(function()
        root.AssemblyLinearVelocity = Vector3.zero
        root.AssemblyAngularVelocity = Vector3.zero
        character:PivotTo(CFrame.new(markedPosition))
    end)
    task.wait(0.05)
    pcall(function() root.CFrame = CFrame.new(markedPosition) end)
    task.wait(0.05)
    if (root.Position - markedPosition).Magnitude < 10 then
        ShowNotification("FORCADO COM SUCESSO", true)
    else
        ShowNotification("FALHA NO FORCADO", false)
    end
end)

-- ============================================================
-- ABA PROX (PROXIMY)
-- ============================================================
CreateSectionLabel(telaProx, "PROXIMY")
AddToggleVertical("ESP DE ITENS", "ProxESP", telaProx)
AddToggleVertical("INTERACAO INSTANTANEA", "ProxInstant", telaProx)
AddToggleVertical("AUTO COLETAR", "ProxAutoFarm", telaProx)
AddToggleVertical("ESCONDER FILTRADOS", "ProxHideFiltered", telaProx)
AddToggleVertical("MOSTRAR SO FILTRADOS", "ProxShowOnlyFiltered", telaProx)

-- BOTÃO TP ROTATIVO
local proxTpBtn = Instance.new("TextButton", telaProx)
proxTpBtn.Size = UDim2.new(0.9, 0, 0, 30)
proxTpBtn.BackgroundColor3 = THEME.accent
proxTpBtn.Text = "TP PARA ITEM (ROTATIVO)"
proxTpBtn.TextColor3 = THEME.white
proxTpBtn.TextStrokeTransparency = 1
proxTpBtn.TextSize = 9
proxTpBtn.Font = Enum.Font.GothamBold
proxTpBtn.AutoButtonColor = false
proxTpBtn.BorderSizePixel = 0
CreateCorner(proxTpBtn, 7)
proxTpBtn.MouseButton1Click:Connect(function()
    ProxTeleportRotativo()
    playPopSound()
end)

-- ADICIONAR MANUAL
CreateSectionLabel(telaProx, "SELECIONAR ITENS")

local proxAddContainer = Instance.new("Frame", telaProx)
proxAddContainer.Size = UDim2.new(0.9, 0, 0, 28)
proxAddContainer.BackgroundTransparency = 1

proxManualBox = Instance.new("TextBox", proxAddContainer)
proxManualBox.Size = UDim2.new(1, -62, 1, 0)
proxManualBox.BackgroundColor3 = THEME.bgSoft
proxManualBox.PlaceholderText = "Item ou tag (ex: pickup)..."
proxManualBox.PlaceholderColor3 = THEME.textSoft
proxManualBox.Text = ""
proxManualBox.TextColor3 = THEME.text
proxManualBox.Font = Enum.Font.GothamBold
proxManualBox.TextSize = 9
proxManualBox.BorderSizePixel = 0
proxManualBox.ClearTextOnFocus = false
CreateCorner(proxManualBox, 6)

local proxAddBtn = Instance.new("TextButton", proxAddContainer)
proxAddBtn.Size = UDim2.new(0, 56, 1, 0)
proxAddBtn.Position = UDim2.new(1, -56, 0, 0)
proxAddBtn.BackgroundColor3 = THEME.ok
proxAddBtn.Text = "+ ADD"
proxAddBtn.TextColor3 = THEME.white
proxAddBtn.TextStrokeTransparency = 1
proxAddBtn.Font = Enum.Font.GothamBold
proxAddBtn.TextSize = 9
proxAddBtn.AutoButtonColor = false
proxAddBtn.BorderSizePixel = 0
CreateCorner(proxAddBtn, 6)

proxAddBtn.MouseButton1Click:Connect(function()
    local texto = proxManualBox.Text:gsub("^%s*(.-)%s*$", "%1")
    if texto == "" then return end
    for _, s in ipairs(ProxSelectedItems) do
        if string.lower(s) == string.lower(texto) then
            ShowNotification("JA ESTA NA LISTA", false)
            proxManualBox.Text = ""
            return
        end
    end
    local tagged = ProxSelectByTag(texto)
    if tagged > 0 then
        ShowNotification("TAG MARCADA: " .. tagged, true)
        proxManualBox.Text = ""
        return
    end
    table.insert(ProxSelectedItems, texto)
    ProxUpdateSelectedLabel()
    ShowNotification("ADICIONADO: " .. texto, true)
    proxManualBox.Text = ""
    local data = ProxItemButtons[texto]
    if data then
        data.button.BackgroundColor3 = THEME.accent
        data.nameLabel.TextColor3 = THEME.white
    end
end)

local proxTagBtn = Instance.new("TextButton", telaProx)
proxTagBtn.Size = UDim2.new(0.9, 0, 0, 26)
proxTagBtn.BackgroundColor3 = THEME.card
proxTagBtn.Text = "MARCAR TAG / PREFIXO"
proxTagBtn.TextColor3 = THEME.accent
proxTagBtn.TextStrokeTransparency = 1
proxTagBtn.Font = Enum.Font.GothamBold
proxTagBtn.TextSize = 8
proxTagBtn.AutoButtonColor = false
proxTagBtn.BorderSizePixel = 0
CreateCorner(proxTagBtn, 6)
local proxTagStroke = CreateStroke(proxTagBtn, THEME.stroke, 1, 0.55)
proxTagBtn.MouseEnter:Connect(function()
    ts:Create(proxTagBtn, TweenInfo.new(0.12), {BackgroundColor3 = THEME.cardHover}):Play()
    ts:Create(proxTagStroke, TweenInfo.new(0.12), {Transparency = 0.1}):Play()
end)
proxTagBtn.MouseLeave:Connect(function()
    ts:Create(proxTagBtn, TweenInfo.new(0.16), {BackgroundColor3 = THEME.card}):Play()
    ts:Create(proxTagStroke, TweenInfo.new(0.16), {Transparency = 0.55}):Play()
end)
proxTagBtn.MouseButton1Click:Connect(function()
    local tag = proxManualBox.Text:gsub("^%s*(.-)%s*$", "%1")
    if tag == "" then return end
    local added = ProxSelectByTag(tag)
    ShowNotification(added > 0 and ("TAG MARCADA: " .. added) or "NENHUM ITEM COM ESSA TAG", added > 0)
    proxManualBox.Text = ""
end)

-- LIMPAR
local proxClearBtn = Instance.new("TextButton", telaProx)
proxClearBtn.Size = UDim2.new(0.9, 0, 0, 24)
proxClearBtn.BackgroundColor3 = THEME.card
proxClearBtn.Text = "LIMPAR SELECIONADOS"
proxClearBtn.TextColor3 = THEME.bad
proxClearBtn.TextStrokeTransparency = 1
proxClearBtn.Font = Enum.Font.GothamBold
proxClearBtn.TextSize = 8
proxClearBtn.AutoButtonColor = false
proxClearBtn.BorderSizePixel = 0
CreateCorner(proxClearBtn, 6)
proxClearBtn.MouseButton1Click:Connect(function()
    ProxSelectedItems = {}
    ProxTPHistory = {}
    ProxUpdateSelectedLabel()
    for _, data in pairs(ProxItemButtons) do
        data.button.BackgroundColor3 = THEME.card
        data.nameLabel.TextColor3 = THEME.text
    end
    ShowNotification("LISTA LIMPA", false)
end)

-- LABEL SELECIONADOS
proxSelectedLabel = Instance.new("TextLabel", telaProx)
proxSelectedLabel.Size = UDim2.new(0.9, 0, 0, 24)
proxSelectedLabel.BackgroundColor3 = THEME.bgSoft
proxSelectedLabel.Text = "Nenhum item selecionado"
proxSelectedLabel.TextColor3 = THEME.textSoft
proxSelectedLabel.TextStrokeTransparency = 1
proxSelectedLabel.Font = Enum.Font.GothamBold
proxSelectedLabel.TextSize = 8
proxSelectedLabel.TextWrapped = true
proxSelectedLabel.BorderSizePixel = 0
CreateCorner(proxSelectedLabel, 6)

-- LISTA DE ITENS (EXPANSÍVEL)
local proxListSection = Instance.new("Frame", telaProx)
proxListSection.Size = UDim2.new(0.9, 0, 0, 30)
proxListSection.BackgroundColor3 = THEME.card
proxListSection.BorderSizePixel = 0
proxListSection.ClipsDescendants = true
CreateCorner(proxListSection, 7)

local proxListHeader = Instance.new("TextButton", proxListSection)
proxListHeader.Size = UDim2.new(1, 0, 0, 30)
proxListHeader.BackgroundTransparency = 1
proxListHeader.Text = ""

proxListTitle = Instance.new("TextLabel", proxListHeader)
proxListTitle.Size = UDim2.new(1, -36, 1, 0)
proxListTitle.Position = UDim2.new(0, 10, 0, 0)
proxListTitle.BackgroundTransparency = 1
proxListTitle.Text = "ITENS DETECTADOS (0)"
proxListTitle.TextColor3 = THEME.text
proxListTitle.TextStrokeTransparency = 1
proxListTitle.Font = Enum.Font.GothamBold
proxListTitle.TextSize = 9
proxListTitle.TextXAlignment = Enum.TextXAlignment.Left

local proxArrow = Instance.new("TextLabel", proxListHeader)
proxArrow.Size = UDim2.new(0, 26, 0, 26)
proxArrow.Position = UDim2.new(1, -30, 0.5, -13)
proxArrow.BackgroundTransparency = 1
proxArrow.Text = "v"
proxArrow.TextColor3 = THEME.accent
proxArrow.TextStrokeTransparency = 1
proxArrow.Font = Enum.Font.GothamBold
proxArrow.TextSize = 12

local proxListBody = Instance.new("Frame", proxListSection)
proxListBody.Size = UDim2.new(1, 0, 0, 170)
proxListBody.Position = UDim2.new(0, 0, 0, 32)
proxListBody.BackgroundTransparency = 1
proxListBody.Visible = false

proxSearchBox = Instance.new("TextBox", proxListBody)
proxSearchBox.Size = UDim2.new(1, -12, 0, 24)
proxSearchBox.Position = UDim2.new(0, 6, 0, 2)
proxSearchBox.BackgroundColor3 = THEME.bgSoft
proxSearchBox.PlaceholderText = "Buscar item..."
proxSearchBox.PlaceholderColor3 = THEME.textSoft
proxSearchBox.Text = ""
proxSearchBox.TextColor3 = THEME.text
proxSearchBox.Font = Enum.Font.GothamBold
proxSearchBox.TextSize = 9
proxSearchBox.BorderSizePixel = 0
proxSearchBox.ClearTextOnFocus = false
CreateCorner(proxSearchBox, 6)

proxItemScroll = Instance.new("ScrollingFrame", proxListBody)
proxItemScroll.Size = UDim2.new(1, -12, 0, 136)
proxItemScroll.Position = UDim2.new(0, 6, 0, 30)
proxItemScroll.CanvasSize = UDim2.new(0, 0, 0, 0)
proxItemScroll.ScrollBarThickness = 2
proxItemScroll.ScrollBarImageColor3 = THEME.accent
proxItemScroll.BackgroundColor3 = THEME.bgSoft
proxItemScroll.BorderSizePixel = 0
CreateCorner(proxItemScroll, 6)
local proxItemLayout = Instance.new("UIListLayout", proxItemScroll)
proxItemLayout.Padding = UDim.new(0, 2)
proxItemLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center

proxListHeader.MouseButton1Click:Connect(function()
    ProxListExpanded = not ProxListExpanded
    proxListBody.Visible = ProxListExpanded
    proxArrow.Text = ProxListExpanded and "^" or "v"
    ts:Create(proxListSection, TweenInfo.new(0.2, Enum.EasingStyle.Quad), {
        Size = UDim2.new(0.9, 0, 0, ProxListExpanded and 204 or 30)
    }):Play()
end)

proxSearchBox:GetPropertyChangedSignal("Text"):Connect(function()
    ProxUpdateItemList(proxSearchBox.Text)
end)

-- ============================================================
-- ABA ATALHO
-- ============================================================
CreateSectionLabel(telaAtalho, "ATALHO")
AddToggleVertical("Button TP", "ButtonTP", telaAtalho)
AddToggleVertical("TP INTERACAO", "TPInteracao", telaAtalho)

local tpDelayContainer = Instance.new("Frame", telaAtalho)
tpDelayContainer.Size = UDim2.new(0.9, 0, 0, 32)
tpDelayContainer.BackgroundColor3 = THEME.card
tpDelayContainer.BorderSizePixel = 0
CreateCorner(tpDelayContainer, 7)

local tpDelayLabel = Instance.new("TextLabel", tpDelayContainer)
tpDelayLabel.Size = UDim2.new(0.4, 0, 1, 0)
tpDelayLabel.Position = UDim2.new(0, 9, 0, 0)
tpDelayLabel.BackgroundTransparency = 1
tpDelayLabel.Text = "DELAY:"
tpDelayLabel.TextColor3 = THEME.text
tpDelayLabel.TextStrokeTransparency = 1
tpDelayLabel.TextSize = 8
tpDelayLabel.Font = Enum.Font.GothamBold
tpDelayLabel.TextXAlignment = Enum.TextXAlignment.Left

local tpDelayBox = Instance.new("TextBox", tpDelayContainer)
tpDelayBox.Size = UDim2.new(0, 80, 0, 24)
tpDelayBox.Position = UDim2.new(0.5, 0, 0.5, -12)
tpDelayBox.BackgroundColor3 = THEME.bgSoft
tpDelayBox.TextColor3 = THEME.text
tpDelayBox.PlaceholderColor3 = THEME.textSoft
tpDelayBox.PlaceholderText = "0.50"
tpDelayBox.Text = tostring(getgenv().Config.TPInteracaoDelay)
tpDelayBox.ClearTextOnFocus = false
tpDelayBox.TextSize = 11
tpDelayBox.Font = Enum.Font.GothamBold
tpDelayBox.TextXAlignment = Enum.TextXAlignment.Center
tpDelayBox.BorderSizePixel = 0
CreateCorner(tpDelayBox, 6)

tpDelayBox.FocusLost:Connect(function()
    local value = tonumber(tpDelayBox.Text) or 0.50
    value = math.max(0, value)
    getgenv().Config.TPInteracaoDelay = value
    tpDelayBox.Text = tostring(value)
    ShowNotification("DELAY: " .. tostring(value) .. "s", true)
end)

AddToggleVertical("Button FREECAM", "ButtonFreecam", telaAtalho)

shortcutConfigButton = Instance.new("TextButton", telaAtalho)
shortcutConfigButton.Size = UDim2.new(0.5, 0, 0, 36)
shortcutConfigButton.BackgroundColor3 = THEME.accent
shortcutConfigButton.Text = "EDITAR"
shortcutConfigButton.TextColor3 = THEME.white
shortcutConfigButton.TextStrokeTransparency = 1
shortcutConfigButton.TextSize = 10
shortcutConfigButton.Font = Enum.Font.GothamBold
shortcutConfigButton.AutoButtonColor = false
shortcutConfigButton.LayoutOrder = 10
shortcutConfigButton.BorderSizePixel = 0
CreateCorner(shortcutConfigButton, 7)
shortcutConfigButton.MouseButton1Click:Connect(function() EnterShortcutEditMode() end)

-- ============================================================
-- BOTÕES FLUTUANTES
-- ============================================================
shortcutTPButton = Instance.new("TextButton", ScreenGui)
shortcutTPButton.Size = UDim2.new(0, 50, 0, 50)
shortcutTPButton.Position = ShortcutPositions.TP
shortcutTPButton.Text = "TP"
shortcutTPButton.TextSize = 12
shortcutTPButton.BackgroundColor3 = THEME.accent
shortcutTPButton.TextColor3 = THEME.white
shortcutTPButton.TextStrokeTransparency = 1
shortcutTPButton.Font = Enum.Font.GothamBold
shortcutTPButton.AutoButtonColor = false
shortcutTPButton.Visible = false
shortcutTPButton.ZIndex = 500
CreateCorner(shortcutTPButton, 25)

shortcutFreecamGroup = Instance.new("Frame", ScreenGui)
shortcutFreecamGroup.Size = UDim2.new(0, 170, 0, 50)
shortcutFreecamGroup.Position = ShortcutPositions.FREECAM
shortcutFreecamGroup.BackgroundTransparency = 1
shortcutFreecamGroup.Visible = false
shortcutFreecamGroup.ZIndex = 500

shortcutFreecamButton = Instance.new("TextButton", shortcutFreecamGroup)
shortcutFreecamButton.Size = UDim2.new(0, 70, 0, 50)
shortcutFreecamButton.Text = "FREECAM"
shortcutFreecamButton.TextSize = 9
shortcutFreecamButton.BackgroundColor3 = THEME.accent
shortcutFreecamButton.TextColor3 = THEME.white
shortcutFreecamButton.TextStrokeTransparency = 1
shortcutFreecamButton.Font = Enum.Font.GothamBold
shortcutFreecamButton.AutoButtonColor = false
CreateCorner(shortcutFreecamButton, 25)

freecamMinusButton = Instance.new("TextButton", shortcutFreecamGroup)
freecamMinusButton.Size = UDim2.new(0, 30, 0, 50)
freecamMinusButton.Position = UDim2.new(0, 75, 0, 0)
freecamMinusButton.BackgroundColor3 = THEME.card
freecamMinusButton.Text = "-"
freecamMinusButton.TextColor3 = THEME.text
freecamMinusButton.TextStrokeTransparency = 1
freecamMinusButton.TextSize = 16
freecamMinusButton.Font = Enum.Font.GothamBold
freecamMinusButton.AutoButtonColor = false
freecamMinusButton.Visible = false
CreateCorner(freecamMinusButton, 25)

freecamSpeedLabel = Instance.new("TextLabel", shortcutFreecamGroup)
freecamSpeedLabel.Size = UDim2.new(0, 40, 0, 50)
freecamSpeedLabel.Position = UDim2.new(0, 105, 0, 0)
freecamSpeedLabel.BackgroundTransparency = 1
freecamSpeedLabel.Text = "1.0x"
freecamSpeedLabel.TextColor3 = THEME.text
freecamSpeedLabel.TextStrokeTransparency = 1
freecamSpeedLabel.TextSize = 10
freecamSpeedLabel.Font = Enum.Font.GothamBold
freecamSpeedLabel.TextXAlignment = Enum.TextXAlignment.Center
freecamSpeedLabel.Visible = false

freecamPlusButton = Instance.new("TextButton", shortcutFreecamGroup)
freecamPlusButton.Size = UDim2.new(0, 30, 0, 50)
freecamPlusButton.Position = UDim2.new(0, 140, 0, 0)
freecamPlusButton.BackgroundColor3 = THEME.card
freecamPlusButton.Text = "+"
freecamPlusButton.TextColor3 = THEME.text
freecamPlusButton.TextStrokeTransparency = 1
freecamPlusButton.TextSize = 16
freecamPlusButton.Font = Enum.Font.GothamBold
freecamPlusButton.AutoButtonColor = false
freecamPlusButton.Visible = false
CreateCorner(freecamPlusButton, 25)

shortcutEditBar = Instance.new("Frame", ScreenGui)
shortcutEditBar.Size = UDim2.new(0, 200, 0, 40)
shortcutEditBar.Position = UDim2.new(0.5, -100, 0.02, 0)
shortcutEditBar.BackgroundColor3 = THEME.bgSoft
shortcutEditBar.BorderSizePixel = 0
shortcutEditBar.Visible = false
shortcutEditBar.ZIndex = 600
CreateCorner(shortcutEditBar, 10)
CreateStroke(shortcutEditBar, THEME.accent, 1.5, 0.3)

local cancelEditBtn = Instance.new("TextButton", shortcutEditBar)
cancelEditBtn.Size = UDim2.new(0.45, 0, 0, 30)
cancelEditBtn.Position = UDim2.new(0.03, 0, 0.5, -15)
cancelEditBtn.BackgroundColor3 = THEME.card
cancelEditBtn.Text = "Cancelar"
cancelEditBtn.TextColor3 = THEME.text
cancelEditBtn.TextStrokeTransparency = 1
cancelEditBtn.TextSize = 10
cancelEditBtn.Font = Enum.Font.GothamBold
cancelEditBtn.AutoButtonColor = false
cancelEditBtn.ZIndex = 601
cancelEditBtn.BorderSizePixel = 0
CreateCorner(cancelEditBtn, 7)

local saveEditBtn = Instance.new("TextButton", shortcutEditBar)
saveEditBtn.Size = UDim2.new(0.45, 0, 0, 30)
saveEditBtn.Position = UDim2.new(0.52, 0, 0.5, -15)
saveEditBtn.BackgroundColor3 = THEME.accent
saveEditBtn.Text = "Salvar"
saveEditBtn.TextColor3 = THEME.white
saveEditBtn.TextStrokeTransparency = 1
saveEditBtn.TextSize = 10
saveEditBtn.Font = Enum.Font.GothamBold
saveEditBtn.AutoButtonColor = false
saveEditBtn.ZIndex = 601
saveEditBtn.BorderSizePixel = 0
CreateCorner(saveEditBtn, 7)

cancelEditBtn.MouseButton1Click:Connect(function() CancelShortcutEdit() end)
saveEditBtn.MouseButton1Click:Connect(function() SaveShortcutEdit() end)

MakeShortcutDraggable(shortcutTPButton)
MakeShortcutDraggable(shortcutFreecamGroup)
UpdateShortcutVisibility()

shortcutTPButton.MouseButton1Click:Connect(function()
    if not ShortcutState.EditMode then TeleportToMarkedPosition() end
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
local abas = {
    { btn = abaAIM,     tela = telaAIM,     pos = 0.005, key = "aim" },
    { btn = abaESP,     tela = telaESP,     pos = 0.205, key = "esp" },
    { btn = abaTP,      tela = telaTP,      pos = 0.405, key = "tp" },
    { btn = abaAtalho,  tela = telaAtalho,  pos = 0.605, key = "atalho" },
    { btn = abaProx,    tela = telaProx,    pos = 0.805, key = "prox" },
}

local function selecionarAba(key)
    for _, aba in ipairs(abas) do
        local ativa = (aba.key == key)
        aba.tela.Visible = ativa
        ts:Create(aba.btn, TweenInfo.new(0.15), {
            BackgroundColor3 = ativa and THEME.accent or THEME.card,
            TextColor3 = ativa and THEME.white or THEME.textSoft
        }):Play()
        if ativa then
            ts:Create(abaIndicator, TweenInfo.new(0.15), {Position = UDim2.new(aba.pos, 0, 1, -1)}):Play()
        end
    end
    if key ~= "tp" then
        CancelDedoSelection()
        if tpOptionFrame then tpOptionFrame:Destroy(); tpOptionFrame = nil end
    end
end

abaAIM.MouseButton1Click:Connect(function() selecionarAba("aim") end)
abaESP.MouseButton1Click:Connect(function() selecionarAba("esp") end)
abaTP.MouseButton1Click:Connect(function() selecionarAba("tp") end)
abaAtalho.MouseButton1Click:Connect(function() selecionarAba("atalho") end)
abaProx.MouseButton1Click:Connect(function() selecionarAba("prox") end)
selecionarAba("aim")

-- ============================================================
-- RESPAWN HANDLER
-- ============================================================
LocalPlayer.CharacterAdded:Connect(function()
    if FreecamEnabled then
        task.wait(0.5)
        if FreecamEnabled then
            Camera.CameraType = Enum.CameraType.Custom
            Camera.CameraSubject = movePart
        end
    end
end)

-- ============================================================
-- ESP SYSTEM (PLAYERS)
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
    label.TextStrokeTransparency = 1
    label.RichText = true
    local hb = Instance.new("Frame", ScreenGui)
    hb.BackgroundColor3 = Color3.new(0, 0, 0)
    hb.BorderSizePixel = 0
    local hm = Instance.new("Frame", hb)
    hm.BackgroundColor3 = Color3.new(0, 1, 0)
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
-- PROXIMY LOOPS
-- ============================================================
-- ESP + contagem de itens
task.spawn(function()
    while true do
        task.wait(0.8)
        if not ScreenGui or not ScreenGui.Parent then break end

        ProxFolder:ClearAllChildren()
        ProxItemCounts = {}

        for _, p in ipairs(workspace:GetDescendants()) do
            if p:IsA("ProximityPrompt") then
                local part = ProxValidPrompt(p)
                if part then
                    if getgenv().Config.ProxInstant then ProxInstant(p) end

                    local itemName = ProxGetName(p)
                    ProxItemCounts[itemName] = (ProxItemCounts[itemName] or 0) + 1

                    local m = ProxMatch(p)
                    local show = true
                    if getgenv().Config.ProxHideFiltered and m then show = false end
                    if getgenv().Config.ProxShowOnlyFiltered and not m then show = false end

                    if getgenv().Config.ProxESP and show then
                        local b = Instance.new("BillboardGui", ProxFolder)
                        b.Adornee = part
                        b.Size = UDim2.new(0, 120, 0, 30)
                        b.AlwaysOnTop = true
                        b.StudsOffset = Vector3.new(0, 2, 0)
                        local txt = Instance.new("TextLabel", b)
                        txt.Size = UDim2.new(1, 0, 1, 0)
                        txt.BackgroundTransparency = 1
                        txt.TextScaled = true
                        txt.Font = Enum.Font.GothamBold
                        txt.TextStrokeTransparency = 1
                        txt.Text = itemName
                        txt.TextColor3 = m and Color3.fromRGB(255, 200, 60) or Color3.fromRGB(120, 220, 150)
                    end
                end
            end
        end

        for _, autoTarget in ipairs(ProxGetAutoPickupTargets()) do
            ProxItemCounts[autoTarget.name] = (ProxItemCounts[autoTarget.name] or 0) + 1
            local filtered = ProxIsFilteredName(autoTarget.name)
            local showAuto = true
            if getgenv().Config.ProxHideFiltered and filtered then showAuto = false end
            if getgenv().Config.ProxShowOnlyFiltered and not filtered then showAuto = false end
            if getgenv().Config.ProxESP and showAuto then
                local b = Instance.new("BillboardGui", ProxFolder)
                b.Name = "AutoPickupESP"
                b.Adornee = autoTarget.part
                b.Size = UDim2.new(0, 150, 0, 30)
                b.AlwaysOnTop = true
                b.StudsOffset = Vector3.new(0, 2.5, 0)
                local txt = Instance.new("TextLabel", b)
                txt.Size = UDim2.new(1, 0, 1, 0)
                txt.BackgroundTransparency = 1
                txt.TextScaled = true
                txt.Font = Enum.Font.GothamBold
                txt.TextStrokeTransparency = 0.35
                txt.Text = autoTarget.name
                txt.TextColor3 = filtered and Color3.fromRGB(255, 200, 60) or Color3.fromRGB(255, 190, 70)
            end
        end

        if telaProx.Visible then
            ProxUpdateItemList(proxSearchBox and proxSearchBox.Text or "")
        end
    end
end)

-- Auto coletar
task.spawn(function()
    while true do
        task.wait(0.1)
        if not ScreenGui or not ScreenGui.Parent then break end

        if getgenv().Config.ProxAutoFarm and not ProxBusy then
            local targets = {}
            for _, p in ipairs(workspace:GetDescendants()) do
                if p:IsA("ProximityPrompt") then
                    local part = ProxValidPrompt(p)
                    if part and ProxMatch(p) then
                        local promptId = tostring(p:GetDebugId())
                        if not ProxInteracted[promptId] then
                            table.insert(targets, { prompt = p, part = part, distance = ProxGetDistance(part), id = promptId, kind = "prompt", name = ProxGetName(p) })
                        end
                    end
                end
            end

            for _, autoTarget in ipairs(ProxGetAutoPickupTargets()) do
                if ProxMatchesSelectedName(autoTarget.name) and not ProxInteracted[autoTarget.id] then
                    table.insert(targets, autoTarget)
                end
            end

            table.sort(targets, function(a, b) return a.distance < b.distance end)

            if #targets > 0 then
                local target = targets[1]
                ProxBusy = true
                local char, root = GetCharacterRoot()
                if root then
                    root.CFrame = target.part.CFrame * CFrame.new(0, 5, 0)
                    task.wait(0.5)
                    if target.kind == "auto" then
                        ProxCollectAutoPickup(target.part)
                    else
                        pcall(function() fireproximityprompt(target.prompt) end)
                    end
                    ProxInteracted[target.id] = true
                    task.wait(1)
                end
                ProxBusy = false
            end
        end
    end
end)

-- Limpeza de prompts inválidos
task.spawn(function()
    while true do
        task.wait(30)
        if not ScreenGui or not ScreenGui.Parent then break end
        if getgenv().Config.ProxAutoFarm then
            local validIds = {}
            for _, p in ipairs(workspace:GetDescendants()) do
                if p:IsA("ProximityPrompt") then
                    validIds[tostring(p:GetDebugId())] = true
                end
            end
            for id in pairs(ProxInteracted) do
                if not validIds[id] then ProxInteracted[id] = nil end
            end
        end
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
                local s = 1000 / pos.Z
                if line then
                    line.From = Vector2.new(Camera.ViewportSize.X / 2, 0)
                    line.To = Vector2.new(pos.X, pos.Y)
                    line.Visible = cfg.ESP_Line
                    line.Color = espColor
                end
                obj.Box.Visible = cfg.ESP_Box
                obj.Box.Position = UDim2.new(0, pos.X - s / 2, 0, pos.Y - s / 1.5)
                obj.Box.Size = UDim2.new(0, s, 0, s * 1.5)
                if obj.Box.UIStroke then obj.Box.UIStroke.Color = espColor end
                obj.Label.Visible = (cfg.ESP or cfg.Distance)
                obj.Label.Position = UDim2.new(0, pos.X, 0, pos.Y - s / 1.5 - 12)
                obj.Label.Text = (cfg.ESP and p.Name or "") .. (cfg.Distance and " [" .. math.floor((root.Position - Camera.CFrame.Position).Magnitude) .. "m]" or "")
                obj.Label.TextColor3 = espColor
                obj.HBack.Visible = cfg.Health
                obj.HBack.Position = UDim2.new(0, pos.X - s / 2 - 4, 0, pos.Y - s / 1.5)
                obj.HBack.Size = UDim2.new(0, 2, 0, s * 1.5)
                local hp = p.Character.Humanoid.Health / p.Character.Humanoid.MaxHealth
                obj.HMain.Size = UDim2.new(1, 0, hp, 0)
                obj.HMain.Position = UDim2.new(0, 0, 1 - hp, 0)

                if cfg.AimbotActive then
                    local dist = (Vector2.new(pos.X, pos.Y) - Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)).Magnitude
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

if getgenv().Config.TPInteracao then
    StartTPInteracao()
end

print("PRIDE HUB carregado | filtros corrigidos e UI animada")
