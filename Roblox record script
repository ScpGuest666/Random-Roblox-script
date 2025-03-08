-- Create ScreenGui
local screenGui = Instance.new("ScreenGui")
screenGui.Parent = game.Players.LocalPlayer:WaitForChild("PlayerGui")
screenGui.ResetOnSpawn = false

-- Create Frame
local frame = Instance.new("Frame")
frame.Parent = screenGui
frame.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
frame.Size = UDim2.new(0, 220, 0, 200)
frame.Position = UDim2.new(0.5, -110, 0.5, -100)
frame.Active = true
frame.Draggable = true

-- Create Buttons
local recordButton = Instance.new("TextButton")
recordButton.Parent = frame
recordButton.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
recordButton.Size = UDim2.new(0, 60, 0, 30)
recordButton.Position = UDim2.new(0, 10, 0, 20)
recordButton.Text = "Record"
recordButton.TextScaled = true

local stopRecordButton = Instance.new("TextButton")
stopRecordButton.Parent = frame
stopRecordButton.BackgroundColor3 = Color3.fromRGB(255, 165, 0)
stopRecordButton.Size = UDim2.new(0, 60, 0, 30)
stopRecordButton.Position = UDim2.new(0, 80, 0, 20)
stopRecordButton.Text = "Stop Record"
stopRecordButton.TextScaled = true

local stopReplayButton = Instance.new("TextButton")
stopReplayButton.Parent = frame
stopReplayButton.BackgroundColor3 = Color3.fromRGB(255, 100, 100)
stopReplayButton.Size = UDim2.new(0, 60, 0, 30)
stopReplayButton.Position = UDim2.new(0, 150, 0, 20)
stopReplayButton.Text = "Stop Replay"
stopReplayButton.TextScaled = true

local destroyButton = Instance.new("TextButton")
destroyButton.Parent = frame
destroyButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
destroyButton.Size = UDim2.new(0, 90, 0, 30)
destroyButton.Position = UDim2.new(0, 10, 0, 60)
destroyButton.Text = "Destroy"
destroyButton.TextScaled = true

local deleteButton = Instance.new("TextButton")
deleteButton.Parent = frame
deleteButton.BackgroundColor3 = Color3.fromRGB(200, 100, 100)
deleteButton.Size = UDim2.new(0, 90, 0, 30)
deleteButton.Position = UDim2.new(0, 120, 0, 60)
deleteButton.Text = "Delete"
deleteButton.TextScaled = true

-- Status Indicator
local statusLabel = Instance.new("TextLabel")
statusLabel.Parent = frame
statusLabel.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
statusLabel.Size = UDim2.new(0, 220, 0, 30)
statusLabel.Position = UDim2.new(0, 0, 0, -30)
statusLabel.Text = "Status: Idle"
statusLabel.TextColor3 = Color3.fromRGB(0, 0, 0)
statusLabel.TextScaled = true

-- Create Scrollable List
local scrollFrame = Instance.new("ScrollingFrame")
scrollFrame.Parent = frame
scrollFrame.Size = UDim2.new(0, 200, 0, 70)
scrollFrame.Position = UDim2.new(0, 10, 0, 100)
scrollFrame.CanvasSize = UDim2.new(0, 0, 0, 0) -- Dynamically adjusted
scrollFrame.ScrollBarThickness = 8

local uiListLayout = Instance.new("UIListLayout")
uiListLayout.Parent = scrollFrame
uiListLayout.Padding = UDim.new(0, 5)

-- Variables for recording and platforms
local recording = false
local replaying = false
local player = game.Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local humanoid = character:WaitForChild("Humanoid") -- Get the Humanoid
local platforms = {}
local yellowPlatforms = {}
local platformData = {}
local platformCounter = 0
local lastPosition = nil
local replayThread

-- Pathfinding service
local PathfindingService = game:GetService("PathfindingService")

-- Function to calculate path
local function calculatePath(start, goal)
    local path = PathfindingService:CreatePath()
    path:ComputeAsync(start, goal)
    return path
end

-- Helper Functions
local function isCharacterMoving()
    local currentPosition = character.PrimaryPart.Position
    if lastPosition then
        local distance = (currentPosition - lastPosition).magnitude
        lastPosition = currentPosition
        return distance > 0.05
    end
    lastPosition = currentPosition
    return false
