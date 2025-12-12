local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
 
local player = Players.LocalPlayer
local mouse = player:GetMouse()
local camera = Workspace.CurrentCamera
 
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "BuildUI"
screenGui.Parent = player:WaitForChild("PlayerGui")
 
local function createSlider(name, posY, default)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(0,240,0,30)
    frame.Position = UDim2.new(0,10,0,posY)
    frame.BackgroundColor3 = Color3.fromRGB(40,40,40)
    frame.Parent = screenGui
 
    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(0,60,1,0)
    label.Position = UDim2.new(0,0,0,0)
    label.TextColor3 = Color3.fromRGB(255,255,255)
    label.Text = name
    label.BackgroundTransparency = 1
    label.Font = Enum.Font.SourceSans
    label.TextSize = 18
    label.Parent = frame
 
    local box = Instance.new("TextBox")
    box.Size = UDim2.new(0,160,1,0)
    box.Position = UDim2.new(0,70,0,0)
    box.Text = tostring(default)
    box.ClearTextOnFocus = false
    box.BackgroundColor3 = Color3.fromRGB(60,60,60)
    box.TextColor3 = Color3.fromRGB(255,255,255)
    box.Font = Enum.Font.SourceSans
    box.TextSize = 18
    box.Parent = frame
 
    return box
end
 
local scaleXSlider = createSlider("Scale X", 10, 1)
local scaleYSlider = createSlider("Scale Y", 50, 1)
local scaleZSlider = createSlider("Scale Z", 90, 1)
local rotationSlider = createSlider("Rotation", 130, 0)
local transparencySlider = createSlider("Transparency", 170, 0.5)
local hueSlider = createSlider("Color Hue", 210, 0.5)
local distanceSlider = createSlider("Fallback Dist", 250, 12)
 
local function createButton(text, posY)
    local button = Instance.new("TextButton")
    button.Size = UDim2.new(0, 240, 0, 32)
    button.Position = UDim2.new(0, 10, 0, posY)
    button.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    button.TextColor3 = Color3.fromRGB(255, 255, 255)
    button.Font = Enum.Font.SourceSansBold
    button.TextSize = 20
    button.Text = text
    button.Parent = screenGui
    return button
end
 
local rotateButton = createButton("Rotate", 300)
local undoButton = createButton("Undo", 340)
local resetButton = createButton("Reset", 380)
 
local buildPlaneY = 1
local gridSize = 1
local placedParts = {}
local currentBuildPart = nil
local debounce = false
local lastMousePos = Vector3.new(math.huge, math.huge, math.huge)
local rotationSnap = false
local defaultFallbackDistance = tonumber(distanceSlider.Text) or 12
 
local function tonumberSafe(v, d)
    local n = tonumber(v)
    if n then return n end
    return d
end
 
local BuildPart = {}
BuildPart.__index = BuildPart
 
function BuildPart.new()
    local self = setmetatable({}, BuildPart)
    local p = Instance.new("Part")
    p.Size = Vector3.new(1,1,1)
    p.Anchored = true
    p.CanCollide = false
    p.CanQuery = false
    p.CanTouch = false
    p.Color = Color3.fromHSV(0.5, 0.5, 1)
    p.Transparency = 0.5
    p.Parent = Workspace
    self.preview = p
    self.scale = Vector3.new(1,1,1)
    self.rotationY = 0
    self.hue = 0.5
    self.transparency = 0.5
    return self
end
 
function BuildPart:SetScale(x,y,z)
    self.scale = Vector3.new(x,y,z)
    if self.preview then
        self.preview.Size = self.scale
    end
end
 
function BuildPart:SetRotation(rot)
    self.rotationY = rot % 360
    if self.preview then
        local pos = self.preview.Position
        self.preview.CFrame = CFrame.new(pos) * CFrame.Angles(0, math.rad(self.rotationY), 0)
    end
end
 
function BuildPart:SetColorHue(h)
    self.hue = math.clamp(h, 0, 1)
    if self.preview then
        self.preview.Color = Color3.fromHSV(self.hue, 0.5, 1)
    end
