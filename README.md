# scripts
my first scripts
-- Services
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- Settings
local LerpSpeed = 0.2 -- speed of smooth animations
local PulseSpeed = 2   -- speed of pulsing effect
local PulseAmount = 50 -- maximum glow change in RGB

-- FOV Circle Setup
local FOVCircle = Drawing.new("Circle")
FOVCircle.Color = Color3.fromRGB(0, 255, 0)
FOVCircle.Thickness = 2
FOVCircle.NumSides = 100
FOVCircle.Radius = 150
FOVCircle.Visible = true
FOVCircle.Filled = false
local FOVCirclePos = Vector2.new(FOVCircle.Radius, FOVCircle.Radius)

-- Lock-on line
local LockLine = Drawing.new("Line")
LockLine.Thickness = 2
LockLine.Visible = true
local LockLineFrom = Vector2.new()
local LockLineTo = Vector2.new()

-- Tracer lines table
local Tracers = {}
local Highlights = {}

-- Pulse timer
local pulseTime = 0

-- === HELLO TEXT ABOVE LOCALPLAYER ===
local helloBillboard
local function createHelloText()
    if not LocalPlayer.Character then return end
    local head = LocalPlayer.Character:FindFirstChild("Head")
    if not head then return end

    helloBillboard = Instance.new("BillboardGui")
    helloBillboard.Adornee = head
    helloBillboard.Size = UDim2.new(0, 100, 0, 50)
    helloBillboard.StudsOffset = Vector3.new(0, 3, 0) -- slightly above head
    helloBillboard.AlwaysOnTop = true
    helloBillboard.Name = "HelloText"
    helloBillboard.Parent = head

    local textLabel = Instance.new("TextLabel")
    textLabel.Size = UDim2.new(1, 0, 1, 0)
    textLabel.BackgroundTransparency = 1
    textLabel.Text = "Hello"
    textLabel.TextColor3 = Color3.fromRGB(0, 255, 0)
    textLabel.TextScaled = true
    textLabel.Font = Enum.Font.SourceSansBold
    textLabel.Parent = helloBillboard

    -- Repeat "Hello" every 1 second
    spawn(function()
        while true do
            textLabel.Text = "Hello"
            wait(1)
        end
    end)
end

if LocalPlayer.Character then
    createHelloText()
end
LocalPlayer.CharacterAdded:Connect(function()
    wait(1) -- give time for head to load
    createHelloText()
end)

-- Function to create a highlight and textlabel for other players
local function createPlayerHighlight(player)
    if player == LocalPlayer then return end

    local function onCharacterAdded(char)
        local highlight = Instance.new("Highlight")
        highlight.Name = "PlayerHighlight"
        highlight.Adornee = char
        highlight.FillColor = Color3.fromRGB(0, 255, 0)
        highlight.FillTransparency = 0.5
        highlight.OutlineColor = Color3.fromRGB(0, 255, 0)
        highlight.OutlineTransparency = 0
        highlight.Parent = char
        Highlights[player] = highlight

        local head = char:WaitForChild("Head", 5)
        if head then
            local billboard = Instance.new("BillboardGui")
            billboard.Name = "PlayerLabel"
            billboard.Adornee = head
            billboard.Size = UDim2.new(0, 100, 0, 50)
            billboard.StudsOffset = Vector3.new(0, 2, 0)
            billboard.AlwaysOnTop = true
            billboard.Parent = head

            local textLabel = Instance.new("TextLabel")
            textLabel.Size = UDim2.new(1, 0, 1, 0)
            textLabel.BackgroundTransparency = 1
            textLabel.Text = player.Name
            textLabel.TextColor3 = Color3.fromRGB(0, 255, 0)
            textLabel.TextScaled = true
            textLabel.Font = Enum.Font.SourceSansBold
            textLabel.Parent = billboard
        end

        local tracer = Drawing.new("Line")
        tracer.Thickness = 2
        tracer.Visible = true
        Tracers[player] = tracer
    end

    if player.Character then
        onCharacterAdded(player.Character)
    end
    player.CharacterAdded:Connect(onCharacterAdded)
end

-- Highlight all current players
for _, player in pairs(Players:GetPlayers()) do
    createPlayerHighlight(player)
