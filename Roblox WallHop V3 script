-- Create ScreenGui
local screenGui = Instance.new("ScreenGui")
screenGui.Parent = game.Players.LocalPlayer:WaitForChild("PlayerGui")
screenGui.ResetOnSpawn = false

-- Create Frame (GUI 1)
local frame = Instance.new("Frame")
frame.Parent = screenGui
frame.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
frame.Size = UDim2.new(0, 200, 0, 100)
frame.Position = UDim2.new(0.5, -100, 0.5, -50)
frame.Active = true
frame.Draggable = true

-- Calculate button positions
local frameWidth = frame.Size.X.Offset
local buttonWidth = 60
local totalButtonWidth = buttonWidth * 3
local totalSpacing = frameWidth - totalButtonWidth
local spacing = totalSpacing / 4

local onButtonX = spacing
local additionButtonX = spacing + buttonWidth + spacing
local offButtonX = spacing + buttonWidth + spacing + buttonWidth + spacing

-- Create On Button
local onButton = Instance.new("TextButton")
onButton.Parent = frame
onButton.BackgroundColor3 = Color3.fromRGB(0, 255, 0)
onButton.Size = UDim2.new(0, buttonWidth, 0, 30)
onButton.Position = UDim2.new(0, onButtonX, 0, 20)
onButton.Text = "On"
onButton.TextScaled = true

-- Create Addition Button
local additionButton = Instance.new("TextButton")
additionButton.Parent = frame
additionButton.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
additionButton.Size = UDim2.new(0, buttonWidth, 0, 30)
additionButton.Position = UDim2.new(0, additionButtonX, 0, 20)
additionButton.Text = "Addition"
additionButton.TextScaled = true
local additionToggle = false
local additionalFrame = nil -- This will hold the Frame instance
local jumpButtonInAddition = nil -- To hold the jump button specific to the additional frame

-- Create Off Button
local offButton = Instance.new("TextButton")
offButton.Parent = frame
offButton.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
offButton.Size = UDim2.new(0, buttonWidth, 0, 30)
offButton.Position = UDim2.new(0, offButtonX, 0, 20)
offButton.Text = "Off"
offButton.TextScaled = true

-- Create Destroy Button
local destroyButton = Instance.new("TextButton")
destroyButton.Parent = frame
destroyButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
destroyButton.Size = UDim2.new(0, 160, 0, 30)
destroyButton.Position = UDim2.new(0, 20, 0, 60)
destroyButton.Text = "Destroy"
destroyButton.TextScaled = true

-- Create Status Indicator
local statusLabel = Instance.new("TextLabel")
statusLabel.Parent = frame
statusLabel.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
statusLabel.Size = UDim2.new(0, 200, 0, 30)
statusLabel.Position = UDim2.new(0, 0, 0, -30)
statusLabel.Text = "WallHop V3: Off" -- Changed V2 to V3
statusLabel.TextColor3 = Color3.fromRGB(255, 0, 0)
statusLabel.TextScaled = true

-- Variables for Wallhop Functionality
local wallhopToggle = false
local InfiniteJumpEnabled = true -- Debounce for main wallhop
local UserInputService = game:GetService("UserInputService")
local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local RunService = game:GetService("RunService")

local raycastParams = RaycastParams.new()
raycastParams.FilterType = Enum.RaycastFilterType.Blacklist

local jumpConnection = nil -- Connection for the main spacebar JumpRequest

-- ============================================================== --
-- Precise wall detection function - REVISED with More Rays + Blockcast --
-- ============================================================== --
local function getWallRaycastResult()
    local player = Players.LocalPlayer
    local character = player.Character
    if not character then return nil end
    local humanoidRootPart = character:FindFirstChild("HumanoidRootPart")
    if not humanoidRootPart then return nil end

    raycastParams.FilterDescendantsInstances = {character}
    local detectionDistance = 2
    local closestHit = nil
    local minDistance = detectionDistance + 1
    local hrpCF = humanoidRootPart.CFrame

    for i = 0, 7 do
        local angle = math.rad(i * 45)
        local direction = (hrpCF * CFrame.Angles(0, angle, 0)).LookVector
        local ray = Workspace:Raycast(humanoidRootPart.Position, direction * detectionDistance, raycastParams)
        if ray and ray.Instance and ray.Distance < minDistance then
            minDistance = ray.Distance
            closestHit = ray
        end
    end

    local blockCastSize = Vector3.new(1.5, 1, 0.5)
    local blockCastOffset = CFrame.new(0, -1, -0.5)
    local blockCastOriginCF = hrpCF * blockCastOffset
    local blockCastDirection = hrpCF.LookVector
    local blockCastDistance = 1.5
    local blockResult = Workspace:Blockcast(blockCastOriginCF, blockCastSize, blockCastDirection * blockCastDistance, raycastParams)

    if blockResult and blockResult.Instance and blockResult.Distance < minDistance then
         minDistance = blockResult.Distance
         closestHit = blockResult
    end

    return closestHit
