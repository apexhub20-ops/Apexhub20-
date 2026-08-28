if not game:IsLoaded() then game.Loaded:Wait() end

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local ts = game:GetService("TweenService")

getgenv().Config = getgenv().Config or {
    AimbotActive = false, CheckTeam = false, CheckWall = false, Radius = 150, FOVVisible = false,
    TargetPart = "Head", ESP = false, ESP_Box = false, ESP_Line = false, Health = false, Distance = false,
    AimSpeed = 50,
}

local espColor = Color3.new(1, 0, 0)
local fovColor = Color3.new(1, 0, 0)
local IsDraggingSlider = false

for _, v in pairs(game.CoreGui:GetChildren()) do if v.Name == "PRIDE_HUB" then v:Destroy() end end
local ScreenGui = Instance.new("ScreenGui", game.CoreGui); ScreenGui.Name = "PRIDE_HUB"; ScreenGui.IgnoreGuiInset = true

-- FOV Ring
local FOV_Ring = Instance.new("Frame", ScreenGui)
FOV_Ring.Name = "FOV_Ring"
FOV_Ring.BackgroundTransparency = 1
FOV_Ring.AnchorPoint = Vector2.new(0.5, 0.5)
FOV_Ring.Visible = false
FOV_Ring.ZIndex = 5
FOV_Ring.BorderSizePixel = 0

local fovCorner = Instance.new("UICorner", FOV_Ring)
fovCorner.CornerRadius = UDim.new(1, 0)

local fovStroke = Instance.new("UIStroke", FOV_Ring)
fovStroke.Color = fovColor
fovStroke.Thickness = 2
fovStroke.Transparency = 0
fovStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border

-- FUNÇÕES AUXILIARES
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
    container.Size = UDim2.new(0.9, 0, 0, 16)
    container.BackgroundTransparency = 1
    
    local indicator = Instance.new("Frame", container)
    indicator.Size = UDim2.new(0, 2, 0, 12)
    indicator.Position = UDim2.new(0, 0, 0.5, -6)
    indicator.BackgroundColor3 = Color3.fromRGB(180, 30, 30)
    indicator.BorderSizePixel = 0
    CreateCorner(indicator, 1)
    
    local label = Instance.new("TextLabel", container)
    label.Size = UDim2.new(1, -8, 1, 0)
    label.Position = UDim2.new(0, 6, 0, 0)
    label.BackgroundTransparency = 1
    label.Text = text
    label.TextColor3 = Color3.fromRGB(160, 160, 170)
    label.TextSize = 9
    label.Font = Enum.Font.GothamBold
    label.TextXAlignment = Enum.TextXAlignment.Left
    
    return container
end

-- SISTEMA DE NOTIFICAÇÃO
local NotifFrame
local NotifLabel
local NotifIcon
local NotifToken = 0

local function CreateNotification()
    if NotifFrame then return end

    NotifFrame = Instance.new("Frame", ScreenGui)
    NotifFrame.Name = "Notification"
    NotifFrame.Size = UDim2.new(0, 190, 0, 40)
    NotifFrame.AnchorPoint = Vector2.new(0.5, 0)
    NotifFrame.Position = UDim2.new(0.5, 0, 0, -50)
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

    local shadow = Instance.new("UIStroke", NotifFrame)
    shadow.Color = Color3.fromRGB(0, 0, 0)
    shadow.Thickness = 3
    shadow.Transparency = 0.6
    shadow.ApplyStrokeMode = Enum.ApplyStrokeMode.Border

    NotifIcon = Instance.new("TextLabel", NotifFrame)
    NotifIcon.Size = UDim2.new(0, 24, 1, 0)
    NotifIcon.Position = UDim2.new(0, 8, 0, 0)
    NotifIcon.BackgroundTransparency = 1
    NotifIcon.Text = "✓"
    NotifIcon.TextColor3 = Color3.fromRGB(80, 255, 120)
    NotifIcon.TextSize = 16
    NotifIcon.Font = Enum.Font.GothamBlack
    NotifIcon.ZIndex = 1001

    NotifLabel = Instance.new("TextLabel", NotifFrame)
    NotifLabel.Size = UDim2.new(1, -40, 1, 0)
    NotifLabel.Position = UDim2.new(0, 36, 0, 0)
    NotifLabel.BackgroundTransparency = 1
    NotifLabel.Text = ""
    NotifLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    NotifLabel.TextSize = 11
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
    NotifFrame.Position = UDim2.new(0.5, 0, 0, -50)

    ts:Create(NotifFrame, TweenInfo.new(0.25, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), {
        Position = UDim2.new(0.5, 0, 0, 10)
    }):Play()

    task.delay(1.3, function()
        if token ~= NotifToken then return end

        local hide = ts:Create(NotifFrame, TweenInfo.new(0.25, Enum.EasingStyle.Quint, Enum.EasingDirection.In), {
            Position = UDim2.new(0.5, 0, 0, -50)
        })
        hide:Play()
        hide.Completed:Connect(function()
            if token == NotifToken then
                NotifFrame.Visible = false
            end
        end)
    end)
