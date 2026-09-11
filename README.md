if _G.THHGrowersCleanup then
    pcall(_G.THHGrowersCleanup)
end

_G.THHGrowersCleanup = nil

local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local TeleportService = game:GetService("TeleportService")
local Lighting = game:GetService("Lighting")
local RbxAnalyticsService = game:GetService("RbxAnalyticsService")

local VIM

pcall(function()
    VIM = game:GetService("VirtualInputManager")
end)

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local PROJECT_ID = "PD6XH2EGXZXUHGED"
local AUTH_SECRET = "04cce9295d9d2476cb2516b3363693a3c401767b4e75bd01"
local KEY_FLOW = "https://vampauth.com/PD6XH2EGXZXUHGED/flow"

local MM2_GAME_ID = 66654135

local MM2_ICON =
    "rbxthumb://type=GameIcon&id="
    .. tostring(MM2_GAME_ID)
    .. "&w=150&h=150"

local C = {
    Background = Color3.fromRGB(25, 26, 30),
    Background2 = Color3.fromRGB(31, 32, 37),
    Sidebar = Color3.fromRGB(35, 36, 41),

    Card = Color3.fromRGB(57, 59, 66),
    Control = Color3.fromRGB(46, 48, 54),
    ControlHover = Color3.fromRGB(55, 57, 64),

    Text = Color3.fromRGB(245, 246, 248),
    SubText = Color3.fromRGB(181, 184, 191),

    Accent = Color3.fromRGB(106, 221, 145),
    AccentDark = Color3.fromRGB(45, 81, 59),

    Stroke = Color3.fromRGB(102, 105, 114),

    Danger = Color3.fromRGB(232, 81, 81),

    Innocent = Color3.fromRGB(72, 226, 113),
    Murderer = Color3.fromRGB(243, 72, 72),
    Sheriff = Color3.fromRGB(69, 147, 255)
}

local alive = true
local runtimeStarted = false

local gui
local blur

local character
local humanoid
local rootPart

local originalWalkSpeed = 16

local connections = {}
local runtimeConnections = {}
local bagTextConnections = {}

local function disconnect(connection)
    if connection then
        pcall(function()
            connection:Disconnect()
        end)
    end
end

local function track(connection)
    table.insert(connections, connection)
    return connection
end

local function trackRuntime(connection)
    table.insert(runtimeConnections, connection)
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

local function corner(object, radius)
    return create(
        "UICorner",
        {
            CornerRadius = UDim.new(0, radius or 9)
        },
        object
    )
end

local function stroke(
    object,
    color,
    thickness,
    transparency
)
    return create(
        "UIStroke",
        {
            Color = color or C.Stroke,
            Thickness = thickness or 1,
            Transparency = transparency or 0.55
        },
        object
    )
end

local function tween(object, duration, properties)
    TweenService:Create(
        object,
        TweenInfo.new(
            duration,
            Enum.EasingStyle.Quint,
            Enum.EasingDirection.Out
        ),
        properties
    ):Play()
end

local function animateButton(button)
    if not UIS.MouseEnabled then
        return
    end

    local original = button.BackgroundColor3

    track(
        button.MouseEnter:Connect(function()
            tween(
                button,
                0.12,
                {
                    BackgroundColor3 = C.ControlHover
                }
            )
        end)
    )

    track(
        button.MouseLeave:Connect(function()
            tween(
                button,
                0.12,
                {
                    BackgroundColor3 = original
                }
            )
        end)
    )
end

local function makeDraggable(frame, handle)
    handle = handle or frame

    local dragging = false
    local dragInput
    local dragStart
    local startPosition

    track(
        handle.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1
                or input.UserInputType == Enum.UserInputType.Touch then

                dragging = true
                dragStart = input.Position
                startPosition = frame.Position
            end
        end)
    )

    track(
        handle.InputChanged:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseMovement
                or input.UserInputType == Enum.UserInputType.Touch then

                dragInput = input
            end
        end)
    )

    track(
        UIS.InputChanged:Connect(function(input)
            if not dragging
                or input ~= dragInput then

                return
            end

            local delta = input.Position - dragStart

            frame.Position =
                UDim2.new(
                    startPosition.X.Scale,
                    startPosition.X.Offset + delta.X,
                    startPosition.Y.Scale,
                    startPosition.Y.Offset + delta.Y
                )
        end)
    )

    track(
        UIS.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1
                or input.UserInputType == Enum.UserInputType.Touch then

                dragging = false
            end
        end)
    )
end

local oldGui = playerGui:FindFirstChild("THH_HUB")

if oldGui then
    oldGui:Destroy()
end

gui =
    create(
        "ScreenGui",
        {
            Name = "THH_HUB",
            ResetOnSpawn = false,
            IgnoreGuiInset = true,
            DisplayOrder = 999999,
            ZIndexBehavior = Enum.ZIndexBehavior.Sibling
        },
        playerGui
    )

blur =
    create(
        "BlurEffect",
        {
            Name = "THH_HUB_Blur",
            Size = 20
        },
        Lighting
    )

local authOverlay =
    create(
        "Frame",
        {
            Size = UDim2.fromScale(1, 1),
            BackgroundColor3 = Color3.fromRGB(8, 9, 12),
            BackgroundTransparency = 0.02,
            BorderSizePixel = 0
        },
        gui
    )

local authGradient =
    create(
        "UIGradient",
        {
            Rotation = -25,

            Color =
                ColorSequence.new({
                    ColorSequenceKeypoint.new(
                        0,
                        Color3.fromRGB(8, 9, 12)
                    ),

                    ColorSequenceKeypoint.new(
                        0.5,
                        Color3.fromRGB(43, 45, 51)
                    ),

                    ColorSequenceKeypoint.new(
                        1,
                        Color3.fromRGB(8, 9, 12)
                    )
                })
        },
        authOverlay
    )

task.spawn(function()
    while alive
        and authGradient.Parent do

        tween(
            authGradient,
            5,
            {
                Rotation = 25
            }
        )

        task.wait(5)

        if not authGradient.Parent then
            break
        end

        tween(
            authGradient,
            5,
            {
                Rotation = -25
            }
        )

        task.wait(5)
    end
end)

local authWindow =
    create(
        "Frame",
        {
            AnchorPoint = Vector2.new(0.5, 0.5),
            Position = UDim2.fromScale(0.5, 0.5),

            Size =
                UDim2.new(
                    0.90,
                    0,
                    0,
                    350
                ),

            BackgroundColor3 =
                Color3.fromRGB(
                    52,
                    54,
                    61
                ),

            BackgroundTransparency = 0.14,
            BorderSizePixel = 0,
            Active = true
        },
        authOverlay
    )

create(
    "UISizeConstraint",
    {
        MinSize = Vector2.new(300, 340),
        MaxSize = Vector2.new(430, 350)
    },
    authWindow
)

corner(authWindow, 17)

stroke(
    authWindow,
    Color3.fromRGB(
        155,
        158,
        166
    ),
    1,
    0.48
)

local authHeader =
    create(
        "Frame",
        {
            Size = UDim2.new(1, 0, 0, 78),
            BackgroundTransparency = 1,
            Active = true
        },
        authWindow
    )

makeDraggable(
    authWindow,
    authHeader
)

local authIcon =
    create(
        "ImageLabel",
        {
            Position = UDim2.fromOffset(18, 15),
            Size = UDim2.fromOffset(48, 48),

            BackgroundColor3 = C.Control,
            BackgroundTransparency = 0.05,

            BorderSizePixel = 0,

            Image = MM2_ICON
        },
        authHeader
    )

corner(authIcon, 12)

create(
    "TextLabel",
    {
        Position = UDim2.fromOffset(79, 15),
        Size = UDim2.new(1, -98, 0, 24),

        BackgroundTransparency = 1,

        Text = "THH HUB",

        TextColor3 = C.Text,
        TextSize = 20,

        Font = Enum.Font.GothamBold,

        TextXAlignment = Enum.TextXAlignment.Left
    },
    authHeader
)

create(
    "TextLabel",
    {
        Position = UDim2.fromOffset(79, 41),
        Size = UDim2.new(1, -98, 0, 18),

        BackgroundTransparency = 1,

        Text = "Authentication",

        TextColor3 = C.SubText,
        TextSize = 12,

        Font = Enum.Font.Gotham,

        TextXAlignment = Enum.TextXAlignment.Left
    },
    authHeader
)

local keyBox =
    create(
        "TextBox",
        {
            Position = UDim2.fromOffset(18, 91),
            Size = UDim2.new(1, -36, 0, 48),

            BackgroundColor3 = C.Control,
            BackgroundTransparency = 0.04,

            BorderSizePixel = 0,

            Text = "",

            PlaceholderText = "Enter access key",
            PlaceholderColor3 = C.SubText,

            TextColor3 = C.Text,
            TextSize = 14,

            Font = Enum.Font.Gotham,

            ClearTextOnFocus = false,

            TextXAlignment = Enum.TextXAlignment.Left
        },
        authWindow
    )

corner(keyBox, 10)
stroke(keyBox)

create(
    "UIPadding",
    {
        PaddingLeft = UDim.new(0, 14),
        PaddingRight = UDim.new(0, 14)
    },
    keyBox
)

local continueButton =
    create(
        "TextButton",
        {
            Position = UDim2.fromOffset(18, 153),
            Size = UDim2.new(1, -36, 0, 45),

            BackgroundColor3 = C.Accent,
            BorderSizePixel = 0,

            Text = "Continue",

            TextColor3 = Color3.fromRGB(18, 24, 20),
            TextSize = 14,

            Font = Enum.Font.GothamBold,

            AutoButtonColor = false
        },
        authWindow
    )

corner(continueButton, 10)

local getKeyButton =
    create(
        "TextButton",
        {
            Position = UDim2.fromOffset(18, 212),
            Size = UDim2.new(1, -36, 0, 45),

            BackgroundColor3 = C.Control,
            BackgroundTransparency = 0.04,

            BorderSizePixel = 0,

            Text = "Get Access Key",

            TextColor3 = C.Text,
            TextSize = 13,

            Font = Enum.Font.GothamBold,

            AutoButtonColor = false
        },
        authWindow
    )

corner(getKeyButton, 10)
animateButton(getKeyButton)

local authStatus =
    create(
        "TextLabel",
        {
            Position = UDim2.fromOffset(18, 276),
            Size = UDim2.new(1, -36, 0, 40),

            BackgroundTransparency = 1,

            Text = "paste your key",

            TextColor3 = C.SubText,
            TextSize = 12,

            TextWrapped = true,

            Font = Enum.Font.Gotham
        },
        authWindow
    )

local movement = {
    speedEnabled = false,
    speed = 32,

    infiniteJump = false,

    spin = false,
    spinSpeed = 300,

    spinVelocity = nil,
    oldAutoRotate = true
}

local collision = {
    sources = {
        noclip = false,
        coinFarm = false,
        underground = false
    },

    originals = {},
    listeners = {},
    descendantConnection = nil
}

local antiFling = {
    enabled = false,
    generation = 0,

    originals = {},
    parts = {},
    connections = {}
}

local esp = {
    enabled = true,

    highlights = {},

    characterConnections = {},
    playerConnections = {}
}

local sheriff = {
    quickShot = false,
    shooting = false
}

local murderer = {
    autoThrow = false,
    throwing = false,

    throwDelay = 0.20,
    lastThrow = 0,

    cachedThrowButton = nil
}

local gunPickup = {
    auto = false,
    busy = false,

    drop = nil,
    lastAttempt = 0
}

local coinFarm = {
    enabled = false,
    busy = false,

    coins = {},
    lookup = {},

    blockedUntil = {},

    current = nil,

    collected = 0,

    status = "Idle",

    travelSpeed = 30,
    verticalSpeed = 150,

    underDistance = 8.5,

    platform = nil,

    fullBagDebounce = false,
    cachedBagText = nil,

    bagCooldownUntil = 0,

    lastCacheClean = 0,

    counter = {
        collected = nil,
        available = nil,
        target = nil,
        status = nil
    }
}

local safety = {
    underground = false,
    depth = 300,

    gravityAttachment = nil,
    gravityForce = nil
}

local replay = {
    recording = false,
    playing = false,

    currentTake = nil,
    takes = {},

    takeCounter = 0,

    maxTakes = 2,
    maxDuration = 120,
    sampleRate = 1 / 20,

    playbackFolder = nil,
    playbackActors = {},
    playbackConnection = nil,

    offset = Vector3.new(0, -12000, 0),

    oldCameraType = nil,
    oldCameraSubject = nil,
    oldCameraCFrame = nil,
    oldCameraFov = nil,

    oldWalkSpeed = nil,
    oldJumpPower = nil,
    oldJumpHeight = nil,
    oldAutoRotate = nil,

    statusLabel = nil
}

local function refreshCharacter()
    character = player.Character

    if not character then
        return false
    end

    humanoid =
        character:FindFirstChildOfClass(
            "Humanoid"
        )

    rootPart =
        character:FindFirstChild(
            "HumanoidRootPart"
        )

    if humanoid then
        originalWalkSpeed =
            humanoid.WalkSpeed
    end

    return humanoid ~= nil
        and rootPart ~= nil
end

local function waitForCharacter()
    while alive do
        if refreshCharacter() then
            return true
        end

        task.wait(0.1)
    end

    return false
end

local function findTool(
    targetPlayer,
    wantedName
)
    if not targetPlayer then
        return nil
    end

    wantedName =
        string.lower(
            wantedName
        )

    local containers = {
        targetPlayer.Character,
        targetPlayer:FindFirstChildOfClass(
            "Backpack"
        )
    }

    for _, container in ipairs(
        containers
    ) do
        if container then
            for _, object in ipairs(
                container:GetChildren()
            ) do
                if object:IsA("Tool")
                    and string.lower(
                        object.Name
                    ) == wantedName then

                    return object
                end
            end
        end
    end

    return nil
end

local function getKnifeTool()
    local exact =
        findTool(
            player,
            "knife"
        )

    if exact then
        return exact
    end

    for _, container in ipairs({
        character,
        player:FindFirstChildOfClass(
            "Backpack"
        )
    }) do
        if container then
            for _, object in ipairs(
                container:GetChildren()
            ) do
                if object:IsA("Tool")
                    and string.lower(
                        object.Name
                    ):find(
                        "knife",
                        1,
                        true
                    ) then

                    return object
                end
            end
        end
    end

    return nil
end

local function getGunTool()
    local exact =
        findTool(
            player,
            "gun"
        )

    if exact then
        return exact
    end

    for _, container in ipairs({
        character,
        player:FindFirstChildOfClass(
            "Backpack"
        )
    }) do
        if container then
            for _, object in ipairs(
                container:GetChildren()
            ) do
                if object:IsA("Tool") then
                    local name =
                        string.lower(
                            object.Name
                        )

                    if name:find(
                        "gun",
                        1,
                        true
                    )
                        or name:find(
                            "revolver",
                            1,
                            true
                        ) then

                        return object
                    end
                end
            end
        end
    end

    return nil
