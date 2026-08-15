-- Services
local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")

local player = Players.LocalPlayer or Players.PlayerAdded:Wait()

-- Determine GUI parent
local guiParent
if type(gethui) == "function" then
    guiParent = gethui()
else
    local success = pcall(function()
        return game:GetService("CoreGui").Name
    end)
    if success then
        guiParent = game:GetService("CoreGui")
    else
        guiParent = player:WaitForChild("PlayerGui")
    end
end

-- Create ScreenGui
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "Gui_" .. math.random(10000, 99999)
screenGui.ResetOnSpawn = false
screenGui.DisplayOrder = 2147483647
screenGui.IgnoreGuiInset = true
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Global
screenGui.Parent = guiParent

-- Main draggable frame (Fixed at 200x100)
local mainFrame = Instance.new("Frame")
mainFrame.Parent = screenGui
mainFrame.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
mainFrame.Size = UDim2.new(0, 200, 0, 100)
mainFrame.Position = UDim2.new(0.3, 0, 0.5, -50)
mainFrame.Draggable = true
mainFrame.Active = true

-- Status label
local status = Instance.new("TextLabel")
status.Parent = mainFrame
status.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
status.Size = UDim2.new(0, 200, 0, 30)
status.Position = UDim2.new(0, 0, 0, -30)
status.Text = "Status: Offline"
status.TextColor3 = Color3.fromRGB(255, 0, 0)
status.TextScaled = true

-- Helper to create buttons
local function createButton(name, text, x, y, width, height, textColor)
    local btn = Instance.new("TextButton")
    btn.Name = name
    btn.Parent = mainFrame
    btn.Text = text
    btn.Size = UDim2.new(0, width, 0, height)
    btn.Position = UDim2.new(0, x, 0, y)
    btn.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    btn.TextColor3 = textColor or Color3.fromRGB(255, 255, 255)
    btn.TextScaled = true
    btn.AutoButtonColor = true
    btn.BorderSizePixel = 0
    return btn
end

-- 5-Button Layout inside 200x100
local toggleBtn    = createButton("ToggleBtn", "START", 8, 8, 88, 26, Color3.fromRGB(0, 255, 0))
local fastBtn      = createButton("FastBtn", "AUTO TOUCH: OFF", 104, 8, 88, 26, Color3.fromRGB(0, 255, 255))
local antiLagBtn   = createButton("AntiLagBtn", "ANTI-LAG: OFF", 8, 38, 88, 26, Color3.fromRGB(255, 255, 255))
local billboardBtn = createButton("BillboardBtn", "BILLBOARD: OFF", 104, 38, 88, 26, Color3.fromRGB(255, 255, 0))
local destroyBtn   = createButton("DestroyBtn", "Destroy UI", 8, 68, 184, 24, Color3.fromRGB(255, 100, 100))

-------------------------------------------------------------------------------
-- CORE VARIABLES & CONFIG
-------------------------------------------------------------------------------

local BATCH_SIZE = 30 -- High-speed parallel batch size

local isScanning = false
local isAutoTouching = false
local isAntiLagOn = false
local isBillboardOn = false

local cachedParts = {}      
local neighborMap = {}      
local numberTiles = {}      
local knownMines = {}       
local knownSafe = {}
local probabilityMap = {}
local currentBestGuess = nil 
local connections = {}
local pendingReset = false 
local trackedBillboards = {}
local activeFlagPositions = {} -- Lightweight Vector3 cache

-------------------------------------------------------------------------------
-- HELPER & PROXIMITY FUNCTIONS
-------------------------------------------------------------------------------

local function getCharPos()
    local char = player.Character
    local root = char and (char:FindFirstChild("HumanoidRootPart") or char:FindFirstChild("Head"))
    return root and root.Position or nil
end

local function sortByProximity(list, charPos)
    if not charPos or #list <= 1 then return end
    table.sort(list, function(a, b)
        return (a.Position - charPos).Magnitude < (b.Position - charPos).Magnitude
    end)
end

local function isColorMatch(c1, r, g, b)
    return math.abs(c1.R - r) < 0.05 and math.abs(c1.G - g) < 0.05 and math.abs(c1.B - b) < 0.05
end

