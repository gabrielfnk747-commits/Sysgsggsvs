-- CONFIGURAÇÕES
local player = game.Players.LocalPlayer
local char = player.Character or player.CharacterAdded:Wait()
local hrp = char:WaitForChild("HumanoidRootPart")
local humanoid = char:WaitForChild("Humanoid")
local Debris = game:GetService("Debris")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

-- ===================== HABILIDADE LINHA/VOO =====================
local LineColor = Color3.fromRGB(0,0,0)
local SoundId = "rbxassetid://168146623"
local SoundDuration = 3
local Speed = 80
local NormalWalkSpeed = 16
local BoostWalkSpeed = 120

local flying = false
local targetPos = nil
local line
local bodyVelocity
local abilityActive = false

-- Item na cintura (Active Gear)
local cinturaItem = nil

-- ===================== TRANSFORMAÇÃO =====================
local audioTransformacao = "rbxassetid://6956875383"
local audioEletrico = "rbxassetid://71157925730235"
local audioGrito = "rbxassetid://7182253058"
local audioPisao = "rbxassetid://91082497920871"
local audioMusica = "rbxassetid://1835336262"

local transformCooldown = 3
local actionCooldown = 3
local transformDelay = 6
local lastGrito = 0
local lastSoco = 0
local lastVoltar = 0
local lastTransform = 0

-- Titan scale
local titanScale = 5

-- ===================== GUI =====================
local screenGui = player:WaitForChild("PlayerGui"):FindFirstChild("ActiveGearGUI")
if not screenGui then
    screenGui = Instance.new("ScreenGui")
    screenGui.Name = "ActiveGearGUI"
    screenGui.Parent = player:WaitForChild("PlayerGui")
end

-- BOTÃO ACTIVE GEAR
local buttonActiveGear = screenGui:FindFirstChild("ActiveGearButton")
if not buttonActiveGear then
    buttonActiveGear = Instance.new("TextButton")
    buttonActiveGear.Name = "ActiveGearButton"
    buttonActiveGear.Size = UDim2.new(0,120,0,40)
    buttonActiveGear.Position = UDim2.new(0.75,0,0.55,0)
    buttonActiveGear.BackgroundColor3 = Color3.fromRGB(0,0,0)
    buttonActiveGear.TextColor3 = Color3.fromRGB(255,255,255)
    buttonActiveGear.Text = "Active Gear"
    buttonActiveGear.Font = Enum.Font.SourceSansBold
    buttonActiveGear.TextScaled = true
    buttonActiveGear.Parent = screenGui
end

-- BOTÃO TRANSFORMAR (igual Active Gear, só mais acima)
local buttonTransform = Instance.new("TextButton")
buttonTransform.Name = "TransformButton"
buttonTransform.Size = buttonActiveGear.Size
buttonTransform.BackgroundColor3 = buttonActiveGear.BackgroundColor3
buttonTransform.TextColor3 = buttonActiveGear.TextColor3
buttonTransform.Font = buttonActiveGear.Font
buttonTransform.TextScaled = buttonActiveGear.TextScaled
buttonTransform.Text = "Transformar"
buttonTransform.Position = buttonActiveGear.Position - UDim2.new(0,0,0.08,0)
buttonTransform.Parent = screenGui

-- BOTÕES DE TRANSFORMAÇÃO
local buttonVoltar = Instance.new("TextButton")
buttonVoltar.Size = UDim2.new(0,120,0,40)
buttonVoltar.Position = UDim2.new(0.2,0,0.5,0)
buttonVoltar.BackgroundColor3 = Color3.fromRGB(0,0,255)
buttonVoltar.TextColor3 = Color3.fromRGB(255,255,255)
buttonVoltar.Text = "Voltar"
buttonVoltar.Font = Enum.Font.SourceSansBold
buttonVoltar.TextScaled = true
buttonVoltar.Visible = false
buttonVoltar.Parent = screenGui

