-- Load Rayfield UI Library
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

-- Create Window
local Window = Rayfield:CreateWindow({
    Name = "Pro Hub | Aimbot & ESP",
    Icon = 0,
    LoadingTitle = "Loading Combat Suite...",
    LoadingSubtitle = "by You",
    Theme = "Default", 
    ToggleUIKeybind = "K", -- Press 'K' to open/close the menu
    ConfigurationSaving = {
        Enabled = true,
        FolderName = "ProHubConfig",
        FileName = "MainSettings"
    },
    KeySystem = false, -- Set to true if you want a key system
})

-- Create Tab
local Tab = Window:CreateTab("Combat & Visuals", 4483362458)

-- Services
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Camera = workspace.CurrentCamera
local LocalPlayer = Players.LocalPlayer

-- Configuration Variables
local AimbotEnabled = false
local AimbotFOV = 80
local AimbotSmooth = 5
local TeamCheck = true
local WallCheck = true
local TargetPart = "Head"

local ESPEnabled = false
local BoxESP = false
local HealthBarESP = false
local NameESP = false
local DistanceESP = false
local TracerESP = false
local ESPLimitDistance = 1500

-- Visual Drawings Storage
local ESPStorage = {}
local FOVCircle = Drawing.new("Circle")
FOVCircle.Visible = false
FOVCircle.Thickness = 1
FOVCircle.Color = Color3.fromRGB(255, 255, 255)
FOVCircle.Filled = false

-- ==========================================
-- 1. RAYFIELD UI COMPONENTS
-- ==========================================

Tab:CreateSection("Aimbot Settings")

Tab:CreateToggle({
    Name = "Enable Aimbot",
    CurrentValue = false,
    Flag = "AimbotToggle",
    Callback = function(Value)
        AimbotEnabled = Value
        FOVCircle.Visible = Value
    end,
})

Tab:CreateSlider({
    Name = "Aimbot FOV",
    Range = {20, 300},
    Increment = 5,
    Suffix = "px",
    CurrentValue = 80,
    Flag = "AimbotFOV",
    Callback = function(Value)
        AimbotFOV = Value
        FOVCircle.Radius = Value
    end,
})

Tab:CreateSlider({
    Name = "Aimbot Smoothness",
    Range = {1, 20},
    Increment = 1,
    Suffix = "",
    CurrentValue = 5,
    Flag = "AimbotSmooth",
    Callback = function(Value)
        AimbotSmooth = Value
    end,
})

Tab:CreateToggle({
    Name = "Team Check",
    CurrentValue = true,
    Flag = "TeamCheck",
    Callback = function(Value)
        TeamCheck = Value
    end,
})

Tab:CreateToggle({
    Name = "Wall Check",
    CurrentValue = true,
    Flag = "WallCheck",
    Callback = function(Value)
        WallCheck = Value
    end,
})

Tab:CreateSection("ESP Settings")

Tab:CreateToggle({
    Name = "Enable ESP",
    CurrentValue = false,
    Flag = "ESPToggle",
    Callback = function(Value)
        ESPEnabled = Value
        if not Value then
            for _, drawings in pairs(ESPStorage) do
                for _, drawing in pairs(drawings) do
                    drawing.Visible = false
                end
            end
        end
    end,
})

Tab:CreateToggle({
    Name = "Box ESP",
    CurrentValue = false,
    Flag = "BoxESP",
    Callback = function(Value) BoxESP = Value end,
})

Tab:CreateToggle({
    Name = "Health Bar",
    CurrentValue = false,
    Flag = "HealthBar",
    Callback = function(Value) HealthBarESP = Value end,
})

Tab:CreateToggle({
    Name = "Name ESP",
    CurrentValue = false,
    Flag = "NameESP",
    Callback = function(Value) NameESP = Value end,
})

Tab:CreateToggle({
    Name = "Distance ESP",
    CurrentValue = false,
    Flag = "DistanceESP",
    Callback = function(Value) DistanceESP = Value end,
})