local function isExcluded(part)
    if not part:IsA("BasePart") then return true end
    return isColorMatch(part.Color, 1.000, 1.000, 0.490)
end

local function isUnrevealed(part)
    if not part or not part:IsA("BasePart") or isExcluded(part) or numberTiles[part] then return false end
    
    local green1 = isColorMatch(part.Color, 0.459, 0.804, 0.392)
    local green2 = isColorMatch(part.Color, 0.404, 0.706, 0.345)
    local red1   = isColorMatch(part.Color, 0.804, 0.392, 0.392) 
    local red2   = isColorMatch(part.Color, 0.706, 0.345, 0.345) 
    local blue1  = isColorMatch(part.Color, 0.392, 0.741, 0.804)
    local blue2  = isColorMatch(part.Color, 0.345, 0.651, 0.706)
    
    return green1 or green2 or red1 or red2 or blue1 or blue2
end

local function getNumber(part)
    if not part or not part:IsA("BasePart") or isExcluded(part) then return nil end
    for _, desc in ipairs(part:GetDescendants()) do
        if desc:IsA("TextLabel") and desc.Visible then
            local num = tonumber(desc.Text:match("%d+"))
            if num and num >= 1 and num <= 8 then return num end
        end
    end
    return nil
end

local function isTileCandidate(part)
    if not part or not part:IsA("BasePart") then return false end
    if isExcluded(part) then return false end
    if numberTiles[part] or isUnrevealed(part) or getNumber(part) ~= nil then return true end
    if part.Name == "Part" and part.Parent and string.find(string.lower(part.Parent.Name), "part") then
        return true
    end
    return false
end

-------------------------------------------------------------------------------
-- ULTRA-FAST SPATIAL FLAG CACHE
-------------------------------------------------------------------------------

local function refreshFlagCache()
    table.clear(activeFlagPositions)
    
    local flagFolder = Workspace:FindFirstChild("Flag") or Workspace:FindFirstChild("Flags")
    if flagFolder then
        for _, desc in ipairs(flagFolder:GetDescendants()) do
            if desc:IsA("BasePart") then
                table.insert(activeFlagPositions, desc.Position)
            end
        end
    end
    
    for _, obj in ipairs(Workspace:GetChildren()) do
        if obj:IsA("Model") and string.find(string.lower(obj.Name), "flag") then
            local primary = obj.PrimaryPart or obj:FindFirstChildWhichIsA("BasePart")
            if primary then
                table.insert(activeFlagPositions, primary.Position)
            end
        end
    end
end

local function hasFlag(part)
    if not part or not part:IsDescendantOf(workspace) then return false end
    local pPos = part.Position
    local pSizeX = part.Size.X * 0.55
    local pSizeZ = part.Size.Z * 0.55

    for _, fPos in ipairs(activeFlagPositions) do
        local dx = math.abs(fPos.X - pPos.X)
        local dz = math.abs(fPos.Z - pPos.Z)
        local dy = fPos.Y - pPos.Y
        if dx < pSizeX and dz < pSizeZ and dy > 0.3 and dy < 6 then
            return true
        end
    end
    return false
end

local function clickTile(part)
    if not part or not part:IsDescendantOf(workspace) then return end
    local cd = part:FindFirstChildOfClass("ClickDetector") or part:FindFirstChildWhichIsA("ClickDetector", true)
    if cd and typeof(fireclickdetector) == "function" then
        fireclickdetector(cd, 0)
    end
end

-------------------------------------------------------------------------------
-- TOUCH TRIGGER (NO LEG OFFSET)
-------------------------------------------------------------------------------

local function triggerTileTouch(targetTile)
    if not targetTile or not targetTile:IsDescendantOf(workspace) then return end
    
    local char = player.Character
    if not char then return end
    
    local limb = char:FindFirstChild("RightFoot") 
        or char:FindFirstChild("Right Leg") 
        or char:FindFirstChild("LeftFoot") 
        or char:FindFirstChild("Left Leg")
        or char:FindFirstChild("HumanoidRootPart")
        
    if not limb then return end

    if typeof(firetouchinterest) == "function" then
        firetouchinterest(limb, targetTile, 0)
        task.wait(0.015)
        firetouchinterest(limb, targetTile, 1)
    else
        local probe = Instance.new("Part")
        probe.Name = "LegProbe"
        probe.Size = limb.Size
        probe.CanCollide = false
        probe.Transparency = 1
        probe.CFrame = targetTile.CFrame
        probe.Parent = char
        task.wait(0.02)
        probe:Destroy()
    end
