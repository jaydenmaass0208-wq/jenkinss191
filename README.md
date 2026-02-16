-- PART 1 — CORE LOGIC + SCANNER + STEAL SYSTEM

-- https://discord.gg/MT6BXdH9q

local CONFIG = {
    AUTO_STEAL_NEAREST = false,
}

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")

local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

local AnimalsData = require(ReplicatedStorage:WaitForChild("Datas"):WaitForChild("Animals"))

local allAnimalsCache = {}
local PromptMemoryCache = {}
local InternalStealCache = {}
local LastTargetUID = nil
local LastPlayerPosition = nil
local PlayerVelocity = Vector3.zero

local AUTO_STEAL_PROX_RADIUS = 20
local IsStealing = false
local StealProgress = 0
local CurrentStealTarget = nil
local StealStartTime = 0

local CIRCLE_RADIUS = AUTO_STEAL_PROX_RADIUS
local PART_THICKNESS = 0.3
local PART_HEIGHT = 0.2
local PART_COLOR = Color3.fromRGB(0, 170, 255) -- Neon Blue
local PartsCount = 65
local circleParts = {}
local circleEnabled = true

local stealConnection = nil
local velocityConnection = nil

local function getHRP()
    local char = LocalPlayer.Character
    if not char then return nil end
    return char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("UpperTorso")
end

local function isMyBase(plotName)
    local plot = workspace.Plots:FindFirstChild(plotName)
    if not plot then return false end
    
    local sign = plot:FindFirstChild("PlotSign")
    if sign then
        local yourBase = sign:FindFirstChild("YourBase")
        if yourBase and yourBase:IsA("BillboardGui") then
            return yourBase.Enabled == true
        end
    end
    return false
end

local function scanSinglePlot(plot)
    if not plot or not plot:IsA("Model") then return end
    if isMyBase(plot.Name) then return end
    
    local podiums = plot:FindFirstChild("AnimalPodiums")
    if not podiums then return end
    
    for _, podium in ipairs(podiums:GetChildren()) do
        if podium:IsA("Model") and podium:FindFirstChild("Base") then
            local animalName = "Unknown"
            local spawn = podium.Base:FindFirstChild("Spawn")
            if spawn then
                for _, child in ipairs(spawn:GetChildren()) do
                    if child:IsA("Model") and child.Name ~= "PromptAttachment" then
                        animalName = child.Name
                        local animalInfo = AnimalsData[animalName]
                        if animalInfo and animalInfo.DisplayName then
                            animalName = animalInfo.DisplayName
                        end
                        break
                    end
                end
            end
            
            table.insert(allAnimalsCache, {
                name = animalName,
                plot = plot.Name,
                slot = podium.Name,
                worldPosition = podium:GetPivot().Position,
                uid = plot.Name .. "_" .. podium.Name,
            })
        end
    end
end

local function initializeScanner()
    task.wait(2)
    
    local plots = workspace:WaitForChild("Plots", 10)
    if not plots then 
        return
    end
    
    for _, plot in ipairs(plots:GetChildren()) do
        if plot:IsA("Model") then
            scanSinglePlot(plot)
        end
    end
    
    plots.ChildAdded:Connect(function(plot)
        if plot:IsA("Model") then
            task.wait(0.5)
            scanSinglePlot(plot)
        end
    end)
    
    task.spawn(function()
        while task.wait(5) do
            allAnimalsCache = {}
            for _, plot in ipairs(plots:GetChildren()) do
                if plot:IsA("Model") then
                    scanSinglePlot(plot)
                end
            end
        end
    end)
end

local function findProximityPromptForAnimal(animalData)
    if not animalData then return nil end
    
    local cachedPrompt = PromptMemoryCache[animalData.uid]
    if cachedPrompt and cachedPrompt.Parent then
        return cachedPrompt
    end
    
    local plot = workspace.Plots:FindFirstChild(animalData.plot)
    if not plot then return nil end
    
    local podiums = plot:FindFirstChild("AnimalPodiums")
    if not podiums then return nil end
    
    local podium = podiums:FindFirstChild(animalData.slot)
    if not podium then return nil end
    
    local base = podium:FindFirstChild("Base")
    if not base then return nil end
    
    local spawn = base:FindFirstChild("Spawn")
    if not spawn then return nil end
    
    local attach = spawn:FindFirstChild("PromptAttachment")
    if not attach then return nil end
    
    for _, p in ipairs(attach:GetChildren()) do
        if p:IsA("ProximityPrompt") then
            PromptMemoryCache[animalData.uid] = p
            return p
        end
    end
    
    return nil
