
-- Configurações Iniciais
getgenv().AimbotEnabled = false
getgenv().TeamCheck = false
getgenv().AimbotPart = "Head"
getgenv().AimbotSmooth = 5
getgenv().FOVEnabled = true
getgenv().FOVRadius = 120
getgenv().FOVColor = Color3.fromRGB(255, 255, 255)
getgenv().ESPEnabled = false
getgenv().ESPSettings = {
    Names = false,
    Boxes = false,
    Tracers = false,
    TeamColor = false,
    BoxColor = Color3.fromRGB(255, 255, 255),
    TracerColor = Color3.fromRGB(255, 255, 255),
    NameColor = Color3.fromRGB(255, 255, 255),
    ShowDistance = false
}
getgenv().IgnoreDead = true

-- Serviços
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Camera = workspace.CurrentCamera
local LocalPlayer = Players.LocalPlayer

-- Funções Utilitárias
local function IsValidTarget(player)
    if not player or player == LocalPlayer then return false end
    if not player.Character or not player.Character:FindFirstChild("Humanoid") then return false end
    if getgenv().TeamCheck and player.Team == LocalPlayer.Team then return false end
    if getgenv().IgnoreDead and player.Character.Humanoid.Health <= 0 then return false end
    return true
end

local function GetClosestPlayer()
    local closestPlayer = nil
    local shortestDistance = math.huge
    local mousePos = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)

    for _, player in pairs(Players:GetPlayers()) do
        if IsValidTarget(player) then
            local character = player.Character
            if character and character:FindFirstChild(getgenv().AimbotPart) then
                local targetPart = character[getgenv().AimbotPart]
                local pos, onScreen = Camera:WorldToViewportPoint(targetPart.Position)
                
                if onScreen then
                    local distance = (Vector2.new(pos.X, pos.Y) - mousePos).Magnitude
                    
                    if distance <= getgenv().FOVRadius and distance < shortestDistance then
                        closestPlayer = player
                        shortestDistance = distance
                    end
                end
            end
        end
    end

    return closestPlayer
end

-- ESP Functions
local ESP = {
    Drawings = {},
    Connections = {}
}

function ESP:CreateDrawing(type, properties)
    local drawing = Drawing.new(type)
    for property, value in pairs(properties) do
        drawing[property] = value
    end
    return drawing
end

function ESP:CreatePlayerESP(player)
    local drawings = {
        Box = ESP:CreateDrawing("Square", {
            Thickness = 1,
            Filled = false,
            Transparency = 1,
            Color = getgenv().ESPSettings.BoxColor,
            Visible = false
        }),
        Name = ESP:CreateDrawing("Text", {
            Text = player.Name,
            Size = 13,
            Center = true,
            Outline = true,
            Color = getgenv().ESPSettings.NameColor,
            Visible = false
        }),
        Tracer = ESP:CreateDrawing("Line", {
            Thickness = 1,
            Transparency = 1,
            Color = getgenv().ESPSettings.TracerColor,
            Visible = false
        }),
        DistanceText = ESP:CreateDrawing("Text", {
            Size = 13,
            Center = true,
            Outline = true,
            Color = Color3.fromRGB(255, 255, 255),
            Visible = false
        })
    }
    
    ESP.Drawings[player] = drawings
end

function ESP:RemovePlayerESP(player)
    if ESP.Drawings[player] then
        for _, drawing in pairs(ESP.Drawings[player]) do
            drawing:Remove()
        end
        ESP.Drawings[player] = nil
    end
end

function ESP:UpdateESP()
    for player, drawings in pairs(ESP.Drawings) do
        if player and player.Character and player.Character:FindFirstChild("HumanoidRootPart") and player ~= LocalPlayer then
            local character = player.Character
            local humanoidRootPart = character:FindFirstChild("HumanoidRootPart")
            local head = character:FindFirstChild("Head")
            
            if not humanoidRootPart then continue end
            
            local vector, onScreen = Camera:WorldToViewportPoint(humanoidRootPart.Position)
            local distance = (Camera.CFrame.Position - humanoidRootPart.Position).Magnitude
            
            if onScreen and getgenv().ESPEnabled then
                if getgenv().TeamCheck and player.Team == LocalPlayer.Team then continue end
                
                -- Box ESP
                if getgenv().ESPSettings.Boxes then
                    local rootPos = Camera:WorldToViewportPoint(humanoidRootPart.Position)
                    local headPos = Camera:WorldToViewportPoint(head.Position + Vector3.new(0, 0.5, 0))
                    local legPos = Camera:WorldToViewportPoint(humanoidRootPart.Position - Vector3.new(0, 3, 0))
                    
                    local boxSize = Vector2.new((headPos.Y - legPos.Y) / 2, headPos.Y - legPos.Y)
                    local boxPos = Vector2.new(rootPos.X - boxSize.X / 2, rootPos.Y - boxSize.Y / 2)
                    
                    drawings.Box.Size = boxSize
                    drawings.Box.Position = boxPos
                    drawings.Box.Visible = true
                    drawings.Box.Color = getgenv().ESPSettings.TeamColor and player.Team.TeamColor.Color or getgenv().ESPSettings.BoxColor
                else
                    drawings.Box.Visible = false
                end
                
                -- Name ESP
                if getgenv().ESPSettings.Names then
                    drawings.Name.Position = Vector2.new(vector.X, vector.Y - 40)
                    drawings.Name.Visible = true
                    drawings.Name.Color = getgenv().ESPSettings.TeamColor and player.Team.TeamColor.Color or getgenv().ESPSettings.NameColor
                else
                    drawings.Name.Visible = false
                end
                
                -- Tracer ESP
                if getgenv().ESPSettings.Tracers then
                    drawings.Tracer.From = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y)
                    drawings.Tracer.To = Vector2.new(vector.X, vector.Y)
                    drawings.Tracer.Visible = true
                    drawings.Tracer.Color = getgenv().ESPSettings.TeamColor and player.Team.TeamColor.Color or getgenv().ESPSettings.TracerColor
                else
                    drawings.Tracer.Visible = false
                end
                
                -- Distance ESP
                if getgenv().ESPSettings.ShowDistance then
                    drawings.DistanceText.Text = string.format("%.0f studs", distance)
                    drawings.DistanceText.Position = Vector2.new(vector.X, vector.Y + 35)
                    drawings.DistanceText.Visible = true
                else
                    drawings.DistanceText.Visible = false
                end
            else
                drawings.Box.Visible = false
                drawings.Name.Visible = false
                drawings.Tracer.Visible = false
                drawings.DistanceText.Visible = false
            end
        end
    end