end
-- ============================================================== --


-- ======================================================================== --
-- Function for the "Jump" button in GUI 2 - Uses revised getWallRaycastResult
-- Includes camera influence (corrected) and cosmetic flick + rotate back
-- MODIFIED: Allows separate Left/Right max influence angles (Instant Rotation)
-- ======================================================================== --
local function performFaceWallJump()
    local player = Players.LocalPlayer
    local character = player.Character
    local humanoid = character and character:FindFirstChildOfClass("Humanoid")
    local rootPart = character and character:FindFirstChild("HumanoidRootPart")
    local camera = Workspace.CurrentCamera

    if not (humanoid and rootPart and camera and humanoid:GetState() ~= Enum.HumanoidStateType.Dead) then return end

    local wallRayResult = getWallRaycastResult()

    if wallRayResult then
        -- === CONFIGURATION FOR JUMP BUTTON ROTATION ===
        local maxInfluenceAngleRight = math.rad(20) -- Max angle if camera is to the RIGHT
        local maxInfluenceAngleLeft  = math.rad(-100) -- Max angle if camera is to the LEFT
        -- ============================================

        local wallNormal = wallRayResult.Normal
        local baseDirectionAwayFromWall = Vector3.new(wallNormal.X, 0, wallNormal.Z).Unit
        if baseDirectionAwayFromWall.Magnitude < 0.1 then
             local dirToHit = (wallRayResult.Position - rootPart.Position) * Vector3.new(1,0,1)
             baseDirectionAwayFromWall = -dirToHit.Unit
             if baseDirectionAwayFromWall.Magnitude < 0.1 then
                 baseDirectionAwayFromWall = -rootPart.CFrame.LookVector * Vector3.new(1, 0, 1)
                 if baseDirectionAwayFromWall.Magnitude > 0.1 then baseDirectionAwayFromWall = baseDirectionAwayFromWall.Unit end
                 if baseDirectionAwayFromWall.Magnitude < 0.1 then baseDirectionAwayFromWall = Vector3.new(0,0,1) end
             end
        end
        baseDirectionAwayFromWall = Vector3.new(baseDirectionAwayFromWall.X, 0, baseDirectionAwayFromWall.Z).Unit
        if baseDirectionAwayFromWall.Magnitude < 0.1 then baseDirectionAwayFromWall = Vector3.new(0,0,1) end

        local cameraLook = camera.CFrame.LookVector
        local horizontalCameraLook = Vector3.new(cameraLook.X, 0, cameraLook.Z).Unit
        if horizontalCameraLook.Magnitude < 0.1 then horizontalCameraLook = baseDirectionAwayFromWall end

        -- Calculate angle and direction between base and camera
        local dot = math.clamp(baseDirectionAwayFromWall:Dot(horizontalCameraLook), -1, 1)
        local angleBetween = math.acos(dot)
        local cross = baseDirectionAwayFromWall:Cross(horizontalCameraLook)
        local rotationSign = -math.sign(cross.Y)
        if rotationSign == 0 then angleBetween = 0 end

        -- Apply the correct maximum influence angle based on direction
        local actualInfluenceAngle
        if rotationSign == 1 then -- Camera influencing RIGHT
            actualInfluenceAngle = math.min(angleBetween, maxInfluenceAngleRight)
        elseif rotationSign == -1 then -- Camera influencing LEFT
            actualInfluenceAngle = math.min(angleBetween, maxInfluenceAngleLeft)
        else -- Aligned
            actualInfluenceAngle = 0
        end

        -- Calculate the final target direction using the clamped angle and the sign
        local adjustmentRotation = CFrame.Angles(0, actualInfluenceAngle * rotationSign, 0)
        local initialTargetLookDirection = adjustmentRotation * baseDirectionAwayFromWall

        -- Apply Instant Rotation
        rootPart.CFrame = CFrame.lookAt(rootPart.Position, rootPart.Position + initialTargetLookDirection)
        RunService.Heartbeat:Wait() -- Wait a frame for rotation to process before jump

        -- Jump, Flick, and Rotate Back Logic
        local didJump = false
        if humanoid and humanoid:GetState() ~= Enum.HumanoidStateType.Dead then
            humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
            didJump = true
            print("Performed jump away from wall (Button).")

            -- Cosmetic Flick
            rootPart.CFrame = rootPart.CFrame * CFrame.Angles(0, -1, 0)
            task.wait(0.15)
            rootPart.CFrame = rootPart.CFrame * CFrame.Angles(0, 1, 0)
            print("Applied cosmetic rotation flick (-1/+1 radians) (Button).")
        end

        -- Rotate back AFTER flick if jump happened
        if didJump then
            local directionTowardsWall = -baseDirectionAwayFromWall
            task.wait(0.05) -- Wait for flick to visually finish
            rootPart.CFrame = CFrame.lookAt(rootPart.Position, rootPart.Position + directionTowardsWall)
            print("Rotated back towards wall after jump (Button).")
        end
    else
        print("No nearby wall found for wall jump (Button).")
    end