end

local function updatePlayerVelocity()
    local hrp = getHRP()
    if not hrp then return end
    
    local currentPos = hrp.Position
    
    if LastPlayerPosition then
        PlayerVelocity = (currentPos - LastPlayerPosition) / task.wait()
    end
    
    LastPlayerPosition = currentPos
end

local function shouldSteal(animalData)
    if not animalData or not animalData.worldPosition then return false end
    
    local hrp = getHRP()
    if not hrp then return false end
    
    local currentDistance = (hrp.Position - animalData.worldPosition).Magnitude
    
    return currentDistance <= AUTO_STEAL_PROX_RADIUS
end

local function buildStealCallbacks(prompt)
    if InternalStealCache[prompt] then return end
    
    local data = {
        holdCallbacks = {},
        triggerCallbacks = {},
        ready = true,
    }
    
    local ok1, conns1 = pcall(getconnections, prompt.PromptButtonHoldBegan)
    if ok1 and type(conns1) == "table" then
        for _, conn in ipairs(conns1) do
            if type(conn.Function) == "function" then
                table.insert(data.holdCallbacks, conn.Function)
            end
        end
    end
    
    local ok2, conns2 = pcall(getconnections, prompt.Triggered)
    if ok2 and type(conns2) == "table" then
        for _, conn in ipairs(conns2) do
            if type(conn.Function) == "function" then
                table.insert(data.triggerCallbacks, conn.Function)
            end
        end
    end
    
    if (#data.holdCallbacks > 0) or (#data.triggerCallbacks > 0) then
        InternalStealCache[prompt] = data
    end
end

local function executeInternalStealAsync(prompt, animalData)
    local data = InternalStealCache[prompt]
    if not data or not data.ready then return false end
    
    data.ready = false
    IsStealing = true
    StealProgress = 0
    CurrentStealTarget = animalData
    StealStartTime = tick()
    
    task.spawn(function()
        if #data.holdCallbacks > 0 then
            for _, fn in ipairs(data.holdCallbacks) do
                task.spawn(fn)
            end
        end
        
        local startTime = tick()
        while tick() - startTime < 1.3 do
            StealProgress = (tick() - startTime) / 1.3
            task.wait(0.05)
        end
        StealProgress = 1
        
        if #data.triggerCallbacks > 0 then
            for _, fn in ipairs(data.triggerCallbacks) do
                task.spawn(fn)
            end
        end
        
        task.wait(0.1)
        data.ready = true
        
        task.wait(0.3)
        IsStealing = false
        StealProgress = 0
        CurrentStealTarget = nil
    end)
    
    return true
end

local function attemptSteal(prompt, animalData)
    if not prompt or not prompt.Parent then return false end
    
    buildStealCallbacks(prompt)
    if not InternalStealCache[prompt] then return false end
    
    return executeInternalStealAsync(prompt, animalData)
end

local function getNearestAnimal()
    local hrp = getHRP()
    if not hrp then return nil end
    
    local nearest = nil
    local minDist = math.huge
    
    for _, animalData in ipairs(allAnimalsCache) do
        if isMyBase(animalData.plot) then continue end
        
        if animalData.worldPosition then
            local dist = (hrp.Position - animalData.worldPosition).Magnitude
            if dist < minDist then
                minDist = dist
                nearest = animalData
            end
        end
    end
    
    return nearest
end

local function autoStealLoop()
    if stealConnection then stealConnection:Disconnect() end
    if velocityConnection then velocityConnection:Disconnect() end
    
    velocityConnection = RunService.Heartbeat:Connect(updatePlayerVelocity)
    
    stealConnection = RunService.Heartbeat:Connect(function()
        if not CONFIG.AUTO_STEAL_NEAREST then return end
        if IsStealing then return end
        
        local targetAnimal = getNearestAnimal()
        if not targetAnimal then return end
        
        if not shouldSteal(targetAnimal) then return end
        
        if LastTargetUID ~= targetAnimal.uid then
            LastTargetUID = targetAnimal.uid
        end
        
        local prompt = PromptMemoryCache[targetAnimal.uid]
        if not prompt or not prompt.Parent then
            prompt = findProximityPromptForAnimal(targetAnimal)
        end
        
        if prompt then
            attemptSteal(prompt, targetAnimal)
        end
    end)
end
-- PART 2 — GUI + SPARK HUB + NEON THEME

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "AutoStealUI"
screenGui.ResetOnSpawn = false
screenGui.DisplayOrder = 999999
screenGui.Parent = PlayerGui

-- SPARK HUB TITLE (Blue + Yellow Neon)
local title = Instance.new("TextLabel")
title.Name = "SparkHubTitle"
title.Size = UDim2.new(1, 0, 0, 50)
title.Position = UDim2.new(0, 0, 0, 0)
title.BackgroundTransparency = 1
title.Text = "SPARK HUB"
title.Font = Enum.Font.GothamBlack
title.TextSize = 38
title.TextColor3 = Color3.fromRGB(0, 170, 255) -- Neon Blue
title.TextStrokeTransparency = 0
title.TextStrokeColor3 = Color3.fromRGB(255, 255, 0) -- Neon Yellow outline
title.Parent = screenGui

local titleGlow = Instance.new("UIGradient")
titleGlow.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(0, 170, 255)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(255, 255, 0)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(0, 170, 255))
})
titleGlow.Rotation = 45
titleGlow.Parent = title