end

-- PAINEL PRINCIPAL (compacto)
local Main = Instance.new("Frame", ScreenGui)
Main.Size = UDim2.new(0, 210, 0, 310)
Main.Position = UDim2.new(0.5, -105, 0.5, -155)
Main.BackgroundColor3 = Color3.fromRGB(20, 20, 25)
Main.BackgroundTransparency = 0.02
Main.Active = true
Main.Draggable = true
Main.ClipsDescendants = true
CreateCorner(Main, 10)
CreateStroke(Main, Color3.fromRGB(50, 50, 55), 1, 0.4)

local gr = Instance.new("UIGradient", Main)
gr.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(24, 24, 30)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(20, 20, 25)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(24, 24, 30))
})
gr.Rotation = 45

-- CABEÇALHO
local hd = Instance.new("Frame", Main)
hd.Size = UDim2.new(1, 0, 0, 38)
hd.BackgroundColor3 = Color3.fromRGB(28, 28, 34)
hd.BackgroundTransparency = 0.05
hd.BorderSizePixel = 0
CreateCorner(hd, 10)

local hdGrad = Instance.new("UIGradient", hd)
hdGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(30, 30, 36)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(38, 26, 28)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(30, 30, 36))
})
hdGrad.Rotation = 90

local hdLine = Instance.new("Frame", hd)
hdLine.Size = UDim2.new(1, 0, 0, 1)
hdLine.Position = UDim2.new(0, 0, 1, -1)
hdLine.BackgroundColor3 = Color3.fromRGB(160, 25, 25)
hdLine.BorderSizePixel = 0

local tt = Instance.new("TextLabel", hd)
tt.Size = UDim2.new(1, -62, 0, 20)
tt.Position = UDim2.new(0, 12, 0, 3)
tt.BackgroundTransparency = 1
tt.Text = "PRIDE HUB"
tt.TextColor3 = Color3.new(1, 1, 1)
tt.TextSize = 13
tt.Font = Enum.Font.GothamBlack
tt.TextXAlignment = Enum.TextXAlignment.Left

local st = Instance.new("TextLabel", hd)
st.Size = UDim2.new(1, -62, 0, 10)
st.Position = UDim2.new(0, 12, 0, 22)
st.BackgroundTransparency = 1
st.Text = "AIM • ESP"
st.TextColor3 = Color3.fromRGB(130, 130, 140)
st.TextSize = 8
st.Font = Enum.Font.GothamMedium
st.TextXAlignment = Enum.TextXAlignment.Left

-- BOTÕES DO HEADER
local btnMin = Instance.new("TextButton", hd)
btnMin.Size = UDim2.new(0, 22, 0, 22)
btnMin.Position = UDim2.new(1, -48, 0.5, -11)
btnMin.BackgroundColor3 = Color3.fromRGB(40, 40, 46)
btnMin.BackgroundTransparency = 0.15
btnMin.Text = "−"
btnMin.TextColor3 = Color3.new(1, 1, 1)
btnMin.TextSize = 14
btnMin.Font = Enum.Font.GothamBold
btnMin.AutoButtonColor = false
CreateCorner(btnMin, 6)
CreateStroke(btnMin, Color3.fromRGB(55, 55, 62), 1, 0.5)