end

local function hasKnife(targetPlayer)
    if not targetPlayer then
        return false
    end

    if findTool(
        targetPlayer,
        "knife"
    ) then
        return true
    end

    for _, container in ipairs({
        targetPlayer.Character,
        targetPlayer:FindFirstChildOfClass(
            "Backpack"
        )
    }) do
        if container then
            for _, object in ipairs(
                container:GetChildren()
            ) do
                if object:IsA("Tool")
                    and string.lower(
                        object.Name
                    ):find(
                        "knife",
                        1,
                        true
                    ) then

                    return true
                end
            end
        end
    end

    return false
end

local function hasGun(targetPlayer)
    if not targetPlayer then
        return false
    end

    if targetPlayer == player then
        return getGunTool() ~= nil
    end

    for _, container in ipairs({
        targetPlayer.Character,
        targetPlayer:FindFirstChildOfClass(
            "Backpack"
        )
    }) do
        if container then
            for _, object in ipairs(
                container:GetChildren()
            ) do
                if object:IsA("Tool") then
                    local name =
                        string.lower(
                            object.Name
                        )

                    if name:find(
                        "gun",
                        1,
                        true
                    )
                        or name:find(
                            "revolver",
                            1,
                            true
                        ) then

                        return true
                    end
                end
            end
        end
    end

    return false
end

local function collisionActive()
    for _, enabled in pairs(
        collision.sources
    ) do
        if enabled then
            return true
        end
    end

    return false
end

local function clearCollisionPart(part)
    disconnect(
        collision.listeners[part]
    )

    collision.listeners[part] =
        nil
end

local function enforceCollisionPart(part)
    if not part
        or not part:IsA("BasePart")
        or not collisionActive() then

        return
    end

    if collision.originals[part] == nil then
        collision.originals[part] =
            part.CanCollide
    end

    part.CanCollide =
        false

    if not collision.listeners[part] then
        collision.listeners[part] =
            part:GetPropertyChangedSignal(
                "CanCollide"
            ):Connect(function()

                if collisionActive()
                    and part.Parent
                    and part.CanCollide then

                    part.CanCollide =
                        false
                end
            end)
    end
end

local function restoreCollision()
    disconnect(
        collision.descendantConnection
    )

    collision.descendantConnection =
        nil

    for part, oldValue in pairs(
        collision.originals
    ) do
        clearCollisionPart(
            part
        )

        if part
            and part.Parent then

            pcall(function()
                part.CanCollide =
                    oldValue
            end)
        end
    end

    table.clear(
        collision.originals
    )
end

local function rebuildCollision()
    disconnect(
        collision.descendantConnection
    )

    collision.descendantConnection =
        nil

    if not collisionActive() then
        restoreCollision()
        return
    end

    if not character then
        return
    end

    for _, object in ipairs(
        character:GetDescendants()
    ) do
        if object:IsA("BasePart") then
            enforceCollisionPart(
                object
            )
        end
    end

    collision.descendantConnection =
        character.DescendantAdded:Connect(function(object)

            if object:IsA("BasePart") then
                enforceCollisionPart(
                    object
                )
            end
        end)
end

local function setCollisionSource(
    source,
    value
)
    collision.sources[source] =
        value == true

    rebuildCollision()
end

local function setSpeedEnabled(value)
    movement.speedEnabled =
        value == true

    if humanoid then
        humanoid.WalkSpeed =
            movement.speedEnabled
            and movement.speed
            or originalWalkSpeed
    end
end

local function setSpeed(value)
    movement.speed =
        value

    if movement.speedEnabled
        and humanoid then

        humanoid.WalkSpeed =
            value
    end
end

local function destroySpin()
    if movement.spinVelocity then
        pcall(function()
            movement.spinVelocity:Destroy()
        end)

        movement.spinVelocity =
            nil
    end

    if rootPart
        and rootPart.Parent then

        rootPart.AssemblyAngularVelocity =
            Vector3.zero
    end

    if humanoid then
        humanoid.PlatformStand =
            false

        humanoid.AutoRotate =
            movement.oldAutoRotate
    end
end

local function rebuildSpin()
    destroySpin()

    if not movement.spin
        or replay.playing
        or not rootPart
        or not humanoid then

        return
    end

    movement.oldAutoRotate =
        humanoid.AutoRotate

    humanoid.AutoRotate =
        false

    humanoid.PlatformStand =
        false

    local controller =
        Instance.new(
            "BodyAngularVelocity"
        )

    controller.Name =
        "THH_SpinVelocity"

    controller.AngularVelocity =
        Vector3.new(
            0,
            math.rad(
                movement.spinSpeed
            ),
            0
        )

    controller.MaxTorque =
        Vector3.new(
            0,
            1000000000,
            0
        )

    controller.P =
        1250

    controller.Parent =
        rootPart

    movement.spinVelocity =
        controller
end

local function setSpinEnabled(value)
    movement.spin =
        value == true

    if movement.spin then
        rebuildSpin()
    else
        destroySpin()
    end
end

local function setSpinSpeed(value)
    movement.spinSpeed =
        value

    if movement.spinVelocity
        and movement.spinVelocity.Parent then

        movement.spinVelocity.AngularVelocity =
            Vector3.new(
                0,
                math.rad(value),
                0
            )
    end
end

local function destroyUndergroundForce()
    if safety.gravityForce then
        pcall(function()
            safety.gravityForce:Destroy()
        end)

        safety.gravityForce =
            nil
    end

    if safety.gravityAttachment then
        pcall(function()
            safety.gravityAttachment:Destroy()
        end)

        safety.gravityAttachment =
            nil
    end
end

local function rebuildUndergroundForce()
    destroyUndergroundForce()

    if replay.playing
        or not safety.underground
        or not rootPart then

        return
    end

    local attachment =
        Instance.new("Attachment")

    attachment.Name =
        "THH_UndergroundAttachment"

    attachment.Parent =
        rootPart

    local force =
        Instance.new("VectorForce")

    force.Name =
        "THH_UndergroundForce"

    force.Attachment0 =
        attachment

    force.RelativeTo =
        Enum.ActuatorRelativeTo.World

    force.ApplyAtCenterOfMass =
        true

    force.Force =
        Vector3.new(
            0,
            rootPart.AssemblyMass
                * workspace.Gravity,
            0
        )

    force.Parent =
        rootPart

    safety.gravityAttachment =
        attachment

    safety.gravityForce =
        force
end

local function setUnderground(value)
    value =
        value == true

    if value
        and (
            replay.playing
            or not rootPart
            or not humanoid
        ) then

        return
    end

    if value then
        safety.underground =
            true

        setCollisionSource(
            "underground",
            true
        )

        humanoid.CameraOffset =
            Vector3.new(
                0,
                safety.depth,
                0
            )

        rootPart.CFrame =
            rootPart.CFrame
            - Vector3.new(
                0,
                safety.depth,
                0
            )

        rootPart.AssemblyLinearVelocity =
            Vector3.zero

        rebuildUndergroundForce()
    else
        if not safety.underground then
            return
        end

        local camera =
            workspace.CurrentCamera

        safety.underground =
            false

        destroyUndergroundForce()

        if humanoid then
            humanoid.CameraOffset =
                Vector3.zero
        end

        setCollisionSource(
            "underground",
            false
        )

        if rootPart
            and camera then

            local rotation =
                rootPart.CFrame
                - rootPart.Position

            rootPart.CFrame =
                CFrame.new(
                    camera.CFrame.Position
                    + Vector3.new(
                        0,
                        2,
                        0
                    )
                )
                * rotation
        end
    end
end

local function disableAntiFling()
    antiFling.enabled =
        false

    antiFling.generation += 1

    for _, connection in ipairs(
        antiFling.connections
    ) do
        disconnect(connection)
    end

    table.clear(
        antiFling.connections
    )

    for part, oldValue in pairs(
        antiFling.originals
    ) do
        if part
            and part.Parent then

            pcall(function()
                part.CanCollide =
                    oldValue
            end)
        end
    end

    table.clear(
        antiFling.originals
    )

    table.clear(
        antiFling.parts
    )
end

local function enableAntiFling()
    if antiFling.enabled then
        return
    end

    antiFling.enabled =
        true

    antiFling.generation += 1

    local generation =
        antiFling.generation

    local function addPart(part)
        if not antiFling.enabled
            or not part:IsA("BasePart") then

            return
        end

        if antiFling.originals[part] == nil then
            antiFling.originals[part] =
                part.CanCollide
        end

        antiFling.parts[part] =
            true

        part.CanCollide =
            false
    end

    local function addCharacter(
        targetPlayer,
        targetCharacter
    )
        if targetPlayer == player then
            return
        end

        for _, object in ipairs(
            targetCharacter:GetDescendants()
        ) do
            if object:IsA("BasePart") then
                addPart(object)
            end
        end

        table.insert(
            antiFling.connections,
            targetCharacter.DescendantAdded:Connect(function(object)

                if object:IsA("BasePart") then
                    addPart(object)
                end
            end)
        )
    end

    local function addPlayer(targetPlayer)
        if targetPlayer == player then
            return
        end

        if targetPlayer.Character then
            addCharacter(
                targetPlayer,
                targetPlayer.Character
            )
        end

        table.insert(
            antiFling.connections,
            targetPlayer.CharacterAdded:Connect(function(targetCharacter)

                if antiFling.enabled then
                    addCharacter(
                        targetPlayer,
                        targetCharacter
                    )
                end
            end)
        )
    end

    for _, targetPlayer in ipairs(
        Players:GetPlayers()
    ) do
        addPlayer(targetPlayer)
    end

    table.insert(
        antiFling.connections,
        Players.PlayerAdded:Connect(function(targetPlayer)
            addPlayer(targetPlayer)
        end)
    )

    task.spawn(function()
        while alive
            and antiFling.enabled
            and antiFling.generation == generation do

            for part in pairs(
                antiFling.parts
            ) do
                if not part
                    or not part.Parent then

                    antiFling.parts[part] =
                        nil

                    antiFling.originals[part] =
                        nil

                elseif part.CanCollide then
                    part.CanCollide =
                        false
                end
            end

            task.wait(0.12)
        end
    end)
end

local function getRoleColor(targetPlayer)
    if hasKnife(targetPlayer) then
        return C.Murderer
    end

    if hasGun(targetPlayer) then
        return C.Sheriff
    end

    return C.Innocent
end

local function removeESP(targetPlayer)
    local highlight =
        esp.highlights[targetPlayer]

    if highlight then
        pcall(function()
            highlight:Destroy()
        end)

        esp.highlights[targetPlayer] =
            nil
    end
end

local function createESP(targetPlayer)
    if not esp.enabled
        or replay.playing
        or targetPlayer == player then

        return
    end

    local targetCharacter =
        targetPlayer.Character

    if not targetCharacter then
        return
    end

    local targetHumanoid =
        targetCharacter:FindFirstChildOfClass(
            "Humanoid"
        )

    if not targetHumanoid
        or targetHumanoid.Health <= 0 then

        return
    end

    local existing =
        esp.highlights[targetPlayer]

    if existing
        and existing.Parent
        and existing.Adornee == targetCharacter then

        local color =
            getRoleColor(
                targetPlayer
            )

        existing.FillColor =
            color

        existing.OutlineColor =
            color

        return
    end

    removeESP(
        targetPlayer
    )

    local color =
        getRoleColor(
            targetPlayer
        )

    local highlight =
        Instance.new("Highlight")

    highlight.Name =
        "THH_ESP_"
        .. targetPlayer.Name

    highlight.Adornee =
        targetCharacter

    highlight.DepthMode =
        Enum.HighlightDepthMode.AlwaysOnTop

    highlight.FillTransparency =
        0.77

    highlight.OutlineTransparency =
        0

    highlight.FillColor =
        color

    highlight.OutlineColor =
        color

    highlight.Parent =
        targetCharacter

    esp.highlights[targetPlayer] =
        highlight
end

local function setupESPCharacter(
    targetPlayer,
    targetCharacter
)
    if targetPlayer == player
        or not targetCharacter then

        return
    end

    removeESP(
        targetPlayer
    )

    disconnect(
        esp.characterConnections[
            targetPlayer
        ]
    )

    esp.characterConnections[targetPlayer] =
        targetCharacter.ChildAdded:Connect(function()

            if esp.enabled
                and not replay.playing then

                task.defer(function()

                    if targetPlayer.Character
                        == targetCharacter then

                        createESP(
                            targetPlayer
                        )
                    end
                end)
            end
        end)

    task.spawn(function()
        local targetHumanoid =
            targetCharacter:WaitForChild(
                "Humanoid",
                10
            )

        targetCharacter:WaitForChild(
            "HumanoidRootPart",
            10
        )

        if not targetHumanoid
            or targetPlayer.Character
                ~= targetCharacter then

            return
        end

        if esp.enabled
            and not replay.playing then

            createESP(
                targetPlayer
            )
        end

        local diedConnection

        diedConnection =
            targetHumanoid.Died:Connect(function()

                removeESP(
                    targetPlayer
                )

                disconnect(
                    diedConnection
                )
            end)
    end)
end

local function setupESPPlayer(targetPlayer)
    if targetPlayer == player then
        return
    end

    disconnect(
        esp.playerConnections[
            targetPlayer
        ]
    )

    esp.playerConnections[targetPlayer] =
        targetPlayer.CharacterAdded:Connect(function(
            newCharacter
        )

            task.defer(function()

                setupESPCharacter(
                    targetPlayer,
                    newCharacter
                )
            end)
        end)

    if targetPlayer.Character then
        setupESPCharacter(
            targetPlayer,
            targetPlayer.Character
        )
    end
end

local function enableESP()
    esp.enabled =
        true

    for _, targetPlayer in ipairs(
        Players:GetPlayers()
    ) do
        if targetPlayer ~= player then
            setupESPPlayer(
                targetPlayer
            )

            createESP(
                targetPlayer
            )
        end
    end
end

local function disableESP()
    esp.enabled =
        false

    for targetPlayer in pairs(
        esp.highlights
    ) do
        removeESP(
            targetPlayer
        )
    end
end

local function removeESPPlayer(targetPlayer)
    removeESP(
        targetPlayer
    )

    disconnect(
        esp.characterConnections[
            targetPlayer
        ]
    )

    disconnect(
        esp.playerConnections[
            targetPlayer
        ]
    )

    esp.characterConnections[
        targetPlayer
    ] = nil

    esp.playerConnections[
        targetPlayer
    ] = nil