end

local function cleanupPlatform(platform)
    for i, p in ipairs(platforms) do
        if p == platform then
            table.remove(platforms, i)
            platformData[platform] = nil
            break
        end
    end
end

local function addPlatformToScrollFrame(platformName)
    local button = Instance.new("TextButton")
    button.Parent = scrollFrame
    button.Size = UDim2.new(1, -10, 0, 25)
    button.Text = platformName
    button.TextScaled = true
    button.BackgroundColor3 = Color3.fromRGB(150, 150, 255)

    local playButton = Instance.new("TextButton")
    playButton.Parent = button
    playButton.Size = UDim2.new(0, 50, 1, 0)
    playButton.Position = UDim2.new(1, -40, 0, 0)
    playButton.Text = "Play"
    playButton.TextScaled = true
    playButton.BackgroundColor3 = Color3.fromRGB(0, 255, 0)

    playButton.MouseButton1Click:Connect(function()
        if replaying then return end
        local platformIndex = tonumber(platformName:match("%d+"))
        local platform = platforms[platformIndex]

        -- Walk to the platform before replaying (Pathfinding Integrated)
        local function walkToPlatform(destination)
            local humanoid = character:WaitForChild("Humanoid")
            local rootPart = character:WaitForChild("HumanoidRootPart")
            local path = calculatePath(rootPart.Position, destination)

            if path.Status == Enum.PathStatus.Success then
                local waypoints = path:GetWaypoints()
                for i, waypoint in ipairs(waypoints) do
                    if not replaying then return end

                    humanoid:MoveTo(waypoint.Position)
                    if waypoint.Action == Enum.PathWaypointAction.Jump then
                        humanoid.Jump = true
                    end
                    humanoid.MoveToFinished:Wait()
                end
            else
                warn("Path not found!")
            end
        end

        -- Replay logic for sequential platforms (Improved)
        local function replayPlatforms(startIndex)
            for i = startIndex, #platforms do
                if not replaying then return end
                local platform = platforms[i]
                --Walk to platform using pathfinding
                walkToPlatform(platform.Position + Vector3.new(0, 3, 0))
                local movements = platformData[platform]
                if movements then
                    for j = 1, #movements - 1 do
                        if not replaying then return end

                        local startMovement = movements[j]
                        local endMovement = movements[j + 1]
                        endMovement.isJumping = startMovement.isJumping

                        local startTime = tick()

                        -- Calculate duration based on distance and a speed factor
                        local distance = (endMovement.position - startMovement.position).magnitude
                        local speedFactor = 0.01  -- Adjust this for desired replay speed (higher = faster)
                        local duration = distance * speedFactor

                        -- Minimum duration to prevent division by zero or extremely fast replays
                        duration = math.max(duration, 0.01)

                        local endTime = startTime + duration

                        while tick() < endTime do
                            if not replaying then return end
                            local alpha = (tick() - startTime) / duration
                            alpha = math.min(alpha, 1)

                            local interpolatedPosition = startMovement.position:Lerp(endMovement.position, alpha)
                            local startOrientation = startMovement.orientation
                            local endOrientation = endMovement.orientation
                            local interpolatedOrientation = CFrame.fromEulerAnglesYXZ(0, math.rad(startOrientation.Y), 0):Lerp(CFrame.fromEulerAnglesYXZ(0, math.rad(endOrientation.Y), 0), alpha)

                            character:SetPrimaryPartCFrame(CFrame.new(interpolatedPosition) * interpolatedOrientation)

                            if endMovement.isJumping then
                                humanoid.Jump = true
                            end

                            game:GetService("RunService").Heartbeat:Wait()
                        end
                    end
                end
                wait(0.5)
            end
            replaying = false
            statusLabel.Text = "Status: Idle"
            statusLabel.TextColor3 = Color3.fromRGB(0, 0, 0)
        end

        -- Update status and start replaying
        statusLabel.Text = "Status: Playing from " .. platformName
        statusLabel.TextColor3 = Color3.fromRGB(0, 0, 255)
        replaying = true
        spawn(function()
            replayPlatforms(platformIndex)
        end)
    end)
end

