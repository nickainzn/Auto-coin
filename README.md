local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")
local GuiService = game:GetService("GuiService")

-- Lock variable to prevent multiple script executions
local isRunning = false
local coinCollectorThread
local hasReset = false -- Flag to track if the character has been reset
local hasExecutedOnce = false -- Flag to ensure second script executes only once

-- Default tween speed
local TWEEN_SPEED = 20
local TELEPORT_DISTANCE = 200

-- Function to execute the coin collection script
local function startCoinCollector()
    if isRunning then return end
    isRunning = true

    -- Get the local player
    local localPlayer = Players.LocalPlayer

    -- Function to get the current character and ensure it's fully loaded
    local function getCharacter()
        return localPlayer.Character or localPlayer.CharacterAdded:Wait()
    end

    -- Initialize character and humanoidRootPart
    local function initializeCharacter()
        local character = getCharacter()
        local humanoidRootPart = character:WaitForChild("HumanoidRootPart", 5)
        local humanoid = character:WaitForChild("Humanoid", 5)
        return character, humanoidRootPart, humanoid
    end

    -- Variables for character and humanoid
    local character, humanoidRootPart, humanoid = initializeCharacter()

    if not humanoidRootPart then
        warn("HumanoidRootPart not found!")
        isRunning = false
        return
    end

    if not humanoid then
        warn("Humanoid not found!")
        isRunning = false
        return
    end

    -- List of possible maps and their CoinContainer paths
    local mapPaths = {
        "IceCastle",
        "SkiLodge",
        "Station",
        "LogCabin",
        "Bank2",
        "BioLab",
        "House2",
        "Factory",
        "Hospital3",
        "Hotel",
        "Mansion2",
        "MilBase",
        "Office3",
        "PoliceStation",
        "Workplace",
        "ResearchFacility",
        "ChristmasItaly"
    }

    -- Keep track of visited coins to prevent revisiting
    local visitedCoins = {}

    -- Function to find the active map's CoinContainer
    local function findActiveCoinContainer()
        for _, mapName in ipairs(mapPaths) do
            local map = Workspace:FindFirstChild(mapName)
            if map then
                local coinContainer = map:FindFirstChild("CoinContainer")
                if coinContainer then
                    return coinContainer
                end
            end
        end
        return nil
    end

    -- Function to find the nearest coin
    local function findNearestCoin(coinContainer)
        local nearestCoin = nil
        local shortestDistance = math.huge

        if coinContainer then
            for _, coin in ipairs(coinContainer:GetChildren()) do
                if coin:IsA("BasePart") and not visitedCoins[coin] then
                    local distance = (humanoidRootPart.Position - coin.Position).Magnitude
                    if distance < shortestDistance then
                        shortestDistance = distance
                        nearestCoin = coin
                    end
                end
            end
        else
            warn("CoinContainer not found or empty!")
        end

        return nearestCoin
    end

    -- Function to teleport to a coin
    local function teleportToCoin(coin)
        if coin then
            humanoidRootPart.CFrame = CFrame.new(coin.Position)
            visitedCoins[coin] = true -- Mark the coin as visited
        else
            warn("No coin to teleport to!")
        end
    end

    -- Function to tween to a coin
    local function tweenToCoin(coin)
        if coin then
            visitedCoins[coin] = true -- Mark the coin as visited
            local distance = (humanoidRootPart.Position - coin.Position).Magnitude
            local tweenInfo = TweenInfo.new(distance / TWEEN_SPEED, Enum.EasingStyle.Linear)
            local goal = {CFrame = CFrame.new(coin.Position)}
            local tween = TweenService:Create(humanoidRootPart, tweenInfo, goal)
            tween:Play()

            -- When the tween starts, enable auto reset
            hasReset = false -- Allow reset once the tween starts

            tween.Completed:Wait() -- Wait for the tween to finish
        else
            warn("No coin to tween to!")
        end
    end

    -- Function to play falling animation
    local function playFallingAnimation()
        humanoid:SetStateEnabled(Enum.HumanoidStateType.Freefall, true)
        humanoid:ChangeState(Enum.HumanoidStateType.Freefall)
    end

    -- Function to check if all CoinVisuals are gone and reset character
    local function checkForAllCoinVisualsGone()
        local coinContainer = findActiveCoinContainer()

        if coinContainer then
            local allCoinVisualsGone = true

            -- Check if any coin still has a CoinVisual
            for _, coin in ipairs(coinContainer:GetChildren()) do
                if coin:IsA("BasePart") then
                    local coinVisual = coin:FindFirstChild("CoinVisual")
                    if coinVisual then
                        allCoinVisualsGone = false
                        break