end

-------------------------------------------------------------------------------
-- HIGHLIGHT VISUALS
-------------------------------------------------------------------------------

local function clearVisuals(part)
    if not part then return end
    if part:FindFirstChild("Mine_Highlight") then part.Mine_Highlight:Destroy() end
    if part:FindFirstChild("Safe_Highlight") then part.Safe_Highlight:Destroy() end
    if part:FindFirstChild("Guess_Highlight") then part.Guess_Highlight:Destroy() end
end

local function createHighlight(part, color, name)
    if part:FindFirstChild(name) then return end
    
    if name == "Guess_Highlight" then
        if part:FindFirstChild("Safe_Highlight") or part:FindFirstChild("Mine_Highlight") then return end
    elseif name == "Safe_Highlight" or name == "Mine_Highlight" then
        if part:FindFirstChild("Guess_Highlight") then part.Guess_Highlight:Destroy() end
    end

    local h = Instance.new("Highlight")
    h.Name = name
    h.Adornee = part
    h.FillColor = color
    h.FillTransparency = 0.4
    h.OutlineTransparency = 0.5
    h.DepthMode = Enum.HighlightDepthMode.Occluded 
    h.Parent = part
end

-------------------------------------------------------------------------------
-- BILLBOARD INSPECTOR (OPTIMIZED)
-------------------------------------------------------------------------------

local function removeBillboard(part)
    if trackedBillboards[part] then
        trackedBillboards[part]:Destroy()
        trackedBillboards[part] = nil
    end
end

local function removeAllBillboards()
    for part, bb in pairs(trackedBillboards) do
        if bb then bb:Destroy() end
    end
    table.clear(trackedBillboards)
end

local function updateBillboard(part)
    if not isBillboardOn or not part or not part:IsDescendantOf(workspace) then
        removeBillboard(part)
        return
    end

    local bb = trackedBillboards[part]
    if not bb or not bb.Parent then
        bb = Instance.new("BillboardGui")
        bb.Name = "TileDebug_" .. part.Name
        bb.Size = UDim2.new(7.5, 0, 4.8, 0)
        bb.StudsOffset = Vector3.new(0, (part.Size.Y / 2) + 3.0, 0)
        bb.AlwaysOnTop = true
        bb.Adornee = part
        bb.Parent = guiParent

        local bg = Instance.new("Frame")
        bg.Name = "BG"
        bg.Size = UDim2.new(1, 0, 1, 0)
        bg.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
        bg.BackgroundTransparency = 0.15
        bg.BorderSizePixel = 2
        bg.Parent = bb

        local lbl = Instance.new("TextLabel")
        lbl.Name = "InfoLabel"
        lbl.Size = UDim2.new(1, -8, 1, -8)
        lbl.Position = UDim2.new(0, 4, 0, 4)
        lbl.BackgroundTransparency = 1
        lbl.TextColor3 = Color3.fromRGB(255, 255, 255)
        lbl.TextScaled = true
        lbl.Font = Enum.Font.SourceSansBold
        lbl.TextXAlignment = Enum.TextXAlignment.Left
        lbl.TextYAlignment = Enum.TextYAlignment.Top
        lbl.Parent = bg

        trackedBillboards[part] = bb
    end

    local num = numberTiles[part] or getNumber(part)
    local unrev = isUnrevealed(part)
    local isFlagged = hasFlag(part)
    local c = part.Color
    local hexColor = string.format("#%02X%02X%02X", math.floor(c.R * 255), math.floor(c.G * 255), math.floor(c.B * 255))
    
    local tag = "🔍 [CANDIDATE BLOCK]"
    local borderColor = Color3.fromRGB(255, 170, 0)

    if knownSafe[part] then
        if isFlagged then
            tag = "⚠️ [SAFE TILE - HAS FLAG] (Will Unflag)"
            borderColor = Color3.fromRGB(255, 150, 0)
        else
            tag = "✅ [CONFIRMED SAFE (100%)]"
            borderColor = Color3.fromRGB(0, 255, 0)
        end
    elseif knownMines[part] then
        if isFlagged then
            tag = "🚩 [MINE - FLAGGED ✅] (Protected)"
            borderColor = Color3.fromRGB(0, 200, 255)
        else
            tag = "💣 [MINE - UNFLAGGED] (Will Flag)"
            borderColor = Color3.fromRGB(255, 0, 0)
        end
    elseif currentBestGuess == part then
        tag = string.format("⚡ [BEST GUESS] (Prob: %.2f)", probabilityMap[part] or 0)
        borderColor = Color3.fromRGB(255, 255, 0)
    elseif num then
        tag = string.format("🔢 [REVEALED NUMBER: %d]", num)
        borderColor = Color3.fromRGB(0, 200, 255)
    elseif unrev then
        local prob = probabilityMap[part]
        tag = prob and string.format("🟩 [UNREVEALED TILE] (Prob: %.2f)", prob) or "🟩 [UNREVEALED TILE]"
    end

    local neighborCount = neighborMap[part] and #neighborMap[part] or 0
    local bg = bb:FindFirstChild("BG")
    local lbl = bg and bg:FindFirstChild("InfoLabel")
    if bg and lbl then
        bg.BorderColor3 = borderColor
        lbl.Text = string.format(
            "%s\n• Name: %s | Flagged: %s\n• Class: %s | Parent: %s\n• Path: %s\n• Color: RGB(%.2f, %.2f, %.2f) | Hex: %s | Mat: %s\n• Size: (%.1f, %.1f, %.1f) | Unrevealed: %s | Number: %s | Neighbors: %d",
            tag,
            part.Name, tostring(isFlagged),
            part.ClassName,
            part.Parent and part.Parent.Name or "None",
            part:GetFullName(),
            c.R, c.G, c.B, hexColor, tostring(part.Material.Name),
            part.Size.X, part.Size.Y, part.Size.Z, tostring(unrev), tostring(num or "N/A"), neighborCount
        )
    end