btnMin.MouseEnter:Connect(function()
    AnimateButton(btnMin, {BackgroundColor3 = Color3.fromRGB(55, 55, 62), BackgroundTransparency = 0.08}, 0.1)
end)
btnMin.MouseLeave:Connect(function()
    AnimateButton(btnMin, {BackgroundColor3 = Color3.fromRGB(40, 40, 46), BackgroundTransparency = 0.15}, 0.1)
end)
btnMin.MouseButton1Down:Connect(function()
    AnimateButton(btnMin, {Size = UDim2.new(0, 20, 0, 20)}, 0.05)
end)
btnMin.MouseButton1Up:Connect(function()
    AnimateButton(btnMin, {Size = UDim2.new(0, 22, 0, 22)}, 0.05)
end)

local CloseBtn = Instance.new("TextButton", hd)
CloseBtn.Size = UDim2.new(0, 22, 0, 22)
CloseBtn.Position = UDim2.new(1, -24, 0.5, -11)
CloseBtn.BackgroundColor3 = Color3.fromRGB(40, 40, 46)
CloseBtn.BackgroundTransparency = 0.15
CloseBtn.Text = "×"
CloseBtn.TextColor3 = Color3.new(1, 1, 1)
CloseBtn.TextSize = 14
CloseBtn.Font = Enum.Font.GothamBold
CloseBtn.AutoButtonColor = false
CreateCorner(CloseBtn, 6)
CreateStroke(CloseBtn, Color3.fromRGB(55, 55, 62), 1, 0.5)

CloseBtn.MouseEnter:Connect(function()
    AnimateButton(CloseBtn, {BackgroundColor3 = Color3.fromRGB(70, 40, 40), BackgroundTransparency = 0.08}, 0.1)
end)
CloseBtn.MouseLeave:Connect(function()
    AnimateButton(CloseBtn, {BackgroundColor3 = Color3.fromRGB(40, 40, 46), BackgroundTransparency = 0.15}, 0.1)
end)
CloseBtn.MouseButton1Down:Connect(function()
    AnimateButton(CloseBtn, {Size = UDim2.new(0, 20, 0, 20)}, 0.05)
end)
CloseBtn.MouseButton1Up:Connect(function()
    AnimateButton(CloseBtn, {Size = UDim2.new(0, 22, 0, 22)}, 0.05)
end)

-- ÍCONE MINIMIZADO
local Icon = Instance.new("ImageButton", ScreenGui)
Icon.Size = UDim2.new(0, 42, 0, 42)
Icon.Position = UDim2.new(0, 10, 0, 100)
Icon.Image = "rbxassetid://73630975144333"
Icon.ImageTransparency = 0.05
Icon.BackgroundTransparency = 1
Icon.Visible = false
Icon.ZIndex = 10
CreateCorner(Icon, 8)
CreateStroke(Icon, Color3.fromRGB(160, 25, 25), 2, 0.4)

local dragging = false
local dragStart = nil
local startPos = nil

local function updateDrag(input)
    local delta = input.Position - dragStart
    local newX = math.clamp(startPos.X.Offset + delta.X, 0, ScreenGui.AbsoluteSize.X - Icon.AbsoluteSize.X)
    local newY = math.clamp(startPos.Y.Offset + delta.Y, 0, ScreenGui.AbsoluteSize.Y - Icon.AbsoluteSize.Y)
    Icon.Position = UDim2.new(0, newX, 0, newY)
end

Icon.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = Icon.Position
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
        updateDrag(input)
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = false
    end
end)

local minimizado = false

local function minimizar()
    minimizado = true
    Main.Visible = false
    Icon.Visible = true
    Icon.Size = UDim2.new(0, 21, 0, 21)
    ts:Create(Icon, TweenInfo.new(0.2, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Size = UDim2.new(0, 42, 0, 42)}):Play()
end

