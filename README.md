-- LocalScript: ESP avançado para treinamento (NÃO é aimbot)
-- Mostra: healthbar, nome, linha do centro até o alvo mais próximo e ponto de previsão (lead)
-- NÃO move a câmera nem dispara automaticamente. Uso: testes/servidores privados.
-- Toggle: RightShift
-- Ajuste 'isAdminCheck' conforme sua lógica de permissões.

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local UserInputService = game:GetService("UserInputService")
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

-- CONFIG ------------------------------------------------
local ENABLE_BY_DEFAULT = false
local SHOW_HEALTHBAR = true
local TARGET_SCREEN_RADIUS = 120 -- pixels para considerar "próximo"
local PREDICTION_MULTIPLIER = 0.3 -- quanto da velocidade do alvo usamos para prever
local TOGGLE_KEY = Enum.KeyCode.RightShift
-- Admin check (ajuste conforme desejar). Aqui: apenas usuario com nome "Gustaaa" é admin por padrao
local function isAdminCheck()
    -- exemplo simples: troque por verificação com ServerStorage, RemoteFunction, DataStore, etc.
    return LocalPlayer.Name == "Gustaaa"
end
if not isAdminCheck() then
    ENABLE_BY_DEFAULT = false
end
-- END CONFIG --------------------------------------------

-- GUI
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "ESP_Adv_HUD"
screenGui.ResetOnSpawn = false
screenGui.Parent = PlayerGui

local infoLabel = Instance.new("TextLabel")
infoLabel.Size = UDim2.new(0,250,0,30)
infoLabel.Position = UDim2.new(0,8,0,8)
infoLabel.BackgroundTransparency = 0.6
infoLabel.BackgroundColor3 = Color3.fromRGB(0,0,0)
infoLabel.TextColor3 = Color3.fromRGB(255,255,255)
infoLabel.TextScaled = true
infoLabel.Font = Enum.Font.SourceSans
infoLabel.Text = "ESP Avançado: Desligado"
infoLabel.Parent = screenGui

-- Canvas para desenhar linhas/elementos 2D
local function newLine(name)
    local frame = Instance.new("Frame")
    frame.Name = name
    frame.AnchorPoint = Vector2.new(0,0)
    frame.Size = UDim2.new(0,0,0,0)
    frame.BackgroundTransparency = 1
    frame.Parent = screenGui
    return frame
end

-- Retângulo para o ponto de previsão (um pequeno quad)
local leadDot = Instance.new("Frame")
leadDot.Name = "LeadDot"
leadDot.Size = UDim2.new(0,8,0,8)
leadDot.AnchorPoint = Vector2.new(0.5,0.5)
leadDot.Position = UDim2.new(0.5,0.5)
leadDot.BackgroundTransparency = 0.1
leadDot.BorderSizePixel = 0
leadDot.Visible = false
leadDot.Parent = screenGui

-- Linha desenhada com UIStroke em uma Frame rotacionada:
local lineFrame = Instance.new("Frame")
lineFrame.Name = "TargetLine"
lineFrame.AnchorPoint = Vector2.new(0,0.5)
lineFrame.Size = UDim2.new(0,0,0,2)
lineFrame.Position = UDim2.new(0.5,0,0.5,0)
lineFrame.BackgroundColor3 = Color3.fromRGB(255,255,255)
lineFrame.Visible = false
lineFrame.Parent = screenGui

local lineStroke = Instance.new("UIGradient")
lineStroke.Rotation = 0
lineStroke.Parent = lineFrame

-- Indicator de texto do alvo
local targetText = Instance.new("TextLabel")
targetText.Size = UDim2.new(0,300,0,28)
targetText.Position = UDim2.new(0.5, -150, 0.9, 0)
targetText.AnchorPoint = Vector2.new(0.5,0)
targetText.BackgroundTransparency = 0.6
targetText.TextScaled = true
targetText.TextColor3 = Color3.fromRGB(255,255,255)
targetText.Text = ""
targetText.Visible = false
targetText.Parent = screenGui

-- Toggle state
local enabled = ENABLE_BY_DEFAULT
infoLabel.Text = "ESP Avançado: " .. (enabled and "Ligado (RightShift para alternar)" or "Desligado (RightShift para alternar)")

-- Função utilitária: converte World -> Viewport (x,y) e visibilidade
local function worldToScreenPoint(pos)
    local onScreen, x, y = pcall(function()
        local v3, onScreenFlag = Camera:WorldToViewportPoint(pos)
        return onScreenFlag, v3.X, v3.Y
    end)
    if not onScreen then return nil end
    local v3, onScreenFlag = Camera:WorldToViewportPoint(pos)
    return onScreenFlag, Vector2.new(v3.X, v3.Y), v3.Z -- z = depth
