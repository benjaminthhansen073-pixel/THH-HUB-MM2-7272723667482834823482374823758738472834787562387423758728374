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

local VirtualInputManager

pcall(function()
    VirtualInputManager = game:GetService("VirtualInputManager")
end)

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
    Background = Color3.fromRGB(31, 32, 36),
    Sidebar = Color3.fromRGB(37, 38, 43),
    Card = Color3.fromRGB(62, 64, 70),
    Control = Color3.fromRGB(48, 50, 55),

    Text = Color3.fromRGB(244, 245, 247),
    SubText = Color3.fromRGB(164, 166, 174),

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

local antiFling = {
    enabled = false,
    originalCollision = {}
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
    throwDelay = 0.25,
    lastThrow = 0
}

local gunPickup = {
    auto = false,
    busy = false
}

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

local throwKnifeOnce

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
        Transparency = transparency or 0.45
    }, object)
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
    character =
        player.Character
        or player.CharacterAdded:Wait()

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

local function restoreOtherPlayerCollision()
    for part, oldValue in pairs(antiFling.originalCollision) do
        if part and part.Parent then
            pcall(function()
                part.CanCollide = oldValue
            end)
        end
    end

    table.clear(antiFling.originalCollision)
end

local function disableOtherPlayerCollision()
    if not antiFling.enabled then
        return
    end

    for _, targetPlayer in ipairs(Players:GetPlayers()) do
        if targetPlayer ~= player
            and targetPlayer.Character then

            for _, object in ipairs(targetPlayer.Character:GetDescendants()) do
                if object:IsA("BasePart") then
                    if antiFling.originalCollision[object] == nil then
                        antiFling.originalCollision[object] =
                            object.CanCollide
                    end

                    object.CanCollide = false
                end
            end
        end
    end
end

local function stopCoinCharacterWatcher()
    if coinFarm.characterConnection then
        pcall(function()
            coinFarm.characterConnection:Disconnect()
        end)

        coinFarm.characterConnection = nil
    end
end

local function restoreCoinCollision()
    stopCoinCharacterWatcher()

    for part, oldValue in pairs(coinFarm.originalCollision) do
        if part and part.Parent then
            pcall(function()
                if movement.noclip then
                    part.CanCollide = false
                else
                    part.CanCollide = oldValue
                end
            end)
        end
    end

    table.clear(coinFarm.originalCollision)
end

local function disableCoinCharacterCollision()
    stopCoinCharacterWatcher()

    if not character then
        return
    end

    for _, object in ipairs(character:GetDescendants()) do
        if object:IsA("BasePart") then
            if coinFarm.originalCollision[object] == nil then
                coinFarm.originalCollision[object] =
                    object.CanCollide
            end

            object.CanCollide = false
        end
    end

    coinFarm.characterConnection =
        character.DescendantAdded:Connect(function(object)
            if not coinFarm.enabled then
                return
            end

            if object:IsA("BasePart") then
                if coinFarm.originalCollision[object] == nil then
                    coinFarm.originalCollision[object] =
                        object.CanCollide
                end

                object.CanCollide = false
            end
        end)
end

local function destroyCoinPlatform()
    if coinFarm.platform then
        pcall(function()
            coinFarm.platform:Destroy()
        end)

        coinFarm.platform = nil
    end
end

local function ensureCoinPlatform()
    if coinFarm.platform
        and coinFarm.platform.Parent then

        return coinFarm.platform
    end

    local platform = Instance.new("Part")

    platform.Name = "THH_CoinPlatform"

    platform.Size =
        Vector3.new(5, 0.45, 5)

    platform.Anchored = true

    -- platform itself stays collidable
    platform.CanCollide = true
    platform.CanTouch = false
    platform.CanQuery = false

    platform.Transparency = 1

    platform.Parent = workspace

    coinFarm.platform = platform

    return platform
end

local function removePlayerESP(targetPlayer)
    local highlight =
        esp.highlights[targetPlayer]

    if highlight then
        pcall(function()
            highlight:Destroy()
        end)

        esp.highlights[targetPlayer] = nil
    end
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

    for _, connection in pairs(esp.characterConnections) do
        pcall(function()
            connection:Disconnect()
        end)
    end

    table.clear(esp.characterConnections)
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

    antiFling.enabled = false

    esp.enabled = false

    sheriff.quickShot = false
    sheriff.shooting = false

    murderer.autoThrow = false
    murderer.throwing = false

    gunPickup.auto = false
    gunPickup.busy = false

    coinFarm.enabled = false
    coinFarm.busy = false

    restoreNoclip()
    restoreOtherPlayerCollision()
    restoreCoinCollision()

    destroyCoinPlatform()
    clearESP()

    if humanoid then
        pcall(function()
            humanoid.WalkSpeed =
                originalWalkSpeed
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

local function makeDraggable(frame, handle)
    handle = handle or frame

    local dragging = false
    local dragInput
    local dragStart
    local frameStart

    track(handle.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then

            dragging = true

            dragStart =
                input.Position

            frameStart =
                frame.Position
        end
    end))

    track(handle.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement
            or input.UserInputType == Enum.UserInputType.Touch then

            dragInput = input
        end
    end))

    track(UserInputService.InputChanged:Connect(function(input)
        if not dragging
            or input ~= dragInput then

            return
        end

        local delta =
            input.Position
            - dragStart

        frame.Position =
            UDim2.new(
                frameStart.X.Scale,
                frameStart.X.Offset + delta.X,
                frameStart.Y.Scale,
                frameStart.Y.Offset + delta.Y
            )
    end))

    track(UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then

            dragging = false
        end
    end))
end

local function animateButton(button)
    local normal =
        button.BackgroundColor3

    local hover =
        Color3.new(
            math.min(normal.R + 0.06, 1),
            math.min(normal.G + 0.06, 1),
            math.min(normal.B + 0.06, 1)
        )

    if UserInputService.MouseEnabled then
        track(button.MouseEnter:Connect(function()
            tween(
                button,
                0.14,
                {
                    BackgroundColor3 =
                        hover
                }
            )
        end))

        track(button.MouseLeave:Connect(function()
            tween(
                button,
                0.14,
                {
                    BackgroundColor3 =
                        normal
                }
            )
        end))
    end

    track(button.MouseButton1Down:Connect(function()
        tween(
            button,
            0.07,
            {
                BackgroundTransparency =
                    0.2
            }
        )
    end))

    track(button.MouseButton1Up:Connect(function()
        tween(
            button,
            0.1,
            {
                BackgroundTransparency =
                    0.08
            }
        )
    end))
end

local function findTool(targetPlayer, wantedName)
    if not targetPlayer then
        return nil
    end

    wantedName =
        string.lower(wantedName)

    local targetCharacter =
        targetPlayer.Character

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
    return findTool(
        targetPlayer,
        "Knife"
    ) ~= nil
end

local function hasGun(targetPlayer)
    if findTool(targetPlayer, "Gun") then
        return true
    end

    local containers = {}

    if targetPlayer.Character then
        table.insert(
            containers,
            targetPlayer.Character
        )
    end

    local backpack =
        targetPlayer:FindFirstChildOfClass("Backpack")

    if backpack then
        table.insert(
            containers,
            backpack
        )
    end

    for _, container in ipairs(containers) do
        for _, object in ipairs(container:GetChildren()) do
            if object:IsA("Tool") then
                local name =
                    string.lower(object.Name)

                if name:find("gun", 1, true)
                    or name:find("revolver", 1, true) then

                    return true
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