end
 
function BuildPart:SetTransparency(t)
    self.transparency = math.clamp(t, 0, 1)
    if self.preview then
        self.preview.Transparency = self.transparency
    end
end
 
function BuildPart:SnapToGrid(v)
    local x = math.floor(v.X / gridSize + 0.5) * gridSize
    local y = math.floor(v.Y / gridSize + 0.5) * gridSize
    local z = math.floor(v.Z / gridSize + 0.5) * gridSize
    return Vector3.new(x, y, z)
end
 
function BuildPart:UpdatePosition(worldPos, surfaceNormal)
    if self.preview == nil then return end
    if surfaceNormal then
        local snap = self:SnapToGrid(worldPos)
        local cf = CFrame.new(snap, snap + surfaceNormal) * CFrame.Angles(0, math.rad(self.rotationY), 0)
        self.preview.CFrame = cf
    else
        local snap = self:SnapToGrid(worldPos)
        self.preview.CFrame = CFrame.new(snap) * CFrame.Angles(0, math.rad(self.rotationY), 0)
    end
    self.preview.Size = self.scale
    self.preview.Color = Color3.fromHSV(self.hue, 0.5, 1)
    self.preview.Transparency = self.transparency
end
 
function BuildPart:Place()
    if debounce then return end
    debounce = true
    if not self.preview then debounce = false return end
    local placed = Instance.new("Part")
    placed.Size = self.preview.Size
    placed.CFrame = self.preview.CFrame
    placed.Anchored = true
    placed.CanCollide = true
    placed.CanQuery = true
    placed.CanTouch = true
    placed.Color = self.preview.Color
    placed.Parent = Workspace
    table.insert(placedParts, placed)
    debounce = false
end
 
function BuildPart:DestroyPreview()
    if self.preview then
        self.preview:Destroy()
        self.preview = nil
    end
end
 
function BuildPart:ResetSize()
    self:SetScale(1,1,1)
    scaleXSlider.Text = "1"
    scaleYSlider.Text = "1"
    scaleZSlider.Text = "1"
end
 
currentBuildPart = BuildPart.new()
 
local function getScreenRayPlaneIntersection(px, py, planeY)
    local cam = camera
    local unitRay = cam:ScreenPointToRay(px, py)
    local origin = unitRay.Origin
    local dir = unitRay.Direction
    local denom = dir.Y
    if math.abs(denom) < 1e-5 then
        local fallback = origin + dir * defaultFallbackDistance
        return fallback, nil
    end
    local t = (planeY - origin.Y) / denom
    local point = origin + dir * t
    return point, nil
end
 
RunService.RenderStepped:Connect(function()
    if not currentBuildPart then return end
    local cam = camera
    if not cam then return end
 
    defaultFallbackDistance = tonumberSafe(distanceSlider.Text, defaultFallbackDistance)
 
    local rayParams = RaycastParams.new()
    rayParams.FilterDescendantsInstances = { currentBuildPart.preview, player.Character }
    rayParams.FilterType = Enum.RaycastFilterType.Exclude
 
    local origin = mouse.UnitRay.Origin
    local direction = mouse.UnitRay.Direction * 1000
    local rayResult = Workspace:Raycast(origin, direction, rayParams)
 
    local worldPos
    local surfaceNormal
 
    if rayResult then
        worldPos = rayResult.Position
        surfaceNormal = rayResult.Normal
    else
        local mouseLoc = UserInputService:GetMouseLocation()
        local posOnPlane, _ = getScreenRayPlaneIntersection(mouseLoc.X, mouseLoc.Y, buildPlaneY)
        if posOnPlane then
            worldPos = cam.CFrame.Position + cam.CFrame.LookVector * defaultFallbackDistance
        else
            worldPos = cam.CFrame.Position + cam.CFrame.LookVector * defaultFallbackDistance
        end
    end
 
    if (worldPos - lastMousePos).magnitude < 0.001 then
        return
    end
 
    lastMousePos = worldPos
    worldPos = Vector3.new(
        math.clamp(worldPos.X, -1000, 1000),
        math.clamp(worldPos.Y, -1000, 1000),
        math.clamp(worldPos.Z, -1000, 1000)
    )
 
    currentBuildPart:UpdatePosition(worldPos, surfaceNormal)
end)
 