end

local function getGunRemote(gun)
    if not gun then
        return nil
    end

    local knifeLocal =
        gun:FindFirstChild(
            "KnifeLocal"
        )

    local beam =
        knifeLocal
        and knifeLocal:FindFirstChild(
            "CreateBeam"
        )

    local remote =
        beam
        and beam:FindFirstChild(
            "RemoteFunction"
        )

    if remote
        and remote:IsA(
            "RemoteFunction"
        ) then

        return remote
    end

    return nil
end

local function fireGunAtPosition(
    gun,
    position
)
    local remote =
        getGunRemote(
            gun
        )

    if remote then
        local success =
            pcall(function()

                remote:InvokeServer(
                    1,
                    position,
                    "AH2"
                )
            end)

        if success then
            return true
        end
    end

    return pcall(function()
        gun:Activate()
    end)
end

local function getPlayerFromPart(part)
    local current =
        part

    while current
        and current ~= workspace do

        if current:IsA("Model") then
            local targetPlayer =
                Players:GetPlayerFromCharacter(
                    current
                )

            if targetPlayer then
                return targetPlayer
            end
        end

        current =
            current.Parent
    end

    return nil
end

local function manualAimShoot(input)
    if replay.playing
        or not sheriff.quickShot
        or sheriff.shooting
        or not character
        or not humanoid then

        return
    end

    local camera =
        workspace.CurrentCamera

    if not camera then
        return
    end

    local screenPosition

    if input.UserInputType
        == Enum.UserInputType.Touch then

        screenPosition =
            Vector2.new(
                input.Position.X,
                input.Position.Y
            )
    else
        screenPosition =
            UIS:GetMouseLocation()
    end

    local ray =
        camera:ViewportPointToRay(
            screenPosition.X,
            screenPosition.Y
        )

    local params =
        RaycastParams.new()

    params.FilterType =
        Enum.RaycastFilterType.Exclude

    params.FilterDescendantsInstances =
        character
        and {character}
        or {}

    local result =
        workspace:Raycast(
            ray.Origin,
            ray.Direction * 3000,
            params
        )

    if not result then
        return
    end

    local targetPlayer =
        getPlayerFromPart(
            result.Instance
        )

    if not targetPlayer
        or targetPlayer == player
        or not hasKnife(
            targetPlayer
        ) then

        return
    end

    local gun =
        getGunTool()

    if not gun then
        return
    end

    sheriff.shooting =
        true

    task.spawn(function()

        if gun.Parent ~= character then
            humanoid:EquipTool(
                gun
            )

            RunService.Heartbeat:Wait()
            RunService.Heartbeat:Wait()
        end

        if gun.Parent == character then
            fireGunAtPosition(
                gun,
                result.Position
            )

            task.wait(0.07)

            pcall(function()
                humanoid:UnequipTools()
            end)
        end

        sheriff.shooting =
            false
    end)
end

local function sendE()
    if UIS:GetFocusedTextBox() then
        return false
    end

    if keypress
        and keyrelease then

        local success =
            pcall(function()

                keypress(0x45)

                task.wait(0.04)

                keyrelease(0x45)
            end)

        if success then
            return true
        end
    end

    if VIM then
        return pcall(function()

            VIM:SendKeyEvent(
                true,
                Enum.KeyCode.E,
                false,
                game
            )

            task.wait(0.04)

            VIM:SendKeyEvent(
                false,
                Enum.KeyCode.E,
                false,
                game
            )
        end)
    end

    return false
end

local function sendRightClick()
    if not VIM
        or UIS:GetFocusedTextBox() then

        return false
    end

    local position =
        UIS:GetMouseLocation()

    return pcall(function()

        VIM:SendMouseButtonEvent(
            position.X,
            position.Y,
            1,
            true,
            game,
            0
        )

        task.wait(0.04)

        VIM:SendMouseButtonEvent(
            position.X,
            position.Y,
            1,
            false,
            game,
            0
        )
    end)
end

local function isThrowButton(object)
    if not object
        or not object:IsA("GuiButton") then

        return false
    end

    local name =
        string.lower(
            object.Name
        )

    local text = ""

    if object:IsA("TextButton") then
        text =
            string.lower(
                object.Text
            )
    end

    return name:find(
        "throw",
        1,
        true
    ) ~= nil
        or text:find(
            "throw",
            1,
            true
        ) ~= nil
end

local function findThrowButton()
    local cached =
        murderer.cachedThrowButton

    if cached
        and cached.Parent
        and cached.Visible then

        return cached
    end

    murderer.cachedThrowButton =
        nil

    for _, object in ipairs(
        playerGui:GetDescendants()
    ) do
        if isThrowButton(object)
            and object.Visible then

            murderer.cachedThrowButton =
                object

            return object
        end
    end

    return nil
end

local function pressMobileThrow()
    if not VIM then
        return false
    end

    local button =
        findThrowButton()

    if not button then
        return false
    end

    local center =
        button.AbsolutePosition
        + button.AbsoluteSize / 2

    return pcall(function()

        VIM:SendMouseButtonEvent(
            center.X,
            center.Y,
            0,
            true,
            game,
            0
        )

        task.wait(0.04)

        VIM:SendMouseButtonEvent(
            center.X,
            center.Y,
            0,
            false,
            game,
            0
        )
    end)
end

local function throwKnifeOnce()
    if replay.playing
        or murderer.throwing
        or not humanoid
        or not character then

        return false
    end

    local knife =
        getKnifeTool()

    if not knife then
        return false
    end

    murderer.throwing =
        true

    if knife.Parent ~= character then
        humanoid:EquipTool(
            knife
        )

        RunService.Heartbeat:Wait()
    end

    local success = false

    local phoneOnly =
        UIS.TouchEnabled
        and not UIS.KeyboardEnabled
        and not UIS.MouseEnabled

    if phoneOnly then
        success =
            pressMobileThrow()

        if not success then
            success =
                sendE()
        end
    else
        success =
            sendRightClick()

        if not success then
            success =
                sendE()
        end
    end

    murderer.throwing =
        false

    return success
end

local function normalizedName(name)
    return string.lower(
        tostring(name or "")
    ):gsub(
        "[%s_%-%.]",
        ""
    )
end

local function isGunDrop(object)
    if not object then
        return false
    end

    local name =
        normalizedName(
            object.Name
        )

    return name == "gundrop"
        or name == "droppedgun"
        or name == "dropgun"
        or name == "droppedrevolver"
end

local function scanGunDrop()
    for _, object in ipairs(
        workspace:GetDescendants()
    ) do
        if isGunDrop(object) then
            gunPickup.drop =
                object

            return
        end
    end
end

local function getObjectPart(object)
    if not object then
        return nil
    end

    if object:IsA("BasePart") then
        return object
    end

    return object:FindFirstChildWhichIsA(
        "BasePart",
        true
    )
end

local function pickupGun()
    if replay.playing
        or gunPickup.busy
        or getGunTool()
        or not rootPart then

        return false
    end

    local drop =
        gunPickup.drop

    if not drop
        or not drop.Parent then

        scanGunDrop()

        drop =
            gunPickup.drop
    end

    local part =
        getObjectPart(
            drop
        )

    if not part then
        return false
    end

    gunPickup.busy =
        true

    local oldCFrame =
        rootPart.CFrame

    rootPart.CFrame =
        part.CFrame
        + Vector3.new(
            0,
            0.4,
            0
        )

    RunService.Heartbeat:Wait()

    if firetouchinterest then
        pcall(function()

            firetouchinterest(
                rootPart,
                part,
                0
            )

            firetouchinterest(
                rootPart,
                part,
                1
            )
        end)
    end

    task.wait(0.05)

    if rootPart
        and rootPart.Parent then

        rootPart.CFrame =
            oldCFrame
    end

    gunPickup.busy =
        false

    return true
end

-- NEW COIN FARM

local function setCoinStatus(text)
    coinFarm.status =
        tostring(text)
end

local function isCoin(object)
    return object
        and object:IsA("BasePart")
        and object.Name == "MainCoin"
end

local function addCoin(coin)
    if not isCoin(coin)
        or coinFarm.lookup[coin] then

        return
    end

    coinFarm.lookup[coin] =
        true

    table.insert(
        coinFarm.coins,
        coin
    )
end

local function removeCoin(coin)
    if not coinFarm.lookup[coin] then
        return
    end

    coinFarm.lookup[coin] =
        nil

    coinFarm.blockedUntil[coin] =
        nil

    if coinFarm.current == coin then
        coinFarm.current =
            nil
    end

    for index = #coinFarm.coins, 1, -1 do
        if coinFarm.coins[index]
            == coin then

            table.remove(
                coinFarm.coins,
                index
            )

            break
        end
    end
end

local function coinVisible(coin)
    return coin
        and coin.Parent
        and coin.Transparency < 0.95
        and coin.LocalTransparencyModifier < 0.95
end

local function coinAvailable(
    coin,
    ignoreBlock
)
    if not coin
        or not coinFarm.lookup[coin]
        or not coin.Parent
        or not coinVisible(coin) then

        return false
    end

    if not ignoreBlock then
        local blocked =
            coinFarm.blockedUntil[coin]

        if blocked
            and blocked > os.clock() then

            return false
        end
    end

    return true
end

local function cleanCoinCache()
    for index = #coinFarm.coins, 1, -1 do
        local coin =
            coinFarm.coins[index]

        if not coin
            or not coin.Parent then

            if coin then
                coinFarm.lookup[coin] =
                    nil

                coinFarm.blockedUntil[coin] =
                    nil
            end

            table.remove(
                coinFarm.coins,
                index
            )
        end
    end
end

local function cacheCoins()
    table.clear(
        coinFarm.coins
    )

    table.clear(
        coinFarm.lookup
    )

    table.clear(
        coinFarm.blockedUntil
    )

    for _, object in ipairs(
        workspace:GetDescendants()
    ) do
        if isCoin(object) then
            addCoin(object)
        end
    end
end

local function availableCoinCount()
    local amount = 0

    for _, coin in ipairs(
        coinFarm.coins
    ) do
        if coinAvailable(
            coin,
            true
        ) then

            amount += 1
        end
    end

    return amount
end

local function nearestCoin()
    if not rootPart then
        return nil
    end

    local best
    local bestDistance =
        math.huge

    local origin =
        rootPart.Position

    for _, coin in ipairs(
        coinFarm.coins
    ) do
        if coinAvailable(coin) then
            local target =
                coin.Position
                - Vector3.new(
                    0,
                    coinFarm.underDistance,
                    0
                )

            local offset =
                target - origin

            local distanceSquared =
                offset.X * offset.X
                + offset.Y * offset.Y
                + offset.Z * offset.Z

            if distanceSquared
                < bestDistance then

                bestDistance =
                    distanceSquared

                best =
                    coin
            end
        end
    end

    return best
end

local function destroyCoinPlatform()
    if coinFarm.platform then
        pcall(function()
            coinFarm.platform:Destroy()
        end)

        coinFarm.platform =
            nil
    end
end

local function ensureCoinPlatform()
    if coinFarm.platform
        and coinFarm.platform.Parent then

        return coinFarm.platform
    end

    local platform =
        Instance.new("Part")

    platform.Name =
        "THH_CoinPlatform"

    platform.Size =
        Vector3.new(
            5,
            0.45,
            5
        )

    platform.Transparency =
        1

    platform.Anchored =
        true

    platform.CanCollide =
        true

    platform.CanTouch =
        false

    platform.CanQuery =
        false

    platform.CastShadow =
        false

    platform.Parent =
        workspace

    coinFarm.platform =
        platform

    return platform
end

local function setFarmPosition(position)
    if not rootPart
        or not rootPart.Parent
        or replay.playing then

        return false
    end

    local rotation =
        rootPart.CFrame
        - rootPart.Position

    rootPart.CFrame =
        CFrame.new(position)
        * rotation

    rootPart.AssemblyLinearVelocity =
        Vector3.zero

    if not movement.spin then
        local angular =
            rootPart.AssemblyAngularVelocity

        rootPart.AssemblyAngularVelocity =
            Vector3.new(
                0,
                angular.Y,
                0
            )
    end

    local platform =
        ensureCoinPlatform()

    platform.CFrame =
        CFrame.new(
            position
            - Vector3.new(
                0,
                3.15,
                0
            )
        )

    return true
end

local function moveFarmTo(
    destination,
    speed,
    stopDistance,
    coin
)
    stopDistance =
        stopDistance or 0.7

    while alive
        and coinFarm.enabled
        and not replay.playing
        and not safety.underground
        and not coinFarm.fullBagDebounce
        and rootPart
        and rootPart.Parent do

        if coin
            and not coinAvailable(
                coin,
                true
            ) then

            return false,
                "gone"
        end

        local position =
            rootPart.Position

        local offset =
            destination
            - position

        local distance =
            offset.Magnitude

        if distance
            <= stopDistance then

            setFarmPosition(
                destination
            )

            return true
        end

        local dt =
            RunService.Heartbeat:Wait()

        local step =
            math.min(
                distance,
                math.max(
                    speed,
                    1
                ) * dt
            )

        if distance > 0 then
            setFarmPosition(
                position
                + offset.Unit
                * step
            )
        end
    end

    return false,
        "stopped"
end

local function touchCoin(coin)
    if not coin
        or not rootPart then

        return
    end

    if firetouchinterest then
        pcall(function()

            firetouchinterest(
                rootPart,
                coin,
                0
            )

            task.wait()

            firetouchinterest(
                rootPart,
                coin,
                1
            )
        end)
    end
end

local function waitForCoinCollected(
    coin,
    timeout
)
    local started =
        os.clock()

    while alive
        and os.clock() - started
            < timeout do

        if not coinAvailable(
            coin,
            true
        ) then

            return true
        end

        task.wait(0.02)
    end

    return not coinAvailable(
        coin,
        true
    )
end

