if _G.THHGrowersCleanup then
    pcall(_G.THHGrowersCleanup)
end

_G.THHGrowersHardUnloaded = false
_G.THHGrowersCleanup = nil

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local TeleportService = game:GetService("TeleportService")
local RbxAnalyticsService = game:GetService("RbxAnalyticsService")
local Lighting = game:GetService("Lighting")

local VirtualInputManager

pcall(function()
    VirtualInputManager = game:GetService("VirtualInputManager")
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

local Colors = {
    Background = Color3.fromRGB(25, 26, 30),
    Background2 = Color3.fromRGB(31, 32, 37),
    Sidebar = Color3.fromRGB(35, 36, 41),

    Card = Color3.fromRGB(57, 59, 66),
    Control = Color3.fromRGB(46, 48, 54),
    ControlHover = Color3.fromRGB(54, 56, 63),

    Text = Color3.fromRGB(246, 247, 249),
    SubText = Color3.fromRGB(184, 186, 194),

    Accent = Color3.fromRGB(106, 221, 145),
    AccentDark = Color3.fromRGB(45, 81, 59),

    Stroke = Color3.fromRGB(102, 105, 114),
    Danger = Color3.fromRGB(232, 81, 81),

    Innocent = Color3.fromRGB(72, 226, 113),
    Murderer = Color3.fromRGB(243, 72, 72),
    Sheriff = Color3.fromRGB(69, 147, 255)
}

local alive = true

local gui
local blur

local character
local humanoid
local rootPart

local originalWalkSpeed = 16

local connections = {}

local movement = {
    speedEnabled = false,
    speed = 32,

    infiniteJump = false,

    spin = false,
    spinSpeed = 300,

    spinVelocity = nil,
    oldAutoRotate = true
}

local localCollision = {
    sources = {
        noclip = false,
        coinFarm = false,
        underground = false
    },

    originals = {},
    propertyConnections = {},
    descendantConnection = nil
}

local antiFling = {
    enabled = false,

    originals = {},
    trackedParts = {},

    partConnections = {},
    characterConnections = {},
    playerConnections = {},

    playerAddedConnection = nil,
    playerRemovingConnection = nil,

    generation = 0
}

local esp = {
    enabled = true,
    highlights = {},
    characterConnections = {}
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

    cachedDrop = nil,
    lastAttempt = 0
}

local coinFarm = {
    enabled = false,
    busy = false,

    coins = {},
    lookup = {},

    target = nil,
    pickedUp = 0,

    travelSpeed = 20,
    verticalSpeed = 420,
    underDistance = 9,

    platform = nil,

    fullBagDebounce = false,
    cachedBagText = nil,

    lastCacheClean = 0
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

    maxTakes = 2,
    maxDuration = 120,
    sampleRate = 1 / 20,

    takeCounter = 0,

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
    return create(
        "UICorner",
        {
            CornerRadius = UDim.new(0, radius or 9)
        },
        object
    )
end

local function addStroke(
    object,
    color,
    thickness,
    transparency
)
    return create(
        "UIStroke",
        {
            Color = color or Colors.Stroke,
            Thickness = thickness or 1,
            Transparency = transparency or 0.55
        },
        object
    )
end

local function tween(
    object,
    duration,
    properties
)
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

local function getUIParent()
    if gethui then
        local success, result =
            pcall(gethui)

        if success and result then
            return result
        end
    end

    return playerGui
end

local function refreshCharacter()
    character =
        player.Character
        or player.CharacterAdded:Wait()

    humanoid =
        character:FindFirstChildOfClass("Humanoid")
        or character:WaitForChild("Humanoid")

    rootPart =
        character:FindFirstChild("HumanoidRootPart")
        or character:WaitForChild("HumanoidRootPart")

    originalWalkSpeed =
        humanoid.WalkSpeed
end

refreshCharacter()

local function animateButton(button)
    if not UserInputService.MouseEnabled then
        return
    end

    local original =
        button.BackgroundColor3

    track(
        button.MouseEnter:Connect(function()
            tween(
                button,
                0.12,
                {
                    BackgroundColor3 =
                        Colors.ControlHover
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
                    BackgroundColor3 =
                        original
                }
            )
        end)
    )
end

local function makeDraggable(frame, handle)
    handle = handle or frame

    local dragging = false
    local activeInput
    local dragStart
    local startPosition

    track(
        handle.InputBegan:Connect(function(input)
            if input.UserInputType
                    == Enum.UserInputType.MouseButton1
                or input.UserInputType
                    == Enum.UserInputType.Touch then

                dragging = true
                dragStart = input.Position
                startPosition = frame.Position
            end
        end)
    )

    track(
        handle.InputChanged:Connect(function(input)
            if input.UserInputType
                    == Enum.UserInputType.MouseMovement
                or input.UserInputType
                    == Enum.UserInputType.Touch then

                activeInput = input
            end
        end)
    )

    track(
        UserInputService.InputChanged:Connect(function(input)
            if not dragging
                or input ~= activeInput then

                return
            end

            local delta =
                input.Position
                - dragStart

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
        UserInputService.InputEnded:Connect(function(input)
            if input.UserInputType
                    == Enum.UserInputType.MouseButton1
                or input.UserInputType
                    == Enum.UserInputType.Touch then

                dragging = false
            end
        end)
    )
end

local function anyCollisionSource()
    for _, enabled in pairs(
        localCollision.sources
    ) do
        if enabled then
            return true
        end
    end

    return false
end

local function clearLocalCollisionPart(part)
    disconnect(
        localCollision.propertyConnections[part]
    )

    localCollision.propertyConnections[part] = nil
end

local function enforceLocalCollisionPart(part)
    if not anyCollisionSource()
        or not part
        or not part:IsA("BasePart") then

        return
    end

    if localCollision.originals[part] == nil then
        localCollision.originals[part] =
            part.CanCollide
    end

    if part.CanCollide then
        part.CanCollide = false
    end

    if not localCollision.propertyConnections[part] then
        localCollision.propertyConnections[part] =
            part:GetPropertyChangedSignal("CanCollide"):Connect(function()

                if not alive
                    or not anyCollisionSource()
                    or not part.Parent then

                    return
                end

                if part.CanCollide then
                    part.CanCollide = false
                end
            end)
    end
end

local function stopLocalCollisionWatcher()
    disconnect(
        localCollision.descendantConnection
    )

    localCollision.descendantConnection = nil
end

local function restoreLocalCollision()
    stopLocalCollisionWatcher()

    for part, oldValue in pairs(
        localCollision.originals
    ) do
        clearLocalCollisionPart(part)

        if part
            and part.Parent then

            pcall(function()
                part.CanCollide =
                    oldValue
            end)
        end
    end

    table.clear(
        localCollision.originals
    )
end

local function rebuildLocalCollision()
    stopLocalCollisionWatcher()

    if not anyCollisionSource() then
        restoreLocalCollision()
        return
    end

    if not character then
        return
    end

    for _, object in ipairs(
        character:GetDescendants()
    ) do
        if object:IsA("BasePart") then
            enforceLocalCollisionPart(object)
        end
    end

    localCollision.descendantConnection =
        character.DescendantAdded:Connect(function(object)

            if object:IsA("BasePart") then
                enforceLocalCollisionPart(object)
            end
        end)
end

local function setCollisionSource(
    source,
    enabled
)
    localCollision.sources[source] =
        enabled == true

    rebuildLocalCollision()
end

local function setSpeedEnabled(value)
    movement.speedEnabled =
        value == true

    if not humanoid then
        return
    end

    if movement.speedEnabled
        and not coinFarm.enabled
        and not safety.underground
        and not replay.playing then

        humanoid.WalkSpeed =
            movement.speed
    else
        humanoid.WalkSpeed =
            originalWalkSpeed
    end
end

local function setSpeed(value)
    movement.speed =
        value

    if movement.speedEnabled
        and humanoid
        and not coinFarm.enabled
        and not safety.underground
        and not replay.playing then

        humanoid.WalkSpeed =
            value
    end
end

local function destroySpin()
    if movement.spinVelocity then
        pcall(function()
            movement.spinVelocity:Destroy()
        end)

        movement.spinVelocity = nil
    end

    if rootPart
        and rootPart.Parent then

        rootPart.AssemblyAngularVelocity =
            Vector3.zero
    end

    if humanoid then
        pcall(function()
            humanoid.AutoRotate =
                movement.oldAutoRotate

            humanoid.PlatformStand =
                false
        end)
    end
end

local function rebuildSpin()
    destroySpin()

    if not movement.spin
        or not rootPart
        or not humanoid
        or replay.playing then

        return
    end

    movement.oldAutoRotate =
        humanoid.AutoRotate

    humanoid.AutoRotate = false
    humanoid.PlatformStand = false

    local spin =
        Instance.new("BodyAngularVelocity")

    spin.Name =
        "THH_SpinVelocity"

    spin.AngularVelocity =
        Vector3.new(
            0,
            math.rad(
                movement.spinSpeed
            ),
            0
        )

    spin.MaxTorque =
        Vector3.new(
            0,
            1000000000,
            0
        )

    spin.P = 1250
    spin.Parent = rootPart

    movement.spinVelocity =
        spin
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

local function clearAntiFlingPart(part)
    disconnect(
        antiFling.partConnections[part]
    )

    antiFling.partConnections[part] = nil
    antiFling.trackedParts[part] = nil
end

local function applyAntiFlingPart(part)
    if not antiFling.enabled
        or not part
        or not part:IsA("BasePart") then

        return
    end

    if antiFling.originals[part] == nil then
        antiFling.originals[part] =
            part.CanCollide
    end

    antiFling.trackedParts[part] =
        true

    if part.CanCollide then
        part.CanCollide =
            false
    end

    if not antiFling.partConnections[part] then
        antiFling.partConnections[part] =
            part:GetPropertyChangedSignal("CanCollide"):Connect(function()

                if antiFling.enabled
                    and part.Parent
                    and part.CanCollide then

                    part.CanCollide =
                        false
                end
            end)
    end
end

local function clearAntiFlingCharacter(targetPlayer)
    local list =
        antiFling.characterConnections[
            targetPlayer
        ]

    if not list then
        return
    end

    for _, connection in ipairs(list) do
        disconnect(connection)
    end

    antiFling.characterConnections[
        targetPlayer
    ] = nil
end

local function applyAntiFlingCharacter(
    targetPlayer,
    targetCharacter
)
    if not antiFling.enabled
        or targetPlayer == player
        or not targetCharacter then

        return
    end

    clearAntiFlingCharacter(targetPlayer)

    for _, object in ipairs(
        targetCharacter:GetDescendants()
    ) do
        if object:IsA("BasePart") then
            applyAntiFlingPart(object)
        end
    end

    local list = {}

    table.insert(
        list,
        targetCharacter.DescendantAdded:Connect(function(object)

            if object:IsA("BasePart") then
                applyAntiFlingPart(object)
            end
        end)
    )

    table.insert(
        list,
        targetCharacter.DescendantRemoving:Connect(function(object)

            if not object:IsA("BasePart") then
                return
            end

            task.defer(function()
                if not object.Parent then
                    clearAntiFlingPart(object)
                    antiFling.originals[object] = nil
                end
            end)
        end)
    )

    antiFling.characterConnections[
        targetPlayer
    ] = list
end

local function setupAntiFlingPlayer(targetPlayer)
    if targetPlayer == player then
        return
    end

    disconnect(
        antiFling.playerConnections[
            targetPlayer
        ]
    )

    antiFling.playerConnections[targetPlayer] =
        targetPlayer.CharacterAdded:Connect(function(targetCharacter)

            if antiFling.enabled then
                task.defer(function()
                    applyAntiFlingCharacter(
                        targetPlayer,
                        targetCharacter
                    )
                end)
            end
        end)

    if targetPlayer.Character then
        applyAntiFlingCharacter(
            targetPlayer,
            targetPlayer.Character
        )
    end
end

local function enableAntiFling()
    if antiFling.enabled then
        return
    end

    antiFling.enabled = true
    antiFling.generation += 1

    local generation =
        antiFling.generation

    for _, targetPlayer in ipairs(
        Players:GetPlayers()
    ) do
        if targetPlayer ~= player then
            setupAntiFlingPlayer(targetPlayer)
        end
    end

    antiFling.playerAddedConnection =
        Players.PlayerAdded:Connect(function(targetPlayer)

            if antiFling.enabled then
                setupAntiFlingPlayer(targetPlayer)
            end
        end)

    antiFling.playerRemovingConnection =
        Players.PlayerRemoving:Connect(function(targetPlayer)

            clearAntiFlingCharacter(targetPlayer)

            disconnect(
                antiFling.playerConnections[
                    targetPlayer
                ]
            )

            antiFling.playerConnections[
                targetPlayer
            ] = nil
        end)

    task.spawn(function()
        while alive
            and antiFling.enabled
            and antiFling.generation == generation do

            for part in pairs(
                antiFling.trackedParts
            ) do
                if not part
                    or not part.Parent then

                    clearAntiFlingPart(part)
                    antiFling.originals[part] = nil

                elseif part.CanCollide then
                    part.CanCollide =
                        false
                end
            end

            task.wait(0.12)
        end
    end)
end

local function disableAntiFling()
    antiFling.enabled = false
    antiFling.generation += 1

    disconnect(
        antiFling.playerAddedConnection
    )

    disconnect(
        antiFling.playerRemovingConnection
    )

    antiFling.playerAddedConnection = nil
    antiFling.playerRemovingConnection = nil

    for targetPlayer, connection in pairs(
        antiFling.playerConnections
    ) do
        disconnect(connection)

        antiFling.playerConnections[
            targetPlayer
        ] = nil
    end

    for targetPlayer in pairs(
        antiFling.characterConnections
    ) do
        clearAntiFlingCharacter(
            targetPlayer
        )
    end

    for part, oldValue in pairs(
        antiFling.originals
    ) do
        clearAntiFlingPart(part)

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
        antiFling.trackedParts
    )
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

    if not safety.underground
        or not rootPart
        or replay.playing then

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

local function enableUnderground()
    if safety.underground
        or not humanoid
        or not rootPart
        or replay.playing then

        return
    end

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

    local rotation =
        rootPart.CFrame
        - rootPart.Position

    rootPart.CFrame =
        CFrame.new(
            rootPart.Position
            - Vector3.new(
                0,
                safety.depth,
                0
            )
        )
        * rotation

    rootPart.AssemblyLinearVelocity =
        Vector3.zero

    rebuildUndergroundForce()
end

local function disableUnderground(returnToCamera)
    if not safety.underground then
        return
    end

    local camera =
        workspace.CurrentCamera

    local targetPosition =
        camera
        and camera.CFrame.Position
        or nil

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

    if returnToCamera
        and targetPosition
        and rootPart
        and rootPart.Parent then

        local rotation =
            rootPart.CFrame
            - rootPart.Position

        rootPart.CFrame =
            CFrame.new(
                targetPosition
                + Vector3.new(
                    0,
                    2,
                    0
                )
            )
            * rotation

        rootPart.AssemblyLinearVelocity =
            Vector3.zero
    end
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
                    and string.lower(object.Name)
                        == wantedName then

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
            "Knife"
        )

    if exact then
        return exact
    end

    local containers = {
        character,
        player:FindFirstChildOfClass(
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
            "Gun"
        )

    if exact then
        return exact
    end

    local containers = {
        character,
        player:FindFirstChildOfClass(
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
                if object:IsA("Tool") then
                    local lower =
                        string.lower(
                            object.Name
                        )

                    if lower:find(
                        "gun",
                        1,
                        true
                    )
                        or lower:find(
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
    if findTool(
        targetPlayer,
        "Knife"
    ) then

        return true
    end

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
    if targetPlayer == player then
        return getGunTool() ~= nil
    end

    if findTool(
        targetPlayer,
        "Gun"
    ) then

        return true
    end

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
                if object:IsA("Tool") then
                    local lower =
                        string.lower(
                            object.Name
                        )

                    if lower:find(
                        "gun",
                        1,
                        true
                    )
                        or lower:find(
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

local function getRoleColor(targetPlayer)
    if hasKnife(targetPlayer) then
        return Colors.Murderer
    end

    if hasGun(targetPlayer) then
        return Colors.Sheriff
    end

    return Colors.Innocent
end

local function removePlayerESP(targetPlayer)
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

local function addPlayerESP(targetPlayer)
    if not esp.enabled
        or targetPlayer == player
        or not targetPlayer.Character then

        return
    end

    removePlayerESP(
        targetPlayer
    )

    local highlight =
        Instance.new("Highlight")

    highlight.Name =
        "THH_ESP_"
        .. targetPlayer.Name

    highlight.DepthMode =
        Enum.HighlightDepthMode.AlwaysOnTop

    highlight.FillTransparency =
        0.77

    highlight.OutlineTransparency =
        0

    local color =
        getRoleColor(
            targetPlayer
        )

    highlight.FillColor =
        color

    highlight.OutlineColor =
        color

    highlight.Adornee =
        targetPlayer.Character

    highlight.Parent =
        targetPlayer.Character

    esp.highlights[targetPlayer] =
        highlight
end

local function setupESPPlayer(targetPlayer)
    if targetPlayer == player then
        return
    end

    if esp.enabled then
        addPlayerESP(
            targetPlayer
        )
    end

    disconnect(
        esp.characterConnections[
            targetPlayer
        ]
    )

    esp.characterConnections[
        targetPlayer
    ] =
        targetPlayer.CharacterAdded:Connect(function()

            removePlayerESP(
                targetPlayer
            )

            task.wait(0.1)

            if alive
                and esp.enabled then

                addPlayerESP(
                    targetPlayer
                )
            end
        end)
end

local function enableESP()
    esp.enabled =
        true

    for _, targetPlayer in ipairs(
        Players:GetPlayers()
    ) do
        setupESPPlayer(
            targetPlayer
        )
    end
end

local function disableESP()
    esp.enabled =
        false

    for targetPlayer in pairs(
        esp.highlights
    ) do
        removePlayerESP(
            targetPlayer
        )
    end
end

for _, targetPlayer in ipairs(
    Players:GetPlayers()
) do
    setupESPPlayer(
        targetPlayer
    )
end

track(
    Players.PlayerAdded:Connect(function(
        targetPlayer
    )
        setupESPPlayer(
            targetPlayer
        )
    end)
)

track(
    Players.PlayerRemoving:Connect(function(
        targetPlayer
    )
        removePlayerESP(
            targetPlayer
        )

        disconnect(
            esp.characterConnections[
                targetPlayer
            ]
        )

        esp.characterConnections[
            targetPlayer
        ] = nil
    end)
)

task.spawn(function()
    while alive do
        if esp.enabled
            and not replay.playing then

            for _, targetPlayer in ipairs(
                Players:GetPlayers()
            ) do
                if targetPlayer ~= player
                    and targetPlayer.Character then

                    local highlight =
                        esp.highlights[
                            targetPlayer
                        ]

                    if not highlight
                        or not highlight.Parent
                        or highlight.Adornee
                            ~= targetPlayer.Character then

                        addPlayerESP(
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
                end
            end
        end

        task.wait(0.1)
    end
end)

local function getGunRemote(gun)
    if not gun then
        return nil
    end

    local knifeLocal =
        gun:FindFirstChild(
            "KnifeLocal"
        )

    if not knifeLocal then
        return nil
    end

    local createBeam =
        knifeLocal:FindFirstChild(
            "CreateBeam"
        )

    if not createBeam then
        return nil
    end

    local remote =
        createBeam:FindFirstChild(
            "RemoteFunction"
        )

    if remote
        and remote:IsA("RemoteFunction") then

        return remote
    end

    return nil
end

local function fireGunAtPosition(
    gun,
    worldPosition
)
    if not gun
        or not worldPosition then

        return false
    end

    local remote =
        getGunRemote(
            gun
        )

    if remote then
        local success =
            pcall(function()

                remote:InvokeServer(
                    1,
                    worldPosition,
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

local function getClickedWorldPosition(input)
    local camera =
        workspace.CurrentCamera

    if not camera then
        return nil, nil
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
            UserInputService:GetMouseLocation()
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

    params.IgnoreWater =
        true

    local result =
        workspace:Raycast(
            ray.Origin,
            ray.Direction * 3000,
            params
        )

    if result then
        return result.Instance,
            result.Position
    end

    return nil,
        ray.Origin
        + ray.Direction * 3000
end

local function manualAimShoot(input)
    if replay.playing
        or sheriff.shooting
        or not sheriff.quickShot
        or not humanoid
        or not character then

        return
    end

    local hitPart,
        clickedPosition =
        getClickedWorldPosition(
            input
        )

    if not hitPart
        or not clickedPosition then

        return
    end

    local targetPlayer =
        getPlayerFromPart(
            hitPart
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
            pcall(function()
                humanoid:EquipTool(
                    gun
                )
            end)

            RunService.Heartbeat:Wait()
            RunService.Heartbeat:Wait()
        end

        if gun.Parent == character then
            fireGunAtPosition(
                gun,
                clickedPosition
            )

            task.wait(0.07)

            if humanoid then
                pcall(function()
                    humanoid:UnequipTools()
                end)
            end
        end

        sheriff.shooting =
            false
    end)
end

track(
    UserInputService.InputBegan:Connect(function(
        input,
        gameProcessed
    )
        if gameProcessed
            or replay.playing
            or not sheriff.quickShot then

            return
        end

        if input.UserInputType
                == Enum.UserInputType.MouseButton1
            or input.UserInputType
                == Enum.UserInputType.Touch then

            manualAimShoot(
                input
            )
        end
    end)
)

local function sendEKey()
    if UserInputService:GetFocusedTextBox() then
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

    if VirtualInputManager then
        local success =
            pcall(function()

                VirtualInputManager:SendKeyEvent(
                    true,
                    Enum.KeyCode.E,
                    false,
                    game
                )

                task.wait(0.04)

                VirtualInputManager:SendKeyEvent(
                    false,
                    Enum.KeyCode.E,
                    false,
                    game
                )
            end)

        if success then
            return true
        end
    end

    return false
end

local function sendRightClick()
    if not VirtualInputManager
        or UserInputService:GetFocusedTextBox() then

        return false
    end

    local mousePosition =
        UserInputService:GetMouseLocation()

    return pcall(function()

        VirtualInputManager:SendMouseButtonEvent(
            mousePosition.X,
            mousePosition.Y,
            1,
            true,
            game,
            0
        )

        task.wait(0.04)

        VirtualInputManager:SendMouseButtonEvent(
            mousePosition.X,
            mousePosition.Y,
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
                object.Text or ""
            )
    end

    return name:find(
        "throw",
        1,
        true
    )
        or text:find(
            "throw",
            1,
            true
        )
end

local function throwButtonUsable(button)
    if not button
        or not button.Parent
        or not button:IsA("GuiButton")
        or not button.Visible then

        return false
    end

    local current =
        button

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

local function findThrowButton()
    if throwButtonUsable(
        murderer.cachedThrowButton
    ) then

        return murderer.cachedThrowButton
    end

    murderer.cachedThrowButton =
        nil

    for _, object in ipairs(
        playerGui:GetDescendants()
    ) do
        if isThrowButton(object)
            and throwButtonUsable(object) then

            murderer.cachedThrowButton =
                object

            return object
        end
    end

    return nil
end

track(
    playerGui.DescendantAdded:Connect(function(
        object
    )
        if isThrowButton(object) then
            murderer.cachedThrowButton =
                object
        end
    end)
)

track(
    playerGui.DescendantRemoving:Connect(function(
        object
    )
        if murderer.cachedThrowButton
            == object then

            murderer.cachedThrowButton =
                nil
        end
    end)
)

local function pressMobileThrow()
    if not VirtualInputManager then
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

        VirtualInputManager:SendMouseButtonEvent(
            center.X,
            center.Y,
            0,
            true,
            game,
            0
        )

        task.wait(0.04)

        VirtualInputManager:SendMouseButtonEvent(
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

    local success =
        false

    if knife.Parent ~= character then
        pcall(function()
            humanoid:EquipTool(
                knife
            )
        end)

        RunService.Heartbeat:Wait()
        RunService.Heartbeat:Wait()
    end

    if knife.Parent == character then
        local phoneOnly =
            UserInputService.TouchEnabled
            and not UserInputService.KeyboardEnabled
            and not UserInputService.MouseEnabled

        if phoneOnly then
            success =
                pressMobileThrow()

            if not success then
                success =
                    sendEKey()
            end
        else
            success =
                sendRightClick()

            if not success then
                success =
                    sendEKey()
            end
        end
    end

    task.wait(0.025)

    murderer.throwing =
        false

    return success
end

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

            throwKnifeOnce()
        end

        task.wait(0.015)
    end
end)

local function normalizeName(name)
    return string.lower(
        tostring(name or "")
    ):gsub(
        "[%s_%-%./]",
        ""
    )
end

local function isGunDropName(name)
    local normalized =
        normalizeName(name)

    return normalized == "gundrop"
        or normalized == "droppedgun"
        or normalized == "dropgun"
        or normalized == "droppedrevolver"
end

local function findInitialGunDrop()
    local exact =
        workspace:FindFirstChild(
            "GunDrop",
            true
        )

    if exact then
        return exact
    end

    for _, object in ipairs(
        workspace:GetDescendants()
    ) do
        if isGunDropName(
            object.Name
        ) then

            return object
        end
    end

    return nil
end

gunPickup.cachedDrop =
    findInitialGunDrop()

track(
    workspace.DescendantAdded:Connect(function(
        object
    )
        if isGunDropName(
            object.Name
        ) then

            gunPickup.cachedDrop =
                object
        end
    end)
)

track(
    workspace.DescendantRemoving:Connect(function(
        object
    )
        if gunPickup.cachedDrop
            == object then

            gunPickup.cachedDrop =
                nil
        end
    end)
)

local function getGunDrop()
    if gunPickup.cachedDrop
        and gunPickup.cachedDrop.Parent then

        return gunPickup.cachedDrop
    end

    return nil
end

local function getObjectPart(object)
    if not object then
        return nil
    end

    if object:IsA("BasePart") then
        return object
    end

    if object:IsA("Tool") then
        local handle =
            object:FindFirstChild(
                "Handle"
            )

        if handle
            and handle:IsA("BasePart") then

            return handle
        end
    end

    if object:IsA("Model")
        and object.PrimaryPart then

        return object.PrimaryPart
    end

    return object:FindFirstChildWhichIsA(
        "BasePart",
        true
    )
end

local function touchPickup(object, part)
    if replay.playing
        or not rootPart
        or not object
        or not part then

        return
    end

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

    if fireproximityprompt then
        for _, descendant in ipairs(
            object:GetDescendants()
        ) do
            if descendant:IsA(
                "ProximityPrompt"
            ) then

                pcall(function()
                    fireproximityprompt(
                        descendant,
                        0
                    )
                end)
            end
        end
    end
end

local function pickupGun()
    if replay.playing
        or gunPickup.busy
        or coinFarm.enabled
        or not rootPart
        or getGunTool() then

        return false
    end

    local drop =
        getGunDrop()

    local part =
        getObjectPart(
            drop
        )

    if not drop
        or not part then

        return false
    end

    gunPickup.busy =
        true

    local oldCFrame =
        rootPart.CFrame

    local oldLinear =
        rootPart.AssemblyLinearVelocity

    local oldAngular =
        rootPart.AssemblyAngularVelocity

    local success =
        pcall(function()

            rootPart.AssemblyLinearVelocity =
                Vector3.zero

            if not movement.spin then
                rootPart.AssemblyAngularVelocity =
                    Vector3.zero
            end

            rootPart.CFrame =
                part.CFrame
                * CFrame.new(
                    0,
                    0.35,
                    0
                )

            touchPickup(
                drop,
                part
            )

            RunService.Heartbeat:Wait()

            touchPickup(
                drop,
                part
            )

            if rootPart
                and rootPart.Parent then

                rootPart.CFrame =
                    oldCFrame

                rootPart.AssemblyLinearVelocity =
                    oldLinear

                if not movement.spin then
                    rootPart.AssemblyAngularVelocity =
                        oldAngular
                end
            end
        end)

    gunPickup.busy =
        false

    return success
end

task.spawn(function()
    while alive do
        if gunPickup.auto
            and not replay.playing
            and not gunPickup.busy
            and not coinFarm.enabled
            and not getGunTool()
            and getGunDrop()
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

local function isMainCoin(object)
    return object
        and object:IsA("BasePart")
        and object.Name == "MainCoin"
end

local function coinIsAvailable(coin)
    return coin
        and coinFarm.lookup[coin]
        and coin.Parent
        and coin.Transparency == 0
        and coin.LocalTransparencyModifier == 0
end

local function addCoin(coin)
    if not isMainCoin(coin)
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

local function buildCoinCache()
    table.clear(
        coinFarm.coins
    )

    table.clear(
        coinFarm.lookup
    )

    for _, object in ipairs(
        workspace:GetDescendants()
    ) do
        if isMainCoin(object) then
            addCoin(object)
        end
    end
end

buildCoinCache()

track(
    workspace.DescendantAdded:Connect(function(
        object
    )
        if isMainCoin(object) then
            addCoin(object)
        end
    end)
)

track(
    workspace.DescendantRemoving:Connect(function(
        object
    )
        if coinFarm.lookup[object] then
            removeCoin(object)
        end
    end)
)

local function cleanCoinCache()
    for index = #coinFarm.coins, 1, -1 do
        local coin =
            coinFarm.coins[index]

        if not coin
            or not coin.Parent then

            if coin then
                coinFarm.lookup[coin] =
                    nil
            end

            table.remove(
                coinFarm.coins,
                index
            )
        end
    end
end

local function getNearestCoin()
    if not rootPart then
        return nil
    end

    local rootPosition =
        rootPart.Position

    local nearestCoin
    local nearestDistanceSquared =
        math.huge

    for _, coin in ipairs(
        coinFarm.coins
    ) do
        if coinIsAvailable(coin) then
            local target =
                coin.Position
                - Vector3.new(
                    0,
                    coinFarm.underDistance,
                    0
                )

            local dx =
                target.X
                - rootPosition.X

            local dy =
                target.Y
                - rootPosition.Y

            local dz =
                target.Z
                - rootPosition.Z

            local distanceSquared =
                dx * dx
                + dy * dy
                + dz * dz

            if distanceSquared
                < nearestDistanceSquared then

                nearestDistanceSquared =
                    distanceSquared

                nearestCoin =
                    coin
            end
        end
    end

    return nearestCoin
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

    platform.Anchored =
        true

    platform.CanCollide =
        true

    platform.CanTouch =
        false

    platform.CanQuery =
        false

    platform.Transparency =
        1

    platform.Parent =
        workspace

    coinFarm.platform =
        platform

    return platform
end

local function setFarmPosition(position)
    if replay.playing
        or not rootPart
        or not rootPart.Parent then

        return
    end

    local rotation =
        rootPart.CFrame
        - rootPart.Position

    rootPart.CFrame =
        CFrame.new(position)
        * rotation

    rootPart.AssemblyLinearVelocity =
        Vector3.zero

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
end

local function moveFarmTo(
    destination,
    speedValue,
    stopDistance,
    coin
)
    stopDistance =
        stopDistance or 0.8

    while alive
        and coinFarm.enabled
        and not replay.playing
        and not coinFarm.fullBagDebounce
        and not safety.underground
        and rootPart
        and rootPart.Parent do

        if coin
            and not coinIsAvailable(
                coin
            ) then

            return false
        end

        local current =
            rootPart.Position

        local offset =
            destination
            - current

        local distance =
            offset.Magnitude

        if distance <= stopDistance then
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
                speedValue * dt
            )

        if distance > 0 then
            setFarmPosition(
                current
                + offset.Unit * step
            )
        end
    end

    return false
end

local function waitForCoinPickup(coin)
    local started =
        os.clock()

    while alive
        and coinFarm.enabled
        and not replay.playing
        and os.clock()
            - started
            < 0.35 do

        if not coinIsAvailable(
            coin
        ) then

            return true
        end

        task.wait(0.015)
    end

    return not coinIsAvailable(
        coin
    )
end

local function collectCoin(coin)
    if replay.playing
        or not coinIsAvailable(coin)
        or not rootPart then

        return false
    end

    local under =
        coin.Position
        - Vector3.new(
            0,
            coinFarm.underDistance,
            0
        )

    if not moveFarmTo(
        under,
        coinFarm.travelSpeed,
        0.75,
        coin
    ) then

        return false
    end

    if not coinIsAvailable(
        coin
    ) then

        return false
    end

    if not moveFarmTo(
        coin.Position,
        coinFarm.verticalSpeed,
        1.4,
        coin
    ) then

        return false
    end

    if firetouchinterest
        and rootPart
        and coinIsAvailable(coin) then

        pcall(function()

            firetouchinterest(
                rootPart,
                coin,
                0
            )

            firetouchinterest(
                rootPart,
                coin,
                1
            )
        end)
    end

    local picked =
        waitForCoinPickup(
            coin
        )

    moveFarmTo(
        under,
        coinFarm.verticalSpeed,
        0.8
    )

    if picked then
        coinFarm.pickedUp += 1
    end

    return picked
end

local function setCoinFarmEnabled(value)
    coinFarm.enabled =
        value == true

    coinFarm.busy =
        false

    coinFarm.target =
        nil

    setCollisionSource(
        "coinFarm",
        coinFarm.enabled
    )

    if coinFarm.enabled then
        ensureCoinPlatform()

        if humanoid then
            humanoid.WalkSpeed =
                originalWalkSpeed
        end
    else
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

task.spawn(function()
    while alive do
        if coinFarm.enabled
            and not replay.playing
            and not coinFarm.busy
            and not coinFarm.fullBagDebounce
            and not safety.underground
            and rootPart then

            if os.clock()
                - coinFarm.lastCacheClean
                >= 5 then

                coinFarm.lastCacheClean =
                    os.clock()

                cleanCoinCache()
            end

            local coin =
                getNearestCoin()

            coinFarm.target =
                coin

            if coin then
                coinFarm.busy =
                    true

                collectCoin(
                    coin
                )

                coinFarm.busy =
                    false
            else
                task.wait(0.08)
            end
        else
            task.wait(0.03)
        end

        task.wait(0.01)
    end
end)

local function guiObjectVisible(object)
    if not object
        or not object:IsA("GuiObject")
        or not object.Visible
        or object.AbsoluteSize.X <= 0
        or object.AbsoluteSize.Y <= 0 then

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

    local camera =
        workspace.CurrentCamera

    if not camera then
        return true
    end

    local position =
        object.AbsolutePosition

    local size =
        object.AbsoluteSize

    local viewport =
        camera.ViewportSize

    return position.X + size.X > 0
        and position.Y + size.Y > 0
        and position.X < viewport.X
        and position.Y < viewport.Y
end

local function textLooksLikeBagFull(object)
    if not object then
        return false
    end

    if not (
        object:IsA("TextLabel")
        or object:IsA("TextButton")
        or object:IsA("TextBox")
    ) then

        return false
    end

    local normalized =
        string.lower(
            tostring(
                object.Text or ""
            )
        )
        :gsub("%s+", "")
        :gsub("[^%w]", "")

    return normalized:find(
        "coinbagfull",
        1,
        true
    ) ~= nil
        or normalized:find(
            "bagfull",
            1,
            true
        ) ~= nil
end

local function watchBagTextObject(object)
    if not (
        object:IsA("TextLabel")
        or object:IsA("TextButton")
        or object:IsA("TextBox")
    ) then

        return
    end

    if textLooksLikeBagFull(object) then
        coinFarm.cachedBagText =
            object
    end

    if bagTextConnections[object] then
        return
    end

    bagTextConnections[object] =
        object:GetPropertyChangedSignal("Text"):Connect(function()

            if textLooksLikeBagFull(object) then
                coinFarm.cachedBagText =
                    object

            elseif coinFarm.cachedBagText
                == object then

                coinFarm.cachedBagText =
                    nil
            end
        end)
end

for _, object in ipairs(
    playerGui:GetDescendants()
) do
    watchBagTextObject(
        object
    )
end

track(
    playerGui.DescendantAdded:Connect(function(
        object
    )
        watchBagTextObject(
            object
        )
    end)
)

track(
    playerGui.DescendantRemoving:Connect(function(
        object
    )
        if coinFarm.cachedBagText
            == object then

            coinFarm.cachedBagText =
                nil
        end

        disconnect(
            bagTextConnections[object]
        )

        bagTextConnections[object] =
            nil
    end)
)

local function scanForBagFull()
    if coinFarm.cachedBagText
        and coinFarm.cachedBagText.Parent
        and textLooksLikeBagFull(
            coinFarm.cachedBagText
        ) then

        return coinFarm.cachedBagText
    end

    for _, object in ipairs(
        playerGui:GetDescendants()
    ) do
        if textLooksLikeBagFull(object) then
            coinFarm.cachedBagText =
                object

            return object
        end
    end

    return nil
end

local function handleCoinBagFull()
    if replay.playing
        or not coinFarm.enabled
        or coinFarm.fullBagDebounce then

        return
    end

    local bagText =
        coinFarm.cachedBagText
        or scanForBagFull()

    if not bagText
        or not bagText.Parent
        or not textLooksLikeBagFull(bagText)
        or (
            bagText:IsA("GuiObject")
            and not guiObjectVisible(bagText)
        ) then

        return
    end

    if bagText:IsA("TextLabel")
        or bagText:IsA("TextButton")
        or bagText:IsA("TextBox") then

        if bagText.TextTransparency
            >= 0.95 then

            return
        end
    end

    coinFarm.fullBagDebounce =
        true

    coinFarm.busy =
        false

    coinFarm.target =
        nil

    destroyCoinPlatform()

    pcall(function()
        bagText:Destroy()
    end)

    coinFarm.cachedBagText =
        nil

    if humanoid
        and humanoid.Health > 0 then

        humanoid.Health =
            0
    end

    task.spawn(function()

        local newCharacter =
            player.CharacterAdded:Wait()

        if not alive then
            return
        end

        task.wait(0.3)

        if player.Character
            == newCharacter then

            refreshCharacter()
        end

        if coinFarm.enabled then
            setCollisionSource(
                "coinFarm",
                true
            )

            ensureCoinPlatform()
        end

        coinFarm.fullBagDebounce =
            false
    end)
end

task.spawn(function()
    while alive do
        if coinFarm.enabled
            and not replay.playing then

            handleCoinBagFull()
        end

        task.wait(0.08)
    end
end)

local function updateReplayStatus(text, color)
    if replay.statusLabel
        and replay.statusLabel.Parent then

        replay.statusLabel.Text =
            text

        replay.statusLabel.TextColor3 =
            color or Colors.SubText
    end
end

local function stripReplayScripts(instance)
    for _, object in ipairs(
        instance:GetDescendants()
    ) do
        if object:IsA("Script")
            or object:IsA("LocalScript")
            or object:IsA("ModuleScript") then

            object:Destroy()

        elseif object:IsA("ProximityPrompt")
            or object:IsA("ClickDetector") then

            object:Destroy()
        end
    end
end

local function safeClone(instance)
    if not instance then
        return nil
    end

    local oldArchivable =
        instance.Archivable

    local success, clone =
        pcall(function()

            instance.Archivable =
                true

            return instance:Clone()
        end)

    pcall(function()
        instance.Archivable =
            oldArchivable
    end)

    if success then
        return clone
    end

    return nil
end

local function isLivePlayerCharacter(object)
    if not object
        or not object:IsA("Model") then

        return false
    end

    return Players:GetPlayerFromCharacter(
        object
    ) ~= nil
end

local function createMapSnapshot()
    local folder =
        Instance.new("Folder")

    folder.Name =
        "THH_ReplayMapTemplate"

    for _, object in ipairs(
        workspace:GetChildren()
    ) do
        local skip =
            object == workspace.CurrentCamera
            or object:IsA("Terrain")
            or isLivePlayerCharacter(object)
            or object.Name == "THH_ReplayWorld"
            or object.Name == "THH_CoinPlatform"

        if not skip then
            local clone =
                safeClone(object)

            if clone then
                stripReplayScripts(clone)

                for _, part in ipairs(
                    clone:GetDescendants()
                ) do
                    if part:IsA("BasePart") then
                        part.Anchored = true
                        part.CanCollide = false
                        part.CanTouch = false
                        part.CanQuery = false
                    end
                end

                if clone:IsA("BasePart") then
                    clone.Anchored = true
                    clone.CanCollide = false
                    clone.CanTouch = false
                    clone.CanQuery = false
                end

                clone.Parent =
                    folder
            end
        end
    end

    return folder
end

local function motorKey(motor)
    local part0 =
        motor.Part0
        and motor.Part0.Name
        or ""

    local part1 =
        motor.Part1
        and motor.Part1.Name
        or ""

    return motor.Name
        .. "|"
        .. part0
        .. "|"
        .. part1
end

local function captureMotors(model)
    local result = {}

    for _, object in ipairs(
        model:GetDescendants()
    ) do
        if object:IsA("Motor6D") then
            result[motorKey(object)] =
                object.Transform
        end
    end

    return result
end

local function buildMotorMap(model)
    local result = {}

    for _, object in ipairs(
        model:GetDescendants()
    ) do
        if object:IsA("Motor6D") then
            result[motorKey(object)] =
                object
        end
    end

    return result
end

local function prepareReplayCharacter(model)
    stripReplayScripts(model)

    local replayHumanoid =
        model:FindFirstChildOfClass(
            "Humanoid"
        )

    if replayHumanoid then
        replayHumanoid.DisplayDistanceType =
            Enum.HumanoidDisplayDistanceType.None

        replayHumanoid.AutoRotate =
            false

        replayHumanoid.PlatformStand =
            true

        replayHumanoid.WalkSpeed =
            0

        replayHumanoid.JumpPower =
            0

        local animator =
            replayHumanoid:FindFirstChildOfClass(
                "Animator"
            )

        if animator then
            animator:Destroy()
        end
    end

    local replayRoot =
        model:FindFirstChild(
            "HumanoidRootPart"
        )

    for _, object in ipairs(
        model:GetDescendants()
    ) do
        if object:IsA("BasePart") then
            object.CanCollide = false
            object.CanTouch = false
            object.CanQuery = false
            object.Massless = true
            object.Anchored =
                object == replayRoot
        end
    end

    if replayRoot then
        replayRoot.Anchored =
            true
    end

    return replayRoot
end

local function addReplayNameTag(
    model,
    displayName,
    username
)
    local head =
        model:FindFirstChild(
            "Head"
        )

    if not head
        or not head:IsA("BasePart") then

        return
    end

    local old =
        head:FindFirstChild(
            "THH_ReplayName"
        )

    if old then
        old:Destroy()
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
        tostring(displayName)
        .. "\n@"
        .. tostring(username)

    label.TextColor3 =
        Colors.Text

    label.TextStrokeTransparency =
        0.45

    label.TextSize =
        14

    label.Font =
        Enum.Font.GothamBold

    label.Parent =
        billboard
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

    prepareReplayCharacter(
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

local function sampleReplayFrame(take)
    local camera =
        workspace.CurrentCamera

    local frame = {
        time =
            os.clock()
            - take.started,

        players = {},

        camera =
            camera
            and camera.CFrame
            or nil,

        fov =
            camera
            and camera.FieldOfView
            or 70
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

            local id =
                tostring(
                    targetPlayer.UserId
                )

            frame.players[id] = {
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
    if replay.playing
        or replay.recording then

        return
    end

    replay.takeCounter += 1

    updateReplayStatus(
        "Preparing map snapshot...",
        Colors.Accent
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
            nil,

        started =
            0,

        duration =
            0
    }

    take.templateFolder.Name =
        "THH_ReplayActorTemplates"

    take.mapTemplate =
        createMapSnapshot()

    for _, targetPlayer in ipairs(
        Players:GetPlayers()
    ) do
        captureReplayTemplate(
            take,
            targetPlayer
        )
    end

    take.started =
        os.clock()

    replay.currentTake =
        take

    replay.recording =
        true

    updateReplayStatus(
        "● Recording Take "
        .. tostring(
            take.number
        ),
        Colors.Danger
    )

    task.spawn(function()

        while alive
            and replay.recording
            and replay.currentTake == take do

            sampleReplayFrame(
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

        updateReplayStatus(
            "Recording too short",
            Colors.Danger
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

        if old then
            if old.templateFolder then
                old.templateFolder:Destroy()
            end

            if old.mapTemplate then
                old.mapTemplate:Destroy()
            end
        end
    end

    updateReplayStatus(
        "Saved Take "
        .. tostring(
            take.number
        )
        .. " • "
        .. string.format(
            "%.1fs",
            take.duration
        ),
        Colors.Accent
    )
end

local function shiftReplayWorld(
    folder,
    offset
)
    for _, object in ipairs(
        folder:GetDescendants()
    ) do
        if object:IsA("BasePart") then
            object.CFrame =
                object.CFrame
                + offset

            object.Anchored =
                true

            object.CanCollide =
                false

            object.CanTouch =
                false

            object.CanQuery =
                false
        end
    end
end

local function setReplayActorVisible(
    actorData,
    visible
)
    if actorData.visible ==
        visible then

        return
    end

    actorData.visible =
        visible

    for _, part in ipairs(
        actorData.parts
    ) do
        if part
            and part.Parent then

            part.LocalTransparencyModifier =
                visible
                and 0
                or 1
        end
    end

    if actorData.nameTag then
        actorData.nameTag.Enabled =
            visible
    end
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

    clone.Name =
        template.Name

    clone.Parent =
        parent

    prepareReplayCharacter(
        clone
    )

    local info =
        take.playerInfo[id]
        or {
            name = template.Name,
            displayName = template.Name
        }

    addReplayNameTag(
        clone,
        info.displayName,
        info.name
    )

    local parts = {}

    for _, object in ipairs(
        clone:GetDescendants()
    ) do
        if object:IsA("BasePart") then
            table.insert(
                parts,
                object
            )
        end
    end

    local head =
        clone:FindFirstChild(
            "Head"
        )

    local nameTag =
        head
        and head:FindFirstChild(
            "THH_ReplayName"
        )

    local actorData = {
        model =
            clone,

        root =
            clone:FindFirstChild(
                "HumanoidRootPart"
            ),

        motors =
            buildMotorMap(
                clone
            ),

        parts =
            parts,

        nameTag =
            nameTag,

        visible =
            true
    }

    setReplayActorVisible(
        actorData,
        false
    )

    return actorData
end

local function restoreReplayCameraAndMovement()
    local camera =
        workspace.CurrentCamera

    if camera then
        pcall(function()

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
        end)
    end

    if humanoid then
        pcall(function()

            if replay.oldWalkSpeed then
                humanoid.WalkSpeed =
                    replay.oldWalkSpeed
            end

            if replay.oldJumpPower then
                humanoid.JumpPower =
                    replay.oldJumpPower
            end

            if replay.oldJumpHeight then
                humanoid.JumpHeight =
                    replay.oldJumpHeight
            end

            if replay.oldAutoRotate ~= nil then
                humanoid.AutoRotate =
                    replay.oldAutoRotate
            end
        end)
    end

    replay.oldCameraType = nil
    replay.oldCameraSubject = nil
    replay.oldCameraCFrame = nil
    replay.oldCameraFov = nil

    replay.oldWalkSpeed = nil
    replay.oldJumpPower = nil
    replay.oldJumpHeight = nil
    replay.oldAutoRotate = nil
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

    restoreReplayCameraAndMovement()

    if replay.playbackFolder then
        pcall(function()
            replay.playbackFolder:Destroy()
        end)

        replay.playbackFolder =
            nil
    end

    table.clear(
        replay.playbackActors
    )

    if esp.enabled then
        enableESP()
    end

    updateReplayStatus(
        "Replay stopped • "
        .. tostring(
            #replay.takes
        )
        .. " saved",
        Colors.SubText
    )
end

local function applyReplayState(
    actorData,
    stateA,
    stateB,
    alpha
)
    if not actorData then
        return
    end

    local state =
        stateA
        or stateB

    if not state then
        setReplayActorVisible(
            actorData,
            false
        )

        return
    end

    setReplayActorVisible(
        actorData,
        true
    )

    local targetCFrame

    if stateA
        and stateB then

        targetCFrame =
            stateA.cframe:Lerp(
                stateB.cframe,
                alpha
            )
    else
        targetCFrame =
            state.cframe
    end

    targetCFrame =
        targetCFrame
        + replay.offset

    if actorData.model
        and actorData.model.Parent then

        pcall(function()
            actorData.model:PivotTo(
                targetCFrame
            )
        end)
    end

    for key, motor in pairs(
        actorData.motors
    ) do
        if motor
            and motor.Parent then

            local transformA =
                stateA
                and stateA.motors
                and stateA.motors[key]
                or nil

            local transformB =
                stateB
                and stateB.motors
                and stateB.motors[key]
                or nil

            if transformA
                and transformB then

                motor.Transform =
                    transformA:Lerp(
                        transformB,
                        alpha
                    )

            elseif transformA then
                motor.Transform =
                    transformA

            elseif transformB then
                motor.Transform =
                    transformB
            end
        end
    end
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

        updateReplayStatus(
            "No recording saved",
            Colors.Danger
        )

        return
    end

    local camera =
        workspace.CurrentCamera

    replay.playing =
        true

    disableESP()

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

        humanoid.WalkSpeed = 0
        humanoid.JumpPower = 0
        humanoid.JumpHeight = 0
        humanoid.AutoRotate = false
    end

    local replayFolder =
        Instance.new("Folder")

    replayFolder.Name =
        "THH_ReplayWorld"

    replayFolder.Parent =
        workspace

    replay.playbackFolder =
        replayFolder

    local mapFolder =
        Instance.new("Folder")

    mapFolder.Name =
        "Map"

    mapFolder.Parent =
        replayFolder

    if take.mapTemplate then
        local mapClone =
            safeClone(
                take.mapTemplate
            )

        if mapClone then
            for _, object in ipairs(
                mapClone:GetChildren()
            ) do
                object.Parent =
                    mapFolder
            end

            mapClone:Destroy()

            shiftReplayWorld(
                mapFolder,
                replay.offset
            )
        end
    end

    local actorsFolder =
        Instance.new("Folder")

    actorsFolder.Name =
        "Players"

    actorsFolder.Parent =
        replayFolder

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

    local playbackStarted =
        os.clock()

    local frameIndex =
        1

    updateReplayStatus(
        "▶ Playing Take "
        .. tostring(
            take.number
        ),
        Colors.Accent
    )

    replay.playbackConnection =
        RunService.RenderStepped:Connect(function()

            if not alive
                or not replay.playing then

                return
            end

            local elapsed =
                os.clock()
                - playbackStarted

            if elapsed >= take.duration then
                stopReplay()
                return
            end

            while frameIndex
                    < #take.frames - 1
                and take.frames[
                    frameIndex + 1
                ].time <= elapsed do

                frameIndex += 1
            end

            local frameA =
                take.frames[
                    frameIndex
                ]

            local frameB =
                take.frames[
                    math.min(
                        frameIndex + 1,
                        #take.frames
                    )
                ]

            local difference =
                math.max(
                    frameB.time
                    - frameA.time,
                    0.0001
                )

            local alpha =
                math.clamp(
                    (
                        elapsed
                        - frameA.time
                    )
                    / difference,
                    0,
                    1
                )

            for id, actorData in pairs(
                replay.playbackActors
            ) do
                applyReplayState(
                    actorData,
                    frameA.players[id],
                    frameB.players[id],
                    alpha
                )
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

                elseif frameA.camera then
                    camera.CFrame =
                        frameA.camera
                        + replay.offset

                    camera.FieldOfView =
                        frameA.fov
                end
            end

            updateReplayStatus(
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
                Colors.Accent
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
        updateReplayStatus(
            "No recording to delete",
            Colors.Danger
        )

        return
    end

    if take.templateFolder then
        take.templateFolder:Destroy()
    end

    if take.mapTemplate then
        take.mapTemplate:Destroy()
    end

    updateReplayStatus(
        "Deleted Take "
        .. tostring(
            take.number
        ),
        Colors.SubText
    )
end

track(
    UserInputService.JumpRequest:Connect(function()

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

track(
    RunService.Heartbeat:Connect(function()

        if not humanoid
            or not rootPart then

            return
        end

        if replay.playing then
            return
        end

        if movement.speedEnabled
            and not coinFarm.enabled
            and not safety.underground then

            if humanoid.WalkSpeed
                ~= movement.speed then

                humanoid.WalkSpeed =
                    movement.speed
            end

        elseif not coinFarm.enabled
            and not safety.underground
            and humanoid.WalkSpeed
                ~= originalWalkSpeed then

            humanoid.WalkSpeed =
                originalWalkSpeed
        end

        if movement.spin then
            humanoid.PlatformStand =
                false

            humanoid.AutoRotate =
                false

            local angular =
                rootPart.AssemblyAngularVelocity

            if math.abs(angular.X) > 0.1
                or math.abs(angular.Z) > 0.1 then

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

            if not coinFarm.enabled then
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
        end
    end)
)

track(
    player.CharacterAdded:Connect(function()

        destroySpin()
        destroyUndergroundForce()

        task.wait(0.2)

        refreshCharacter()

        sheriff.shooting =
            false

        murderer.throwing =
            false

        gunPickup.busy =
            false

        coinFarm.busy =
            false

        for part, connection in pairs(
            localCollision.propertyConnections
        ) do
            disconnect(connection)

            localCollision.propertyConnections[
                part
            ] = nil
        end

        table.clear(
            localCollision.originals
        )

        rebuildLocalCollision()

        if movement.spin then
            rebuildSpin()
        end

        if safety.underground then
            humanoid.CameraOffset =
                Vector3.new(
                    0,
                    safety.depth,
                    0
                )

            local rotation =
                rootPart.CFrame
                - rootPart.Position

            rootPart.CFrame =
                CFrame.new(
                    rootPart.Position
                    - Vector3.new(
                        0,
                        safety.depth,
                        0
                    )
                )
                * rotation

            rebuildUndergroundForce()
        end

        if coinFarm.enabled then
            ensureCoinPlatform()
        end

        if movement.speedEnabled
            and not coinFarm.enabled
            and not safety.underground
            and not replay.playing then

            humanoid.WalkSpeed =
                movement.speed
        end
    end)
)

local function cleanup()
    if not alive then
        return
    end

    if replay.recording then
        replay.recording =
            false
    end

    if replay.playing then
        stopReplay()
    end

    alive =
        false

    sheriff.quickShot =
        false

    sheriff.shooting =
        false

    murderer.autoThrow =
        false

    murderer.throwing =
        false

    gunPickup.auto =
        false

    gunPickup.busy =
        false

    coinFarm.enabled =
        false

    coinFarm.busy =
        false

    movement.speedEnabled =
        false

    movement.infiniteJump =
        false

    movement.spin =
        false

    destroySpin()

    safety.underground =
        false

    destroyUndergroundForce()

    localCollision.sources.noclip =
        false

    localCollision.sources.coinFarm =
        false

    localCollision.sources.underground =
        false

    restoreLocalCollision()

    disableAntiFling()
    disableESP()

    destroyCoinPlatform()

    for _, take in ipairs(
        replay.takes
    ) do
        if take.templateFolder then
            pcall(function()
                take.templateFolder:Destroy()
            end)
        end

        if take.mapTemplate then
            pcall(function()
                take.mapTemplate:Destroy()
            end)
        end
    end

    table.clear(
        replay.takes
    )

    if replay.currentTake then
        if replay.currentTake.templateFolder then
            pcall(function()
                replay.currentTake.templateFolder:Destroy()
            end)
        end

        if replay.currentTake.mapTemplate then
            pcall(function()
                replay.currentTake.mapTemplate:Destroy()
            end)
        end
    end

    replay.currentTake =
        nil

    for targetPlayer, connection in pairs(
        esp.characterConnections
    ) do
        disconnect(connection)

        esp.characterConnections[
            targetPlayer
        ] = nil
    end

    for object, connection in pairs(
        bagTextConnections
    ) do
        disconnect(connection)

        bagTextConnections[object] =
            nil
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

    for _, connection in ipairs(
        connections
    ) do
        disconnect(connection)
    end

    table.clear(
        connections
    )

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

    if _G.THHGrowersCleanup
        == cleanup then

        _G.THHGrowersCleanup =
            nil
    end
end

_G.THHGrowersCleanup =
    cleanup

gui =
    create(
        "ScreenGui",
        {
            Name = "THH_HUB",
            ResetOnSpawn = false,
            IgnoreGuiInset = true,
            DisplayOrder = 999999,
            ZIndexBehavior =
                Enum.ZIndexBehavior.Sibling
        },
        getUIParent()
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

local vampauthClient

local function getHWID()
    local success, result =
        pcall(function()
            return RbxAnalyticsService:GetClientId()
        end)

    if success then
        return result
    end

    return tostring(
        player.UserId
    )
end

local function loadVampauth()
    local success, result =
        pcall(function()

            local source =
                game:HttpGet(
                    "https://vampauth.com/client/vampauth.lua"
                )

            local loader =
                loadstring(source)

            if not loader then
                error(
                    "Vampauth failed"
                )
            end

            return loader()
        end)

    if success
        and type(result)
            == "table" then

        return result
    end

    return nil
end

local function validateKey(key)
    key =
        tostring(
            key or ""
        )
        :gsub(
            "^%s+",
            ""
        )
        :gsub(
            "%s+$",
            ""
        )

    if key == "" then
        return false,
            "enter a key first"
    end

    if not vampauthClient then
        local Vampauth =
            loadVampauth()

        if not Vampauth then
            return false,
                "could not load Vampauth"
        end

        local success, result =
            pcall(function()

                return Vampauth.new({
                    projectId =
                        PROJECT_ID,

                    authSecret =
                        AUTH_SECRET,

                    hwid =
                        getHWID(),

                    debug =
                        false
                })
            end)

        if not success
            or not result then

            return false,
                "authentication failed"
        end

        vampauthClient =
            result
    end

    local success, valid, data =
        pcall(function()

            return vampauthClient:Check(
                key
            )
        end)

    if not success then
        return false,
            "authentication request failed"
    end

    if valid then
        return true,
            data
    end

    local messages = {
        KEY_NOT_FOUND =
            "invalid key",

        KEY_EXPIRED =
            "key expired",

        KEY_BANNED =
            "key banned",

        FINGERPRINT_MISMATCH =
            "key is linked to another device",

        RATE_LIMITED =
            "too many attempts",

        BAD_SIGNATURE =
            "authentication error",

        UNAUTHORIZED =
            "authentication error"
    }

    local reason =
        tostring(
            data
            or "invalid key"
        )

    return false,
        messages[reason]
        or reason
end

local buildChooser
local buildMainMenu

local authOverlay =
    create(
        "Frame",
        {
            Size =
                UDim2.fromScale(
                    1,
                    1
                ),

            BackgroundColor3 =
                Color3.fromRGB(
                    7,
                    8,
                    10
                ),

            BackgroundTransparency =
                0.03,

            BorderSizePixel =
                0
        },
        gui
    )

local authGradient =
    create(
        "UIGradient",
        {
            Rotation =
                -24,

            Color =
                ColorSequence.new({
                    ColorSequenceKeypoint.new(
                        0,
                        Color3.fromRGB(
                            7,
                            8,
                            10
                        )
                    ),

                    ColorSequenceKeypoint.new(
                        0.5,
                        Color3.fromRGB(
                            39,
                            41,
                            47
                        )
                    ),

                    ColorSequenceKeypoint.new(
                        1,
                        Color3.fromRGB(
                            7,
                            8,
                            10
                        )
                    )
                })
        },
        authOverlay
    )

task.spawn(function()
    while alive
        and authOverlay.Parent do

        tween(
            authGradient,
            5,
            {
                Rotation = 24
            }
        )

        task.wait(5)

        if not authOverlay.Parent then
            break
        end

        tween(
            authGradient,
            5,
            {
                Rotation = -24
            }
        )

        task.wait(5)
    end
end)

local authWindow =
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
                    0.9,
                    0,
                    0,
                    345
                ),

            BackgroundColor3 =
                Color3.fromRGB(
                    54,
                    56,
                    63
                ),

            BackgroundTransparency =
                0.16,

            BorderSizePixel =
                0,

            Active =
                true
        },
        authOverlay
    )

create(
    "UISizeConstraint",
    {
        MinSize =
            Vector2.new(
                300,
                335
            ),

        MaxSize =
            Vector2.new(
                430,
                345
            )
    },
    authWindow
)

addCorner(
    authWindow,
    17
)

addStroke(
    authWindow,
    Color3.fromRGB(
        150,
        153,
        160
    ),
    1,
    0.50
)

local authHeader =
    create(
        "Frame",
        {
            Size =
                UDim2.new(
                    1,
                    0,
                    0,
                    78
                ),

            BackgroundTransparency =
                1,

            Active =
                true
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
            Position =
                UDim2.fromOffset(
                    18,
                    15
                ),

            Size =
                UDim2.fromOffset(
                    48,
                    48
                ),

            BackgroundColor3 =
                Colors.Control,

            BackgroundTransparency =
                0.05,

            BorderSizePixel =
                0,

            Image =
                MM2_ICON
        },
        authHeader
    )

addCorner(
    authIcon,
    12
)

create(
    "TextLabel",
    {
        Position =
            UDim2.fromOffset(
                79,
                15
            ),

        Size =
            UDim2.new(
                1,
                -98,
                0,
                24
            ),

        BackgroundTransparency =
            1,

        Text =
            "THH HUB",

        TextColor3 =
            Colors.Text,

        TextSize =
            20,

        Font =
            Enum.Font.GothamBold,

        TextXAlignment =
            Enum.TextXAlignment.Left
    },
    authHeader
)

create(
    "TextLabel",
    {
        Position =
            UDim2.fromOffset(
                79,
                41
            ),

        Size =
            UDim2.new(
                1,
                -98,
                0,
                18
            ),

        BackgroundTransparency =
            1,

        Text =
            "Authentication",

        TextColor3 =
            Colors.SubText,

        TextSize =
            12,

        Font =
            Enum.Font.Gotham,

        TextXAlignment =
            Enum.TextXAlignment.Left
    },
    authHeader
)

local keyBox =
    create(
        "TextBox",
        {
            Position =
                UDim2.fromOffset(
                    18,
                    91
                ),

            Size =
                UDim2.new(
                    1,
                    -36,
                    0,
                    48
                ),

            BackgroundColor3 =
                Colors.Control,

            BackgroundTransparency =
                0.04,

            BorderSizePixel =
                0,

            Text =
                "",

            PlaceholderText =
                "Enter access key",

            PlaceholderColor3 =
                Colors.SubText,

            TextColor3 =
                Colors.Text,

            TextSize =
                14,

            Font =
                Enum.Font.Gotham,

            ClearTextOnFocus =
                false,

            TextXAlignment =
                Enum.TextXAlignment.Left
        },
        authWindow
    )

addCorner(
    keyBox,
    10
)

addStroke(
    keyBox
)

create(
    "UIPadding",
    {
        PaddingLeft =
            UDim.new(
                0,
                14
            ),

        PaddingRight =
            UDim.new(
                0,
                14
            )
    },
    keyBox
)

local continueButton =
    create(
        "TextButton",
        {
            Position =
                UDim2.fromOffset(
                    18,
                    153
                ),

            Size =
                UDim2.new(
                    1,
                    -36,
                    0,
                    45
                ),

            BackgroundColor3 =
                Colors.Accent,

            BorderSizePixel =
                0,

            Text =
                "Continue",

            TextColor3 =
                Color3.fromRGB(
                    18,
                    24,
                    20
                ),

            TextSize =
                14,

            Font =
                Enum.Font.GothamBold,

            AutoButtonColor =
                false
        },
        authWindow
    )

addCorner(
    continueButton,
    10
)

local getKeyButton =
    create(
        "TextButton",
        {
            Position =
                UDim2.fromOffset(
                    18,
                    212
                ),

            Size =
                UDim2.new(
                    1,
                    -36,
                    0,
                    45
                ),

            BackgroundColor3 =
                Colors.Control,

            BackgroundTransparency =
                0.04,

            BorderSizePixel =
                0,

            Text =
                "Get Access Key",

            TextColor3 =
                Colors.Text,

            TextSize =
                13,

            Font =
                Enum.Font.GothamBold,

            AutoButtonColor =
                false
        },
        authWindow
    )

addCorner(
    getKeyButton,
    10
)

animateButton(
    getKeyButton
)

local authStatus =
    create(
        "TextLabel",
        {
            Position =
                UDim2.fromOffset(
                    18,
                    276
                ),

            Size =
                UDim2.new(
                    1,
                    -36,
                    0,
                    34
                ),

            BackgroundTransparency =
                1,

            Text =
                "paste your key",

            TextColor3 =
                Colors.SubText,

            TextSize =
                12,

            Font =
                Enum.Font.Gotham
        },
        authWindow
    )

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
                Colors.Accent
        else
            authStatus.Text =
                KEY_FLOW

            authStatus.TextColor3 =
                Colors.Text
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
            Colors.SubText

        task.spawn(function()

            local valid, result =
                validateKey(
                    keyBox.Text
                )

            if not alive then
                return
            end

            if valid then
                authStatus.Text =
                    "key accepted"

                authStatus.TextColor3 =
                    Colors.Accent

                task.wait(0.12)

                authOverlay:Destroy()

                buildChooser()
            else
                authStatus.Text =
                    tostring(result)

                authStatus.TextColor3 =
                    Colors.Danger

                continueButton.Text =
                    "Continue"

                checkingKey =
                    false
            end
        end)
    end)
)

buildChooser = function()
    local overlay =
        create(
            "Frame",
            {
                Size =
                    UDim2.fromScale(
                        1,
                        1
                    ),

                BackgroundColor3 =
                    Color3.fromRGB(
                        8,
                        9,
                        12
                    ),

                BackgroundTransparency =
                    0.12,

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
                        0.84,
                        0,
                        0,
                        250
                    ),

                BackgroundColor3 =
                    Color3.fromRGB(
                        52,
                        54,
                        61
                    ),

                BackgroundTransparency =
                    0.15,

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
                    240
                ),

            MaxSize =
                Vector2.new(
                    490,
                    250
                )
        },
        panel
    )

    addCorner(
        panel,
        16
    )

    addStroke(
        panel,
        Color3.fromRGB(
            150,
            153,
            160
        ),
        1,
        0.50
    )

    makeDraggable(
        panel
    )

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
                    26
                ),

            BackgroundTransparency =
                1,

            Text =
                "what are you on?",

            TextColor3 =
                Colors.Text,

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
                    46
                ),

            Size =
                UDim2.new(
                    1,
                    -40,
                    0,
                    20
                ),

            BackgroundTransparency =
                1,

            Text =
                "pick your device",

            TextColor3 =
                Colors.SubText,

            TextSize =
                12,

            Font =
                Enum.Font.Gotham
        },
        panel
    )

    local pcButton =
        create(
            "TextButton",
            {
                Position =
                    UDim2.new(
                        0.05,
                        0,
                        0,
                        82
                    ),

                Size =
                    UDim2.new(
                        0.425,
                        0,
                        0,
                        140
                    ),

                BackgroundColor3 =
                    Colors.Control,

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

    addCorner(
        pcButton,
        12
    )

    addStroke(
        pcButton,
        Colors.Stroke,
        1,
        0.58
    )

    local pcIconHolder =
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
                        14
                    ),

                Size =
                    UDim2.fromOffset(
                        92,
                        78
                    ),

                BackgroundTransparency =
                    1
            },
            pcButton
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
                        0
                    ),

                Size =
                    UDim2.fromOffset(
                        84,
                        51
                    ),

                BackgroundColor3 =
                    Color3.fromRGB(
                        27,
                        29,
                        34
                    ),

                BorderSizePixel =
                    0
            },
            pcIconHolder
        )

    addCorner(
        monitor,
        7
    )

    addStroke(
        monitor,
        Colors.Text,
        2,
        0.12
    )

    local monitorScreen =
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
                        42,
                        47,
                        52
                    ),

                BorderSizePixel =
                    0
            },
            monitor
        )

    addCorner(
        monitorScreen,
        4
    )

    create(
        "UIGradient",
        {
            Rotation =
                -25,

            Color =
                ColorSequence.new({
                    ColorSequenceKeypoint.new(
                        0,
                        Color3.fromRGB(
                            42,
                            47,
                            52
                        )
                    ),

                    ColorSequenceKeypoint.new(
                        1,
                        Color3.fromRGB(
                            79,
                            87,
                            96
                        )
                    )
                })
        },
        monitorScreen
    )

    local standStem =
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
                        51
                    ),

                Size =
                    UDim2.fromOffset(
                        7,
                        15
                    ),

                BackgroundColor3 =
                    Colors.Text,

                BorderSizePixel =
                    0
            },
            pcIconHolder
        )

    addCorner(
        standStem,
        2
    )

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
                        65
                    ),

                Size =
                    UDim2.fromOffset(
                        38,
                        6
                    ),

                BackgroundColor3 =
                    Colors.Text,

                BorderSizePixel =
                    0
            },
            pcIconHolder
        )

    addCorner(
        standBase,
        3
    )

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
                        73
                    ),

                Size =
                    UDim2.fromOffset(
                        52,
                        5
                    ),

                BackgroundColor3 =
                    Colors.SubText,

                BorderSizePixel =
                    0
            },
            pcIconHolder
        )

    addCorner(
        keyboard,
        2
    )

    create(
        "TextLabel",
        {
            Position =
                UDim2.new(
                    0,
                    0,
                    1,
                    -37
                ),

            Size =
                UDim2.new(
                    1,
                    0,
                    0,
                    24
                ),

            BackgroundTransparency =
                1,

            Text =
                "PC",

            TextColor3 =
                Colors.Text,

            TextSize =
                15,

            Font =
                Enum.Font.GothamBold
        },
        pcButton
    )

    local phoneButton =
        create(
            "TextButton",
            {
                Position =
                    UDim2.new(
                        0.525,
                        0,
                        0,
                        82
                    ),

                Size =
                    UDim2.new(
                        0.425,
                        0,
                        0,
                        140
                    ),

                BackgroundColor3 =
                    Colors.Control,

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

    addCorner(
        phoneButton,
        12
    )

    addStroke(
        phoneButton,
        Colors.Stroke,
        1,
        0.58
    )

    local phoneIconHolder =
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
                        10
                    ),

                Size =
                    UDim2.fromOffset(
                        68,
                        84
                    ),

                BackgroundTransparency =
                    1
            },
            phoneButton
        )

    local phoneBody =
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
                        0
                    ),

                Size =
                    UDim2.fromOffset(
                        43,
                        79
                    ),

                BackgroundColor3 =
                    Color3.fromRGB(
                        28,
                        30,
                        35
                    ),

                BorderSizePixel =
                    0
            },
            phoneIconHolder
        )

    addCorner(
        phoneBody,
        9
    )

    addStroke(
        phoneBody,
        Colors.Text,
        2,
        0.10
    )

    local phoneScreen =
        create(
            "Frame",
            {
                Position =
                    UDim2.fromOffset(
                        4,
                        8
                    ),

                Size =
                    UDim2.new(
                        1,
                        -8,
                        1,
                        -16
                    ),

                BackgroundColor3 =
                    Color3.fromRGB(
                        42,
                        47,
                        52
                    ),

                BorderSizePixel =
                    0
            },
            phoneBody
        )

    addCorner(
        phoneScreen,
        6
    )

    create(
        "UIGradient",
        {
            Rotation =
                20,

            Color =
                ColorSequence.new({
                    ColorSequenceKeypoint.new(
                        0,
                        Color3.fromRGB(
                            41,
                            46,
                            51
                        )
                    ),

                    ColorSequenceKeypoint.new(
                        1,
                        Color3.fromRGB(
                            77,
                            84,
                            93
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
                        3
                    ),

                Size =
                    UDim2.fromOffset(
                        17,
                        4
                    ),

                BackgroundColor3 =
                    Colors.Text,

                BorderSizePixel =
                    0
            },
            phoneBody
        )

    addCorner(
        notch,
        3
    )

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
                        -3
                    ),

                Size =
                    UDim2.fromOffset(
                        15,
                        3
                    ),

                BackgroundColor3 =
                    Colors.Text,

                BorderSizePixel =
                    0
            },
            phoneBody
        )

    addCorner(
        homeBar,
        3
    )

    create(
        "TextLabel",
        {
            Position =
                UDim2.new(
                    0,
                    0,
                    1,
                    -37
                ),

            Size =
                UDim2.new(
                    1,
                    0,
                    0,
                    24
                ),

            BackgroundTransparency =
                1,

            Text =
                "PHONE",

            TextColor3 =
                Colors.Text,

            TextSize =
                15,

            Font =
                Enum.Font.GothamBold
        },
        phoneButton
    )

    animateButton(
        pcButton
    )

    animateButton(
        phoneButton
    )

    track(
        pcButton.MouseButton1Click:Connect(function()

            overlay:Destroy()

            if blur then
                blur:Destroy()
                blur = nil
            end

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

            buildMainMenu(
                "Phone"
            )
        end)
    )
