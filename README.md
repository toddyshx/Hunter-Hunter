local JustHub = loadstring(game:HttpGet('https://raw.githubusercontent.com/Lilith-VnK/JustHub-UI/refs/heads/main/TestingUpdateUI/JustHub%20(2).lua'))()

JustHub:InitializeUI({
    Name = "Justhub UI",
    SubTitle = "Version 1.0",
    Theme = "Midnight"
})
wait(6)

if not JustHub.Window then
	warn("Window belum dibuat.")
	return
end

local window = JustHub.Window

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")

local LocalPlayer = Players.LocalPlayer
if not LocalPlayer then
    warn("Player tidak ditemukan! Script dihentikan.")
    return
end

local Cooldown = 0
local IsParried = false
local Connection = nil
local LastSpamTime = 0
local BaseSpamInterval = 0.1


local MobileOptimized = false
local lastOptimizedTime = 0

local DEBUG_MODE = true
local function DebugLog(message)
    if DEBUG_MODE then
        print("[DEBUG]: " .. message)
    end
end

local function validateCharacter()
    if not LocalPlayer.Character then return false, "Character belum tersedia" end
    if not LocalPlayer.Character:FindFirstChildOfClass("Humanoid") then return false, "Humanoid tidak ditemukan" end
    if not LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then return false, "HumanoidRootPart tidak ditemukan" end
    return true
end

local function safeCall(fn, ...)
    local success, result = pcall(fn, ...)
    if not success then
        warn("[SafeCall] Error:", result)
        return nil
    end
    return result
end

local function clamp(x, minVal, maxVal)
    if x < minVal then return minVal end
    if x > maxVal then return maxVal end
    return x
end

local function getPingValue()
    local ping = 0
    pcall(function()
        ping = game:GetService("Stats").Network.ServerStatsItem["Data Ping"].Value / 1000
    end)
    return ping
end

local function NetworkLatencyAdjustment()
    local ping = getPingValue()
    if ping > 0.2 then
        return (ping - 0.2)
    else
        return 0
    end
end

local angleDifferences = {}
local maxStoredAngles = 5
local baseCurveThreshold = math.rad(10)
local previousVelocity = nil

local function GetDynamicCurveThreshold(speed)
    local baseThreshold = baseCurveThreshold
    if speed > 50 then
        local factor = clamp(1 - ((speed - 50) / 150), 0.5, 1)
        return baseThreshold * factor
    else
        return baseThreshold
    end
end

local function UpdateAngleDifference(currentVelocity)
    local angleDiff = 0
    if previousVelocity then
        local magProduct = currentVelocity.Magnitude * previousVelocity.Magnitude
        if magProduct > 0 then
            local dotVal = currentVelocity:Dot(previousVelocity) / magProduct
            dotVal = clamp(dotVal, -1, 1)
            angleDiff = math.acos(dotVal)
        end
    end
    previousVelocity = currentVelocity
    table.insert(angleDifferences, angleDiff)
    if #angleDifferences > maxStoredAngles then
        table.remove(angleDifferences, 1)
    end
    return angleDiff
end

local function IsAccelerationHigh(currentVelocity)
    local accDiff = 0
    if previousVelocity then
        accDiff = math.abs(currentVelocity.Magnitude - previousVelocity.Magnitude)
    end
    local accelerationThreshold = 5
    if accDiff > accelerationThreshold then
        DebugLog("Deteksi percepatan mendadak: " .. tostring(accDiff))
        return true
    end
    return false
end