local function addPlayerESP(targetPlayer)
    if not esp.enabled
        or targetPlayer == player
        or not targetPlayer.Character then

        return
    end

    removePlayerESP(targetPlayer)

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
        getRoleColor(targetPlayer)

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
    if not esp.enabled
        or targetPlayer == player then

        return
    end

    addPlayerESP(targetPlayer)

    if esp.characterConnections[targetPlayer] then
        pcall(function()
            esp.characterConnections[targetPlayer]:Disconnect()
        end)
    end

    esp.characterConnections[targetPlayer] =
        targetPlayer.CharacterAdded:Connect(function()
            removePlayerESP(targetPlayer)

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
    esp.enabled = true

    for _, targetPlayer in ipairs(Players:GetPlayers()) do
        setupESPPlayer(targetPlayer)
    end
end

local function disableESP()
    esp.enabled = false
    clearESP()
end

for _, targetPlayer in ipairs(Players:GetPlayers()) do
    setupESPPlayer(targetPlayer)
end

track(Players.PlayerAdded:Connect(function(targetPlayer)
    if esp.enabled then
        setupESPPlayer(targetPlayer)
    end
end))

track(Players.PlayerRemoving:Connect(function(targetPlayer)
    removePlayerESP(targetPlayer)

    local connection =
        esp.characterConnections[targetPlayer]

    if connection then
        connection:Disconnect()

        esp.characterConnections[targetPlayer] =
            nil
    end
end))

task.spawn(function()
    while alive do
        if esp.enabled then
            for _, targetPlayer in ipairs(Players:GetPlayers()) do
                if targetPlayer ~= player
                    and targetPlayer.Character then

                    local highlight =
                        esp.highlights[targetPlayer]

                    if not highlight
                        or not highlight.Parent
                        or highlight.Adornee ~= targetPlayer.Character then

                        addPlayerESP(targetPlayer)

                        highlight =
                            esp.highlights[targetPlayer]
                    end

                    if highlight then
                        local color =
                            getRoleColor(targetPlayer)

                        highlight.FillColor =
                            color

                        highlight.OutlineColor =
                            color

                        highlight.Enabled =
                            true
                    end
                end
            end
        end

        task.wait(0.1)
    end
end)

local function getGunTool()
    local gun =
        findTool(
            player,
            "Gun"
        )

    if gun then
        return gun
    end

    local containers = {}

    if character then
        table.insert(
            containers,
            character
        )
    end

    local backpack =
        player:FindFirstChildOfClass("Backpack")

    if backpack then
        table.insert(
            containers,
            backpack
        )
    end

    for _, container in ipairs(containers) do
        for _, object in ipairs(container:GetChildren()) do
            if object:IsA("Tool") then
                local name =
                    string.lower(
                        object.Name
                    )

                if name:find("gun", 1, true)
                    or name:find("revolver", 1, true) then

                    return object
                end
            end
        end
    end

    return nil
end

local function quickShoot()
    if sheriff.shooting
        or not sheriff.quickShot
        or not humanoid
        or not character then

        return
    end

    local gun =
        getGunTool()

    if not gun then
        return
    end

    sheriff.shooting = true

    task.spawn(function()
        if gun.Parent ~= character then
            pcall(function()
                humanoid:EquipTool(gun)
            end)

            RunService.Heartbeat:Wait()
        end

        if gun.Parent == character then
            pcall(function()
                gun:Activate()
            end)

            task.wait(0.06)

            if humanoid then
                pcall(function()
                    humanoid:UnequipTools()
                end)
            end
        end

        sheriff.shooting = false
    end)
end

track(UserInputService.InputBegan:Connect(function(
    input,
    gameProcessed
)
    if gameProcessed
        or not sheriff.quickShot then

        return
    end

    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then

        quickShoot()
    end
end))

local function pressThrowKey()
    if UserInputService:GetFocusedTextBox() then
        return false
    end

    if keypress
        and keyrelease then

        local success =
            pcall(function()
                keypress(0x45)

                task.wait(0.06)

                keyrelease(0x45)
            end)

        if success then
            return true
        end
    end

    if not VirtualInputManager then
        return false
    end

    return pcall(function()
        VirtualInputManager:SendKeyEvent(
            true,
            Enum.KeyCode.E,
            false,
            game
        )

        task.wait(0.06)

        VirtualInputManager:SendKeyEvent(
            false,
            Enum.KeyCode.E,
            false,
            game
        )
    end)
end

local function findVisibleThrowButton()
    local playerGui =
        player:FindFirstChildOfClass("PlayerGui")

    if not playerGui then
        return nil
    end

    for _, object in ipairs(playerGui:GetDescendants()) do
        if object:IsA("GuiButton")
            and object.Visible then

            local name =
                string.lower(object.Name)

            local text = ""

            if object:IsA("TextButton") then
                text =
                    string.lower(
                        object.Text or ""
                    )
            end

            if name:find("throw", 1, true)
                or text:find("throw", 1, true) then

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

throwKnifeOnce = function()
    if murderer.throwing
        or not humanoid
        or not character then

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

    murderer.throwing = true

    local success = false

    if knife.Parent ~= character then
        pcall(function()
            humanoid:EquipTool(knife)
        end)

        RunService.Heartbeat:Wait()
        RunService.Heartbeat:Wait()
    end

    if knife.Parent == character then
        success =
            pressThrowKey()

        if not success
            and UserInputService.TouchEnabled then

            success =
                pressMobileThrowButton()
        end
    end

    task.wait(0.05)

    murderer.throwing =
        false

    return success
end

local function findGunDrop()
    local exact =
        workspace:FindFirstChild(
            "GunDrop",
            true
        )

    if exact then
        return exact
    end

    for _, object in ipairs(workspace:GetDescendants()) do
        local name =
            string.lower(object.Name)
            :gsub("[%s_%-%./]", "")

        if name == "gundrop"
            or name == "droppedgun"
            or name == "dropgun"
            or name == "droppedrevolver" then

            return object
        end
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
            object:FindFirstChild("Handle")

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

local function touchObject(object, part)
    if not object
        or not part
        or not rootPart then

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

    for _, descendant in ipairs(object:GetDescendants()) do
        if descendant:IsA("ProximityPrompt")
            and fireproximityprompt then

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
        or not rootPart
        or hasGun(player) then

        return false
    end

    local drop =
        findGunDrop()

    local part =
        getObjectPart(drop)

    if not drop
        or not part then

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

            rootPart.CFrame =
                part.CFrame
                * CFrame.new(
                    0,
                    0.35,
                    0
                )

            touchObject(
                drop,
                part
            )

            RunService.Heartbeat:Wait()

            touchObject(
                drop,
                part
            )

            if rootPart
                and rootPart.Parent then

                rootPart.CFrame =
                    oldCFrame

                rootPart.AssemblyLinearVelocity =
                    oldVelocity

                rootPart.AssemblyAngularVelocity =
                    oldAngular
            end
        end)

    gunPickup.busy =
        false

    return success