end

-- cria / atualiza ESP de head + healthbar (simples)
local function createOrUpdateBillboard(character)
    if not character then return end
    local head = character:FindFirstChild("Head")
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if not head or not humanoid then return end

    local bb = head:FindFirstChild("ESP_Billboard")
    if not bb then
        bb = Instance.new("BillboardGui")
        bb.Name = "ESP_Billboard"
        bb.Size = UDim2.new(0,120,0,38)
        bb.AlwaysOnTop = true
        bb.StudsOffset = Vector3.new(0,1.6,0)
        bb.Parent = head

        local frame = Instance.new("Frame")
        frame.BackgroundTransparency = 1
        frame.Size = UDim2.new(1,0,1,0)
        frame.Parent = bb

        local nameLabel = Instance.new("TextLabel")
        nameLabel.Name = "Name"
        nameLabel.TextScaled = true
        nameLabel.Size = UDim2.new(1,-6,0,18)
        nameLabel.Position = UDim2.new(0,3,0,0)
        nameLabel.BackgroundTransparency = 1
        nameLabel.Font = Enum.Font.SourceSansBold
        nameLabel.TextStrokeTransparency = 0.6
        nameLabel.Parent = frame

        local healthBG = Instance.new("Frame")
        healthBG.Name = "HealthBG"
        healthBG.Position = UDim2.new(0.05,0,0,20)
        healthBG.Size = UDim2.new(0.9,0,0,8)
        healthBG.BackgroundTransparency = 0.5
        healthBG.Parent = frame

        local healthFill = Instance.new("Frame")
        healthFill.Name = "HealthFill"
        healthFill.Size = UDim2.new(1,0,1,0)
        healthFill.BorderSizePixel = 0
        healthFill.Parent = healthBG
    end

    local nameLabel = bb:FindFirstChildWhichIsA("Frame"):FindFirstChild("Name")
    local healthFill = bb:FindFirstChildWhichIsA("Frame"):FindFirstChild("HealthBG"):FindFirstChild("HealthFill")
    nameLabel.Text = character.Name
    if SHOW_HEALTHBAR and humanoid then
        local pct = math.clamp(humanoid.Health / math.max(1, humanoid.MaxHealth), 0, 1)
        healthFill.Size = UDim2.new(pct,0,1,0)
        if pct > 0.6 then
            healthFill.BackgroundColor3 = Color3.fromRGB(0,200,0)
        elseif pct > 0.3 then
            healthFill.BackgroundColor3 = Color3.fromRGB(240,200,0)
        else
            healthFill.BackgroundColor3 = Color3.fromRGB(220,60,60)
        end
    else
        local hb = bb:FindFirstChildWhichIsA("Frame"):FindFirstChild("HealthBG")
        if hb then hb:Destroy() end
    end
end

-- remove billboard
local function removeBillboard(character)
    if not character then return end
    local head = character:FindFirstChild("Head")
    if head then
        local bb = head:FindFirstChild("ESP_Billboard")
        if bb then bb:Destroy() end
    end
end

-- Conecta jogadores atuais e futuros
Players.PlayerAdded:Connect(function(p)
    if p == LocalPlayer then return end
    p.CharacterAdded:Connect(function(char)
        -- esperar humanoid/head
        char:WaitForChild("Humanoid", 5)
        char:WaitForChild("Head", 5)
        createOrUpdateBillboard(char)
    end)
end)
for _,p in pairs(Players:GetPlayers()) do
    if p ~= LocalPlayer and p.Character then
        createOrUpdateBillboard(p.Character)
    end
end

-- Determina o jogador mais próximo do centro da tela (visível)
local function getClosestVisiblePlayer()
    local bestDist = math.huge
    local bestPlayer = nil
    local screenCenter = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
    for _,p in pairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character and p.Character:FindFirstChild("Head") and p.Character:FindFirstChildOfClass("Humanoid") then
            local head = p.Character.Head
            local humanoid = p.Character:FindFirstChildOfClass("Humanoid")
            if humanoid and humanoid.Health > 0 then
                local onScreen, screenPos, depth = worldToScreenPoint(head.Position)
                if onScreen then
                    local dist = (screenPos - screenCenter).Magnitude
                    if dist < bestDist then
                        bestDist = dist
                        bestPlayer = p
                    end
                end
            end
        end
    end
    return bestPlayer, bestDist
end

