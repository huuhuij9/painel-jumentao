--[[
    PAINEL JUMENTÃO V4.5 - ESP RAINBOW + TÍTULO
    - Adicionado título visível no painel principal.
    - ESP dos inimigos com efeito rainbow automático.
    - Aliados mantêm cor azul fixa.
]]

local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- CONFIGURAÇÕES GLOBAIS
getgenv().HBE_Enabled = true
getgenv().ESP_Enabled = false
getgenv().WallCheck = true
getgenv().TeamCheck = true 
getgenv().HitboxSize = 10
getgenv().ProximityRange = 55
getgenv().MaxAllies = 4

local Allies = {}
local PinkColor = Color3.fromRGB(0, 255, 0)

-- ===========================
-- FUNÇÃO RAINBOW
-- ===========================
local RAINBOW_SPEED = 0.9

local function GetRainbowColor()
    local hue = (tick() * RAINBOW_SPEED) % 1
    return Color3.fromHSV(hue, 1, 1)
end

-- NOTIFICAÇÃO
local function Notify(title, text, duration)
    local sg = Instance.new("ScreenGui", LocalPlayer.PlayerGui)
    sg.Name = "Jumentao_Notify"
    local f = Instance.new("Frame", sg)
    f.Size = UDim2.new(0, 220, 0, 70)
    f.Position = UDim2.new(1, 10, 0.8, 0)
    f.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
    local UICorner = Instance.new("UICorner", f)
    UICorner.CornerRadius = UDim.new(0, 8)
    local UIStroke = Instance.new("UIStroke", f)
    UIStroke.Color = PinkColor
    UIStroke.Thickness = 2
    
    local TitleLabel = Instance.new("TextLabel", f)
    TitleLabel.BackgroundTransparency = 1
    TitleLabel.Position = UDim2.new(0, 10, 0, 5)
    TitleLabel.Size = UDim2.new(1, -20, 0, 20)
    TitleLabel.Font = Enum.Font.GothamBold
    TitleLabel.Text = title
    TitleLabel.TextColor3 = PinkColor
    TitleLabel.TextSize = 14
    TitleLabel.TextXAlignment = Enum.TextXAlignment.Left
    
    local TextLabel = Instance.new("TextLabel", f)
    TextLabel.BackgroundTransparency = 1
    TextLabel.Position = UDim2.new(0, 10, 0, 25)
    TextLabel.Size = UDim2.new(1, -20, 0, 40)
    TextLabel.Font = Enum.Font.Gotham
    TextLabel.Text = text
    TextLabel.TextColor3 = Color3.new(1, 1, 1)
    TextLabel.TextSize = 12
    TextLabel.TextWrapped = true
    TextLabel.TextXAlignment = Enum.TextXAlignment.Left
    
    TweenService:Create(f, TweenInfo.new(0.5, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {Position = UDim2.new(1, -230, 0.8, 0)}):Play()
    task.delay(duration or 3, function()
        local exit = TweenService:Create(f, TweenInfo.new(0.5, Enum.EasingStyle.Back, Enum.EasingDirection.In), {Position = UDim2.new(1, 10, 0.8, 0)})
        exit:Play()
        exit.Completed:Connect(function() sg:Destroy() end)
    end)
end

-- ===========================
-- LÓGICA DE TEAMCHECK
-- ===========================
local function AutoDetectAllies()
    local char = LocalPlayer.Character
    if not char or not char:FindFirstChild("HumanoidRootPart") then return end
    local myPos = char.HumanoidRootPart.Position
    local potentialAllies = {}
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            local dist = (p.Character.HumanoidRootPart.Position - myPos).Magnitude
            if dist <= getgenv().ProximityRange then
                table.insert(potentialAllies, {player = p, distance = dist})
            end
        end
    end
    table.sort(potentialAllies, function(a, b) return a.distance < b.distance end)
    local newAllies = {}
    for i = 1, math.min(#potentialAllies, getgenv().MaxAllies) do
        local p = potentialAllies[i].player
        newAllies[p.UserId] = true
    end
    Allies = newAllies
    Notify("TeamCheck", "Aliados detectados!", 2)
end

LocalPlayer.CharacterAdded:Connect(function()
    task.wait(1.5)
    AutoDetectAllies()
end)

local function IsEnemy(otherPlayer)
    if not getgenv().TeamCheck then return true end
    return not Allies[otherPlayer.UserId]
end

-- ===========================
-- WALLCHECK
-- ===========================
local function IsVisible(targetPart)
    if not getgenv().WallCheck then return true end
    local origin = Camera.CFrame.Position
    local direction = (targetPart.Position - origin)
    local raycastParams = RaycastParams.new()
    local ignoreList = {}
    for _, p in pairs(Players:GetPlayers()) do
        if p.Character then table.insert(ignoreList, p.Character) end
    end
    raycastParams.FilterDescendantsInstances = ignoreList
    raycastParams.FilterType = Enum.RaycastFilterType.Exclude
    local result = workspace:Raycast(origin, direction, raycastParams)
    return result == nil
end

-- ===========================
-- LOOP PRINCIPAL
-- ===========================
RunService.Heartbeat:Connect(function()
    local rainbowColor = GetRainbowColor()
    for _, p in pairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            local root = p.Character.HumanoidRootPart
            local hum = p.Character:FindFirstChild("Humanoid")
            if hum and hum.Health > 0 then
                local isEnemy = IsEnemy(p)
                local visible = IsVisible(root)
                
                if getgenv().HBE_Enabled and isEnemy and visible then
                    local size = getgenv().HitboxSize
                    root.Size = Vector3.new(size, size, size)
                    root.Transparency = 1
                    root.CanCollide = false
                else
                    root.Size = Vector3.new(2, 2, 2)
                    root.Transparency = 1
                    root.CanCollide = true
                end

                local hl = p.Character:FindFirstChild("JUMENTAO_HL")
                if getgenv().ESP_Enabled then
                    if not hl then 
                        hl = Instance.new("Highlight", p.Character)
                        hl.Name = "JUMENTAO_HL" 
                    end
                    if not isEnemy then
                        hl.FillColor = Color3.new(0, 0.5, 1)
                        hl.FillTransparency = 0.5
                        hl.OutlineColor = Color3.new(0, 0.5, 1)
                        hl.OutlineTransparency = 0
                    else
                        hl.FillColor = rainbowColor
                        hl.FillTransparency = 0.5
                        hl.OutlineColor = rainbowColor
                        hl.OutlineTransparency = 0
                    end
                    hl.Enabled = true
                elseif hl then 
                    hl:Destroy() 
                end
            end
        end
    end
end)

-- ===========================
-- INTERFACE COM TÍTULO
-- ===========================
local ScreenGui = Instance.new("ScreenGui", LocalPlayer.PlayerGui)
ScreenGui.Name = "Jumentao_Final_UI"
ScreenGui.ResetOnSpawn = false

local Main = Instance.new("Frame", ScreenGui)
Main.Size = UDim2.new(0, 300, 0, 280) -- Aumentado um pouco para o título
Main.Position = UDim2.new(0.5, -150, 0.5, -140)
Main.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
Main.Visible = false
Instance.new("UICorner", Main)
Instance.new("UIStroke", Main).Color = PinkColor

-- ADICIONANDO O TÍTULO DO PAINEL
local Title = Instance.new("TextLabel", Main)
Title.Size = UDim2.new(1, 0, 0, 40)
Title.Position = UDim2.new(0, 0, 0, 5)
Title.BackgroundTransparency = 1
Title.Text = "HRZ VIP"
Title.TextColor3 = PinkColor
Title.TextSize = 18
Title.Font = Enum.Font.GothamBold

local function AddBtn(text, y, var, callback)
    local b = Instance.new("TextButton", Main)
    b.Size = UDim2.new(0.8, 0, 0, 35)
    b.Position = UDim2.new(0.1, 0, 0, y)
    b.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    b.Text = text .. (getgenv()[var] and ": ON" or ": OFF")
    b.BackgroundColor3 = getgenv()[var] and PinkColor or Color3.fromRGB(30, 30, 30)
    b.TextColor3 = Color3.new(0, 0, 0)
    Instance.new("UICorner", b)
    b.MouseButton1Click:Connect(function()
        getgenv()[var] = not getgenv()[var]
        b.Text = text .. (getgenv()[var] and ": ON" or ": OFF")
        b.BackgroundColor3 = getgenv()[var] and PinkColor or Color3.fromRGB(30, 30, 30)
        if callback then callback() end
    end)
end

-- Ajustado o Y dos botões para não sobrepor o título
AddBtn("HITBOX 🎯", 60, "HBE_Enabled")
AddBtn("ESP 🌈", 110, "ESP_Enabled")
AddBtn("WALLCHECK 🧱", 160, "WallCheck")
AddBtn("INGNORAR ALIADOS 👥", 210, "TeamCheck", function() 
    AutoDetectAllies() 
end)

local TBtn = Instance.new("TextButton", ScreenGui)
TBtn.Size = UDim2.new(0, 50, 0, 50)
TBtn.Position = UDim2.new(0.9, 0, 0.1, 0)
TBtn.BackgroundColor3 = PinkColor
TBtn.Text = "X"
Instance.new("UICorner", TBtn).CornerRadius = UDim.new(1, 0)
TBtn.MouseButton1Click:Connect(function() Main.Visible = not Main.Visible end)

task.spawn(AutoDetectAllies)
Notify("HRZ", "PAINEL CARREGADO COM SUCESSO!", 4)
# painel-jumentao