-- Handle character respawn or reset
player.CharacterAdded:Connect(function(newCharacter)
    character = newCharacter
    humanoid = newCharacter:WaitForChild("Humanoid") -- Get Humanoid for new character
    lastPosition = nil
    statusLabel.Text = "Status: Idle"
    statusLabel.TextColor3 = Color3.fromRGB(0, 0, 0)

    if recording then
        platformCounter = 0
        platforms = {}
        yellowPlatforms = {}
        platformData = {}
        scrollFrame:ClearAllChildren()
        scrollFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
    end
end)

-- Button Functions
recordButton.MouseButton1Click:Connect(function()
    if not recording then
        recording = true
        statusLabel.Text = "Status: Recording"
        statusLabel.TextColor3 = Color3.fromRGB(255, 0, 0)

        if #yellowPlatforms > 0 then
            local lastYellowPlatform = yellowPlatforms[#yellowPlatforms]
            character:SetPrimaryPartCFrame(CFrame.new(lastYellowPlatform.Position + Vector3.new(0, 3, 0)))
        end

        platformCounter += 1
        local platform = Instance.new("Part")
        platform.Name = "Platform " .. platformCounter
        platform.Size = Vector3.new(5, 1, 5)
        platform.Position = character.PrimaryPart.Position - Vector3.new(0, 3, 0)
        platform.Anchored = true
        platform.BrickColor = BrickColor.Red()
        platform.CanCollide = false
        platform.Parent = workspace

        table.insert(platforms, platform)
        platformData[platform] = {}

        addPlatformToScrollFrame("Platform " .. platformCounter)
        scrollFrame.CanvasSize = UDim2.new(0, 0, 0, #platforms * 30)

        -- Start recording movement, orientation, and jump state
        spawn(function()
            while recording do
                if isCharacterMoving() then
                    table.insert(platformData[platform], {
                        position = character.PrimaryPart.Position,
                        orientation = character.PrimaryPart.Orientation,
                        isJumping = humanoid.Jump -- Record jump state
                    })
                end
                game:GetService("RunService").Heartbeat:Wait() -- Use Heartbeat for recording
            end
        end)
    end
end)

stopRecordButton.MouseButton1Click:Connect(function()
    if recording then
        recording = false
        statusLabel.Text = "Status: Stopped Recording"
        statusLabel.TextColor3 = Color3.fromRGB(255, 165, 0)

        local yellowPlatform = Instance.new("Part")
        yellowPlatform.Size = Vector3.new(5, 1, 5)
        yellowPlatform.Position = character.PrimaryPart.Position - Vector3.new(0, 3, 0)
        yellowPlatform.Anchored = true
        yellowPlatform.BrickColor = BrickColor.Yellow()
        yellowPlatform.CanCollide = false
        yellowPlatform.Parent = workspace

        table.insert(yellowPlatforms, yellowPlatform)
    end
end)

stopReplayButton.MouseButton1Click:Connect(function()
    if replaying then
        replaying = false
        statusLabel.Text = "Status: Replay Stopped"
        statusLabel.TextColor3 = Color3.fromRGB(255, 0, 0)
    end
end)

destroyButton.MouseButton1Click:Connect(function()
    for _, platform in ipairs(platforms) do
        platform:Destroy()
    end
    for _, yellowPlatform in ipairs(yellowPlatforms) do
        yellowPlatform:Destroy()
    end
    platforms = {}
    yellowPlatforms = {}
    platformData = {}
    platformCounter = 0
    screenGui:Destroy()
end)

deleteButton.MouseButton1Click:Connect(function()
    if #platforms > 0 then
        local lastPlatform = platforms[#platforms]
        lastPlatform:Destroy()
        cleanupPlatform(lastPlatform)

        if #yellowPlatforms > 0 then
            local lastYellowPlatform = yellowPlatforms[#yellowPlatforms]
            lastYellowPlatform:Destroy()
            table.remove(yellowPlatforms, #yellowPlatforms)
        end

        if #scrollFrame:GetChildren() > 1 then
            local lastButton = scrollFrame:GetChildren()[#scrollFrame:GetChildren()]
            lastButton:Destroy()
        end

        platformCounter -= 1
        scrollFrame.CanvasSize = UDim2.new(0, 0, 0, #platforms * 30)
    end
end)
