local player = game.Players.LocalPlayer
local ts = game:GetService("TweenService")
local uis = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local SoundService = game:GetService("SoundService")
local Debris = game:GetService("Debris")
local Camera = workspace.CurrentCamera or workspace:FindFirstChildWhichIsA("Camera")

local cor = {
    roxo = Color3.fromRGB(140, 60, 255), roxoC = Color3.fromRGB(190, 140, 255),
    roxoE = Color3.fromRGB(45, 18, 90), ciano = Color3.fromRGB(0, 240, 255),
    pink = Color3.fromRGB(255, 50, 180), verde = Color3.fromRGB(40, 230, 40),
    vermelho = Color3.fromRGB(230, 40, 40), amarelo = Color3.fromRGB(255, 220, 0),
    branco = Color3.fromRGB(240, 240, 255), cinza = Color3.fromRGB(150, 150, 185),
    fundo = Color3.fromRGB(5, 2, 14),
}

local st = {pos=nil,marker=nil,tp=false,dedo=false,dc=nil,min=false,ui=false,configs={},cfgTela=false}
local spdVal = 16; local jmpVal = 50
local spdAlways = false; local jmpAlways = false
local noclipOn = false; local noclipAlways = false; local noclipConn = nil
local alwaysConns = {}
local afkOn = false; local afkConn = nil
local spinOn = false; local spinConn = nil
local gui,main,sta,led,tpBtn,barra,saveBtn,notif,ba,bd
local telaHOME, telaPLAYER, telaESP, telaConfig

-- ============================================================
--  ESP SETTINGS
-- ============================================================
local esp = {
    ativo = false, cor = Color3.fromRGB(0, 240, 255),
    box = false, tracerCima = false, tracerBaixo = false,
    vidaBarra = false, dist = false, nome = false,
    chams = false,
}
local espD = {}; local espConn = nil; local chamsObjs = {}
local espAlvo = nil; local espSetaDraw = nil

local function criarDrawing(tipo)
    local ok, d = pcall(Drawing.new, tipo)
    return ok and d or nil
end

local function atualizarChams()
    for _, v in pairs(chamsObjs) do
        pcall(function() v:Destroy() end)
    end
    chamsObjs = {}
    if not esp.ativo or not esp.chams then return end
    for _, plr in pairs(game.Players:GetPlayers()) do
        if plr == player then continue end
        local char = plr.Character
        if not char then continue end
        local hl = Instance.new("Highlight")
        hl.Adornee = char; hl.FillColor = esp.cor; hl.OutlineColor = Color3.fromRGB(255,255,255)
        hl.FillTransparency = 0.65; hl.OutlineTransparency = 0.2
        hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop; hl.Parent = gui
        chamsObjs[plr] = hl
    end
end

local function limparSeta()
    if espSetaDraw then pcall(function() espSetaDraw.Visible = false end) end
    espAlvo = nil
end

local function espLigar()
    if espConn then espConn:Disconnect() espConn = nil end
    for _, objs in pairs(espD) do
        for _, d in pairs(objs) do
            pcall(function() d.Visible = false end)
        end
    end
    espD = {}
    atualizarChams()
    if not esp.ativo then
        if espSetaDraw then pcall(function() espSetaDraw.Visible = false end) end
        return
    end

    espConn = RunService.RenderStepped:Connect(function()
        local vp = Camera.ViewportSize

        -- Seta da pessoa especifica
        if espAlvo and esp.ativo then
            local char = espAlvo.Character
            if char then
                local head = char:FindFirstChild("Head")
                local hrp = char:FindFirstChild("HumanoidRootPart")
                if head or hrp then
                    local pos = (head or hrp).Position + Vector3.new(0, 3.5, 0)
                    local scr, onScr = Camera:WorldToViewportPoint(pos)
                    if onScr then
                        if not espSetaDraw then
                            espSetaDraw = criarDrawing("Text")
                            if espSetaDraw then espSetaDraw.Size = 28 espSetaDraw.Center = true espSetaDraw.Outline = true end
                        end
                        if espSetaDraw then
                            espSetaDraw.Visible = true
                            espSetaDraw.Position = Vector2.new(scr.X, scr.Y)
                            espSetaDraw.Text = "▼"
                            espSetaDraw.Color = esp.cor
                        end
                    elseif espSetaDraw then
                        espSetaDraw.Visible = false
                    end
                end
            elseif espSetaDraw then
                espSetaDraw.Visible = false
            end
        elseif espSetaDraw then
            espSetaDraw.Visible = false
        end

        for _, plr in pairs(game.Players:GetPlayers()) do
            if plr == player then continue end
            local char = plr.Character
            if not char then continue end
            local hrp = char:FindFirstChild("HumanoidRootPart")
            local head = char:FindFirstChild("Head")
            local hum = char:FindFirstChildOfClass("Humanoid")
            if not hrp or not hum then continue end
            local hp = hum.Health
            if hp <= 0 then continue end

            local headPos = (head or hrp).Position
            local footPos = hrp.Position - Vector3.new(0, 3, 0)
            local hScr, onScr = Camera:WorldToViewportPoint(headPos + Vector3.new(0, 0.5, 0))
            local fScr, _ = Camera:WorldToViewportPoint(footPos)

            if not onScr then
                if espD[plr] then
                    for _, d in pairs(espD[plr]) do pcall(function() d.Visible = false end) end
                end
                continue
            end

            local cx = hScr.X
            local boxH = math.max(fScr.Y - hScr.Y, 20)
            local boxW = boxH * 0.45

            if not espD[plr] then
                local b = criarDrawing("Square"); if b then b.Filled = false b.Thickness = 1.2 end
                local tc = criarDrawing("Line"); if tc then tc.Thickness = 1.2 end
                local tb = criarDrawing("Line"); if tb then tb.Thickness = 1.2 end
                local hbg = criarDrawing("Square"); if hbg then hbg.Filled = true end
                local hfl = criarDrawing("Square"); if hfl then hfl.Filled = true end
                local dt = criarDrawing("Text"); if dt then dt.Size = 13 dt.Center = true dt.Outline = true end
                local nm = criarDrawing("Text"); if nm then nm.Size = 12 nm.Center = true nm.Outline = true end
                espD[plr] = {box=b, tc=tc, tb=tb, hbg=hbg, hfl=hfl, distTxt=dt, nomeTxt=nm}
            end

            local o = espD[plr]
            local c = esp.cor
            local topY = hScr.Y; local botY = fScr.Y; local ht = boxH; local wd = boxW
            local midY = topY + ht/2

            -- BOX
            if esp.box and o.box then
                o.box.Visible = true; o.box.Color = c
                o.box.Position = Vector2.new(cx - wd/2, topY); o.box.Size = Vector2.new(wd, ht)
            elseif o.box then o.box.Visible = false end

            -- TRACER CIMA
            if esp.tracerCima and o.tc then
                o.tc.Visible = true; o.tc.Color = c
                o.tc.From = Vector2.new(vp.X/2, 0); o.tc.To = Vector2.new(cx, midY)
            elseif o.tc then o.tc.Visible = false end

            -- TRACER BAIXO
            if esp.tracerBaixo and o.tb then
                o.tb.Visible = true; o.tb.Color = c
                o.tb.From = Vector2.new(vp.X/2, vp.Y); o.tb.To = Vector2.new(cx, midY)
            elseif o.tb then o.tb.Visible = false end

            -- VIDA BARRA
            if esp.vidaBarra and o.hbg and o.hfl then
                local bw = 4; local bx = cx + wd/2 + 2
                o.hbg.Visible = true; o.hbg.Color = Color3.fromRGB(30,30,30)
                o.hbg.Position = Vector2.new(bx, topY); o.hbg.Size = Vector2.new(bw, ht)
                local pct = math.clamp(hp / hum.MaxHealth, 0, 1)
                local fh = ht * pct; local fy = topY + ht - fh
                local hc = pct > 0.5 and Color3.fromRGB(0,255,0) or (pct > 0.25 and Color3.fromRGB(255,255,0) or Color3.fromRGB(255,0,0))
                o.hfl.Visible = true; o.hfl.Color = hc
                o.hfl.Position = Vector2.new(bx, fy); o.hfl.Size = Vector2.new(bw, fh)
            else
                if o.hbg then o.hbg.Visible = false end; if o.hfl then o.hfl.Visible = false end
            end

            -- DISTANCIA
            if esp.dist and o.distTxt then
                o.distTxt.Visible = true
                local myHrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
                local dist = myHrp and (hrp.Position - myHrp.Position).Magnitude or 0
                o.distTxt.Position = Vector2.new(cx, topY - 20)
                o.distTxt.Text = tostring(math.floor(dist)).."m"; o.distTxt.Color = c
            elseif o.distTxt then o.distTxt.Visible = false end

            -- NOME
            if esp.nome and o.nomeTxt then
                o.nomeTxt.Visible = true
                o.nomeTxt.Position = Vector2.new(cx, botY + 4)
                o.nomeTxt.Text = plr.Name; o.nomeTxt.Color = c
            elseif o.nomeTxt then o.nomeTxt.Visible = false end
        end
    end)
end

-- ============================================================
--  SOUND & EFFECTS (mantido igual)
-- ============================================================
local function snd(id,v)
    local s=Instance.new("Sound") s.SoundId=id s.Volume=v or 0.3 s.Parent=SoundService return s
end
local sv=snd("rbxassetid://91203790950646",0.3)
local tpSnd=snd("rbxassetid://91203854328530",0.2)

local function tracer(a,b)
    local d=(b-a).Magnitude local q=math.max(math.floor(d*2.5),15)
    for i=0,q do
        local p=Instance.new("Part") p.Size=Vector3.new(0.12,0.12,0.12) p.Shape=Enum.PartType.Ball
        p.Position=a:Lerp(b,i/q) p.Anchored=true p.CanCollide=false p.Material=Enum.Material.Neon
        p.Color=({cor.ciano,cor.roxoC,cor.pink})[math.random(1,3)] p.Transparency=0.15 p.Parent=workspace Debris:AddItem(p,0.4)
        task.spawn(function()task.wait(0.15)for t=1,10 do p.Transparency=p.Transparency+0.08 if p.Transparency>=1 then break end task.wait()end if p.Parent then p:Destroy()end end)
    end