end
Players.PlayerAdded:Connect(createPlayerHighlight)

-- Function to find the closest player to the mouse within FOV
local function getClosestPlayer()
    local mousePos = UserInputService:GetMouseLocation()
    local closestPlayer
    local shortestDistance = FOVCircle.Radius

    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("Head") then
            local headPos = Camera:WorldToViewportPoint(player.Character.Head.Position)
            local screenPos = Vector2.new(headPos.X, headPos.Y)
            local distance = (Vector2.new(mousePos.X, mousePos.Y) - screenPos).Magnitude

            if distance < shortestDistance then
                shortestDistance = distance
                closestPlayer = player
            end
        end
    end
    return closestPlayer
end

-- Function to get color based on distance
local function getColorByDistance(distance, maxDistance)
    local t = math.clamp(distance / maxDistance, 0, 1)
    if t < 0.5 then
        return Color3.fromRGB(255, math.floor(255 * (t*2)), 0)
    else
        return Color3.fromRGB(math.floor(255 * (1-t)*2), 255, 255)
    end
end

-- Linear interpolation
local function lerp(a, b, t)
    return a + (b - a) * t
end

-- Update visuals every frame
RunService.RenderStepped:Connect(function(dt)
    local mousePos = UserInputService:GetMouseLocation()
    FOVCirclePos = Vector2.new(lerp(FOVCirclePos.X, mousePos.X, LerpSpeed), lerp(FOVCirclePos.Y, mousePos.Y, LerpSpeed))
    FOVCircle.Position = FOVCirclePos

    -- Lock-on line
    local target = getClosestPlayer()
    if target and target.Character and target.Character:FindFirstChild("Head") then
        local headPos = Camera:WorldToViewportPoint(target.Character.Head.Position)
        LockLineFrom = Vector2.new(lerp(LockLineFrom.X, mousePos.X, LerpSpeed), lerp(LockLineFrom.Y, mousePos.Y, LerpSpeed))
        LockLineTo = Vector2.new(lerp(LockLineTo.X, headPos.X, LerpSpeed), lerp(LockLineTo.Y, headPos.Y, LerpSpeed))
        LockLine.From = LockLineFrom
        LockLine.To = LockLineTo
        LockLine.Color = Color3.fromRGB(255, 0, 0)
        LockLine.Visible = true
    else
        LockLine.Visible = false
    end

    -- Pulse timer
    pulseTime = pulseTime + dt * PulseSpeed
    local pulse = math.sin(pulseTime) * PulseAmount

    -- Update tracer lines and highlight colors
    for player, tracer in pairs(Tracers) do
        if player.Character and player.Character:FindFirstChild("Head") then
            local headPos = Camera:WorldToViewportPoint(player.Character.Head.Position)
            local screenHead = Vector2.new(headPos.X, headPos.Y)
            local bottomCenter = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y)
            
            tracer.From = Vector2.new(lerp(tracer.From.X, bottomCenter.X, LerpSpeed), lerp(tracer.From.Y, bottomCenter.Y, LerpSpeed))
            tracer.To = Vector2.new(lerp(tracer.To.X, screenHead.X, LerpSpeed), lerp(tracer.To.Y, screenHead.Y, LerpSpeed))

            local distance = (bottomCenter - screenHead).Magnitude
            tracer.Color = getColorByDistance(distance, Camera.ViewportSize.Y)
            tracer.Visible = true

            if (Vector2.new(mousePos.X, mousePos.Y) - screenHead).Magnitude <= FOVCircle.Radius then
                local glow = math.clamp(255 + pulse, 0, 255)
                Highlights[player].FillColor = Color3.fromRGB(0, glow, 0)
                Highlights[player].OutlineColor = Color3.fromRGB(0, glow, 0)
                FOVCircle.Color = Color3.fromRGB(0, glow, 0)
            else
                Highlights[player].FillColor = Color3.fromRGB(0, 150, 0)
                Highlights[player].OutlineColor = Color3.fromRGB(0, 150, 0)
                FOVCircle.Color = Color3.fromRGB(0, 255, 0)
            end
        else
            tracer.Visible = false
        end
    end
end)
