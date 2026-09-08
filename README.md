if _G.THHGrowersHardUnloaded then
    return
end

if _G.THHGrowersCleanup then
    pcall(_G.THHGrowersCleanup)
end

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local TeleportService = game:GetService("TeleportService")
local RbxAnalyticsService = game:GetService("RbxAnalyticsService")
local Lighting = game:GetService("Lighting")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

local PROJECT_ID = "PD6XH2EGXZXUHGED"
local AUTH_SECRET = "04cce9295d9d2476cb2516b3363693a3c401767b4e75bd01"
local KEY_FLOW = "https://vampauth.com/PD6XH2EGXZXUHGED/flow"

local MM2_GAME_ID = 66654135

local MM2_ICON =
    "rbxthumb://type=GameIcon&id="
    .. tostring(MM2_GAME_ID)
    .. "&w=150&h=150"

local Colors = {
    Background = Color3.fromRGB(24, 25, 29),
    Sidebar = Color3.fromRGB(30, 31, 36),
    Card = Color3.fromRGB(47, 49, 55),
    CardHover = Color3.fromRGB(57, 59, 66),
    Control = Color3.fromRGB(37, 39, 44),

    Accent = Color3.fromRGB(105, 218, 143),
    AccentDark = Color3.fromRGB(42, 77, 56),

    Text = Color3.fromRGB(244, 245, 247),
    SubText = Color3.fromRGB(157, 160, 169),
    Stroke = Color3.fromRGB(87, 90, 99),

    Danger = Color3.fromRGB(229, 79, 79),

    Green = Color3.fromRGB(68, 226, 112),
    Red = Color3.fromRGB(242, 70, 70),
    Blue = Color3.fromRGB(69, 146, 255)
}

local alive = true
local gui
local blur

local connections = {}

local character
local humanoid
local rootPart

local originalWalkSpeed = 16

local movement = {
    speedEnabled = false,
    speed = 32,
    infiniteJump = false,
    noclip = false,
    noclipParts = {},
    spin = false,
    spinSpeed = 300
}

local esp = {
    enabled = false,
    highlights = {}
}

local sheriff = {
    aimBot = false
}

local knife = {
    autoThrow = false,
    throwDelay = 0.2,
    lastThrow = 0
}

local gunPickup = {
    auto = false,
    busy = false,
    checkDelay = 0.08
}

local function track(connection)
    table.insert(connections, connection)
    return connection
end

local function create(className, properties, parent)
    local object = Instance.new(className)

    for property, value in pairs(properties or {}) do
        object[property] = value
    end

    if parent then
        object.Parent = parent
    end

    return object
end

local function addCorner(object, radius)
    return create("UICorner", {
        CornerRadius = UDim.new(0, radius or 9)
    }, object)
end

local function addStroke(object, color, thickness, transparency)
    return create("UIStroke", {
        Color = color or Colors.Stroke,
        Thickness = thickness or 1,
        Transparency = transparency or 0.4
    }, object)
end

local function animateButton(button, normal, hover, pressed)
    if UserInputService.MouseEnabled then
        track(button.MouseEnter:Connect(function()
            TweenService:Create(
                button,
                TweenInfo.new(0.14),
                {
                    BackgroundColor3 = hover
                }
            ):Play()
        end))

        track(button.MouseLeave:Connect(function()
            TweenService:Create(
                button,
                TweenInfo.new(0.14),
                {
                    BackgroundColor3 = normal
                }
            ):Play()
        end))
    end

    track(button.MouseButton1Down:Connect(function()
        TweenService:Create(
            button,
            TweenInfo.new(0.07),
            {
                BackgroundColor3 = pressed
            }
        ):Play()
    end))

    track(button.MouseButton1Up:Connect(function()
        TweenService:Create(
            button,
            TweenInfo.new(0.1),
            {
                BackgroundColor3 = hover
            }
        ):Play()
    end))
end

local function getUIParent()
    if gethui then
        local success, result = pcall(gethui)

        if success and result then
            return result
        end
    end

    return player:WaitForChild("PlayerGui")
end

local function refreshCharacter()
    character = player.Character or player.CharacterAdded:Wait()

    humanoid =
        character:FindFirstChildOfClass("Humanoid")
        or character:WaitForChild("Humanoid")

    rootPart =
        character:FindFirstChild("HumanoidRootPart")
        or character:WaitForChild("HumanoidRootPart")

    originalWalkSpeed = humanoid.WalkSpeed
end

refreshCharacter()

local function restoreNoclip()
    for part, oldValue in pairs(movement.noclipParts) do
        if part and part.Parent then
            pcall(function()
                part.CanCollide = oldValue
            end)
        end
    end

    table.clear(movement.noclipParts)
end

local function clearESP()
    for _, highlight in pairs(esp.highlights) do
        if highlight then
            pcall(function()
                highlight:Destroy()
            end)
        end
    end

    table.clear(esp.highlights)
end

local function cleanup()
    if not alive then
        return
    end

    alive = false

    movement.speedEnabled = false
    movement.infiniteJump = false
    movement.noclip = false
    movement.spin = false

    sheriff.aimBot = false

    knife.autoThrow = false

    gunPickup.auto = false
    gunPickup.busy = false

    esp.enabled = false

    restoreNoclip()
    clearESP()

    if humanoid then
        pcall(function()
            humanoid.WalkSpeed = originalWalkSpeed
        end)
    end

    for _, connection in ipairs(connections) do
        pcall(function()
            connection:Disconnect()
        end)
    end

    table.clear(connections)

    if blur then
        pcall(function()
            blur:Destroy()
        end)
    end

    if gui then
        pcall(function()
            gui:Destroy()
        end)
    end

    if _G.THHGrowersCleanup == cleanup then
        _G.THHGrowersCleanup = nil
    end
end

_G.THHGrowersCleanup = cleanup

track(player.CharacterAdded:Connect(function()
    task.wait(0.35)
    refreshCharacter()
end))

local function makeDraggable(frame, handle)
    handle = handle or frame

    local dragging = false
    local dragInput
    local dragStart
    local startingPosition

    track(handle.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then

            dragging = true
            dragStart = input.Position
            startingPosition = frame.Position

            local connection

            connection = input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false

                    if connection then
                        connection:Disconnect()
                    end
                end
            end)
        end
    end))

    track(handle.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement
            or input.UserInputType == Enum.UserInputType.Touch then

            dragInput = input
        end
    end))

    track(UserInputService.InputChanged:Connect(function(input)
        if dragging and input == dragInput then
            local delta = input.Position - dragStart

            frame.Position = UDim2.new(
                startingPosition.X.Scale,
                startingPosition.X.Offset + delta.X,
                startingPosition.Y.Scale,
                startingPosition.Y.Offset + delta.Y
            )
        end
    end))
end

local function findTool(targetPlayer, wantedName)
    if not targetPlayer then
        return nil
    end

    wantedName = string.lower(wantedName)

    local targetCharacter = targetPlayer.Character

    if targetCharacter then
        for _, object in ipairs(targetCharacter:GetChildren()) do
            if object:IsA("Tool")
                and string.lower(object.Name) == wantedName then

                return object
            end
        end
    end

    local backpack =
        targetPlayer:FindFirstChildOfClass("Backpack")

    if backpack then
        for _, object in ipairs(backpack:GetChildren()) do
            if object:IsA("Tool")
                and string.lower(object.Name) == wantedName then

                return object
            end
        end
    end

    return nil
end

local function hasKnife(targetPlayer)
    return findTool(targetPlayer, "Knife") ~= nil
end

local function hasGun(targetPlayer)
    if findTool(targetPlayer, "Gun") then
        return true
    end

    local containers = {}

    if targetPlayer.Character then
        table.insert(containers, targetPlayer.Character)
    end

    local backpack =
        targetPlayer:FindFirstChildOfClass("Backpack")

    if backpack then
        table.insert(containers, backpack)
    end

    for _, container in ipairs(containers) do
        for _, object in ipairs(container:GetChildren()) do
            if object:IsA("Tool") then
                local lower =
                    string.lower(object.Name)

                if lower:find("gun", 1, true)
                    or lower:find("revolver", 1, true) then

                    return true
                end
            end
        end
    end

    return false
end

local function getRoleColor(targetPlayer)
    if hasKnife(targetPlayer) then
        return Colors.Red
    end

    if hasGun(targetPlayer) then
        return Colors.Blue
    end

    return Colors.Green
end

local function isPlayerAlive(targetPlayer)
    if not targetPlayer
        or targetPlayer == player
        or not targetPlayer.Character then

        return false
    end

    local targetHumanoid =
        targetPlayer.Character:FindFirstChildOfClass("Humanoid")

    local targetRoot =
        targetPlayer.Character:FindFirstChild("HumanoidRootPart")

    local targetHead =
        targetPlayer.Character:FindFirstChild("Head")

    return targetHumanoid
        and targetHumanoid.Health > 0
        and targetRoot
        and targetHead
end