end
-- ======================================================================== --


-- Button Functions
onButton.MouseButton1Click:Connect(function()
    statusLabel.Text = "WallHop V3: On" 
    statusLabel.TextColor3 = Color3.fromRGB(0, 255, 0)
    wallhopToggle = true
end)

offButton.MouseButton1Click:Connect(function()
    statusLabel.Text = "WallHop V3: Off" 
    statusLabel.TextColor3 = Color3.fromRGB(255, 0, 0)
    wallhopToggle = false
end)

-- Addition Button Functionality (Connects to performFaceWallJump)
additionButton.MouseButton1Click:Connect(function()
    additionToggle = not additionToggle
    if additionToggle then
        if not additionalFrame then
            additionButton.BackgroundColor3 = Color3.fromRGB(0, 255, 0)
            additionalFrame = Instance.new("Frame")
            additionalFrame.Name = "AdditionalWallHopFrame"
            additionalFrame.Parent = frame
            additionalFrame.BackgroundColor3 = Color3.fromRGB(200, 200, 200)
            additionalFrame.Size = UDim2.new(1, 0, 1, 0)
            additionalFrame.Position = UDim2.new(0, 0, 1, 0)
            additionalFrame.Active = false; additionalFrame.Draggable = false; additionalFrame.BorderSizePixel = 1
            jumpButtonInAddition = Instance.new("TextButton")
            jumpButtonInAddition.Name = "FaceWallJumpButton"; jumpButtonInAddition.Parent = additionalFrame
            jumpButtonInAddition.BackgroundColor3 = Color3.fromRGB(0, 150, 255)
            jumpButtonInAddition.Size = UDim2.new(0.8, 0, 0.4, 0); jumpButtonInAddition.Position = UDim2.new(0.1, 0, 0.3, 0)
            jumpButtonInAddition.Text = "Jump"; jumpButtonInAddition.TextColor3 = Color3.fromRGB(255,255,255)
            jumpButtonInAddition.TextScaled = true; jumpButtonInAddition.Active = true
            jumpButtonInAddition.MouseButton1Click:Connect(performFaceWallJump) -- Connects to the updated function
        end
    else
        if additionalFrame then
            additionButton.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
            additionalFrame:Destroy(); additionalFrame = nil; jumpButtonInAddition = nil
        end
    end
end)

destroyButton.MouseButton1Click:Connect(function()
    if jumpConnection then jumpConnection:Disconnect(); jumpConnection = nil end
    wallhopToggle = false; screenGui:Destroy(); additionalFrame = nil; jumpButtonInAddition = nil
    print("GUI Destroyed.")
end)