local function IsBallCurving(currentVelocity)
    if IsAccelerationHigh(currentVelocity) then
        return true
    end
    local currentAngleDiff = UpdateAngleDifference(currentVelocity)
    local sum = 0
    for _, a in ipairs(angleDifferences) do
        sum = sum + a
    end
    local averageAngle = (#angleDifferences > 0) and (sum / #angleDifferences) or 0
    local dynamicThreshold = GetDynamicCurveThreshold(currentVelocity.Magnitude)
    DebugLog("Rata-rata perbedaan sudut: " .. tostring(math.deg(averageAngle)) .. "°, Dynamic threshold: " .. tostring(math.deg(dynamicThreshold)) .. "°")
    return averageAngle > dynamicThreshold
end

local function CountNearbyPlayers(radius)
    local count = 0
    local localHRP = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not localHRP then return count end
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
            local otherHRP = player.Character.HumanoidRootPart
            local dist = (localHRP.Position - otherHRP.Position).Magnitude
            if dist <= radius then
                count = count + 1
            end
        end
    end
    return count
end

local ToggleSystem = {
    NormalMode = false,
    SpamMode = false,
    AdvancedMode = false,
    HybridParry = false,
    MultipleParry = false,
    KalmanParry = false
}

local function GetBall()
    local ballsFolder = workspace:FindFirstChild("Balls")
    if not ballsFolder then return nil end
    for _, ball in ipairs(ballsFolder:GetChildren()) do
        if ball:GetAttribute("realBall") then
            return ball
        end
    end
    return nil
end

local function ResetConnection()
    if Connection then
        Connection:Disconnect()
        Connection = nil
    end
end

workspace.Balls.ChildAdded:Connect(function()
    local ball = GetBall()
    if not ball then return end
    ResetConnection()
    previousVelocity = nil 
    local anticipated = false
    Connection = ball:GetAttributeChangedSignal("target"):Connect(function()
        local target = ball:GetAttribute("target")
        if target == LocalPlayer.Name then
            IsParried = false
            anticipated = false
            DebugLog("Target berubah: Bola kini mengarah ke pemain!")
        else
            if IsParried and not anticipated then
                local targetPlayer = game:GetService("Players"):FindFirstChild(target)
                if targetPlayer and targetPlayer.Character and targetPlayer.Character:FindFirstChild("HumanoidRootPart") and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
                    local distBetweenPlayers = (LocalPlayer.Character.HumanoidRootPart.Position - targetPlayer.Character.HumanoidRootPart.Position).Magnitude
                    if distBetweenPlayers < 10 then
                        DebugLog("Anticipation Parry: Target (" .. target .. ") dekat (" .. tostring(distBetweenPlayers) .. " studs).")
                        Parry()
                        anticipated = true
                    end
                end
            end
        end
    end)
end)

local function GetBallVelocity(ball)
    local velocity = nil
    local success, result = pcall(function() return ball.AssemblyLinearVelocity end)
    if success and typeof(result) == "Vector3" then
        velocity = result
    else
        local zoomies = ball:FindFirstChild("zoomies")
        if zoomies and zoomies:IsA("Vector3Value") then
            velocity = zoomies.Value
        end
    end
    return velocity
end

local function Parry()
    if IsParried then return end
    VirtualInputManager:SendMouseButtonEvent(0, 0, 0, true, game, 0)
    task.wait(0.005)
    VirtualInputManager:SendMouseButtonEvent(0, 0, 0, false, game, 0)
    IsParried = true
    Cooldown = tick()
    DebugLog("Parry dilakukan!")
end

local advancedPredictions = {}
local function AdvancedAutoParry(Speed, Distance)
    local pingValue = getPingValue() or 0
    local predictedTime = Distance / math.max(Speed, 0.01)
    table.insert(advancedPredictions, predictedTime)
    if #advancedPredictions > 5 then
        table.remove(advancedPredictions, 1)
    end
    local sumTime = 0
    for _, t in ipairs(advancedPredictions) do
        sumTime = sumTime + t
    end
    local avgPredictedTime = sumTime / #advancedPredictions

    local baseReactionDelay = 0.3
    local speedAdjustment = clamp(0.5 - 0.002 * Speed, 0.2, 0.5)
    local latencyAdjustment = pingValue * 0.5
    local reactionThreshold = baseReactionDelay + speedAdjustment - latencyAdjustment

    local networkAdjustment = NetworkLatencyAdjustment()
    reactionThreshold = reactionThreshold + networkAdjustment

    if Speed > 100 then
        reactionThreshold = reactionThreshold * 0.8
    end

    DebugLog("Advanced Parry: avgPredictedTime = " .. tostring(avgPredictedTime) ..
             ", reactionThreshold = " .. tostring(reactionThreshold) ..
             ", Speed = " .. tostring(Speed) ..
             ", Distance = " .. tostring(Distance) ..
             ", Ping = " .. tostring(pingValue))
    
    return avgPredictedTime < reactionThreshold
end

local function HybridParry(Speed, Distance)
    local freezeThreshold = 1
    if Speed < freezeThreshold then
        DebugLog("Hybrid Parry: Skill freeze terdeteksi (Speed = " .. tostring(Speed) .. "), skip parry.")
        return false
    end

    local normalValue = Distance / math.max(Speed, 0.01)
    local normalThreshold = 0.55

    local baseReactionDelay = 0.3
    local speedAdjustment = clamp(0.5 - 0.002 * Speed, 0.2, 0.5)
    local latencyAdjustment = getPingValue() * 0.5
    local advancedThreshold = baseReactionDelay + speedAdjustment - latencyAdjustment

    local networkAdjustment = NetworkLatencyAdjustment()
    advancedThreshold = advancedThreshold + networkAdjustment

    if Speed > 100 then
        advancedThreshold = advancedThreshold * 0.8
    end

    local combinedThreshold = (normalThreshold + advancedThreshold) / 2

    local nearbyCount = CountNearbyPlayers(20) 
    if nearbyCount > 0 then
        DebugLog("Nearby players count: " .. nearbyCount)
        combinedThreshold = math.max(combinedThreshold - 0.05 * nearbyCount, 0.3)
    end

    if Distance < 10 then
        DebugLog("Hybrid Parry: Kondisi spam terpenuhi (Distance < 10).")
        return true
    end

    DebugLog("Hybrid Parry: normalValue = " .. tostring(normalValue) ..
             ", combinedThreshold = " .. tostring(combinedThreshold))
    
    return normalValue <= combinedThreshold
end

local kalmanEstimate = 0
local kalmanP = 1
local kalmanQ = 0.1
local kalmanR = 0.05
local lastFrameTime = tick()

local function updateKalman(measurement)
    local kalmanK = kalmanP / (kalmanP + kalmanR)
    kalmanEstimate = kalmanEstimate + kalmanK * (measurement - kalmanEstimate)
    kalmanP = (1 - kalmanK) * kalmanP + kalmanQ
    return kalmanEstimate
end

local function KalmanParry(Speed, Distance)
    local measurement = Distance / math.max(Speed, 0.01)
    if kalmanEstimate == 0 then
        kalmanEstimate = measurement
    end
    local currentFrameTime = tick()
    local dt = currentFrameTime - lastFrameTime
    lastFrameTime = currentFrameTime
    if dt > 0.1 then
        kalmanR = 0.2
    else
        kalmanR = 0.05
    end
    local predictedTime = updateKalman(measurement)
    DebugLog("Kalman Parry: predictedTime = " .. tostring(predictedTime))
    local reactionThreshold = 0.35
    return predictedTime < reactionThreshold
end

local function RunMultipleParryMode()
    local ballsFolder = workspace:FindFirstChild("Balls")
    if not ballsFolder then return end
    if not validateCharacter() then return end
    local HRP = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not HRP then return end

    for _, ball in ipairs(ballsFolder:GetChildren()) do
        if ball:GetAttribute("realBall") and ball:GetAttribute("target") == LocalPlayer.Name then
            local distance = (HRP.Position - ball.Position).Magnitude
            if distance < 20 then
                Parry()
                task.wait(0.05)
            end
        end
    end
end

local previousSpamDistance = nil

RunService.PreSimulation:Connect(function()
    if MobileOptimized then
        local currentTime = tick()
        if currentTime - lastOptimizedTime < 0.05 then
            return
        else
            lastOptimizedTime = currentTime
        end
    end

    if ToggleSystem.MultipleParry then
        RunMultipleParryMode()
        return
    end

    local ball = GetBall()
    if not ball then return end
    if ball:GetAttribute("target") ~= LocalPlayer.Name then return end

    local character = LocalPlayer.Character
    if not character then return end

    local HRP = character:FindFirstChild("HumanoidRootPart")
    if not HRP then return end

    local ballVelocity = GetBallVelocity(ball)
    if not ballVelocity or typeof(ballVelocity) ~= "Vector3" then
        DebugLog("Tidak bisa mendapatkan kecepatan bola!")
        return
    end

    if ballVelocity.Y > 50 and ballVelocity.Magnitude > 150 then
        DebugLog("Special Skill Detected: Bola spin upward dan kecepatan tinggi!")
        Parry()
        return
    end

    if IsBallCurving(ballVelocity) then
        DebugLog("Deteksi Curve: Bola sedang melengkung!")
        local curvingThreshold = 15
        local distanceCurve = (HRP.Position - ball.Position).Magnitude
        if distanceCurve < curvingThreshold then
            DebugLog("Anticipation Parry karena bola melengkung dan jarak (" .. tostring(distanceCurve) .. " studs) dekat!")
            Parry()
            return
        end
    end

    local Speed = ballVelocity.Magnitude
    local Distance = (HRP.Position - ball.Position).Magnitude

    if previousSpamDistance then
        if Distance > previousSpamDistance * 2 or Distance < previousSpamDistance / 2 then
            DebugLog("Outlier detected in Distance: " .. tostring(Distance) .. " vs previous " .. tostring(previousSpamDistance))
            previousSpamDistance = Distance
            return
        end
    end
    previousSpamDistance = Distance

    local directionToPlayer = (HRP.Position - ball.Position).Unit
    local ballDir = ballVelocity.Unit
    local angleBetween = math.acos(clamp(ballDir:Dot(directionToPlayer), -1, 1))
    local trajectoryAngleThreshold = math.rad(45)
    if angleBetween > trajectoryAngleThreshold then
        DebugLog("Trajectory validation failed: angle = " .. tostring(math.deg(angleBetween)) .. "° exceeds threshold")
        return
    end

    if ToggleSystem.KalmanParry then
        if KalmanParry(Speed, Distance) then
            Parry()
        end
        return
    end

    if ToggleSystem.HybridParry then
        if HybridParry(Speed, Distance) then
            Parry()
        end
        return
    end

    if ToggleSystem.AdvancedMode then
        if AdvancedAutoParry(Speed, Distance) then
            Parry()
        end
        return
    end

    if ToggleSystem.SpamMode then
        if not validateCharacter() then return end
        local hum = LocalPlayer.Character
        local HRP = hum:FindFirstChild("HumanoidRootPart")
        if not HRP then return end
        local currentDist = (ball.Position - HRP.Position).Magnitude
        local ballVel = GetBallVelocity(ball)
        if not ballVel then return end

        local baseThreshold = 6
        local dynamicOffset = ballVel.Magnitude * 0.05
        local effectiveThreshold = baseThreshold + dynamicOffset

        local spamSmoothedDistance = currentDist
        if hum:FindFirstChild("SpamSmoothedDistance") then
            spamSmoothedDistance = hum.SpamSmoothedDistance.Value
        end
        local alpha = 0.8
        spamSmoothedDistance = alpha * spamSmoothedDistance + (1 - alpha) * currentDist
        if not hum:FindFirstChild("SpamSmoothedDistance") then
            local spamDistanceValue = Instance.new("NumberValue", hum)
            spamDistanceValue.Name = "SpamSmoothedDistance"
            spamDistanceValue.Value = spamSmoothedDistance
        else
            hum.SpamSmoothedDistance.Value = spamSmoothedDistance
        end

        local relativeSpeed = ballVel.Magnitude - HRP.Velocity.Magnitude

        if not hum:FindFirstChild("SpamConditionStart") then
            local conditionStart = Instance.new("NumberValue", hum)
            conditionStart.Name = "SpamConditionStart"
            conditionStart.Value = 0
        end

        if spamSmoothedDistance <= effectiveThreshold and hum:FindFirstChild("Highlight") then
            if hum.SpamConditionStart.Value == 0 then
                hum.SpamConditionStart.Value = tick()
            end
            local conditionDuration = tick() - hum.SpamConditionStart.Value
            if conditionDuration >= 0.2 or relativeSpeed > 5 then
                DebugLog("Spam Parry Condition met: distance="..tostring(spamSmoothedDistance).." effectiveThreshold="..tostring(effectiveThreshold).." relativeSpeed="..tostring(relativeSpeed))
                Parry()
                hum.SpamConditionStart.Value = tick()
            end
        else
            hum.SpamConditionStart.Value = 0
        end
        return
    end

    if ToggleSystem.NormalMode then
        if IsParried and (tick() - Cooldown) < 1 then return end
        if (Distance / math.max(Speed, 0.01)) <= 0.55 then
            Parry()
        end
    end
end)

RunService.RenderStepped:Connect(function(dt)
    local ball = GetBall()
    if ball and ball:GetAttribute("target") ~= LocalPlayer.Name then
        if validateCharacter() then
            local localHRP = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
            local targetPlayer = game:GetService("Players"):FindFirstChild(ball:GetAttribute("target"))
            if targetPlayer and targetPlayer.Character and targetPlayer.Character:FindFirstChild("HumanoidRootPart") then
                local targetHRP = targetPlayer.Character.HumanoidRootPart
                local dist = (localHRP.Position - targetHRP.Position).Magnitude
                if dist < 10 then
                    DebugLog("Anticipation Parry (Local Movement): Local player mendekati target (" .. tostring(dist) .. " studs).")
                    Parry()
                end
            end
        end
    end
end)

local statusGui = Instance.new("ScreenGui")
statusGui.Name = "StatusGui"
statusGui.Parent = LocalPlayer:WaitForChild("PlayerGui")
statusGui.Enabled = true

local statusFrame = Instance.new("Frame")
statusFrame.Size = UDim2.new(0, 200, 0, 120)
statusFrame.Position = UDim2.new(0, 100, 0, 100)
statusFrame.BackgroundTransparency = 0.3
statusFrame.BackgroundColor3 = Color3.new(0, 0, 0)
statusFrame.Active = true
statusFrame.Parent = statusGui

local targetLabel = Instance.new("TextLabel")
targetLabel.Size = UDim2.new(1, 0, 0, 20)
targetLabel.Position = UDim2.new(0, 0, 0, 0)
targetLabel.BackgroundTransparency = 1
targetLabel.TextColor3 = Color3.new(1, 1, 1)
targetLabel.TextScaled = true
targetLabel.Text = "Target: N/A"
targetLabel.Parent = statusFrame

local speedLabel = Instance.new("TextLabel")
speedLabel.Size = UDim2.new(1, 0, 0, 20)
speedLabel.Position = UDim2.new(0, 0, 0, 22)
speedLabel.BackgroundTransparency = 1
speedLabel.TextColor3 = Color3.new(1, 1, 1)
speedLabel.TextScaled = true
speedLabel.Text = "Speed: N/A"
speedLabel.Parent = statusFrame

local distanceLabel = Instance.new("TextLabel")
distanceLabel.Size = UDim2.new(1, 0, 0, 20)
distanceLabel.Position = UDim2.new(0, 0, 0, 44)
distanceLabel.BackgroundTransparency = 1
distanceLabel.TextColor3 = Color3.new(1, 1, 1)
distanceLabel.TextScaled = true
distanceLabel.Text = "Distance: N/A"
distanceLabel.Parent = statusFrame

local curveLabel = Instance.new("TextLabel")
curveLabel.Size = UDim2.new(1, 0, 0, 20)
curveLabel.Position = UDim2.new(0, 0, 0, 66)
curveLabel.BackgroundTransparency = 1
curveLabel.TextColor3 = Color3.new(1, 1, 1)
curveLabel.TextScaled = true
curveLabel.Text = "Curve: N/A"
curveLabel.Parent = statusFrame

local suggestionLabel = Instance.new("TextLabel")
suggestionLabel.Size = UDim2.new(1, 0, 0, 20)
suggestionLabel.Position = UDim2.new(0, 0, 0, 88)
suggestionLabel.BackgroundTransparency = 1
suggestionLabel.TextColor3 = Color3.new(1, 1, 1)
suggestionLabel.TextScaled = true
suggestionLabel.Text = "Suggestion: N/A"
suggestionLabel.Parent = statusFrame

local dragging = false
local dragInput, dragStart, startPos

statusFrame.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
        dragStart = input.Position
        startPos = statusFrame.AbsolutePosition
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

statusFrame.InputChanged:Connect(function(input)
    if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
        local delta = input.Position - dragStart
        statusFrame.Position = UDim2.new(0, startPos.X + delta.X, 0, startPos.Y + delta.Y)
    end
end)

RunService.RenderStepped:Connect(function()
    local ball = GetBall()
    if ball then
        local targetAttr = ball:GetAttribute("target") or "N/A"
        targetLabel.Text = "Target: " .. tostring(targetAttr)
        
        local ballVel = GetBallVelocity(ball)
        if ballVel then
            speedLabel.Text = "Speed: " .. string.format("%.2f", ballVel.Magnitude)
        else
            speedLabel.Text = "Speed: N/A"
        end

        if validateCharacter() then
            local HRP = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
            if HRP and ball then
                local dist = (HRP.Position - ball.Position).Magnitude
                distanceLabel.Text = "Distance: " .. string.format("%.2f", dist)
            else
                distanceLabel.Text = "Distance: N/A"
            end
        end

        if ballVel then
            local curving = IsBallCurving(ballVel)
            curveLabel.Text = "Curve: " .. (curving and "Yes" or "No")
        else
            curveLabel.Text = "Curve: N/A"
        end

        if ballVel and validateCharacter() then
            local HRP = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
            local dist = HRP and (HRP.Position - ball.Position).Magnitude or 0
            local suggestion = "Parry Not Recommended"
            if HybridParry(ballVel.Magnitude, dist) then
                suggestion = "Parry Recommended"
            end
            suggestionLabel.Text = "Suggestion: " .. suggestion
        else
            suggestionLabel.Text = "Suggestion: N/A"
        end
    else
        targetLabel.Text = "Target: N/A"
        speedLabel.Text = "Speed: N/A"
        distanceLabel.Text = "Distance: N/A"
        curveLabel.Text = "Curve: N/A"
        suggestionLabel.Text = "Suggestion: N/A"
    end
end)

local Home = window:addTab("•Home")
local ParryTab = Home:addSection("Home Section", 120)

ParryTab:addToggle({
    Name = "Normal Parry (50%)",
    Default = false,
    Callback = function(state)
        ToggleSystem.NormalMode = state
        if state then
            ToggleSystem.SpamMode = false
            ToggleSystem.AdvancedMode = false
            ToggleSystem.HybridParry = false
            ToggleSystem.MultipleParry = false
            ToggleSystem.KalmanParry = false
            DebugLog("Mode Normal diaktifkan, mode lainnya dinonaktifkan.")
        else
            DebugLog("Mode Normal dinonaktifkan.")
        end
    end
})

ParryTab:addToggle({
    Name = "Spam Parry (10%) (Not recommended)",
    Default = false,
    Callback = function(state)
        ToggleSystem.SpamMode = state
        if state then
            ToggleSystem.NormalMode = false
            ToggleSystem.AdvancedMode = false
            ToggleSystem.HybridParry = false
            ToggleSystem.MultipleParry = false
            ToggleSystem.KalmanParry = false
            DebugLog("Mode Spam diaktifkan, mode lainnya dinonaktifkan.")
        else
            DebugLog("Mode Spam dinonaktifkan.")
        end
    end
})

ParryTab:addToggle({
    Name = "Advanced Auto Parry (55%)",
    Default = false,
    Callback = function(state)
        ToggleSystem.AdvancedMode = state
        if state then
            ToggleSystem.NormalMode = false
            ToggleSystem.SpamMode = false
            ToggleSystem.HybridParry = false
            ToggleSystem.MultipleParry = false
            ToggleSystem.KalmanParry = false
            DebugLog("Advanced Auto Parry diaktifkan, mode lainnya dinonaktifkan.")
        else
            DebugLog("Advanced Auto Parry dinonaktifkan.")
        end
    end
})

ParryTab:addToggle({
    Name = "Hybrid Parry (80%) (recommended)",
    Default = false,
    Callback = function(state)
        ToggleSystem.HybridParry = state
        if state then
            ToggleSystem.NormalMode = false
            ToggleSystem.SpamMode = false
            ToggleSystem.AdvancedMode = false
            ToggleSystem.MultipleParry = false
            ToggleSystem.KalmanParry = false
            DebugLog("Hybrid Parry diaktifkan, mode lainnya dinonaktifkan.")
        else
            DebugLog("Hybrid Parry dinonaktifkan.")
        end
    end
})

ParryTab:addToggle({
    Name = "Multiple Parry (Dungeon Recommended)",
    Default = false,
    Callback = function(state)
        ToggleSystem.MultipleParry = state
        if state then
            ToggleSystem.NormalMode = false
            ToggleSystem.SpamMode = false
            ToggleSystem.AdvancedMode = false
            ToggleSystem.HybridParry = false
            ToggleSystem.KalmanParry = false
            DebugLog("Multiple Parry (Dungeon Recommended) diaktifkan.")
        else
            DebugLog("Multiple Parry (Dungeon Recommended) dinonaktifkan.")
        end
    end
})

ParryTab:addToggle({
    Name = "Auto Parry V3 (Beta Test)",
    Default = false,
    Callback = function(state)
        ToggleSystem.KalmanParry = state
        if state then
            ToggleSystem.NormalMode = false
            ToggleSystem.SpamMode = false
            ToggleSystem.AdvancedMode = false
            ToggleSystem.HybridParry = false
            ToggleSystem.MultipleParry = false
            DebugLog("Auto Parry V3 diaktifkan, mode lainnya dinonaktifkan.")
            kalmanEstimate = 0
            kalmanP = 1
        else
            DebugLog("Auto Parry V3 dinonaktifkan.")
        end
    end
})

local Settings = window:addTab("•Settings")
local SettingsTab = Settings:addSection("Settings", 120)

local walkSpeedSlider = SettingsTab:addSlider({
    Name = "WalkSpeed",
    Min = 16,
    Max = 500,
    Default = 50,
    Color = Color3.fromRGB(255,255,255),
    Increment = 1,
    ValueName = "WalkSpeed",
    Callback = function(Value)
        if validateCharacter() then
            LocalPlayer.Character.Humanoid.WalkSpeed = Value
        end
    end    
})

local jumpPowerSlider = SettingsTab:addSlider({
    Name = "JumpPower",
    Min = 50,
    Max = 500,
    Default = 50,
    Color = Color3.fromRGB(255,255,255),
    Increment = 1,
    ValueName = "JumpPower",
    Callback = function(Value)
        if validateCharacter() then
            LocalPlayer.Character.Humanoid.JumpPower = Value
        end
    end    
})

SettingsTab:addToggle({
    Name = "No Clip",
    Default = false,
    Callback = function(Value)
        spawn(function()
            while Value do
                if validateCharacter() then
                    for _, part in pairs(LocalPlayer.Character:GetChildren()) do
                        if part:IsA("BasePart") then
                            part.CanCollide = false
                        end
                    end
                end
                task.wait(0.1)
            end
        end)
    end
})

SettingsTab:addToggle({
    Name = "Walk On Water",
    Default = false,
    Callback = function(Value)
        spawn(function()
            while Value do
                if validateCharacter() then
                    local root = LocalPlayer.Character.HumanoidRootPart
                    if root and root.Position.Y < 5 then
                        root.Velocity = Vector3.new(root.Velocity.X, 2, root.Velocity.Z)
                    end
                end
                task.wait(0.1)
            end
        end)
    end
})

SettingsTab:addToggle({
    Name = "Infinity Jump",
    Default = false,
    Callback = function(Value)
        UserInputService.JumpRequest:Connect(function()
            if Value and validateCharacter() then
                LocalPlayer.Character.Humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
            end
        end)
    end
})

SettingsTab:addToggle({
    Name = "Mobile Optimization",
    Default = false,
    Callback = function(Value)
        MobileOptimized = Value
        DebugLog("Mobile Optimization: " .. tostring(MobileOptimized))
    end
})

SettingsTab:addButton({
    Name = "Anti AFK",
    Callback = function()
        LocalPlayer.Idled:Connect(function()
            safeCall(function()
                local vu = game:GetService("VirtualUser")
                vu:CaptureController()
                vu:ClickButton2(Vector2.new())
            end)
        end)
    end
})

SettingsTab:addDropdown({
 Name = "Theme",
 Items = {"Darker","Dark","Purple","Light","Neon","Forest","Aqua","Crimson","Solar","Pastel","Cyber","Ocean","Desert","Galaxy","Vintage","Rainbow","Midnight"},
 Default = "",
 Callback = function(choice)
     JustHub:SetTheme(choice)
 end
})