end

-------------------------------------------------------------------------------
-- NEIGHBOR CACHING & RESET
-------------------------------------------------------------------------------

local function cacheNeighbors(centerPart, forceRefresh)
    if not forceRefresh and neighborMap[centerPart] then return end
    
    local neighbors = {}
    local boxSize = centerPart.Size * 2.5 
    local partsInBox = workspace:GetPartBoundsInBox(centerPart.CFrame, boxSize)
    
    for _, p in ipairs(partsInBox) do
        if p ~= centerPart and isTileCandidate(p) then
            if math.abs(p.Position.Y - centerPart.Position.Y) < 1.5 then
                local dist = (p.Position - centerPart.Position).Magnitude
                if dist < (centerPart.Size.X * 1.85) then 
                    table.insert(neighbors, p)
                end
            end
        end
    end
    neighborMap[centerPart] = neighbors
end

local function requestReset()
    if pendingReset then return end
    pendingReset = true
    
    task.delay(0.2, function()
        pendingReset = false
        if not isScanning then return end
        
        for p, _ in pairs(knownSafe) do clearVisuals(p) end
        for p, _ in pairs(knownMines) do clearVisuals(p) end
        if currentBestGuess then clearVisuals(currentBestGuess) end
        
        knownSafe = {}
        knownMines = {}
        currentBestGuess = nil
    end)
end

-------------------------------------------------------------------------------
-- SOLVER LOGIC
-------------------------------------------------------------------------------