local function collectCoin(coin)
    if not coinAvailable(
        coin
    )
        or not rootPart then

        return false
    end

    coinFarm.current =
        coin

    setCoinStatus(
        "Moving to coin"
    )

    local under =
        coin.Position
        - Vector3.new(
            0,
            coinFarm.underDistance,
            0
        )

    local reachedUnder =
        moveFarmTo(
            under,
            coinFarm.travelSpeed,
            0.65,
            coin
        )

    if not reachedUnder then
        coinFarm.current =
            nil

        return false
    end

    if not coinAvailable(
        coin,
        true
    ) then

        coinFarm.current =
            nil

        return false
    end

    setCoinStatus(
        "Collecting"
    )

    local reachedCoin =
        moveFarmTo(
            coin.Position,
            coinFarm.verticalSpeed,
            1.25,
            coin
        )

    if not reachedCoin
        and not coinAvailable(
            coin,
            true
        ) then

        coinFarm.collected += 1

        coinFarm.current =
            nil

        setCoinStatus(
            "Collected"
        )

        return true
    end

    if not coinAvailable(
        coin,
        true
    ) then

        coinFarm.collected += 1

        coinFarm.current =
            nil

        setCoinStatus(
            "Collected"
        )

        return true
    end

    touchCoin(
        coin
    )

    local collected =
        waitForCoinCollected(
            coin,
            0.22
        )

    if not collected
        and coinAvailable(
            coin,
            true
        ) then

        local originalPosition =
            coin.Position

        local pulsePositions = {
            originalPosition
                + Vector3.new(
                    0,
                    0.75,
                    0
                ),

            originalPosition
                + Vector3.new(
                    0.6,
                    0,
                    0
                ),

            originalPosition
                + Vector3.new(
                    -0.6,
                    0,
                    0
                )
        }

        for _, pulse in ipairs(
            pulsePositions
        ) do
            if not coinFarm.enabled
                or not coinAvailable(
                    coin,
                    true
                ) then

                break
            end

            setFarmPosition(
                pulse
            )

            touchCoin(
                coin
            )

            task.wait(0.035)
        end

        collected =
            waitForCoinCollected(
                coin,
                0.18
            )
    end

    if collected then
        coinFarm.collected += 1

        setCoinStatus(
            "Collected"
        )
    else
        coinFarm.blockedUntil[coin] =
            os.clock() + 1.25

        setCoinStatus(
            "Skipped stuck coin"
        )
    end

    if coinFarm.enabled
        and rootPart
        and rootPart.Parent then

        moveFarmTo(
            under,
            coinFarm.verticalSpeed,
            0.7
        )
    end

    coinFarm.current =
        nil

    return collected
end

local function setCoinFarm(value)
    coinFarm.enabled =
        value == true

    coinFarm.busy =
        false

    coinFarm.current =
        nil

    setCollisionSource(
        "coinFarm",
        coinFarm.enabled
    )

    if coinFarm.enabled then
        cacheCoins()

        ensureCoinPlatform()

        setCoinStatus(
            "Searching"
        )

        if humanoid then
            humanoid.WalkSpeed =
                originalWalkSpeed
        end
    else
        setCoinStatus(
            "Idle"
        )

        destroyCoinPlatform()

        if movement.speedEnabled
            and humanoid
            and not safety.underground
            and not replay.playing then

            humanoid.WalkSpeed =
                movement.speed
        end
    end
end

local function textLooksLikeBagFull(object)
    if not object
        or not (
            object:IsA("TextLabel")
            or object:IsA("TextButton")
            or object:IsA("TextBox")
        ) then

        return false
    end

    local text =
        string.lower(
            tostring(
                object.Text
                or ""
            )
        )
        :gsub("%s+", "")
        :gsub("[^%w]", "")

    return text:find(
        "coinbagfull",
        1,
        true
    ) ~= nil
        or text:find(
            "bagfull",
            1,
            true
        ) ~= nil
end

local function visibleGui(object)
    if not object
        or not object.Parent
        or not object:IsA("GuiObject")
        or not object.Visible then

        return false
    end

    local current =
        object

    while current do
        if current:IsA("GuiObject")
            and not current.Visible then

            return false
        end

        if current:IsA("ScreenGui")
            and not current.Enabled then

            return false
        end

        current =
            current.Parent
    end

    return true
end

local function watchBagText(object)
    if not (
        object:IsA("TextLabel")
        or object:IsA("TextButton")
        or object:IsA("TextBox")
    ) then

        return
    end

    if textLooksLikeBagFull(
        object
    ) then

        coinFarm.cachedBagText =
            object
    end

    if bagTextConnections[object] then
        return
    end

    bagTextConnections[object] =
        object:GetPropertyChangedSignal(
            "Text"
        ):Connect(function()

            if textLooksLikeBagFull(
                object
            ) then

                coinFarm.cachedBagText =
                    object

            elseif coinFarm.cachedBagText
                == object then

                coinFarm.cachedBagText =
                    nil
            end
        end)
end

local function scanBagText()
    for _, object in ipairs(
        playerGui:GetDescendants()
    ) do
        watchBagText(
            object
        )

        if textLooksLikeBagFull(
            object
        ) then

            return object
        end
    end

    return nil
end

local function handleBagFull()
    if replay.playing
        or not coinFarm.enabled
        or coinFarm.fullBagDebounce
        or os.clock()
            < coinFarm.bagCooldownUntil then

        return
    end

    local label =
        coinFarm.cachedBagText

    if not label
        or not label.Parent then

        label =
            scanBagText()

        coinFarm.cachedBagText =
            label
    end

    if not label
        or not visibleGui(
            label
        )
        or not textLooksLikeBagFull(
            label
        ) then

        return
    end

    if label.TextTransparency
        >= 0.95 then

        return
    end

    coinFarm.fullBagDebounce =
        true

    coinFarm.busy =
        false

    coinFarm.current =
        nil

    setCoinStatus(
        "Bag full - resetting"
    )

    destroyCoinPlatform()

    if humanoid
        and humanoid.Health > 0 then

        humanoid.Health =
            0
    end

    task.spawn(function()

        local newCharacter =
            player.CharacterAdded:Wait()

        task.wait(0.45)

        if player.Character
            == newCharacter then

            waitForCharacter()
        end

        coinFarm.cachedBagText =
            nil

        coinFarm.bagCooldownUntil =
            os.clock() + 2

        if coinFarm.enabled then
            cacheCoins()

            setCollisionSource(
                "coinFarm",
                true
            )

            ensureCoinPlatform()

            setCoinStatus(
                "Resumed"
            )
        end

        coinFarm.fullBagDebounce =
            false
    end)
end

-- REPLAY

local function replayStatus(text, color)
    if replay.statusLabel
        and replay.statusLabel.Parent then

        replay.statusLabel.Text =
            text

        replay.statusLabel.TextColor3 =
            color or C.SubText
    end
end

local function safeClone(object)
    if not object then
        return nil
    end

    local old =
        object.Archivable

    local success, clone =
        pcall(function()

            object.Archivable =
                true

            return object:Clone()
        end)

    pcall(function()
        object.Archivable =
            old
    end)

    if success then
        return clone
    end

    return nil
end

local function cleanReplayClone(object)
    for _, descendant in ipairs(
        object:GetDescendants()
    ) do
        if descendant:IsA("Script")
            or descendant:IsA("LocalScript")
            or descendant:IsA("ModuleScript") then

            descendant:Destroy()

        elseif descendant:IsA("ProximityPrompt")
            or descendant:IsA("ClickDetector") then

            descendant:Destroy()

        elseif descendant:IsA("BasePart") then

            descendant.CanCollide =
                false

            descendant.CanTouch =
                false

            descendant.CanQuery =
                false
        end
    end
end

local function isPlayerCharacter(object)
    return object:IsA("Model")
        and Players:GetPlayerFromCharacter(
            object
        ) ~= nil
end

local function snapshotMap()
    local folder =
        Instance.new("Folder")

    folder.Name =
        "THH_ReplayMap"

    for _, object in ipairs(
        workspace:GetChildren()
    ) do
        if object ~= workspace.CurrentCamera
            and not object:IsA("Terrain")
            and not isPlayerCharacter(
                object
            )
            and object.Name ~= "THH_ReplayWorld"
            and object.Name ~= "THH_CoinPlatform" then

            local clone =
                safeClone(
                    object
                )

            if clone then
                cleanReplayClone(
                    clone
                )

                for _, part in ipairs(
                    clone:GetDescendants()
                ) do
                    if part:IsA("BasePart") then
                        part.Anchored =
                            true
                    end
                end

                if clone:IsA("BasePart") then
                    clone.Anchored =
                        true
                end

                clone.Parent =
                    folder
            end
        end
    end

    return folder
end

local function motorKey(motor)
    return motor.Name
        .. "|"
        .. (
            motor.Part0
            and motor.Part0.Name
            or ""
        )
        .. "|"
        .. (
            motor.Part1
            and motor.Part1.Name
            or ""
        )
end

local function captureMotors(model)
    local motors = {}

    for _, object in ipairs(
        model:GetDescendants()
    ) do
        if object:IsA("Motor6D") then
            motors[
                motorKey(object)
            ] =
                object.Transform
        end
    end

    return motors
end

local function captureReplayTemplate(
    take,
    targetPlayer
)
    local id =
        tostring(
            targetPlayer.UserId
        )

    if take.templates[id] then
        return
    end

    local targetCharacter =
        targetPlayer.Character

    if not targetCharacter then
        return
    end

    local clone =
        safeClone(
            targetCharacter
        )

    if not clone then
        return
    end

    cleanReplayClone(
        clone
    )

    clone.Name =
        targetPlayer.Name

    clone.Parent =
        take.templateFolder

    take.templates[id] =
        clone

    take.playerInfo[id] = {
        name = targetPlayer.Name,
        displayName = targetPlayer.DisplayName
    }
end

local function sampleReplay(take)
    local camera =
        workspace.CurrentCamera

    local frame = {
        time =
            os.clock()
            - take.started,

        camera =
            camera
            and camera.CFrame
            or nil,

        fov =
            camera
            and camera.FieldOfView
            or 70,

        players = {}
    }

    for _, targetPlayer in ipairs(
        Players:GetPlayers()
    ) do
        local targetCharacter =
            targetPlayer.Character

        local targetRoot =
            targetCharacter
            and targetCharacter:FindFirstChild(
                "HumanoidRootPart"
            )

        if targetCharacter
            and targetRoot then

            captureReplayTemplate(
                take,
                targetPlayer
            )

            frame.players[
                tostring(
                    targetPlayer.UserId
                )
            ] = {
                cframe =
                    targetRoot.CFrame,

                motors =
                    captureMotors(
                        targetCharacter
                    )
            }
        end
    end

    table.insert(
        take.frames,
        frame
    )

    take.duration =
        frame.time
end

local stopRecording

local function startRecording()
    if replay.recording
        or replay.playing then

        return
    end

    replay.takeCounter += 1

    replayStatus(
        "Preparing recording...",
        C.Accent
    )

    local take = {
        number =
            replay.takeCounter,

        frames = {},

        templates = {},
        playerInfo = {},

        templateFolder =
            Instance.new("Folder"),

        mapTemplate =
            snapshotMap(),

        duration =
            0,

        started =
            os.clock()
    }

    take.templateFolder.Name =
        "THH_ReplayCharacters"

    for _, targetPlayer in ipairs(
        Players:GetPlayers()
    ) do
        captureReplayTemplate(
            take,
            targetPlayer
        )
    end

    replay.currentTake =
        take

    replay.recording =
        true

    replayStatus(
        "● Recording Take "
        .. tostring(
            take.number
        ),
        C.Danger
    )

    task.spawn(function()

        while alive
            and replay.recording
            and replay.currentTake
                == take do

            sampleReplay(
                take
            )

            if take.duration
                >= replay.maxDuration then

                stopRecording()

                break
            end

            task.wait(
                replay.sampleRate
            )
        end
    end)
end

stopRecording = function()
    if not replay.recording then
        return
    end

    replay.recording =
        false

    local take =
        replay.currentTake

    replay.currentTake =
        nil

    if not take then
        return
    end

    if #take.frames < 2 then
        if take.templateFolder then
            take.templateFolder:Destroy()
        end

        if take.mapTemplate then
            take.mapTemplate:Destroy()
        end

        replayStatus(
            "Recording too short",
            C.Danger
        )

        return
    end

    table.insert(
        replay.takes,
        take
    )

    while #replay.takes
        > replay.maxTakes do

        local old =
            table.remove(
                replay.takes,
                1
            )

        if old.templateFolder then
            old.templateFolder:Destroy()
        end

        if old.mapTemplate then
            old.mapTemplate:Destroy()
        end
    end

    replayStatus(
        "Saved Take "
        .. tostring(
            take.number
        )
        .. " • "
        .. string.format(
            "%.1fs",
            take.duration
        ),
        C.Accent
    )
end

local function stopReplay()
    if not replay.playing then
        return
    end

    replay.playing =
        false

    disconnect(
        replay.playbackConnection
    )

    replay.playbackConnection =
        nil

    if replay.playbackFolder then
        replay.playbackFolder:Destroy()

        replay.playbackFolder =
            nil
    end

    local camera =
        workspace.CurrentCamera

    if camera then
        camera.CameraType =
            replay.oldCameraType
            or Enum.CameraType.Custom

        camera.CameraSubject =
            replay.oldCameraSubject

        if replay.oldCameraCFrame then
            camera.CFrame =
                replay.oldCameraCFrame
        end

        if replay.oldCameraFov then
            camera.FieldOfView =
                replay.oldCameraFov
        end
    end

    if humanoid then
        humanoid.WalkSpeed =
            replay.oldWalkSpeed
            or originalWalkSpeed

        if replay.oldJumpPower then
            humanoid.JumpPower =
                replay.oldJumpPower
        end

        if replay.oldJumpHeight then
            humanoid.JumpHeight =
                replay.oldJumpHeight
        end

        if replay.oldAutoRotate
            ~= nil then

            humanoid.AutoRotate =
                replay.oldAutoRotate
        end
    end

    table.clear(
        replay.playbackActors
    )

    if esp.enabled then
        enableESP()
    end

    replayStatus(
        "Replay stopped",
        C.SubText
    )
end

local function addReplayNameTag(
    clone,
    info
)
    local head =
        clone:FindFirstChild(
            "Head"
        )

    if not head then
        return
    end

    local billboard =
        Instance.new("BillboardGui")

    billboard.Name =
        "THH_ReplayName"

    billboard.Size =
        UDim2.fromOffset(
            220,
            46
        )

    billboard.StudsOffset =
        Vector3.new(
            0,
            2.7,
            0
        )

    billboard.AlwaysOnTop =
        true

    billboard.Adornee =
        head

    billboard.Parent =
        head

    local label =
        Instance.new("TextLabel")

    label.Size =
        UDim2.fromScale(
            1,
            1
        )

    label.BackgroundTransparency =
        1

    label.Text =
        info.displayName
        .. "\n@"
        .. info.name

    label.TextColor3 =
        C.Text

    label.TextStrokeTransparency =
        0.4

    label.TextSize =
        14

    label.Font =
        Enum.Font.GothamBold

    label.Parent =
        billboard
end

