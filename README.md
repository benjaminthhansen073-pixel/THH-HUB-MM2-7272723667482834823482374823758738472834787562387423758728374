--// ============================================================
--// THH HUB - FULL SINGLE LOCALSCRIPT
--// PC + COMPACT PHONE
--//
--// CURRENT FIXES:
--//
--// Aim Bot:
--//   - finds alive player with Knife
--//   - equips YOUR Gun
--//   - DOES NOT move camera
--//   - moves screen/mouse aim point onto murderer
--//   - clicks / activates Gun
--//   - unequips Gun
--//
--// Pick Up Gun:
--//   - teleports YOU to GunDrop
--//   - gun itself does NOT teleport
--//   - waits 0.001
--//   - teleports YOU back
--//
--// Nvis:
--//   - named exactly Nvis
--//
--// 0 Grab:
--//   - GrabParts / DragPart / DragAttach
--//   - zero grab distance
--//
--// Grab No Fall:
--//   - invisible floor/hitbox
--//   - does NOT anchor player
--//   - Sit forced false
--//   - PlatformStand forced false
--//   - WALK + JUMP still work
--//
--// ============================================================


--// ============================================================
--// CLEAN OLD COPY
--// ============================================================

if _G.THHGrowersHardUnloaded then
    return
end

if _G.THHGrowersCleanup then
    pcall(_G.THHGrowersCleanup)
end


--// ============================================================
--// SERVICES
--// ============================================================

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local TeleportService = game:GetService("TeleportService")
local RbxAnalyticsService = game:GetService("RbxAnalyticsService")
local Lighting = game:GetService("Lighting")
local ReplicatedFirst = game:GetService("ReplicatedFirst")

local VirtualInputManager

pcall(function()
    VirtualInputManager = game:GetService("VirtualInputManager")
end)


--// ============================================================
--// PLAYER
--// ============================================================

local player = Players.LocalPlayer

local character
local humanoid
local rootPart

local originalWalkSpeed = 16


--// ============================================================
--// VAMPAUTH
--// ============================================================

local PROJECT_ID = "PD6XH2EGXZXUHGED"

local AUTH_SECRET =
    "04cce9295d9d2476cb2516b3363693a3c401767b4e75bd01"

local KEY_FLOW =
    "https://vampauth.com/PD6XH2EGXZXUHGED/flow"


--// ============================================================
--// GAME
--// ============================================================

local MM2_GAME_ID = 66654135

local MM2_ICON =
    "rbxthumb://type=GameIcon&id="
    .. tostring(MM2_GAME_ID)
    .. "&w=150&h=150"


--// ============================================================
--// THEME
--// ============================================================

local Colors = {

    Background =
        Color3.fromRGB(
            31,
            32,
            36
        ),

    Sidebar =
        Color3.fromRGB(
            37,
            38,
            43
        ),

    Card =
        Color3.fromRGB(
            62,
            64,
            70
        ),

    Control =
        Color3.fromRGB(
            48,
            50,
            55
        ),

    Text =
        Color3.fromRGB(
            244,
            245,
            247
        ),

    SubText =
        Color3.fromRGB(
            164,
            166,
            174
        ),

    Accent =
        Color3.fromRGB(
            106,
            221,
            145
        ),

    AccentDark =
        Color3.fromRGB(
            45,
            81,
            59
        ),

    Stroke =
        Color3.fromRGB(
            102,
            105,
            114
        ),

    Danger =
        Color3.fromRGB(
            232,
            81,
            81
        ),

    Innocent =
        Color3.fromRGB(
            72,
            226,
            113
        ),

    Murderer =
        Color3.fromRGB(
            243,
            72,
            72
        ),

    Sheriff =
        Color3.fromRGB(
            69,
            147,
            255
        )
}


--// ============================================================
--// MAIN STATE
--// ============================================================

local alive = true

local gui
local blur

local connections = {}


--// ============================================================
--// MOVEMENT
--// ============================================================

local movement = {

    speedEnabled = false,
    speed = 32,

    infiniteJump = false,

    noclip = false,
    noclipParts = {},

    spin = false,
    spinSpeed = 300,

    spinAttachment = nil,
    spinVelocity = nil,

    oldAutoRotate = true
}


--// ============================================================
--// SHERIFF
--// ============================================================

local sheriff = {

    quickShot = false,

    shooting = false
}


--// ============================================================
--// MURDERER
--// ============================================================

local murderer = {

    autoThrow = false,

    throwing = false,

    throwDelay = 0.2,

    lastThrow = 0
}


--// ============================================================
--// GUN PICKUP
--// ============================================================

local gunPickup = {

    auto = false,

    busy = false,

    cachedDrop = nil
}


--// ============================================================
--// ESP
--// ============================================================

local esp = {

    enabled = true,

    highlights = {},

    characterConnections = {}
}


--// ============================================================
--// ANTI FLING
--// ============================================================

local antiFling = {

    enabled = false,

    originalCollision = {},

    trackedParts = {},

    partConnections = {},

    characterConnections = {},

    playerConnections = {},

    playerAddedConnection = nil,

    playerRemovingConnection = nil,

    enforceThread = nil
}


--// ============================================================
--// COIN FARM
--// ============================================================

local coinFarm = {

    enabled = false,

    underDistance = 9,

    travelSpeed = 95,

    verticalSpeed = 420,

    coins = {},

    coinLookup = {},

    target = nil,

    pickedUp = 0,

    busy = false,

    fullBagDebounce = false,

    platform = nil,

    originalCollision = {},

    characterConnection = nil
}


--// ============================================================
--// UNDERGROUND SAFE MODE
--// ============================================================

local safety = {

    underground = false,

    depth = 300,

    originalCollision = {},

    descendantConnection = nil
}


--// ============================================================
--// NVIS
--// ============================================================

local nvis = {

    enabled = false,

    saved = {},

    connection = nil
}


--// ============================================================
--// ZERO GRAB
--// ============================================================

local zeroGrab = {

    enabled = false,

    containers = {},

    savedValues = {},

    savedAttachments = {},

    lastScan = 0
}


--// ============================================================
--// GRAB NO FALL
--// ============================================================

local grabNoFall = {

    enabled = false,

    floor = nil,

    connection = nil,

    safeGroundY = nil,

    floorSize =
        Vector3.new(
            7,
            0.45,
            7
        )
}


--// ============================================================
--// GENERAL HELPERS
--// ============================================================

local function track(connection)

    table.insert(
        connections,
        connection
    )

    return connection
end


local function create(
    className,
    properties,
    parent
)

    local object =
        Instance.new(
            className
        )

    for property, value in pairs(
        properties or {}
    ) do

        object[property] =
            value

    end

    if parent then

        object.Parent =
            parent

    end

    return object
end