local function restaurar()
    minimizado = false
    Main.Visible = true
    Icon.Visible = false
    Main.Size = UDim2.new(0, 190, 0, 290)
    ts:Create(Main, TweenInfo.new(0.2, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), {Size = UDim2.new(0, 210, 0, 310)}):Play()
end

btnMin.MouseButton1Click:Connect(minimizar)
Icon.MouseButton1Click:Connect(restaurar)
CloseBtn.MouseButton1Click:Connect(function()
    Main.Visible = false
    Icon.Visible = false
end)

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

-- ABAS
local barraAbas = Instance.new("Frame", Main)
barraAbas.Size = UDim2.new(1, 0, 0, 28)
barraAbas.Position = UDim2.new(0, 0, 0, 38)
barraAbas.BackgroundColor3 = Color3.fromRGB(14, 14, 18)
barraAbas.BackgroundTransparency = 0.2
barraAbas.BorderSizePixel = 0

local abaAIM = Instance.new("TextButton", barraAbas)
abaAIM.Size = UDim2.new(0.48, 0, 0, 24)
abaAIM.Position = UDim2.new(0.01, 0, 0, 2)
abaAIM.BackgroundColor3 = Color3.fromRGB(140, 20, 20)
abaAIM.BackgroundTransparency = 0.1
abaAIM.Text = "AIM"
abaAIM.TextColor3 = Color3.new(1, 1, 1)
abaAIM.TextSize = 10
abaAIM.Font = Enum.Font.GothamBlack
abaAIM.AutoButtonColor = false
CreateCorner(abaAIM, 6)
CreateStroke(abaAIM, Color3.fromRGB(160, 25, 25), 1, 0.3)

local abaESP = Instance.new("TextButton", barraAbas)
abaESP.Size = UDim2.new(0.48, 0, 0, 24)
abaESP.Position = UDim2.new(0.51, 0, 0, 2)
abaESP.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
abaESP.BackgroundTransparency = 0.15
abaESP.Text = "ESP"
abaESP.TextColor3 = Color3.fromRGB(140, 140, 150)
abaESP.TextSize = 10
abaESP.Font = Enum.Font.GothamBlack
abaESP.AutoButtonColor = false
CreateCorner(abaESP, 6)
CreateStroke(abaESP, Color3.fromRGB(40, 40, 46), 1, 0.5)

local abaIndicator = Instance.new("Frame", barraAbas)
abaIndicator.Size = UDim2.new(0, 24, 0, 2)
abaIndicator.Position = UDim2.new(0.01, 0, 1, -1)
abaIndicator.BackgroundColor3 = Color3.fromRGB(180, 30, 30)
abaIndicator.BorderSizePixel = 0
CreateCorner(abaIndicator, 1)

-- ÁREA DE CONTEÚDO
local contentY = 66
local contentH = 244

local telaAIM = Instance.new("ScrollingFrame", Main)
telaAIM.Size = UDim2.new(1, 0, 0, contentH)
telaAIM.Position = UDim2.new(0, 0, 0, contentY)
telaAIM.BackgroundTransparency = 1
telaAIM.ScrollBarThickness = 3
telaAIM.ScrollBarImageColor3 = Color3.fromRGB(140, 20, 20)
telaAIM.BorderSizePixel = 0
telaAIM.Visible = true
telaAIM.CanvasSize = UDim2.new(0, 0, 0, 0)
telaAIM.AutomaticCanvasSize = Enum.AutomaticSize.Y

local layAIM = Instance.new("UIListLayout", telaAIM)
layAIM.Padding = UDim.new(0, 5)
layAIM.HorizontalAlignment = Enum.HorizontalAlignment.Center
layAIM.SortOrder = Enum.SortOrder.LayoutOrder

local padAIM = Instance.new("UIPadding", telaAIM)
padAIM.PaddingTop = UDim.new(0, 8)
padAIM.PaddingBottom = UDim.new(0, 8)
padAIM.PaddingLeft = UDim.new(0, 6)
padAIM.PaddingRight = UDim.new(0, 6)