-- Calcula posição prevista (lead) no mundo: pos + velocity * factor
local function computeLeadPosition(part)
    if not part then return part and part.Position or nil end
    local vel = Vector3.new(0,0,0)
    if part:IsA("BasePart") then vel = part.Velocity or Vector3.new(0,0,0) end
    -- multiplicador simples: quanto maior a distancia, maior o lead
    local distance = (part.Position - Camera.CFrame.Position).Magnitude
    local t = PREDICTION_MULTIPLIER * (distance / 50) -- escala simples
    return part.Position + vel * t
end

-- Atualiza overlay a cada frame
RunService.RenderStepped:Connect(function()
    -- atualiza billboards (vida + nome)
    for _,p in pairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character then
            createOrUpdateBillboard(p.Character)
        end
    end

    if not enabled then
        lineFrame.Visible = false
        leadDot.Visible = false
        targetText.Visible = false
        infoLabel.Text = "ESP Avançado: Desligado (RightShift)"
        return
    end

    infoLabel.Text = "ESP Avançado: Ligado (RightShift)"

    -- encontra alvo mais próximo do centro
    local target, dist = getClosestVisiblePlayer()
    if not target or dist > TARGET_SCREEN_RADIUS then
        lineFrame.Visible = false
        leadDot.Visible = false
        targetText.Visible = false
        return
    end

    -- calcula posição de cabeça e lead
    local head = target.Character and target.Character:FindFirstChild("Head")
    if not head then
        lineFrame.Visible = false
        leadDot.Visible = false
        targetText.Visible = false
        return
    end

    local onScreenHead, headScreenPos, depth = worldToScreenPoint(head.Position)
    if not onScreenHead then
        lineFrame.Visible = false
        leadDot.Visible = false
        targetText.Visible = false
        return
    end

    -- Predição: calcula posição prevista no mundo, converte pra tela
    local leadWorld = computeLeadPosition(head)
    local onScreenLead, leadScreenPos = false, nil
    do
        local ok, v = pcall(function()
            local v3 = Camera:WorldToViewportPoint(leadWorld)
            return v3.Z > 0, Vector2.new(v3.X, v3.Y)
        end)
        if ok then
            local v3 = Camera:WorldToViewportPoint(leadWorld)
            onScreenLead = v3.Z > 0
            leadScreenPos = Vector2.new(v3.X, v3.Y)
        end
    end

    -- Linha do centro até a cabeça (rotacionada)
    local screenCenter = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
    local dir = headScreenPos - screenCenter
    local length = dir.Magnitude
    if length < 2 then length = 2 end
    local angle = math.deg(math.atan2(dir.Y, dir.X))

    -- update lineFrame: posicionar na metade entre centro e head
    lineFrame.Size = UDim2.new(0, length, 0, 2)
    lineFrame.Position = UDim2.new(0, screenCenter.X, 0, screenCenter.Y)
    lineFrame.AnchorPoint = Vector2.new(0.5, 0.5)
    lineFrame.Rotation = angle
    lineFrame.Visible = true

    -- colocar leadDot se visivel
    if onScreenLead and leadScreenPos then
        leadDot.Position = UDim2.new(0, leadScreenPos.X, 0, leadScreenPos.Y)
        leadDot.Visible = true
        leadDot.Size = UDim2.new(0, 8, 0, 8)
        -- cor indicativa: mais longe => amarelo/ vermelho
        local distance = (head.Position - Camera.CFrame.Position).Magnitude
        if distance > 80 then
            leadDot.BackgroundColor3 = Color3.fromRGB(240,200,0)
        else
            leadDot.BackgroundColor3 = Color3.fromRGB(0,200,0)
        end
    else
        leadDot.Visible = false
    end

    -- target text
    targetText.Text = string.format("Alvo: %s  |  Dist: %.0f | Vel: %.1f", target.Name,
        (head.Position - Camera.CFrame.Position).Magnitude,
        head.Velocity.Magnitude)
    targetText.Visible = true
end)

-- Toggle com tecla
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == TOGGLE_KEY then
        if not isAdminCheck() then
            -- opcional: aviso se não autorizado
            infoLabel.Text = "ESP Avançado: sem permissão"
            wait(1.2)
            infoLabel.Text = "ESP Avançado: Desligado (RightShift)"
            return
        end
        enabled = not enabled
        infoLabel.Text = "ESP Avançado: " .. (enabled and "Ligado (RightShift)" or "Desligado (RightShift)")
    end
end)

-- Mensagem de carregamento
warn("ESP Avançado (visual only) carregado. Lembre-se de usar apenas em servidores privados/testes.")