Tab:CreateToggle({
    Name = "Snaplines (Tracers)",
    CurrentValue = false,
    Flag = "TracerESP",
    Callback = function(Value) TracerESP = Value end,
})

-- ==========================================
-- 2. ESP SETUP FUNCTIONS
-- ==========================================
local function CreateESP(player)
    if ESPStorage[player] then return end

    local drawings = {
        Box = Drawing.new("Square"),
        HealthBarBg = Drawing.new("Square"),
        HealthBar = Drawing.new("Square"),
        Name = Drawing.new("Text"),
        Tracer = Drawing.new("Line"),
    }

    drawings.Box.Visible = false
    drawings.Box.Color = Color3.fromRGB(255, 255, 255)
    drawings.Box.Thickness = 1.5
    drawings.Box.Filled = false

    drawings.HealthBarBg.Visible = false
    drawings.HealthBarBg.Color = Color3.fromRGB(0, 0, 0)
    drawings.HealthBarBg.Thickness = 1
    drawings.HealthBarBg.Filled = true

    drawings.HealthBar.Visible = false
    drawings.HealthBar.Color = Color3.fromRGB(0, 255, 0)
    drawings.HealthBar.Thickness = 1
    drawings.HealthBar.Filled = true

    drawings.Name.Visible = false
    drawings.Name.Color = Color3.fromRGB(255, 255, 255)
    drawings.Name.Size = 14
    drawings.Name.Center = true
    drawings.Name.Outline = true

    drawings.Tracer.Visible = false
    drawings.Tracer.Color = Color3.fromRGB(255, 255, 255)
    drawings.Tracer.Thickness = 1

    ESPStorage[player] = drawings
end

local function RemoveESP(player)
    if ESPStorage[player] then
        for _, drawing in pairs(ESPStorage[player]) do
            drawing:Remove()
        end
        ESPStorage[player] = nil
    end
end

Players.PlayerAdded:Connect(CreateESP)
Players.PlayerRemoving:Connect(RemoveESP)

for _, player in ipairs(Players:GetPlayers()) do
    if player ~= LocalPlayer then CreateESP(player) end
end

-- ==========================================
-- 3. HELPER FUNCTIONS (Raycasting & Targeting)
-- ==========================================
local function IsVisible(part, character)
    if not WallCheck then return true end
    local origin = Camera.CFrame.Position
    local raycastParams = RaycastParams.new()
    raycastParams.FilterType = Enum.RaycastFilterType.Blacklist
    raycastParams.FilterDescendantsInstances = {LocalPlayer.Character, character}
    raycastParams.IgnoreWater = true
    
    local result = workspace:Raycast(origin, part.Position - origin, raycastParams)
    return result == nil
end

local function GetClosestTarget()
    local target = nil
    local shortestDist = AimbotFOV

    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character then
            if TeamCheck and player.Team == LocalPlayer.Team then continue end

            local humanoid = player.Character:FindFirstChildOfClass("Humanoid")
            local part = player.Character:FindFirstChild(TargetPart)
            
            if humanoid and humanoid.Health > 0 and part then
                if not IsVisible(part, player.Character) then continue end

                local screenPos, onScreen = Camera:WorldToViewportPoint(part.Position)
                if onScreen then
                    local mousePos = UserInputService:GetMouseLocation()
                    local magnitude = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
                    
                    if magnitude < shortestDist then
                        shortestDist = magnitude
                        target = part
                    end
                end
            end
        end
    end
    
    return target
end

