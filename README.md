-- Serviços essenciais
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

-- Configurações visuais
local CONFIG = {
    Assassino = {
        Cor = Color3.new(1, 0, 0), -- Vermelho intenso
        Espessura = 4,
        Transparencia = 0.4,
        Texto = "[ASSASSINO] ",
        Offset = Vector3.new(0, 4.5, 0)
    },
    Xerife = {
        Cor = Color3.new(0, 0, 1), -- Azul
        Espessura = 3,
        Transparencia = 0.5,
        Texto = "[XERIFE] ",
        Offset = Vector3.new(0, 4, 0)
    },
    SemPapel = {
        RemoverESP = true -- Não mostra nada em inocentes
    },
    SemLimiteDistancia = true
}

-- Armazena ESPs criados para evitar duplicações
local ESPsAtivos = {}

-- Função para remover ESP antigo se existir
local function removerESP(jogador)
    if ESPsAtivos[jogador.UserId] then
        for _, elemento in ipairs(ESPsAtivos[jogador.UserId]) do
            pcall(function() elemento:Destroy() end)
        end
        ESPsAtivos[jogador.UserId] = nil
    end
end

-- Função para criar ESP só para o papel correto
local function criarESP(jogador, papelAtual)
    -- Só continua se o papel for válido
    if not CONFIG[papelAtual] then
        removerESP(jogador)
        return
    end

    -- Verifica se o personagem está carregado
    local personagem = jogador.Character
    if not personagem or not personagem:FindFirstChild("HumanoidRootPart") or not personagem:FindFirstChildOfClass("Humanoid") then
        return
    end

    local root = personagem.HumanoidRootPart
    local configPapel = CONFIG[papelAtual]

    -- Remove ESP antigo antes de criar novo
    removerESP(jogador)

    -- Cria BillboardGui (texto)
    local billboard = Instance.new("BillboardGui")
    billboard.Name = "MM2_PreciseESP"
    billboard.Adornee = root
    billboard.Size = UDim2.new(0, 350, 0, 70)
    billboard.StudsOffset = configPapel.Offset
    billboard.AlwaysOnTop = true
    billboard.Parent = root

    -- Cria texto
    local texto = Instance.new("TextLabel")
    texto.Size = UDim2.new(1, 0, 1, 0)
    texto.BackgroundTransparency = 1
    texto.Text = configPapel.Texto .. jogador.Name
    texto.TextColor3 = configPapel.Cor
    texto.Font = Enum.Font.SourceSansBold
    texto.TextSize = 28
    texto.TextScaled = true
    texto.Parent = billboard

    -- Cria caixa 3D
    local caixa = Instance.new("BoxHandleAdornment")
    caixa.Name = "MM2_PreciseBox"
    caixa.Adornee = root
    caixa.Size = personagem:GetExtentsSize() + Vector3.new(0.6, 0.6, 0.6)
    caixa.Color3 = configPapel.Cor
    caixa.Thickness = configPapel.Espessura
    caixa.Transparency = configPapel.Transparencia
    caixa.AlwaysOnTop = true
    caixa.ZIndex = 15
    caixa.Parent = root

    -- Cria linha de rastreamento (para ver de MUITO longe)
    local linha = Instance.new("LineHandleAdornment")
    linha.Name = "MM2_PreciseLine"
    linha.Adornee = root
    linha.From = Vector3.new(0, 0, 0)
    linha.To = Players.LocalPlayer.Character and Players.LocalPlayer.Character.HumanoidRootPart.Position or Vector3.new(0,0,0)
    linha.Color3 = configPapel.Cor
    linha.Thickness = 2
    linha.AlwaysOnTop = true
    linha.Parent = root

    -- Armazena elementos para controle posterior
    ESPsAtivos[jogador.UserId] = {billboard, caixa, linha}

    -- Atualiza constante para manter precisão
    local conexao = RunService.RenderStepped:Connect(function()
        if not jogador or not jogador.Character or jogador.Character.Humanoid.Health <= 0 then
            conexao:Disconnect()
            removerESP(jogador)
            return
        end

        -- Atualiza posição da linha e tamanho da caixa
        local novoRoot = jogador.Character.HumanoidRootPart
        if novoRoot and Players.LocalPlayer.Character then
            linha.To = Players.LocalPlayer.Character.HumanoidRootPart.Position
            caixa.Size = jogador.Character:GetExtentsSize() + Vector3.new(0.6, 0.6, 0.6)
        end

        -- Garante que a cor não mude
        texto.TextColor3 = configPapel.Cor
        caixa.Color3 = configPapel.Cor
        linha.Color3 = configPapel.Cor
    end)
end

-- Função para detectar o papel CORRETO do jogador
local function getPapelCorreto(jogador)
    -- Métodos de detecção precisos (adaptados ao MM2 oficial)
    local papel = nil

    -- Método 1: Atributo oficial do jogo
    if jogador:GetAttribute("Role") then
        papel = jogador:GetAttribute("Role")
    -- Método 2: Valor armazenado no personagem
    elseif jogador.Character and jogador.Character:FindFirstChild("Role") then
        papel = jogador.Character.Role.Value
    -- Método 3: Eventos replicados do jogo
    elseif ReplicatedStorage:FindFirstChild("Events") and ReplicatedStorage.Events:FindFirstChild("SetRole") then
        local args = ReplicatedStorage.Events.SetRole:GetAttribute("Args")
        if args and args[jogador.UserId] then
            papel = args[jogador.UserId]
        end
    end

    -- Converte para nome padrão
    if papel then
        papel = string.upper(papel)
        if papel == "ASSASSIN" then return "Assassino"
        elseif papel == "SHERIFF" then return "Xerife"
        else return "SemPapel" end
    end

    return "SemPapel"
end

-- Sistema principal de detecção
local function monitorarPapeis()
    -- Verifica jogadores existentes
    for _, jogador in ipairs(Players:GetPlayers()) do
        if jogador ~= Players.LocalPlayer then
            criarESP(jogador, getPapelCorreto(jogador))
        end
    end

    -- Verifica novos jogadores
    Players.PlayerAdded:Connect(function(jogador)
        jogador.CharacterAdded:Connect(function()
            wait(1.5) -- Espera o jogo definir o papel
            criarESP(jogador, getPapelCorreto(jogador))
        end)
    end)

    -- Atualiza papeis constantemente (caso mude durante a partida)
    RunService.Heartbeat:Connect(function()
        for _, jogador in ipairs(Players:GetPlayers()) do
            if jogador ~= Players.LocalPlayer then
                local papelAtual = getPapelCorreto(jogador)
                criarESP(jogador, papelAtual)
            end
        end
    end)
end

-- Inicia o sistema após o jogo carregar completamente
wait(3) -- Tempo suficiente para o jogo definir os papeis iniciais
monitorarPapeis()