local function buildReplayActor(
    take,
    id,
    parent
)
    local template =
        take.templates[id]

    if not template then
        return nil
    end

    local clone =
        safeClone(
            template
        )

    if not clone then
        return nil
    end

    cleanReplayClone(
        clone
    )

    clone.Parent =
        parent

    addReplayNameTag(
        clone,
        take.playerInfo[id]
        or {
            name = clone.Name,
            displayName = clone.Name
        }
    )

    local root =
        clone:FindFirstChild(
            "HumanoidRootPart"
        )

    local motors = {}

    for _, object in ipairs(
        clone:GetDescendants()
    ) do
        if object:IsA("BasePart") then
            object.CanCollide =
                false

            object.CanTouch =
                false

            object.CanQuery =
                false
        end

        if object:IsA("Motor6D") then
            motors[
                motorKey(object)
            ] =
                object
        end
    end

    if root then
        root.Anchored =
            true
    end

    local replayHumanoid =
        clone:FindFirstChildOfClass(
            "Humanoid"
        )

    if replayHumanoid then
        replayHumanoid.PlatformStand =
            true

        replayHumanoid.AutoRotate =
            false

        replayHumanoid.DisplayDistanceType =
            Enum.HumanoidDisplayDistanceType.None
    end

    return {
        model = clone,
        root = root,
        motors = motors
    }
end

local function viewRecorded()
    if replay.recording then
        stopRecording()
    end

    if replay.playing then
        stopReplay()
    end

    local take =
        replay.takes[
            #replay.takes
        ]

    if not take
        or #take.frames < 2 then

        replayStatus(
            "No recording saved",
            C.Danger
        )

        return
    end

    replay.playing =
        true

    disableESP()

    local camera =
        workspace.CurrentCamera

    if camera then
        replay.oldCameraType =
            camera.CameraType

        replay.oldCameraSubject =
            camera.CameraSubject

        replay.oldCameraCFrame =
            camera.CFrame

        replay.oldCameraFov =
            camera.FieldOfView
    end

    if humanoid then
        replay.oldWalkSpeed =
            humanoid.WalkSpeed

        replay.oldJumpPower =
            humanoid.JumpPower

        replay.oldJumpHeight =
            humanoid.JumpHeight

        replay.oldAutoRotate =
            humanoid.AutoRotate

        humanoid.WalkSpeed =
            0

        humanoid.JumpPower =
            0

        humanoid.JumpHeight =
            0

        humanoid.AutoRotate =
            false
    end

    local folder =
        Instance.new("Folder")

    folder.Name =
        "THH_ReplayWorld"

    folder.Parent =
        workspace

    replay.playbackFolder =
        folder

    local mapFolder =
        Instance.new("Folder")

    mapFolder.Name =
        "Map"

    mapFolder.Parent =
        folder

    if take.mapTemplate then
        local map =
            safeClone(
                take.mapTemplate
            )

        if map then
            for _, object in ipairs(
                map:GetChildren()
            ) do
                object.Parent =
                    mapFolder
            end

            map:Destroy()

            for _, part in ipairs(
                mapFolder:GetDescendants()
            ) do
                if part:IsA("BasePart") then
                    part.CFrame =
                        part.CFrame
                        + replay.offset
                end
            end
        end
    end

    local actorsFolder =
        Instance.new("Folder")

    actorsFolder.Name =
        "Players"

    actorsFolder.Parent =
        folder

    table.clear(
        replay.playbackActors
    )

    for id in pairs(
        take.templates
    ) do
        replay.playbackActors[id] =
            buildReplayActor(
                take,
                id,
                actorsFolder
            )
    end

    local started =
        os.clock()

    local index = 1

    replayStatus(
        "▶ Playing Take "
        .. tostring(
            take.number
        ),
        C.Accent
    )

    replay.playbackConnection =
        RunService.RenderStepped:Connect(function()

            if not replay.playing then
                return
            end

            local elapsed =
                os.clock()
                - started

            if elapsed
                >= take.duration then

                stopReplay()

                return
            end

            while index
                    < #take.frames - 1
                and take.frames[
                    index + 1
                ].time <= elapsed do

                index += 1
            end

            local frameA =
                take.frames[index]

            local frameB =
                take.frames[
                    math.min(
                        index + 1,
                        #take.frames
                    )
                ]

            local duration =
                math.max(
                    frameB.time
                    - frameA.time,
                    0.001
                )

            local alpha =
                math.clamp(
                    (
                        elapsed
                        - frameA.time
                    )
                    / duration,
                    0,
                    1
                )

            for id, actor in pairs(
                replay.playbackActors
            ) do
                local stateA =
                    frameA.players[id]

                local stateB =
                    frameB.players[id]

                local state =
                    stateA or stateB

                if actor
                    and state then

                    local cf

                    if stateA
                        and stateB then

                        cf =
                            stateA.cframe:Lerp(
                                stateB.cframe,
                                alpha
                            )
                    else
                        cf =
                            state.cframe
                    end

                    actor.model:PivotTo(
                        cf
                        + replay.offset
                    )

                    for key, motor in pairs(
                        actor.motors
                    ) do
                        local a =
                            stateA
                            and stateA.motors[key]

                        local b =
                            stateB
                            and stateB.motors[key]

                        if a
                            and b then

                            motor.Transform =
                                a:Lerp(
                                    b,
                                    alpha
                                )

                        elseif a then
                            motor.Transform =
                                a

                        elseif b then
                            motor.Transform =
                                b
                        end
                    end
                end
            end

            if camera then
                camera.CameraType =
                    Enum.CameraType.Scriptable

                if frameA.camera
                    and frameB.camera then

                    camera.CFrame =
                        frameA.camera:Lerp(
                            frameB.camera,
                            alpha
                        )
                        + replay.offset

                    camera.FieldOfView =
                        frameA.fov
                        + (
                            frameB.fov
                            - frameA.fov
                        )
                        * alpha
                end
            end

            replayStatus(
                "▶ Take "
                .. tostring(
                    take.number
                )
                .. "  "
                .. string.format(
                    "%.1f / %.1fs",
                    elapsed,
                    take.duration
                ),
                C.Accent
            )
        end)
end

local function deleteLastRecording()
    if replay.recording
        or replay.playing then

        return
    end

    local take =
        table.remove(
            replay.takes,
            #replay.takes
        )

    if not take then
        replayStatus(
            "No recording",
            C.Danger
        )

        return
    end

    if take.templateFolder then
        take.templateFolder:Destroy()
    end

    if take.mapTemplate then
        take.mapTemplate:Destroy()
    end

    replayStatus(
        "Deleted Take "
        .. tostring(
            take.number
        ),
        C.SubText
    )
end

local function startRuntime()
    if runtimeStarted then
        return
    end

    runtimeStarted =
        true

    task.spawn(function()
        waitForCharacter()

        if not alive then
            return
        end

        cacheCoins()
        scanGunDrop()
        scanBagText()

        for _, targetPlayer in ipairs(
            Players:GetPlayers()
        ) do
            if targetPlayer
                ~= player then

                setupESPPlayer(
                    targetPlayer
                )
            end
        end

        trackRuntime(
            Players.PlayerAdded:Connect(function(
                targetPlayer
            )

                setupESPPlayer(
                    targetPlayer
                )
            end)
        )

        trackRuntime(
            Players.PlayerRemoving:Connect(function(
                targetPlayer
            )

                removeESPPlayer(
                    targetPlayer
                )
            end)
        )

        trackRuntime(
            workspace.DescendantAdded:Connect(function(object)

                if isCoin(object) then
                    addCoin(
                        object
                    )
                end

                if isGunDrop(object) then
                    gunPickup.drop =
                        object
                end
            end)
        )

        trackRuntime(
            workspace.DescendantRemoving:Connect(function(object)

                if coinFarm.lookup[
                    object
                ] then

                    removeCoin(
                        object
                    )
                end

                if gunPickup.drop
                    == object then

                    gunPickup.drop =
                        nil
                end
            end)
        )

        trackRuntime(
            playerGui.DescendantAdded:Connect(function(object)

                if isThrowButton(
                    object
                ) then

                    murderer.cachedThrowButton =
                        object
                end

                watchBagText(
                    object
                )
            end)
        )

        trackRuntime(
            playerGui.DescendantRemoving:Connect(function(object)

                if murderer.cachedThrowButton
                    == object then

                    murderer.cachedThrowButton =
                        nil
                end

                if coinFarm.cachedBagText
                    == object then

                    coinFarm.cachedBagText =
                        nil
                end

                disconnect(
                    bagTextConnections[
                        object
                    ]
                )

                bagTextConnections[
                    object
                ] = nil
            end)
        )

        trackRuntime(
            UIS.InputBegan:Connect(function(
                input,
                processed
            )
                if processed then
                    return
                end

                if sheriff.quickShot
                    and (
                        input.UserInputType
                            == Enum.UserInputType.MouseButton1
                        or input.UserInputType
                            == Enum.UserInputType.Touch
                    ) then

                    manualAimShoot(
                        input
                    )
                end
            end)
        )

        trackRuntime(
            UIS.JumpRequest:Connect(function()

                if replay.playing then
                    return
                end

                if movement.infiniteJump
                    and humanoid
                    and not safety.underground then

                    humanoid:ChangeState(
                        Enum.HumanoidStateType.Jumping
                    )
                end
            end)
        )

        trackRuntime(
            player.CharacterAdded:Connect(function()

                task.wait(0.3)

                refreshCharacter()

                rebuildCollision()

                sheriff.shooting =
                    false

                murderer.throwing =
                    false

                gunPickup.busy =
                    false

                coinFarm.busy =
                    false

                coinFarm.current =
                    nil

                if movement.spin then
                    rebuildSpin()
                end

                if safety.underground
                    and humanoid
                    and rootPart then

                    humanoid.CameraOffset =
                        Vector3.new(
                            0,
                            safety.depth,
                            0
                        )

                    rootPart.CFrame =
                        rootPart.CFrame
                        - Vector3.new(
                            0,
                            safety.depth,
                            0
                        )

                    rebuildUndergroundForce()
                end

                if coinFarm.enabled then
                    cacheCoins()

                    ensureCoinPlatform()

                    setCoinStatus(
                        "Searching"
                    )
                end

                if movement.speedEnabled
                    and humanoid
                    and not coinFarm.enabled
                    and not safety.underground then

                    humanoid.WalkSpeed =
                        movement.speed
                end
            end)
        )

        trackRuntime(
            RunService.Heartbeat:Connect(function()

                if replay.playing
                    or not humanoid
                    or not rootPart then

                    return
                end

                if movement.speedEnabled
                    and not coinFarm.enabled
                    and not safety.underground then

                    humanoid.WalkSpeed =
                        movement.speed
                end

                if movement.spin then
                    humanoid.PlatformStand =
                        false

                    humanoid.AutoRotate =
                        false

                    local angular =
                        rootPart.AssemblyAngularVelocity

                    if math.abs(
                        angular.X
                    ) > 0.1
                        or math.abs(
                            angular.Z
                        ) > 0.1 then

                        rootPart.AssemblyAngularVelocity =
                            Vector3.new(
                                0,
                                angular.Y,
                                0
                            )
                    end
                end

                if safety.underground then
                    if safety.gravityForce then
                        safety.gravityForce.Force =
                            Vector3.new(
                                0,
                                rootPart.AssemblyMass
                                    * workspace.Gravity,
                                0
                            )
                    end

                    local direction =
                        humanoid.MoveDirection

                    local moveSpeed =
                        movement.speedEnabled
                        and movement.speed
                        or originalWalkSpeed

                    rootPart.AssemblyLinearVelocity =
                        Vector3.new(
                            direction.X
                                * moveSpeed,
                            0,
                            direction.Z
                                * moveSpeed
                        )
                end
            end)
        )

        task.spawn(function()
            while alive do
                if esp.enabled
                    and not replay.playing then

                    for _, targetPlayer in ipairs(
                        Players:GetPlayers()
                    ) do
                        if targetPlayer
                            ~= player then

                            local targetCharacter =
                                targetPlayer.Character

                            local targetHumanoid =
                                targetCharacter
                                and targetCharacter:FindFirstChildOfClass(
                                    "Humanoid"
                                )

                            if targetCharacter
                                and targetHumanoid
                                and targetHumanoid.Health > 0 then

                                local highlight =
                                    esp.highlights[
                                        targetPlayer
                                    ]

                                if not highlight
                                    or not highlight.Parent
                                    or highlight.Adornee
                                        ~= targetCharacter then

                                    createESP(
                                        targetPlayer
                                    )

                                    highlight =
                                        esp.highlights[
                                            targetPlayer
                                        ]
                                end

                                if highlight then
                                    local color =
                                        getRoleColor(
                                            targetPlayer
                                        )

                                    highlight.FillColor =
                                        color

                                    highlight.OutlineColor =
                                        color
                                end
                            else
                                removeESP(
                                    targetPlayer
                                )
                            end
                        end
                    end
                end

                task.wait(0.1)
            end
        end)

        task.spawn(function()
            while alive do
                if murderer.autoThrow
                    and not replay.playing
                    and not murderer.throwing
                    and os.clock()
                        - murderer.lastThrow
                        >= murderer.throwDelay then

                    murderer.lastThrow =
                        os.clock()

                    task.spawn(
                        throwKnifeOnce
                    )
                end

                task.wait(0.02)
            end
        end)

        task.spawn(function()
            while alive do
                if gunPickup.auto
                    and not replay.playing
                    and not gunPickup.busy
                    and not coinFarm.enabled
                    and not getGunTool()
                    and os.clock()
                        - gunPickup.lastAttempt
                        >= 0.08 then

                    gunPickup.lastAttempt =
                        os.clock()

                    pickupGun()
                end

                task.wait(0.04)
            end
        end)

        task.spawn(function()
            while alive do
                if coinFarm.enabled
                    and not replay.playing
                    and not safety.underground
                    and not coinFarm.fullBagDebounce
                    and not coinFarm.busy
                    and rootPart
                    and rootPart.Parent then

                    if os.clock()
                            - coinFarm.lastCacheClean
                            >= 4 then

                        coinFarm.lastCacheClean =
                            os.clock()

                        cleanCoinCache()
                    end

                    local coin =
                        nearestCoin()

                    if coin then
                        coinFarm.busy =
                            true

                        collectCoin(
                            coin
                        )

                        coinFarm.busy =
                            false

                        if coinFarm.enabled
                            and not coinFarm.fullBagDebounce then

                            setCoinStatus(
                                "Searching"
                            )
                        end
                    else
                        setCoinStatus(
                            "Waiting for coins"
                        )

                        task.wait(0.10)
                    end
                else
                    task.wait(0.04)
                end

                task.wait(0.01)
            end
        end)

        task.spawn(function()
            while alive do
                if coinFarm.enabled
                    and not replay.playing then

                    handleBagFull()
                end

                task.wait(0.08)
            end
        end)

        task.spawn(function()
            while alive do
                local counter =
                    coinFarm.counter

                if counter.collected
                    and counter.collected.Parent then

                    counter.collected.Text =
                        tostring(
                            coinFarm.collected
                        )
                end

                if counter.available
                    and counter.available.Parent then

                    counter.available.Text =
                        tostring(
                            availableCoinCount()
                        )
                end

                if counter.target
                    and counter.target.Parent then

                    local target =
                        coinFarm.current

                    if target
                        and target.Parent then

                        local distance = 0

                        if rootPart then
                            distance =
                                (
                                    target.Position
                                    - rootPart.Position
                                ).Magnitude
                        end

                        counter.target.Text =
                            string.format(
                                "%.0f studs",
                                distance
                            )
                    else
                        counter.target.Text =
                            "None"
                    end
                end

                if counter.status
                    and counter.status.Parent then

                    counter.status.Text =
                        coinFarm.status
                end

                task.wait(0.12)
            end
        end)
    end)