end

buildMainMenu = function(deviceMode)
    local isPhone =
        deviceMode == "Phone"

    local compactHeight =
        42

    local sliderHeight =
        isPhone
        and 68
        or 60

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
                    Colors.Background,

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

    addCorner(
        menu,
        14
    )

    addStroke(
        menu,
        Colors.Stroke,
        1,
        0.43
    )

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
                    Colors.Background2,

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

    local headerIcon =
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
                    Colors.Control,

                BackgroundTransparency =
                    0.05,

                BorderSizePixel =
                    0,

                Image =
                    MM2_ICON
            },
            header
        )

    addCorner(
        headerIcon,
        9
    )

    create(
        "TextLabel",
        {
            Position =
                UDim2.fromOffset(
                    57,
                    8
                ),

            Size =
                UDim2.new(
                    0,
                    190,
                    0,
                    21
                ),

            BackgroundTransparency =
                1,

            Text =
                "THH HUB",

            TextColor3 =
                Colors.Text,

            TextSize =
                isPhone
                and 17
                or 18,

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
                UDim2.new(
                    0,
                    190,
                    0,
                    15
                ),

            BackgroundTransparency =
                1,

            Text =
                "MM2",

            TextColor3 =
                Colors.SubText,

            TextSize =
                10,

            Font =
                Enum.Font.Gotham,

            TextXAlignment =
                Enum.TextXAlignment.Left
        },
        header
    )

    local minimizeButton =
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
                    Colors.Control,

                BackgroundTransparency =
                    0.05,

                BorderSizePixel =
                    0,

                Text =
                    "—",

                TextColor3 =
                    Colors.Text,

                TextSize =
                    15,

                Font =
                    Enum.Font.GothamBold,

                AutoButtonColor =
                    false
            },
            header
        )

    addCorner(
        minimizeButton,
        8
    )

    local closeButton =
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
                    Colors.Control,

                BackgroundTransparency =
                    0.05,

                BorderSizePixel =
                    0,

                Text =
                    "×",

                TextColor3 =
                    Colors.Danger,

                TextSize =
                    18,

                Font =
                    Enum.Font.GothamBold,

                AutoButtonColor =
                    false
            },
            header
        )

    addCorner(
        closeButton,
        8
    )

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
                    4,

                SortOrder =
                    Enum.SortOrder.LayoutOrder
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
                        1,

                    ClipsDescendants =
                        true
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
                        Colors.Sidebar,

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
                    Enum.HorizontalAlignment.Center,

                SortOrder =
                    Enum.SortOrder.LayoutOrder
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
                        1,

                    ClipsDescendants =
                        true
                },
                menu
            )
    end

    local pages = {}
    local navButtons = {}
    local navOrder = 0

    local function createPage(name)
        local page =
            create(
                "ScrollingFrame",
                {
                    Name =
                        name .. "Page",

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

                    CanvasSize =
                        UDim2.fromOffset(
                            0,
                            0
                        ),

                    AutomaticCanvasSize =
                        Enum.AutomaticSize.Y,

                    ScrollBarThickness =
                        isPhone
                        and 2
                        or 3,

                    ScrollBarImageColor3 =
                        Colors.Accent
                },
                pageHolder
            )

        create(
            "UIListLayout",
            {
                Padding =
                    UDim.new(
                        0,
                        isPhone
                        and 7
                        or 10
                    ),

                SortOrder =
                    Enum.SortOrder.LayoutOrder
            },
            page
        )

        create(
            "UIPadding",
            {
                PaddingLeft =
                    UDim.new(
                        0,
                        isPhone
                        and 4
                        or 13
                    ),

                PaddingRight =
                    UDim.new(
                        0,
                        isPhone
                        and 4
                        or 13
                    ),

                PaddingTop =
                    UDim.new(
                        0,
                        isPhone
                        and 5
                        or 13
                    ),

                PaddingBottom =
                    UDim.new(
                        0,
                        12
                    )
            },
            page
        )

        pages[name] =
            page

        return page
    end

    local function showPage(name)
        for pageName, page in pairs(
            pages
        ) do
            page.Visible =
                pageName == name
        end

        for buttonName, button in pairs(
            navButtons
        ) do
            local selected =
                buttonName == name

            tween(
                button,
                0.12,
                {
                    BackgroundColor3 =
                        selected
                        and Colors.AccentDark
                        or Colors.Control,

                    TextColor3 =
                        selected
                        and Colors.Accent
                        or (
                            isPhone
                            and Colors.Text
                            or Colors.SubText
                        )
                }
            )
        end
    end

    local function createNav(name)
        navOrder += 1

        local button =
            create(
                "TextButton",
                {
                    LayoutOrder =
                        navOrder,

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
                        Colors.Control,

                    BackgroundTransparency =
                        0.05,

                    BorderSizePixel =
                        0,

                    Text =
                        name,

                    TextColor3 =
                        isPhone
                        and Colors.Text
                        or Colors.SubText,

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

        addCorner(
            button,
            8
        )

        navButtons[name] =
            button

        track(
            button.MouseButton1Click:Connect(function()

                showPage(
                    name
                )
            end)
        )
    end

    local function createSection(page, title)
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
                        Colors.Card,

                    BackgroundTransparency =
                        0.21,

                    BorderSizePixel =
                        0
                },
                page
            )

        addCorner(
            card,
            11
        )

        addStroke(
            card,
            Colors.Stroke,
            1,
            0.66
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
                        isPhone
                        and 7
                        or 8
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
                        isPhone
                        and 8
                        or 12
                    ),

                PaddingRight =
                    UDim.new(
                        0,
                        isPhone
                        and 8
                        or 12
                    ),

                PaddingTop =
                    UDim.new(
                        0,
                        isPhone
                        and 8
                        or 11
                    ),

                PaddingBottom =
                    UDim.new(
                        0,
                        isPhone
                        and 8
                        or 11
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
                    Colors.Text,

                TextSize =
                    isPhone
                    and 13
                    or 14,

                Font =
                    Enum.Font.GothamBold,

                TextXAlignment =
                    Enum.TextXAlignment.Left
            },
            holder
        )

        return holder
    end

    local function createAction(
        parent,
        text,
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
                            compactHeight
                        ),

                    BackgroundColor3 =
                        danger
                        and Color3.fromRGB(
                            74,
                            38,
                            42
                        )
                        or Colors.Control,

                    BackgroundTransparency =
                        0.04,

                    BorderSizePixel =
                        0,

                    Text =
                        text,

                    TextColor3 =
                        danger
                        and Colors.Danger
                        or Colors.Text,

                    TextSize =
                        12,

                    Font =
                        Enum.Font.GothamMedium,

                    AutoButtonColor =
                        false
                },
                parent
            )

        addCorner(
            button,
            8
        )

        animateButton(
            button
        )

        track(
            button.MouseButton1Click:Connect(function()

                if callback then
                    callback()
                end
            end)
        )

        return button
    end

    local function createToggle(
        parent,
        text,
        default,
        callback
    )
        local state =
            default == true

        local row =
            create(
                "TextButton",
                {
                    Size =
                        UDim2.new(
                            1,
                            0,
                            0,
                            compactHeight
                        ),

                    BackgroundColor3 =
                        Colors.Control,

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

        addCorner(
            row,
            8
        )

        create(
            "TextLabel",
            {
                Position =
                    UDim2.new(
                        0,
                        11,
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

                BackgroundTransparency =
                    1,

                Text =
                    text,

                TextColor3 =
                    Colors.Text,

                TextSize =
                    12,

                Font =
                    Enum.Font.GothamMedium,

                TextXAlignment =
                    Enum.TextXAlignment.Left
            },
            row
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
                row
            )

        addCorner(
            switch,
            100
        )

        local dotSize =
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
                            dotSize,
                            dotSize
                        ),

                    BorderSizePixel =
                        0
                },
                switch
            )

        addCorner(
            dot,
            100
        )

        local function refresh()
            tween(
                switch,
                0.12,
                {
                    BackgroundColor3 =
                        state
                        and Colors.AccentDark
                        or Colors.Stroke
                }
            )

            tween(
                dot,
                0.12,
                {
                    BackgroundColor3 =
                        state
                        and Colors.Accent
                        or Colors.SubText,

                    Position =
                        state
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
            )
        end

        local controller = {}

        function controller:Get()
            return state
        end

        function controller:Set(
            value,
            runCallback
        )
            state =
                value == true

            refresh()

            if runCallback ~= false
                and callback then

                callback(state)
            end
        end

        refresh()

        track(
            row.MouseButton1Click:Connect(function()

                controller:Set(
                    not state,
                    true
                )
            end)
        )

        return controller
    end

    local function createSlider(
        parent,
        text,
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
                            sliderHeight
                        ),

                    BackgroundColor3 =
                        Colors.Control,

                    BackgroundTransparency =
                        0.04,

                    BorderSizePixel =
                        0
                },
                parent
            )

        addCorner(
            holder,
            8
        )

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
                        -90,
                        0,
                        20
                    ),

                BackgroundTransparency =
                    1,

                Text =
                    text,

                TextColor3 =
                    Colors.Text,

                TextSize =
                    12,

                Font =
                    Enum.Font.GothamMedium,

                TextXAlignment =
                    Enum.TextXAlignment.Left
            },
            holder
        )

        local valueLabel =
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

                    Text =
                        tostring(value),

                    TextColor3 =
                        Colors.Accent,

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
                        Colors.Stroke,

                    BorderSizePixel =
                        0
                },
                holder
            )

        addCorner(
            bar,
            100
        )

        local fill =
            create(
                "Frame",
                {
                    BackgroundColor3 =
                        Colors.Accent,

                    BorderSizePixel =
                        0
                },
                bar
            )

        addCorner(
            fill,
            100
        )

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
                        Colors.Text,

                    BorderSizePixel =
                        0
                },
                bar
            )

        addCorner(
            knob,
            100
        )

        addStroke(
            knob,
            Colors.Accent,
            2,
            0
        )

        local function refresh()
            local percent =
                (value - minimum)
                / (maximum - minimum)

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

            valueLabel.Text =
                tostring(value)
        end

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
                    ) * percent
                    + 0.5
                )

            refresh()

            if callback then
                callback(value)
            end
        end

        refresh()

        local dragging =
            false

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
            UserInputService.InputChanged:Connect(function(
                input
            )
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
            UserInputService.InputEnded:Connect(function(
                input
            )
                if input.UserInputType
                        == Enum.UserInputType.MouseButton1
                    or input.UserInputType
                        == Enum.UserInputType.Touch then

                    dragging =
                        false
                end
            end)
        )
    end

    local Home =
        createPage("Home")

    local SheriffPage =
        createPage("Sheriff")

    local MurdererPage =
        createPage("Murderer")

    local InnocentPage =
        createPage("Innocent")

    local CoinPage =
        createPage("Coin Grab")

    local ReplayPage =
        createPage("Replay Studio")

    local SafetyPage =
        createPage("Safety")

    local UpdatesPage =
        createPage("Updates")

    local UtilityPage =
        createPage("Utility")

    createNav("Home")
    createNav("Sheriff")
    createNav("Murderer")
    createNav("Innocent")
    createNav("Coin Grab")
    createNav("Replay Studio")
    createNav("Safety")
    createNav("Updates")
    createNav("Utility")

    local homeSection =
        createSection(
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
                    27
                ),

            BackgroundTransparency =
                1,

            Text =
                "welcome "
                .. player.DisplayName,

            TextColor3 =
                Colors.Accent,

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
        createSection(
            SheriffPage,
            "Sheriff"
        )

    createToggle(
        sheriffSection,
        "Aim Bot",
        sheriff.quickShot,
        function(value)

            sheriff.quickShot =
                value
        end
    )

    create(
        "TextLabel",
        {
            Size =
                UDim2.new(
                    1,
                    0,
                    0,
                    36
                ),

            BackgroundTransparency =
                1,

            Text =
                "click the knife holder — gun equips, shoots where you clicked, then gets put away",

            TextWrapped =
                true,

            TextColor3 =
                Colors.SubText,

            TextSize =
                10,

            Font =
                Enum.Font.Gotham,

            TextXAlignment =
                Enum.TextXAlignment.Left
        },
        sheriffSection
    )

    createAction(
        sheriffSection,
        "Pick Up Gun Drop",
        pickupGun
    )

    local autoGunToggle1
    local autoGunToggle2

    local function setAutoGun(
        value,
        source
    )
        gunPickup.auto =
            value

        if autoGunToggle1
            and source ~= autoGunToggle1 then

            autoGunToggle1:Set(
                value,
                false
            )
        end

        if autoGunToggle2
            and source ~= autoGunToggle2 then

            autoGunToggle2:Set(
                value,
                false
            )
        end
    end

    autoGunToggle1 =
        createToggle(
            sheriffSection,
            "Auto Pick Up Gun",
            gunPickup.auto,
            function(value)

                setAutoGun(
                    value,
                    autoGunToggle1
                )
            end
        )

    local murdererSection =
        createSection(
            MurdererPage,
            "Murderer"
        )

    createAction(
        murdererSection,
        "Throw Knife",
        function()

            task.spawn(
                throwKnifeOnce
            )
        end
    )

    createToggle(
        murdererSection,
        "Auto Throw Knife",
        murderer.autoThrow,
        function(value)

            murderer.autoThrow =
                value
        end
    )

    createSlider(
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
        createSection(
            InnocentPage,
            "ESP"
        )

    createToggle(
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
        createSection(
            InnocentPage,
            "Gun"
        )

    createAction(
        gunSection,
        "Pick Up Gun",
        pickupGun
    )

    autoGunToggle2 =
        createToggle(
            gunSection,
            "Auto Pick Up Gun",
            gunPickup.auto,
            function(value)

                setAutoGun(
                    value,
                    autoGunToggle2
                )
            end
        )

    local movementSection =
        createSection(
            InnocentPage,
            "Movement"
        )

    createToggle(
        movementSection,
        "Speed",
        movement.speedEnabled,
        function(value)

            setSpeedEnabled(
                value
            )
        end
    )

    createSlider(
        movementSection,
        "Speed",
        16,
        250,
        movement.speed,
        function(value)

            setSpeed(
                value
            )
        end
    )

    createToggle(
        movementSection,
        "Infinite Jump",
        movement.infiniteJump,
        function(value)

            movement.infiniteJump =
                value
        end
    )

    createToggle(
        movementSection,
        "Noclip",
        localCollision.sources.noclip,
        function(value)

            setCollisionSource(
                "noclip",
                value
            )
        end
    )

    createToggle(
        movementSection,
        "Spin Bot",
        movement.spin,
        function(value)

            setSpinEnabled(
                value
            )
        end
    )

    createSlider(
        movementSection,
        "Spin Speed",
        30,
        1500,
        movement.spinSpeed,
        function(value)

            setSpinSpeed(
                value
            )
        end
    )

    local coinSection =
        createSection(
            CoinPage,
            "Coin Grab"
        )

    createToggle(
        coinSection,
        "Coin Farm",
        coinFarm.enabled,
        function(value)

            setCoinFarmEnabled(
                value
            )
        end
    )

    local coinCounter =
        create(
            "TextLabel",
            {
                Size =
                    UDim2.new(
                        1,
                        0,
                        0,
                        compactHeight
                    ),

                BackgroundColor3 =
                    Colors.Control,

                BackgroundTransparency =
                    0.04,

                BorderSizePixel =
                    0,

                Text =
                    "Coins Picked Up: 0",

                TextColor3 =
                    Colors.Accent,

                TextSize =
                    12,

                Font =
                    Enum.Font.GothamBold
            },
            coinSection
        )

    addCorner(
        coinCounter,
        8
    )

    task.spawn(function()
        while alive
            and coinCounter.Parent do

            coinCounter.Text =
                "Coins Picked Up: "
                .. tostring(
                    coinFarm.pickedUp
                )

            task.wait(
                isPhone
                and 0.20
                or 0.12
            )
        end
    end)

    local replaySection =
        createSection(
            ReplayPage,
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
                    Colors.Control,

                BackgroundTransparency =
                    0.04,

                BorderSizePixel =
                    0,

                Text =
                    "Ready • 0 recordings",

                TextColor3 =
                    Colors.SubText,

                TextSize =
                    11,

                TextWrapped =
                    true,

                Font =
                    Enum.Font.GothamMedium
            },
            replaySection
        )

    addCorner(
        replay.statusLabel,
        8
    )

    createAction(
        replaySection,
        "Start Recording",
        function()

            startRecording()
        end
    )

    createAction(
        replaySection,
        "Stop Recording",
        function()

            stopRecording()
        end
    )

    createAction(
        replaySection,
        "View Recorded",
        function()

            viewRecorded()
        end
    )

    createAction(
        replaySection,
        "Stop Replay",
        function()

            stopReplay()
        end
    )

    createAction(
        replaySection,
        "Delete Last Recording",
        function()

            deleteLastRecording()
        end,
        true
    )

    create(
        "TextLabel",
        {
            Size =
                UDim2.new(
                    1,
                    0,
                    0,
                    54
                ),

            BackgroundTransparency =
                1,

            Text =
                "Records player position, rotation, body pose, avatar, names and your camera. Map is frozen when recording starts. Maximum 120 seconds per take.",

            TextWrapped =
                true,

            TextColor3 =
                Colors.SubText,

            TextSize =
                10,

            Font =
                Enum.Font.Gotham,

            TextXAlignment =
                Enum.TextXAlignment.Left,

            TextYAlignment =
                Enum.TextYAlignment.Top
        },
        replaySection
    )

    local safetySection =
        createSection(
            SafetyPage,
            "Safety"
        )

    createToggle(
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

    createToggle(
        safetySection,
        "Underground Safe Mode",
        safety.underground,
        function(value)

            if value then
                enableUnderground()
            else
                disableUnderground(
                    true
                )
            end
        end
    )

    local updatesSection =
        createSection(
            UpdatesPage,
            "Updates"
        )

    local updateLines = {
        "key screen now opens again after unloading",
        "PC and Phone selector now uses real drawn device icons",
        "added Replay Studio",
        "restored Coin Bag Full watcher",
        "Coin Bag Full automatically resets and resumes farm",
        "records all players at 20 samples per second",
        "replays avatar position and body pose",
        "replays recorded names and camera",
        "replay uses a frozen copy of the map",
        "up to 120 seconds per recording",
        "keeps newest 2 recordings this session",
        "gray transparent mod-menu style",
        "phone pages scroll properly",
        "phone sliders are larger",
        "spin bot uses Y-only rotation",
        "aim bot shoots where you manually click or tap",
        "knife throw uses current cursor without locking it",
        "noclip uses shared collision handling",
        "anti fling caches player parts",
        "gun drop detection is cached",
        "coin farm uses cached MainCoin objects",
        "coin farm only targets fully visible MainCoin parts",
        "coin farm moves at speed 20",
        "coin farm stays 9 studs under each coin"
    }

    for _, text in ipairs(
        updateLines
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
                    Colors.SubText,

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
        createSection(
            UtilityPage,
            "Utility"
        )

    createAction(
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

    createAction(
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

    createAction(
        utilitySection,
        "Reset Character",
        function()

            if humanoid then
                humanoid.Health =
                    0
            end
        end
    )

    createAction(
        utilitySection,
        "Unload Menu",
        function()

            cleanup()
        end,
        true
    )

    local floatingButton

    if isPhone then
        floatingButton =
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
                        Colors.Card,

                    BackgroundTransparency =
                        0.10,

                    BorderSizePixel =
                        0,

                    Image =
                        MM2_ICON,

                    Visible =
                        false,

                    AutoButtonColor =
                        false,

                    ZIndex =
                        500
                },
                gui
            )

        addCorner(
            floatingButton,
            14
        )

        addStroke(
            floatingButton,
            Colors.Stroke,
            1,
            0.48
        )
    end

    local function setMenuVisible(value)
        menu.Visible =
            value

        if floatingButton then
            floatingButton.Visible =
                not value
        end
    end

    track(
        minimizeButton.MouseButton1Click:Connect(function()

            setMenuVisible(
                false
            )
        end)
    )

    track(
        closeButton.MouseButton1Click:Connect(function()

            setMenuVisible(
                false
            )
        end)
    )

    if floatingButton then
        track(
            floatingButton.MouseButton1Click:Connect(function()

                setMenuVisible(
                    true
                )
            end)
        )
    end

    track(
        UserInputService.InputBegan:Connect(function(
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
        end)
    )

    showPage(
        "Home"
    )
end