local telaESP = Instance.new("ScrollingFrame", Main)
telaESP.Size = UDim2.new(1, 0, 0, contentH)
telaESP.Position = UDim2.new(0, 0, 0, contentY)
telaESP.BackgroundTransparency = 1
telaESP.ScrollBarThickness = 3
telaESP.ScrollBarImageColor3 = Color3.fromRGB(140, 20, 20)
telaESP.BorderSizePixel = 0
telaESP.Visible = false
telaESP.CanvasSize = UDim2.new(0, 0, 0, 0)
telaESP.AutomaticCanvasSize = Enum.AutomaticSize.Y

local layESP = Instance.new("UIListLayout", telaESP)
layESP.Padding = UDim.new(0, 5)
layESP.HorizontalAlignment = Enum.HorizontalAlignment.Center
layESP.SortOrder = Enum.SortOrder.LayoutOrder

local padESP = Instance.new("UIPadding", telaESP)
padESP.PaddingTop = UDim.new(0, 8)
padESP.PaddingBottom = UDim.new(0, 8)
padESP.PaddingLeft = UDim.new(0, 6)
padESP.PaddingRight = UDim.new(0, 6)

-- SELETOR DE CORES COMPACTO
local selectedColorBtn = nil

local function criarSeletorCores(parent, callback)
    local frame = Instance.new("Frame", parent)
    frame.Size = UDim2.new(0.9, 0, 0, 48)
    frame.BackgroundTransparency = 1
    frame.LayoutOrder = 99
    
    local grid = Instance.new("UIGridLayout", frame)
    grid.CellSize = UDim2.new(0, 28, 0, 20)
    grid.CellPadding = UDim2.new(0, 4, 0, 4)
    grid.HorizontalAlignment = Enum.HorizontalAlignment.Center
    
    local cores = {
        Color3.new(1, 0, 0), Color3.new(1, 0.5, 0), Color3.new(1, 1, 0),
        Color3.new(0, 1, 0), Color3.new(0, 1, 1), Color3.new(0, 0, 1),
        Color3.new(0.5, 0, 1), Color3.new(1, 0, 1), Color3.new(1, 1, 1),
        Color3.new(0, 0, 0)
    }
    
    for _, c in ipairs(cores) do
        local bt = Instance.new("TextButton", frame)
        bt.Size = UDim2.new(0, 28, 0, 20)
        bt.BackgroundColor3 = c
        bt.Text = ""
        bt.AutoButtonColor = false
        CreateCorner(bt, 4)
        CreateStroke(bt, Color3.fromRGB(50, 50, 55), 1, 0.3)
        
        bt.MouseEnter:Connect(function()
            AnimateButton(bt, {Size = UDim2.new(0, 30, 0, 22)}, 0.08)
        end)
        bt.MouseLeave:Connect(function()
            AnimateButton(bt, {Size = UDim2.new(0, 28, 0, 20)}, 0.08)
        end)
        bt.MouseButton1Down:Connect(function()
            AnimateButton(bt, {Size = UDim2.new(0, 26, 0, 18)}, 0.05)
        end)
        bt.MouseButton1Up:Connect(function()
            AnimateButton(bt, {Size = UDim2.new(0, 28, 0, 20)}, 0.05)
        end)
        
        bt.MouseButton1Click:Connect(function()
            if selectedColorBtn then
                if selectedColorBtn:FindFirstChild("UIStroke2") then
                    selectedColorBtn:FindFirstChild("UIStroke2"):Destroy()
                end
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

-- TOGGLE MODERNO CORRIGIDO
local function AddToggleVertical(name, prop, parent)
    local btn = Instance.new("TextButton", parent)
    btn.Size = UDim2.new(0.9, 0, 0, 36)
    btn.BackgroundColor3 = Color3.fromRGB(30, 30, 36)
    btn.BackgroundTransparency = 0.05
    btn.Text = ""
    btn.AutoButtonColor = false
    btn.BorderSizePixel = 0
    CreateCorner(btn, 7)
    CreateStroke(btn, Color3.fromRGB(50, 50, 56), 1, 0.5)
    
    local label = Instance.new("TextLabel", btn)
    label.Size = UDim2.new(0.55, 0, 1, 0)
    label.Position = UDim2.new(0, 10, 0, 0)
    label.BackgroundTransparency = 1
    label.Text = name
    label.TextColor3 = Color3.new(1, 1, 1)
    label.TextSize = 9
    label.Font = Enum.Font.GothamBold
    label.TextXAlignment = Enum.TextXAlignment.Left
    
    local toggleDot = Instance.new("Frame", btn)
    toggleDot.Size = UDim2.new(0, 34, 0, 18)
    toggleDot.Position = UDim2.new(1, -42, 0.5, -9)
    toggleDot.BackgroundColor3 = Color3.fromRGB(45, 45, 52)
    toggleDot.BorderSizePixel = 0
    CreateCorner(toggleDot, 9)
    
    local dot = Instance.new("Frame", toggleDot)
    dot.Size = UDim2.new(0, 14, 0, 14)
    dot.Position = UDim2.new(0, 2, 0.5, -7)
    dot.BackgroundColor3 = Color3.fromRGB(100, 100, 110)
    dot.BorderSizePixel = 0
    CreateCorner(dot, 7)
    
    local function updateToggle(showNotif)
        local on = getgenv().Config[prop]
        ts:Create(dot, TweenInfo.new(0.15), {Position = UDim2.new(0, on and 18 or 2, 0.5, -7), BackgroundColor3 = on and Color3.fromRGB(220, 50, 50) or Color3.fromRGB(100, 100, 110)}):Play()
        ts:Create(toggleDot, TweenInfo.new(0.15), {BackgroundColor3 = on and Color3.fromRGB(60, 20, 20) or Color3.fromRGB(45, 45, 52)}):Play()
        ts:Create(btn, TweenInfo.new(0.15), {BackgroundColor3 = on and Color3.fromRGB(40, 20, 20) or Color3.fromRGB(30, 30, 36)}):Play()
        
        if showNotif then
            ShowNotification(name .. (on and " ATIVO" or " DESATIVADO"), on)
        end
    end
    
    updateToggle(false)

    btn.MouseEnter:Connect(function()
        AnimateButton(btn, {BackgroundTransparency = 0}, 0.08)
    end)
    btn.MouseLeave:Connect(function()
        AnimateButton(btn, {BackgroundTransparency = 0.05}, 0.08)
    end)
    btn.MouseButton1Down:Connect(function()
        AnimateButton(btn, {Size = UDim2.new(0.88, 0, 0, 34)}, 0.05)
    end)
    btn.MouseButton1Up:Connect(function()
        AnimateButton(btn, {Size = UDim2.new(0.9, 0, 0, 36)}, 0.05)
    end)

    btn.MouseButton1Click:Connect(function()
        getgenv().Config[prop] = not getgenv().Config[prop]
        updateToggle(true)
    end)
    return btn
end

-- SLIDER COMPACTO
local function AddSliderVertical(name, prop, parent, max, min, suffix)
    min = min or 0
    suffix = suffix or ""
    local frame = Instance.new("Frame", parent)
    frame.Size = UDim2.new(0.9, 0, 0, 40)
    frame.BackgroundColor3 = Color3.fromRGB(26, 26, 32)
    frame.BackgroundTransparency = 0.05
    frame.BorderSizePixel = 0
    CreateCorner(frame, 7)
    CreateStroke(frame, Color3.fromRGB(50, 50, 56), 1, 0.5)
    
    local label = Instance.new("TextLabel", frame)
    label.Size = UDim2.new(0.5, 0, 0, 18)
    label.Position = UDim2.new(0, 8, 0, 2)
    label.BackgroundTransparency = 1
    label.Text = name
    label.TextColor3 = Color3.new(1, 1, 1)
    label.TextSize = 8
    label.Font = Enum.Font.GothamBold
    label.TextXAlignment = Enum.TextXAlignment.Left
    
    local valueLabel = Instance.new("TextLabel", frame)
    valueLabel.Size = UDim2.new(0.4, 0, 0, 18)
    valueLabel.Position = UDim2.new(0.6, 0, 0, 2)
    valueLabel.BackgroundTransparency = 1
    valueLabel.Text = getgenv().Config[prop] .. suffix
    valueLabel.TextColor3 = Color3.fromRGB(220, 60, 60)
    valueLabel.TextSize = 9
    valueLabel.Font = Enum.Font.GothamBlack
    valueLabel.TextXAlignment = Enum.TextXAlignment.Right

    local barBg = Instance.new("Frame", frame)
    barBg.Size = UDim2.new(0.9, 0, 0, 8)
    barBg.Position = UDim2.new(0.05, 0, 0, 22)
    barBg.BackgroundColor3 = Color3.fromRGB(45, 45, 52)
    barBg.BorderSizePixel = 0
    CreateCorner(barBg, 4)

    local barFill = Instance.new("Frame", barBg)
    local percent = (getgenv().Config[prop] - min) / (max - min)
    barFill.Size = UDim2.new(percent, 0, 1, 0)
    barFill.BackgroundColor3 = Color3.fromRGB(160, 25, 25)
    barFill.BorderSizePixel = 0
    CreateCorner(barFill, 4)

    local dot = Instance.new("TextButton", barBg)
    dot.Size = UDim2.new(0, 16, 0, 16)
    dot.Position = UDim2.new(percent, -8, 0.5, -8)
    dot.BackgroundColor3 = Color3.new(1, 1, 1)
    dot.Text = ""
    dot.AutoButtonColor = false
    CreateCorner(dot, 8)
    CreateStroke(dot, Color3.fromRGB(160, 25, 25), 2, 0)

    local sliderConn = nil
    local function updateSlider(input)
        local p = math.clamp((input.Position.X - barBg.AbsolutePosition.X) / barBg.AbsoluteSize.X, 0, 1)
        local val = math.floor(min + p * (max - min))
        getgenv().Config[prop] = val
        valueLabel.Text = val .. suffix
        barFill.Size = UDim2.new(p, 0, 1, 0)
        dot.Position = UDim2.new(p, -8, 0.5, -8)
    end

    barBg.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            IsDraggingSlider = true
            sliderConn = UserInputService.InputChanged:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
                    updateSlider(input)
                end
            end)
            updateSlider(input)
        end
    end)

    dot.MouseButton1Down:Connect(function()
        IsDraggingSlider = true
        sliderConn = UserInputService.InputChanged:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
                updateSlider(input)
            end
        end)
    end)
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            if sliderConn then 
                sliderConn:Disconnect()
                sliderConn = nil
                ShowNotification(name .. ": " .. getgenv().Config[prop] .. suffix, true)
            end
            IsDraggingSlider = false
        end
    end)
    return frame