-- BUTTON FRAME
local buttonFrame = Instance.new("Frame")
buttonFrame.Name = "ButtonFrame"
buttonFrame.Size = UDim2.new(0, 145, 0, 45)
buttonFrame.Position = UDim2.new(0, 10, 0, 60)
buttonFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
buttonFrame.BackgroundTransparency = 0.2
buttonFrame.BorderSizePixel = 0
buttonFrame.Parent = screenGui

local frameCorner = Instance.new("UICorner")
frameCorner.CornerRadius = UDim.new(0, 10)
frameCorner.Parent = buttonFrame

local frameStroke = Instance.new("UIStroke")
frameStroke.Thickness = 1.3
frameStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
frameStroke.Color = Color3.fromRGB(255, 255, 255)
frameStroke.Parent = buttonFrame

local frameGradient = Instance.new("UIGradient")
frameGradient.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0.00, Color3.fromRGB(0, 170, 255)),
    ColorSequenceKeypoint.new(0.25, Color3.fromRGB(255, 255, 0)),
    ColorSequenceKeypoint.new(0.50, Color3.fromRGB(0, 170, 255)),
    ColorSequenceKeypoint.new(0.75, Color3.fromRGB(255, 255, 0)),
    ColorSequenceKeypoint.new(1.00, Color3.fromRGB(0, 170, 255))
})
frameGradient.Parent = frameStroke

task.spawn(function()
    while true do
        frameGradient.Rotation = frameGradient.Rotation + 2
        task.wait(0.02)
    end
end)

-- BUTTON
local button = Instance.new("TextButton")
button.Name = "AutoStealButton"
button.Size = UDim2.new(0, 135, 0, 35)
button.AnchorPoint = Vector2.new(0.5, 0.5)
button.Position = UDim2.new(0.5, 0, 0.5, 0)
button.BackgroundColor3 = Color3.fromRGB(255, 255, 0) -- Yellow (OFF)
button.Text = "INSTA GRAB: OFF"
button.Font = Enum.Font.GothamBold
button.TextSize = 13
button.TextColor3 = Color3.fromRGB(0, 0, 0)
button.BorderSizePixel = 0
button.Parent = buttonFrame

local buttonCorner = Instance.new("UICorner")
buttonCorner.CornerRadius = UDim.new(0, 8)
buttonCorner.Parent = button

button.MouseButton1Click:Connect(function()
    CONFIG.AUTO_STEAL_NEAREST = not CONFIG.AUTO_STEAL_NEAREST

    if CONFIG.AUTO_STEAL_NEAREST then
        button.Text = "INSTA GRAB: ON"
        button.BackgroundColor3 = Color3.fromRGB(0, 170, 255) -- Blue
        button.TextColor3 = Color3.fromRGB(255, 255, 255)
    else
        button.Text = "INSTA GRAB: OFF"
        button.BackgroundColor3 = Color3.fromRGB(255, 255, 0) -- Yellow
        button.TextColor3 = Color3.fromRGB(0, 0, 0)
    end
end)

-- PROGRESS BAR FRAME (LOWERED TO SHOW SPARK HUB TITLE)
local showbarFrame = Instance.new("Frame")
showbarFrame.Size = UDim2.new(0, 220, 0, 22)
showbarFrame.Position = UDim2.new(0.5, -110, 0, 70) -- FIXED (was 20)
showbarFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
showbarFrame.BackgroundTransparency = 0.2
showbarFrame.BorderSizePixel = 0
showbarFrame.Visible = true
showbarFrame.Parent = screenGui