end

task.spawn(function()
    while alive do
        if gunPickup.auto
            and not hasGun(player)
            and findGunDrop() then

            pickupGun()
        end

        task.wait(0.03)
    end
end)

local function isMainCoin(object)
    return object
        and object:IsA("BasePart")
        and object.Name == "MainCoin"
end

local function coinIsFullyOpaque(coin)
    if not coin
        or not coin:IsA("BasePart") then

        return false
    end

    -- user requested absolutely no transparency at all
    if coin.Transparency ~= 0 then
        return false
    end

    if coin.LocalTransparencyModifier ~= 0 then
        return false
    end

    return true
end

local function addCoin(coin)
    if not isMainCoin(coin)
        or coinFarm.coinLookup[coin] then

        return
    end

    coinFarm.coinLookup[coin] =
        true

    table.insert(
        coinFarm.coins,
        coin
    )
end

local function removeCoin(coin)
    if not coinFarm.coinLookup[coin] then
        return
    end

    coinFarm.coinLookup[coin] =
        nil

    for index = #coinFarm.coins, 1, -1 do
        if coinFarm.coins[index] == coin then
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
        coinFarm.coinLookup
    )

    -- only one full scan when script starts
    for _, object in ipairs(workspace:GetDescendants()) do
        if isMainCoin(object) then
            addCoin(object)
        end
    end
end

buildCoinCache()

track(workspace.DescendantAdded:Connect(function(object)
    if isMainCoin(object) then
        addCoin(object)
    end
end))

track(workspace.DescendantRemoving:Connect(function(object)
    if coinFarm.coinLookup[object] then
        removeCoin(object)
    end
end))

local function coinExists(coin)
    return coin
        and coinFarm.coinLookup[coin]
        and coin.Parent
        and coin:IsDescendantOf(workspace)
        and coinIsFullyOpaque(coin)
end

local function cleanCoinCache()
    for index = #coinFarm.coins, 1, -1 do
        local coin =
            coinFarm.coins[index]

        if not coin
            or not coin.Parent
            or not coin:IsDescendantOf(workspace) then

            if coin then
                coinFarm.coinLookup[coin] =
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

    local bestCoin
    local bestDistance =
        math.huge

    local rootPosition =
        rootPart.Position

    for _, coin in ipairs(coinFarm.coins) do
        if coinExists(coin) then
            local below =
                coin.Position
                - Vector3.new(
                    0,
                    coinFarm.underDistance,
                    0
                )

            local distance =
                (
                    below
                    - rootPosition
                ).Magnitude

            if distance < bestDistance then
                bestDistance =
                    distance

                bestCoin =
                    coin
            end
        end
    end

    return bestCoin
end

local function moveCoinPlatform(position)
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

local function setFarmPosition(position)
    if not rootPart
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

    rootPart.AssemblyAngularVelocity =
        Vector3.zero

    moveCoinPlatform(
        position
    )
end

local function smoothMove(
    destination,
    speed,
    stopDistance,
    targetCoin
)
    stopDistance =
        stopDistance or 0.8

    while alive
        and coinFarm.enabled
        and not coinFarm.fullBagDebounce
        and rootPart
        and rootPart.Parent do

        if targetCoin
            and not coinExists(targetCoin) then

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

        if not coinFarm.enabled then
            return false
        end

        local step =
            math.min(
                distance,
                speed * dt
            )

        local direction =
            offset.Unit

        setFarmPosition(
            current
            + direction * step
        )
    end

    return false
end

local function waitForCoinPickup(coin)
    local started =
        os.clock()

    while alive
        and coinFarm.enabled
        and os.clock() - started < 0.35 do

        if not coinExists(coin) then
            return true
        end

        task.wait(0.015)
    end

    return not coinExists(
        coin
    )
end

local function farmCoin(coin)
    if not coinExists(coin)
        or not rootPart then

        return false
    end

    local below =
        coin.Position
        - Vector3.new(
            0,
            coinFarm.underDistance,
            0
        )

    local movedBelow =
        smoothMove(
            below,
            coinFarm.travelSpeed,
            0.75,
            coin
        )

    if not movedBelow
        or not coinExists(coin) then

        return false
    end

    local coinPosition =
        coin.Position

    local wentUp =
        smoothMove(
            coinPosition,
            coinFarm.verticalSpeed,
            1.5,
            coin
        )

    if not wentUp then
        return false
    end

    if firetouchinterest
        and rootPart
        and coinExists(coin) then

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

    local pickedUp =
        waitForCoinPickup(
            coin
        )

    smoothMove(
        below,
        coinFarm.verticalSpeed,
        0.8
    )

    if pickedUp then
        coinFarm.pickedUp += 1
    end

    return pickedUp
end

local function isGuiActuallyVisible(guiObject)
    if not guiObject
        or not guiObject:IsA("GuiObject")
        or not guiObject.Visible then

        return false
    end

    if guiObject.AbsoluteSize.X <= 0
        or guiObject.AbsoluteSize.Y <= 0 then

        return false
    end

    local current =
        guiObject

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

    local position =
        guiObject.AbsolutePosition

    local size =
        guiObject.AbsoluteSize

    local viewport =
        camera.ViewportSize

    if position.X + size.X <= 0
        or position.Y + size.Y <= 0
        or position.X >= viewport.X
        or position.Y >= viewport.Y then

        return false
    end

    return true
end

local function findVisibleCoinBagFullText()
    local playerGui =
        player:FindFirstChildOfClass("PlayerGui")

    if not playerGui then
        return nil
    end

    for _, object in ipairs(playerGui:GetDescendants()) do
        if object:IsA("TextLabel")
            or object:IsA("TextButton")
            or object:IsA("TextBox") then

            local text =
                tostring(
                    object.Text or ""
                )

            local normalized =
                string.lower(text)
                :gsub("%s+", "")
                :gsub("[^%w]", "")

            if normalized:find(
                    "coinbagfull",
                    1,
                    true
                )
                and object.TextTransparency < 0.95
                and isGuiActuallyVisible(object) then

                return object
            end
        end
    end

    return nil
end

local function handleCoinBagFull()
    if not coinFarm.enabled
        or coinFarm.fullBagDebounce then

        return
    end

    local bagText =
        findVisibleCoinBagFullText()

    if not bagText then
        return
    end

    coinFarm.fullBagDebounce =
        true

    coinFarm.busy =
        false

    coinFarm.target =
        nil

    destroyCoinPlatform()
    restoreCoinCollision()

    pcall(function()
        bagText:Destroy()
    end)

    if humanoid
        and humanoid.Health > 0 then

        pcall(function()
            humanoid.Health = 0
        end)
    end

    task.spawn(function()
        local newCharacter =
            player.CharacterAdded:Wait()

        if not alive then
            return
        end

        task.wait(0.3)

        if player.Character == newCharacter then
            refreshCharacter()
        end

        if coinFarm.enabled then
            disableCoinCharacterCollision()
            ensureCoinPlatform()
        end

        coinFarm.fullBagDebounce =
            false
    end)
end

task.spawn(function()
    while alive do
        if coinFarm.enabled then
            handleCoinBagFull()
        end

        task.wait(0.05)
    end
end)