end

local buildMainMenu

local function buildDeviceChooser()
    local overlay =
        create(
            "Frame",
            {
                Size = UDim2.fromScale(1, 1),

                BackgroundColor3 =
                    Color3.fromRGB(
                        8,
                        9,
                        12
                    ),

                BackgroundTransparency =
                    0.10,

                BorderSizePixel =
                    0
            },
            gui
        )

    local panel =
        create(
            "Frame",
            {
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
                        0.88,
                        0,
                        0,
                        260
                    ),

                BackgroundColor3 =
                    Color3.fromRGB(
                        52,
                        54,
                        61
                    ),

                BackgroundTransparency =
                    0.14,

                BorderSizePixel =
                    0,

                Active =
                    true
            },
            overlay
        )

    create(
        "UISizeConstraint",
        {
            MinSize =
                Vector2.new(
                    300,
                    250
                ),

            MaxSize =
                Vector2.new(
                    500,
                    260
                )
        },
        panel
    )

    corner(panel, 16)

    stroke(
        panel,
        Color3.fromRGB(
            150,
            153,
            160
        ),
        1,
        0.50
    )

    makeDraggable(panel)

    create(
        "TextLabel",
        {
            Position =
                UDim2.fromOffset(
                    20,
                    18
                ),

            Size =
                UDim2.new(
                    1,
                    -40,
                    0,
                    28
                ),

            BackgroundTransparency =
                1,

            Text =
                "what are you on?",

            TextColor3 =
                C.Text,

            TextSize =
                20,

            Font =
                Enum.Font.GothamBold
        },
        panel
    )

    create(
        "TextLabel",
        {
            Position =
                UDim2.fromOffset(
                    20,
                    47
                ),

            Size =
                UDim2.new(
                    1,
                    -40,
                    0,
                    18
                ),

            BackgroundTransparency =
                1,

            Text =
                "pick your device",

            TextColor3 =
                C.SubText,

            TextSize =
                12,

            Font =
                Enum.Font.Gotham
        },
        panel
    )

    local function deviceCard(
        x,
        title
    )
        local button =
            create(
                "TextButton",
                {
                    Position =
                        UDim2.new(
                            x,
                            0,
                            0,
                            82
                        ),

                    Size =
                        UDim2.new(
                            0.425,
                            0,
                            0,
                            145
                        ),

                    BackgroundColor3 =
                        C.Control,

                    BackgroundTransparency =
                        0.04,

                    BorderSizePixel =
                        0,

                    Text =
                        "",

                    AutoButtonColor =
                        false
                },
                panel
            )

        corner(button, 12)
        stroke(button)
        animateButton(button)

        create(
            "TextLabel",
            {
                Position =
                    UDim2.new(
                        0,
                        0,
                        1,
                        -36
                    ),

                Size =
                    UDim2.new(
                        1,
                        0,
                        0,
                        22
                    ),

                BackgroundTransparency =
                    1,

                Text =
                    title,

                TextColor3 =
                    C.Text,

                TextSize =
                    15,

                Font =
                    Enum.Font.GothamBold
            },
            button
        )

        return button
    end

    local pcButton =
        deviceCard(
            0.05,
            "PC"
        )

    local monitor =
        create(
            "Frame",
            {
                AnchorPoint =
                    Vector2.new(
                        0.5,
                        0
                    ),

                Position =
                    UDim2.new(
                        0.5,
                        0,
                        0,
                        16
                    ),

                Size =
                    UDim2.fromOffset(
                        88,
                        54
                    ),

                BackgroundColor3 =
                    Color3.fromRGB(
                        26,
                        28,
                        33
                    ),

                BorderSizePixel =
                    0
            },
            pcButton
        )

    corner(monitor, 7)

    stroke(
        monitor,
        C.Text,
        2,
        0.14
    )

    local pcScreen =
        create(
            "Frame",
            {
                Position =
                    UDim2.fromOffset(
                        6,
                        6
                    ),

                Size =
                    UDim2.new(
                        1,
                        -12,
                        1,
                        -13
                    ),

                BackgroundColor3 =
                    Color3.fromRGB(
                        49,
                        55,
                        61
                    ),

                BorderSizePixel =
                    0
            },
            monitor
        )

    corner(pcScreen, 4)

    create(
        "UIGradient",
        {
            Rotation = -25,

            Color =
                ColorSequence.new({
                    ColorSequenceKeypoint.new(
                        0,
                        Color3.fromRGB(
                            43,
                            48,
                            54
                        )
                    ),

                    ColorSequenceKeypoint.new(
                        1,
                        Color3.fromRGB(
                            79,
                            88,
                            98
                        )
                    )
                })
        },
        pcScreen
    )

    local stand =
        create(
            "Frame",
            {
                AnchorPoint =
                    Vector2.new(
                        0.5,
                        0
                    ),

                Position =
                    UDim2.new(
                        0.5,
                        0,
                        0,
                        70
                    ),

                Size =
                    UDim2.fromOffset(
                        7,
                        16
                    ),

                BackgroundColor3 =
                    C.Text,

                BorderSizePixel =
                    0
            },
            pcButton
        )

    corner(stand, 2)

    local standBase =
        create(
            "Frame",
            {
                AnchorPoint =
                    Vector2.new(
                        0.5,
                        0
                    ),

                Position =
                    UDim2.new(
                        0.5,
                        0,
                        0,
                        84
                    ),

                Size =
                    UDim2.fromOffset(
                        40,
                        6
                    ),

                BackgroundColor3 =
                    C.Text,

                BorderSizePixel =
                    0
            },
            pcButton
        )

    corner(standBase, 3)

    local keyboard =
        create(
            "Frame",
            {
                AnchorPoint =
                    Vector2.new(
                        0.5,
                        0
                    ),

                Position =
                    UDim2.new(
                        0.5,
                        0,
                        0,
                        94
                    ),

                Size =
                    UDim2.fromOffset(
                        58,
                        7
                    ),

                BackgroundColor3 =
                    C.SubText,

                BorderSizePixel =
                    0
            },
            pcButton
        )

    corner(keyboard, 2)

    local phoneButton =
        deviceCard(
            0.525,
            "PHONE"
        )

    local phone =
        create(
            "Frame",
            {
                AnchorPoint =
                    Vector2.new(
                        0.5,
                        0
                    ),

                Position =
                    UDim2.new(
                        0.5,
                        0,
                        0,
                        12
                    ),

                Size =
                    UDim2.fromOffset(
                        46,
                        89
                    ),

                BackgroundColor3 =
                    Color3.fromRGB(
                        26,
                        28,
                        33
                    ),

                BorderSizePixel =
                    0
            },
            phoneButton
        )

    corner(phone, 10)

    stroke(
        phone,
        C.Text,
        2,
        0.12
    )

    local phoneScreen =
        create(
            "Frame",
            {
                Position =
                    UDim2.fromOffset(
                        4,
                        9
                    ),

                Size =
                    UDim2.new(
                        1,
                        -8,
                        1,
                        -18
                    ),

                BackgroundColor3 =
                    Color3.fromRGB(
                        49,
                        55,
                        61
                    ),

                BorderSizePixel =
                    0
            },
            phone
        )

    corner(phoneScreen, 6)

    create(
        "UIGradient",
        {
            Rotation = 20,

            Color =
                ColorSequence.new({
                    ColorSequenceKeypoint.new(
                        0,
                        Color3.fromRGB(
                            43,
                            48,
                            54
                        )
                    ),

                    ColorSequenceKeypoint.new(
                        1,
                        Color3.fromRGB(
                            80,
                            88,
                            98
                        )
                    )
                })
        },
        phoneScreen
    )

    local notch =
        create(
            "Frame",
            {
                AnchorPoint =
                    Vector2.new(
                        0.5,
                        0
                    ),

                Position =
                    UDim2.new(
                        0.5,
                        0,
                        0,
                        4
                    ),

                Size =
                    UDim2.fromOffset(
                        18,
                        4
                    ),

                BackgroundColor3 =
                    C.Text,

                BorderSizePixel =
                    0
            },
            phone
        )

    corner(notch, 3)

    local homeBar =
        create(
            "Frame",
            {
                AnchorPoint =
                    Vector2.new(
                        0.5,
                        1
                    ),

                Position =
                    UDim2.new(
                        0.5,
                        0,
                        1,
                        -4
                    ),

                Size =
                    UDim2.fromOffset(
                        16,
                        3
                    ),

                BackgroundColor3 =
                    C.Text,

                BorderSizePixel =
                    0
            },
            phone
        )

    corner(homeBar, 3)

    track(
        pcButton.MouseButton1Click:Connect(function()

            overlay:Destroy()

            if blur then
                blur:Destroy()
                blur = nil
            end

            startRuntime()

            buildMainMenu(
                "PC"
            )
        end)
    )

    track(
        phoneButton.MouseButton1Click:Connect(function()

            overlay:Destroy()

            if blur then
                blur:Destroy()
                blur = nil
            end

            startRuntime()

            buildMainMenu(
                "Phone"
            )
        end)
    )
end