local showbarCorner = Instance.new("UICorner")
showbarCorner.CornerRadius = UDim.new(0, 6)
showbarCorner.Parent = showbarFrame

local uiStroke = Instance.new("UIStroke")
uiStroke.Thickness = 1.2
uiStroke.Transparency = 0
uiStroke.Color = Color3.fromRGB(255, 255, 255)
uiStroke.Parent = showbarFrame

local uiGradient = Instance.new("UIGradient")
uiGradient.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0.00, Color3.fromRGB(0, 170, 255)),
    ColorSequenceKeypoint.new(0.20, Color3.fromRGB(255, 255, 0)),
    ColorSequenceKeypoint.new(0.50, Color3.fromRGB(0, 170, 255)),
    ColorSequenceKeypoint.new(0.80, Color3.fromRGB(255, 255, 0)),
    ColorSequenceKeypoint.new(1.00, Color3.fromRGB(0, 170, 255))
})
uiGradient.Parent = uiStroke

-- PROGRESS BAR
local progressBarBg = Instance.new("Frame")
progressBarBg.Size = UDim2.new(0.9, 0, 0, 8)
progressBarBg.Position = UDim2.new(0.05, 0, 0.5, -4)
progressBarBg.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
progressBarBg.BorderSizePixel = 0
progressBarBg.Parent = showbarFrame

local progressBarCorner = Instance.new("UICorner")
progressBarCorner.CornerRadius = UDim.new(1, 0)
progressBarCorner.Parent = progressBarBg

local progressBarFill = Instance.new("Frame")
progressBarFill.Size = UDim2.new(0, 0, 1, 0)
progressBarFill.Position = UDim2.new(0, 0, 0, 0)
progressBarFill.BackgroundColor3 = Color3.fromRGB(0, 170, 255)
progressBarFill.BorderSizePixel = 0
progressBarFill.Parent = progressBarBg

local fillCorner = Instance.new("UICorner")
fillCorner.CornerRadius = UDim.new(1, 0)
fillCorner.Parent = progressBarFill

local fillGradient = Instance.new("UIGradient")
fillGradient.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(0, 170, 255)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(255, 255, 0)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(0, 170, 255))
})
fillGradient.Parent = progressBarFill

-- RADIUS FRAME
local radiusFrame = Instance.new("Frame")
radiusFrame.Size = UDim2.new(0, 40, 0, 22)
radiusFrame.Position = UDim2.new(0.5, 115, 0, 70)
radiusFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
radiusFrame.BackgroundTransparency = 0.2
radiusFrame.BorderSizePixel = 0
radiusFrame.Visible = true
radiusFrame.Parent = screenGui

local radiusCorner = Instance.new("UICorner")
radiusCorner.CornerRadius = UDim.new(0, 6)
radiusCorner.Parent = radiusFrame

local radiusStroke = Instance.new("UIStroke")
radiusStroke.Thickness = 1.2
radiusStroke.Transparency = 0
radiusStroke.Color = Color3.fromRGB(255, 255, 255)
radiusStroke.Parent = radiusFrame

local radiusGradient = Instance.new("UIGradient")
radiusGradient.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0.00, Color3.fromRGB(0, 170, 255)),
    ColorSequenceKeypoint.new(0.20, Color3.fromRGB(255, 255, 0)),
    ColorSequenceKeypoint.new(0.50, Color3.fromRGB(0, 170, 255)),
    ColorSequenceKeypoint.new(0.80, Color3.fromRGB(255, 255, 0)),
    ColorSequenceKeypoint.new(1.00, Color3.fromRGB(0, 170, 255))
})
radiusGradient.Parent = radiusStroke

local radiusText = Instance.new("TextButton")
radiusText.Size = UDim2.new(1, 0, 1, 0)
radiusText.Position = UDim2.new(0, 0, 0, 0)
radiusText.BackgroundTransparency = 1
radiusText.Text = AUTO_STEAL_PROX_RADIUS
radiusText.Font = Enum.Font.GothamBold
radiusText.TextSize = 13
radiusText.TextColor3 = Color3.fromRGB(255, 255, 255)
radiusText.Parent = radiusFrame
-- PART 3 — CIRCLE SYSTEM + PROGRESS LOGIC + FINAL CONNECTIONS

local typing = false
local inputConnection