end

-- ABA AIM
CreateSectionLabel(telaAIM, "AIMBOT")
AddToggleVertical("ATIVAR AIM", "AimbotActive", telaAIM)
AddToggleVertical("TIME", "CheckTeam", telaAIM)
AddToggleVertical("PAREDE", "CheckWall", telaAIM)
AddToggleVertical("MOSTRAR FOV", "FOVVisible", telaAIM)

CreateSectionLabel(telaAIM, "FOV")
AddSliderVertical("RAIO FOV", "Radius", telaAIM, 600)

CreateSectionLabel(telaAIM, "ALVO")
local PartBtn = Instance.new("TextButton", telaAIM)
PartBtn.Size = UDim2.new(0.9, 0, 0, 30)
PartBtn.BackgroundColor3 = Color3.fromRGB(30, 24, 24)
PartBtn.BackgroundTransparency = 0.05
PartBtn.Text = "ALVO: CABEÇA"
PartBtn.TextColor3 = Color3.new(1, 1, 1)
PartBtn.TextSize = 9
PartBtn.Font = Enum.Font.GothamBold
PartBtn.AutoButtonColor = false
CreateCorner(PartBtn, 7)
CreateStroke(PartBtn, Color3.fromRGB(160, 25, 25), 1, 0.4)

PartBtn.MouseEnter:Connect(function()
    AnimateButton(PartBtn, {BackgroundColor3 = Color3.fromRGB(50, 25, 25)}, 0.1)
end)
PartBtn.MouseLeave:Connect(function()
    AnimateButton(PartBtn, {BackgroundColor3 = Color3.fromRGB(30, 24, 24)}, 0.1)
end)
PartBtn.MouseButton1Down:Connect(function()
    AnimateButton(PartBtn, {Size = UDim2.new(0.88, 0, 0, 28)}, 0.05)
end)
PartBtn.MouseButton1Up:Connect(function()
    AnimateButton(PartBtn, {Size = UDim2.new(0.9, 0, 0, 30)}, 0.05)
end)