local function getKnifeHolder()
    local bestPlayer
    local bestDistance = math.huge

    for _, targetPlayer in ipairs(Players:GetPlayers()) do
        if targetPlayer ~= player
            and isPlayerAlive(targetPlayer)
            and hasKnife(targetPlayer) then

            local targetRoot =
                targetPlayer.Character.HumanoidRootPart

            local distance = rootPart
                and (targetRoot.Position - rootPart.Position).Magnitude
                or 0

            if distance < bestDistance then
                bestDistance = distance
                bestPlayer = targetPlayer
            end
        end
    end

    return bestPlayer
end

local function updateESP()
    if not esp.enabled then
        clearESP()
        return
    end

    local seen = {}

    for _, targetPlayer in ipairs(Players:GetPlayers()) do
        if targetPlayer ~= player
            and isPlayerAlive(targetPlayer) then

            seen[targetPlayer] = true

            local highlight =
                esp.highlights[targetPlayer]

            if not highlight
                or not highlight.Parent then

                highlight =
                    Instance.new("Highlight")

                highlight.Name =
                    "THH_ESP_" .. targetPlayer.Name

                highlight.DepthMode =
                    Enum.HighlightDepthMode.AlwaysOnTop

                highlight.FillTransparency = 0.77
                highlight.OutlineTransparency = 0

                highlight.Adornee =
                    targetPlayer.Character

                highlight.Parent =
                    targetPlayer.Character

                esp.highlights[targetPlayer] =
                    highlight
            end

            local color =
                getRoleColor(targetPlayer)

            highlight.Adornee =
                targetPlayer.Character

            highlight.FillColor = color
            highlight.OutlineColor = color
            highlight.Enabled = true
        end
    end

    for targetPlayer, highlight in pairs(esp.highlights) do
        if not seen[targetPlayer] then
            if highlight then
                pcall(function()
                    highlight:Destroy()
                end)
            end

            esp.highlights[targetPlayer] = nil
        end
    end
end

local function normalizedName(name)
    return string.lower(tostring(name or ""))
        :gsub("[%s_%-%./]", "")
end

local function findGunDrop()
    local exact =
        workspace:FindFirstChild("GunDrop", true)

    if exact then
        return exact
    end

    local names = {
        gundrop = true,
        droppedgun = true,
        dropgun = true,
        sheriffgundrop = true,
        droppedrevolver = true
    }

    for _, object in ipairs(workspace:GetDescendants()) do
        if names[normalizedName(object.Name)] then
            return object
        end
    end

    return nil
end

local function getGunPart(object)
    if not object then
        return nil
    end

    if object:IsA("BasePart") then
        return object
    end

    if object:IsA("Tool") then
        local handle =
            object:FindFirstChild("Handle")

        if handle and handle:IsA("BasePart") then
            return handle
        end
    end

    if object:IsA("Model") then
        if object.PrimaryPart then
            return object.PrimaryPart
        end

        return object:FindFirstChildWhichIsA(
            "BasePart",
            true
        )
    end

    return object:FindFirstChildWhichIsA(
        "BasePart",
        true
    )
end

local function interactWithGun(object, part)
    if not object or not part then
        return
    end

    for _, descendant in ipairs(object:GetDescendants()) do
        if descendant:IsA("ProximityPrompt")
            and fireproximityprompt then

            pcall(function()
                fireproximityprompt(descendant, 0)
            end)
        end

        if descendant:IsA("ClickDetector")
            and fireclickdetector then

            pcall(function()
                fireclickdetector(descendant)
            end)
        end
    end

    if firetouchinterest
        and rootPart
        and part:IsA("BasePart") then

        pcall(function()
            firetouchinterest(rootPart, part, 0)

            RunService.Heartbeat:Wait()

            firetouchinterest(rootPart, part, 1)
        end)
    end
end

local function alreadyHasGun()
    return hasGun(player)
end

local function pickupGun()
    if gunPickup.busy
        or not alive
        or not rootPart
        or not humanoid then

        return false
    end

    if alreadyHasGun() then
        return false
    end

    local gunObject =
        findGunDrop()

    if not gunObject then
        return false
    end

    local gunPart =
        getGunPart(gunObject)

    if not gunPart then
        return false
    end

    gunPickup.busy = true

    local returnCFrame =
        rootPart.CFrame

    local oldVelocity =
        rootPart.AssemblyLinearVelocity

    local oldAngular =
        rootPart.AssemblyAngularVelocity

    local success = pcall(function()
        rootPart.AssemblyLinearVelocity =
            Vector3.zero

        rootPart.AssemblyAngularVelocity =
            Vector3.zero

        rootPart.CFrame =
            gunPart.CFrame
            * CFrame.new(0, 1, 0)

        RunService.Heartbeat:Wait()

        interactWithGun(
            gunObject,
            gunPart
        )

        task.wait(0.04)

        if gunObject.Parent
            and gunPart.Parent then

            rootPart.CFrame =
                gunPart.CFrame
                * CFrame.new(0, 0.15, 0)

            interactWithGun(
                gunObject,
                gunPart
            )
        end

        task.wait(0.04)

        if rootPart
            and rootPart.Parent then

            rootPart.CFrame =
                returnCFrame

            rootPart.AssemblyLinearVelocity =
                oldVelocity

            rootPart.AssemblyAngularVelocity =
                oldAngular
        end
    end)

    gunPickup.busy = false

    return success
end

local function throwKnifeOnce()
    if not character
        or not humanoid then

        return false
    end

    local knifeTool =
        findTool(player, "Knife")

    if not knifeTool then
        return false
    end

    if knifeTool.Parent ~= character then
        pcall(function()
            humanoid:EquipTool(knifeTool)
        end)

        RunService.Heartbeat:Wait()
    end

    if knifeTool.Parent == character then
        return pcall(function()
            knifeTool:Activate()
        end)
    end

    return false
end

track(UserInputService.JumpRequest:Connect(function()
    if movement.infiniteJump
        and humanoid then

        humanoid:ChangeState(
            Enum.HumanoidStateType.Jumping
        )
    end
end))

track(RunService.Stepped:Connect(function()
    if not alive
        or not character then

        return
    end

    if movement.noclip then
        for _, object in ipairs(character:GetDescendants()) do
            if object:IsA("BasePart") then
                if movement.noclipParts[object] == nil then
                    movement.noclipParts[object] =
                        object.CanCollide
                end

                object.CanCollide = false
            end
        end
    elseif next(movement.noclipParts) then
        restoreNoclip()
    end
end))

track(RunService.Heartbeat:Connect(function(deltaTime)
    if not humanoid
        or not rootPart then

        return
    end

    if movement.speedEnabled then
        humanoid.WalkSpeed =
            movement.speed
    elseif humanoid.WalkSpeed ~= originalWalkSpeed then
        humanoid.WalkSpeed =
            originalWalkSpeed
    end

    if movement.spin then
        rootPart.CFrame =
            rootPart.CFrame
            * CFrame.Angles(
                0,
                math.rad(
                    movement.spinSpeed
                    * deltaTime
                ),
                0
            )
    end

    if knife.autoThrow then
        if os.clock() - knife.lastThrow
            >= knife.throwDelay then

            knife.lastThrow = os.clock()
            throwKnifeOnce()
        end
    end
end))

track(RunService.RenderStepped:Connect(function()
    if not alive
        or not sheriff.aimBot then

        return
    end

    local target =
        getKnifeHolder()

    if not target
        or not target.Character then

        return
    end

    local head =
        target.Character:FindFirstChild("Head")

    if not head then
        return
    end

    camera.CFrame =
        CFrame.lookAt(
            camera.CFrame.Position,
            head.Position
        )
end))

task.spawn(function()
    while alive do
        if esp.enabled then
            pcall(updateESP)
        end

        task.wait(0.1)
    end
end)

task.spawn(function()
    while alive do
        if gunPickup.auto
            and not alreadyHasGun()
            and findGunDrop() then

            pcall(pickupGun)
        end

        task.wait(gunPickup.checkDelay)
    end
end)

gui = create("ScreenGui", {
    Name = "THH_HUB",
    ResetOnSpawn = false,
    IgnoreGuiInset = true,
    DisplayOrder = 999999,
    ZIndexBehavior = Enum.ZIndexBehavior.Sibling
}, getUIParent())

blur = create("BlurEffect", {
    Name = "THH_HUB_Blur",
    Size = 20
}, Lighting)

local function loadVampauth()
    local success, result =
        pcall(function()
            if not loadstring then
                error("loadstring unavailable")
            end

            local source =
                game:HttpGet(
                    "https://vampauth.com/client/vampauth.lua"
                )

            local loader =
                loadstring(source)

            if not loader then
                error("Vampauth failed")
            end

            return loader()
        end)

    if success
        and type(result) == "table" then

        return result
    end

    return nil
end

local vampauthClient

local function getHWID()
    local success, result =
        pcall(function()
            return RbxAnalyticsService:GetClientId()
        end)

    if success then
        return result
    end

    return tostring(player.UserId)
end