local buttonGritar = Instance.new("TextButton")
buttonGritar.Size = UDim2.new(0,120,0,40)
buttonGritar.Position = UDim2.new(0.45,0,0.5,0)
buttonGritar.BackgroundColor3 = Color3.fromRGB(255,165,0)
buttonGritar.TextColor3 = Color3.fromRGB(255,255,255)
buttonGritar.Text = "Gritar"
buttonGritar.Font = Enum.Font.SourceSansBold
buttonGritar.TextScaled = true
buttonGritar.Visible = false
buttonGritar.Parent = screenGui

local buttonSoco = Instance.new("TextButton")
buttonSoco.Size = UDim2.new(0,120,0,40)
buttonSoco.Position = UDim2.new(0.7,0,0.5,0)
buttonSoco.BackgroundColor3 = Color3.fromRGB(0,255,0)
buttonSoco.TextColor3 = Color3.fromRGB(255,255,255)
buttonSoco.Text = "Soco"
buttonSoco.Font = Enum.Font.SourceSansBold
buttonSoco.TextScaled = true
buttonSoco.Visible = false
buttonSoco.Parent = screenGui

-- ===================== FUNÇÕES =====================
local function PlayAudio(id)
    local sound = Instance.new("Sound", hrp)
    sound.SoundId = id
    sound.Volume = 1
    sound:Play()
    Debris:AddItem(sound, 10)
end

local function SpawnSmoke(pos)
    local part = Instance.new("Part")
    part.Anchored = true
    part.CanCollide = false
    part.Size = Vector3.new(1,1,1)
    part.Transparency = 1
    part.Position = pos
    part.Parent = workspace

    local smoke = Instance.new("ParticleEmitter")
    smoke.Texture = "rbxassetid://258128463"
    smoke.Rate = 200
    smoke.Lifetime = NumberRange.new(0.5,1)
    smoke.Speed = NumberRange.new(2,4)
    smoke.SpreadAngle = Vector2.new(360,360)
    smoke.Size = NumberSequence.new({NumberSequenceKeypoint.new(0,1), NumberSequenceKeypoint.new(1,3)})
    smoke.Parent = part

    Debris:AddItem(part,2)
end

local function MakeLine(startPos,endPos)
    if workspace:FindFirstChild("LinePart") then workspace.LinePart:Destroy() end
    local line = Instance.new("Part")
    line.Name = "LinePart"
    line.Anchored = true
    line.CanCollide = false
    line.Size = Vector3.new(0.2,0.2,(endPos - startPos).Magnitude)
    line.Material = Enum.Material.Neon
    line.Color = LineColor
    line.CFrame = CFrame.new(startPos,endPos)*CFrame.new(0,0,-(endPos-startPos).Magnitude/2)
    line.Parent = workspace
end

local bodyVelocity

local function StartFly(pos)
    flying = true
    MakeLine(hrp.Position,pos)
    PlayAudio(SoundId)
    SpawnSmoke(pos)
    humanoid.WalkSpeed = BoostWalkSpeed

    if bodyVelocity then bodyVelocity:Destroy() end
    bodyVelocity = Instance.new("BodyVelocity")
    bodyVelocity.MaxForce = Vector3.new(1e5,1e5,1e5)
    bodyVelocity.Velocity = (pos - hrp.Position).Unit * Speed
    bodyVelocity.Parent = hrp

    RunService:BindToRenderStep("ContinuousFly", Enum.RenderPriority.Character.Value, function()
        if flying and bodyVelocity then
            bodyVelocity.Velocity = (pos - hrp.Position).Unit * Speed
        end
    end)
end

local function StopFly()
    flying = false
    humanoid.WalkSpeed = NormalWalkSpeed
    if bodyVelocity then bodyVelocity:Destroy() end
    if workspace:FindFirstChild("LinePart") then workspace.LinePart:Destroy() end
    RunService:UnbindFromRenderStep("ContinuousFly")
end

local function ProcessTap(pos)
    if flying then
        StopFly()
    else
        StartFly(pos)
    end
end

local function Explode()
    local explosion = Instance.new("Explosion")
    explosion.Position = hrp.Position
    explosion.BlastRadius = 50
    explosion.BlastPressure = 500000
    explosion.DestroyJointRadiusPercent = 0
    explosion.Parent = workspace