task.spawn(function()
    local lastCleanup =
        os.clock()

    while alive do
        if not coinFarm.enabled
            or coinFarm.busy
            or coinFarm.fullBagDebounce
            or not rootPart then

            task.wait(0.025)

            continue
        end

        if os.clock() - lastCleanup > 5 then
            lastCleanup =
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

            farmCoin(
                coin
            )

            coinFarm.busy =
                false
        else
            coinFarm.target =
                nil

            task.wait(0.06)
        end

        task.wait(0.008)
    end
end)

track(player.CharacterAdded:Connect(function()
    task.wait(0.2)

    refreshCharacter()

    coinFarm.busy =
        false

    gunPickup.busy =
        false

    sheriff.shooting =
        false

    murderer.throwing =
        false

    table.clear(
        coinFarm.originalCollision
    )

    if coinFarm.enabled then
        disableCoinCharacterCollision()
        ensureCoinPlatform()
    end
end))

track(UserInputService.JumpRequest:Connect(function()
    if movement.infiniteJump
        and humanoid then

        humanoid:ChangeState(
            Enum.HumanoidStateType.Jumping
        )
    end
end))

track(RunService.Stepped:Connect(function()
    if character
        and not coinFarm.enabled then

        if movement.noclip then
            for _, object in ipairs(character:GetDescendants()) do
                if object:IsA("BasePart") then
                    if movement.noclipParts[object] == nil then
                        movement.noclipParts[object] =
                            object.CanCollide
                    end

                    object.CanCollide =
                        false
                end
            end

        elseif next(movement.noclipParts) then
            restoreNoclip()
        end
    end

    if antiFling.enabled then
        disableOtherPlayerCollision()

    elseif next(antiFling.originalCollision) then
        restoreOtherPlayerCollision()
    end
end))

track(RunService.Heartbeat:Connect(function(dt)
    if not humanoid
        or not rootPart then

        return
    end

    if movement.speedEnabled
        and not coinFarm.enabled then

        humanoid.WalkSpeed =
            movement.speed

    elseif not coinFarm.enabled
        and humanoid.WalkSpeed ~= originalWalkSpeed then

        humanoid.WalkSpeed =
            originalWalkSpeed
    end

    if movement.spin
        and not coinFarm.enabled then

        rootPart.CFrame =
            rootPart.CFrame
            * CFrame.Angles(
                0,
                math.rad(
                    movement.spinSpeed * dt
                ),
                0
            )
    end

    if murderer.autoThrow
        and throwKnifeOnce
        and not murderer.throwing
        and os.clock() - murderer.lastThrow >= murderer.throwDelay then

        murderer.lastThrow =
            os.clock()

        task.spawn(
            throwKnifeOnce
        )
    end
end))

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

    return tostring(
        player.UserId
    )
end

local function validateKey(key)
    key =
        tostring(key or "")
        :gsub("^%s+", "")
        :gsub("%s+$", "")

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
                    projectId = PROJECT_ID,
                    authSecret = AUTH_SECRET,
                    hwid = getHWID(),
                    debug = false
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

    local reason =
        tostring(
            data or "invalid key"
        )

    return false,
        messages[reason]
        or reason
end

local buildChooser
local buildMainMenu

local keyOverlay = create("Frame", {
    Size =
        UDim2.fromScale(
            1,
            1
        ),

    BackgroundColor3 =
        Color3.fromRGB(
            6,
            7,
            9
        ),

    BackgroundTransparency =
        0.04,

    BorderSizePixel =
        0
}, gui)

local backgroundGradient =
    create("UIGradient", {
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
                        31,
                        33,
                        38
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
            }),

        Rotation =
            -20
    }, keyOverlay)

task.spawn(function()
    while alive
        and keyOverlay.Parent do

        tween(
            backgroundGradient,
            4,
            {
                Rotation = 20
            }
        )

        task.wait(4)

        if not keyOverlay.Parent then
            break
        end

        tween(
            backgroundGradient,
            4,
            {
                Rotation = -20
            }
        )

        task.wait(4)
    end
end)

local keyWindow = create("Frame", {
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
            348
        ),

    BackgroundColor3 =
        Color3.fromRGB(
            66,
            68,
            74
        ),

    BackgroundTransparency =
        0.22,

    BorderSizePixel =
        0,

    Active =
        true
}, keyOverlay)

create("UISizeConstraint", {
    MinSize =
        Vector2.new(
            300,
            338
        ),

    MaxSize =
        Vector2.new(
            440,
            348
        )
}, keyWindow)

addCorner(
    keyWindow,
    17
)

addStroke(
    keyWindow,
    Color3.fromRGB(
        147,
        150,
        158
    ),
    1,
    0.52
)

local keyHeader = create("Frame", {
    Size =
        UDim2.new(
            1,
            0,
            0,
            76
        ),

    BackgroundTransparency =
        1,

    Active =
        true
}, keyWindow)

makeDraggable(
    keyWindow,
    keyHeader
)