local function addCorner(
    object,
    radius
)

    return create(
        "UICorner",
        {
            CornerRadius =
                UDim.new(
                    0,
                    radius or 9
                )
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
            Color =
                color
                or
                Colors.Stroke,

            Thickness =
                thickness
                or
                1,

            Transparency =
                transparency
                or
                0.45
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
            pcall(
                gethui
            )

        if success
            and result then

            return result

        end

    end

    return player:
        WaitForChild(
            "PlayerGui"
        )
end


--// ============================================================
--// CHARACTER
--// ============================================================

local function refreshCharacter()

    character =
        player.Character
        or
        player.CharacterAdded:
        Wait()

    humanoid =
        character:
        FindFirstChildOfClass(
            "Humanoid"
        )
        or
        character:
        WaitForChild(
            "Humanoid"
        )

    rootPart =
        character:
        FindFirstChild(
            "HumanoidRootPart"
        )
        or
        character:
        WaitForChild(
            "HumanoidRootPart"
        )

    originalWalkSpeed =
        humanoid.WalkSpeed

end


refreshCharacter()


--// ============================================================
--// TOOL HELPERS
--// ============================================================

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

    local targetCharacter =
        targetPlayer.Character

    if targetCharacter then

        for _, object in ipairs(
            targetCharacter:
            GetChildren()
        ) do

            if object:IsA("Tool")
                and
                string.lower(
                    object.Name
                ) == wantedName
            then

                return object

            end

        end

    end


    local backpack =
        targetPlayer:
        FindFirstChildOfClass(
            "Backpack"
        )

    if backpack then

        for _, object in ipairs(
            backpack:
            GetChildren()
        ) do

            if object:IsA("Tool")
                and
                string.lower(
                    object.Name
                ) == wantedName
            then

                return object

            end

        end

    end


    return nil

end


local function hasKnife(
    targetPlayer
)

    return findTool(
        targetPlayer,
        "Knife"
    ) ~= nil

end


local function hasGun(
    targetPlayer
)

    if findTool(
        targetPlayer,
        "Gun"
    ) then

        return true

    end


    local backpack =
        targetPlayer:
        FindFirstChildOfClass(
            "Backpack"
        )


    local containers = {

        targetPlayer.Character,

        backpack

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
                        or
                        lower:find(
                            "revolver",
                            1,
                            true
                        )
                    then

                        return true

                    end

                end

            end

        end

    end


    return false

end


local function getGunTool()

    local gun =
        findTool(
            player,
            "Gun"
        )

    if gun then
        return gun
    end


    local backpack =
        player:
        FindFirstChildOfClass(
            "Backpack"
        )


    local containers = {

        character,

        backpack

    }


    for _, container in ipairs(
        containers
    ) do

        if container then

            for _, object in ipairs(
                container:
                GetChildren()
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
                        or
                        lower:find(
                            "revolver",
                            1,
                            true
                        )
                    then

                        return object

                    end

                end

            end

        end

    end


    return nil

end


--// ============================================================
--// ROLE COLOR
--// ============================================================

local function getRoleColor(
    targetPlayer
)

    if hasKnife(
        targetPlayer
    ) then

        return Colors.Murderer

    end


    if hasGun(
        targetPlayer
    ) then

        return Colors.Sheriff

    end


    return Colors.Innocent

end


--// ============================================================
--// ESP
--// ============================================================

local function removePlayerESP(
    targetPlayer
)

    local highlight =
        esp.highlights[
            targetPlayer
        ]

    if highlight then

        pcall(function()

            highlight:
            Destroy()

        end)

        esp.highlights[
            targetPlayer
        ] = nil

    end

end


local function addPlayerESP(
    targetPlayer
)

    if not esp.enabled
        or
        targetPlayer == player
        or
        not targetPlayer.Character
    then

        return

    end


    removePlayerESP(
        targetPlayer
    )


    local highlight =
        Instance.new(
            "Highlight"
        )


    highlight.Name =
        "THH_ESP_"
        ..
        targetPlayer.Name


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


    esp.highlights[
        targetPlayer
    ] =
        highlight

end


local function setupESPPlayer(
    targetPlayer
)

    if targetPlayer == player then
        return
    end


    if esp.enabled then

        addPlayerESP(
            targetPlayer
        )

    end


    if esp.characterConnections[
        targetPlayer
    ] then

        pcall(function()

            esp.characterConnections[
                targetPlayer
            ]:
            Disconnect()

        end)

    end


    esp.characterConnections[
        targetPlayer
    ] =
        targetPlayer.CharacterAdded:
        Connect(function()

            removePlayerESP(
                targetPlayer
            )

            task.wait(
                0.1
            )

            if alive
                and
                esp.enabled
            then

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


    for _, highlight in pairs(
        esp.highlights
    ) do

        if highlight then

            pcall(function()

                highlight:
                Destroy()

            end)

        end

    end


    table.clear(
        esp.highlights
    )

end


for _, targetPlayer in ipairs(
    Players:GetPlayers()
) do

    setupESPPlayer(
        targetPlayer
    )

end


track(
    Players.PlayerAdded:
    Connect(function(
        targetPlayer
    )

        setupESPPlayer(
            targetPlayer
        )

    end)
)


track(
    Players.PlayerRemoving:
    Connect(function(
        targetPlayer
    )

        removePlayerESP(
            targetPlayer
        )

    end)
)


task.spawn(function()

    while alive do

        if esp.enabled then

            for _, targetPlayer in ipairs(
                Players:GetPlayers()
            ) do

                if targetPlayer ~= player
                    and
                    targetPlayer.Character
                then

                    local highlight =
                        esp.highlights[
                            targetPlayer
                        ]

                    if not highlight
                        or
                        not highlight.Parent
                    then

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


        task.wait(
            0.1
        )

    end

end)


--// ============================================================
--// AIM BOT
--//
--// NO CAMERA MOVEMENT.
--//
--// Player with Knife -> screen position -> mouse/click -> Gun.
--// ============================================================

local function getMurdererPlayer()

    local bestPlayer =
        nil

    local bestDistance =
        math.huge


    for _, targetPlayer in ipairs(
        Players:GetPlayers()
    ) do

        if targetPlayer ~= player
            and
            targetPlayer.Character
            and
            hasKnife(
                targetPlayer
            )
        then

            local targetHumanoid =
                targetPlayer.Character:
                FindFirstChildOfClass(
                    "Humanoid"
                )


            local targetRoot =
                targetPlayer.Character:
                FindFirstChild(
                    "HumanoidRootPart"
                )

                or

                targetPlayer.Character:
                FindFirstChild(
                    "UpperTorso"
                )

                or

                targetPlayer.Character:
                FindFirstChild(
                    "Torso"
                )

                or

                targetPlayer.Character:
                FindFirstChild(
                    "Head"
                )


            if targetHumanoid
                and
                targetHumanoid.Health > 0
                and
                targetRoot
            then

                local distance = 0


                if rootPart then

                    distance =
                        (
                            targetRoot.Position
                            -
                            rootPart.Position
                        ).Magnitude

                end


                if not bestPlayer
                    or
                    distance < bestDistance
                then

                    bestPlayer =
                        targetPlayer

                    bestDistance =
                        distance

                end

            end

        end

    end


    return bestPlayer

end


local function getMurdererAimPart(
    targetPlayer
)

    if not targetPlayer
        or
        not targetPlayer.Character
    then

        return nil

    end


    return

        targetPlayer.Character:
        FindFirstChild(
            "HumanoidRootPart"
        )

        or

        targetPlayer.Character:
        FindFirstChild(
            "UpperTorso"
        )

        or

        targetPlayer.Character:
        FindFirstChild(
            "Torso"
        )

        or

        targetPlayer.Character:
        FindFirstChild(
            "Head"
        )

end


local function getTargetScreenPoint(
    targetPart
)

    if not targetPart
        or
        not targetPart.Parent
    then

        return nil

    end


    local camera =
        workspace.CurrentCamera


    if not camera then

        return nil

    end


    local screenPosition,
    onScreen =

        camera:
        WorldToScreenPoint(
            targetPart.Position
        )


    if not onScreen
        or
        screenPosition.Z <= 0
    then

        return nil

    end


    return Vector2.new(

        math.floor(
            screenPosition.X
            +
            0.5
        ),

        math.floor(
            screenPosition.Y
            +
            0.5
        )

    )

end


local function moveShotPointer(
    screenPoint
)

    if not screenPoint then
        return false
    end


    local x =
        screenPoint.X

    local y =
        screenPoint.Y


    local moved =
        false


    if type(
        mousemoveabs
    ) == "function"
    then

        local success =
            pcall(function()

                mousemoveabs(
                    x,
                    y
                )

            end)

        if success then

            moved =
                true

        end

    end


    if not moved
        and
        type(
            mousemoverel
        ) == "function"
    then

        local success =
            pcall(function()

                local current =
                    UserInputService:
                    GetMouseLocation()


                mousemoverel(

                    x
                    -
                    current.X,

                    y
                    -
                    current.Y

                )

            end)

        if success then

            moved =
                true

        end

    end


    if VirtualInputManager then

        local success =
            pcall(function()

                VirtualInputManager:
                SendMouseMoveEvent(

                    x,
                    y,

                    game

                )

            end)

        if success then

            moved =
                true

        end

    end


    return moved

end


local function fireGunAtScreenPoint(
    gun,
    screenPoint
)

    if not gun
        or
        not screenPoint
    then

        return false

    end


    local x =
        screenPoint.X

    local y =
        screenPoint.Y


    moveShotPointer(
        screenPoint
    )


    RunService.RenderStepped:
    Wait()


    local fired =
        false


    if type(
        mouse1press
    ) == "function"
        and
        type(
            mouse1release
        ) == "function"
    then

        local success =
            pcall(function()

                mouse1press()

                task.wait(
                    0.015
                )

                mouse1release()

            end)

        if success then

            fired =
                true

        end


    elseif type(
        mouse1click
    ) == "function"
    then

        local success =
            pcall(function()

                mouse1click()

            end)

        if success then

            fired =
                true

        end

    end


    if not fired
        and
        VirtualInputManager
    then

        local success =
            pcall(function()

                VirtualInputManager:
                SendMouseButtonEvent(

                    x,
                    y,

                    0,
                    true,

                    game,

                    0

                )


                task.wait(
                    0.015
                )


                VirtualInputManager:
                SendMouseButtonEvent(

                    x,
                    y,

                    0,
                    false,

                    game,

                    0

                )

            end)

        if success then

            fired =
                true

        end

    end


    if not fired then

        local success =
            pcall(function()

                gun:
                Activate()

            end)

        if success then

            fired =
                true

        end

    end


    return fired

end


local function quickShoot()

    if sheriff.shooting
        or
        not sheriff.quickShot
        or
        not humanoid
        or
        not character
    then

        return false

    end


    local murdererPlayer =
        getMurdererPlayer()


    if not murdererPlayer then

        return false

    end


    local targetPart =
        getMurdererAimPart(
            murdererPlayer
        )


    if not targetPart then

        return false

    end


    local screenPoint =
        getTargetScreenPoint(
            targetPart
        )


    -- Don't snap camera if murderer is off screen.

    if not screenPoint then

        return false

    end


    local gun =
        getGunTool()


    if not gun then

        return false

    end


    sheriff.shooting =
        true


    task.spawn(function()

        local success,
        errorMessage =

            pcall(function()


                if not murdererPlayer.Parent
                    or
                    not murdererPlayer.Character
                    or
                    not hasKnife(
                        murdererPlayer
                    )
                then

                    return

                end


                local targetHumanoid =
                    murdererPlayer.Character:
                    FindFirstChildOfClass(
                        "Humanoid"
                    )


                if not targetHumanoid
                    or
                    targetHumanoid.Health <= 0
                then

                    return

                end


                -- Equip real Gun Tool.

                if gun.Parent
                    ~= character
                then

                    humanoid:
                    EquipTool(
                        gun
                    )


                    local started =
                        os.clock()


                    while alive
                        and
                        gun.Parent ~= character
                        and
                        os.clock()
                        -
                        started
                        <
                        0.4
                    do

                        RunService.Heartbeat:
                        Wait()

                    end

                end


                if gun.Parent
                    ~= character
                then

                    return

                end


                targetPart =
                    getMurdererAimPart(
                        murdererPlayer
                    )


                screenPoint =
                    getTargetScreenPoint(
                        targetPart
                    )


                if not screenPoint then

                    return

                end


                moveShotPointer(
                    screenPoint
                )


                RunService.RenderStepped:
                Wait()


                -- Re-check the Knife right before shot.

                if not murdererPlayer.Character
                    or
                    not hasKnife(
                        murdererPlayer
                    )
                then

                    return

                end


                targetPart =
                    getMurdererAimPart(
                        murdererPlayer
                    )


                screenPoint =
                    getTargetScreenPoint(
                        targetPart
                    )


                if not screenPoint then

                    return

                end


                fireGunAtScreenPoint(

                    gun,

                    screenPoint

                )


                task.wait(
                    0.07
                )


                pcall(function()

                    gun:
                    Deactivate()

                end)

            end)


        -- Put gun away.

        if humanoid
            and
            humanoid.Parent
            and
            gun
            and
            gun.Parent == character
        then

            pcall(function()

                humanoid:
                UnequipTools()

            end)

        end


        sheriff.shooting =
            false


        if not success then

            warn(
                "[THH Aim Bot] "
                ..
                tostring(
                    errorMessage
                )
            )

        end

    end)


    return true

end


track(
    UserInputService.InputBegan:
    Connect(function(
        input,
        gameProcessed
    )

        if gameProcessed
            or
            not sheriff.quickShot
        then

            return

        end


        if input.UserInputType
                ==
                Enum.UserInputType.MouseButton1

            or

            input.UserInputType
                ==
                Enum.UserInputType.Touch
        then

            quickShoot()

        end

    end)
)


--// ============================================================
--// KNIFE THROW
--// ============================================================

local function pressThrowKey()

    if UserInputService:
        GetFocusedTextBox()
    then

        return false

    end


    if keypress
        and
        keyrelease
    then

        local success =
            pcall(function()

                keypress(
                    0x45
                )

                task.wait(
                    0.05
                )

                keyrelease(
                    0x45
                )

            end)

        if success then

            return true

        end

    end


    if not VirtualInputManager then

        return false

    end


    return pcall(function()

        VirtualInputManager:
        SendKeyEvent(

            true,

            Enum.KeyCode.E,

            false,

            game

        )


        task.wait(
            0.05
        )


        VirtualInputManager:
        SendKeyEvent(

            false,

            Enum.KeyCode.E,

            false,

            game

        )

    end)

end


local function findVisibleThrowButton()

    local playerGui =
        player:
        FindFirstChildOfClass(
            "PlayerGui"
        )

    if not playerGui then
        return nil
    end


    for _, object in ipairs(
        playerGui:
        GetDescendants()
    ) do

        if object:IsA(
            "GuiButton"
        )
            and
            object.Visible
        then

            local name =
                string.lower(
                    object.Name
                )


            local text = ""


            if object:IsA(
                "TextButton"
            ) then

                text =
                    string.lower(
                        object.Text
                        or
                        ""
                    )

            end


            if name:find(
                    "throw",
                    1,
                    true
                )
                or
                text:find(
                    "throw",
                    1,
                    true
                )
            then

                return object

            end

        end

    end


    return nil

end


local function pressMobileThrowButton()

    if not VirtualInputManager then
        return false
    end


    local button =
        findVisibleThrowButton()


    if not button then
        return false
    end


    local center =
        button.AbsolutePosition
        +
        button.AbsoluteSize
        /
        2


    return pcall(function()

        VirtualInputManager:
        SendMouseButtonEvent(

            center.X,
            center.Y,

            0,

            true,

            game,

            0

        )


        task.wait(
            0.03
        )


        VirtualInputManager:
        SendMouseButtonEvent(

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

    if murderer.throwing
        or
        not humanoid
        or
        not character
    then

        return false

    end


    local knife =
        findTool(
            player,
            "Knife"
        )


    if not knife then

        return false

    end


    murderer.throwing =
        true


    local success =
        false


    if knife.Parent
        ~= character
    then

        pcall(function()

            humanoid:
            EquipTool(
                knife
            )

        end)


        RunService.Heartbeat:
        Wait()

    end


    if knife.Parent
        == character
    then

        success =
            pressThrowKey()


        if not success
            and
            UserInputService.TouchEnabled
        then

            success =
                pressMobileThrowButton()

        end

    end


    task.wait(
        0.03
    )


    murderer.throwing =
        false


    return success

end


--// ============================================================
--// PICK UP GUN
--// ============================================================

local function isGunDropName(
    name
)

    name =
        string.lower(
            tostring(
                name
            )
        )
        :
        gsub(
            "[%s_%-%./]",
            ""
        )


    return

        name == "gundrop"

        or

        name == "droppedgun"

        or

        name == "dropgun"

        or

        name == "droppedrevolver"

end


local function findInitialGunDrop()

    local exact =
        workspace:
        FindFirstChild(
            "GunDrop",
            true
        )


    if exact then

        return exact

    end


    for _, object in ipairs(
        workspace:
        GetDescendants()
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
    workspace.DescendantAdded:
    Connect(function(
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
    workspace.DescendantRemoving:
    Connect(function(
        object
    )

        if gunPickup.cachedDrop
            ==
            object
        then

            gunPickup.cachedDrop =
                nil

        end

    end)
)


local function findGunDrop()

    local cached =
        gunPickup.cachedDrop


    if cached
        and
        cached.Parent
        and
        cached:IsDescendantOf(
            workspace
        )
    then

        return cached

    end


    local exact =
        workspace:
        FindFirstChild(
            "GunDrop",
            true
        )


    if exact then

        gunPickup.cachedDrop =
            exact

        return exact

    end


    return nil

end


local function getObjectPart(
    object
)

    if not object then
        return nil
    end


    if object:IsA(
        "BasePart"
    ) then

        return object

    end


    if object:IsA(
        "Tool"
    ) then

        local handle =
            object:
            FindFirstChild(
                "Handle"
            )

        if handle
            and
            handle:IsA(
                "BasePart"
            )
        then

            return handle

        end

    end


    if object:IsA(
        "Model"
    )
        and
        object.PrimaryPart
    then

        return object.PrimaryPart

    end


    return object:
        FindFirstChildWhichIsA(
            "BasePart",
            true
        )

end


local function touchObject(
    object,
    part
)

    if not object
        or
        not part
        or
        not rootPart
    then

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


    for _, descendant in ipairs(
        object:
        GetDescendants()
    ) do

        if descendant:IsA(
            "ProximityPrompt"
        )
            and
            fireproximityprompt
        then

            pcall(function()

                fireproximityprompt(
                    descendant,
                    0
                )

            end)

        end

    end

end


local function pickupGun()

    if gunPickup.busy
        or
        not rootPart
        or
        hasGun(
            player
        )
    then

        return false

    end


    local drop =
        findGunDrop()


    local part =
        getObjectPart(
            drop
        )


    if not drop
        or
        not part
    then

        return false

    end


    gunPickup.busy =
        true


    local oldCFrame =
        rootPart.CFrame


    local oldVelocity =
        rootPart.AssemblyLinearVelocity


    local oldAngular =
        rootPart.AssemblyAngularVelocity


    local success =
        pcall(function()


            rootPart.AssemblyLinearVelocity =
                Vector3.zero


            rootPart.AssemblyAngularVelocity =
                Vector3.zero


            -- TELEPORT PLAYER TO GUN.

            rootPart.CFrame =
                part.CFrame
                *
                CFrame.new(
                    0,
                    0.35,
                    0
                )


            touchObject(
                drop,
                part
            )


            -- Requested return delay.

            task.wait(
                0.001
            )


            touchObject(
                drop,
                part
            )


            if rootPart
                and
                rootPart.Parent
            then


                rootPart.CFrame =
                    oldCFrame


                rootPart.AssemblyLinearVelocity =
                    oldVelocity


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
            and
            not hasGun(
                player
            )
        then

            if findGunDrop() then

                pickupGun()

            end

        end


        task.wait(
            0.05
        )

    end

end)


--// ============================================================
--// NVIS
--// ============================================================

local function applyNvisPart(
    object
)

    if object:IsA(
        "BasePart"
    ) then


        if nvis.saved[
            object
        ] == nil
        then

            nvis.saved[
                object
            ] = {

                type =
                    "part",

                value =
                    object.LocalTransparencyModifier
            }

        end


        object.LocalTransparencyModifier =
            1


    elseif object:IsA(
        "Decal"
    )
        or
        object:IsA(
            "Texture"
        )
    then


        if nvis.saved[
            object
        ] == nil
        then

            nvis.saved[
                object
            ] = {

                type =
                    "texture",

                value =
                    object.Transparency
            }

        end


        object.Transparency =
            1

    end

end


local function enableNvis()

    nvis.enabled =
        true


    if not character then
        return
    end


    for _, object in ipairs(
        character:
        GetDescendants()
    ) do

        applyNvisPart(
            object
        )

    end


    if nvis.connection then

        nvis.connection:
        Disconnect()

    end


    nvis.connection =
        character.DescendantAdded:
        Connect(function(
            object
        )

            if not nvis.enabled then
                return
            end


            task.defer(function()

                if object.Parent then

                    applyNvisPart(
                        object
                    )

                end

            end)

        end)

end


local function disableNvis()

    nvis.enabled =
        false


    if nvis.connection then

        nvis.connection:
        Disconnect()

        nvis.connection =
            nil

    end


    for object, info in pairs(
        nvis.saved
    ) do

        if object
            and
            object.Parent
        then

            pcall(function()

                if info.type
                    ==
                    "part"
                then

                    object.LocalTransparencyModifier =
                        info.value

                else

                    object.Transparency =
                        info.value

                end

            end)

        end

    end


    table.clear(
        nvis.saved
    )

end


--// ============================================================
--// 0 GRAB
--// ============================================================

local function isGrabDistanceValueName(
    name
)

    name =
        string.lower(
            tostring(
                name
            )
        )


    return

        name ==
        "distance"

        or

        name ==
        "grabdistance"

        or

        name ==
        "dragdistance"

        or

        name ==
        "currentdistance"

        or

        name ==
        "mindistance"

        or

        name ==
        "targetdistance"

end


local function zeroGrabContainer(
    container
)

    for _, object in ipairs(
        container:
        GetDescendants()
    ) do


        if (
            object:IsA(
                "NumberValue"
            )
            or
            object:IsA(
                "IntValue"
            )
        )
            and
            isGrabDistanceValueName(
                object.Name
            )
        then


            if zeroGrab.savedValues[
                object
            ] == nil
            then

                zeroGrab.savedValues[
                    object
                ] =
                    object.Value

            end


            object.Value =
                0


        elseif object:IsA(
            "Attachment"
        )
            and
            string.lower(
                object.Name
            )
            ==
            "dragattach"
        then


            if zeroGrab.savedAttachments[
                object
            ] == nil
            then

                zeroGrab.savedAttachments[
                    object
                ] =
                    object.Position

            end


            object.Position =
                Vector3.zero

        end

    end

end


local function scanGrabContainers()

    table.clear(
        zeroGrab.containers
    )


    local seen = {}


    local function scan(
        root
    )

        if not root then
            return
        end


        if string.lower(
            root.Name
        ) == "grabparts"
            and
            not seen[root]
        then

            seen[root] =
                true

            table.insert(
                zeroGrab.containers,
                root
            )

        end


        for _, object in ipairs(
            root:
            GetDescendants()
        ) do

            if string.lower(
                object.Name
            ) == "grabparts"
                and
                not seen[
                    object
                ]
            then

                seen[object] =
                    true

                table.insert(
                    zeroGrab.containers,
                    object
                )

            end

        end

    end


    scan(
        workspace
    )


    scan(
        ReplicatedFirst
    )


    for _, container in ipairs(
        zeroGrab.containers
    ) do

        zeroGrabContainer(
            container
        )

    end

end


local function enforceZeroGrab()

    if not zeroGrab.enabled then
        return
    end


    local camera =
        workspace.CurrentCamera


    if not camera then
        return
    end


    if os.clock()
        -
        zeroGrab.lastScan
        >=
        0.45
    then

        zeroGrab.lastScan =
            os.clock()

        scanGrabContainers()

    end


    for _, container in ipairs(
        zeroGrab.containers
    ) do

        if container
            and
            container.Parent
        then


            zeroGrabContainer(
                container
            )


            local dragPart =
                container:
                FindFirstChild(
                    "DragPart",
                    true
                )


            local dragAttach =
                container:
                FindFirstChild(
                    "DragAttach",
                    true
                )


            if dragPart
                and
                dragPart:IsA(
                    "BasePart"
                )
            then

                pcall(function()

                    -- FULL 0 DISTANCE.

                    dragPart.CFrame =
                        camera.CFrame


                    dragPart.AssemblyLinearVelocity =
                        Vector3.zero


                    dragPart.AssemblyAngularVelocity =
                        Vector3.zero

                end)

            end


            if dragAttach
                and
                dragAttach:IsA(
                    "Attachment"
                )
            then

                dragAttach.Position =
                    Vector3.zero

            end

        end

    end

end


local function enableZeroGrab()

    zeroGrab.enabled =
        true


    zeroGrab.lastScan =
        0


    scanGrabContainers()


    pcall(function()

        RunService:
        UnbindFromRenderStep(
            "THH_ZERO_GRAB"
        )

    end)


    RunService:
    BindToRenderStep(

        "THH_ZERO_GRAB",

        Enum.RenderPriority.Camera.Value
        +
        50,

        enforceZeroGrab

    )

end


local function disableZeroGrab()

    zeroGrab.enabled =
        false


    pcall(function()

        RunService:
        UnbindFromRenderStep(
            "THH_ZERO_GRAB"
        )

    end)


    for object, oldValue in pairs(
        zeroGrab.savedValues
    ) do

        if object
            and
            object.Parent
        then

            pcall(function()

                object.Value =
                    oldValue

            end)

        end

    end


    for attachment, oldPosition in pairs(
        zeroGrab.savedAttachments
    ) do

        if attachment
            and
            attachment.Parent
        then

            pcall(function()

                attachment.Position =
                    oldPosition

            end)

        end

    end


    table.clear(
        zeroGrab.savedValues
    )


    table.clear(
        zeroGrab.savedAttachments
    )


    table.clear(
        zeroGrab.containers
    )

end


--// ============================================================
--// GRAB NO FALL
--//
--// Character remains controllable.
--// ============================================================

local function destroyGrabNoFall()

    if grabNoFall.connection then

        grabNoFall.connection:
        Disconnect()

        grabNoFall.connection =
            nil

    end


    if grabNoFall.floor then

        grabNoFall.floor:
        Destroy()

        grabNoFall.floor =
            nil

    end


    grabNoFall.safeGroundY =
        nil

end


local function getFootPositionY()

    if not humanoid
        or
        not rootPart
    then

        return nil

    end


    return

        rootPart.Position.Y

        -

        humanoid.HipHeight

        -

        rootPart.Size.Y
        /
        2

end


local function findGroundBelow()

    if not character
        or
        not rootPart
    then

        return nil

    end


    local params =
        RaycastParams.new()


    params.FilterType =
        Enum.RaycastFilterType.Exclude


    params.IgnoreWater =
        true


    local ignoreList = {

        character

    }


    if grabNoFall.floor then

        table.insert(
            ignoreList,
            grabNoFall.floor
        )

    end


    params.FilterDescendantsInstances =
        ignoreList


    local result =
        workspace:
        Raycast(

            rootPart.Position
            +
            Vector3.new(
                0,
                2,
                0
            ),

            Vector3.new(
                0,
                -12,
                0
            ),

            params

        )


    if result
        and
        result.Normal.Y > 0.3
    then

        return result.Position.Y

    end


    return nil

end


local function enableGrabNoFall()

    destroyGrabNoFall()


    grabNoFall.enabled =
        true


    if not character
        or
        not humanoid
        or
        not rootPart
    then

        return

    end


    -- NEVER freeze player.

    rootPart.Anchored =
        false


    humanoid.PlatformStand =
        false


    humanoid.Sit =
        false


    humanoid.AutoRotate =
        true


    grabNoFall.safeGroundY =

        findGroundBelow()

        or

        getFootPositionY()

        or

        rootPart.Position.Y
        -
        3


    local floor =
        Instance.new(
            "Part"
        )


    floor.Name =
        "THH_GrabNoFall"


    floor.Size =
        grabNoFall.floorSize


    floor.Anchored =
        true


    floor.CanCollide =
        true


    floor.CanTouch =
        false


    floor.CanQuery =
        false


    floor.Transparency =
        1


    floor.CastShadow =
        false


    floor.Parent =
        workspace


    grabNoFall.floor =
        floor


    grabNoFall.connection =
        RunService.Heartbeat:
        Connect(function()


            if not grabNoFall.enabled then
                return
            end


            if not humanoid
                or
                not rootPart
                or
                not rootPart.Parent
            then

                return

            end


            -- KEEP WALKING ENABLED.

            if rootPart.Anchored then

                rootPart.Anchored =
                    false

            end


            if humanoid.PlatformStand then

                humanoid.PlatformStand =
                    false

            end


            -- KEEP PLAYER OUT OF SIT.

            if humanoid.Sit then

                humanoid.Sit =
                    false


                humanoid:
                ChangeState(
                    Enum.HumanoidStateType.GettingUp
                )

            end


            humanoid.AutoRotate =
                true


            local realGround =
                findGroundBelow()


            if realGround then

                local distance =
                    rootPart.Position.Y
                    -
                    realGround


                if distance <= 8 then

                    grabNoFall.safeGroundY =
                        realGround

                end

            end


            if not grabNoFall.safeGroundY then

                grabNoFall.safeGroundY =

                    getFootPositionY()

                    or

                    rootPart.Position.Y
                    -
                    3

            end


            -- X/Z follows player.
            --
            -- Y stays on the saved ground level.
            --
            -- So jumping does NOT pull the floor up.

            if floor
                and
                floor.Parent
            then

                floor.CFrame =
                    CFrame.new(

                        rootPart.Position.X,

                        grabNoFall.safeGroundY
                        -
                        floor.Size.Y
                        /
                        2,

                        rootPart.Position.Z

                    )

            end

        end)

end


local function disableGrabNoFall()

    grabNoFall.enabled =
        false


    destroyGrabNoFall()

end


--// ============================================================
--// JUMP
--// ============================================================

track(
    UserInputService.JumpRequest:
    Connect(function()


        if not humanoid
            or
            humanoid.Health <= 0
        then

            return

        end


        if grabNoFall.enabled then

            humanoid.Sit =
                false


            humanoid.PlatformStand =
                false


            humanoid:
            ChangeState(
                Enum.HumanoidStateType.Jumping
            )

            return

        end


        if movement.infiniteJump
            and
            not safety.underground
        then

            humanoid:
            ChangeState(
                Enum.HumanoidStateType.Jumping
            )

        end

    end)
)


--// ============================================================
--// NOCLIP
--// ============================================================

local function restoreNoclip()

    for part, oldValue in pairs(
        movement.noclipParts
    ) do

        if part
            and
            part.Parent
        then

            pcall(function()

                part.CanCollide =
                    oldValue

            end)

        end

    end


    table.clear(
        movement.noclipParts
    )

end


track(
    RunService.Stepped:
    Connect(function()


        if not character then
            return
        end


        if coinFarm.enabled
            or
            safety.underground
        then

            return

        end


        if movement.noclip then

            for _, object in ipairs(
                character:
                GetDescendants()
            ) do

                if object:IsA(
                    "BasePart"
                ) then


                    if movement.noclipParts[
                        object
                    ] == nil
                    then

                        movement.noclipParts[
                            object
                        ] =
                            object.CanCollide

                    end


                    object.CanCollide =
                        false

                end

            end


        elseif next(
            movement.noclipParts
        ) then

            restoreNoclip()

        end

    end)
)


--// ============================================================
--// SPIN BOT
--// ============================================================

local function destroySpinController()

    if movement.spinVelocity then

        pcall(function()

            movement.spinVelocity:
            Destroy()

        end)

        movement.spinVelocity =
            nil

    end


    if movement.spinAttachment then

        pcall(function()

            movement.spinAttachment:
            Destroy()

        end)

        movement.spinAttachment =
            nil

    end


    if humanoid then

        humanoid.AutoRotate =
            movement.oldAutoRotate

    end

end


local function updateSpinController()

    if not movement.spin then

        destroySpinController()

        return

    end


    if not rootPart
        or
        not humanoid
    then

        return

    end


    if not movement.spinAttachment
        or
        movement.spinAttachment.Parent
        ~=
        rootPart
    then


        destroySpinController()


        movement.oldAutoRotate =
            humanoid.AutoRotate


        humanoid.AutoRotate =
            false


        local attachment =
            Instance.new(
                "Attachment"
            )


        attachment.Name =
            "THH_SpinAttachment"


        attachment.Parent =
            rootPart


        local angularVelocity =
            Instance.new(
                "AngularVelocity"
            )


        angularVelocity.Name =
            "THH_SpinVelocity"


        angularVelocity.Attachment0 =
            attachment


        angularVelocity.RelativeTo =
            Enum.ActuatorRelativeTo.World


        angularVelocity.MaxTorque =
            math.huge


        angularVelocity.Parent =
            rootPart


        movement.spinAttachment =
            attachment


        movement.spinVelocity =
            angularVelocity

    end


    humanoid.AutoRotate =
        false


    movement.spinVelocity.AngularVelocity =
        Vector3.new(

            0,

            math.rad(
                movement.spinSpeed
            ),

            0

        )

end


--// ============================================================
--// ANTI FLING
--// ============================================================

local function applyAntiFlingPart(
    part
)

    if not antiFling.enabled
        or
        not part
        or
        not part:IsA(
            "BasePart"
        )
    then

        return

    end


    if antiFling.originalCollision[
        part
    ] == nil
    then

        antiFling.originalCollision[
            part
        ] =
            part.CanCollide

    end


    antiFling.trackedParts[
        part
    ] =
        true


    part.CanCollide =
        false

end


local function applyAntiFlingCharacter(
    targetPlayer,
    targetCharacter
)

    if targetPlayer == player
        or
        not targetCharacter
    then

        return

    end


    for _, object in ipairs(
        targetCharacter:
        GetDescendants()
    ) do

        if object:IsA(
            "BasePart"
        ) then

            applyAntiFlingPart(
                object
            )

        end

    end


    local connection =
        targetCharacter.DescendantAdded:
        Connect(function(
            object
        )

            if antiFling.enabled
                and
                object:IsA(
                    "BasePart"
                )
            then

                applyAntiFlingPart(
                    object
                )

            end

        end)


    antiFling.characterConnections[
        targetPlayer
    ] =
        connection

end


local function setupAntiFlingPlayer(
    targetPlayer
)

    if targetPlayer == player then
        return
    end


    if targetPlayer.Character then

        applyAntiFlingCharacter(

            targetPlayer,

            targetPlayer.Character

        )

    end


    antiFling.playerConnections[
        targetPlayer
    ] =
        targetPlayer.CharacterAdded:
        Connect(function(
            targetCharacter
        )

            if antiFling.enabled then

                applyAntiFlingCharacter(

                    targetPlayer,

                    targetCharacter

                )

            end

        end)

end


local function enableAntiFling()

    antiFling.enabled =
        true


    for _, targetPlayer in ipairs(
        Players:GetPlayers()
    ) do

        setupAntiFlingPlayer(
            targetPlayer
        )

    end


    antiFling.enforceThread =
        task.spawn(function()

            while alive
                and
                antiFling.enabled
            do

                for part in pairs(
                    antiFling.trackedParts
                ) do

                    if part
                        and
                        part.Parent
                    then

                        part.CanCollide =
                            false

                    else

                        antiFling.trackedParts[
                            part
                        ] =
                            nil

                    end

                end


                task.wait(
                    0.1
                )

            end

        end)

end


local function disableAntiFling()

    antiFling.enabled =
        false


    for part, oldValue in pairs(
        antiFling.originalCollision
    ) do

        if part
            and
            part.Parent
        then

            pcall(function()

                part.CanCollide =
                    oldValue

            end)

        end

    end


    table.clear(
        antiFling.originalCollision
    )


    table.clear(
        antiFling.trackedParts
    )


    for _, connection in pairs(
        antiFling.characterConnections
    ) do

        pcall(function()

            connection:
            Disconnect()

        end)

    end


    table.clear(
        antiFling.characterConnections
    )


    for _, connection in pairs(
        antiFling.playerConnections
    ) do

        pcall(function()

            connection:
            Disconnect()

        end)

    end


    table.clear(
        antiFling.playerConnections
    )

end


--// ============================================================
--// UNDERGROUND SAFE MODE
--// ============================================================

local function applySafetyCollision()

    if not character then
        return
    end


    for _, object in ipairs(
        character:
        GetDescendants()
    ) do

        if object:IsA(
            "BasePart"
        ) then


            if safety.originalCollision[
                object
            ] == nil
            then

                safety.originalCollision[
                    object
                ] =
                    object.CanCollide

            end


            object.CanCollide =
                false

        end

    end


    if safety.descendantConnection then

        safety.descendantConnection:
        Disconnect()

    end


    safety.descendantConnection =
        character.DescendantAdded:
        Connect(function(
            object
        )

            if safety.underground
                and
                object:IsA(
                    "BasePart"
                )
            then

                if safety.originalCollision[
                    object
                ] == nil
                then

                    safety.originalCollision[
                        object
                    ] =
                        object.CanCollide

                end


                object.CanCollide =
                    false

            end

        end)

end


local function restoreSafetyCollision()

    if safety.descendantConnection then

        safety.descendantConnection:
        Disconnect()

        safety.descendantConnection =
            nil

    end


    for part, oldValue in pairs(
        safety.originalCollision
    ) do

        if part
            and
            part.Parent
        then

            pcall(function()

                if movement.noclip
                    or
                    coinFarm.enabled
                then

                    part.CanCollide =
                        false

                else

                    part.CanCollide =
                        oldValue

                end

            end)

        end

    end


    table.clear(
        safety.originalCollision
    )

end


local function enableUnderground()

    if safety.underground
        or
        not rootPart
        or
        not humanoid
    then

        return

    end


    safety.underground =
        true


    humanoid.CameraOffset =
        Vector3.new(

            0,

            safety.depth,

            0

        )


    applySafetyCollision()


    rootPart.CFrame =
        rootPart.CFrame

        -

        Vector3.new(

            0,

            safety.depth,

            0

        )


    rootPart.AssemblyLinearVelocity =
        Vector3.zero

end


local function disableUnderground(
    returnToCamera
)

    if not safety.underground then
        return
    end


    local camera =
        workspace.CurrentCamera


    local cameraPosition =

        camera

        and

        camera.CFrame.Position

        or

        nil


    safety.underground =
        false


    if humanoid then

        humanoid.CameraOffset =
            Vector3.zero

    end


    restoreSafetyCollision()


    if returnToCamera
        and
        rootPart
        and
        cameraPosition
    then

        rootPart.CFrame =
            CFrame.new(

                cameraPosition
                +
                Vector3.new(
                    0,
                    2,
                    0
                )

            )


        rootPart.AssemblyLinearVelocity =
            Vector3.zero

    end

end


--// ============================================================
--// COIN FARM
--// ============================================================

local function isMainCoin(
    object
)

    return

        object

        and

        object:IsA(
            "BasePart"
        )

        and

        object.Name
        ==
        "MainCoin"

end


local function coinVisible(
    coin
)

    return

        coin

        and

        coin.Parent

        and

        coin.Transparency
        ==
        0

        and

        coin.LocalTransparencyModifier
        ==
        0

end


local function addCoin(
    coin
)

    if not isMainCoin(
        coin
    )
        or
        coinFarm.coinLookup[
            coin
        ]
    then

        return

    end


    coinFarm.coinLookup[
        coin
    ] =
        true


    table.insert(
        coinFarm.coins,
        coin
    )

end


local function removeCoin(
    coin
)

    coinFarm.coinLookup[
        coin
    ] =
        nil


    for index =
        #coinFarm.coins,
        1,
        -1
    do

        if coinFarm.coins[
            index
        ]
        ==
        coin
        then

            table.remove(
                coinFarm.coins,
                index
            )

            break

        end

    end

end


for _, object in ipairs(
    workspace:
    GetDescendants()
) do

    if isMainCoin(
        object
    ) then

        addCoin(
            object
        )

    end

end


track(
    workspace.DescendantAdded:
    Connect(function(
        object
    )

        if isMainCoin(
            object
        ) then

            addCoin(
                object
            )

        end

    end)
)


track(
    workspace.DescendantRemoving:
    Connect(function(
        object
    )

        if coinFarm.coinLookup[
            object
        ] then

            removeCoin(
                object
            )

        end

    end)
)


local function ensureCoinPlatform()

    if coinFarm.platform
        and
        coinFarm.platform.Parent
    then

        return coinFarm.platform

    end


    local platform =
        Instance.new(
            "Part"
        )


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


local function destroyCoinPlatform()

    if coinFarm.platform then

        coinFarm.platform:
        Destroy()

        coinFarm.platform =
            nil

    end

end


local function getNearestCoin()

    if not rootPart then
        return nil
    end


    local bestCoin =
        nil


    local bestDistance =
        math.huge


    for _, coin in ipairs(
        coinFarm.coins
    ) do

        if coinVisible(
            coin
        ) then

            local target =
                coin.Position

                -

                Vector3.new(
                    0,
                    coinFarm.underDistance,
                    0
                )


            local distance =
                (
                    target
                    -
                    rootPart.Position
                ).Magnitude


            if distance
                <
                bestDistance
            then

                bestDistance =
                    distance

                bestCoin =
                    coin

            end

        end

    end


    return bestCoin

end


local function setFarmPosition(
    position
)

    if not rootPart then
        return
    end


    rootPart.CFrame =
        CFrame.new(
            position
        )


    rootPart.AssemblyLinearVelocity =
        Vector3.zero


    local platform =
        ensureCoinPlatform()


    platform.CFrame =
        CFrame.new(

            position

            -

            Vector3.new(
                0,
                3.15,
                0
            )

        )

end


local function smoothMove(
    destination,
    speed,
    stopDistance
)

    stopDistance =
        stopDistance
        or
        0.8


    while alive
        and
        coinFarm.enabled
        and
        rootPart
    do

        local offset =
            destination
            -
            rootPart.Position


        local distance =
            offset.Magnitude


        if distance
            <=
            stopDistance
        then

            setFarmPosition(
                destination
            )

            return true

        end


        local dt =
            RunService.Heartbeat:
            Wait()


        local step =
            math.min(

                distance,

                speed
                *
                dt

            )


        if distance > 0 then

            setFarmPosition(

                rootPart.Position

                +

                offset.Unit
                *
                step

            )

        end

    end


    return false

end


task.spawn(function()

    while alive do

        if coinFarm.enabled
            and
            not coinFarm.busy
            and
            rootPart
        then


            local coin =
                getNearestCoin()


            coinFarm.target =
                coin


            if coin then

                coinFarm.busy =
                    true


                local under =
                    coin.Position

                    -

                    Vector3.new(
                        0,
                        coinFarm.underDistance,
                        0
                    )


                smoothMove(

                    under,

                    coinFarm.travelSpeed,

                    0.75

                )


                if coinVisible(
                    coin
                ) then

                    smoothMove(

                        coin.Position,

                        coinFarm.verticalSpeed,

                        1.4

                    )


                    if firetouchinterest
                        and
                        rootPart
                    then

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


                    task.wait(
                        0.05
                    )


                    if not coinVisible(
                        coin
                    ) then

                        coinFarm.pickedUp +=
                            1

                    end


                    smoothMove(

                        under,

                        coinFarm.verticalSpeed,

                        0.8

                    )

                end


                coinFarm.busy =
                    false

            end

        end


        task.wait(
            0.02
        )

    end

end)


--// ============================================================
--// HEARTBEAT
--// ============================================================

track(
    RunService.Heartbeat:
    Connect(function()


        if not humanoid
            or
            not rootPart
        then

            return

        end


        if movement.speedEnabled
            and
            not coinFarm.enabled
            and
            not safety.underground
        then

            humanoid.WalkSpeed =
                movement.speed

        elseif not coinFarm.enabled
            and
            not safety.underground
        then

            humanoid.WalkSpeed =
                originalWalkSpeed

        end


        if safety.underground
            and
            not coinFarm.enabled
        then

            local moveDirection =
                humanoid.MoveDirection


            local speed =

                movement.speedEnabled

                and

                movement.speed

                or

                originalWalkSpeed


            rootPart.AssemblyLinearVelocity =
                Vector3.new(

                    moveDirection.X
                    *
                    speed,

                    0,

                    moveDirection.Z
                    *
                    speed

                )

        end


        updateSpinController()


        if murderer.autoThrow
            and
            not murderer.throwing
            and
            os.clock()
            -
            murderer.lastThrow
            >=
            murderer.throwDelay
        then

            murderer.lastThrow =
                os.clock()


            task.spawn(
                throwKnifeOnce
            )

        end

    end)
)


--// ============================================================
--// CHARACTER RESPAWN
--// ============================================================

track(
    player.CharacterAdded:
    Connect(function()


        destroySpinController()


        task.wait(
            0.25
        )


        refreshCharacter()


        sheriff.shooting =
            false


        murderer.throwing =
            false


        gunPickup.busy =
            false


        coinFarm.busy =
            false


        if nvis.enabled then

            enableNvis()

        end


        if grabNoFall.enabled then

            enableGrabNoFall()

        end


        if movement.spin then

            updateSpinController()

        end


        if safety.underground then

            enableUnderground()

        end

    end)
)


--// ============================================================
--// AUTH
--// ============================================================

local function loadVampauth()

    local success,
    result =

        pcall(function()


            if not loadstring then

                error(
                    "loadstring unavailable"
                )

            end


            local source =
                game:
                HttpGet(
                    "https://vampauth.com/client/vampauth.lua"
                )


            local loader =
                loadstring(
                    source
                )


            if not loader then

                error(
                    "Vampauth loader failed"
                )

            end


            return loader()

        end)


    if success
        and
        type(
            result
        ) == "table"
    then

        return result

    end


    return nil

end


local vampauthClient


local function getHWID()

    local success,
    result =

        pcall(function()

            return RbxAnalyticsService:
                GetClientId()

        end)


    if success then

        return result

    end


    return tostring(
        player.UserId
    )

end


local function validateKey(
    key
)

    key =
        tostring(
            key or ""
        )
        :
        gsub(
            "^%s+",
            ""
        )
        :
        gsub(
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


        local success,
        result =

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
            or
            not result
        then

            return false,
            "authentication failed"

        end


        vampauthClient =
            result

    end


    local success,
    valid,
    data =

        pcall(function()

            return vampauthClient:
                Check(
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


    return false,
    tostring(
        data
        or
        "invalid key"
    )

end


--// ============================================================
--// GUI
--// ============================================================

gui =
    create(
        "ScreenGui",
        {
            Name =
                "THH_HUB",

            ResetOnSpawn =
                false,

            IgnoreGuiInset =
                true,

            DisplayOrder =
                999999
        },
        getUIParent()
    )


blur =
    create(
        "BlurEffect",
        {
            Name =
                "THH_HUB_Blur",

            Size =
                18
        },
        Lighting
    )


--// ============================================================
--// DRAG
--// ============================================================

local function makeDraggable(
    frame,
    handle
)

    handle =
        handle
        or
        frame


    local dragging =
        false


    local dragInput

    local dragStart

    local frameStart


    track(
        handle.InputBegan:
        Connect(function(
            input
        )

            if input.UserInputType
                    ==
                    Enum.UserInputType.MouseButton1

                or

                input.UserInputType
                    ==
                    Enum.UserInputType.Touch
            then

                dragging =
                    true

                dragStart =
                    input.Position

                frameStart =
                    frame.Position

            end

        end)
    )


    track(
        handle.InputChanged:
        Connect(function(
            input
        )

            if input.UserInputType
                    ==
                    Enum.UserInputType.MouseMovement

                or

                input.UserInputType
                    ==
                    Enum.UserInputType.Touch
            then

                dragInput =
                    input

            end

        end)
    )


    track(
        UserInputService.InputChanged:
        Connect(function(
            input
        )

            if not dragging
                or
                input ~= dragInput
            then

                return

            end


            local delta =
                input.Position
                -
                dragStart


            frame.Position =
                UDim2.new(

                    frameStart.X.Scale,

                    frameStart.X.Offset
                    +
                    delta.X,

                    frameStart.Y.Scale,

                    frameStart.Y.Offset
                    +
                    delta.Y

                )

        end)
    )


    track(
        UserInputService.InputEnded:
        Connect(function(
            input
        )

            if input.UserInputType
                    ==
                    Enum.UserInputType.MouseButton1

                or

                input.UserInputType
                    ==
                    Enum.UserInputType.Touch
            then

                dragging =
                    false

            end

        end)
    )

end


--// ============================================================
--// MAIN MENU CREATOR
--// ============================================================

local buildMainMenu


local function buildChooser()


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
                        7,
                        8,
                        10
                    ),

                BackgroundTransparency =
                    0.1,

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
                    UDim2.fromOffset(
                        410,
                        215
                    ),

                BackgroundColor3 =
                    Colors.Card,

                BorderSizePixel =
                    0,

                Active =
                    true
            },
            overlay
        )


    addCorner(
        panel,
        14
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
                    15
                ),

            Size =
                UDim2.new(
                    1,
                    -40,
                    0,
                    30
                ),

            BackgroundTransparency =
                1,

            Text =
                "What are you on?",

            TextColor3 =
                Colors.Text,

            TextSize =
                19,

            Font =
                Enum.Font.GothamBold
        },
        panel
    )


    local pc =
        create(
            "TextButton",
            {
                Position =
                    UDim2.fromOffset(
                        20,
                        70
                    ),

                Size =
                    UDim2.fromOffset(
                        175,
                        110
                    ),

                BackgroundColor3 =
                    Colors.Control,

                BorderSizePixel =
                    0,

                Text =
                    "PC",

                TextColor3 =
                    Colors.Text,

                TextSize =
                    20,

                Font =
                    Enum.Font.GothamBold
            },
            panel
        )


    addCorner(
        pc,
        10
    )


    local phone =
        create(
            "TextButton",
            {
                Position =
                    UDim2.fromOffset(
                        215,
                        70
                    ),

                Size =
                    UDim2.fromOffset(
                        175,
                        110
                    ),

                BackgroundColor3 =
                    Colors.Control,

                BorderSizePixel =
                    0,

                Text =
                    "PHONE",

                TextColor3 =
                    Colors.Text,

                TextSize =
                    20,

                Font =
                    Enum.Font.GothamBold
            },
            panel
        )


    addCorner(
        phone,
        10
    )


    track(
        pc.MouseButton1Click:
        Connect(function()

            overlay:
            Destroy()

            if blur then

                blur:
                Destroy()

                blur =
                    nil

            end


            buildMainMenu(
                false
            )

        end)
    )


    track(
        phone.MouseButton1Click:
        Connect(function()

            overlay:
            Destroy()

            if blur then

                blur:
                Destroy()

                blur =
                    nil

            end


            buildMainMenu(
                true
            )

        end)
    )

end


--// ============================================================
--// MAIN MENU
--// ============================================================

buildMainMenu =
function(
    isPhone
)


    local menuWidth =
        isPhone
        and
        340
        or
        700


    local menuHeight =
        isPhone
        and
        330
        or
        455


    local sidebarWidth =
        isPhone
        and
        85
        or
        150


    local headerHeight =
        isPhone
        and
        42
        or
        55


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
                    UDim2.fromOffset(
                        menuWidth,
                        menuHeight
                    ),

                BackgroundColor3 =
                    Colors.Background,

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
        13
    )


    addStroke(
        menu
    )


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
                    Colors.Sidebar,

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


    create(
        "TextLabel",
        {
            Position =
                UDim2.fromOffset(
                    12,
                    0
                ),

            Size =
                UDim2.new(
                    1,
                    -90,
                    1,
                    0
                ),

            BackgroundTransparency =
                1,

            Text =
                "THH HUB",

            TextColor3 =
                Colors.Text,

            TextSize =
                isPhone
                and
                14
                or
                18,

            Font =
                Enum.Font.GothamBold,

            TextXAlignment =
                Enum.TextXAlignment.Left
        },
        header
    )


    local hide =
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

                BorderSizePixel =
                    0,

                Text =
                    "—",

                TextColor3 =
                    Colors.Text,

                Font =
                    Enum.Font.GothamBold
            },
            header
        )


    addCorner(
        hide,
        7
    )


    local sidebar =
        create(
            "ScrollingFrame",
            {
                Position =
                    UDim2.fromOffset(
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

                BorderSizePixel =
                    0,

                CanvasSize =
                    UDim2.fromOffset(
                        0,
                        0
                    ),

                AutomaticCanvasSize =
                    Enum.AutomaticSize.Y,

                ScrollBarThickness =
                    2
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
                )
        },
        sidebar
    )


    create(
        "UIPadding",
        {
            PaddingTop =
                UDim.new(
                    0,
                    7
                ),

            PaddingLeft =
                UDim.new(
                    0,
                    5
                ),

            PaddingRight =
                UDim.new(
                    0,
                    5
                )
        },
        sidebar
    )


    local content =
        create(
            "Frame",
            {
                Position =
                    UDim2.fromOffset(
                        sidebarWidth,
                        headerHeight
                    ),

                Size =
                    UDim2.new(
                        1,
                        -sidebarWidth,
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


    local pages = {}

    local navButtons = {}


    local function makePage(
        name
    )

        local page =
            create(
                "ScrollingFrame",
                {
                    Name =
                        name,

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
                        3
                },
                content
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
            page
        )


        create(
            "UIPadding",
            {
                PaddingTop =
                    UDim.new(
                        0,
                        8
                    ),

                PaddingLeft =
                    UDim.new(
                        0,
                        8
                    ),

                PaddingRight =
                    UDim.new(
                        0,
                        8
                    ),

                PaddingBottom =
                    UDim.new(
                        0,
                        8
                    )
            },
            page
        )


        pages[name] =
            page


        return page

    end


    local function showPage(
        name
    )

        for pageName, page in pairs(
            pages
        ) do

            page.Visible =
                pageName
                ==
                name

        end


        for buttonName, button in pairs(
            navButtons
        ) do

            button.BackgroundColor3 =

                buttonName
                ==
                name

                and

                Colors.AccentDark

                or

                Colors.Control

        end

    end


    local function makeNav(
        name
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
                            isPhone
                            and
                            28
                            or
                            36
                        ),

                    BackgroundColor3 =
                        Colors.Control,

                    BorderSizePixel =
                        0,

                    Text =
                        name,

                    TextColor3 =
                        Colors.Text,

                    TextSize =
                        isPhone
                        and
                        8
                        or
                        11,

                    Font =
                        Enum.Font.GothamMedium
                },
                sidebar
            )


        addCorner(
            button,
            7
        )


        navButtons[name] =
            button


        track(
            button.MouseButton1Click:
            Connect(function()

                showPage(
                    name
                )

            end)
        )

    end


    local function section(
        page,
        name
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

                    BackgroundColor3 =
                        Colors.Card,

                    BorderSizePixel =
                        0
                },
                page
            )


        addCorner(
            holder,
            9
        )


        create(
            "UIListLayout",
            {
                Padding =
                    UDim.new(
                        0,
                        5
                    )
            },
            holder
        )


        create(
            "UIPadding",
            {
                PaddingTop =
                    UDim.new(
                        0,
                        7
                    ),

                PaddingBottom =
                    UDim.new(
                        0,
                        7
                    ),

                PaddingLeft =
                    UDim.new(
                        0,
                        7
                    ),

                PaddingRight =
                    UDim.new(
                        0,
                        7
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
                        20
                    ),

                BackgroundTransparency =
                    1,

                Text =
                    name,

                TextColor3 =
                    Colors.Text,

                TextSize =
                    isPhone
                    and
                    10
                    or
                    13,

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
        text,
        callback
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
                            isPhone
                            and
                            29
                            or
                            38
                        ),

                    BackgroundColor3 =
                        Colors.Control,

                    BorderSizePixel =
                        0,

                    Text =
                        text,

                    TextColor3 =
                        Colors.Text,

                    TextSize =
                        isPhone
                        and
                        8
                        or
                        11,

                    Font =
                        Enum.Font.GothamMedium
                },
                parent
            )


        addCorner(
            button,
            7
        )


        track(
            button.MouseButton1Click:
            Connect(function()

                callback()

            end)
        )


        return button

    end


    local function toggle(
        parent,
        text,
        default,
        callback
    )

        local state =
            default


        local button =
            action(
                parent,
                text,
                function()

                    state =
                        not state


                    button.BackgroundColor3 =

                        state

                        and

                        Colors.AccentDark

                        or

                        Colors.Control


                    callback(
                        state
                    )

                end
            )


        button.BackgroundColor3 =

            state

            and

            Colors.AccentDark

            or

            Colors.Control


        return button

    end


    local Home =
        makePage(
            "Home"
        )

    local SheriffPage =
        makePage(
            "Sheriff"
        )

    local MurdererPage =
        makePage(
            "Murderer"
        )

    local InnocentPage =
        makePage(
            "Innocent"
        )

    local CoinsPage =
        makePage(
            "Coins"
        )

    local SafetyPage =
        makePage(
            "Safety"
        )

    local UtilityPage =
        makePage(
            "Utility"
        )


    makeNav(
        "Home"
    )

    makeNav(
        "Sheriff"
    )

    makeNav(
        "Murderer"
    )

    makeNav(
        "Innocent"
    )

    makeNav(
        "Coins"
    )

    makeNav(
        "Safety"
    )

    makeNav(
        "Utility"
    )


    local home =
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
                "Welcome "
                ..
                player.DisplayName,

            TextColor3 =
                Colors.Accent,

            TextSize =
                isPhone
                and
                9
                or
                12,

            Font =
                Enum.Font.GothamMedium
        },
        home
    )


    --// Sheriff

    local sheriffSection =
        section(
            SheriffPage,
            "Sheriff"
        )


    toggle(
        sheriffSection,
        "Aim Bot",
        sheriff.quickShot,
        function(
            value
        )

            sheriff.quickShot =
                value

        end
    )


    action(
        sheriffSection,
        "Shoot Murderer",
        function()

            quickShoot()

        end
    )


    action(
        sheriffSection,
        "Pick Up Gun",
        function()

            pickupGun()

        end
    )


    toggle(
        sheriffSection,
        "Auto Pick Up Gun",
        gunPickup.auto,
        function(
            value
        )

            gunPickup.auto =
                value

        end
    )


    --// Murderer

    local murdererSection =
        section(
            MurdererPage,
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
        function(
            value
        )

            murderer.autoThrow =
                value

        end
    )


    --// Innocent

    local movementSection =
        section(
            InnocentPage,
            "Movement"
        )


    toggle(
        movementSection,
        "Speed",
        movement.speedEnabled,
        function(
            value
        )

            movement.speedEnabled =
                value

        end
    )


    toggle(
        movementSection,
        "Infinite Jump",
        movement.infiniteJump,
        function(
            value
        )

            movement.infiniteJump =
                value

        end
    )


    toggle(
        movementSection,
        "Noclip",
        movement.noclip,
        function(
            value
        )

            movement.noclip =
                value

            if not value then

                restoreNoclip()

            end

        end
    )


    toggle(
        movementSection,
        "Nvis",
        nvis.enabled,
        function(
            value
        )

            if value then

                enableNvis()

            else

                disableNvis()

            end

        end
    )


    toggle(
        movementSection,
        "Spin Bot",
        movement.spin,
        function(
            value
        )

            movement.spin =
                value

            if not value then

                destroySpinController()

            end

        end
    )


    local visualSection =
        section(
            InnocentPage,
            "Visual"
        )


    toggle(
        visualSection,
        "ESP",
        esp.enabled,
        function(
            value
        )

            if value then

                enableESP()

            else

                disableESP()

            end

        end
    )


    --// Coins

    local coinSection =
        section(
            CoinsPage,
            "Coin Grab"
        )


    toggle(
        coinSection,
        "Coin Farm",
        coinFarm.enabled,
        function(
            value
        )

            coinFarm.enabled =
                value

            if value then

                ensureCoinPlatform()

            else

                coinFarm.target =
                    nil

                coinFarm.busy =
                    false

                destroyCoinPlatform()

            end

        end
    )


    --// Safety

    local safetySection =
        section(
            SafetyPage,
            "Safety"
        )


    toggle(
        safetySection,
        "Anti Fling",
        antiFling.enabled,
        function(
            value
        )

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
        function(
            value
        )

            if value then

                enableUnderground()

            else

                disableUnderground(
                    true
                )

            end

        end
    )


    toggle(
        safetySection,
        "0 Grab",
        zeroGrab.enabled,
        function(
            value
        )

            if value then

                enableZeroGrab()

            else

                disableZeroGrab()

            end

        end
    )


    toggle(
        safetySection,
        "Grab No Fall",
        grabNoFall.enabled,
        function(
            value
        )

            if value then

                enableGrabNoFall()

            else

                disableGrabNoFall()

            end

        end
    )


    --// Utility

    local utility =
        section(
            UtilityPage,
            "Utility"
        )


    action(
        utility,
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


    action(
        utility,
        "Reset Character",
        function()

            if humanoid then

                humanoid.Health =
                    0

            end

        end
    )


    action(
        utility,
        "Hide Menu",
        function()

            menu.Visible =
                false

        end
    )


    track(
        hide.MouseButton1Click:
        Connect(function()

            menu.Visible =
                false

        end)
    )


    -- PC RightShift.

    track(
        UserInputService.InputBegan:
        Connect(function(
            input,
            processed
        )

            if processed
                or
                isPhone
            then

                return

            end


            if input.KeyCode
                ==
                Enum.KeyCode.RightShift
            then

                menu.Visible =
                    not menu.Visible

            end

        end)
    )


    showPage(
        "Home"
    )

end


--// ============================================================
--// KEY WINDOW
--// ============================================================

local keyOverlay =
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

            BorderSizePixel =
                0
        },
        gui
    )