-- CREATE CIRCLE AROUND PLAYER
local function createCircle(character)
    for _, part in ipairs(circleParts) do
        if part then part:Destroy() end
    end
    circleParts = {}

    CIRCLE_RADIUS = AUTO_STEAL_PROX_RADIUS
    local root = character:WaitForChild("HumanoidRootPart")

    local points = {}
    for i = 0, PartsCount - 1 do
        local angle = math.rad(i * 360 / PartsCount)
        table.insert(points, Vector3.new(math.cos(angle), 0, math.sin(angle)) * CIRCLE_RADIUS)
    end

    for i = 1, #points do
        local nextIndex = i % #points + 1
        local p1 = points[i]
        local p2 = points[nextIndex]

        local part = Instance.new("Part")
        part.Anchored = true
        part.CanCollide = false
        part.Size = Vector3.new((p2 - p1).Magnitude, PART_HEIGHT, PART_THICKNESS)
        part.Color = PART_COLOR
        part.Material = Enum.Material.Neon
        part.Transparency = 0.3
        part.TopSurface = Enum.SurfaceType.Smooth
        part.BottomSurface = Enum.SurfaceType.Smooth
        part.Parent = workspace
        table.insert(circleParts, part)
    end
end

-- UPDATE CIRCLE POSITION
local function updateCircle(character)
    local root = character:FindFirstChild("HumanoidRootPart")
    if not root then return end

    local points = {}
    for i = 0, PartsCount - 1 do
        local angle = math.rad(i * 360 / PartsCount)
        table.insert(points, Vector3.new(math.cos(angle), 0, math.sin(angle)) * CIRCLE_RADIUS)
    end

    for i, part in ipairs(circleParts) do
        local nextIndex = i % #points + 1
        local p1 = points[i]
        local p2 = points[nextIndex]
        local center = (p1 + p2) / 2 + root.Position

        part.CFrame = CFrame.new(center, center + Vector3.new(p2.X - p1.X, 0, p2.Z - p1.Z))
            * CFrame.Angles(0, math.pi/2, 0)
    end
end

-- CHARACTER ADDED
local function onCharacterAdded(character)
    if circleEnabled then
        createCircle(character)
        RunService:BindToRenderStep("CircleFollow", Enum.RenderPriority.Camera.Value + 1, function()
            updateCircle(character)
        end)
    end
end

-- UPDATE RADIUS
local function updateCircleRadius()
    CIRCLE_RADIUS = AUTO_STEAL_PROX_RADIUS
    local character = LocalPlayer.Character
    if character and circleEnabled then
        createCircle(character)
    end
end

-- RADIUS TEXT INPUT
radiusText.MouseButton1Click:Connect(function()
    if typing then return end

    typing = true
    local textBox = Instance.new("TextBox")
    textBox.Size = UDim2.new(1, 0, 1, 0)
    textBox.BackgroundTransparency = 1
    textBox.Text = AUTO_STEAL_PROX_RADIUS
    textBox.Font = Enum.Font.GothamBold
    textBox.TextSize = 13
    textBox.TextColor3 = Color3.fromRGB(255, 255, 255)
    textBox.ClearTextOnFocus = false
    textBox.Parent = radiusFrame

    textBox:CaptureFocus()

    inputConnection = textBox.FocusLost:Connect(function(enterPressed)
        if enterPressed then
            local num = tonumber(textBox.Text)
            if num and num >= 5 and num <= 200 then
                AUTO_STEAL_PROX_RADIUS = math.floor(num)
                updateCircleRadius()
            end
        end

        textBox:Destroy()
        radiusText.Text = AUTO_STEAL_PROX_RADIUS
        typing = false
        inputConnection:Disconnect()
    end)
end)

-- PROGRESS BAR UPDATE LOOP
local progressTween = nil

task.spawn(function()
    while task.wait(0.03) do
        if not typing then
            radiusText.Text = AUTO_STEAL_PROX_RADIUS
        end

        if IsStealing then
            if progressTween then progressTween:Cancel() end

            progressTween = TweenService:Create(
                progressBarFill,
                TweenInfo.new(0.1, Enum.EasingStyle.Linear),
                { Size = UDim2.new(StealProgress, 0, 1, 0) }
            )
            progressTween:Play()
        else
            if progressTween then
                progressTween:Cancel()
                progressTween = nil
            end

            if progressBarFill.Size.X.Scale > 0 then
                progressBarFill.Size = UDim2.new(
                    math.max(0, progressBarFill.Size.X.Scale - 0.03),
                    0,
                    1,
                    0
                )
            end
        end
    end
end)

-- INITIALIZE EVERYTHING
initializeScanner()
autoStealLoop()

if LocalPlayer.Character then
    onCharacterAdded(LocalPlayer.Character)
end

LocalPlayer.CharacterAdded:Connect(onCharacterAdded)