-- ======================================================================== --
-- Main Wallhop Function (Activated by Spacebar via JumpRequest)
-- MODIFIED: Uses the SAME L/R rotation logic & limits as the Jump Button.
-- ======================================================================== --
jumpConnection = UserInputService.JumpRequest:Connect(function()
    if not wallhopToggle or not InfiniteJumpEnabled then return end

    local player = Players.LocalPlayer
    local character = player.Character
    local humanoid = character and character:FindFirstChildOfClass("Humanoid")
    local rootPart = character and character:FindFirstChild("HumanoidRootPart")
    local camera = Workspace.CurrentCamera

    if not (humanoid and rootPart and camera and humanoid:GetState() ~= Enum.HumanoidStateType.Dead) then return end

    local wallRayResult = getWallRaycastResult()

    if wallRayResult then
        InfiniteJumpEnabled = false -- Start debounce

        -- === CONFIGURATION FOR MAIN JUMP ROTATION (Mirrors Jump Button) ===
        local maxInfluenceAngleRight = math.rad(20) -- Max angle if camera is to the RIGHT
        local maxInfluenceAngleLeft  = math.rad(-100) -- Max angle if camera is to the LEFT
        -- ==================================================================

        -- 1. Calculate Base Direction Away from Wall
        local wallNormal = wallRayResult.Normal
        local baseDirectionAwayFromWall = Vector3.new(wallNormal.X, 0, wallNormal.Z).Unit
        if baseDirectionAwayFromWall.Magnitude < 0.1 then
             local dirToHit = (wallRayResult.Position - rootPart.Position) * Vector3.new(1,0,1)
             baseDirectionAwayFromWall = -dirToHit.Unit
             if baseDirectionAwayFromWall.Magnitude < 0.1 then
                 baseDirectionAwayFromWall = -rootPart.CFrame.LookVector * Vector3.new(1, 0, 1)
                 if baseDirectionAwayFromWall.Magnitude > 0.1 then baseDirectionAwayFromWall = baseDirectionAwayFromWall.Unit end
                 if baseDirectionAwayFromWall.Magnitude < 0.1 then baseDirectionAwayFromWall = Vector3.new(0,0,1) end
             end
        end
        baseDirectionAwayFromWall = Vector3.new(baseDirectionAwayFromWall.X, 0, baseDirectionAwayFromWall.Z).Unit
        if baseDirectionAwayFromWall.Magnitude < 0.1 then baseDirectionAwayFromWall = Vector3.new(0,0,1) end

        -- 2. Get Camera's Horizontal Look Direction
        local cameraLook = camera.CFrame.LookVector
        local horizontalCameraLook = Vector3.new(cameraLook.X, 0, cameraLook.Z).Unit
        if horizontalCameraLook.Magnitude < 0.1 then horizontalCameraLook = baseDirectionAwayFromWall end

        -- 3. Calculate Initial Rotation Target Direction (Away from wall, camera influenced)
        local dot = math.clamp(baseDirectionAwayFromWall:Dot(horizontalCameraLook), -1, 1)
        local angleBetween = math.acos(dot)
        local cross = baseDirectionAwayFromWall:Cross(horizontalCameraLook)
        local rotationSign = -math.sign(cross.Y)
        if rotationSign == 0 then angleBetween = 0 end

        -- Apply the correct maximum influence angle based on direction (Mirrors Jump Button)
        local actualInfluenceAngle
        if rotationSign == 1 then -- Camera influencing RIGHT
            actualInfluenceAngle = math.min(angleBetween, maxInfluenceAngleRight)
        elseif rotationSign == -1 then -- Camera influencing LEFT
            actualInfluenceAngle = math.min(angleBetween, maxInfluenceAngleLeft)
        else -- Aligned
            actualInfluenceAngle = 0
        end

        -- Calculate the final target direction
        local adjustmentRotation = CFrame.Angles(0, actualInfluenceAngle * rotationSign, 0)
        local initialTargetLookDirection = adjustmentRotation * baseDirectionAwayFromWall

        -- 4. Apply Initial Rotation AWAY FIRST (Instant LookAt)
        rootPart.CFrame = CFrame.lookAt(rootPart.Position, rootPart.Position + initialTargetLookDirection)

        -- 5. Wait a frame for rotation to process before jump
        RunService.Heartbeat:Wait()

        -- 6. Apply Jump & Cosmetic Flick
        local didJump = false
        if humanoid and humanoid:GetState() ~= Enum.HumanoidStateType.Dead then
             humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
             didJump = true
             print("Performed jump away from wall (Main).")

             -- === Apply Cosmetic Rotation Flick ===
             rootPart.CFrame = rootPart.CFrame * CFrame.Angles(0, -1, 0)
             task.wait(0.15)
             rootPart.CFrame = rootPart.CFrame * CFrame.Angles(0, 1, 0)
             print("Applied cosmetic rotation flick (-1/+1 radians) (Main).")
             -- ====================================
        end

        -- 7. Apply Rotation BACK towards the wall IF the jump occurred
        if didJump then
             local directionTowardsWall = -baseDirectionAwayFromWall
             task.wait(0.05) -- Wait for flick to visually finish
             rootPart.CFrame = CFrame.lookAt(rootPart.Position, rootPart.Position + directionTowardsWall)
             print("Rotated back towards wall after jump (Main).")
        end

        -- Post-jump cooldown
        InfiniteJumpEnabled = true -- End debounce
    end
end)
-- ======================================================================== --