buildMainMenu = function(deviceMode)
    local isPhone =
        deviceMode == "Phone"

    local menu =
        create(
            "Frame",
            {
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
                    isPhone
                    and UDim2.new(
                        1,
                        -12,
                        1,
                        -20
                    )
                    or UDim2.fromOffset(
                        750,
                        500
                    ),

                BackgroundColor3 =
                    C.Background,

                BackgroundTransparency =
                    isPhone
                    and 0.12
                    or 0.09,

                BorderSizePixel =
                    0,

                ClipsDescendants =
                    true,

                Active =
                    true
            },
            gui
        )

    corner(menu, 14)
    stroke(menu)

    local headerHeight =
        isPhone
        and 52
        or 58

    local header =
        create(
            "Frame",
            {
                Size =
                    UDim2.new(
                        1,
                        0,
                        0,
                        headerHeight
                    ),

                BackgroundColor3 =
                    C.Background2,

                BackgroundTransparency =
                    0.08,

                BorderSizePixel =
                    0,

                Active =
                    true
            },
            menu
        )

    makeDraggable(
        menu,
        header
    )

    local icon =
        create(
            "ImageLabel",
            {
                Position =
                    UDim2.new(
                        0,
                        10,
                        0.5,
                        -18
                    ),

                Size =
                    UDim2.fromOffset(
                        36,
                        36
                    ),

                BackgroundColor3 =
                    C.Control,

                BackgroundTransparency =
                    0.05,

                BorderSizePixel =
                    0,

                Image =
                    MM2_ICON
            },
            header
        )

    corner(icon, 9)

    create(
        "TextLabel",
        {
            Position =
                UDim2.fromOffset(
                    57,
                    8
                ),

            Size =
                UDim2.fromOffset(
                    190,
                    21
                ),

            BackgroundTransparency =
                1,

            Text =
                "THH HUB",

            TextColor3 =
                C.Text,

            TextSize =
                18,

            Font =
                Enum.Font.GothamBold,

            TextXAlignment =
                Enum.TextXAlignment.Left
        },
        header
    )

    create(
        "TextLabel",
        {
            Position =
                UDim2.fromOffset(
                    57,
                    30
                ),

            Size =
                UDim2.fromOffset(
                    190,
                    15
                ),

            BackgroundTransparency =
                1,

            Text =
                "MM2",

            TextColor3 =
                C.SubText,

            TextSize =
                10,

            Font =
                Enum.Font.Gotham,

            TextXAlignment =
                Enum.TextXAlignment.Left
        },
        header
    )

    local minimize =
        create(
            "TextButton",
            {
                AnchorPoint =
                    Vector2.new(
                        1,
                        0.5
                    ),

                Position =
                    UDim2.new(
                        1,
                        -45,
                        0.5,
                        0
                    ),

                Size =
                    UDim2.fromOffset(
                        28,
                        28
                    ),

                BackgroundColor3 =
                    C.Control,

                BorderSizePixel =
                    0,

                Text =
                    "—",

                TextColor3 =
                    C.Text,

                TextSize =
                    15,

                Font =
                    Enum.Font.GothamBold
            },
            header
        )

    corner(minimize, 8)

    local close =
        create(
            "TextButton",
            {
                AnchorPoint =
                    Vector2.new(
                        1,
                        0.5
                    ),

                Position =
                    UDim2.new(
                        1,
                        -10,
                        0.5,
                        0
                    ),

                Size =
                    UDim2.fromOffset(
                        28,
                        28
                    ),

                BackgroundColor3 =
                    C.Control,

                BorderSizePixel =
                    0,

                Text =
                    "×",

                TextColor3 =
                    C.Danger,

                TextSize =
                    18,

                Font =
                    Enum.Font.GothamBold
            },
            header
        )

    corner(close, 8)

    local navHolder
    local pageHolder

    if isPhone then
        navHolder =
            create(
                "Frame",
                {
                    Position =
                        UDim2.new(
                            0,
                            7,
                            0,
                            headerHeight + 6
                        ),

                    Size =
                        UDim2.new(
                            1,
                            -14,
                            0,
                            112
                        ),

                    BackgroundTransparency =
                        1
                },
                menu
            )

        create(
            "UIGridLayout",
            {
                CellSize =
                    UDim2.new(
                        0.24,
                        -3,
                        0,
                        33
                    ),

                CellPadding =
                    UDim2.fromOffset(
                        5,
                        5
                    ),

                FillDirectionMaxCells =
                    4
            },
            navHolder
        )

        pageHolder =
            create(
                "Frame",
                {
                    Position =
                        UDim2.new(
                            0,
                            7,
                            0,
                            headerHeight + 124
                        ),

                    Size =
                        UDim2.new(
                            1,
                            -14,
                            1,
                            -(headerHeight + 131)
                        ),

                    BackgroundTransparency =
                        1
                },
                menu
            )
    else
        navHolder =
            create(
                "Frame",
                {
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
                            160,
                            1,
                            -headerHeight
                        ),

                    BackgroundColor3 =
                        C.Sidebar,

                    BackgroundTransparency =
                        0.08,

                    BorderSizePixel =
                        0
                },
                menu
            )

        create(
            "UIListLayout",
            {
                Padding =
                    UDim.new(
                        0,
                        5
                    ),

                HorizontalAlignment =
                    Enum.HorizontalAlignment.Center
            },
            navHolder
        )

        create(
            "UIPadding",
            {
                PaddingTop =
                    UDim.new(
                        0,
                        9
                    )
            },
            navHolder
        )

        pageHolder =
            create(
                "Frame",
                {
                    Position =
                        UDim2.new(
                            0,
                            160,
                            0,
                            headerHeight
                        ),

                    Size =
                        UDim2.new(
                            1,
                            -160,
                            1,
                            -headerHeight
                        ),

                    BackgroundTransparency =
                        1
                },
                menu
            )
    end

    local pages = {}
    local navs = {}

    local function page(name)
        local object =
            create(
                "ScrollingFrame",
                {
                    Size =
                        UDim2.fromScale(
                            1,
                            1
                        ),

                    BackgroundTransparency =
                        1,

                    BorderSizePixel =
                        0,

                    Visible =
                        false,

                    AutomaticCanvasSize =
                        Enum.AutomaticSize.Y,

                    CanvasSize =
                        UDim2.fromOffset(
                            0,
                            0
                        ),

                    ScrollBarThickness =
                        isPhone
                        and 2
                        or 3,

                    ScrollBarImageColor3 =
                        C.Accent
                },
                pageHolder
            )

        create(
            "UIListLayout",
            {
                Padding =
                    UDim.new(
                        0,
                        9
                    )
            },
            object
        )

        create(
            "UIPadding",
            {
                PaddingLeft =
                    UDim.new(
                        0,
                        10
                    ),

                PaddingRight =
                    UDim.new(
                        0,
                        10
                    ),

                PaddingTop =
                    UDim.new(
                        0,
                        10
                    ),

                PaddingBottom =
                    UDim.new(
                        0,
                        10
                    )
            },
            object
        )

        pages[name] =
            object

        return object
    end

    local function show(name)
        for pageName, object in pairs(
            pages
        ) do
            object.Visible =
                pageName == name
        end

        for navName, button in pairs(
            navs
        ) do
            local selected =
                navName == name

            tween(
                button,
                0.12,
                {
                    BackgroundColor3 =
                        selected
                        and C.AccentDark
                        or C.Control,

                    TextColor3 =
                        selected
                        and C.Accent
                        or C.SubText
                }
            )
        end
    end

    local function nav(name)
        local button =
            create(
                "TextButton",
                {
                    Size =
                        isPhone
                        and UDim2.fromScale(
                            1,
                            1
                        )
                        or UDim2.fromOffset(
                            137,
                            36
                        ),

                    BackgroundColor3 =
                        C.Control,

                    BackgroundTransparency =
                        0.05,

                    BorderSizePixel =
                        0,

                    Text =
                        name,

                    TextColor3 =
                        C.SubText,

                    TextSize =
                        isPhone
                        and 9
                        or 11,

                    Font =
                        Enum.Font.GothamMedium,

                    AutoButtonColor =
                        false
                },
                navHolder
            )

        corner(button, 8)

        navs[name] =
            button

        track(
            button.MouseButton1Click:Connect(function()
                show(name)
            end)
        )
    end

    local function section(
        parent,
        title
    )
        local card =
            create(
                "Frame",
                {
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
                        C.Card,

                    BackgroundTransparency =
                        0.21,

                    BorderSizePixel =
                        0
                },
                parent
            )

        corner(card, 11)
        stroke(card)

        local holder =
            create(
                "Frame",
                {
                    Size =
                        UDim2.new(
                            1,
                            0,
                            0,
                            0
                        ),

                    AutomaticSize =
                        Enum.AutomaticSize.Y,

                    BackgroundTransparency =
                        1
                },
                card
            )

        create(
            "UIListLayout",
            {
                Padding =
                    UDim.new(
                        0,
                        8
                    )
            },
            holder
        )

        create(
            "UIPadding",
            {
                PaddingLeft =
                    UDim.new(
                        0,
                        11
                    ),

                PaddingRight =
                    UDim.new(
                        0,
                        11
                    ),

                PaddingTop =
                    UDim.new(
                        0,
                        10
                    ),

                PaddingBottom =
                    UDim.new(
                        0,
                        10
                    )
            },
            holder
        )

        create(
            "TextLabel",
            {
                Size =
                    UDim2.new(
                        1,
                        0,
                        0,
                        21
                    ),

                BackgroundTransparency =
                    1,

                Text =
                    title,

                TextColor3 =
                    C.Text,

                TextSize =
                    14,

                Font =
                    Enum.Font.GothamBold,

                TextXAlignment =
                    Enum.TextXAlignment.Left
            },
            holder
        )

        return holder
    end

    local function action(
        parent,
        title,
        callback,
        danger
    )
        local button =
            create(
                "TextButton",
                {
                    Size =
                        UDim2.new(
                            1,
                            0,
                            0,
                            42
                        ),

                    BackgroundColor3 =
                        danger
                        and Color3.fromRGB(
                            74,
                            38,
                            42
                        )
                        or C.Control,

                    BackgroundTransparency =
                        0.04,

                    BorderSizePixel =
                        0,

                    Text =
                        title,

                    TextColor3 =
                        danger
                        and C.Danger
                        or C.Text,

                    TextSize =
                        12,

                    Font =
                        Enum.Font.GothamMedium,

                    AutoButtonColor =
                        false
                },
                parent
            )

        corner(button, 8)
        animateButton(button)

        track(
            button.MouseButton1Click:Connect(function()
                callback()
            end)
        )

        return button
    end

    local function toggle(
        parent,
        title,
        default,
        callback
    )
        local state =
            default == true

        local button =
            create(
                "TextButton",
                {
                    Size =
                        UDim2.new(
                            1,
                            0,
                            0,
                            42
                        ),

                    BackgroundColor3 =
                        C.Control,

                    BackgroundTransparency =
                        0.04,

                    BorderSizePixel =
                        0,

                    Text =
                        "",

                    AutoButtonColor =
                        false
                },
                parent
            )

        corner(button, 8)

        create(
            "TextLabel",
            {
                Position =
                    UDim2.fromOffset(
                        11,
                        0
                    ),

                Size =
                    UDim2.new(
                        1,
                        -70,
                        1,
                        0
                    ),

                BackgroundTransparency =
                    1,

                Text =
                    title,

                TextColor3 =
                    C.Text,

                TextSize =
                    12,

                Font =
                    Enum.Font.GothamMedium,

                TextXAlignment =
                    Enum.TextXAlignment.Left
            },
            button
        )

        local switch =
            create(
                "Frame",
                {
                    AnchorPoint =
                        Vector2.new(
                            1,
                            0.5
                        ),

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
                            or 40,

                            isPhone
                            and 24
                            or 22
                        ),

                    BorderSizePixel =
                        0
                },
                button
            )

        corner(switch, 100)

        local size =
            isPhone
            and 18
            or 16

        local dot =
            create(
                "Frame",
                {
                    AnchorPoint =
                        Vector2.new(
                            0.5,
                            0.5
                        ),

                    Size =
                        UDim2.fromOffset(
                            size,
                            size
                        ),

                    BorderSizePixel =
                        0
                },
                switch
            )

        corner(dot, 100)

        local controller = {}

        local function refresh()
            switch.BackgroundColor3 =
                state
                and C.AccentDark
                or C.Stroke

            dot.BackgroundColor3 =
                state
                and C.Accent
                or C.SubText

            dot.Position =
                state
                and UDim2.new(
                    1,
                    -(size / 2 + 3),
                    0.5,
                    0
                )
                or UDim2.new(
                    0,
                    size / 2 + 3,
                    0.5,
                    0
                )
        end

        function controller:Set(
            value,
            call
        )
            state =
                value == true

            refresh()

            if call ~= false then
                callback(state)
            end
        end

        function controller:Get()
            return state
        end

        refresh()

        track(
            button.MouseButton1Click:Connect(function()

                controller:Set(
                    not state
                )
            end)
        )

        return controller
    end

    local function slider(
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
            create(
                "Frame",
                {
                    Size =
                        UDim2.new(
                            1,
                            0,
                            0,
                            isPhone
                            and 68
                            or 60
                        ),

                    BackgroundColor3 =
                        C.Control,

                    BackgroundTransparency =
                        0.04,

                    BorderSizePixel =
                        0
                },
                parent
            )

        corner(holder, 8)

        create(
            "TextLabel",
            {
                Position =
                    UDim2.fromOffset(
                        11,
                        7
                    ),

                Size =
                    UDim2.new(
                        1,
                        -85,
                        0,
                        20
                    ),

                BackgroundTransparency =
                    1,

                Text =
                    title,

                TextColor3 =
                    C.Text,

                TextSize =
                    12,

                Font =
                    Enum.Font.GothamMedium,

                TextXAlignment =
                    Enum.TextXAlignment.Left
            },
            holder
        )

        local valueText =
            create(
                "TextLabel",
                {
                    AnchorPoint =
                        Vector2.new(
                            1,
                            0
                        ),

                    Position =
                        UDim2.new(
                            1,
                            -11,
                            0,
                            7
                        ),

                    Size =
                        UDim2.fromOffset(
                            70,
                            20
                        ),

                    BackgroundTransparency =
                        1,

                    TextColor3 =
                        C.Accent,

                    TextSize =
                        12,

                    Font =
                        Enum.Font.GothamBold,

                    TextXAlignment =
                        Enum.TextXAlignment.Right
                },
                holder
            )

        local bar =
            create(
                "Frame",
                {
                    Position =
                        UDim2.new(
                            0,
                            13,
                            1,
                            isPhone
                            and -22
                            or -18
                        ),

                    Size =
                        UDim2.new(
                            1,
                            -26,
                            0,
                            isPhone
                            and 10
                            or 7
                        ),

                    BackgroundColor3 =
                        C.Stroke,

                    BorderSizePixel =
                        0
                },
                holder
            )

        corner(bar, 100)

        local fill =
            create(
                "Frame",
                {
                    BackgroundColor3 =
                        C.Accent,

                    BorderSizePixel =
                        0
                },
                bar
            )

        corner(fill, 100)

        local knob =
            create(
                "Frame",
                {
                    AnchorPoint =
                        Vector2.new(
                            0.5,
                            0.5
                        ),

                    Size =
                        UDim2.fromOffset(
                            isPhone
                            and 22
                            or 18,

                            isPhone
                            and 22
                            or 18
                        ),

                    BackgroundColor3 =
                        C.Text,

                    BorderSizePixel =
                        0
                },
                bar
            )

        corner(knob, 100)

        stroke(
            knob,
            C.Accent,
            2,
            0
        )

        local function refresh()
            local percent =
                (value - minimum)
                / (
                    maximum
                    - minimum
                )

            fill.Size =
                UDim2.fromScale(
                    percent,
                    1
                )

            knob.Position =
                UDim2.new(
                    percent,
                    0,
                    0.5,
                    0
                )

            valueText.Text =
                tostring(value)
        end

        local dragging =
            false

        local function update(x)
            local percent =
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
                    * percent
                    + 0.5
                )

            refresh()

            callback(
                value
            )
        end

        local function begin(input)
            if input.UserInputType
                    == Enum.UserInputType.MouseButton1
                or input.UserInputType
                    == Enum.UserInputType.Touch then

                dragging =
                    true

                update(
                    input.Position.X
                )
            end
        end

        track(
            bar.InputBegan:Connect(
                begin
            )
        )

        track(
            knob.InputBegan:Connect(
                begin
            )
        )

        track(
            UIS.InputChanged:Connect(function(input)

                if dragging
                    and (
                        input.UserInputType
                            == Enum.UserInputType.MouseMovement
                        or input.UserInputType
                            == Enum.UserInputType.Touch
                    ) then

                    update(
                        input.Position.X
                    )
                end
            end)
        )

        track(
            UIS.InputEnded:Connect(function(input)

                if input.UserInputType
                        == Enum.UserInputType.MouseButton1
                    or input.UserInputType
                        == Enum.UserInputType.Touch then

                    dragging =
                        false
                end
            end)
        )

        refresh()
    end

    local Home =
        page("Home")

    local Sheriff =
        page("Sheriff")

    local Murderer =
        page("Murderer")

    local Innocent =
        page("Innocent")

    local Coin =
        page("Coin Grab")

    local Replay =
        page("Replay Studio")

    local Safety =
        page("Safety")

    local Updates =
        page("Updates")

    local Utility =
        page("Utility")

    nav("Home")
    nav("Sheriff")
    nav("Murderer")
    nav("Innocent")
    nav("Coin Grab")
    nav("Replay Studio")
    nav("Safety")
    nav("Updates")
    nav("Utility")

    local homeSection =
        section(
            Home,
            "THH HUB"
        )

    create(
        "TextLabel",
        {
            Size =
                UDim2.new(
                    1,
                    0,
                    0,
                    28
                ),

            BackgroundTransparency =
                1,

            Text =
                "welcome "
                .. player.DisplayName,

            TextColor3 =
                C.Accent,

            TextSize =
                12,

            Font =
                Enum.Font.GothamMedium,

            TextXAlignment =
                Enum.TextXAlignment.Left
        },
        homeSection
    )

    local sheriffSection =
        section(
            Sheriff,
            "Sheriff"
        )

    toggle(
        sheriffSection,
        "Aim Bot",
        sheriff.quickShot,
        function(value)

            sheriff.quickShot =
                value
        end
    )

    action(
        sheriffSection,
        "Pick Up Gun Drop",
        pickupGun
    )

    local sheriffAutoGun
    local innocentAutoGun

    local function syncAutoGun(
        value,
        source
    )
        gunPickup.auto =
            value

        if sheriffAutoGun
            and source
                ~= sheriffAutoGun then

            sheriffAutoGun:Set(
                value,
                false
            )
        end

        if innocentAutoGun
            and source
                ~= innocentAutoGun then

            innocentAutoGun:Set(
                value,
                false
            )
        end
    end

    sheriffAutoGun =
        toggle(
            sheriffSection,
            "Auto Pick Up Gun",
            gunPickup.auto,
            function(value)

                syncAutoGun(
                    value,
                    sheriffAutoGun
                )
            end
        )

    local murdererSection =
        section(
            Murderer,
            "Murderer"
        )

    action(
        murdererSection,
        "Throw Knife",
        function()

            task.spawn(
                throwKnifeOnce
            )
        end
    )

    toggle(
        murdererSection,
        "Auto Throw Knife",
        murderer.autoThrow,
        function(value)

            murderer.autoThrow =
                value
        end
    )

    slider(
        murdererSection,
        "Throw Delay",
        5,
        100,
        math.floor(
            murderer.throwDelay
            * 100
        ),
        function(value)

            murderer.throwDelay =
                value / 100
        end
    )

    local espSection =
        section(
            Innocent,
            "ESP"
        )

    toggle(
        espSection,
        "ESP",
        esp.enabled,
        function(value)

            if value then
                enableESP()
            else
                disableESP()
            end
        end
    )

    local gunSection =
        section(
            Innocent,
            "Gun"
        )

    action(
        gunSection,
        "Pick Up Gun",
        pickupGun
    )

    innocentAutoGun =
        toggle(
            gunSection,
            "Auto Pick Up Gun",
            gunPickup.auto,
            function(value)

                syncAutoGun(
                    value,
                    innocentAutoGun
                )
            end
        )

    local movementSection =
        section(
            Innocent,
            "Movement"
        )

    toggle(
        movementSection,
        "Speed",
        movement.speedEnabled,
        setSpeedEnabled
    )

    slider(
        movementSection,
        "Speed",
        16,
        250,
        movement.speed,
        setSpeed
    )

    toggle(
        movementSection,
        "Infinite Jump",
        movement.infiniteJump,
        function(value)

            movement.infiniteJump =
                value
        end
    )

    toggle(
        movementSection,
        "Noclip",
        collision.sources.noclip,
        function(value)

            setCollisionSource(
                "noclip",
                value
            )
        end
    )

    toggle(
        movementSection,
        "Spin Bot",
        movement.spin,
        setSpinEnabled
    )

    slider(
        movementSection,
        "Spin Speed",
        30,
        1500,
        movement.spinSpeed,
        setSpinSpeed
    )

    local coinSection =
        section(
            Coin,
            "Coin Farm"
        )

    toggle(
        coinSection,
        "Coin Farm",
        coinFarm.enabled,
        setCoinFarm
    )

    slider(
        coinSection,
        "Farm Speed",
        10,
        70,
        coinFarm.travelSpeed,
        function(value)

            coinFarm.travelSpeed =
                value

            coinFarm.verticalSpeed =
                math.max(
                    100,
                    value * 5
                )
        end
    )

    local counterCard =
        create(
            "Frame",
            {
                Size =
                    UDim2.new(
                        1,
                        0,
                        0,
                        128
                    ),

                BackgroundColor3 =
                    C.Control,

                BackgroundTransparency =
                    0.04,

                BorderSizePixel =
                    0
            },
            coinSection
        )

    corner(
        counterCard,
        10
    )

    local counterGrid =
        create(
            "Frame",
            {
                Position =
                    UDim2.fromOffset(
                        10,
                        10
                    ),

                Size =
                    UDim2.new(
                        1,
                        -20,
                        1,
                        -20
                    ),

                BackgroundTransparency =
                    1
            },
            counterCard
        )

    create(
        "UIGridLayout",
        {
            CellSize =
                UDim2.new(
                    0.5,
                    -5,
                    0.5,
                    -5
                ),

            CellPadding =
                UDim2.fromOffset(
                    10,
                    10
                ),

            FillDirectionMaxCells =
                2
        },
        counterGrid
    )

    local function statCard(
        title,
        default
    )
        local card =
            create(
                "Frame",
                {
                    BackgroundColor3 =
                        C.Background2,

                    BackgroundTransparency =
                        0.15,

                    BorderSizePixel =
                        0
                },
                counterGrid
            )

        corner(
            card,
            8
        )

        create(
            "TextLabel",
            {
                Position =
                    UDim2.fromOffset(
                        8,
                        5
                    ),

                Size =
                    UDim2.new(
                        1,
                        -16,
                        0,
                        16
                    ),

                BackgroundTransparency =
                    1,

                Text =
                    title,

                TextColor3 =
                    C.SubText,

                TextSize =
                    9,

                Font =
                    Enum.Font.GothamMedium,

                TextXAlignment =
                    Enum.TextXAlignment.Left
            },
            card
        )

        local value =
            create(
                "TextLabel",
                {
                    Position =
                        UDim2.fromOffset(
                            8,
                            21
                        ),

                    Size =
                        UDim2.new(
                            1,
                            -16,
                            1,
                            -25
                        ),

                    BackgroundTransparency =
                        1,

                    Text =
                        default,

                    TextColor3 =
                        C.Accent,

                    TextSize =
                        14,

                    Font =
                        Enum.Font.GothamBold,

                    TextXAlignment =
                        Enum.TextXAlignment.Left
                },
                card
            )

        return value
    end

    coinFarm.counter.collected =
        statCard(
            "COLLECTED",
            "0"
        )

    coinFarm.counter.available =
        statCard(
            "AVAILABLE",
            "0"
        )

    coinFarm.counter.target =
        statCard(
            "TARGET",
            "None"
        )

    coinFarm.counter.status =
        statCard(
            "STATUS",
            "Idle"
        )

    action(
        coinSection,
        "Refresh Coin Cache",
        function()

            cacheCoins()

            setCoinStatus(
                "Cache refreshed"
            )
        end
    )

    action(
        coinSection,
        "Reset Session Counter",
        function()

            coinFarm.collected =
                0

            setCoinStatus(
                "Counter reset"
            )
        end
    )

    local replaySection =
        section(
            Replay,
            "Replay Studio"
        )

    replay.statusLabel =
        create(
            "TextLabel",
            {
                Size =
                    UDim2.new(
                        1,
                        0,
                        0,
                        42
                    ),

                BackgroundColor3 =
                    C.Control,

                BackgroundTransparency =
                    0.04,

                BorderSizePixel =
                    0,

                Text =
                    "Ready • 0 recordings",

                TextColor3 =
                    C.SubText,

                TextSize =
                    11,

                Font =
                    Enum.Font.GothamMedium
            },
            replaySection
        )

    corner(
        replay.statusLabel,
        8
    )

    action(
        replaySection,
        "Start Recording",
        startRecording
    )

    action(
        replaySection,
        "Stop Recording",
        stopRecording
    )

    action(
        replaySection,
        "View Recorded",
        viewRecorded
    )

    action(
        replaySection,
        "Stop Replay",
        stopReplay
    )

    action(
        replaySection,
        "Delete Last Recording",
        deleteLastRecording,
        true
    )

    local safetySection =
        section(
            Safety,
            "Safety"
        )

    toggle(
        safetySection,
        "Anti Fling",
        antiFling.enabled,
        function(value)

            if value then
                enableAntiFling()
            else
                disableAntiFling()
            end
        end
    )

    toggle(
        safetySection,
        "Underground Safe Mode",
        safety.underground,
        setUnderground
    )

    local updatesSection =
        section(
            Updates,
            "Updates"
        )

    local updates = {
        "completely rebuilt Coin Farm",
        "new Collected / Available / Target / Status counter",
        "Coin Farm now skips stuck coins instead of looping forever",
        "Coin Farm confirms collection before adding to the counter",
        "new Farm Speed slider",
        "new manual Refresh Coin Cache button",
        "new Reset Session Counter button",
        "better MainCoin live cache",
        "better bag-full reset and resume",
        "ESP stays enabled through player respawns",
        "PC and phone device chooser",
        "Replay Studio",
        "Y-only Spin Bot",
        "cached Gun Drop pickup",
        "manual click/tap Aim Bot",
        "cursor-based Knife Throw",
        "Anti Fling",
        "Underground Safe Mode"
    }

    for _, text in ipairs(
        updates
    ) do
        create(
            "TextLabel",
            {
                Size =
                    UDim2.new(
                        1,
                        0,
                        0,
                        0
                    ),

                AutomaticSize =
                    Enum.AutomaticSize.Y,

                BackgroundTransparency =
                    1,

                Text =
                    "• "
                    .. text,

                TextColor3 =
                    C.SubText,

                TextSize =
                    11,

                TextWrapped =
                    true,

                Font =
                    Enum.Font.Gotham,

                TextXAlignment =
                    Enum.TextXAlignment.Left
            },
            updatesSection
        )
    end

    local utilitySection =
        section(
            Utility,
            "Utility"
        )

    action(
        utilitySection,
        "Copy Server ID",
        function()

            if setclipboard then
                pcall(function()
                    setclipboard(
                        game.JobId
                    )
                end)

            elseif toclipboard then
                pcall(function()
                    toclipboard(
                        game.JobId
                    )
                end)
            end
        end
    )

    action(
        utilitySection,
        "Rejoin Server",
        function()

            pcall(function()

                TeleportService:TeleportToPlaceInstance(
                    game.PlaceId,
                    game.JobId,
                    player
                )
            end)
        end
    )

    action(
        utilitySection,
        "Reset Character",
        function()

            if humanoid then
                humanoid.Health =
                    0
            end
        end
    )

    action(
        utilitySection,
        "Unload Menu",
        function()

            if _G.THHGrowersCleanup then
                _G.THHGrowersCleanup()
            end
        end,
        true
    )

    local floating

    if isPhone then
        floating =
            create(
                "ImageButton",
                {
                    AnchorPoint =
                        Vector2.new(
                            1,
                            1
                        ),

                    Position =
                        UDim2.new(
                            1,
                            -12,
                            1,
                            -16
                        ),

                    Size =
                        UDim2.fromOffset(
                            56,
                            56
                        ),

                    BackgroundColor3 =
                        C.Card,

                    BorderSizePixel =
                        0,

                    Image =
                        MM2_ICON,

                    Visible =
                        false,

                    ZIndex =
                        500
                },
                gui
            )

        corner(
            floating,
            14
        )
    end

    local function setVisible(value)
        menu.Visible =
            value

        if floating then
            floating.Visible =
                not value
        end
    end

    track(
        minimize.MouseButton1Click:Connect(function()

            setVisible(
                false
            )
        end)
    )

    track(
        close.MouseButton1Click:Connect(function()

            setVisible(
                false
            )
        end)
    )

    if floating then
        track(
            floating.MouseButton1Click:Connect(function()

                setVisible(
                    true
                )
            end)
        )
    end

    track(
        UIS.InputBegan:Connect(function(
            input,
            processed
        )
            if processed
                or isPhone then

                return
            end

            if input.KeyCode
                == Enum.KeyCode.RightShift then

                setVisible(
                    not menu.Visible
                )
            end
        end)
    )

    show(
        "Home"
    )
