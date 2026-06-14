-- leaked by script haven join for cheapest sources https://discord.gg/xfwA9fNFMe

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local CoreGui = game:GetService("CoreGui")
local UserInputService = game:GetService("UserInputService")
local lp = Players.LocalPlayer

-- Configuration
local RANGE = 70           
local SPEED = 58.76       

-- State
local enabled = false
local connections = {}
local gui = nil
local btn = nil

-- Helper: get a bat tool from the player's inventory
local function getBat()
    local char = lp.Character
    if char then
        for _, tool in ipairs(char:GetChildren()) do
            if tool:IsA("Tool") and tool.Name:lower():find("bat") then
                return tool
            end
        end
    end
    for _, tool in ipairs(lp.Backpack:GetChildren()) do
        if tool:IsA("Tool") and tool.Name:lower():find("bat") then
            return tool
        end
    end
    return nil
end

-- Find nearest enemy humanoid root part
local function getNearestEnemy(hrp)
    local nearest, minDist = nil, RANGE
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= lp and p.Character then
            local targetHRP = p.Character:FindFirstChild("HumanoidRootPart")
            local targetHum = p.Character:FindFirstChildOfClass("Humanoid")
            if targetHRP and targetHum and targetHum.Health > 0 then
                local d = (targetHRP.Position - hrp.Position).Magnitude
                if d < minDist then
                    nearest = targetHRP
                    minDist = d
                end
            end
        end
    end
    return nearest
end

-- Create draggable toggle button (black background + white outline)
local function createButton()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "MeleeAimbot"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = CoreGui

    btn = Instance.new("TextButton")
    btn.Size = UDim2.new(0, 160, 0, 45)
    btn.Position = UDim2.new(0.5, -80, 0.8, 0)
    btn.Text = "MELEE AIMBOT: OFF"
    btn.TextColor3 = Color3.new(1, 1, 1)          -- white text
    btn.BackgroundColor3 = Color3.new(0, 0, 0)    -- black background
    btn.Font = Enum.Font.GothamBold
    btn.TextSize = 15
    btn.Parent = screenGui

    -- Rounded corners
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 8)
    corner.Parent = btn

    -- White outline
    local outline = Instance.new("UIStroke")
    outline.Color = Color3.new(1, 1, 1)
    outline.Thickness = 2
    outline.Parent = btn

    -- Make button draggable
    local dragging, dragStart, startPos = false, nil, nil
    btn.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = Vector2.new(input.Position.X, input.Position.Y)
            startPos = btn.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false
                end
            end)
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
            local delta = Vector2.new(input.Position.X - dragStart.X, input.Position.Y - dragStart.Y)
            btn.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X,
                                     startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end
    end)

    return screenGui, btn
end

-- Main aimbot logic
local function startAimbot()
    if connections.aimbot then return end

    local char = lp.Character or lp.CharacterAdded:Wait()
    local hrp = char:WaitForChild("HumanoidRootPart")
    local hum = char:WaitForChild("Humanoid")

    -- Attachments and physics objects
    local attach = Instance.new("Attachment", hrp)

    local linVel = Instance.new("LinearVelocity", hrp)
    linVel.Attachment0 = attach
    linVel.MaxForce = 1e9
    linVel.ForceLimitMode = Enum.ForceLimitMode.PerAxis
    linVel.MaxAxesForce = Vector3.new(1e9, 1e9, 1e9)
    linVel.RelativeTo = Enum.ActuatorRelativeTo.World
    linVel.Enabled = false

    local align = Instance.new("AlignOrientation", hrp)
    align.Attachment0 = attach
    align.MaxTorque = math.huge
    align.Responsiveness = 200
    align.Enabled = false

    local currentTarget = nil

    connections.aimbot = RunService.Heartbeat:Connect(function()
        if not enabled then return end

        char = lp.Character
        if not char then return end
        hrp = char:FindFirstChild("HumanoidRootPart")
        hum = char:FindFirstChildOfClass("Humanoid")
        if not hrp or not hum then return end

        -- Validate target
        if currentTarget and currentTarget.Parent then
            local tHum = currentTarget.Parent:FindFirstChildOfClass("Humanoid")
            if not tHum or tHum.Health <= 0 then
                currentTarget = nil
            end
        else
            currentTarget = nil
        end

        if not currentTarget then
            currentTarget = getNearestEnemy(hrp)
        end

        if not currentTarget then
            linVel.Enabled = false
            align.Enabled = false
            hum.AutoRotate = true
            return
        end

        -- Disable auto rotate to let alignment control facing
        hum.AutoRotate = false

        -- Move towards target (aim slightly ahead)
        local targetPos = currentTarget.Position + currentTarget.CFrame.LookVector * 0.4
        local dir = targetPos - hrp.Position

        linVel.Enabled = true
        linVel.VectorVelocity = dir.Magnitude > 0.1
            and Vector3.new(dir.Unit.X * SPEED, dir.Unit.Y * SPEED, dir.Unit.Z * SPEED)
            or Vector3.zero

        -- Face target
        local lookDir = Vector3.new(targetPos.X - hrp.Position.X, 0, targetPos.Z - hrp.Position.Z)
        if lookDir.Magnitude > 0.01 then
            align.Enabled = true
            align.CFrame = CFrame.lookAt(hrp.Position, hrp.Position + lookDir)
        end

        -- Equip and swing bat
        local bat = getBat()
        if bat then
            if bat.Parent ~= char then
                hum:EquipTool(bat)
            end
            bat:Activate()

            -- Force touch interest to guarantee hit detection
            local handle = bat:FindFirstChild("Handle")
            if handle then
                for _, p in ipairs(Players:GetPlayers()) do
                    if p ~= lp and p.Character then
                        local tHRP = p.Character:FindFirstChild("HumanoidRootPart")
                        if tHRP and (tHRP.Position - hrp.Position).Magnitude <= 8 then
                            for _, part in ipairs(p.Character:GetChildren()) do
                                if part:IsA("BasePart") then
                                    pcall(function()
                                        firetouchinterest(handle, part, 0)
                                        firetouchinterest(handle, part, 1)
                                    end)
                                end
                            end
                        end
                    end
                end
            end
        end
    end)

    -- Store objects for cleanup
    connections.attach = attach
    connections.linVel = linVel
    connections.align = align
end

local function stopAimbot()
    if connections.aimbot then
        connections.aimbot:Disconnect()
        connections.aimbot = nil
    end
    if connections.attach then
        connections.attach:Destroy()
        connections.attach = nil
    end
    if connections.linVel then
        connections.linVel:Destroy()
        connections.linVel = nil
    end
    if connections.align then
        connections.align:Destroy()
        connections.align = nil
    end
    -- Restore auto rotate
    local char = lp.Character
    if char then
        local hum = char:FindFirstChildOfClass("Humanoid")
        if hum then hum.AutoRotate = true end
    end
end

-- Toggle function (only changes text, background stays black with white outline)
local function setEnabled(state)
    enabled = state
    if enabled then
        startAimbot()
        btn.Text = "MELEE AIMBOT: ON"
    else
        stopAimbot()
        btn.Text = "MELEE AIMBOT: OFF"
    end
end

-- Create UI and button
local guiObj, button = createButton()
gui = guiObj
btn = button

btn.MouseButton1Click:Connect(function()
    setEnabled(not enabled)
end)

-- Reset when player respawns
lp.CharacterAdded:Connect(function()
    if enabled then
        task.wait(0.5)
        stopAimbot()
        startAimbot()
    end
end)