end
local function raio(a,b)
    local d=(b-a).Magnitude local q=math.max(math.floor(d*3),10)
    for i=0,q do
        local p=Instance.new("Part") p.Size=Vector3.new(0.2,0.2,0.2) p.Shape=Enum.PartType.Ball
        local off=Vector3.new(math.random(-0.5,0.5),math.random(-0.5,0.5),math.random(-0.5,0.5))
        if i==0 or i==q then off=Vector3.new()end
        p.Position=a:Lerp(b,i/q)+off p.Anchored=true p.CanCollide=false p.Material=Enum.Material.Neon p.Color=cor.branco p.Transparency=0.05 p.Parent=workspace Debris:AddItem(p,0.3)
        task.spawn(function()task.wait(0.05)for t=1,6 do p.Transparency=p.Transparency+0.15 if p.Transparency>=1 then break end task.wait()end if p.Parent then p:Destroy()end end)
    end
end
local function anel(pos)
    for i=1,4 do
        local a=Instance.new("Part") a.Size=Vector3.new(0.15,0.15,0.15) a.Shape=Enum.PartType.Ball a.Position=pos
        a.Anchored=true a.CanCollide=false a.Material=Enum.Material.Neon a.Color=cor.ciano a.Transparency=0.3 a.Parent=workspace Debris:AddItem(a,0.8)
        local ang=0 local raio2=0.5+i*0.8
        task.spawn(function()while a and a.Parent do ang=ang+0.1 raio2=raio2+0.06 a.Position=pos+Vector3.new(math.cos(ang)*raio2,math.sin(ang*2)*0.2,math.sin(ang)*raio2)a.Transparency=a.Transparency+0.05 if a.Transparency>=0.95 then break end task.wait(0.02)end if a and a.Parent then a:Destroy()end end)
    end
end
local function estrelas(pos)
    for i=1,15 do
        local p=Instance.new("Part") p.Size=Vector3.new(0.12,0.12,0.12) p.Shape=Enum.PartType.Ball
        p.Position=pos p.Anchored=true p.CanCollide=false p.Material=Enum.Material.Neon
        p.Color=({cor.roxoC,cor.ciano,cor.pink,cor.branco})[math.random(1,4)] p.Transparency=0.05 p.Parent=workspace Debris:AddItem(p,0.8)
        local dir=Vector3.new(math.random(-8,8),math.random(-4,8),math.random(-8,8))
        task.spawn(function()for t=1,12 do p.Position=p.Position+dir*0.1 dir=dir+Vector3.new(0,-0.15,0) p.Transparency=p.Transparency+0.08 if p.Transparency>=1 then break end task.wait()end if p.Parent then p:Destroy()end end)
    end
end

local function chao(pos)
    local p=RaycastParams.new() p.FilterType=Enum.RaycastFilterType.Blacklist p.FilterDescendantsInstances={player.Character}
    local h=workspace:Raycast(pos+Vector3.new(0,20,0),Vector3.new(0,-100,0),p)
    if h then return h.Position+Vector3.new(0,3.2,0) end
    h=workspace:Raycast(pos+Vector3.new(0,50,0),Vector3.new(0,-500,0),p)
    if h then return h.Position+Vector3.new(0,3.2,0) end
    return Vector3.new(pos.X,pos.Y+3,pos.Z)
end

local function tp(alvo)
    if not alvo or st.tp then return end
    local char=player.Character if not char then return end
    local hrp=char:FindFirstChild("HumanoidRootPart")or char:FindFirstChildWhichIsA("BasePart")
    local hum=char:FindFirstChildOfClass("Humanoid") if not hrp then return end
    st.tp=true
    local o=hrp.Position local d=chao(alvo)
    if sta then sta.Text="TP..."sta.TextColor3=cor.amarelo end
    if led then led.BackgroundColor3=cor.amarelo end
    tpSnd:Play()
    tracer(o,d) raio(o,d) anel(o) estrelas(o)
    local w=Instance.new("Part") w.Size=Vector3.new(1,1,1) w.Position=d w.Anchored=true w.Transparency=1 w.Parent=workspace
    pcall(function()hrp:SetNetworkOwner(player)end) pcall(function()hum.PlatformStand=true end)
    hrp.Anchored=true hrp.AssemblyLinearVelocity=Vector3.new(0,0,0)
    hrp.CFrame=CFrame.new(d) task.wait(0.01)
    hrp.CFrame=CFrame.new(d) task.wait(0.01)
    hrp.CFrame=CFrame.new(d) hrp.Anchored=false
    if hum then pcall(function()hum.PlatformStand=false end)end
    local bp=Instance.new("BodyPosition") bp.MaxForce=Vector3.new(9e9,9e9,9e9) bp.P=3000 bp.D=500 bp.Position=d bp.Parent=hrp task.wait(0.03) bp:Destroy()
    w:Destroy() anel(d) estrelas(d)
    if barra then ts:Create(barra,TweenInfo.new(0.1),{Size=UDim2.new(1,0,1,0)}):Play()end
    if sta then sta.Text="Teleportado!"sta.TextColor3=cor.verde end
    if led then led.BackgroundColor3=cor.verde end
    task.delay(0.6,function()if sta then sta.Text="TP pronto"sta.TextColor3=cor.cinza end if led then led.BackgroundColor3=cor.roxo end if barra then ts:Create(barra,TweenInfo.new(0.3),{Size=UDim2.new(0,0,1,0)}):Play()end end)
    st.tp=false
end

local function marcar(pos)
    if not pos then return end
    st.pos=pos
    if st.marker then st.marker:Destroy()end
    st.marker=Instance.new("Part") st.marker.Name="RatHub_Marker" st.marker.Shape=Enum.PartType.Ball
    st.marker.Size=Vector3.new(2,2,2) st.marker.Position=pos st.marker.Anchored=true st.marker.CanCollide=false st.marker.Material=Enum.Material.Neon
    st.marker.Color=cor.roxoC st.marker.Transparency=0.1 st.marker.Parent=workspace
    local lu=Instance.new("PointLight",st.marker) lu.Range=8 lu.Color=cor.roxoC lu.Brightness=0.8
    for i=1,12 do
        local p=Instance.new("Part") p.Size=Vector3.new(0.15,0.15,0.15) p.Shape=Enum.PartType.Ball
        p.Position=pos+Vector3.new(math.random(-3,3),math.random(-2,2),math.random(-3,3))
        p.Anchored=true p.CanCollide=false p.Material=Enum.Material.Neon p.Color=i%2==0 and cor.roxo or cor.roxoC p.Transparency=0.1 p.Parent=workspace Debris:AddItem(p,1)
        local vel=Vector3.new(math.random(-4,4),math.random(2,6),math.random(-4,4))
        task.spawn(function()for t=1,12 do p.Position=p.Position+vel*0.02 vel=vel+Vector3.new(0,-0.3,0)p.Transparency=p.Transparency+0.07 if p.Transparency>=1 then break end task.wait()end if p.Parent then p:Destroy()end end)
    end
    anel(pos)
    local bill=Instance.new("BillboardGui") bill.Size=UDim2.new(0,70,0,22) bill.StudsOffset=Vector3.new(0,2.5,0) bill.AlwaysOnTop=true bill.Adornee=st.marker bill.Parent=gui
    local txt=Instance.new("TextLabel",bill) txt.Size=UDim2.new(1,0,1,0) txt.BackgroundTransparency=1 txt.Text="TP" txt.TextColor3=cor.branco txt.TextSize=12 txt.Font=Enum.Font.GothamBlack
    sv:Play()
    if tpBtn then tpBtn.Visible=true end
    if sta then sta.Text="Salvo!"sta.TextColor3=cor.verde end
    if led then led.BackgroundColor3=cor.verde end
    if barra then ts:Create(barra,TweenInfo.new(0.3),{Size=UDim2.new(1,0,1,0)}):Play()end
    task.delay(1.2,function()if barra then ts:Create(barra,TweenInfo.new(0.3),{Size=UDim2.new(0,0,1,0)}):Play()end if sta then sta.Text="TP pronto"sta.TextColor3=cor.cinza end if led then led.BackgroundColor3=cor.roxo end end)
end

-- ============================================================
--  SPEED & JUMP
-- ============================================================
local function aplicarSpeed(val)
    local char = player.Character
    if char then
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then hum.WalkSpeed = val end
    end
end
local function aplicarJump(val)
    local char = player.Character
    if char then
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then hum.JumpPower = val end
    end
end
local function setupAlways()
    for _, conn in pairs(alwaysConns) do pcall(function() conn:Disconnect() end) end
    alwaysConns = {}
    if spdAlways then
        aplicarSpeed(spdVal)
        local conn = player.CharacterAdded:Connect(function() task.wait(0.5) aplicarSpeed(spdVal) end)
        table.insert(alwaysConns, conn)
    end
    if jmpAlways then
        aplicarJump(jmpVal)
        local conn = player.CharacterAdded:Connect(function() task.wait(0.5) aplicarJump(jmpVal) end)
        table.insert(alwaysConns, conn)
    end
    if noclipAlways then
        if player.Character then toggleNoclip(true) end
        local conn = player.CharacterAdded:Connect(function() task.wait(0.5) toggleNoclip(true) end)
        table.insert(alwaysConns, conn)
    end
end



-- Noclip
local function toggleNoclip(on)
    if noclipConn then noclipConn:Disconnect(); noclipConn = nil end
    noclipOn = on
    if not on then return end
    noclipConn = RunService.Stepped:Connect(function()
        local char = player.Character
        if not char then return end
        for _, p in pairs(char:GetDescendants()) do
            if p:IsA("BasePart") then p.CanCollide = false end
        end
    end)
end

-- Anti AFK
local function toggleAFK(on)
    if afkConn then afkConn:Disconnect(); afkConn = nil end
    afkOn = on
    if not on then return end
    afkConn = RunService.Stepped:Connect(function()
        if not afkOn then return end
        local char = player.Character
        if char and char:FindFirstChild("Humanoid") then
            char.Humanoid:Move(Vector3.new(0.001, 0, 0), false)
        end
    end)
end

-- Spin Bot
local function toggleSpin(on)
    if spinConn then spinConn:Disconnect(); spinConn = nil end
    spinOn = on
    if not on then return end
    spinConn = RunService.RenderStepped:Connect(function()
        if not spinOn then return end
        local char = player.Character
        if char and char:FindFirstChild("HumanoidRootPart") then
            char.HumanoidRootPart.CFrame = char.HumanoidRootPart.CFrame * CFrame.Angles(0, math.rad(15), 0)
        end
    end)