local function validateKey(key)
    key =
        tostring(key or "")
        :gsub("^%s+", "")
        :gsub("%s+$", "")

    if key == "" then
        return false, "enter a key first"
    end

    if not vampauthClient then
        local Vampauth =
            loadVampauth()

        if not Vampauth then
            return false, "could not load Vampauth"
        end

        local success, result =
            pcall(function()
                return Vampauth.new({
                    projectId = PROJECT_ID,
                    authSecret = AUTH_SECRET,
                    hwid = getHWID(),
                    debug = false
                })
            end)

        if not success or not result then
            return false,
                "authentication failed"
        end

        vampauthClient = result
    end

    local success, valid, data =
        pcall(function()
            return vampauthClient:Check(key)
        end)

    if not success then
        return false,
            "authentication request failed"
    end

    if valid then
        return true, data
    end

    local reason =
        tostring(data or "invalid key")

    local messages = {
        KEY_NOT_FOUND = "invalid key",
        KEY_EXPIRED = "key expired",
        KEY_BANNED = "key banned",

        FINGERPRINT_MISMATCH =
            "key is linked to another device",

        RATE_LIMITED =
            "too many attempts",

        BAD_SIGNATURE =
            "authentication error",

        UNAUTHORIZED =
            "authentication error"
    }

    return false,
        messages[reason] or reason
end

local buildChooser
local buildMainMenu

local keyOverlay = create("Frame", {
    Name = "KeyOverlay",
    Size = UDim2.fromScale(1, 1),
    BackgroundColor3 = Color3.fromRGB(3, 4, 6),
    BackgroundTransparency = 0.03,
    BorderSizePixel = 0
}, gui)

local backgroundGradient =
    create("UIGradient", {
        Color = ColorSequence.new({
            ColorSequenceKeypoint.new(
                0,
                Color3.fromRGB(7, 8, 11)
            ),
            ColorSequenceKeypoint.new(
                0.5,
                Color3.fromRGB(23, 25, 30)
            ),
            ColorSequenceKeypoint.new(
                1,
                Color3.fromRGB(7, 8, 11)
            )
        }),

        Rotation = -25
    }, keyOverlay)

task.spawn(function()
    while alive
        and keyOverlay.Parent do

        TweenService:Create(
            backgroundGradient,
            TweenInfo.new(
                5,
                Enum.EasingStyle.Sine,
                Enum.EasingDirection.InOut
            ),
            {
                Rotation = 25
            }
        ):Play()

        task.wait(5)

        if not keyOverlay.Parent then
            break
        end

        TweenService:Create(
            backgroundGradient,
            TweenInfo.new(
                5,
                Enum.EasingStyle.Sine,
                Enum.EasingDirection.InOut
            ),
            {
                Rotation = -25
            }
        ):Play()

        task.wait(5)
    end
end)

local keyWindow = create("Frame", {
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(0.5, 0.5),
    Size = UDim2.new(0.88, 0, 0, 350),

    BackgroundColor3 =
        Color3.fromRGB(52, 54, 60),

    BackgroundTransparency = 0.2,

    BorderSizePixel = 0,
    Active = true
}, keyOverlay)

create("UISizeConstraint", {
    MinSize = Vector2.new(300, 340),
    MaxSize = Vector2.new(445, 350)
}, keyWindow)

addCorner(keyWindow, 17)

addStroke(
    keyWindow,
    Color3.fromRGB(144, 147, 156),
    1,
    0.52
)

local keyHeader = create("Frame", {
    Size = UDim2.new(1, 0, 0, 76),
    BackgroundTransparency = 1,
    Active = true
}, keyWindow)

makeDraggable(keyWindow, keyHeader)

local iconBack = create("Frame", {
    Position = UDim2.new(0, 17, 0, 14),
    Size = UDim2.fromOffset(48, 48),

    BackgroundColor3 =
        Color3.fromRGB(78, 81, 89),

    BackgroundTransparency = 0.15,

    BorderSizePixel = 0
}, keyHeader)

addCorner(iconBack, 13)

local keyIcon = create("ImageLabel", {
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(0.5, 0.5),
    Size = UDim2.fromOffset(42, 42),

    BackgroundTransparency = 1,

    Image = MM2_ICON
}, iconBack)

addCorner(keyIcon, 11)

task.spawn(function()
    while alive
        and iconBack.Parent do

        TweenService:Create(
            iconBack,
            TweenInfo.new(
                1.8,
                Enum.EasingStyle.Sine,
                Enum.EasingDirection.InOut
            ),
            {
                BackgroundTransparency = 0.03
            }
        ):Play()

        task.wait(1.8)

        if not iconBack.Parent then
            break
        end

        TweenService:Create(
            iconBack,
            TweenInfo.new(
                1.8,
                Enum.EasingStyle.Sine,
                Enum.EasingDirection.InOut
            ),
            {
                BackgroundTransparency = 0.18
            }
        ):Play()

        task.wait(1.8)
    end
end)

create("TextLabel", {
    Position = UDim2.new(0, 78, 0, 15),
    Size = UDim2.new(1, -98, 0, 23),

    BackgroundTransparency = 1,

    Text = "THH HUB",

    TextColor3 = Colors.Text,

    TextSize = 20,

    Font = Enum.Font.GothamBold,

    TextXAlignment =
        Enum.TextXAlignment.Left
}, keyHeader)

create("TextLabel", {
    Position = UDim2.new(0, 78, 0, 40),
    Size = UDim2.new(1, -98, 0, 18),

    BackgroundTransparency = 1,

    Text = "Authentication",

    TextColor3 = Colors.SubText,

    TextSize = 12,

    Font = Enum.Font.Gotham,

    TextXAlignment =
        Enum.TextXAlignment.Left
}, keyHeader)

local keyBox = create("TextBox", {
    Position = UDim2.new(0, 18, 0, 92),
    Size = UDim2.new(1, -36, 0, 48),

    BackgroundColor3 =
        Color3.fromRGB(34, 36, 41),

    BackgroundTransparency = 0.04,

    BorderSizePixel = 0,

    Text = "",

    PlaceholderText = "Enter access key",

    PlaceholderColor3 =
        Colors.SubText,

    TextColor3 =
        Colors.Text,

    TextSize = 14,

    Font =
        Enum.Font.Gotham,

    ClearTextOnFocus = false,

    TextXAlignment =
        Enum.TextXAlignment.Left
}, keyWindow)

addCorner(keyBox, 11)
addStroke(keyBox)

create("UIPadding", {
    PaddingLeft = UDim.new(0, 14),
    PaddingRight = UDim.new(0, 14)
}, keyBox)

local continueButton = create("TextButton", {
    Position = UDim2.new(0, 18, 0, 154),
    Size = UDim2.new(1, -36, 0, 46),

    BackgroundColor3 =
        Colors.Accent,

    BorderSizePixel = 0,

    Text = "Continue",

    TextColor3 =
        Color3.fromRGB(14, 20, 16),

    TextSize = 14,

    Font =
        Enum.Font.GothamBold,

    AutoButtonColor = false
}, keyWindow)

addCorner(continueButton, 11)

animateButton(
    continueButton,
    Colors.Accent,
    Color3.fromRGB(124, 231, 158),
    Color3.fromRGB(81, 188, 117)
)

local keyAccessCard = create("Frame", {
    Position = UDim2.new(0, 18, 0, 214),
    Size = UDim2.new(1, -36, 0, 58),

    BackgroundColor3 =
        Color3.fromRGB(68, 71, 78),

    BackgroundTransparency = 0.2,

    BorderSizePixel = 0
}, keyWindow)

addCorner(keyAccessCard, 12)

addStroke(
    keyAccessCard,
    Color3.fromRGB(125, 129, 138),
    1,
    0.5
)

local cardGradient =
    create("UIGradient", {
        Color =
            ColorSequence.new({
                ColorSequenceKeypoint.new(
                    0,
                    Color3.fromRGB(56, 59, 66)
                ),
                ColorSequenceKeypoint.new(
                    0.5,
                    Color3.fromRGB(81, 84, 92)
                ),
                ColorSequenceKeypoint.new(
                    1,
                    Color3.fromRGB(56, 59, 66)
                )
            }),

        Rotation = 0
    }, keyAccessCard)

task.spawn(function()
    while alive
        and keyAccessCard.Parent do

        TweenService:Create(
            cardGradient,
            TweenInfo.new(
                2.5,
                Enum.EasingStyle.Sine,
                Enum.EasingDirection.InOut
            ),
            {
                Rotation = 15
            }
        ):Play()

        task.wait(2.5)

        if not keyAccessCard.Parent then
            break
        end

        TweenService:Create(
            cardGradient,
            TweenInfo.new(
                2.5,
                Enum.EasingStyle.Sine,
                Enum.EasingDirection.InOut
            ),
            {
                Rotation = -15
            }
        ):Play()

        task.wait(2.5)
    end
end)

local getKeyButton = create("TextButton", {
    AnchorPoint = Vector2.new(0.5, 0.5),
    Position = UDim2.fromScale(0.5, 0.5),

    Size = UDim2.new(1, -12, 1, -12),

    BackgroundColor3 =
        Color3.fromRGB(45, 47, 53),

    BackgroundTransparency = 0.08,

    BorderSizePixel = 0,

    Text = "Get Access Key",

    TextColor3 =
        Colors.Text,

    TextSize = 13,

    Font =
        Enum.Font.GothamBold,

    AutoButtonColor = false
}, keyAccessCard)

addCorner(getKeyButton, 9)