local function solveBoard()
    if pendingReset then return end 
    
    local activeNumbers = {} 
    probabilityMap = {} 
    
    -- PASS 1: Basic Logic
    for centerPart, targetNum in pairs(numberTiles) do
        if not centerPart:IsDescendantOf(workspace) then 
            numberTiles[centerPart] = nil
            continue 
        end

        local neighbors = neighborMap[centerPart]
        if not neighbors then continue end 
        
        local unrevealed = {}
        local mineCount = 0
        
        for _, n in ipairs(neighbors) do
            if knownMines[n] then 
                mineCount = mineCount + 1
            elseif isUnrevealed(n) and not knownSafe[n] then 
                table.insert(unrevealed, n) 
            end
        end
        
        local remainingMines = targetNum - mineCount
        
        if remainingMines == #unrevealed and #unrevealed > 0 then
            for _, p in ipairs(unrevealed) do 
                if not knownMines[p] then 
                    knownMines[p] = true
                    clearVisuals(p)
                    createHighlight(p, Color3.new(1,0,0), "Mine_Highlight") 
                end 
            end
        elseif remainingMines == 0 and #unrevealed > 0 then
            for _, p in ipairs(unrevealed) do 
                if not knownSafe[p] then 
                    knownSafe[p] = true
                    clearVisuals(p)
                    createHighlight(p, Color3.new(0,1,0), "Safe_Highlight") 
                end 
            end
        else
            if #unrevealed > 0 then
                local prob = remainingMines / #unrevealed
                for _, p in ipairs(unrevealed) do
                    if not probabilityMap[p] or prob > probabilityMap[p] then
                        probabilityMap[p] = prob
                    end
                end
            end
        end
        
        if #unrevealed > 0 then
            table.insert(activeNumbers, {
                part = centerPart,
                unrevealed = unrevealed,
                needed = remainingMines
            })
        end
    end
    
    -- PASS 2: Advanced Set Difference
    local numActive = #activeNumbers
    for i = 1, numActive do
        local A = activeNumbers[i]
        for j = 1, numActive do
            if i == j then continue end
            local B = activeNumbers[j]
            if (A.part.Position - B.part.Position).Magnitude > (A.part.Size.X * 3) then continue end
            
            local isSubset = true
            for _, uA in ipairs(A.unrevealed) do
                local found = false
                for _, uB in ipairs(B.unrevealed) do
                    if uA == uB then found = true; break end
                end
                if not found then isSubset = false; break end
            end
            
            if isSubset then
                local diffMines = B.needed - A.needed
                local diffSet = {}
                for _, uB in ipairs(B.unrevealed) do
                    local inA = false
                    for _, uA in ipairs(A.unrevealed) do
                        if uB == uA then inA = true; break end
                    end
                    if not inA then table.insert(diffSet, uB) end
                end
                
                if #diffSet > 0 then
                    if diffMines == 0 then
                        for _, p in ipairs(diffSet) do
                            if not knownSafe[p] then
                                knownSafe[p] = true
                                clearVisuals(p)
                                createHighlight(p, Color3.new(0,1,0), "Safe_Highlight")
                            end
                        end
                    elseif diffMines == #diffSet then
                        for _, p in ipairs(diffSet) do
                            if not knownMines[p] then
                                knownMines[p] = true
                                clearVisuals(p)
                                createHighlight(p, Color3.new(1,0,0), "Mine_Highlight")
                            end
                        end
                    end
                end
            end
        end
    end

    -- PASS 3: Best Guess (Yellow)
    local hasSafeMoves = false
    for _, _ in pairs(knownSafe) do hasSafeMoves = true; break end
    
    if currentBestGuess and (not currentBestGuess:IsDescendantOf(workspace) or knownSafe[currentBestGuess] or knownMines[currentBestGuess] or hasSafeMoves) then
        if currentBestGuess:IsDescendantOf(workspace) and currentBestGuess:FindFirstChild("Guess_Highlight") then
            currentBestGuess.Guess_Highlight:Destroy()
        end
        currentBestGuess = nil
    end

    if not hasSafeMoves then
        local bestPart = nil
        local lowestProb = 1.1 
        
        for part, prob in pairs(probabilityMap) do
            if not knownMines[part] and not knownSafe[part] then
                if prob < lowestProb then
                    lowestProb = prob
                    bestPart = part
                end
            end
        end
        
        if bestPart and bestPart ~= currentBestGuess then
            if currentBestGuess and currentBestGuess:IsDescendantOf(workspace) and currentBestGuess:FindFirstChild("Guess_Highlight") then
                currentBestGuess.Guess_Highlight:Destroy()
            end
            currentBestGuess = bestPart
            createHighlight(bestPart, Color3.fromRGB(255, 255, 0), "Guess_Highlight") 
        end
    end
end