end

-- Server Hop
local function serverHop()
    local http = game:GetService("HttpService")
    local ts = game:GetService("TeleportService")
    local api = "https://games.roblox.com/v1/games/"..game.PlaceId.."/servers/Public?limit=100"
    local success, data = pcall(function()
        return http:JSONDecode(game:HttpGetAsync(api))
    end)
    if success and data and data.data and #data.data > 0 then
        local servers = data.data
        local chosen = servers[math.random(1, #servers)]
        ts:TeleportToPlaceInstance(game.PlaceId, chosen.id, player)
    end
end

-- Rejoin
local function rejoin()
    local ts = game:GetService("TeleportService")
    ts:Teleport(game.PlaceId, player)
end

-- ============================================================
--  UI
-- ============================================================
local function criarUI()
    if st.ui then return end
    st.ui=true

    gui=Instance.new("ScreenGui") gui.Name="RatHub" gui.ResetOnSpawn=false gui.Parent=player:WaitForChild("PlayerGui")

    main=Instance.new("Frame") main.Size=UDim2.new(0,200,0,280) main.Position=UDim2.new(0.5,-100,0.5,-140)
    main.BackgroundColor3=cor.fundo main.BackgroundTransparency=0.1 main.Active=true main.Draggable=true main.ClipsDescendants=true main.Parent=gui
    Instance.new("UIStroke",main).Color=cor.roxo Instance.new("UIStroke",main).Transparency=0.4 Instance.new("UIStroke",main).Thickness=1.5
    Instance.new("UICorner",main).CornerRadius=UDim.new(0,14)
    local gr=Instance.new("UIGradient",main) gr.Color=ColorSequence.new({ColorSequenceKeypoint.new(0,cor.fundo),ColorSequenceKeypoint.new(0.5,Color3.fromRGB(12,3,30)),ColorSequenceKeypoint.new(1,cor.fundo)}) gr.Rotation=45
    local bordaGlow=Instance.new("UIStroke",main) bordaGlow.Color=cor.ciano bordaGlow.Transparency=0.85 bordaGlow.Thickness=5
    task.spawn(function()local up=true while bordaGlow and bordaGlow.Parent do if up then bordaGlow.Transparency=bordaGlow.Transparency+0.01 if bordaGlow.Transparency>=0.92 then up=false end else bordaGlow.Transparency=bordaGlow.Transparency-0.01 if bordaGlow.Transparency<=0.78 then up=true end end task.wait(0.03)end end)

    local hd=Instance.new("Frame",main) hd.Size=UDim2.new(1,0,0,34) hd.BackgroundColor3=cor.roxoE hd.BackgroundTransparency=0.15
    Instance.new("UICorner",hd).CornerRadius=UDim.new(0,14,0,0)
    local hdGrad=Instance.new("UIGradient",hd) hdGrad.Color=ColorSequence.new({ColorSequenceKeypoint.new(0,cor.roxoE),ColorSequenceKeypoint.new(0.5,cor.roxo),ColorSequenceKeypoint.new(1,cor.roxoE)}) hdGrad.Rotation=90
    local tt=Instance.new("TextLabel",hd) tt.Size=UDim2.new(1,-65,1,0) tt.Position=UDim2.new(0,8,0,0)
    tt.BackgroundTransparency=1 tt.Text="APEX HUB" tt.TextColor3=cor.branco tt.TextSize=13 tt.Font=Enum.Font.GothamBlack tt.TextXAlignment=Enum.TextXAlignment.Left tt.TextYAlignment=Enum.TextYAlignment.Center
    local ttStroke=Instance.new("UIStroke",tt) ttStroke.Color=Color3.fromRGB(0,0,0) ttStroke.Thickness=2.5 ttStroke.Transparency=0.2
    local btnCfg=Instance.new("TextButton",hd) btnCfg.Size=UDim2.new(0,20,0,20) btnCfg.Position=UDim2.new(1,-50,0.5,-10)
    btnCfg.BackgroundTransparency=1 btnCfg.Text="c" btnCfg.TextColor3=cor.cinza btnCfg.TextSize=14 btnCfg.Font=Enum.Font.GothamBold
    local btnMin=Instance.new("TextButton",hd) btnMin.Size=UDim2.new(0,20,0,20) btnMin.Position=UDim2.new(1,-24,0.5,-10)
    btnMin.BackgroundTransparency=1 btnMin.Text="-" btnMin.TextColor3=cor.branco btnMin.TextSize=16 btnMin.Font=Enum.Font.GothamBold

    local barraAbas=Instance.new("Frame",main) barraAbas.Size=UDim2.new(1,0,0,26) barraAbas.Position=UDim2.new(0,0,0,34) barraAbas.BackgroundTransparency=1
    local abaHome=Instance.new("TextButton",barraAbas) abaHome.Size=UDim2.new(0.33,-2,0,20) abaHome.Position=UDim2.new(0.01,0,0,3)
    abaHome.BackgroundColor3=cor.roxo abaHome.BackgroundTransparency=0.1 abaHome.Text="HOME" abaHome.TextColor3=cor.branco abaHome.TextSize=10 abaHome.Font=Enum.Font.GothamBlack
    Instance.new("UICorner",abaHome).CornerRadius=UDim.new(0,6)
    local abaESP=Instance.new("TextButton",barraAbas) abaESP.Size=UDim2.new(0.33,-2,0,20) abaESP.Position=UDim2.new(0.34,1,0,3)
    abaESP.BackgroundColor3=cor.roxoE abaESP.BackgroundTransparency=0.2 abaESP.Text="ESP" abaESP.TextColor3=cor.branco abaESP.TextSize=10 abaESP.Font=Enum.Font.GothamBlack
    Instance.new("UICorner",abaESP).CornerRadius=UDim.new(0,6)
    local abaPLAYER=Instance.new("TextButton",barraAbas) abaPLAYER.Size=UDim2.new(0.33,-2,0,20) abaPLAYER.Position=UDim2.new(0.67,-1,0,3)
    abaPLAYER.BackgroundColor3=cor.roxoE abaPLAYER.BackgroundTransparency=0.25 abaPLAYER.Text="PLAYER" abaPLAYER.TextColor3=cor.branco abaPLAYER.TextSize=10 abaPLAYER.Font=Enum.Font.GothamBlack
    Instance.new("UICorner",abaPLAYER).CornerRadius=UDim.new(0,6)

    local contentY=34+26; local contentH=280-contentY

    -- HOME
    telaHOME=Instance.new("ScrollingFrame",main) telaHOME.Size=UDim2.new(1,0,0,contentH) telaHOME.Position=UDim2.new(0,0,0,contentY) telaHOME.BackgroundTransparency=1 telaHOME.ScrollBarThickness=3 telaHOME.ScrollBarImageColor3=cor.roxo telaHOME.BorderSizePixel=0
    local layH=Instance.new("UIListLayout",telaHOME) layH.Padding=UDim.new(0,4) layH.HorizontalAlignment=Enum.HorizontalAlignment.Center
    local padH=Instance.new("UIPadding",telaHOME) padH.PaddingTop=UDim.new(0,6) padH.PaddingBottom=UDim.new(0,6)

    sta=Instance.new("TextLabel",telaHOME) sta.Size=UDim2.new(0.9,0,0,18) sta.BackgroundTransparency=1 sta.Text="Nenhum TP" sta.TextColor3=cor.branco sta.TextSize=10 sta.Font=Enum.Font.GothamBold sta.TextStrokeColor3=Color3.fromRGB(0,0,0) sta.TextStrokeTransparency=0.2
    led=Instance.new("Frame",telaHOME) led.Size=UDim2.new(0,8,0,8) led.BackgroundColor3=cor.vermelho led.BorderSizePixel=0
    Instance.new("UICorner",led).CornerRadius=UDim.new(1,0)
    local bb=Instance.new("Frame",telaHOME) bb.Size=UDim2.new(0.85,0,0,4) bb.BackgroundColor3=Color3.fromRGB(20,15,40) bb.BackgroundTransparency=0.3 Instance.new("UICorner",bb).CornerRadius=UDim.new(1,0)
    barra=Instance.new("Frame",bb) barra.Size=UDim2.new(0,0,1,0) barra.BackgroundColor3=cor.roxoC barra.BorderSizePixel=0 Instance.new("UICorner",barra).CornerRadius=UDim.new(1,0)
    saveBtn=Instance.new("TextButton",telaHOME) saveBtn.Size=UDim2.new(0.88,0,0,34) saveBtn.BackgroundColor3=cor.roxo saveBtn.BackgroundTransparency=0.15 saveBtn.Text="SALVAR" saveBtn.TextColor3=cor.branco saveBtn.TextSize=12 saveBtn.Font=Enum.Font.GothamBlack
    Instance.new("UICorner",saveBtn).CornerRadius=UDim.new(0,9) Instance.new("UIStroke",saveBtn).Color=cor.roxoC Instance.new("UIStroke",saveBtn).Transparency=0.6
    tpBtn=Instance.new("TextButton",telaHOME) tpBtn.Size=UDim2.new(0.88,0,0,34) tpBtn.BackgroundColor3=cor.roxoE tpBtn.BackgroundTransparency=0.25 tpBtn.Text="TP BRUTO" tpBtn.TextColor3=cor.branco tpBtn.TextSize=12 tpBtn.Font=Enum.Font.GothamBlack tpBtn.Visible=false
    Instance.new("UICorner",tpBtn).CornerRadius=UDim.new(0,9) Instance.new("UIStroke",tpBtn).Color=cor.roxo Instance.new("UIStroke",tpBtn).Transparency=0.7

    -- PLAYER
    telaPLAYER=Instance.new("ScrollingFrame",main) telaPLAYER.Size=UDim2.new(1,0,0,contentH) telaPLAYER.Position=UDim2.new(0,0,0,contentY) telaPLAYER.BackgroundTransparency=1 telaPLAYER.ScrollBarThickness=3 telaPLAYER.ScrollBarImageColor3=cor.roxo telaPLAYER.BorderSizePixel=0 telaPLAYER.Visible=false
    local layP=Instance.new("UIListLayout",telaPLAYER) layP.Padding=UDim.new(0,5) layP.HorizontalAlignment=Enum.HorizontalAlignment.Center
    local padP=Instance.new("UIPadding",telaPLAYER) padP.PaddingTop=UDim.new(0,8) padP.PaddingBottom=UDim.new(0,6)

    -- SPEED
    local spdLabel=Instance.new("TextLabel",telaPLAYER) spdLabel.Size=UDim2.new(0.88,0,0,14) spdLabel.BackgroundTransparency=1 spdLabel.Text="VELOCIDADE" spdLabel.TextColor3=cor.roxoC spdLabel.TextSize=9 spdLabel.Font=Enum.Font.GothamBold spdLabel.TextStrokeColor3=Color3.fromRGB(0,0,0) spdLabel.TextStrokeTransparency=0.2
    
    local spdFrame=Instance.new("Frame",telaPLAYER) spdFrame.Size=UDim2.new(0.88,0,0,28) spdFrame.BackgroundTransparency=1
    
    local spdInput=Instance.new("TextBox",spdFrame) spdInput.Size=UDim2.new(0.26,0,0,24) spdInput.Position=UDim2.new(0,0,0,2) spdInput.BackgroundColor3=Color3.fromRGB(25,15,45) spdInput.BackgroundTransparency=0.3 spdInput.Text="16" spdInput.TextColor3=cor.branco spdInput.TextSize=10 spdInput.Font=Enum.Font.GothamBold spdInput.PlaceholderColor3=cor.cinza spdInput.ClearTextOnFocus=false spdInput.TextXAlignment=Enum.TextXAlignment.Center
    Instance.new("UICorner",spdInput).CornerRadius=UDim.new(0,5)
    
    local spdOnce=Instance.new("TextButton",spdFrame) spdOnce.Size=UDim2.new(0.34,-2,0,24) spdOnce.Position=UDim2.new(0.3,4,0,2) spdOnce.BackgroundColor3=cor.roxo spdOnce.BackgroundTransparency=0.15 spdOnce.Text="UMA VEZ" spdOnce.TextColor3=cor.branco spdOnce.TextSize=9 spdOnce.Font=Enum.Font.GothamBlack spdOnce.TextStrokeColor3=Color3.fromRGB(0,0,0) spdOnce.TextStrokeTransparency=0.2
    Instance.new("UICorner",spdOnce).CornerRadius=UDim.new(0,6)
    spdOnce.MouseEnter:Connect(function() ts:Create(spdOnce,TweenInfo.new(0.08),{Size=UDim2.new(0.36,-2,0,26),BackgroundColor3=cor.ciano,BackgroundTransparency=0.05}):Play() end)
    spdOnce.MouseLeave:Connect(function() ts:Create(spdOnce,TweenInfo.new(0.1),{Size=UDim2.new(0.34,-2,0,24),BackgroundColor3=cor.roxo,BackgroundTransparency=0.15}):Play() end)
    
    local spdAlwaysBtn=Instance.new("TextButton",spdFrame) spdAlwaysBtn.Size=UDim2.new(0.34,-2,0,24) spdAlwaysBtn.Position=UDim2.new(0.66,6,0,2) spdAlwaysBtn.BackgroundColor3=cor.roxoE spdAlwaysBtn.BackgroundTransparency=0.2 spdAlwaysBtn.Text="SEMPRE" spdAlwaysBtn.TextColor3=cor.branco spdAlwaysBtn.TextSize=9 spdAlwaysBtn.Font=Enum.Font.GothamBlack spdAlwaysBtn.TextStrokeColor3=Color3.fromRGB(0,0,0) spdAlwaysBtn.TextStrokeTransparency=0.2
    Instance.new("UICorner",spdAlwaysBtn).CornerRadius=UDim.new(0,6)
    spdAlwaysBtn.MouseEnter:Connect(function() ts:Create(spdAlwaysBtn,TweenInfo.new(0.08),{Size=UDim2.new(0.36,-2,0,26)}):Play() end)
    spdAlwaysBtn.MouseLeave:Connect(function() ts:Create(spdAlwaysBtn,TweenInfo.new(0.1),{Size=UDim2.new(0.34,-2,0,24)}):Play() end)
    
    spdOnce.MouseButton1Click:Connect(function()
        local v = tonumber(spdInput.Text) or 16
        spdVal = v
        aplicarSpeed(v)
    end)
    spdAlwaysBtn.MouseButton1Click:Connect(function()
        spdAlways = not spdAlways
        spdAlwaysBtn.BackgroundColor3 = spdAlways and cor.verde or cor.roxoE
        spdAlwaysBtn.BackgroundTransparency = spdAlways and 0.15 or 0.2
        local v = tonumber(spdInput.Text) or 16
        spdVal = v
        if spdAlways then aplicarSpeed(v) end
        setupAlways()
    end)

    -- SEPARADOR
    local sepMov=Instance.new("TextLabel",telaPLAYER) sepMov.Size=UDim2.new(0.9,0,0,12) sepMov.BackgroundTransparency=1 sepMov.Text="─ PULO ─" sepMov.TextColor3=cor.branco sepMov.TextSize=8 sepMov.Font=Enum.Font.GothamBold sepMov.TextTransparency=0.2 sepMov.TextStrokeColor3=Color3.fromRGB(0,0,0) sepMov.TextStrokeTransparency=0.2
    
    local jmpFrame=Instance.new("Frame",telaPLAYER) jmpFrame.Size=UDim2.new(0.88,0,0,28) jmpFrame.BackgroundTransparency=1
    
    local jmpInput=Instance.new("TextBox",jmpFrame) jmpInput.Size=UDim2.new(0.26,0,0,24) jmpInput.Position=UDim2.new(0,0,0,2) jmpInput.BackgroundColor3=Color3.fromRGB(25,15,45) jmpInput.BackgroundTransparency=0.3 jmpInput.Text="50" jmpInput.TextColor3=cor.branco jmpInput.TextSize=10 jmpInput.Font=Enum.Font.GothamBold jmpInput.PlaceholderColor3=cor.cinza jmpInput.ClearTextOnFocus=false jmpInput.TextXAlignment=Enum.TextXAlignment.Center
    Instance.new("UICorner",jmpInput).CornerRadius=UDim.new(0,5)
    
    local jmpOnce=Instance.new("TextButton",jmpFrame) jmpOnce.Size=UDim2.new(0.34,-2,0,24) jmpOnce.Position=UDim2.new(0.3,4,0,2) jmpOnce.BackgroundColor3=cor.roxo jmpOnce.BackgroundTransparency=0.15 jmpOnce.Text="UMA VEZ" jmpOnce.TextColor3=cor.branco jmpOnce.TextSize=9 jmpOnce.Font=Enum.Font.GothamBlack jmpOnce.TextStrokeColor3=Color3.fromRGB(0,0,0) jmpOnce.TextStrokeTransparency=0.2
    Instance.new("UICorner",jmpOnce).CornerRadius=UDim.new(0,6)
    jmpOnce.MouseEnter:Connect(function() ts:Create(jmpOnce,TweenInfo.new(0.08),{Size=UDim2.new(0.36,-2,0,26),BackgroundColor3=cor.ciano,BackgroundTransparency=0.05}):Play() end)
    jmpOnce.MouseLeave:Connect(function() ts:Create(jmpOnce,TweenInfo.new(0.1),{Size=UDim2.new(0.34,-2,0,24),BackgroundColor3=cor.roxo,BackgroundTransparency=0.15}):Play() end)
    
    local jmpAlwaysBtn=Instance.new("TextButton",jmpFrame) jmpAlwaysBtn.Size=UDim2.new(0.34,-2,0,24) jmpAlwaysBtn.Position=UDim2.new(0.66,6,0,2) jmpAlwaysBtn.BackgroundColor3=cor.roxoE jmpAlwaysBtn.BackgroundTransparency=0.2 jmpAlwaysBtn.Text="SEMPRE" jmpAlwaysBtn.TextColor3=cor.branco jmpAlwaysBtn.TextSize=9 jmpAlwaysBtn.Font=Enum.Font.GothamBlack jmpAlwaysBtn.TextStrokeColor3=Color3.fromRGB(0,0,0) jmpAlwaysBtn.TextStrokeTransparency=0.2
    Instance.new("UICorner",jmpAlwaysBtn).CornerRadius=UDim.new(0,6)
    jmpAlwaysBtn.MouseEnter:Connect(function() ts:Create(jmpAlwaysBtn,TweenInfo.new(0.08),{Size=UDim2.new(0.36,-2,0,26)}):Play() end)
    jmpAlwaysBtn.MouseLeave:Connect(function() ts:Create(jmpAlwaysBtn,TweenInfo.new(0.1),{Size=UDim2.new(0.34,-2,0,24)}):Play() end)
    
    jmpOnce.MouseButton1Click:Connect(function()
        local v = tonumber(jmpInput.Text) or 50
        jmpVal = v
        aplicarJump(v)
    end)
    jmpAlwaysBtn.MouseButton1Click:Connect(function()
        jmpAlways = not jmpAlways
        jmpAlwaysBtn.BackgroundColor3 = jmpAlways and cor.verde or cor.roxoE
        jmpAlwaysBtn.BackgroundTransparency = jmpAlways and 0.15 or 0.2
        local v = tonumber(jmpInput.Text) or 50
        jmpVal = v
        if jmpAlways then aplicarJump(v) end
        setupAlways()
    end)

    -- SEPARADOR
    local sepNoc=Instance.new("TextLabel",telaPLAYER) sepNoc.Size=UDim2.new(0.9,0,0,12) sepNoc.BackgroundTransparency=1 sepNoc.Text="─ NOCLIP ─" sepNoc.TextColor3=cor.branco sepNoc.TextSize=8 sepNoc.Font=Enum.Font.GothamBold sepNoc.TextTransparency=0.2 sepNoc.TextStrokeColor3=Color3.fromRGB(0,0,0) sepNoc.TextStrokeTransparency=0.2

    -- NOCLIP
    local nocFrame=Instance.new("Frame",telaPLAYER) nocFrame.Size=UDim2.new(0.88,0,0,28) nocFrame.BackgroundTransparency=1
    
    local nocAtv=Instance.new("TextButton",nocFrame) nocAtv.Size=UDim2.new(0.5,-2,0,24) nocAtv.Position=UDim2.new(0,0,0,2) nocAtv.BackgroundColor3=cor.vermelho nocAtv.BackgroundTransparency=0.15 nocAtv.Text="NOCLIP" nocAtv.TextColor3=cor.branco nocAtv.TextSize=9 nocAtv.Font=Enum.Font.GothamBlack nocAtv.TextStrokeColor3=Color3.fromRGB(0,0,0) nocAtv.TextStrokeTransparency=0.2
    Instance.new("UICorner",nocAtv).CornerRadius=UDim.new(0,6)
    nocAtv.MouseEnter:Connect(function() ts:Create(nocAtv,TweenInfo.new(0.08),{Size=UDim2.new(0.52,-2,0,26),BackgroundColor3=cor.ciano,BackgroundTransparency=0.05}):Play() end)
    nocAtv.MouseLeave:Connect(function() ts:Create(nocAtv,TweenInfo.new(0.1),{Size=UDim2.new(0.5,-2,0,24),BackgroundColor3=noclipOn and cor.verde or cor.vermelho,BackgroundTransparency=0.15}):Play() end)
    
    local nocAlwaysBtn=Instance.new("TextButton",nocFrame) nocAlwaysBtn.Size=UDim2.new(0.46,-2,0,24) nocAlwaysBtn.Position=UDim2.new(0.54,4,0,2) nocAlwaysBtn.BackgroundColor3=cor.roxoE nocAlwaysBtn.BackgroundTransparency=0.2 nocAlwaysBtn.Text="SEMPRE" nocAlwaysBtn.TextColor3=cor.branco nocAlwaysBtn.TextSize=9 nocAlwaysBtn.Font=Enum.Font.GothamBlack nocAlwaysBtn.TextStrokeColor3=Color3.fromRGB(0,0,0) nocAlwaysBtn.TextStrokeTransparency=0.2
    Instance.new("UICorner",nocAlwaysBtn).CornerRadius=UDim.new(0,6)
    nocAlwaysBtn.MouseEnter:Connect(function() ts:Create(nocAlwaysBtn,TweenInfo.new(0.08),{Size=UDim2.new(0.48,-2,0,26)}):Play() end)
    nocAlwaysBtn.MouseLeave:Connect(function() ts:Create(nocAlwaysBtn,TweenInfo.new(0.1),{Size=UDim2.new(0.46,-2,0,24)}):Play() end)
    
    nocAtv.MouseButton1Click:Connect(function()
        if not noclipOn then
            toggleNoclip(true)
            nocAtv.BackgroundColor3 = cor.verde; nocAtv.Text = "NOCLIP ON"
        else
            toggleNoclip(false)
            nocAtv.BackgroundColor3 = cor.vermelho; nocAtv.Text = "NOCLIP"
        end
    end)
    nocAlwaysBtn.MouseButton1Click:Connect(function()
        noclipAlways = not noclipAlways
        nocAlwaysBtn.BackgroundColor3 = noclipAlways and cor.verde or cor.roxoE
        nocAlwaysBtn.BackgroundTransparency = noclipAlways and 0.15 or 0.2
        setupAlways()
    end)

    -- SEPARADOR EXTRAS
    local sepP=Instance.new("TextLabel",telaPLAYER) sepP.Size=UDim2.new(0.9,0,0,12) sepP.BackgroundTransparency=1 sepP.Text="─ EXTRAS ─" sepP.TextColor3=cor.branco sepP.TextSize=8 sepP.Font=Enum.Font.GothamBold sepP.TextTransparency=0.2 sepP.TextStrokeColor3=Color3.fromRGB(0,0,0) sepP.TextStrokeTransparency=0.2

    -- ANTI AFK
    local afkBtn=Instance.new("TextButton",telaPLAYER) afkBtn.Size=UDim2.new(0.88,0,0,28) afkBtn.BackgroundColor3=cor.roxoE afkBtn.BackgroundTransparency=0.2 afkBtn.Text="ANTI AFK" afkBtn.TextColor3=cor.branco afkBtn.TextSize=9 afkBtn.Font=Enum.Font.GothamBlack afkBtn.TextStrokeColor3=Color3.fromRGB(0,0,0) afkBtn.TextStrokeTransparency=0.2
    Instance.new("UICorner",afkBtn).CornerRadius=UDim.new(0,6)
    afkBtn.MouseEnter:Connect(function() ts:Create(afkBtn,TweenInfo.new(0.08),{Size=UDim2.new(0.9,0,0,30),BackgroundColor3=cor.ciano,BackgroundTransparency=0.1}):Play() end)
    afkBtn.MouseLeave:Connect(function() ts:Create(afkBtn,TweenInfo.new(0.1),{Size=UDim2.new(0.88,0,0,28),BackgroundColor3=afkOn and cor.verde or cor.roxoE,BackgroundTransparency=0.2}):Play() end)
    afkBtn.MouseButton1Click:Connect(function() toggleAFK(not afkOn); afkBtn.BackgroundColor3=afkOn and cor.verde or cor.roxoE; afkBtn.Text=afkOn and "ANTI AFK ON" or "ANTI AFK" end)

    -- SPIN BOT
    local spinBtn=Instance.new("TextButton",telaPLAYER) spinBtn.Size=UDim2.new(0.88,0,0,28) spinBtn.BackgroundColor3=cor.roxoE spinBtn.BackgroundTransparency=0.2 spinBtn.Text="SPIN BOT" spinBtn.TextColor3=cor.branco spinBtn.TextSize=9 spinBtn.Font=Enum.Font.GothamBlack spinBtn.TextStrokeColor3=Color3.fromRGB(0,0,0) spinBtn.TextStrokeTransparency=0.2
    Instance.new("UICorner",spinBtn).CornerRadius=UDim.new(0,6)
    spinBtn.MouseEnter:Connect(function() ts:Create(spinBtn,TweenInfo.new(0.08),{Size=UDim2.new(0.9,0,0,30),BackgroundColor3=cor.ciano,BackgroundTransparency=0.1}):Play() end)
    spinBtn.MouseLeave:Connect(function() ts:Create(spinBtn,TweenInfo.new(0.1),{Size=UDim2.new(0.88,0,0,28),BackgroundColor3=spinOn and cor.verde or cor.roxoE,BackgroundTransparency=0.2}):Play() end)
    spinBtn.MouseButton1Click:Connect(function() toggleSpin(not spinOn); spinBtn.BackgroundColor3=spinOn and cor.verde or cor.roxoE; spinBtn.Text=spinOn and "SPIN ON" or "SPIN BOT" end)

    -- SERVER HOP
    local shBtn=Instance.new("TextButton",telaPLAYER) shBtn.Size=UDim2.new(0.88,0,0,28) shBtn.BackgroundColor3=cor.roxo shBtn.BackgroundTransparency=0.15 shBtn.Text="SERVER HOP" shBtn.TextColor3=cor.branco shBtn.TextSize=9 shBtn.Font=Enum.Font.GothamBlack shBtn.TextStrokeColor3=Color3.fromRGB(0,0,0) shBtn.TextStrokeTransparency=0.2
    Instance.new("UICorner",shBtn).CornerRadius=UDim.new(0,6)
    shBtn.MouseEnter:Connect(function() ts:Create(shBtn,TweenInfo.new(0.08),{Size=UDim2.new(0.9,0,0,30),BackgroundColor3=cor.ciano,BackgroundTransparency=0.05}):Play() end)
    shBtn.MouseLeave:Connect(function() ts:Create(shBtn,TweenInfo.new(0.1),{Size=UDim2.new(0.88,0,0,28),BackgroundColor3=cor.roxo,BackgroundTransparency=0.15}):Play() end)
    shBtn.MouseButton1Click:Connect(serverHop)

    -- REJOIN
    local rjBtn=Instance.new("TextButton",telaPLAYER) rjBtn.Size=UDim2.new(0.88,0,0,28) rjBtn.BackgroundColor3=cor.roxo rjBtn.BackgroundTransparency=0.15 rjBtn.Text="REJOIN" rjBtn.TextColor3=cor.branco rjBtn.TextSize=9 rjBtn.Font=Enum.Font.GothamBlack rjBtn.TextStrokeColor3=Color3.fromRGB(0,0,0) rjBtn.TextStrokeTransparency=0.2
    Instance.new("UICorner",rjBtn).CornerRadius=UDim.new(0,6)
    rjBtn.MouseEnter:Connect(function() ts:Create(rjBtn,TweenInfo.new(0.08),{Size=UDim2.new(0.9,0,0,30),BackgroundColor3=cor.ciano,BackgroundTransparency=0.05}):Play() end)
    rjBtn.MouseLeave:Connect(function() ts:Create(rjBtn,TweenInfo.new(0.1),{Size=UDim2.new(0.88,0,0,28),BackgroundColor3=cor.roxo,BackgroundTransparency=0.15}):Play() end)
    rjBtn.MouseButton1Click:Connect(rejoin)


    -- ESP
    telaESP=Instance.new("ScrollingFrame",main) telaESP.Size=UDim2.new(1,0,0,contentH) telaESP.Position=UDim2.new(0,0,0,contentY) telaESP.BackgroundTransparency=1 telaESP.ScrollBarThickness=3 telaESP.ScrollBarImageColor3=cor.roxo telaESP.BorderSizePixel=0 telaESP.Visible=false
    local layE=Instance.new("UIListLayout",telaESP) layE.Padding=UDim.new(0,4) layE.HorizontalAlignment=Enum.HorizontalAlignment.Center
    local padE=Instance.new("UIPadding",telaESP) padE.PaddingTop=UDim.new(0,5) padE.PaddingBottom=UDim.new(0,6)

    -- ESP ATIVAR
    local btnAtv=Instance.new("TextButton",telaESP) btnAtv.Size=UDim2.new(0.88,0,0,34) btnAtv.BackgroundColor3=cor.vermelho btnAtv.BackgroundTransparency=0.15 btnAtv.Text="ESP ATIVAR" btnAtv.TextColor3=cor.branco btnAtv.TextSize=13 btnAtv.Font=Enum.Font.GothamBlack btnAtv.TextStrokeColor3=Color3.fromRGB(0,0,0) btnAtv.TextStrokeTransparency=0.15
    Instance.new("UICorner",btnAtv).CornerRadius=UDim.new(0,9) Instance.new("UIStroke",btnAtv).Color=Color3.fromRGB(0,0,0) Instance.new("UIStroke",btnAtv).Transparency=0.3 Instance.new("UIStroke",btnAtv).Thickness=1.5
    local atvGlow=Instance.new("UIStroke",btnAtv) atvGlow.Color=cor.ciano atvGlow.Thickness=4 atvGlow.Transparency=1
    btnAtv.MouseEnter:Connect(function() ts:Create(btnAtv,TweenInfo.new(0.1),{Size=UDim2.new(0.9,0,0,32),BackgroundTransparency=0.05}):Play() end)
    btnAtv.MouseLeave:Connect(function() ts:Create(btnAtv,TweenInfo.new(0.1),{Size=UDim2.new(0.88,0,0,34),BackgroundTransparency=esp.ativo and 0.15 or 0.15}):Play() end)

    local sep1=Instance.new("TextLabel",telaESP) sep1.Size=UDim2.new(0.9,0,0,12) sep1.BackgroundTransparency=1 sep1.Text="─ FUNCOES ─" sep1.TextColor3=cor.branco sep1.TextSize=8 sep1.Font=Enum.Font.GothamBold sep1.TextTransparency=0.2 sep1.TextStrokeColor3=Color3.fromRGB(0,0,0) sep1.TextStrokeTransparency=0.2

    -- Toggles (2 por linha)
    local toggleData = {
        {text="BOX", var="box"}, {text="TRACER CIMA", var="tracerCima"},
        {text="TRACER BAIXO", var="tracerBaixo"}, {text="VIDA BARRA", var="vidaBarra"},
        {text="DISTANCIA", var="dist"}, {text="NOME", var="nome"},
        {text="CHAMS", var="chams"},
    }
    local toggles = {}
    for i=1, #toggleData, 2 do
        local ln=Instance.new("Frame",telaESP) ln.Size=UDim2.new(0.88,0,0,28) ln.BackgroundTransparency=1
        for j=0,1 do
            local idx=i+j; if idx>#toggleData then break end
            local td=toggleData[idx]
            local btn=Instance.new("TextButton",ln) btn.Size=UDim2.new(0.5,-2,0,26) btn.Position=UDim2.new(j*0.5, j*2,0,1)
            btn.BackgroundColor3=cor.roxoE btn.BackgroundTransparency=0.2 btn.Text=td.text btn.TextColor3=cor.branco btn.TextSize=10 btn.Font=Enum.Font.GothamBlack btn.TextStrokeColor3=Color3.fromRGB(0,0,0) btn.TextStrokeTransparency=0.2
            Instance.new("UICorner",btn).CornerRadius=UDim.new(0,6)
            Instance.new("UIStroke",btn).Color=Color3.fromRGB(0,0,0) Instance.new("UIStroke",btn).Transparency=0.4 Instance.new("UIStroke",btn).Thickness=1
            btn.MouseEnter:Connect(function() ts:Create(btn,TweenInfo.new(0.1),{Size=UDim2.new(0.5,-1,0,28),BackgroundTransparency=0.1}):Play() end)
            btn.MouseLeave:Connect(function() ts:Create(btn,TweenInfo.new(0.1),{Size=UDim2.new(0.5,-2,0,26),BackgroundTransparency=esp.ativo and 0.2 or 0.6}):Play() end)
            toggles[td.var]=btn
        end
    end

    local function atualizarToggles()
        for var, btn in pairs(toggles) do
            local on=esp[var]
            btn.BackgroundColor3=on and cor.verde or cor.roxoE
            btn.BackgroundTransparency=esp.ativo and 0.2 or 0.6
            btn.TextTransparency=esp.ativo and 0 or 0.5
        end
    end

    -- COR (50 cores)
    local sep2=Instance.new("TextLabel",telaESP) sep2.Size=UDim2.new(0.9,0,0,12) sep2.BackgroundTransparency=1 sep2.Text="─ COR ─" sep2.TextColor3=cor.branco sep2.TextSize=8 sep2.Font=Enum.Font.GothamBold sep2.TextTransparency=0.2 sep2.TextStrokeColor3=Color3.fromRGB(0,0,0) sep2.TextStrokeTransparency=0.2

    local coresLista = {
        Color3.fromRGB(255,255,255),Color3.fromRGB(200,200,200),Color3.fromRGB(150,150,150),Color3.fromRGB(100,100,100),Color3.fromRGB(50,50,50),
        Color3.fromRGB(255,0,0),Color3.fromRGB(200,0,0),Color3.fromRGB(150,0,0),Color3.fromRGB(255,100,100),Color3.fromRGB(200,50,50),
        Color3.fromRGB(255,150,0),Color3.fromRGB(200,100,0),Color3.fromRGB(255,200,0),Color3.fromRGB(255,255,0),Color3.fromRGB(200,200,0),
        Color3.fromRGB(150,255,0),Color3.fromRGB(100,200,0),Color3.fromRGB(0,255,0),Color3.fromRGB(0,200,0),Color3.fromRGB(0,150,0),
        Color3.fromRGB(0,255,150),Color3.fromRGB(0,200,100),Color3.fromRGB(0,255,200),Color3.fromRGB(0,240,255),Color3.fromRGB(0,200,200),
        Color3.fromRGB(0,150,200),Color3.fromRGB(0,100,255),Color3.fromRGB(0,50,200),Color3.fromRGB(50,100,255),Color3.fromRGB(100,150,255),
        Color3.fromRGB(140,60,255),Color3.fromRGB(100,0,200),Color3.fromRGB(180,100,255),Color3.fromRGB(190,140,255),Color3.fromRGB(200,150,255),
        Color3.fromRGB(255,0,255),Color3.fromRGB(200,0,200),Color3.fromRGB(255,50,180),Color3.fromRGB(200,50,150),Color3.fromRGB(255,100,200),
        Color3.fromRGB(255,150,200),Color3.fromRGB(150,75,0),Color3.fromRGB(200,100,50),Color3.fromRGB(255,215,0),Color3.fromRGB(200,170,0),
        Color3.fromRGB(100,200,255),Color3.fromRGB(50,150,200),Color3.fromRGB(255,100,50),Color3.fromRGB(200,80,40),Color3.fromRGB(0,255,100),
    }
    local corGrid=Instance.new("Frame",telaESP) corGrid.Size=UDim2.new(0.88,0,0,170) corGrid.BackgroundTransparency=1
    for i, c in ipairs(coresLista) do
        local col=(i-1)%5; local row=math.floor((i-1)/5)
        local bt=Instance.new("TextButton",corGrid) bt.Size=UDim2.new(0,34,0,15) bt.Position=UDim2.new(0,col*36,0,row*17) bt.BackgroundColor3=c bt.Text=""
        Instance.new("UICorner",bt).CornerRadius=UDim.new(0,3)
        bt.MouseEnter:Connect(function() ts:Create(bt,TweenInfo.new(0.08),{Size=UDim2.new(0,37,0,17)}):Play() end)
        bt.MouseLeave:Connect(function() ts:Create(bt,TweenInfo.new(0.1),{Size=UDim2.new(0,34,0,15)}):Play() end)
        bt.MouseButton1Click:Connect(function() esp.cor=c; atualizarChams() end)
    end

    -- RGB
    local rgbT=Instance.new("TextLabel",telaESP) rgbT.Size=UDim2.new(0.9,0,0,12) rgbT.BackgroundTransparency=1 rgbT.Text="RGB" rgbT.TextColor3=cor.branco rgbT.TextSize=8 rgbT.Font=Enum.Font.GothamBold rgbT.TextTransparency=0.2 rgbT.TextStrokeColor3=Color3.fromRGB(0,0,0) rgbT.TextStrokeTransparency=0.2
    local rgbF=Instance.new("Frame",telaESP) rgbF.Size=UDim2.new(0.88,0,0,26) rgbF.BackgroundTransparency=1
    local rI=Instance.new("TextBox",rgbF) rI.Size=UDim2.new(0,40,0,22) rI.Position=UDim2.new(0,0,0,2) rI.BackgroundColor3=Color3.fromRGB(25,15,45) rI.BackgroundTransparency=0.3 rI.Text="0" rI.TextColor3=cor.branco rI.TextSize=9 rI.Font=Enum.Font.GothamBold rI.PlaceholderColor3=cor.cinza rI.ClearTextOnFocus=false rI.TextXAlignment=Enum.TextXAlignment.Center
    Instance.new("UICorner",rI).CornerRadius=UDim.new(0,4)
    local gI=Instance.new("TextBox",rgbF) gI.Size=UDim2.new(0,40,0,22) gI.Position=UDim2.new(0,44,0,2) gI.BackgroundColor3=Color3.fromRGB(25,15,45) gI.BackgroundTransparency=0.3 gI.Text="240" gI.TextColor3=cor.branco gI.TextSize=9 gI.Font=Enum.Font.GothamBold gI.PlaceholderColor3=cor.cinza gI.ClearTextOnFocus=false gI.TextXAlignment=Enum.TextXAlignment.Center
    Instance.new("UICorner",gI).CornerRadius=UDim.new(0,4)
    local bI=Instance.new("TextBox",rgbF) bI.Size=UDim2.new(0,40,0,22) bI.Position=UDim2.new(0,88,0,2) bI.BackgroundColor3=Color3.fromRGB(25,15,45) bI.BackgroundTransparency=0.3 bI.Text="255" bI.TextColor3=cor.branco bI.TextSize=9 bI.Font=Enum.Font.GothamBold bI.PlaceholderColor3=cor.cinza bI.ClearTextOnFocus=false bI.TextXAlignment=Enum.TextXAlignment.Center
    Instance.new("UICorner",bI).CornerRadius=UDim.new(0,4)
    local rgbBt=Instance.new("TextButton",rgbF) rgbBt.Size=UDim2.new(0,40,0,22) rgbBt.Position=UDim2.new(0,134,0,2) rgbBt.BackgroundColor3=cor.roxo rgbBt.BackgroundTransparency=0.15 rgbBt.Text=">" rgbBt.TextColor3=cor.branco rgbBt.TextSize=10 rgbBt.Font=Enum.Font.GothamBlack
    Instance.new("UICorner",rgbBt).CornerRadius=UDim.new(0,5)
    rgbBt.MouseEnter:Connect(function() ts:Create(rgbBt,TweenInfo.new(0.08),{Size=UDim2.new(0,44,0,24),BackgroundColor3=cor.ciano,BackgroundTransparency=0.05}):Play() end)
    rgbBt.MouseLeave:Connect(function() ts:Create(rgbBt,TweenInfo.new(0.1),{Size=UDim2.new(0,40,0,22),BackgroundColor3=cor.roxo,BackgroundTransparency=0.15}):Play() end)
    rgbBt.MouseButton1Click:Connect(function()
        local r=math.clamp(tonumber(rI.Text)or 0,0,255); local g=math.clamp(tonumber(gI.Text)or 0,0,255); local b=math.clamp(tonumber(bI.Text)or 0,0,255)
        esp.cor=Color3.fromRGB(r,g,b); atualizarChams()
    end)

    -- SETA PESSOA
    local sep3=Instance.new("TextLabel",telaESP) sep3.Size=UDim2.new(0.9,0,0,12) sep3.BackgroundTransparency=1 sep3.Text="─ SETA PESSOA ─" sep3.TextColor3=cor.branco sep3.TextSize=8 sep3.Font=Enum.Font.GothamBold sep3.TextTransparency=0.2 sep3.TextStrokeColor3=Color3.fromRGB(0,0,0) sep3.TextStrokeTransparency=0.2

    local setaFrame=Instance.new("Frame",telaESP) setaFrame.Size=UDim2.new(0.88,0,0,28) setaFrame.BackgroundTransparency=1

    local nomeSeta=Instance.new("TextBox",setaFrame) nomeSeta.Size=UDim2.new(0.58,0,0,24) nomeSeta.Position=UDim2.new(0,0,0,2) nomeSeta.BackgroundColor3=Color3.fromRGB(25,15,45) nomeSeta.BackgroundTransparency=0.3 nomeSeta.Text="" nomeSeta.TextColor3=cor.branco nomeSeta.TextSize=9 nomeSeta.Font=Enum.Font.Gotham nomeSeta.PlaceholderText="nick do cara" nomeSeta.PlaceholderColor3=cor.cinza nomeSeta.ClearTextOnFocus=false nomeSeta.TextXAlignment=Enum.TextXAlignment.Center
    Instance.new("UICorner",nomeSeta).CornerRadius=UDim.new(0,5)

    local btnSeta=Instance.new("TextButton",setaFrame) btnSeta.Size=UDim2.new(0.38,0,0,24) btnSeta.Position=UDim2.new(0.6,4,0,2) btnSeta.BackgroundColor3=cor.roxo btnSeta.BackgroundTransparency=0.15 btnSeta.Text="SETA PESSOA" btnSeta.TextColor3=cor.branco btnSeta.TextSize=8 btnSeta.Font=Enum.Font.GothamBlack btnSeta.TextStrokeColor3=Color3.fromRGB(0,0,0) btnSeta.TextStrokeTransparency=0.15
    Instance.new("UICorner",btnSeta).CornerRadius=UDim.new(0,5) Instance.new("UIStroke",btnSeta).Color=Color3.fromRGB(0,0,0) Instance.new("UIStroke",btnSeta).Transparency=0.3 Instance.new("UIStroke",btnSeta).Thickness=1.5
    btnSeta.MouseEnter:Connect(function() ts:Create(btnSeta,TweenInfo.new(0.08),{Size=UDim2.new(0.4,0,0,26),BackgroundColor3=cor.ciano,BackgroundTransparency=0.05}):Play() end)
    btnSeta.MouseLeave:Connect(function() ts:Create(btnSeta,TweenInfo.new(0.1),{Size=UDim2.new(0.38,0,0,24),BackgroundColor3=cor.roxo,BackgroundTransparency=0.15}):Play() end)

    local setaStatus=Instance.new("TextLabel",telaESP) setaStatus.Size=UDim2.new(0.88,0,0,14) setaStatus.BackgroundTransparency=1 setaStatus.Text="" setaStatus.TextColor3=cor.cinza setaStatus.TextSize=8 setaStatus.Font=Enum.Font.Gotham setaStatus.TextStrokeColor3=Color3.fromRGB(0,0,0) setaStatus.TextStrokeTransparency=0.2

    btnSeta.MouseButton1Click:Connect(function()
        local texto=nomeSeta.Text:lower():gsub("%s+","")
        if texto=="" then return end
        local achou=nil
        for _, plr in pairs(game.Players:GetPlayers()) do
            if plr==player then continue end
            local n=plr.Name:lower(); local dn=(plr.DisplayName or ""):lower()
            if n:sub(1,#texto)==texto or dn:sub(1,#texto)==texto then achou=plr; break end
        end
        if achou then
            espAlvo=achou
            setaStatus.Text="Seta em: "..achou.Name
            setaStatus.TextColor3=cor.verde
        else
            limparSeta()
            setaStatus.Text="Ninguem encontrado"
            setaStatus.TextColor3=cor.vermelho
        end
    end)

    -- Botao limpar seta
    local btnLimparSeta=Instance.new("TextButton",telaESP) btnLimparSeta.Size=UDim2.new(0.88,0,0,26) btnLimparSeta.BackgroundColor3=cor.roxoE btnLimparSeta.BackgroundTransparency=0.2 btnLimparSeta.Text="LIMPAR SETA" btnLimparSeta.TextColor3=cor.branco btnLimparSeta.TextSize=8 btnLimparSeta.Font=Enum.Font.GothamBlack btnLimparSeta.TextStrokeColor3=Color3.fromRGB(0,0,0) btnLimparSeta.TextStrokeTransparency=0.15
    Instance.new("UICorner",btnLimparSeta).CornerRadius=UDim.new(0,5)
    btnLimparSeta.MouseEnter:Connect(function() ts:Create(btnLimparSeta,TweenInfo.new(0.08),{Size=UDim2.new(0.9,0,0,28),BackgroundColor3=cor.vermelho,BackgroundTransparency=0.1}):Play() end)
    btnLimparSeta.MouseLeave:Connect(function() ts:Create(btnLimparSeta,TweenInfo.new(0.1),{Size=UDim2.new(0.88,0,0,26),BackgroundColor3=cor.roxoE,BackgroundTransparency=0.2}):Play() end)
    btnLimparSeta.MouseButton1Click:Connect(function()
        limparSeta()
        setaStatus.Text=""; nomeSeta.Text=""
    end)

    -- Atualizar canvas
    local function atualizarCanvas(tela)
        local h=0; for _,v in pairs(tela:GetChildren()) do if v:IsA("TextButton")or v:IsA("Frame")or v:IsA("TextLabel") then h=h+v.AbsoluteSize.Y+4 end end
        tela.CanvasSize=UDim2.new(0,0,0,h+10)
    end
    task.spawn(function() atualizarCanvas(telaESP) end)

    -- Eventos ESP
    btnAtv.MouseButton1Click:Connect(function()
        esp.ativo=not esp.ativo
        btnAtv.BackgroundColor3=esp.ativo and cor.verde or cor.vermelho
        btnAtv.Text=esp.ativo and "ESP ATIVO" or "ESP ATIVAR"
        if not esp.ativo then limparSeta(); atvGlow.Transparency=1 end
        if esp.ativo then
            task.spawn(function()
                local up=true
                while esp.ativo and atvGlow and atvGlow.Parent do
                    if up then atvGlow.Transparency=atvGlow.Transparency+0.03 if atvGlow.Transparency>=0.7 then up=false end
                    else atvGlow.Transparency=atvGlow.Transparency-0.03 if atvGlow.Transparency<=0.3 then up=true end end
                    task.wait(0.04)
                end
            end)
        end
        espLigar(); atualizarToggles()
    end)

    for _, td in ipairs(toggleData) do
        toggles[td.var].MouseButton1Click:Connect(function()
            if not esp.ativo then return end
            esp[td.var]=not esp[td.var]
            toggles[td.var].BackgroundColor3=esp[td.var] and cor.verde or cor.roxoE
            if td.var=="chams" then atualizarChams() end
        end)
    end
    atualizarToggles()

    -- CONFIG
    telaConfig=Instance.new("ScrollingFrame",main) telaConfig.Size=UDim2.new(1,0,0,contentH) telaConfig.Position=UDim2.new(0,0,0,contentY) telaConfig.BackgroundTransparency=1 telaConfig.ScrollBarThickness=3 telaConfig.ScrollBarImageColor3=cor.roxo telaConfig.BorderSizePixel=0 telaConfig.Visible=false
    local layC=Instance.new("UIListLayout",telaConfig) layC.Padding=UDim.new(0,4) layC.HorizontalAlignment=Enum.HorizontalAlignment.Center
    local padC=Instance.new("UIPadding",telaConfig) padC.PaddingTop=UDim.new(0,6) padC.PaddingBottom=UDim.new(0,6)
    local cfgH=Instance.new("TextLabel",telaConfig) cfgH.Size=UDim2.new(0.9,0,0,14) cfgH.BackgroundTransparency=1 cfgH.Text="CONFIG" cfgH.TextColor3=cor.roxoC cfgH.TextSize=10 cfgH.Font=Enum.Font.GothamBlack
    local inpF=Instance.new("Frame") inpF.Size=UDim2.new(0.9,0,0,22) inpF.BackgroundColor3=Color3.fromRGB(20,15,35) inpF.BackgroundTransparency=0.3 inpF.Parent=telaConfig
    Instance.new("UICorner",inpF).CornerRadius=UDim.new(0,6)
    local nInput=Instance.new("TextBox",inpF) nInput.Size=UDim2.new(0.55,0,1,0) nInput.Position=UDim2.new(0,5,0,0) nInput.BackgroundTransparency=1
    nInput.Text="" nInput.TextColor3=cor.branco nInput.TextSize=9 nInput.Font=Enum.Font.Gotham nInput.TextXAlignment=Enum.TextXAlignment.Left nInput.PlaceholderText="nome" nInput.PlaceholderColor3=cor.cinza nInput.ClearTextOnFocus=false
    local svCfg=Instance.new("TextButton",inpF) svCfg.Size=UDim2.new(0.35,0,0.8,0) svCfg.Position=UDim2.new(0.6,2,0.5,-9) svCfg.BackgroundColor3=cor.roxo svCfg.BackgroundTransparency=0.15 svCfg.Text="SALVAR" svCfg.TextColor3=cor.branco svCfg.TextSize=8 svCfg.Font=Enum.Font.GothamBlack
    Instance.new("UICorner",svCfg).CornerRadius=UDim.new(0,6)
    local function atualizarConfigs()
        for _,v in pairs(telaConfig:GetChildren()) do if v:IsA("Frame") and v~=inpF then v:Destroy() end end
        for idx,cfg in pairs(st.configs) do
            local item=Instance.new("Frame") item.Size=UDim2.new(0.92,0,0,28) item.BackgroundColor3=Color3.fromRGB(16,10,32) item.BackgroundTransparency=0.2 item.Parent=telaConfig
            Instance.new("UICorner",item).CornerRadius=UDim.new(0,7)
            local nome=Instance.new("TextBox",item) nome.Size=UDim2.new(0.45,0,1,0) nome.Position=UDim2.new(0,6,0,0) nome.BackgroundTransparency=1
            nome.Text=cfg.nome nome.TextColor3=cor.branco nome.TextSize=9 nome.Font=Enum.Font.GothamBold nome.TextXAlignment=Enum.TextXAlignment.Left nome.ClearTextOnFocus=false
            nome.FocusLost:Connect(function() cfg.nome=nome.Text end)
            local lBtn=Instance.new("TextButton",item) lBtn.Size=UDim2.new(0,34,0,20) lBtn.Position=UDim2.new(0.55,0,0.5,-10) lBtn.BackgroundColor3=cor.ciano lBtn.BackgroundTransparency=0.1 lBtn.Text="TP" lBtn.TextColor3=Color3.fromRGB(0,0,0) lBtn.TextSize=9 lBtn.Font=Enum.Font.GothamBlack
            Instance.new("UICorner",lBtn).CornerRadius=UDim.new(0,6)
            lBtn.MouseButton1Click:Connect(function() tp(cfg.pos) end)
            local dBtn=Instance.new("TextButton",item) dBtn.Size=UDim2.new(0,16,0,16) dBtn.Position=UDim2.new(0.82,0,0.5,-8) dBtn.BackgroundColor3=cor.vermelho dBtn.BackgroundTransparency=0.2 dBtn.Text="X" dBtn.TextColor3=cor.branco dBtn.TextSize=8 dBtn.Font=Enum.Font.GothamBlack
            Instance.new("UICorner",dBtn).CornerRadius=UDim.new(0,5)
            dBtn.MouseButton1Click:Connect(function() table.remove(st.configs,idx) atualizarConfigs() end)
        end
        local h=0; for _,v in pairs(telaConfig:GetChildren()) do if v:IsA("Frame") then h=h+v.AbsoluteSize.Y+4 end end
        telaConfig.CanvasSize=UDim2.new(0,0,0,math.max(h+40,50))
    end
    svCfg.MouseButton1Click:Connect(function()
        if not st.pos then return end
        local nome=nInput.Text if nome==""then nome="Config "..#st.configs+1 end
        table.insert(st.configs,{nome=nome,pos=st.pos}) nInput.Text="" atualizarConfigs()
    end)

    -- NAVEGACAO
    local function selecionarAba(aba)
        telaHOME.Visible=(aba=="home"); telaESP.Visible=(aba=="esp"); telaPLAYER.Visible=(aba=="player"); telaConfig.Visible=false; st.cfgTela=false
        abaHome.BackgroundColor3=(aba=="home") and cor.roxo or cor.roxoE; abaHome.BackgroundTransparency=(aba=="home") and 0.1 or 0.25
        abaPLAYER.BackgroundColor3=(aba=="player") and cor.roxo or cor.roxoE; abaPLAYER.BackgroundTransparency=(aba=="player") and 0.1 or 0.25
        abaESP.BackgroundColor3=(aba=="esp") and cor.roxo or cor.roxoE; abaESP.BackgroundTransparency=(aba=="esp") and 0.1 or 0.25
    end
    abaHome.MouseButton1Click:Connect(function() selecionarAba("home") end)
    abaPLAYER.MouseButton1Click:Connect(function() selecionarAba("player") end)
    abaESP.MouseButton1Click:Connect(function() selecionarAba("esp") end)
    btnCfg.MouseButton1Click:Connect(function()
        st.cfgTela=not st.cfgTela
        if st.cfgTela then telaHOME.Visible=false; telaPLAYER.Visible=false; telaESP.Visible=false; telaConfig.Visible=true; atualizarConfigs()
        else selecionarAba("home") end
    end)
    btnMin.MouseButton1Click:Connect(function()
        st.min=not st.min; btnMin.Text=st.min and "+"or"-"
        ts:Create(main,TweenInfo.new(0.2),{Size=UDim2.new(0,200,0,st.min and 34 or 280)}):Play()
    end)

    -- NOTIFICACAO
    notif=Instance.new("Frame") notif.Size=UDim2.new(0,220,0,95) notif.Position=UDim2.new(0.5,-110,0,-105)
    notif.BackgroundColor3=Color3.fromRGB(12,8,24) notif.BackgroundTransparency=0.05 notif.BorderSizePixel=0 notif.Visible=false notif.Parent=gui
    Instance.new("UICorner",notif).CornerRadius=UDim.new(0,20)
    local ni=Instance.new("TextLabel",notif) ni.Size=UDim2.new(0,26,0,26) ni.Position=UDim2.new(0,12,0.3,-13) ni.BackgroundTransparency=1 ni.Text="@" ni.TextSize=22 ni.TextColor3=cor.roxoC
    local nt=Instance.new("TextLabel",notif) nt.Size=UDim2.new(1,-50,0,18) nt.Position=UDim2.new(0,44,0,8) nt.BackgroundTransparency=1 nt.Text="Marcar Posicao" nt.TextColor3=cor.branco nt.TextSize=12 nt.Font=Enum.Font.GothamBlack nt.TextXAlignment=Enum.TextXAlignment.Left
    local ns=Instance.new("TextLabel",notif) ns.Size=UDim2.new(1,-50,0,14) ns.Position=UDim2.new(0,44,0,28) ns.BackgroundTransparency=1 ns.Text="Escolha como marcar" ns.TextColor3=cor.cinza ns.TextSize=9 ns.Font=Enum.Font.Gotham ns.TextXAlignment=Enum.TextXAlignment.Left
    ba=Instance.new("TextButton",notif) ba.Size=UDim2.new(0,92,0,28) ba.Position=UDim2.new(0.05,0,1,-36)
    ba.BackgroundColor3=cor.verde ba.BackgroundTransparency=0.12 ba.Text="AGORA" ba.TextColor3=cor.branco ba.TextSize=11 ba.Font=Enum.Font.GothamBlack Instance.new("UICorner",ba).CornerRadius=UDim.new(0,14)
    bd=Instance.new("TextButton",notif) bd.Size=UDim2.new(0,92,0,28) bd.Position=UDim2.new(0.52,0,1,-36)
    bd.BackgroundColor3=cor.amarelo bd.BackgroundTransparency=0.12 bd.Text="A DEDO" bd.TextColor3=cor.branco bd.TextSize=11 bd.Font=Enum.Font.GothamBlack Instance.new("UICorner",bd).CornerRadius=UDim.new(0,14)
    local function sN()notif.Visible=true ts:Create(notif,TweenInfo.new(0.25,Enum.EasingStyle.Back),{Position=UDim2.new(0.5,-110,0,30)}):Play()end
    local function hN()ts:Create(notif,TweenInfo.new(0.2,Enum.EasingStyle.Back),{Position=UDim2.new(0.5,-110,0,-105)}):Play()task.delay(0.2,function()notif.Visible=false end)end
    ba.MouseButton1Click:Connect(function()hN()local c=player.Character if c then local h=c:FindFirstChild("HumanoidRootPart")if h then marcar(h.Position)end end end)
    bd.MouseButton1Click:Connect(function()hN()st.dedo=true sta.Text="Toque no chao"sta.TextColor3=cor.amarelo led.BackgroundColor3=cor.amarelo
        if st.dc then st.dc:Disconnect()end
        st.dc=uis.InputBegan:Connect(function(i)
            if not st.dedo then return end
            local m if i.UserInputType==Enum.UserInputType.Touch then m=i.Position elseif i.UserInputType==Enum.UserInputType.MouseButton1 then m=uis:GetMouseLocation()end
            if not m then return end
            local r=Camera:ViewportPointToRay(m.X,m.Y) local p=RaycastParams.new() p.FilterType=Enum.RaycastFilterType.Blacklist p.FilterDescendantsInstances={player.Character}
            local res=workspace:Raycast(r.Origin,r.Direction*500,p)
            if res and player.Character and player.Character:FindFirstChild("HumanoidRootPart")then
                local pos=res.Position if pos.Y<player.Character.HumanoidRootPart.Position.Y+5 then
                    st.dedo=false if st.dc then st.dc:Disconnect()end marcar(pos)sta.Text="Marcado a dedo!"sta.TextColor3=cor.verde
                    task.delay(0.8,function()sta.Text="TP pronto"sta.TextColor3=cor.cinza led.BackgroundColor3=cor.roxo end)
                end
            end
        end)
    end)
    saveBtn.MouseButton1Click:Connect(sN)
    tpBtn.MouseButton1Click:Connect(function()if st.pos and not st.tp then tp(st.pos)end end)
    player.CharacterAdded:Connect(function()
        task.wait(0.5) if sta then sta.Text="Respawnado!"sta.TextColor3=cor.amarelo led.BackgroundColor3=cor.amarelo end
        task.delay(1.5,function()
            if st.pos then sta.Text="TP pronto"sta.TextColor3=cor.cinza led.BackgroundColor3=cor.roxo tpBtn.Visible=true
            else sta.Text="Nenhum TP"sta.TextColor3=cor.cinza led.BackgroundColor3=cor.vermelho end
        end)
    end)
    selecionarAba("home")
end

player.Chatted:Connect(function(m)
    if m=="root"and not st.ui then criarUI()end
    if m=="bypass"and st.ui then esp.ativo=false; espLigar(); if gui then gui.Parent=nil end; if st.marker and st.marker.Parent then st.marker.Parent=nil end end
    if m=="root"and st.ui and gui then gui.Parent=player:WaitForChild("PlayerGui")if st.marker then st.marker.Parent=workspace end end
end)