animateButton(
    getKeyButton,
    Color3.fromRGB(45, 47, 53),
    Color3.fromRGB(62, 65, 72),
    Color3.fromRGB(35, 37, 42)
)

local keyStatus = create("TextLabel", {
    Position = UDim2.new(0, 18, 0, 286),
    Size = UDim2.new(1, -36, 0, 40),

    BackgroundTransparency = 1,

    Text = "paste your Vampauth key",

    TextColor3 =
        Colors.SubText,

    TextSize = 12,

    Font =
        Enum.Font.Gotham,

    TextWrapped = true
}, keyWindow)

local keyBusy = false

track(getKeyButton.MouseButton1Click:Connect(function()
    local copied = false

    if setclipboard then
        copied =
            pcall(function()
                setclipboard(KEY_FLOW)
            end)

    elseif toclipboard then
        copied =
            pcall(function()
                toclipboard(KEY_FLOW)
            end)
    end

    if copied then
        keyStatus.Text =
            "access key link copied"

        keyStatus.TextColor3 =
            Colors.Accent
    else
        keyStatus.Text =
            KEY_FLOW

        keyStatus.TextColor3 =
            Colors.Text
    end
end))

track(continueButton.MouseButton1Click:Connect(function()
    if keyBusy then
        return
    end

    keyBusy = true

    continueButton.Text =
        "Checking..."

    keyStatus.Text =
        "checking key..."

    keyStatus.TextColor3 =
        Colors.SubText

    task.spawn(function()
        local valid, result =
            validateKey(keyBox.Text)

        if not alive then
            return
        end

        if valid then
            keyStatus.Text =
                "key accepted"

            keyStatus.TextColor3 =
                Colors.Accent

            task.wait(0.25)

            keyOverlay:Destroy()

            buildChooser()
        else
            keyStatus.Text =
                tostring(result)

            keyStatus.TextColor3 =
                Colors.Danger

            continueButton.Text =
                "Continue"

            keyBusy = false
        end
    end)
end))

buildChooser = function()
    local overlay = create("Frame", {
        Name = "DeviceChooser",

        Size = UDim2.fromScale(1, 1),

        BackgroundColor3 =
            Color3.fromRGB(5, 6, 8),

        BackgroundTransparency = 0.22,

        BorderSizePixel = 0
    }, gui)

    local panel = create("Frame", {
        AnchorPoint = Vector2.new(0.5, 0.5),

        Position = UDim2.fromScale(0.5, 0.5),

        Size = UDim2.new(0.82, 0, 0, 230),

        BackgroundColor3 =
            Color3.fromRGB(77, 79, 84),

        BackgroundTransparency = 0.22,

        BorderSizePixel = 0,

        Active = true
    }, overlay)

    create("UISizeConstraint", {
        MinSize = Vector2.new(300, 210),
        MaxSize = Vector2.new(470, 230)
    }, panel)

    addCorner(panel, 14)

    addStroke(
        panel,
        Color3.fromRGB(160, 163, 170),
        1,
        0.45
    )

    makeDraggable(panel)

    create("TextLabel", {
        Position = UDim2.new(0, 20, 0, 20),

        Size = UDim2.new(1, -40, 0, 26),

        BackgroundTransparency = 1,

        Text = "what are you on?",

        TextColor3 =
            Color3.fromRGB(245, 245, 247),

        TextSize = 20,

        Font = Enum.Font.GothamBold
    }, panel)

    create("TextLabel", {
        Position = UDim2.new(0, 20, 0, 47),

        Size = UDim2.new(1, -40, 0, 20),

        BackgroundTransparency = 1,

        Text = "pick pc or phone",

        TextColor3 =
            Color3.fromRGB(197, 199, 205),

        TextSize = 13,

        Font = Enum.Font.Gotham
    }, panel)

    local pcButton = create("TextButton", {
        Position = UDim2.new(0.05, 0, 0, 85),

        Size = UDim2.new(0.425, 0, 0, 120),

        BackgroundColor3 =
            Color3.fromRGB(41, 43, 48),

        BackgroundTransparency = 0.16,

        BorderSizePixel = 0,

        Text = "",

        AutoButtonColor = false
    }, panel)

    addCorner(pcButton, 12)

    addStroke(
        pcButton,
        Color3.fromRGB(137, 140, 147),
        1,
        0.5
    )

    local monitor = create("Frame", {
        AnchorPoint = Vector2.new(0.5, 0),

        Position = UDim2.new(0.5, 0, 0, 22),

        Size = UDim2.fromOffset(60, 39),

        BackgroundTransparency = 1,

        BorderSizePixel = 0
    }, pcButton)

    addStroke(
        monitor,
        Color3.fromRGB(230, 230, 233),
        3
    )

    addCorner(monitor, 5)

    create("Frame", {
        AnchorPoint = Vector2.new(0.5, 0),

        Position = UDim2.new(0.5, 0, 0, 42),

        Size = UDim2.fromOffset(4, 13),

        BackgroundColor3 =
            Color3.fromRGB(230, 230, 233),

        BorderSizePixel = 0
    }, pcButton)

    create("Frame", {
        AnchorPoint = Vector2.new(0.5, 0),

        Position = UDim2.new(0.5, 0, 0, 54),

        Size = UDim2.fromOffset(28, 3),

        BackgroundColor3 =
            Color3.fromRGB(230, 230, 233),

        BorderSizePixel = 0
    }, pcButton)

    create("TextLabel", {
        Position = UDim2.new(0, 0, 1, -38),

        Size = UDim2.new(1, 0, 0, 24),

        BackgroundTransparency = 1,

        Text = "PC",

        TextColor3 =
            Color3.fromRGB(245, 245, 247),

        TextSize = 16,

        Font = Enum.Font.GothamBold
    }, pcButton)

    local phoneButton = create("TextButton", {
        Position = UDim2.new(0.525, 0, 0, 85),

        Size = UDim2.new(0.425, 0, 0, 120),

        BackgroundColor3 =
            Color3.fromRGB(41, 43, 48),

        BackgroundTransparency = 0.16,

        BorderSizePixel = 0,

        Text = "",

        AutoButtonColor = false
    }, panel)

    addCorner(phoneButton, 12)

    addStroke(
        phoneButton,
        Color3.fromRGB(137, 140, 147),
        1,
        0.5
    )

    local phoneDrawing = create("Frame", {
        AnchorPoint = Vector2.new(0.5, 0),

        Position = UDim2.new(0.5, 0, 0, 15),

        Size = UDim2.fromOffset(34, 54),

        BackgroundTransparency = 1,

        BorderSizePixel = 0
    }, phoneButton)

    addCorner(phoneDrawing, 6)

    addStroke(
        phoneDrawing,
        Color3.fromRGB(230, 230, 233),
        3
    )

    create("Frame", {
        AnchorPoint = Vector2.new(0.5, 0),

        Position = UDim2.new(0.5, 0, 0, 5),

        Size = UDim2.fromOffset(10, 2),

        BackgroundColor3 =
            Color3.fromRGB(230, 230, 233),

        BorderSizePixel = 0
    }, phoneDrawing)

    create("TextLabel", {
        Position = UDim2.new(0, 0, 1, -38),

        Size = UDim2.new(1, 0, 0, 24),

        BackgroundTransparency = 1,

        Text = "PHONE",

        TextColor3 =
            Color3.fromRGB(245, 245, 247),

        TextSize = 16,

        Font = Enum.Font.GothamBold
    }, phoneButton)

    animateButton(
        pcButton,
        Color3.fromRGB(41, 43, 48),
        Color3.fromRGB(55, 57, 63),
        Color3.fromRGB(32, 34, 38)
    )

    animateButton(
        phoneButton,
        Color3.fromRGB(41, 43, 48),
        Color3.fromRGB(55, 57, 63),
        Color3.fromRGB(32, 34, 38)
    )

    track(pcButton.MouseButton1Click:Connect(function()
        overlay:Destroy()

        if blur then
            blur:Destroy()
            blur = nil
        end

        buildMainMenu("PC")
    end))

    track(phoneButton.MouseButton1Click:Connect(function()
        overlay:Destroy()

        if blur then
            blur:Destroy()
            blur = nil
        end

        buildMainMenu("Phone")
    end))
end