local keyIcon = create("ImageLabel", {
    Position =
        UDim2.new(
            0,
            17,
            0,
            14
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
}, keyHeader)

addCorner(
    keyIcon,
    12
)

create("TextLabel", {
    Position =
        UDim2.new(
            0,
            78,
            0,
            15
        ),

    Size =
        UDim2.new(
            1,
            -98,
            0,
            23
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
}, keyHeader)

create("TextLabel", {
    Position =
        UDim2.new(
            0,
            78,
            0,
            40
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
}, keyHeader)

local keyBox = create("TextBox", {
    Position =
        UDim2.new(
            0,
            18,
            0,
            92
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
        0.05,

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
}, keyWindow)

addCorner(
    keyBox,
    11
)

addStroke(
    keyBox
)

create("UIPadding", {
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
}, keyBox)

local continueButton = create("TextButton", {
    Position =
        UDim2.new(
            0,
            18,
            0,
            154
        ),

    Size =
        UDim2.new(
            1,
            -36,
            0,
            46
        ),

    BackgroundColor3 =
        Colors.Accent,

    BackgroundTransparency =
        0,

    BorderSizePixel =
        0,

    Text =
        "Continue",

    TextColor3 =
        Color3.fromRGB(
            14,
            20,
            16
        ),

    TextSize =
        14,

    Font =
        Enum.Font.GothamBold,

    AutoButtonColor =
        false
}, keyWindow)

addCorner(
    continueButton,
    11
)

animateButton(
    continueButton
)

local keyCard = create("Frame", {
    Position =
        UDim2.new(
            0,
            18,
            0,
            214
        ),

    Size =
        UDim2.new(
            1,
            -36,
            0,
            57
        ),

    BackgroundColor3 =
        Color3.fromRGB(
            83,
            85,
            93
        ),

    BackgroundTransparency =
        0.22,

    BorderSizePixel =
        0
}, keyWindow)

addCorner(
    keyCard,
    12
)

addStroke(
    keyCard
)

local getKeyButton = create("TextButton", {
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
            1,
            -12,
            1,
            -12
        ),

    BackgroundColor3 =
        Colors.Control,

    BackgroundTransparency =
        0.08,

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
}, keyCard)

addCorner(
    getKeyButton,
    9
)

animateButton(
    getKeyButton
)

local keyStatus = create("TextLabel", {
    Position =
        UDim2.new(
            0,
            18,
            0,
            286
        ),

    Size =
        UDim2.new(
            1,
            -36,
            0,
            38
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
}, keyWindow)

local keyBusy = false

track(getKeyButton.MouseButton1Click:Connect(function()
    local copied = false

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

    keyStatus.Text =
        copied
        and "access key link copied"
        or KEY_FLOW

    keyStatus.TextColor3 =
        copied
        and Colors.Accent
        or Colors.Text
end))

track(continueButton.MouseButton1Click:Connect(function()
    if keyBusy then
        return
    end

    keyBusy =
        true

    continueButton.Text =
        "Checking..."

    keyStatus.Text =
        "checking key..."

    keyStatus.TextColor3 =
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
            keyStatus.Text =
                "key accepted"

            keyStatus.TextColor3 =
                Colors.Accent

            task.wait(0.2)

            keyOverlay:Destroy()

            buildChooser()
        else
            keyStatus.Text =
                tostring(
                    result
                )

            keyStatus.TextColor3 =
                Colors.Danger

            continueButton.Text =
                "Continue"

            keyBusy =
                false
        end
    end)
end))

buildChooser = function()
    local overlay = create("Frame", {
        Size =
            UDim2.fromScale(
                1,
                1
            ),

        BackgroundColor3 =
            Color3.fromRGB(
                5,
                6,
                8
            ),

        BackgroundTransparency =
            0.22,

        BorderSizePixel =
            0
    }, gui)

    local panel = create("Frame", {
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
                0.82,
                0,
                0,
                230
            ),

        BackgroundColor3 =
            Color3.fromRGB(
                77,
                79,
                84
            ),

        BackgroundTransparency =
            0.22,

        BorderSizePixel =
            0,

        Active =
            true
    }, overlay)

    create("UISizeConstraint", {
        MinSize =
            Vector2.new(
                300,
                210
            ),

        MaxSize =
            Vector2.new(
                470,
                230
            )
    }, panel)

    addCorner(
        panel,
        14
    )

    addStroke(
        panel,
        Color3.fromRGB(
            160,
            163,
            170
        ),
        1,
        0.45
    )

    makeDraggable(
        panel
    )

    create("TextLabel", {
        Position =
            UDim2.new(
                0,
                20,
                0,
                20
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
            Color3.fromRGB(
                245,
                245,
                247
            ),

        TextSize =
            20,

        Font =
            Enum.Font.GothamBold
    }, panel)

    create("TextLabel", {
        Position =
            UDim2.new(
                0,
                20,
                0,
                47
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
            "pick pc or phone",

        TextColor3 =
            Color3.fromRGB(
                197,
                199,
                205
            ),

        TextSize =
            13,

        Font =
            Enum.Font.Gotham
    }, panel)

    local pcButton = create("TextButton", {
        Position =
            UDim2.new(
                0.05,
                0,
                0,
                85
            ),

        Size =
            UDim2.new(
                0.425,
                0,
                0,
                120
            ),

        BackgroundColor3 =
            Color3.fromRGB(
                41,
                43,
                48
            ),

        BackgroundTransparency =
            0.16,

        BorderSizePixel =
            0,

        Text =
            "",

        AutoButtonColor =
            false
    }, panel)

    addCorner(
        pcButton,
        12
    )

    addStroke(
        pcButton
    )

    local monitorScreen = create("Frame", {
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
                66,
                43
            ),

        BackgroundTransparency =
            1,

        BorderSizePixel =
            0
    }, pcButton)

    addCorner(
        monitorScreen,
        6
    )

    addStroke(
        monitorScreen,
        Color3.fromRGB(
            235,
            235,
            238
        ),
        3,
        0
    )

    local monitorStand = create("Frame", {
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
                60
            ),

        Size =
            UDim2.fromOffset(
                5,
                13
            ),

        BackgroundColor3 =
            Color3.fromRGB(
                235,
                235,
                238
            ),

        BorderSizePixel =
            0
    }, pcButton)

    local monitorBase = create("Frame", {
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
                72
            ),

        Size =
            UDim2.fromOffset(
                31,
                4
            ),

        BackgroundColor3 =
            Color3.fromRGB(
                235,
                235,
                238
            ),

        BorderSizePixel =
            0
    }, pcButton)

    addCorner(
        monitorBase,
        2
    )

    create("TextLabel", {
        Position =
            UDim2.new(
                0,
                0,
                1,
                -35
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
            "PC",

        TextColor3 =
            Colors.Text,

        TextSize =
            15,

        Font =
            Enum.Font.GothamBold
    }, pcButton)

    local phoneButton = create("TextButton", {
        Position =
            UDim2.new(
                0.525,
                0,
                0,
                85
            ),

        Size =
            UDim2.new(
                0.425,
                0,
                0,
                120
            ),

        BackgroundColor3 =
            Color3.fromRGB(
                41,
                43,
                48
            ),

        BackgroundTransparency =
            0.16,

        BorderSizePixel =
            0,

        Text =
            "",

        AutoButtonColor =
            false
    }, panel)

    addCorner(
        phoneButton,
        12
    )

    addStroke(
        phoneButton
    )

    local phoneDrawing = create("Frame", {
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
                13
            ),

        Size =
            UDim2.fromOffset(
                36,
                60
            ),

        BackgroundTransparency =
            1,

        BorderSizePixel =
            0
    }, phoneButton)

    addCorner(
        phoneDrawing,
        7
    )

    addStroke(
        phoneDrawing,
        Color3.fromRGB(
            235,
            235,
            238
        ),
        3,
        0
    )

    create("Frame", {
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
                6
            ),

        Size =
            UDim2.fromOffset(
                11,
                2
            ),

        BackgroundColor3 =
            Color3.fromRGB(
                235,
                235,
                238
            ),

        BorderSizePixel =
            0
    }, phoneDrawing)

    local phoneHome = create("Frame", {
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
                -5
            ),

        Size =
            UDim2.fromOffset(
                5,
                5
            ),

        BackgroundTransparency =
            1,

        BorderSizePixel =
            0
    }, phoneDrawing)

    addCorner(
        phoneHome,
        100
    )

    addStroke(
        phoneHome,
        Color3.fromRGB(
            235,
            235,
            238
        ),
        1,
        0
    )

    create("TextLabel", {
        Position =
            UDim2.new(
                0,
                0,
                1,
                -35
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
            "PHONE",

        TextColor3 =
            Colors.Text,

        TextSize =
            15,

        Font =
            Enum.Font.GothamBold
    }, phoneButton)

    animateButton(
        pcButton
    )

    animateButton(
        phoneButton
    )

    track(pcButton.MouseButton1Click:Connect(function()
        overlay:Destroy()

        if blur then
            blur:Destroy()
            blur = nil
        end

        buildMainMenu(
            "PC"
        )
    end))

    track(phoneButton.MouseButton1Click:Connect(function()
        overlay:Destroy()

        if blur then
            blur:Destroy()
            blur = nil
        end

        buildMainMenu(
            "Phone"
        )
    end))
end

buildMainMenu = function(deviceMode)
    local isPhone =
        deviceMode == "Phone"

    local compactHeight =
        isPhone and 34 or 42

    local menu = create("Frame", {
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

        BackgroundColor3 =
            Colors.Background,

        BackgroundTransparency =
            0.16,

        BorderSizePixel =
            0,

        ClipsDescendants =
            true,

        Active =
            true
    }, gui)

    if isPhone then
        menu.Size =
            UDim2.new(
                1,
                -10,
                1,
                -18
            )
    else
        menu.Size =
            UDim2.fromOffset(
                700,
                455
            )
    end

    addCorner(
        menu,
        14
    )

    addStroke(
        menu
    )

    local headerHeight =
        isPhone and 50 or 58

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

        BackgroundTransparency =
            0.12,

        BorderSizePixel =
            0,

        Active =
            true
    }, menu)

    makeDraggable(
        menu,
        header
    )

    local headerIcon = create("ImageLabel", {
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

        BorderSizePixel =
            0,

        Image =
            MM2_ICON
    }, header)

    addCorner(
        headerIcon,
        9
    )

    create("TextLabel", {
        Position =
            UDim2.new(
                0,
                57,
                0,
                8
            ),

        Size =
            UDim2.new(
                0,
                150,
                0,
                20
            ),

        BackgroundTransparency =
            1,

        Text =
            "THH HUB",

        TextColor3 =
            Colors.Text,

        TextSize =
            isPhone and 16 or 18,

        Font =
            Enum.Font.GothamBold,

        TextXAlignment =
            Enum.TextXAlignment.Left
    }, header)

    create("TextLabel", {
        Position =
            UDim2.new(
                0,
                57,
                0,
                29
            ),

        Size =
            UDim2.new(
                0,
                100,
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
    }, header)

    local minimizeButton = create("TextButton", {
        AnchorPoint =
            Vector2.new(
                1,
                0.5
            ),

        Position =
            UDim2.new(
                1,
                -44,
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
            0.08,

        BorderSizePixel =
            0,

        Text =
            "—",

        TextColor3 =
            Colors.Text,

        TextSize =
            16,

        Font =
            Enum.Font.GothamBold,

        AutoButtonColor =
            false
    }, header)

    addCorner(
        minimizeButton,
        8
    )

    local closeButton = create("TextButton", {
        AnchorPoint =
            Vector2.new(
                1,
                0.5
            ),

        Position =
            UDim2.new(
                1,
                -9,
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
            0.08,

        BorderSizePixel =
            0,

        Text =
            "×",

        TextColor3 =
            Colors.Danger,

        TextSize =
            19,

        Font =
            Enum.Font.Gotham,

        AutoButtonColor =
            false
    }, header)

    addCorner(
        closeButton,
        8
    )

    local navHolder
    local pagesHolder

    if isPhone then
        navHolder = create("Frame", {
            Position =
                UDim2.new(
                    0,
                    7,
                    0,
                    headerHeight + 5
                ),

            Size =
                UDim2.new(
                    1,
                    -14,
                    0,
                    67
                ),

            BackgroundTransparency =
                1
        }, menu)

        create("UIGridLayout", {
            CellSize =
                UDim2.new(
                    0.24,
                    -3,
                    0,
                    29
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
        }, navHolder)

        pagesHolder = create("Frame", {
            Position =
                UDim2.new(
                    0,
                    7,
                    0,
                    headerHeight + 77
                ),

            Size =
                UDim2.new(
                    1,
                    -14,
                    1,
                    -(headerHeight + 84)
                ),

            BackgroundTransparency =
                1,

            ClipsDescendants =
                true
        }, menu)
    else
        navHolder = create("Frame", {
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
                    155,
                    1,
                    -headerHeight
                ),

            BackgroundColor3 =
                Colors.Sidebar,

            BackgroundTransparency =
                0.12,

            BorderSizePixel =
                0
        }, menu)

        create("UIListLayout", {
            Padding =
                UDim.new(
                    0,
                    6
                ),

            HorizontalAlignment =
                Enum.HorizontalAlignment.Center,

            SortOrder =
                Enum.SortOrder.LayoutOrder
        }, navHolder)

        create("UIPadding", {
            PaddingTop =
                UDim.new(
                    0,
                    10
                )
        }, navHolder)

        pagesHolder = create("Frame", {
            Position =
                UDim2.new(
                    0,
                    155,
                    0,
                    headerHeight
                ),

            Size =
                UDim2.new(
                    1,
                    -155,
                    1,
                    -headerHeight
                ),

            BackgroundTransparency =
                1,

            ClipsDescendants =
                true
        }, menu)
    end

    local pages = {}
    local navButtons = {}
    local navOrder = 0

    local function createPage(name)
        local page

        if isPhone then
            page = create("Frame", {
                Name =
                    name .. "Page",

                Size =
                    UDim2.fromScale(
                        1,
                        1
                    ),

                BackgroundTransparency =
                    1,

                Visible =
                    false,

                ClipsDescendants =
                    true
            }, pagesHolder)
        else
            page = create("ScrollingFrame", {
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
                    3,

                ScrollBarImageColor3 =
                    Colors.Accent
            }, pagesHolder)
        end

        create("UIListLayout", {
            Padding =
                UDim.new(
                    0,
                    isPhone and 6 or 9
                ),

            SortOrder =
                Enum.SortOrder.LayoutOrder
        }, page)

        create("UIPadding", {
            PaddingLeft =
                UDim.new(
                    0,
                    isPhone and 4 or 13
                ),

            PaddingRight =
                UDim.new(
                    0,
                    isPhone and 4 or 13
                ),

            PaddingTop =
                UDim.new(
                    0,
                    isPhone and 4 or 13
                ),

            PaddingBottom =
                UDim.new(
                    0,
                    10
                )
        }, page)

        pages[name] =
            page

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

        local button = create("TextButton", {
            LayoutOrder =
                navOrder,

            Size =
                isPhone
                and UDim2.fromScale(
                    1,
                    1
                )
                or UDim2.fromOffset(
                    132,
                    38
                ),

            BackgroundColor3 =
                Colors.Control,

            BackgroundTransparency =
                0.08,

            BorderSizePixel =
                0,

            Text =
                name,

            TextColor3 =
                Colors.SubText,

            TextSize =
                isPhone and 9 or 11,

            Font =
                Enum.Font.GothamMedium,

            AutoButtonColor =
                false
        }, navHolder)

        addCorner(
            button,
            8
        )

        navButtons[name] =
            button

        track(button.MouseButton1Click:Connect(function()
            showPage(
                name
            )
        end))
    end

    local function section(page, title)
        local card = create("Frame", {
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
                0.2,

            BorderSizePixel =
                0
        }, page)

        addCorner(
            card,
            10
        )

        addStroke(
            card
        )

        local holder = create("Frame", {
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
        }, card)

        create("UIListLayout", {
            Padding =
                UDim.new(
                    0,
                    isPhone and 5 or 8
                )
        }, holder)

        create("UIPadding", {
            PaddingLeft =
                UDim.new(
                    0,
                    isPhone and 8 or 12
                ),

            PaddingRight =
                UDim.new(
                    0,
                    isPhone and 8 or 12
                ),

            PaddingTop =
                UDim.new(
                    0,
                    isPhone and 7 or 12
                ),

            PaddingBottom =
                UDim.new(
                    0,
                    isPhone and 7 or 12
                )
        }, holder)

        create("TextLabel", {
            Size =
                UDim2.new(
                    1,
                    0,
                    0,
                    isPhone and 18 or 22
                ),

            BackgroundTransparency =
                1,

            Text =
                title,

            TextColor3 =
                Colors.Text,

            TextSize =
                isPhone and 12 or 14,

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

        local row = create("TextButton", {
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
                0.08,

            BorderSizePixel =
                0,

            Text =
                "",

            AutoButtonColor =
                false
        }, parent)

        addCorner(
            row,
            8
        )

        create("TextLabel", {
            Position =
                UDim2.new(
                    0,
                    10,
                    0,
                    0
                ),

            Size =
                UDim2.new(
                    1,
                    -65,
                    1,
                    0
                ),

            BackgroundTransparency =
                1,

            Text =
                title,

            TextColor3 =
                Colors.Text,

            TextSize =
                isPhone and 10 or 12,

            Font =
                Enum.Font.GothamMedium,

            TextXAlignment =
                Enum.TextXAlignment.Left
        }, row)

        local switch = create("Frame", {
            AnchorPoint =
                Vector2.new(
                    1,
                    0.5
                ),

            Position =
                UDim2.new(
                    1,
                    -8,
                    0.5,
                    0
                ),

            Size =
                UDim2.fromOffset(
                    isPhone and 36 or 39,
                    isPhone and 19 or 21
                ),

            BackgroundColor3 =
                state
                and Colors.AccentDark
                or Colors.Stroke,

            BorderSizePixel =
                0
        }, row)

        addCorner(
            switch,
            100
        )

        local dotSize =
            isPhone and 13 or 15

        local dot = create("Frame", {
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

            BackgroundColor3 =
                state
                and Colors.Accent
                or Colors.SubText,

            BorderSizePixel =
                0
        }, switch)

        addCorner(
            dot,
            100
        )

        local function refresh()
            switch.BackgroundColor3 =
                state
                and Colors.AccentDark
                or Colors.Stroke

            dot.BackgroundColor3 =
                state
                and Colors.Accent
                or Colors.SubText

            dot.Position =
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
        end

        local function setState(value)
            state =
                value

            refresh()

            if callback then
                callback(
                    state
                )
            end
        end

        track(row.MouseButton1Click:Connect(function()
            setState(
                not state
            )
        end))

        refresh()

        return {
            Get =
                function()
                    return state
                end,

            Set =
                function(_, value)
                    setState(
                        value
                    )
                end
        }
    end

    local function createAction(
        parent,
        title,
        callback,
        danger
    )
        local button = create("TextButton", {
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
                    77,
                    37,
                    40
                )
                or Colors.Control,

            BackgroundTransparency =
                0.08,

            BorderSizePixel =
                0,

            Text =
                title,

            TextColor3 =
                danger
                and Colors.Danger
                or Colors.Text,

            TextSize =
                isPhone and 10 or 12,

            Font =
                Enum.Font.GothamMedium,

            AutoButtonColor =
                false
        }, parent)

        addCorner(
            button,
            8
        )

        animateButton(
            button
        )

        track(button.MouseButton1Click:Connect(function()
            if callback then
                callback()
            end
        end))

        return button
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

        local holder = create("Frame", {
            Size =
                UDim2.new(
                    1,
                    0,
                    0,
                    isPhone and 44 or 60
                ),

            BackgroundColor3 =
                Colors.Control,

            BackgroundTransparency =
                0.08,

            BorderSizePixel =
                0
        }, parent)

        addCorner(
            holder,
            8
        )

        create("TextLabel", {
            Position =
                UDim2.new(
                    0,
                    10,
                    0,
                    4
                ),

            Size =
                UDim2.new(
                    1,
                    -75,
                    0,
                    18
                ),

            BackgroundTransparency =
                1,

            Text =
                title,

            TextColor3 =
                Colors.Text,

            TextSize =
                isPhone and 9 or 11,

            Font =
                Enum.Font.GothamMedium,

            TextXAlignment =
                Enum.TextXAlignment.Left
        }, holder)

        local valueLabel = create("TextLabel", {
            AnchorPoint =
                Vector2.new(
                    1,
                    0
                ),

            Position =
                UDim2.new(
                    1,
                    -10,
                    0,
                    4
                ),

            Size =
                UDim2.fromOffset(
                    60,
                    18
                ),

            BackgroundTransparency =
                1,

            Text =
                tostring(
                    value
                ),

            TextColor3 =
                Colors.Accent,

            TextSize =
                isPhone and 9 or 11,

            Font =
                Enum.Font.GothamBold,

            TextXAlignment =
                Enum.TextXAlignment.Right
        }, holder)

        local bar = create("Frame", {
            Position =
                UDim2.new(
                    0,
                    10,
                    1,
                    isPhone and -13 or -17
                ),

            Size =
                UDim2.new(
                    1,
                    -20,
                    0,
                    isPhone and 5 or 7
                ),

            BackgroundColor3 =
                Colors.Stroke,

            BorderSizePixel =
                0
        }, holder)

        addCorner(
            bar,
            100
        )

        local fill = create("Frame", {
            BackgroundColor3 =
                Colors.Accent,

            BorderSizePixel =
                0
        }, bar)

        addCorner(
            fill,
            100
        )

        fill.Size =
            UDim2.fromScale(
                (
                    value
                    - minimum
                )
                /
                (
                    maximum
                    - minimum
                ),
                1
            )

        local dragging = false

        local function updateFromX(x)
            local percent =
                math.clamp(
                    (
                        x
                        - bar.AbsolutePosition.X
                    )
                    /
                    math.max(
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

            valueLabel.Text =
                tostring(
                    value
                )

            local display =
                (
                    value
                    - minimum
                )
                /
                (
                    maximum
                    - minimum
                )

            fill.Size =
                UDim2.fromScale(
                    display,
                    1
                )

            if callback then
                callback(
                    value
                )
            end
        end

        track(bar.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1
                or input.UserInputType == Enum.UserInputType.Touch then

                dragging = true

                updateFromX(
                    input.Position.X
                )
            end
        end))

        track(UserInputService.InputChanged:Connect(function(input)
            if dragging
                and (
                    input.UserInputType == Enum.UserInputType.MouseMovement
                    or input.UserInputType == Enum.UserInputType.Touch
                ) then

                updateFromX(
                    input.Position.X
                )
            end
        end))

        track(UserInputService.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1
                or input.UserInputType == Enum.UserInputType.Touch then

                dragging = false
            end
        end))
    end

    local Home = createPage("Home")
    local SheriffPage = createPage("Sheriff")
    local MurdererPage = createPage("Murderer")
    local InnocentPage = createPage("Innocent")
    local CoinPage = createPage("Coin Grab")
    local UpdatesPage = createPage("Updates")
    local UtilityPage = createPage("Utility")

    createNav("Home")
    createNav("Sheriff")
    createNav("Murderer")
    createNav("Innocent")
    createNav("Coin Grab")
    createNav("Updates")
    createNav("Utility")

    local home =
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
                isPhone and 20 or 25
            ),

        BackgroundTransparency =
            1,

        Text =
            "welcome "
            .. player.DisplayName,

        TextColor3 =
            Colors.Accent,

        TextSize =
            isPhone and 10 or 12,

        Font =
            Enum.Font.GothamMedium,

        TextXAlignment =
            Enum.TextXAlignment.Left
    }, home)

    local sheriffSection =
        section(
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

    create("TextLabel", {
        Size =
            UDim2.new(
                1,
                0,
                0,
                isPhone and 29 or 36
            ),

        BackgroundTransparency =
            1,

        Text =
            "click where you want to shoot — the Gun gets equipped and put away for you",

        TextColor3 =
            Colors.SubText,

        TextSize =
            isPhone and 8 or 10,

        Font =
            Enum.Font.Gotham,

        TextWrapped =
            true,

        TextXAlignment =
            Enum.TextXAlignment.Left
    }, sheriffSection)

    local murdererSection =
        section(
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
        15,
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
        section(
            InnocentPage,
            "Gun"
        )

    createAction(
        gunSection,
        "Pick Up Gun",
        function()
            pickupGun()
        end
    )

    createToggle(
        gunSection,
        "Auto Pick Up Gun",
        gunPickup.auto,
        function(value)
            gunPickup.auto =
                value
        end
    )

    local movementSection =
        section(
            InnocentPage,
            "Movement"
        )

    createToggle(
        movementSection,
        "Speed",
        movement.speedEnabled,
        function(value)
            movement.speedEnabled =
                value
        end
    )

    createSlider(
        movementSection,
        "Speed",
        16,
        250,
        movement.speed,
        function(value)
            movement.speed =
                value
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
        movement.noclip,
        function(value)
            movement.noclip =
                value

            if not value
                and not coinFarm.enabled then

                restoreNoclip()
            end
        end
    )

    createToggle(
        movementSection,
        "Spin Bot",
        movement.spin,
        function(value)
            movement.spin =
                value
        end
    )

    createSlider(
        movementSection,
        "Spin Speed",
        30,
        1500,
        movement.spinSpeed,
        function(value)
            movement.spinSpeed =
                value
        end
    )

    local coinSection =
        section(
            CoinPage,
            "Coin Grab"
        )

    createToggle(
        coinSection,
        "Coin Farm",
        coinFarm.enabled,
        function(value)
            coinFarm.enabled =
                value

            if value then
                disableCoinCharacterCollision()
                ensureCoinPlatform()
            else
                coinFarm.target =
                    nil

                coinFarm.busy =
                    false

                restoreCoinCollision()
                destroyCoinPlatform()
            end
        end
    )

    local coinCounter = create("TextLabel", {
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
            0.08,

        BorderSizePixel =
            0,

        Text =
            "Coins Picked Up: 0",

        TextColor3 =
            Colors.Accent,

        TextSize =
            isPhone and 10 or 12,

        Font =
            Enum.Font.GothamBold
    }, coinSection)

    addCorner(
        coinCounter,
        8
    )

    local coinStatus = create("TextLabel", {
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
            0.08,

        BorderSizePixel =
            0,

        Text =
            "Idle",

        TextColor3 =
            Colors.SubText,

        TextSize =
            isPhone and 9 or 11,

        Font =
            Enum.Font.GothamMedium
    }, coinSection)

    addCorner(
        coinStatus,
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

            if coinFarm.fullBagDebounce then
                coinStatus.Text =
                    "Coin bag full - resetting"

                coinStatus.TextColor3 =
                    Colors.Danger

            elseif not coinFarm.enabled then
                coinStatus.Text =
                    "Idle"

                coinStatus.TextColor3 =
                    Colors.SubText

            elseif coinFarm.target
                and coinExists(
                    coinFarm.target
                ) then

                local vertical =
                    coinFarm.target.Position.Y
                    - rootPart.Position.Y

                if vertical > 3 then
                    coinStatus.Text =
                        "Going up"

                elseif vertical < -3 then
                    coinStatus.Text =
                        "Going down"

                else
                    coinStatus.Text =
                        "Getting coin"
                end

                coinStatus.TextColor3 =
                    Colors.Accent
            else
                coinStatus.Text =
                    "Finding visible MainCoin"

                coinStatus.TextColor3 =
                    Colors.SubText
            end

            task.wait(0.1)
        end
    end)

    local updates =
        section(
            UpdatesPage,
            "what's new"
        )

    local updateLines = {
        "coin farm only grabs fully visible coins now",
        "even slightly transparent coins get skipped",
        "coin farm moves 9 studs under the coin",
        "then shoots straight up and straight back down",
        "maincoin cache keeps the farm lighter",
        "coin collision scan doesnt run every frame anymore",
        "support platform stays collidable",
        "pc / phone picker actually looks like a pc and phone now",
        "esp can still be turned on and off",
        "coin bag reset still only works on visible text",
        "anti fling is still in utility"
    }

    for _, line in ipairs(updateLines) do
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

            BackgroundTransparency =
                1,

            Text =
                "• " .. line,

            TextColor3 =
                Colors.SubText,

            TextSize =
                isPhone and 9 or 11,

            Font =
                Enum.Font.Gotham,

            TextWrapped =
                true,

            TextXAlignment =
                Enum.TextXAlignment.Left
        }, updates)
    end

    local utility =
        section(
            UtilityPage,
            "Utility"
        )

    createToggle(
        utility,
        "Anti Fling",
        antiFling.enabled,
        function(value)
            antiFling.enabled =
                value

            if value then
                disableOtherPlayerCollision()
            else
                restoreOtherPlayerCollision()
            end
        end
    )

    createAction(
        utility,
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
        utility,
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
        utility,
        "Reset Character",
        function()
            if humanoid then
                humanoid.Health = 0
            end
        end
    )

    createAction(
        utility,
        "Unload Menu",
        function()
            _G.THHGrowersHardUnloaded =
                true

            cleanup()
        end,
        true
    )

    local floatingButton

    if isPhone then
        floatingButton = create("ImageButton", {
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
                    -15
                ),

            Size =
                UDim2.fromOffset(
                    48,
                    48
                ),

            BackgroundColor3 =
                Colors.Card,

            BackgroundTransparency =
                0.08,

            BorderSizePixel =
                0,

            Image =
                MM2_ICON,

            Visible =
                false,

            AutoButtonColor =
                false,

            ZIndex =
                400
        }, gui)

        addCorner(
            floatingButton,
            13
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

    track(minimizeButton.MouseButton1Click:Connect(function()
        setMenuVisible(
            false
        )
    end))

    track(closeButton.MouseButton1Click:Connect(function()
        setMenuVisible(
            false
        )
    end))

    if floatingButton then
        track(floatingButton.MouseButton1Click:Connect(function()
            setMenuVisible(
                true
            )
        end))
    end

    track(UserInputService.InputBegan:Connect(function(
        input,
        gameProcessed
    )
        if gameProcessed
            or isPhone then

            return
        end

        if input.KeyCode == Enum.KeyCode.RightShift then
            setMenuVisible(
                not menu.Visible
            )
        end
    end))

    showPage(
        "Home"
    )
end