end

-- FOV Circle
local FOVCircle = Drawing.new("Circle")
FOVCircle.Visible = getgenv().FOVEnabled
FOVCircle.Radius = getgenv().FOVRadius
FOVCircle.Color = getgenv().FOVColor
FOVCircle.Thickness = 1
FOVCircle.Filled = false
FOVCircle.Transparency = 1
FOVCircle.Position = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)

-- Interface
local Library = loadstring(game:HttpGet("https://raw.githubusercontent.com/xHeptc/Kavo-UI-Library/main/source.lua"))()
local Window = Library.CreateLib("Ultimate Aimbot", "Ocean")

-- Combat Tab
local CombatTab = Window:NewTab("Combat")
local CombatSection = CombatTab:NewSection("Aimbot Settings")

CombatSection:NewToggle("Enable Aimbot", "Liga/Desliga o aimbot", function(state)
    getgenv().AimbotEnabled = state
end)

CombatSection:NewDropdown("Aimbot Part", "Escolha a parte para mirar", {"Head", "HumanoidRootPart"}, function(currentOption)
    getgenv().AimbotPart = currentOption
end)

CombatSection:NewToggle("Team Check", "Verifica se é aliado", function(state)
    getgenv().TeamCheck = state
end)

CombatSection:NewToggle("FOV Circle", "Mostra/Esconde o círculo", function(state)
    getgenv().FOVEnabled = state
    FOVCircle.Visible = state
end)

CombatSection:NewSlider("FOV Size", "Ajusta o tamanho do FOV", 500, 50, function(value)
    getgenv().FOVRadius = value
    FOVCircle.Radius = value
end)

CombatSection:NewSlider("Aimbot Smoothness", "Ajusta a suavidade do aimbot", 20, 1, function(value)
    getgenv().AimbotSmooth = value
end)

-- ESP Tab
local ESPTab = Window:NewTab("ESP")
local ESPSection = ESPTab:NewSection("ESP Settings")

ESPSection:NewToggle("Enable ESP", "Liga/Desliga o ESP", function(state)
    getgenv().ESPEnabled = state
end)

ESPSection:NewToggle("Show Names", "Mostra nomes dos jogadores", function(state)
    getgenv().ESPSettings.Names = state
end)

ESPSection:NewToggle("Show Boxes", "Mostra caixas ao redor dos jogadores", function(state)
    getgenv().ESPSettings.Boxes = state
end)

ESPSection:NewToggle("Show Tracers", "Mostra linhas até os jogadores", function(state)
    getgenv().ESPSettings.Tracers = state
end)

ESPSection:NewToggle("Show Distance", "Mostra distância dos jogadores", function(state)
    getgenv().ESPSettings.ShowDistance = state
end)

ESPSection:NewToggle("Team Color", "Usa cor do time", function(state)
    getgenv().ESPSettings.TeamColor = state
end)

ESPSection:NewColorPicker("Box Color", "Escolha a cor das caixas", Color3.fromRGB(255, 255, 255), function(color)
    getgenv().ESPSettings.BoxColor = color
end)

-- Settings Tab
local SettingsTab = Window:NewTab("Settings")
local SettingsSection = SettingsTab:NewSection("UI Settings")

SettingsSection:NewKeybind("Toggle UI", "Tecla para mostrar/esconder o menu", Enum.KeyCode.RightShift, function()
    Library:ToggleUI()
end)

-- Player Connections
Players.PlayerAdded:Connect(function(player)
    ESP:CreatePlayerESP(player)
end)

Players.PlayerRemoving:Connect(function(player)
    ESP:RemovePlayerESP(player)
end)

-- Initialize ESP for existing players
for _, player in ipairs(Players:GetPlayers()) do
    if player ~= LocalPlayer then
        ESP:CreatePlayerESP(player)
    end
end

-- Main Loop
RunService.RenderStepped:Connect(function()
    FOVCircle.Position = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    FOVCircle.Visible = getgenv().FOVEnabled
    ESP:UpdateESP()
    
    if getgenv().AimbotEnabled then
        local Target = GetClosestPlayer()
        if Target and Target.Character and Target.Character:FindFirstChild(getgenv().AimbotPart) then
            local TargetPart = Target.Character[getgenv().AimbotPart]
            local pos = Camera:WorldToViewportPoint(TargetPart.Position)
            
            if pos.Z > 0 then
                local targetPos = Vector2.new(pos.X, pos.Y)
                local mousePos = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
                local moveVector = (targetPos - mousePos) / getgenv().AimbotSmooth
                
                Camera.CFrame = Camera.CFrame:Lerp(CFrame.new(Camera.CFrame.Position, TargetPart.Position), 1 - (getgenv().AimbotSmooth / 20))
            end
        end
    end
end)