buildMainMenu = function(deviceMode)
    local isPhone =
        deviceMode == "Phone"

    local menu = create("Frame", {
        AnchorPoint = Vector2.new(0.5, 0.5),

        Position = UDim2.fromScale(0.5, 0.5),

        BackgroundColor3 =
            Colors.Background,

        BackgroundTransparency = 0.1,

        BorderSizePixel = 0,

        ClipsDescendants = true,

        Active = true
    }, gui)

    addCorner(menu, 15)

    addStroke(
        menu,
        Color3.fromRGB(115, 118, 128),
        1,
        0.55
    )

    local function updateSize()
        local viewport =
            camera.ViewportSize

        if isPhone then
            menu.Size =
                UDim2.fromOffset(
                    math.max(
                        300,
                        math.min(
                            viewport.X - 16,
                            620
                        )
                    ),
                    math.max(
                        390,
                        math.min(
                            viewport.Y - 24,
                            710
                        )
                    )
                )
        else
            menu.Size =
                UDim2.fromOffset(
                    math.max(
                        520,
                        math.min(
                            700,
                            viewport.X - 30
                        )
                    ),
                    math.max(
                        360,
                        math.min(
                            455,
                            viewport.Y - 30
                        )
                    )
                )
        end
    end

    updateSize()

    track(
        camera:GetPropertyChangedSignal(
            "ViewportSize"
        ):Connect(updateSize)
    )

    local headerHeight =
        isPhone and 62 or 58

    local sidebarWidth =
        isPhone and 0 or 160

    local phoneNavHeight =
        isPhone and 58 or 0

    local header = create("Frame", {
        Size =
            UDim2.new(
                1,
                0,
                0,
                headerHeight
            ),

        BackgroundColor3 =
            Colors.Sidebar,

        BackgroundTransparency = 0.12,

        BorderSizePixel = 0,

        Active = true
    }, menu)

    makeDraggable(menu, header)

    local icon = create("ImageLabel", {
        Position =
            UDim2.new(
                0,
                11,
                0.5,
                -19
            ),

        Size =
            UDim2.fromOffset(
                38,
                38
            ),

        BackgroundColor3 =
            Colors.Control,

        BackgroundTransparency = 0.1,

        BorderSizePixel = 0,

        Image = MM2_ICON
    }, header)

    addCorner(icon, 9)

    create("TextLabel", {
        Position =
            UDim2.new(
                0,
                60,
                0,
                9
            ),

        Size =
            UDim2.new(
                0,
                150,
                0,
                22
            ),

        BackgroundTransparency = 1,

        Text = "THH HUB",

        TextColor3 =
            Colors.Text,

        TextSize = 18,

        Font =
            Enum.Font.GothamBold,

        TextXAlignment =
            Enum.TextXAlignment.Left
    }, header)

    create("TextLabel", {
        Position =
            UDim2.new(
                0,
                60,
                0,
                31
            ),

        Size =
            UDim2.new(
                0,
                100,
                0,
                17
            ),

        BackgroundTransparency = 1,

        Text = "MM2",

        TextColor3 =
            Colors.SubText,

        TextSize = 10,

        Font =
            Enum.Font.Gotham,

        TextXAlignment =
            Enum.TextXAlignment.Left
    }, header)

    local minimizeButton =
        create("TextButton", {
            AnchorPoint =
                Vector2.new(1, 0.5),

            Position =
                UDim2.new(
                    1,
                    -45,
                    0.5,
                    0
                ),

            Size =
                UDim2.fromOffset(
                    29,
                    29
                ),

            BackgroundColor3 =
                Colors.Control,

            BackgroundTransparency =
                0.08,

            BorderSizePixel = 0,

            Text = "—",

            TextColor3 =
                Colors.SubText,

            TextSize = 17,

            Font =
                Enum.Font.GothamBold,

            AutoButtonColor = false
        }, header)

    addCorner(minimizeButton, 8)

    local closeButton =
        create("TextButton", {
            AnchorPoint =
                Vector2.new(1, 0.5),

            Position =
                UDim2.new(
                    1,
                    -9,
                    0.5,
                    0
                ),

            Size =
                UDim2.fromOffset(
                    29,
                    29
                ),

            BackgroundColor3 =
                Colors.Control,

            BackgroundTransparency =
                0.08,

            BorderSizePixel = 0,

            Text = "×",

            TextColor3 =
                Colors.Danger,

            TextSize = 20,

            Font =
                Enum.Font.Gotham,

            AutoButtonColor = false
        }, header)

    addCorner(closeButton, 8)

    local navHolder

    if isPhone then
        navHolder =
            create("ScrollingFrame", {
                Position =
                    UDim2.new(
                        0,
                        0,
                        0,
                        headerHeight
                    ),

                Size =
                    UDim2.new(
                        1,
                        0,
                        0,
                        phoneNavHeight
                    ),

                BackgroundColor3 =
                    Colors.Sidebar,

                BackgroundTransparency =
                    0.12,

                BorderSizePixel = 0,

                CanvasSize =
                    UDim2.fromOffset(
                        0,
                        0
                    ),

                AutomaticCanvasSize =
                    Enum.AutomaticSize.X,

                ScrollingDirection =
                    Enum.ScrollingDirection.X,

                ScrollBarThickness = 4,

                ScrollBarImageColor3 =
                    Colors.Accent
            }, menu)

        create("UIListLayout", {
            FillDirection =
                Enum.FillDirection.Horizontal,

            SortOrder =
                Enum.SortOrder.LayoutOrder,

            Padding =
                UDim.new(0, 7),

            VerticalAlignment =
                Enum.VerticalAlignment.Center
        }, navHolder)

        create("UIPadding", {
            PaddingLeft =
                UDim.new(0, 9),

            PaddingRight =
                UDim.new(0, 9)
        }, navHolder)
    else
        navHolder =
            create("ScrollingFrame", {
                Position =
                    UDim2.new(
                        0,
                        0,
                        0,
                        headerHeight
                    ),

                Size =
                    UDim2.new(
                        0,
                        sidebarWidth,
                        1,
                        -headerHeight
                    ),

                BackgroundColor3 =
                    Colors.Sidebar,

                BackgroundTransparency =
                    0.12,

                BorderSizePixel = 0,

                CanvasSize =
                    UDim2.fromOffset(
                        0,
                        0
                    ),

                AutomaticCanvasSize =
                    Enum.AutomaticSize.Y,

                ScrollBarThickness = 2,

                ScrollBarImageColor3 =
                    Colors.Accent
            }, menu)

        create("UIListLayout", {
            SortOrder =
                Enum.SortOrder.LayoutOrder,

            Padding =
                UDim.new(0, 6),

            HorizontalAlignment =
                Enum.HorizontalAlignment.Center
        }, navHolder)

        create("UIPadding", {
            PaddingTop =
                UDim.new(0, 11),

            PaddingBottom =
                UDim.new(0, 11)
        }, navHolder)
    end

    local pagesHolder =
        create("Frame", {
            Position = isPhone
                and UDim2.new(
                    0,
                    0,
                    0,
                    headerHeight
                    + phoneNavHeight
                )
                or UDim2.new(
                    0,
                    sidebarWidth,
                    0,
                    headerHeight
                ),

            Size = isPhone
                and UDim2.new(
                    1,
                    0,
                    1,
                    -(
                        headerHeight
                        + phoneNavHeight
                    )
                )
                or UDim2.new(
                    1,
                    -sidebarWidth,
                    1,
                    -headerHeight
                ),

            BackgroundTransparency = 1,

            ClipsDescendants = true
        }, menu)

    local notificationHolder =
        create("Frame", {
            AnchorPoint =
                Vector2.new(1, 0),

            Position =
                UDim2.new(
                    1,
                    -12,
                    0,
                    12
                ),

            Size =
                UDim2.fromOffset(
                    isPhone
                        and 250
                        or 280,
                    310
                ),

            BackgroundTransparency = 1,

            ZIndex = 500
        }, gui)

    create("UIListLayout", {
        Padding =
            UDim.new(0, 7),

        HorizontalAlignment =
            Enum.HorizontalAlignment.Right,

        SortOrder =
            Enum.SortOrder.LayoutOrder
    }, notificationHolder)

    local function notifications(title, message)
        local card =
            create("Frame", {
                Size =
                    UDim2.new(
                        1,
                        0,
                        0,
                        64
                    ),

                BackgroundColor3 =
                    Colors.Card,

                BackgroundTransparency =
                    0.1,

                BorderSizePixel = 0,

                ZIndex = 501
            }, notificationHolder)

        addCorner(card, 10)
        addStroke(card)

        create("TextLabel", {
            Position =
                UDim2.new(
                    0,
                    13,
                    0,
                    9
                ),

            Size =
                UDim2.new(
                    1,
                    -26,
                    0,
                    18
                ),

            BackgroundTransparency = 1,

            Text = title,

            TextColor3 =
                Colors.Text,

            TextSize = 12,

            Font =
                Enum.Font.GothamBold,

            TextXAlignment =
                Enum.TextXAlignment.Left,

            ZIndex = 502
        }, card)

        create("TextLabel", {
            Position =
                UDim2.new(
                    0,
                    13,
                    0,
                    30
                ),

            Size =
                UDim2.new(
                    1,
                    -26,
                    0,
                    24
                ),

            BackgroundTransparency = 1,

            Text = message,

            TextColor3 =
                Colors.SubText,

            TextSize = 10,

            Font =
                Enum.Font.Gotham,

            TextXAlignment =
                Enum.TextXAlignment.Left,

            ZIndex = 502
        }, card)

        task.delay(2.4, function()
            if card and card.Parent then
                card:Destroy()
            end
        end)
    end

    local pages = {}
    local navButtons = {}
    local navOrder = 0

    local function createPage(name)
        local page =
            create("ScrollingFrame", {
                Size =
                    UDim2.fromScale(
                        1,
                        1
                    ),

                BackgroundTransparency = 1,

                BorderSizePixel = 0,

                Visible = false,

                CanvasSize =
                    UDim2.fromOffset(
                        0,
                        0
                    ),

                AutomaticCanvasSize =
                    Enum.AutomaticSize.Y,

                ScrollBarThickness =
                    isPhone and 7 or 4,

                ScrollBarImageColor3 =
                    Colors.Accent
            }, pagesHolder)

        create("UIListLayout", {
            SortOrder =
                Enum.SortOrder.LayoutOrder,

            Padding =
                UDim.new(
                    0,
                    isPhone
                        and 12
                        or 10
                )
        }, page)

        create("UIPadding", {
            PaddingLeft =
                UDim.new(
                    0,
                    isPhone
                        and 12
                        or 15
                ),

            PaddingRight =
                UDim.new(
                    0,
                    isPhone
                        and 12
                        or 15
                ),

            PaddingTop =
                UDim.new(
                    0,
                    isPhone
                        and 12
                        or 15
                ),

            PaddingBottom =
                UDim.new(0, 18)
        }, page)

        pages[name] = page

        return page
    end

    local function showPage(name)
        for pageName, page in pairs(pages) do
            page.Visible =
                pageName == name
        end

        for buttonName, button in pairs(navButtons) do
            if buttonName == name then
                button.BackgroundColor3 =
                    Colors.AccentDark

                button.TextColor3 =
                    Colors.Accent
            else
                button.BackgroundColor3 =
                    Colors.Control

                button.TextColor3 =
                    Colors.SubText
            end
        end
    end

    local function createNav(name)
        navOrder += 1

        local button =
            create("TextButton", {
                LayoutOrder = navOrder,

                Size = isPhone
                    and UDim2.fromOffset(
                        105,
                        42
                    )
                    or UDim2.fromOffset(
                        135,
                        39
                    ),

                BackgroundColor3 =
                    Colors.Control,

                BackgroundTransparency =
                    0.08,

                BorderSizePixel = 0,

                Text = name,

                TextColor3 =
                    Colors.SubText,

                TextSize =
                    isPhone and 12 or 11,

                Font =
                    Enum.Font.GothamMedium,

                AutoButtonColor = false
            }, navHolder)

        addCorner(button, 9)

        navButtons[name] = button

        track(button.MouseButton1Click:Connect(function()
            showPage(name)
        end))
    end

    local function section(page, title)
        local card =
            create("Frame", {
                Size =
                    UDim2.new(
                        1,
                        0,
                        0,
                        0
                    ),

                AutomaticSize =
                    Enum.AutomaticSize.Y,

                BackgroundColor3 =
                    Colors.Card,

                BackgroundTransparency =
                    0.18,

                BorderSizePixel = 0
            }, page)

        addCorner(card, 11)
        addStroke(card)

        local holder =
            create("Frame", {
                Size =
                    UDim2.new(
                        1,
                        0,
                        0,
                        0
                    ),

                AutomaticSize =
                    Enum.AutomaticSize.Y,

                BackgroundTransparency = 1
            }, card)

        create("UIListLayout", {
            Padding =
                UDim.new(
                    0,
                    isPhone
                        and 10
                        or 8
                )
        }, holder)

        create("UIPadding", {
            PaddingLeft =
                UDim.new(0, 13),

            PaddingRight =
                UDim.new(0, 13),

            PaddingTop =
                UDim.new(0, 13),

            PaddingBottom =
                UDim.new(0, 13)
        }, holder)

        create("TextLabel", {
            Size =
                UDim2.new(
                    1,
                    0,
                    0,
                    22
                ),

            BackgroundTransparency = 1,

            Text = title,

            TextColor3 =
                Colors.Text,

            TextSize =
                isPhone and 15 or 14,

            Font =
                Enum.Font.GothamBold,

            TextXAlignment =
                Enum.TextXAlignment.Left
        }, holder)

        return holder
    end

    local function createToggle(
        parent,
        title,
        default,
        callback
    )
        local state =
            default == true

        local row =
            create("TextButton", {
                Size =
                    UDim2.new(
                        1,
                        0,
                        0,
                        isPhone
                            and 49
                            or 42
                    ),

                BackgroundColor3 =
                    Colors.Control,

                BackgroundTransparency =
                    0.08,

                BorderSizePixel = 0,

                Text = "",

                AutoButtonColor = false
            }, parent)

        addCorner(row, 9)

        create("TextLabel", {
            Position =
                UDim2.new(
                    0,
                    12,
                    0,
                    0
                ),

            Size =
                UDim2.new(
                    1,
                    -72,
                    1,
                    0
                ),

            BackgroundTransparency = 1,

            Text = title,

            TextColor3 =
                Colors.Text,

            TextSize =
                isPhone and 13 or 12,

            Font =
                Enum.Font.GothamMedium,

            TextXAlignment =
                Enum.TextXAlignment.Left
        }, row)

        local toggle =
            create("Frame", {
                AnchorPoint =
                    Vector2.new(1, 0.5),

                Position =
                    UDim2.new(
                        1,
                        -10,
                        0.5,
                        0
                    ),

                Size =
                    UDim2.fromOffset(
                        isPhone
                            and 44
                            or 38,

                        isPhone
                            and 24
                            or 21
                    ),

                BackgroundColor3 =
                    state
                        and Colors.AccentDark
                        or Colors.Stroke,

                BorderSizePixel = 0
            }, row)

        addCorner(toggle, 100)

        local dotSize =
            isPhone and 18 or 15

        local dot =
            create("Frame", {
                AnchorPoint =
                    Vector2.new(0.5, 0.5),

                Position = state
                    and UDim2.new(
                        1,
                        -(dotSize / 2 + 3),
                        0.5,
                        0
                    )
                    or UDim2.new(
                        0,
                        dotSize / 2 + 3,
                        0.5,
                        0
                    ),

                Size =
                    UDim2.fromOffset(
                        dotSize,
                        dotSize
                    ),

                BackgroundColor3 =
                    state
                        and Colors.Accent
                        or Colors.SubText,

                BorderSizePixel = 0
            }, toggle)

        addCorner(dot, 100)

        local function setState(value)
            state = value

            toggle.BackgroundColor3 =
                state
                    and Colors.AccentDark
                    or Colors.Stroke

            dot.BackgroundColor3 =
                state
                    and Colors.Accent
                    or Colors.SubText

            TweenService:Create(
                dot,
                TweenInfo.new(0.13),
                {
                    Position = state
                        and UDim2.new(
                            1,
                            -(dotSize / 2 + 3),
                            0.5,
                            0
                        )
                        or UDim2.new(
                            0,
                            dotSize / 2 + 3,
                            0.5,
                            0
                        )
                }
            ):Play()

            if callback then
                callback(state)
            end
        end

        track(row.MouseButton1Click:Connect(function()
            setState(not state)
        end))

        return {
            Get = function()
                return state
            end,

            Set = function(_, value)
                setState(value)
            end
        }
    end

    local function createSlider(
        parent,
        title,
        minimum,
        maximum,
        default,
        callback
    )
        local value =
            math.clamp(
                default,
                minimum,
                maximum
            )

        local holder =
            create("Frame", {
                Size =
                    UDim2.new(
                        1,
                        0,
                        0,
                        isPhone
                            and 72
                            or 62
                    ),

                BackgroundColor3 =
                    Colors.Control,

                BackgroundTransparency =
                    0.08,

                BorderSizePixel = 0
            }, parent)

        addCorner(holder, 9)

        create("TextLabel", {
            Position =
                UDim2.new(
                    0,
                    12,
                    0,
                    8
                ),

            Size =
                UDim2.new(
                    1,
                    -90,
                    0,
                    20
                ),

            BackgroundTransparency = 1,

            Text = title,

            TextColor3 =
                Colors.Text,

            TextSize = 12,

            Font =
                Enum.Font.GothamMedium,

            TextXAlignment =
                Enum.TextXAlignment.Left
        }, holder)

        local valueLabel =
            create("TextLabel", {
                AnchorPoint =
                    Vector2.new(1, 0),

                Position =
                    UDim2.new(
                        1,
                        -12,
                        0,
                        8
                    ),

                Size =
                    UDim2.fromOffset(
                        75,
                        20
                    ),

                BackgroundTransparency = 1,

                Text =
                    tostring(value),

                TextColor3 =
                    Colors.Accent,

                TextSize = 12,

                Font =
                    Enum.Font.GothamBold,

                TextXAlignment =
                    Enum.TextXAlignment.Right
            }, holder)

        local bar =
            create("Frame", {
                Position =
                    UDim2.new(
                        0,
                        12,
                        0,
                        isPhone
                            and 45
                            or 39
                    ),

                Size =
                    UDim2.new(
                        1,
                        -24,
                        0,
                        isPhone
                            and 10
                            or 7
                    ),

                BackgroundColor3 =
                    Colors.Stroke,

                BorderSizePixel = 0
            }, holder)

        addCorner(bar, 100)

        local percentage =
            (value - minimum)
            / (maximum - minimum)

        local fill =
            create("Frame", {
                Size =
                    UDim2.fromScale(
                        percentage,
                        1
                    ),

                BackgroundColor3 =
                    Colors.Accent,

                BorderSizePixel = 0
            }, bar)

        addCorner(fill, 100)

        local knob =
            create("Frame", {
                AnchorPoint =
                    Vector2.new(0.5, 0.5),

                Position =
                    UDim2.new(
                        percentage,
                        0,
                        0.5,
                        0
                    ),

                Size =
                    UDim2.fromOffset(
                        isPhone
                            and 20
                            or 16,

                        isPhone
                            and 20
                            or 16
                    ),

                BackgroundColor3 =
                    Colors.Text,

                BorderSizePixel = 0
            }, bar)

        addCorner(knob, 100)

        local dragging = false

        local function update(x)
            local p =
                math.clamp(
                    (
                        x
                        - bar.AbsolutePosition.X
                    )
                    / math.max(
                        bar.AbsoluteSize.X,
                        1
                    ),
                    0,
                    1
                )

            value =
                math.floor(
                    minimum
                    + (
                        maximum
                        - minimum
                    )
                    * p
                    + 0.5
                )

            local display =
                (value - minimum)
                / (maximum - minimum)

            valueLabel.Text =
                tostring(value)

            fill.Size =
                UDim2.fromScale(
                    display,
                    1
                )

            knob.Position =
                UDim2.new(
                    display,
                    0,
                    0.5,
                    0
                )

            if callback then
                callback(value)
            end
        end

        track(bar.InputBegan:Connect(function(input)
            if input.UserInputType
                    == Enum.UserInputType.MouseButton1
                or input.UserInputType
                    == Enum.UserInputType.Touch then

                dragging = true

                update(input.Position.X)
            end
        end))

        track(UserInputService.InputChanged:Connect(function(input)
            if dragging
                and (
                    input.UserInputType
                        == Enum.UserInputType.MouseMovement
                    or input.UserInputType
                        == Enum.UserInputType.Touch
                ) then

                update(input.Position.X)
            end
        end))

        track(UserInputService.InputEnded:Connect(function(input)
            if input.UserInputType
                    == Enum.UserInputType.MouseButton1
                or input.UserInputType
                    == Enum.UserInputType.Touch then

                dragging = false
            end
        end))

        return holder
    end

    local function createNumberInput(
        parent,
        title,
        default,
        minimum,
        maximum,
        callback
    )
        local holder =
            create("Frame", {
                Size =
                    UDim2.new(
                        1,
                        0,
                        0,
                        isPhone
                            and 49
                            or 42
                    ),

                BackgroundColor3 =
                    Colors.Control,

                BackgroundTransparency =
                    0.08,

                BorderSizePixel = 0
            }, parent)

        addCorner(holder, 9)

        create("TextLabel", {
            Position =
                UDim2.new(
                    0,
                    12,
                    0,
                    0
                ),

            Size =
                UDim2.new(
                    1,
                    -115,
                    1,
                    0
                ),

            BackgroundTransparency = 1,

            Text = title,

            TextColor3 =
                Colors.Text,

            TextSize = 12,

            Font =
                Enum.Font.GothamMedium,

            TextXAlignment =
                Enum.TextXAlignment.Left
        }, holder)

        local box =
            create("TextBox", {
                AnchorPoint =
                    Vector2.new(1, 0.5),

                Position =
                    UDim2.new(
                        1,
                        -10,
                        0.5,
                        0
                    ),

                Size =
                    UDim2.fromOffset(
                        isPhone
                            and 90
                            or 80,

                        isPhone
                            and 34
                            or 30
                    ),

                BackgroundColor3 =
                    Colors.Card,

                BackgroundTransparency = 0.1,

                BorderSizePixel = 0,

                Text =
                    tostring(default),

                TextColor3 =
                    Colors.Accent,

                TextSize = 12,

                Font =
                    Enum.Font.GothamBold,

                ClearTextOnFocus = false
            }, holder)

        addCorner(box, 7)
        addStroke(box)

        track(box.FocusLost:Connect(function()
            local value =
                tonumber(box.Text)
                or default

            value =
                math.clamp(
                    value,
                    minimum,
                    maximum
                )

            box.Text =
                tostring(value)

            if callback then
                callback(value)
            end
        end))

        return box
    end

    local function createAction(
        parent,
        title,
        callback,
        danger
    )
        local normal =
            danger
                and Color3.fromRGB(73, 36, 40)
                or Colors.Control

        local hover =
            danger
                and Color3.fromRGB(90, 42, 46)
                or Colors.CardHover

        local pressed =
            danger
                and Color3.fromRGB(58, 28, 31)
                or Color3.fromRGB(30, 32, 36)

        local button =
            create("TextButton", {
                Size =
                    UDim2.new(
                        1,
                        0,
                        0,
                        isPhone
                            and 49
                            or 42
                    ),

                BackgroundColor3 = normal,

                BackgroundTransparency = 0.08,

                BorderSizePixel = 0,

                Text = title,

                TextColor3 =
                    danger
                        and Colors.Danger
                        or Colors.Text,

                TextSize = 12,

                Font =
                    Enum.Font.GothamMedium,

                AutoButtonColor = false
            }, parent)

        addCorner(button, 9)

        animateButton(
            button,
            normal,
            hover,
            pressed
        )

        track(button.MouseButton1Click:Connect(function()
            if callback then
                callback()
            end
        end))

        return button
    end

    local function createUpdateCard(
        parent,
        title,
        date,
        lines
    )
        local card =
            create("Frame", {
                Size =
                    UDim2.new(
                        1,
                        0,
                        0,
                        0
                    ),

                AutomaticSize =
                    Enum.AutomaticSize.Y,

                BackgroundColor3 =
                    Colors.Card,

                BackgroundTransparency =
                    0.18,

                BorderSizePixel = 0
            }, parent)

        addCorner(card, 11)
        addStroke(card)

        local holder =
            create("Frame", {
                Size =
                    UDim2.new(
                        1,
                        0,
                        0,
                        0
                    ),

                AutomaticSize =
                    Enum.AutomaticSize.Y,

                BackgroundTransparency = 1
            }, card)

        create("UIListLayout", {
            Padding =
                UDim.new(0, 6)
        }, holder)

        create("UIPadding", {
            PaddingLeft =
                UDim.new(0, 14),

            PaddingRight =
                UDim.new(0, 14),

            PaddingTop =
                UDim.new(0, 14),

            PaddingBottom =
                UDim.new(0, 14)
        }, holder)

        create("TextLabel", {
            Size =
                UDim2.new(
                    1,
                    0,
                    0,
                    23
                ),

            BackgroundTransparency = 1,

            Text = title,

            TextColor3 =
                Colors.Text,

            TextSize = 16,

            Font =
                Enum.Font.GothamBold,

            TextXAlignment =
                Enum.TextXAlignment.Left
        }, holder)

        create("TextLabel", {
            Size =
                UDim2.new(
                    1,
                    0,
                    0,
                    18
                ),

            BackgroundTransparency = 1,

            Text = date,

            TextColor3 =
                Colors.Accent,

            TextSize = 11,

            Font =
                Enum.Font.GothamMedium,

            TextXAlignment =
                Enum.TextXAlignment.Left
        }, holder)

        for _, line in ipairs(lines) do
            create("TextLabel", {
                Size =
                    UDim2.new(
                        1,
                        0,
                        0,
                        0
                    ),

                AutomaticSize =
                    Enum.AutomaticSize.Y,

                BackgroundTransparency = 1,

                Text = line,

                TextColor3 =
                    Colors.SubText,

                TextSize =
                    isPhone and 12 or 11,

                Font =
                    Enum.Font.Gotham,

                TextWrapped = true,

                TextXAlignment =
                    Enum.TextXAlignment.Left
            }, holder)
        end
    end

    local Home = createPage("Home")
    local Sheriff = createPage("Sheriff")
    local Murderer = createPage("Murderer")
    local Innocent = createPage("Innocent")
    local Updates = createPage("Updates")
    local Utility = createPage("Utility")

    createNav("Home")
    createNav("Sheriff")
    createNav("Murderer")
    createNav("Innocent")
    createNav("Updates")
    createNav("Utility")

    local homeSection =
        section(
            Home,
            "THH HUB"
        )

    create("TextLabel", {
        Size =
            UDim2.new(
                1,
                0,
                0,
                28
            ),

        BackgroundTransparency = 1,

        Text =
            "welcome "
            .. player.DisplayName,

        TextColor3 =
            Colors.Accent,

        TextSize = 12,

        Font =
            Enum.Font.GothamMedium,

        TextXAlignment =
            Enum.TextXAlignment.Left
    }, homeSection)

    local sheriffSection =
        section(
            Sheriff,
            "Sheriff"
        )

    createToggle(
        sheriffSection,
        "Aim Bot",
        sheriff.aimBot,
        function(value)
            sheriff.aimBot = value
        end
    )

    local murdererSection =
        section(
            Murderer,
            "Murderer"
        )

    createAction(
        murdererSection,
        "Throw Knife",
        function()
            if not throwKnifeOnce() then
                notifications(
                    "Knife",
                    "no Knife found"
                )
            end
        end
    )

    createToggle(
        murdererSection,
        "Auto Throw Knife",
        knife.autoThrow,
        function(value)
            knife.autoThrow = value
        end
    )

    createSlider(
        murdererSection,
        "Throw Delay",
        5,
        100,
        20,
        function(value)
            knife.throwDelay =
                value / 100
        end
    )

    local espSection =
        section(
            Innocent,
            "ESP"
        )

    createToggle(
        espSection,
        "ESP",
        esp.enabled,
        function(value)
            esp.enabled = value

            if value then
                updateESP()
            else
                clearESP()
            end
        end
    )

    local gunSection =
        section(
            Innocent,
            "Gun"
        )

    createAction(
        gunSection,
        "Pick Up Gun",
        function()
            if alreadyHasGun() then
                notifications(
                    "Gun",
                    "you already have it"
                )

                return
            end

            if not findGunDrop() then
                notifications(
                    "Gun",
                    "no gun drop"
                )

                return
            end

            pickupGun()
        end
    )

    createToggle(
        gunSection,
        "Auto Pick Up Gun",
        gunPickup.auto,
        function(value)
            gunPickup.auto = value
        end
    )

    local movementSection =
        section(
            Innocent,
            "Movement"
        )

    createToggle(
        movementSection,
        "Speed",
        movement.speedEnabled,
        function(value)
            movement.speedEnabled = value

            if not value
                and humanoid then

                humanoid.WalkSpeed =
                    originalWalkSpeed
            end
        end
    )

    createSlider(
        movementSection,
        "Speed",
        16,
        250,
        movement.speed,
        function(value)
            movement.speed = value
        end
    )

    createToggle(
        movementSection,
        "Infinite Jump",
        movement.infiniteJump,
        function(value)
            movement.infiniteJump = value
        end
    )

    createToggle(
        movementSection,
        "Noclip",
        movement.noclip,
        function(value)
            movement.noclip = value

            if not value then
                restoreNoclip()
            end
        end
    )

    createToggle(
        movementSection,
        "Spin Bot",
        movement.spin,
        function(value)
            movement.spin = value
        end
    )

    createSlider(
        movementSection,
        "Spin Speed",
        30,
        1500,
        movement.spinSpeed,
        function(value)
            movement.spinSpeed = value
        end
    )

    createUpdateCard(
        Updates,
        "what's new",
        "09/08/26",
        {
            "• cleaned the menu up a lot",
            "• removed the random text under esp",
            "• aim bot is max now",
            "• removed aim settings",
            "• made the key screen look nicer",
            "• added more button animations",
            "• made get key look way better",
            "• esp still updates every 0.1 sec",
            "• fixed the pc / phone picker"
        }
    )

    local utilitySection =
        section(
            Utility,
            "Utility"
        )

    createAction(
        utilitySection,
        "Copy Server ID",
        function()
            local copied = false

            if setclipboard then
                copied =
                    pcall(function()
                        setclipboard(
                            game.JobId
                        )
                    end)

            elseif toclipboard then
                copied =
                    pcall(function()
                        toclipboard(
                            game.JobId
                        )
                    end)
            end

            notifications(
                "Server ID",
                copied
                    and "copied"
                    or game.JobId
            )
        end
    )

    createAction(
        utilitySection,
        "Rejoin Server",
        function()
            pcall(function()
                TeleportService:
                    TeleportToPlaceInstance(
                        game.PlaceId,
                        game.JobId,
                        player
                    )
            end)
        end
    )

    createAction(
        utilitySection,
        "Reset Character",
        function()
            if humanoid then
                humanoid.Health = 0
            end
        end
    )

    local floatingButton

    if isPhone then
        floatingButton =
            create("ImageButton", {
                AnchorPoint =
                    Vector2.new(1, 1),

                Position =
                    UDim2.new(
                        1,
                        -17,
                        1,
                        -20
                    ),

                Size =
                    UDim2.fromOffset(
                        52,
                        52
                    ),

                BackgroundColor3 =
                    Colors.Card,

                BackgroundTransparency =
                    0.08,

                BorderSizePixel = 0,

                Image = MM2_ICON,

                Visible = false,

                AutoButtonColor = false,

                ZIndex = 300
            }, gui)

        addCorner(
            floatingButton,
            14
        )

        addStroke(
            floatingButton,
            Colors.Accent,
            1,
            0.25
        )
    end

    local function setMenuVisible(value)
        menu.Visible = value

        if floatingButton then
            floatingButton.Visible =
                not value
        end
    end

    track(minimizeButton.MouseButton1Click:Connect(function()
        setMenuVisible(false)
    end))

    track(closeButton.MouseButton1Click:Connect(function()
        setMenuVisible(false)
    end))

    if floatingButton then
        track(
            floatingButton.MouseButton1Click:
                Connect(function()

                    setMenuVisible(true)
                end)
        )
    end

    track(UserInputService.InputBegan:Connect(function(
        input,
        gameProcessed
    )
        if gameProcessed
            or isPhone then

            return
        end

        if input.KeyCode
            == Enum.KeyCode.RightShift then

            setMenuVisible(
                not menu.Visible
            )
        end
    end))

    createAction(
        utilitySection,
        "Unload Menu",
        function()
            local overlay =
                create("Frame", {
                    Size =
                        UDim2.fromScale(
                            1,
                            1
                        ),

                    BackgroundColor3 =
                        Color3.fromRGB(
                            3,
                            4,
                            5
                        ),

                    BackgroundTransparency =
                        0.15,

                    BorderSizePixel = 0,

                    ZIndex = 800
                }, gui)

            local dialog =
                create("Frame", {
                    AnchorPoint =
                        Vector2.new(
                            0.5,
                            0.5
                        ),

                    Position =
                        UDim2.fromScale(
                            0.5,
                            0.5
                        ),

                    Size =
                        UDim2.new(
                            0.86,
                            0,
                            0,
                            220
                        ),

                    BackgroundColor3 =
                        Colors.Card,

                    BackgroundTransparency =
                        0.08,

                    BorderSizePixel = 0,

                    ZIndex = 801
                }, overlay)

            create("UISizeConstraint", {
                MinSize =
                    Vector2.new(
                        285,
                        210
                    ),

                MaxSize =
                    Vector2.new(
                        410,
                        230
                    )
            }, dialog)

            addCorner(dialog, 13)

            addStroke(
                dialog,
                Colors.Danger,
                1,
                0.35
            )

            create("TextLabel", {
                Position =
                    UDim2.new(
                        0,
                        18,
                        0,
                        17
                    ),

                Size =
                    UDim2.new(
                        1,
                        -36,
                        0,
                        27
                    ),

                BackgroundTransparency = 1,

                Text =
                    "Unload THH HUB?",

                TextColor3 =
                    Colors.Text,

                TextSize = 18,

                Font =
                    Enum.Font.GothamBold,

                TextXAlignment =
                    Enum.TextXAlignment.Left,

                ZIndex = 802
            }, dialog)

            create("TextLabel", {
                Position =
                    UDim2.new(
                        0,
                        18,
                        0,
                        55
                    ),

                Size =
                    UDim2.new(
                        1,
                        -36,
                        0,
                        60
                    ),

                BackgroundTransparency = 1,

                Text =
                    "ALL MODS WILL BE TURNED OFF.\n"
                    .. "YOU WILL NO LONGER BE ABLE TO OPEN THIS MENU.",

                TextColor3 =
                    Colors.Danger,

                TextSize = 11,

                Font =
                    Enum.Font.GothamBold,

                TextWrapped = true,

                TextXAlignment =
                    Enum.TextXAlignment.Left,

                ZIndex = 802
            }, dialog)

            local cancel =
                create("TextButton", {
                    Position =
                        UDim2.new(
                            0,
                            18,
                            1,
                            -59
                        ),

                    Size =
                        UDim2.new(
                            0.47,
                            -8,
                            0,
                            41
                        ),

                    BackgroundColor3 =
                        Colors.Control,

                    BorderSizePixel = 0,

                    Text = "Cancel",

                    TextColor3 =
                        Colors.Text,

                    TextSize = 12,

                    Font =
                        Enum.Font.GothamMedium,

                    ZIndex = 802
                }, dialog)

            addCorner(cancel, 8)

            local unload =
                create("TextButton", {
                    AnchorPoint =
                        Vector2.new(1, 0),

                    Position =
                        UDim2.new(
                            1,
                            -18,
                            1,
                            -59
                        ),

                    Size =
                        UDim2.new(
                            0.53,
                            -8,
                            0,
                            41
                        ),

                    BackgroundColor3 =
                        Color3.fromRGB(
                            76,
                            34,
                            38
                        ),

                    BorderSizePixel = 0,

                    Text =
                        "Unload Menu",

                    TextColor3 =
                        Colors.Danger,

                    TextSize = 12,

                    Font =
                        Enum.Font.GothamBold,

                    ZIndex = 802
                }, dialog)

            addCorner(unload, 8)

            track(cancel.MouseButton1Click:Connect(function()
                overlay:Destroy()
            end))

            track(unload.MouseButton1Click:Connect(function()
                _G.THHGrowersHardUnloaded =
                    true

                cleanup()
            end))
        end,
        true
    )

    showPage("Home")

    notifications(
        "THH HUB",
        string.lower(deviceMode)
        .. " mode loaded"
    )
end