-------------------------------------------------------------------------------
-- MAIN SOLVER LOOP
-------------------------------------------------------------------------------

local function startSolverLoop()
    task.spawn(function()
        while isScanning do
            local validParts = {}
            local charPos = getCharPos()
            
            for _, part in ipairs(cachedParts) do
                if part and part:IsDescendantOf(workspace) then
                    table.insert(validParts, part)
                    
                    -- Anti-Lag Distance Filter
                    if isAntiLagOn and charPos then
                        local dist = (part.Position - charPos).Magnitude
                        if dist > 60 then
                            clearVisuals(part)
                            removeBillboard(part)
                            numberTiles[part] = nil
                            knownSafe[part] = nil
                            knownMines[part] = nil
                            continue 
                        end
                    end
                    
                    if not neighborMap[part] then cacheNeighbors(part, false) end
                    
                    if isExcluded(part) then
                        if numberTiles[part] or knownSafe[part] or knownMines[part] then
                            numberTiles[part] = nil; knownSafe[part] = nil; knownMines[part] = nil
                            clearVisuals(part)
                        end
                    elseif not isUnrevealed(part) and not numberTiles[part] then
                        local num = getNumber(part)
                        if num then
                            numberTiles[part] = num
                            knownSafe[part] = nil; knownMines[part] = nil
                            clearVisuals(part)
                        end
                    end

                    if isBillboardOn then
                        updateBillboard(part)
                    end
                else
                    removeBillboard(part)
                    neighborMap[part] = nil
                    numberTiles[part] = nil
                    knownMines[part] = nil
                    knownSafe[part] = nil
                end
            end
            cachedParts = validParts
            
            solveBoard()
            task.wait(0.12) -- Optimized loop yield
        end
    end)
end

local function onDescendantAdded(descendant)
    if isScanning and isTileCandidate(descendant) then
        table.insert(cachedParts, descendant)
        task.delay(0.1, function() 
            if descendant:IsDescendantOf(workspace) then 
                cacheNeighbors(descendant, true) 
                for _, neighbor in ipairs(neighborMap[descendant] or {}) do
                    cacheNeighbors(neighbor, true)
                end
                requestReset()
            end 
        end)
    end
end

-------------------------------------------------------------------------------
-- UI CONNECTIONS
-------------------------------------------------------------------------------

toggleBtn.MouseButton1Click:Connect(function()
    if isScanning then
        isScanning = false
        isAutoTouching = false
        pendingReset = false
        
        toggleBtn.Text = "START"
        toggleBtn.TextColor3 = Color3.fromRGB(0, 255, 0)
        fastBtn.Text = "AUTO TOUCH: OFF"
        fastBtn.TextColor3 = Color3.fromRGB(0, 255, 255)
        status.Text = "Status: Offline"
        status.TextColor3 = Color3.fromRGB(255, 0, 0)
        
        for _, conn in pairs(connections) do conn:Disconnect() end
        connections = {}
        for _, p in pairs(cachedParts) do if p:IsDescendantOf(workspace) then clearVisuals(p) end end
        removeAllBillboards()
        cachedParts = {}; neighborMap = {}; numberTiles = {}; knownMines = {}; knownSafe = {}; currentBestGuess = nil
    else
        isScanning = true
        toggleBtn.Text = "STOP"
        toggleBtn.TextColor3 = Color3.fromRGB(255, 0, 0)
        status.Text = "Status: Scanning..."
        status.TextColor3 = Color3.fromRGB(0, 255, 0)
        
        cachedParts = {}; neighborMap = {}
        
        for _, obj in ipairs(workspace:GetDescendants()) do
            if isTileCandidate(obj) then 
                table.insert(cachedParts, obj)
                cacheNeighbors(obj, false)
            end
        end
        table.insert(connections, workspace.DescendantAdded:Connect(onDescendantAdded))
        startSolverLoop()
    end
end)