local keyWindow =
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
                UDim2.fromOffset(
                    390,
                    275
                ),

            BackgroundColor3 =
                Colors.Card,

            BorderSizePixel =
                0,

            Active =
                true
        },
        keyOverlay
    )


addCorner(
    keyWindow,
    15
)


makeDraggable(
    keyWindow
)


create(
    "TextLabel",
    {
        Position =
            UDim2.fromOffset(
                20,
                15
            ),

        Size =
            UDim2.new(
                1,
                -40,
                0,
                30
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
            Enum.Font.GothamBold
    },
    keyWindow
)


local keyBox =
    create(
        "TextBox",
        {
            Position =
                UDim2.fromOffset(
                    20,
                    65
                ),

            Size =
                UDim2.new(
                    1,
                    -40,
                    0,
                    45
                ),

            BackgroundColor3 =
                Colors.Control,

            BorderSizePixel =
                0,

            Text =
                "",

            PlaceholderText =
                "Enter access key",

            TextColor3 =
                Colors.Text,

            PlaceholderColor3 =
                Colors.SubText,

            ClearTextOnFocus =
                false
        },
        keyWindow
    )


addCorner(
    keyBox,
    9
)


local continue =
    create(
        "TextButton",
        {
            Position =
                UDim2.fromOffset(
                    20,
                    125
                ),

            Size =
                UDim2.new(
                    1,
                    -40,
                    0,
                    42
                ),

            BackgroundColor3 =
                Colors.Accent,

            BorderSizePixel =
                0,

            Text =
                "Continue",

            TextColor3 =
                Color3.fromRGB(
                    10,
                    20,
                    14
                ),

            Font =
                Enum.Font.GothamBold
        },
        keyWindow
    )


addCorner(
    continue,
    9
)


local getKey =
    create(
        "TextButton",
        {
            Position =
                UDim2.fromOffset(
                    20,
                    180
                ),

            Size =
                UDim2.new(
                    1,
                    -40,
                    0,
                    42
                ),

            BackgroundColor3 =
                Colors.Control,

            BorderSizePixel =
                0,

            Text =
                "Get Key",

            TextColor3 =
                Colors.Text,

            Font =
                Enum.Font.GothamBold
        },
        keyWindow
    )


addCorner(
    getKey,
    9
)


local status =
    create(
        "TextLabel",
        {
            Position =
                UDim2.fromOffset(
                    20,
                    230
                ),

            Size =
                UDim2.new(
                    1,
                    -40,
                    0,
                    25
                ),

            BackgroundTransparency =
                1,

            Text =
                "Paste your key",

            TextColor3 =
                Colors.SubText,

            TextSize =
                11,

            Font =
                Enum.Font.Gotham
        },
        keyWindow
    )


track(
    getKey.MouseButton1Click:
    Connect(function()


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


        status.Text =

            copied

            and

            "Key link copied"

            or

            KEY_FLOW

    end)
)


local checking =
    false


track(
    continue.MouseButton1Click:
    Connect(function()


        if checking then
            return
        end


        checking =
            true


        continue.Text =
            "Checking..."


        task.spawn(function()


            local valid,
            result =

                validateKey(
                    keyBox.Text
                )


            if valid then


                status.Text =
                    "Key accepted"


                status.TextColor3 =
                    Colors.Accent


                task.wait(
                    0.15
                )


                keyOverlay:
                Destroy()


                buildChooser()


            else


                status.Text =
                    tostring(
                        result
                    )


                status.TextColor3 =
                    Colors.Danger


                continue.Text =
                    "Continue"


                checking =
                    false

            end

        end)

    end)
)


--// ============================================================
--// CLEANUP
--// ============================================================

local function cleanup()

    if not alive then
        return
    end


    alive =
        false


    sheriff.quickShot =
        false


    murderer.autoThrow =
        false


    gunPickup.auto =
        false


    movement.speedEnabled =
        false


    movement.infiniteJump =
        false


    movement.noclip =
        false


    movement.spin =
        false


    coinFarm.enabled =
        false


    disableNvis()

    disableZeroGrab()

    disableGrabNoFall()

    disableAntiFling()

    disableUnderground(
        false
    )


    restoreNoclip()

    destroySpinController()

    destroyCoinPlatform()

    disableESP()


    if humanoid then

        pcall(function()

            humanoid.WalkSpeed =
                originalWalkSpeed


            humanoid.Sit =
                false


            humanoid.PlatformStand =
                false


            humanoid.CameraOffset =
                Vector3.zero

        end)

    end


    for _, connection in ipairs(
        connections
    ) do

        pcall(function()

            connection:
            Disconnect()

        end)

    end


    table.clear(
        connections
    )


    if blur then

        pcall(function()

            blur:
            Destroy()

        end)

    end


    if gui then

        pcall(function()

            gui:
            Destroy()

        end)

    end


    _G.THHGrowersCleanup =
        nil

end


_G.THHGrowersCleanup =
    cleanup