end

-- ===================== TRANSFORMAÇÃO EM TITAN COM BODY SCALE =====================
local function TransformToTitan()
    humanoid.WalkSpeed = 50
    -- BodyScale (mais seguro que multiplicar partes)
    if not humanoid:FindFirstChild("BodyHeightScale") then
        local hScale = Instance.new("NumberValue")
        hScale.Name = "BodyHeightScale"
        hScale.Value = titanScale
        hScale.Parent = humanoid
    else
        humanoid.BodyHeightScale.Value = titanScale
    end

    if not humanoid:FindFirstChild("BodyWidthScale") then
        local wScale = Instance.new("NumberValue")
        wScale.Name = "BodyWidthScale"
        wScale.Value = titanScale
        wScale.Parent = humanoid
    else
        humanoid.BodyWidthScale.Value = titanScale
    end

    if not humanoid:FindFirstChild("BodyDepthScale") then
        local dScale = Instance.new("NumberValue")
        dScale.Name = "BodyDepthScale"
        dScale.Value = titanScale
        dScale.Parent = humanoid
    else
        humanoid.BodyDepthScale.Value = titanScale
    end
end

local function RevertFromTitan()
    humanoid.WalkSpeed = NormalWalkSpeed
    if humanoid:FindFirstChild("BodyHeightScale") then humanoid.BodyHeightScale.Value = 1 end
    if humanoid:FindFirstChild("BodyWidthScale") then humanoid.BodyWidthScale.Value = 1 end
    if humanoid:FindFirstChild("BodyDepthScale") then humanoid.BodyDepthScale.Value = 1 end
end

-- ===================== EVENTOS BOTÕES =====================
-- Active Gear
buttonActiveGear.MouseButton1Click:Connect(function()
    abilityActive = not abilityActive
    if abilityActive then
        cinturaItem = Instance.new("Model")
        cinturaItem.Name = "CinturaItem"
        cinturaItem.Parent = char
    else
        if cinturaItem then cinturaItem:Destroy() cinturaItem = nil end
        StopFly()
    end
end)

-- Transformar
buttonTransform.MouseButton1Click:Connect(function()
    if tick() - lastTransform < transformCooldown then return end
    lastTransform = tick()
    buttonActiveGear.Visible = false
    buttonTransform.Visible = false

    PlayAudio(audioMusica)

    task.delay(transformDelay,function()
        Explode()
        PlayAudio(audioTransformacao)
        PlayAudio(audioEletrico)
        TransformToTitan()
        buttonVoltar.Visible = true
        buttonGritar.Visible = true
        buttonSoco.Visible = true
    end)
end)

-- Voltar
buttonVoltar.MouseButton1Click:Connect(function()
    if tick() - lastVoltar < actionCooldown then return end
    lastVoltar = tick()
    buttonVoltar.Visible = false
    buttonGritar.Visible = false
    buttonSoco.Visible = false
    buttonActiveGear.Visible = true
    buttonTransform.Visible = true
    RevertFromTitan()
end)

-- Gritar
buttonGritar.MouseButton1Click:Connect(function()
    if tick() - lastGrito < actionCooldown then return end
    lastGrito = tick()
    PlayAudio(audioGrito)
end)

-- Soco
buttonSoco.MouseButton1Click:Connect(function()
    if tick() - lastSoco < actionCooldown then return end
    lastSoco = tick()
    PlayAudio(audioPisao)
end)

-- Detectar clique/tap para voo
UserInputService.InputEnded:Connect(function(input,gameProcessed)
    if not abilityActive then return end
    if input.UserInputType == Enum.UserInputType.Touch then
        local vp = Vector2.new(input.Position.X,input.Position.Y)
        local ray = workspace.CurrentCamera:ViewportPointToRay(vp.X,vp.Y)
        local result = workspace:Raycast(ray.Origin,ray.Direction*1000)
        if result then
            ProcessTap(result.Position)
        end
    end
end)

-- PC
local mouse = player:GetMouse()
mouse.Button1Down:Connect(function()
    if abilityActive then
        ProcessTap(mouse.Hit.Position)
    end
end)