-- AUTO TOUCH: Proximity-Prioritized Flagging & Touching
fastBtn.MouseButton1Click:Connect(function()
    if not isScanning then return end
    
    isAutoTouching = not isAutoTouching 
    
    if isAutoTouching then
        fastBtn.Text = "AUTO TOUCH: ON"
        fastBtn.TextColor3 = Color3.fromRGB(0, 255, 0)
        status.Text = "Status: Proximity Active"
        
        task.spawn(function()
            while isAutoTouching and isScanning do
                if not pendingReset then
                    local charPos = getCharPos()
                    refreshFlagCache() -- Ultra-fast cache refresh
                    
                    -- 1. GATHER & REMOVE MISPLACED FLAGS (CLOSEST FIRST)
                    local misplacedFlags = {}
                    for part, _ in pairs(knownSafe) do
                        if part and part:IsDescendantOf(workspace) and isUnrevealed(part) and hasFlag(part) then
                            table.insert(misplacedFlags, part)
                        end
                    end
                    sortByProximity(misplacedFlags, charPos)
                    
                    for _, part in ipairs(misplacedFlags) do
                        if not isAutoTouching or not isScanning or pendingReset then break end
                        clickTile(part)
                        task.wait(0.02)
                    end
                    
                    -- 2. GATHER & FLAG MINES (CLOSEST FIRST)
                    local minesToFlag = {}
                    for part, _ in pairs(knownMines) do
                        if part and part:IsDescendantOf(workspace) and isUnrevealed(part) and not hasFlag(part) then
                            table.insert(minesToFlag, part)
                        end
                    end
                    sortByProximity(minesToFlag, charPos)
                    
                    for _, part in ipairs(minesToFlag) do
                        if not isAutoTouching or not isScanning or pendingReset then break end
                        clickTile(part)
                        task.wait(0.02)
                    end
                    
                    -- 3. GATHER & REVEAL SAFE TILES (CLOSEST FIRST)
                    local safeToReveal = {}
                    for part, _ in pairs(knownSafe) do
                        if part and part:IsDescendantOf(workspace) and isUnrevealed(part) and not hasFlag(part) then
                            table.insert(safeToReveal, part)
                        end
                    end
                    sortByProximity(safeToReveal, charPos)
                    
                    if #safeToReveal > 0 then
                        local batchCount = math.min(#safeToReveal, BATCH_SIZE)
                        for i = 1, batchCount do
                            local part = safeToReveal[i]
                            task.spawn(function()
                                if isAutoTouching and isScanning and not pendingReset then
                                    triggerTileTouch(part)
                                end
                            end)
                        end
                        task.wait(0.05)
                    else
                        task.wait(0.08)
                    end
                else
                    task.wait(0.1) 
                end
            end
            
            isAutoTouching = false
            fastBtn.Text = "AUTO TOUCH: OFF"
            fastBtn.TextColor3 = Color3.fromRGB(0, 255, 255)
            if isScanning then status.Text = "Status: Scanning..." end
        end)
    else
        isAutoTouching = false
        fastBtn.Text = "AUTO TOUCH: OFF"
        fastBtn.TextColor3 = Color3.fromRGB(0, 255, 255)
        status.Text = "Status: Scanning..."
    end
end)

antiLagBtn.MouseButton1Click:Connect(function()
    isAntiLagOn = not isAntiLagOn
    if isAntiLagOn then
        antiLagBtn.Text = "ANTI-LAG: ON"
        antiLagBtn.TextColor3 = Color3.fromRGB(0, 255, 0)
    else
        antiLagBtn.Text = "ANTI-LAG: OFF"
        antiLagBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    end
end)

billboardBtn.MouseButton1Click:Connect(function()
    isBillboardOn = not isBillboardOn
    if isBillboardOn then
        billboardBtn.Text = "BILLBOARD: ON"
        billboardBtn.TextColor3 = Color3.fromRGB(0, 255, 0)
        if isScanning then
            for _, part in ipairs(cachedParts) do
                updateBillboard(part)
            end
        end
    else
        billboardBtn.Text = "BILLBOARD: OFF"
        billboardBtn.TextColor3 = Color3.fromRGB(255, 255, 0)
        removeAllBillboards()
    end
end)

destroyBtn.MouseButton1Click:Connect(function()
    isScanning = false
    isAutoTouching = false
    isBillboardOn = false
    isAntiLagOn = false
    for _, conn in pairs(connections) do conn:Disconnect() end
    removeAllBillboards()
    screenGui:Destroy()
end)