mouse.Button1Down:Connect(function()
    if currentBuildPart then currentBuildPart:Place() end
end)
 
local function applySlidersToPreview()
    if not currentBuildPart then return end
    local sx = tonumberSafe(scaleXSlider.Text, 1)
    local sy = tonumberSafe(scaleYSlider.Text, 1)
    local sz = tonumberSafe(scaleZSlider.Text, 1)
    local rot = tonumberSafe(rotationSlider.Text, 0)
    local trans = tonumberSafe(transparencySlider.Text, 0.5)
    local hue = tonumberSafe(hueSlider.Text, 0.5)
    if rotationSnap then rot = math.floor(rot / 15) * 15 end
    currentBuildPart:SetScale(math.max(0.01, sx), math.max(0.01, sy), math.max(0.01, sz))
    currentBuildPart:SetRotation(rot)
    currentBuildPart:SetTransparency(trans)
    currentBuildPart:SetColorHue(hue)
end
 
scaleXSlider.Changed:Connect(applySlidersToPreview)
scaleYSlider.Changed:Connect(applySlidersToPreview)
scaleZSlider.Changed:Connect(applySlidersToPreview)
rotationSlider.Changed:Connect(applySlidersToPreview)
transparencySlider.Changed:Connect(applySlidersToPreview)
hueSlider.Changed:Connect(applySlidersToPreview)
distanceSlider.Changed:Connect(function() defaultFallbackDistance = tonumberSafe(distanceSlider.Text, defaultFallbackDistance) end)
 
-- Buttons
rotateButton.MouseButton1Click:Connect(function()
    if currentBuildPart then
        local newRot = (currentBuildPart.rotationY + 15) % 360
        currentBuildPart:SetRotation(newRot)
        rotationSlider.Text = tostring(newRot)
    end
end)
 
undoButton.MouseButton1Click:Connect(function()
    if #placedParts > 0 then
        local last = table.remove(placedParts)
        if last then last:Destroy() end
    end
end)
 
resetButton.MouseButton1Click:Connect(function()
    for _, p in pairs(placedParts) do
        if p then p:Destroy() end
    end
    placedParts = {}
    if currentBuildPart then
        currentBuildPart:DestroyPreview()
        currentBuildPart = BuildPart.new()
    end
end)
 
UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.UserInputType == Enum.UserInputType.Keyboard then
        if input.KeyCode == Enum.KeyCode.E then
            if #placedParts > 0 then
                local last = table.remove(placedParts)
                if last then last:Destroy() end
            end
        elseif input.KeyCode == Enum.KeyCode.X then
            for _, p in pairs(placedParts) do
                if p then p:Destroy() end
            end
            placedParts = {}
            if currentBuildPart then
                currentBuildPart:DestroyPreview()
                currentBuildPart = BuildPart.new()
            end
        elseif input.KeyCode == Enum.KeyCode.LeftShift then
            rotationSnap = true
        elseif input.KeyCode == Enum.KeyCode.R then
            if currentBuildPart then
                local newRot = (currentBuildPart.rotationY + 15) % 360
                currentBuildPart:SetRotation(newRot)
                rotationSlider.Text = tostring(newRot)
            end
        elseif input.KeyCode == Enum.KeyCode.Z then
            if currentBuildPart then currentBuildPart:ResetSize() end
        elseif input.KeyCode == Enum.KeyCode.Equals then
            gridSize = gridSize + 1
        elseif input.KeyCode == Enum.KeyCode.Minus then
            gridSize = math.max(1, gridSize - 1)
        end
    end
end)
 
UserInputService.InputEnded:Connect(function(input, processed)
    if input.UserInputType == Enum.UserInputType.Keyboard then
        if input.KeyCode == Enum.KeyCode.LeftShift then
            rotationSnap = false
        end
    end
end)
 