end

local function cleanup()
    if not alive then
        return
    end

    alive =
        false

    replay.recording =
        false

    if replay.playing then
        stopReplay()
    end

    sheriff.quickShot =
        false

    murderer.autoThrow =
        false

    gunPickup.auto =
        false

    coinFarm.enabled =
        false

    coinFarm.busy =
        false

    coinFarm.current =
        nil

    movement.speedEnabled =
        false

    movement.infiniteJump =
        false

    movement.spin =
        false

    safety.underground =
        false

    destroySpin()
    destroyUndergroundForce()
    destroyCoinPlatform()

    collision.sources.noclip =
        false

    collision.sources.coinFarm =
        false

    collision.sources.underground =
        false

    restoreCollision()

    disableAntiFling()
    disableESP()

    for targetPlayer in pairs(
        esp.playerConnections
    ) do
        disconnect(
            esp.playerConnections[
                targetPlayer
            ]
        )
    end

    for targetPlayer in pairs(
        esp.characterConnections
    ) do
        disconnect(
            esp.characterConnections[
                targetPlayer
            ]
        )
    end

    table.clear(
        esp.playerConnections
    )

    table.clear(
        esp.characterConnections
    )

    for _, connection in ipairs(
        runtimeConnections
    ) do
        disconnect(
            connection
        )
    end

    table.clear(
        runtimeConnections
    )

    for _, connection in ipairs(
        connections
    ) do
        disconnect(
            connection
        )
    end

    table.clear(
        connections
    )

    for object, connection in pairs(
        bagTextConnections
    ) do
        disconnect(
            connection
        )

        bagTextConnections[
            object
        ] = nil
    end

    if humanoid then
        pcall(function()

            humanoid.WalkSpeed =
                originalWalkSpeed

            humanoid.CameraOffset =
                Vector3.zero

            humanoid.PlatformStand =
                false
        end)
    end

    if blur then
        pcall(function()
            blur:Destroy()
        end)

        blur =
            nil
    end

    if gui then
        pcall(function()
            gui:Destroy()
        end)

        gui =
            nil
    end

    if _G.THHGrowersCleanup
        == cleanup then

        _G.THHGrowersCleanup =
            nil
    end
end

_G.THHGrowersCleanup =
    cleanup

local checkingKey =
    false

track(
    getKeyButton.MouseButton1Click:Connect(function()

        local copied =
            false

        if setclipboard then
            copied =
                pcall(function()

                    setclipboard(
                        KEY_FLOW
                    )
                end)

        elseif toclipboard then
            copied =
                pcall(function()

                    toclipboard(
                        KEY_FLOW
                    )
                end)
        end

        if copied then
            authStatus.Text =
                "access key link copied"

            authStatus.TextColor3 =
                C.Accent
        else
            authStatus.Text =
                KEY_FLOW

            authStatus.TextColor3 =
                C.Text
        end
    end)
)

track(
    continueButton.MouseButton1Click:Connect(function()

        if checkingKey then
            return
        end

        checkingKey =
            true

        continueButton.Text =
            "Checking..."

        authStatus.Text =
            "checking key..."

        authStatus.TextColor3 =
            C.SubText

        task.spawn(function()

            local success,
                valid,
                data =
                pcall(function()

                    local key =
                        keyBox.Text

                    local source =
                        game:HttpGet(
                            "https://vampauth.com/client/vampauth.lua"
                        )

                    local Vampauth =
                        loadstring(
                            source
                        )()

                    local vampauthClient =
                        Vampauth.new({
                            projectId =
                                PROJECT_ID,

                            authSecret =
                                AUTH_SECRET,

                            hwid =
                                RbxAnalyticsService:GetClientId(),

                            debug =
                                false
                        })

                    local accepted,
                        information =
                        vampauthClient:Check(
                            key
                        )

                    return accepted,
                        information
                end)

            if not success then
                authStatus.Text =
                    "authentication error"

                authStatus.TextColor3 =
                    C.Danger

                continueButton.Text =
                    "Continue"

                checkingKey =
                    false

                return
            end

            if valid then
                authStatus.Text =
                    "key accepted"

                authStatus.TextColor3 =
                    C.Accent

                task.wait(0.15)

                if authOverlay
                    and authOverlay.Parent then

                    authOverlay:Destroy()
                end

                buildDeviceChooser()
            else
                authStatus.Text =
                    tostring(
                        data
                        or "invalid key"
                    )

                authStatus.TextColor3 =
                    C.Danger

                continueButton.Text =
                    "Continue"

                checkingKey =
                    false
            end
        end)
    end)
)