-- ==========================================
-- 4. MAIN RENDER LOOP (Aimbot + ESP Updates)
-- ==========================================
RunService.RenderStepped:Connect(function()
    -- Update FOV Circle
    if AimbotEnabled then
        FOVCircle.Position = UserInputService:GetMouseLocation()
    end

    -- --- Aimbot Logic (Hold Right Click) ---
    if AimbotEnabled and UserInputService:IsMouseButtonPressed(Enum.UserInputType.MouseButton2) then
        local targetPart = GetClosestTarget()
        if targetPart then
            local targetCFrame = CFrame.new(Camera.CFrame.Position, targetPart.Position)
            Camera.CFrame = Camera.CFrame:Lerp(targetCFrame, 1 / math.max(AimbotSmooth, 1))
        end
    end

    -- --- ESP Logic ---
    if not ESPEnabled then return end

    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            local drawings = ESPStorage[player]
            local character = player.Character
            local humanoidRootPart = character and character:FindFirstChild("HumanoidRootPart")
            local humanoid = character and character:FindFirstChildOfClass("Humanoid")

            local shouldShow = false

            if character and humanoidRootPart and humanoid and humanoid.Health > 0 then
                if not (TeamCheck and player.Team == LocalPlayer.Team) then
                    local rootPos, onScreen = Camera:WorldToViewportPoint(humanoidRootPart.Position)
                    local distance = (Camera.CFrame.Position - humanoidRootPart.Position).Magnitude

                    if onScreen and distance <= ESPLimitDistance then
                        shouldShow = true

                        local topPos = Camera:WorldToViewportPoint((humanoidRootPart.CFrame * CFrame.new(0, 3, 0)).Position)
                        local bottomPos = Camera:WorldToViewportPoint((humanoidRootPart.CFrame * CFrame.new(0, -3.5, 0)).Position)
                        
                        local height = math.abs(topPos.Y - bottomPos.Y)
                        local width = height / 2

                        -- Box ESP
                        if BoxESP then
                            drawings.Box.Size = Vector2.new(width, height)
                            drawings.Box.Position = Vector2.new(rootPos.X - width / 2, rootPos.Y - height / 2)
                            drawings.Box.Visible = true
                        else
                            drawings.Box.Visible = false
                        end

                        -- Health Bar
                        if HealthBarESP then
                            local healthPercent = humanoid.Health / humanoid.MaxHealth
                            local barHeight = height * healthPercent
                            
                            drawings.HealthBarBg.Size = Vector2.new(3, height + 2)
                            drawings.HealthBarBg.Position = Vector2.new((rootPos.X - width / 2) - 6, (rootPos.Y - height / 2) - 1)
                            drawings.HealthBarBg.Visible = true

                            drawings.HealthBar.Size = Vector2.new(1, barHeight)
                            drawings.HealthBar.Position = Vector2.new((rootPos.X - width / 2) - 5, (rootPos.Y - height / 2) + (height - barHeight))
                            drawings.HealthBar.Color = Color3.fromRGB(255 * (1 - healthPercent), 255 * healthPercent, 0)
                            drawings.HealthBar.Visible = true
                        else
                            drawings.HealthBarBg.Visible = false
                            drawings.HealthBar.Visible = false
                        end

                        -- Name & Distance Text
                        local infoText = ""
                        if NameESP then infoText = player.Name end
                        if DistanceESP then infoText = infoText .. " [" .. math.floor(distance) .. "m]" end

                        if (NameESP or DistanceESP) and infoText ~= "" then
                            drawings.Name.Text = infoText
                            drawings.Name.Position = Vector2.new(rootPos.X, (rootPos.Y - height / 2) - 20)
                            drawings.Name.Visible = true
                        else
                            drawings.Name.Visible = false
                        end

                        -- Tracer
                        if TracerESP then
                            drawings.Tracer.From = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y)
                            drawings.Tracer.To = Vector2.new(rootPos.X, rootPos.Y + height / 2)
                            drawings.Tracer.Visible = true
                        else
                            drawings.Tracer.Visible = false
                        end
                    end
                end
            end

            if not shouldShow and drawings then
                drawings.Box.Visible = false
                drawings.HealthBarBg.Visible = false
                drawings.HealthBar.Visible = false
                drawings.Name.Visible = false
                drawings.Tracer.Visible = false
            end
        end
    end
end)

-- Notify user that script is loaded
Rayfield:Notify({
    Title = "Script Loaded!",
    Content = "Press 'K' to open or close the Rayfield menu.",
    Duration = 6.5,
    Image = 4483362458,
})