PartBtn.MouseButton1Click:Connect(function()
    getgenv().Config.TargetPart = (getgenv().Config.TargetPart == "Head" and "HumanoidRootPart" or "Head")
    PartBtn.Text = "ALVO: " .. (getgenv().Config.TargetPart == "Head" and "CABEÇA" or "TRONCO")
    ShowNotification("ALVO: " .. (getgenv().Config.TargetPart == "Head" and "CABEÇA" or "TRONCO"), true)
end)

CreateSectionLabel(telaAIM, "VELOCIDADE")
AddSliderVertical("VELOC AIMBOT", "AimSpeed", telaAIM, 100, 1, "%")

CreateSectionLabel(telaAIM, "COR FOV")
criarSeletorCores(telaAIM, function(cor)
    fovColor = cor
    fovStroke.Color = cor
end)

-- ABA ESP
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

-- NAVEGAÇÃO
local function selecionarAba(aba)
    telaAIM.Visible = (aba == "aim")
    telaESP.Visible = (aba == "esp")
    
    if aba == "aim" then
        ts:Create(abaAIM, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(140, 20, 20), BackgroundTransparency = 0.1, TextColor3 = Color3.new(1, 1, 1)}):Play()
        ts:Create(abaESP, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(24, 24, 30), BackgroundTransparency = 0.15, TextColor3 = Color3.fromRGB(140, 140, 150)}):Play()
        ts:Create(abaIndicator, TweenInfo.new(0.15), {Position = UDim2.new(0.01, 0, 1, -1)}):Play()
        if abaAIM:FindFirstChild("UIStroke") then abaAIM.UIStroke.Color = Color3.fromRGB(160, 25, 25) abaAIM.UIStroke.Transparency = 0.3 end
        if abaESP:FindFirstChild("UIStroke") then abaESP.UIStroke.Color = Color3.fromRGB(40, 40, 46) abaESP.UIStroke.Transparency = 0.5 end
    else
        ts:Create(abaESP, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(140, 20, 20), BackgroundTransparency = 0.1, TextColor3 = Color3.new(1, 1, 1)}):Play()
        ts:Create(abaAIM, TweenInfo.new(0.15), {BackgroundColor3 = Color3.fromRGB(24, 24, 30), BackgroundTransparency = 0.15, TextColor3 = Color3.fromRGB(140, 140, 150)}):Play()
        ts:Create(abaIndicator, TweenInfo.new(0.15), {Position = UDim2.new(0.51, 0, 1, -1)}):Play()
        if abaESP:FindFirstChild("UIStroke") then abaESP.UIStroke.Color = Color3.fromRGB(160, 25, 25) abaESP.UIStroke.Transparency = 0.3 end
        if abaAIM:FindFirstChild("UIStroke") then abaAIM.UIStroke.Color = Color3.fromRGB(40, 40, 46) abaAIM.UIStroke.Transparency = 0.5 end
    end
end

abaAIM.MouseButton1Click:Connect(function() selecionarAba("aim") end)
abaESP.MouseButton1Click:Connect(function() selecionarAba("esp") end)
selecionarAba("aim")

-- ESP SYSTEM
local ESP_Lines = {}
local ESP_Table = {}

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

RunService.RenderStepped:Connect(function()
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
                                local aimSpeed = cfg.AimSpeed / 100
                                local currentCF = Camera.CFrame
                                local targetCF = CFrame.new(currentCF.Position, Target.Position)
                                Camera.CFrame = currentCF:Lerp(targetCF, aimSpeed * 0.5)
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
end)
