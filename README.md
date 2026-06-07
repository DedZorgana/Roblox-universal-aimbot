# Roblox-universal-aimbot
```
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Camera = workspace.CurrentCamera

local LocalPlayer = Players.LocalPlayer

-- Configuration
local Settings = {
    Enabled = false,
    FOV = 180,
    TeamCheck = true,
    WallCheck = true,
    TargetPart = "HumanoidRootPart",
    TargetType = "ALL", -- ALL, PLAYER, NPC
    Smoothness = 0.3,   -- Aim smoothing (0 = instant, 1 = slow)
    Keybind = Enum.KeyCode.Q
}

-- Constants
local MIN_FOV = 50
local MAX_FOV = 360
local FOV_STEP = 10

-- GUI Elements
local GUI = {
    ScreenGui = nil,
    MainFrame = nil,
    Labels = {},
    Buttons = {}
}

-- Drawing objects
local FOVCircle = Drawing.new("Circle")
FOVCircle.Color = Color3.fromRGB(255, 255, 255)
FOVCircle.Thickness = 2
FOVCircle.Filled = false
FOVCircle.Transparency = 0.5
FOVCircle.Visible = false

local CurrentTarget = nil
local StartTime = tick()

-- Helper Functions
local function UpdateRuntimeLabel()
    local runtime = math.floor(tick() - StartTime)
    local hours = math.floor(runtime / 3600)
    local minutes = math.floor((runtime % 3600) / 60)
    local seconds = runtime % 60
    
    local timeString = string.format("%02d:%02d:%02d", hours, minutes, seconds)
    GUI.Labels.Runtime.Text = "Runtime: " .. timeString
end

local function IsCharacterValid(character)
    if not character or not character.Parent then
        return false
    end
    
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    local targetPart = character:FindFirstChild(Settings.TargetPart)
    
    return humanoid and targetPart and humanoid.Health > 0
end

local function IsVisible(character)
    if not Settings.WallCheck then
        return true
    end
    
    local targetPart = character:FindFirstChild(Settings.TargetPart)
    if not targetPart then
        return false
    end
    
    local origin = Camera.CFrame.Position
    local direction = targetPart.Position - origin
    
    local raycastParams = RaycastParams.new()
    raycastParams.FilterType = Enum.RaycastFilterType.Blacklist
    raycastParams.FilterDescendantsInstances = {
        LocalPlayer.Character,
        character
    }
    
    local result = workspace:Raycast(origin, direction, raycastParams)
    return result == nil
end

local function GetTargets()
    local targets = {}
    
    -- Get players
    if Settings.TargetType == "ALL" or Settings.TargetType == "PLAYER" then
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer 
               and player.Character 
               and IsCharacterValid(player.Character) then
                
                if Settings.TeamCheck and player.Team == LocalPlayer.Team then
                    continue
                end
                
                table.insert(targets, {
                    Character = player.Character,
                    Type = "PLAYER"
                })
            end
        end
    end
    
    -- Get NPCs
    if Settings.TargetType == "ALL" or Settings.TargetType == "NPC" then
        for _, object in ipairs(workspace:GetChildren()) do
            if object:IsA("Model") 
               and object:FindFirstChildOfClass("Humanoid")
               and not Players:GetPlayerFromCharacter(object)
               and IsCharacterValid(object) then
                
                table.insert(targets, {
                    Character = object,
                    Type = "NPC"
                })
            end
        end
    end
    
    return targets
end

local function GetClosestTarget()
    local centerScreen = Vector2.new(
        Camera.ViewportSize.X / 2,
        Camera.ViewportSize.Y / 2
    )
    
    local closestDistance = Settings.FOV
    local closestTarget = nil
    
    local targets = GetTargets()
    
    for _, targetData in ipairs(targets) do
        local character = targetData.Character
        local targetPart = character:FindFirstChild(Settings.TargetPart)
        
        if targetPart then
            local screenPoint, onScreen = Camera:WorldToViewportPoint(targetPart.Position)
            
            if onScreen and IsVisible(character) then
                local distance = (Vector2.new(screenPoint.X, screenPoint.Y) - centerScreen).Magnitude
                
                if distance < closestDistance then
                    closestDistance = distance
                    closestTarget = character
                end
            end
        end
    end
    
    return closestTarget
end

local function SmoothAim(targetPosition)
    local currentCFrame = Camera.CFrame
    local targetCFrame = CFrame.new(currentCFrame.Position, targetPosition)
    
    -- Apply smoothing using interpolation
    local smoothedCFrame = currentCFrame:Lerp(targetCFrame, 1 - Settings.Smoothness)
    Camera.CFrame = smoothedCFrame
end

-- GUI Creation Functions
local function CreateButton(text, position, callback)
    local button = Instance.new("TextButton")
    button.Size = UDim2.new(0, 110, 0, 35)
    button.Position = UDim2.new(position.X, position.Y, position.Z, position.W)
    button.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
    button.BackgroundTransparency = 0
    button.TextColor3 = Color3.fromRGB(255, 255, 255)
    button.Text = text
    button.Font = Enum.Font.GothamSemibold
    button.TextSize = 14
    button.Parent = GUI.MainFrame
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = button
    
    if callback then
        button.MouseButton1Click:Connect(callback)
    end
    
    return button
end

local function CreateLabel(text, position, color)
    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, 0, 0, 25)
    label.Position = UDim2.new(position.X, position.Y, position.Z, position.W)
    label.BackgroundTransparency = 1
    label.Text = text
    label.TextColor3 = color or Color3.fromRGB(180, 180, 180)
    label.Font = Enum.Font.Gotham
    label.TextSize = 13
    label.Parent = GUI.MainFrame
    
    return label
end

local function SetupGUI()
    GUI.ScreenGui = Instance.new("ScreenGui")
    GUI.ScreenGui.Name = "AimBotGUI"
    GUI.ScreenGui.ResetOnSpawn = false
    GUI.ScreenGui.Parent = LocalPlayer.PlayerGui
    
    GUI.MainFrame = Instance.new("Frame")
    GUI.MainFrame.Size = UDim2.new(0, 280, 0, 360)
    GUI.MainFrame.Position = UDim2.new(0, 20, 0.5, -180)
    GUI.MainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
    GUI.MainFrame.BackgroundTransparency = 0
    GUI.MainFrame.Active = true
    GUI.MainFrame.Draggable = true
    GUI.MainFrame.Parent = GUI.ScreenGui
    
    local mainCorner = Instance.new("UICorner")
    mainCorner.CornerRadius = UDim.new(0, 10)
    mainCorner.Parent = GUI.MainFrame
    
    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, 0, 0, 40)
    title.BackgroundTransparency = 1
    title.Text = "AimBot v2.0"
    title.TextColor3 = Color3.fromRGB(255, 255, 255)
    title.Font = Enum.Font.GothamBold
    title.TextSize = 20
    title.Parent = GUI.MainFrame
    
    local separator = Instance.new("Frame")
    separator.Size = UDim2.new(1, -20, 0, 1)
    separator.Position = UDim2.new(0, 10, 0, 40)
    separator.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    separator.BackgroundTransparency = 0
    separator.Parent = GUI.MainFrame
    
    GUI.Labels.Runtime = CreateLabel("Runtime: 00:00:00", {X = 0, Y = 0, Z = 0, W = 45}, Color3.fromRGB(200, 200, 200))
    
    local fovLabel = CreateLabel("FOV: " .. Settings.FOV, {X = 0, Y = 0, Z = 0, W = 75}, Color3.fromRGB(200, 200, 200))
    
    -- Create buttons
    GUI.Buttons.Toggle = CreateButton("DISABLED", {X = 0.05, Y = 0, Z = 0, W = 110}, function()
        Settings.Enabled = not Settings.Enabled
        GUI.Buttons.Toggle.Text = Settings.Enabled and "ENABLED" or "DISABLED"
        GUI.Buttons.Toggle.BackgroundColor3 = Settings.Enabled and Color3.fromRGB(0, 150, 0) or Color3.fromRGB(35, 35, 35)
        FOVCircle.Visible = Settings.Enabled
    end)
    
    GUI.Buttons.Team = CreateButton("TEAM: ON", {X = 0.05, Y = 0, Z = 0, W = 155}, function()
        Settings.TeamCheck = not Settings.TeamCheck
        GUI.Buttons.Team.Text = Settings.TeamCheck and "TEAM: ON" or "TEAM: OFF"
    end)
    
    GUI.Buttons.Wall = CreateButton("WALL: ON", {X = 0.05, Y = 0, Z = 0, W = 200}, function()
        Settings.WallCheck = not Settings.WallCheck
        GUI.Buttons.Wall.Text = Settings.WallCheck and "WALL: ON" or "WALL: OFF"
    end)
    
    GUI.Buttons.FOVPlus = CreateButton("FOV +" .. FOV_STEP, {X = 0.05, Y = 0, Z = 0, W = 65}, function()
        Settings.FOV = math.min(MAX_FOV, Settings.FOV + FOV_STEP)
        fovLabel.Text = "FOV: " .. Settings.FOV
    end)
    
    GUI.Buttons.FOVMinus = CreateButton("FOV -" .. FOV_STEP, {X = 0.05, Y = 0, Z = 0, W = 110}, function()
        Settings.FOV = math.max(MIN_FOV, Settings.FOV - FOV_STEP)
        fovLabel.Text = "FOV: " .. Settings.FOV
    end)
    
    GUI.Buttons.Part = CreateButton("TARGET: BODY", {X = 0.05, Y = 0, Z = 0, W = 155}, function()
        Settings.TargetPart = Settings.TargetPart == "HumanoidRootPart" and "Head" or "HumanoidRootPart"
        GUI.Buttons.Part.Text = Settings.TargetPart == "HumanoidRootPart" and "TARGET: BODY" or "TARGET: HEAD"
    end)
    
    GUI.Buttons.Type = CreateButton("TYPE: ALL", {X = 0.05, Y = 0, Z = 0, W = 200}, function()
        if Settings.TargetType == "ALL" then
            Settings.TargetType = "PLAYER"
            GUI.Buttons.Type.Text = "TYPE: PLAYER"
        elseif Settings.TargetType == "PLAYER" then
            Settings.TargetType = "NPC"
            GUI.Buttons.Type.Text = "TYPE: NPC"
        else
            Settings.TargetType = "ALL"
            GUI.Buttons.Type.Text = "TYPE: ALL"
        end
    end)
    
    -- Adjust button positions
    local yOffset = 65
    for _, button in pairs(GUI.Buttons) do
        button.Position = UDim2.new(0.05, 0, 0, yOffset)
        yOffset = yOffset + 45
    end
    
    -- Reposition specific buttons to two columns
    GUI.Buttons.Toggle.Position = UDim2.new(0.05, 0, 0, 65)
    GUI.Buttons.Team.Position = UDim2.new(0.05, 0, 0, 110)
    GUI.Buttons.Wall.Position = UDim2.new(0.05, 0, 0, 155)
    GUI.Buttons.FOVPlus.Position = UDim2.new(0.55, 0, 0, 65)
    GUI.Buttons.FOVMinus.Position = UDim2.new(0.55, 0, 0, 110)
    GUI.Buttons.Part.Position = UDim2.new(0.55, 0, 0, 155)
    GUI.Buttons.Type.Position = UDim2.new(0.55, 0, 0, 200)
end

-- Keybind Handler
local function SetupKeybind()
    UserInputService.InputBegan:Connect(function(input, gameProcessed)
        if gameProcessed then return end
        
        if input.KeyCode == Settings.Keybind then
            Settings.Enabled = not Settings.Enabled
            GUI.Buttons.Toggle.Text = Settings.Enabled and "ENABLED" or "DISABLED"
            GUI.Buttons.Toggle.BackgroundColor3 = Settings.Enabled and Color3.fromRGB(0, 150, 0) or Color3.fromRGB(35, 35, 35)
            FOVCircle.Visible = Settings.Enabled
        end
    end)
end

-- Main Loop
local function Initialize()
    SetupGUI()
    SetupKeybind()
    
    RunService.RenderStepped:Connect(function()
        UpdateRuntimeLabel()
        
        -- Update FOV Circle
        FOVCircle.Position = Vector2.new(
            Camera.ViewportSize.X / 2,
            Camera.ViewportSize.Y / 2
        )
        FOVCircle.Radius = Settings.FOV
        FOVCircle.Visible = Settings.Enabled
        
        if not Settings.Enabled then
            return
        end
        
        CurrentTarget = GetClosestTarget()
        
        if CurrentTarget then
            local targetPart = CurrentTarget:FindFirstChild(Settings.TargetPart)
            if targetPart then
                SmoothAim(targetPart.Position)
            end
        end
    end)
end

-- Start the script
Initialize()
```

```
📖 Description:

📢 Open Source – This script is completely open source. Modifications, improvements, and customization are fully welcome!
Distributed under GNU General Public License (GPL).

🎯 How it works
The aimbot only locks onto targets if there are no obstacles between you and the enemy.

✅ WallCheck is ON by default – the script performs a raycast check to ensure the target is visible

✅ If something blocks the view (wall, building, terrain), the aimbot will NOT lock onto that target

✅ You can toggle WallCheck OFF via the GUI if you want to aim through walls
```
