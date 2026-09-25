--============================================================
-- DRIVZX RACE BUILDER V8
-- Delta / Xeno
-- Car Dealership Tycoon
--============================================================
-- • Checkpoints em ordem
-- • Cor configurável
-- • Cor de checkpoint coletado configurável
-- • Transparência 0% a 100% FUNCIONANDO
-- • Transparência continua igual quando fica vermelho
-- • Aplicar aparência no selecionado ou em todos
-- • Linha de chegada xadrez feita com Parts
-- • Loop de voltas
-- • Cronômetro
-- • Melhor volta
-- • Limite de tempo opcional
-- • Teleporte para largada no chão
-- • Teleporte com carro
-- • 3, 2, 1, VAI!
-- • Menu arrastável PC/Mobile
--============================================================

--============================================================
-- LIMPAR VERSÃO ANTERIOR
--============================================================

if getgenv().DRIVZX_RACE_CLEANUP then
    pcall(getgenv().DRIVZX_RACE_CLEANUP)
end

--============================================================
-- SERVIÇOS
--============================================================

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local SoundService = game:GetService("SoundService")

local player = Players.LocalPlayer

repeat
    task.wait()
until player.Character

--============================================================
-- CONFIGURAÇÃO
--============================================================

local CONFIG = {

    DEFAULT_CP_COLOR =
        Color3.fromRGB(45,235,90),

    DEFAULT_TAKEN_COLOR =
        Color3.fromRGB(255,55,65),

    DEFAULT_TRANSPARENCY = 0.52,

    SELECT_COLOR =
        Color3.fromRGB(255,220,60),

    BG =
        Color3.fromRGB(12,13,17),

    PANEL =
        Color3.fromRGB(22,24,30),

    BUTTON =
        Color3.fromRGB(31,34,42),

    BLUE =
        Color3.fromRGB(50,135,255),

    GREEN =
        Color3.fromRGB(35,180,90),

    RED =
        Color3.fromRGB(220,60,70),

    DEFAULT_SIZE =
        Vector3.new(22,12,1.5),

    MOVE_STEP = 1,
    ROTATE_STEP = 5,
    SIZE_STEP = 1,

    START_DISTANCE = 20,

    COUNTDOWN_TIME = 0.85,

    CAR_GROUND_CLEARANCE = 0.15
}

--============================================================
-- ESTADO
--============================================================

local checkpoints = {}
local finishLine = nil

local selectedData = nil
local selectedObject = nil

local currentCheckpoint = 1

local currentLap = 1
local totalLaps = 8

local loopEnabled = false

local raceActive = false
local raceFinished = false
local countdownRunning = false

local finishArmed = true
local previousPosition = nil

-- TIMER

local timerRunning = false

local raceStartTime = 0
local raceElapsed = 0

local lapStartTime = 0
local lapTimes = {}

-- LIMITE

local timeLimitEnabled = false
local timeLimitSeconds = 120

-- APARÊNCIA

local defaultCheckpointColor =
    CONFIG.DEFAULT_CP_COLOR

local defaultTakenColor =
    CONFIG.DEFAULT_TAKEN_COLOR

local defaultTransparency =
    CONFIG.DEFAULT_TRANSPARENCY

-- FREEZE

local currentlyFrozenCar = nil
local currentlyFrozenRoot = nil

local connections = {}

--============================================================
-- PASTA
--============================================================

local raceFolder = Instance.new("Folder")

raceFolder.Name = "DRIVZX_RACE_V8"
raceFolder.Parent = workspace

--============================================================
-- CHARACTER
--============================================================

local function getCharacter()
    return player.Character
end

local function getHumanoid()

    local character = getCharacter()

    if not character then
        return nil
    end

    return character:FindFirstChildOfClass("Humanoid")
end

local function getRoot()

    local character = getCharacter()

    if not character then
        return nil
    end

    return character:FindFirstChild("HumanoidRootPart")
end

--============================================================
-- CARRO
--============================================================

local function getCar()

    local char = player.Character

    if not char then
        return nil
    end

    local hum =
        char:FindFirstChildOfClass("Humanoid")

    if not hum then
        return nil
    end

    local seat = hum.SeatPart

    if not seat then
        return nil
    end

    local root = seat.AssemblyRootPart

    if not root then
        return nil
    end

    return root:FindFirstAncestorWhichIsA("Model")
end

local function getCurrentSeat()

    local humanoid = getHumanoid()

    if not humanoid then
        return nil
    end

    return humanoid.SeatPart
end

--============================================================
-- FREEZE CAR
--============================================================

local function freezeCar(car, state)

    if not car then
        return
    end

    for _,object in ipairs(car:GetDescendants()) do

        if object:IsA("BasePart") then

            object.Anchored = state

            object.AssemblyLinearVelocity =
                Vector3.zero

            object.AssemblyAngularVelocity =
                Vector3.zero
        end
    end
end

local function stopCarVelocity(car)

    if not car then
        return
    end

    for _,object in ipairs(car:GetDescendants()) do

        if object:IsA("BasePart") then

            object.AssemblyLinearVelocity =
                Vector3.zero

            object.AssemblyAngularVelocity =
                Vector3.zero
        end
    end
end

--============================================================
-- RACER POSITION
--============================================================

local function getRacerPosition()

    local car = getCar()

    if car and car.PrimaryPart then
        return car.PrimaryPart.Position
    end

    local seat = getCurrentSeat()

    if seat then
        return seat.Position
    end

    local root = getRoot()

    if root then
        return root.Position
    end

    return nil
end

--============================================================
-- TEMPO
--============================================================

local function formatTime(seconds)

    seconds = math.max(0, seconds or 0)

    local minutes =
        math.floor(seconds / 60)

    local remaining =
        seconds % 60

    return string.format(
        "%02d:%05.2f",
        minutes,
        remaining
    )
end

local function getBestLap()

    local best = nil

    for _,value in ipairs(lapTimes) do

        if not best or value < best then
            best = value
        end
    end

    return best
end

--============================================================
-- RAYCAST DO CHÃO
--============================================================

local function getGroundPosition(position, extraIgnore)

    local params = RaycastParams.new()

    params.FilterType =
        Enum.RaycastFilterType.Exclude

    local ignore = {
        raceFolder
    }

    local character = getCharacter()

    if character then
        table.insert(ignore, character)
    end

    if extraIgnore then
        table.insert(ignore, extraIgnore)
    end

    params.FilterDescendantsInstances =
        ignore

    params.IgnoreWater = false

    local origin =
        Vector3.new(
            position.X,
            position.Y + 180,
            position.Z
        )

    local result =
        workspace:Raycast(
            origin,
            Vector3.new(0,-600,0),
            params
        )

    if result then
        return result.Position, result.Normal
    end

    return position, Vector3.yAxis
end

--============================================================
-- LOCAL PARA CRIAR CHECKPOINT
--============================================================

local function getPlacementCFrame()

    local sourceCF = nil

    local car = getCar()

    if car and car.PrimaryPart then

        sourceCF =
            car.PrimaryPart.CFrame

    else

        local root = getRoot()

        if root then
            sourceCF = root.CFrame
        end
    end

    if not sourceCF then
        return CFrame.new()
    end

    local raw =
        (
            sourceCF *
            CFrame.new(0,0,-25)
        ).Position

    local ground =
        getGroundPosition(
            raw,
            car
        )

    local look =
        sourceCF.LookVector

    local flat =
        Vector3.new(
            look.X,
            0,
            look.Z
        )

    if flat.Magnitude < 0.01 then
        flat = Vector3.new(0,0,-1)
    end

    flat = flat.Unit

    local position =
        ground +
        Vector3.new(
            0,
            CONFIG.DEFAULT_SIZE.Y / 2,
            0
        )

    return CFrame.lookAt(
        position,
        position + flat
    )
end

--============================================================
-- SELECTION BOX
--============================================================

local function addSelectionBox(part)

    local box =
        Instance.new("SelectionBox")

    box.Name = "DZSelection"

    box.Adornee = part

    box.Visible = false

    box.SurfaceTransparency = 1

    box.LineThickness = 0.045

    box.Color3 =
        CONFIG.SELECT_COLOR

    box.Parent = part
end

--============================================================
-- VISUAL CHECKPOINT
-- TRANSPARÊNCIA CORRIGIDA
--============================================================

local function applyCheckpointVisual(data)

    if not data or not data.part then
        return
    end

    -- MUITO IMPORTANTE:
    -- A transparência escolhida é SEMPRE respeitada.

    data.part.Transparency =
        math.clamp(
            data.transparency
            or
            defaultTransparency,
            0,
            1
        )

    if data.taken then

        data.part.Color =
            data.takenColor
            or
            defaultTakenColor

    else

        data.part.Color =
            data.color
            or
            defaultCheckpointColor
    end
end

--============================================================
-- CHECKPOINT
--============================================================

local function createCheckpoint()

    local index =
        #checkpoints + 1

    local part =
        Instance.new("Part")

    part.Name =
        "Checkpoint_"
        ..
        tostring(index)

    part.Anchored = true

    part.CanCollide = false
    part.CanTouch = false
    part.CanQuery = false

    part.CastShadow = false

    part.Material =
        Enum.Material.Neon

    part.Size =
        CONFIG.DEFAULT_SIZE

    part.CFrame =
        getPlacementCFrame()

    part.Parent =
        raceFolder

    addSelectionBox(part)

    local data = {

        type = "Checkpoint",

        number = index,

        part = part,

        taken = false,

        color =
            defaultCheckpointColor,

        takenColor =
            defaultTakenColor,

        transparency =
            defaultTransparency
    }

    table.insert(
        checkpoints,
        data
    )

    applyCheckpointVisual(data)

    return data
end

--============================================================
-- CHEGADA XADREZ
--============================================================

local function destroyFinishTiles(data)

    if not data or not data.tiles then
        return
    end

    for _,tile in ipairs(data.tiles) do

        if tile and tile.Parent then
            tile:Destroy()
        end
    end

    data.tiles = {}
end

local function updateFinishTiles(data)

    if not data or not data.part then
        return
    end

    destroyFinishTiles(data)

    local trigger =
        data.part

    local columns = 10
    local rows = 4

    local width =
        trigger.Size.X

    local depth =
        math.max(
            trigger.Size.Z * 3,
            4
        )

    local tileWidth =
        width / columns

    local tileDepth =
        depth / rows

    local localY =
        -(trigger.Size.Y / 2)
        +
        0.08

    for row = 1,rows do

        for column = 1,columns do

            local tile =
                Instance.new("Part")

            tile.Name =
                "FinishTile"

            tile.Anchored = true

            tile.CanCollide = false
            tile.CanTouch = false
            tile.CanQuery = false

            tile.CastShadow = false

            tile.Material =
                Enum.Material.SmoothPlastic

            tile.Size =
                Vector3.new(
                    tileWidth + 0.02,
                    0.08,
                    tileDepth + 0.02
                )

            if
                (row + column) % 2 == 0
            then

                tile.Color =
                    Color3.fromRGB(
                        245,
                        245,
                        245
                    )
            else

                tile.Color =
                    Color3.fromRGB(
                        15,
                        15,
                        15
                    )
            end

            local x =
                -width/2
                +
                tileWidth/2
                +
                (column - 1)
                *
                tileWidth

            local z =
                -depth/2
                +
                tileDepth/2
                +
                (row - 1)
                *
                tileDepth

            tile.CFrame =
                trigger.CFrame
                *
                CFrame.new(
                    x,
                    localY,
                    z
                )

            tile.Parent =
                raceFolder

            table.insert(
                data.tiles,
                tile
            )
        end
    end
end

local function createFinish()

    if finishLine then
        return finishLine
    end

    local part =
        Instance.new("Part")

    part.Name =
        "FinishTrigger"

    part.Anchored = true

    part.CanCollide = false
    part.CanTouch = false
    part.CanQuery = false

    part.CastShadow = false

    part.Transparency = 1

    part.Size =
        CONFIG.DEFAULT_SIZE

    part.CFrame =
        getPlacementCFrame()

    part.Parent =
        raceFolder

    addSelectionBox(part)

    finishLine = {

        type = "Finish",

        part = part,

        tiles = {}
    }

    updateFinishTiles(
        finishLine
    )

    return finishLine
end

local function refreshFinish()

    if
        selectedData
        and
        selectedData.type == "Finish"
    then

        updateFinishTiles(
            selectedData
        )
    end
end

--============================================================
-- CHECKPOINT NUMBERS
--============================================================

local function updateNumbers()

    for index,data
        in ipairs(checkpoints)
    do

        data.number = index

        data.part.Name =
            "Checkpoint_"
            ..
            tostring(index)
    end
end

--============================================================
-- RESET
--============================================================

local function resetCheckpoints()

    currentCheckpoint = 1

    for _,data in ipairs(checkpoints) do

        data.taken = false

        applyCheckpointVisual(data)
    end
end

--============================================================
-- HITBOX
--============================================================

local function pointInsidePart(part, position)

    if not part or not position then
        return false
    end

    local localPoint =
        part.CFrame:
        PointToObjectSpace(
            position
        )

    local half =
        part.Size / 2
        +
        Vector3.new(
            3,
            4,
            4
        )

    return
        math.abs(localPoint.X)
        <=
        half.X

        and

        math.abs(localPoint.Y)
        <=
        half.Y

        and

        math.abs(localPoint.Z)
        <=
        half.Z
end

--============================================================
-- DETECÇÃO ENTRE FRAMES
--============================================================

local function segmentHitsPart(
    part,
    worldStart,
    worldEnd
)

    if
        not part
        or
        not worldStart
        or
        not worldEnd
    then
        return false
    end

    if pointInsidePart(part,worldEnd) then
        return true
    end

    local p1 =
        part.CFrame:
        PointToObjectSpace(
            worldStart
        )

    local p2 =
        part.CFrame:
        PointToObjectSpace(
            worldEnd
        )

    local direction =
        p2 - p1

    local half =
        part.Size / 2
        +
        Vector3.new(
            3,
            4,
            4
        )

    local tMin = 0
    local tMax = 1

    local function axis(
        startValue,
        directionValue,
        minimum,
        maximum
    )

        if
            math.abs(directionValue)
            <
            0.00001
        then

            return
                startValue >= minimum
                and
                startValue <= maximum
        end

        local inv =
            1 / directionValue

        local t1 =
            (minimum - startValue)
            *
            inv

        local t2 =
            (maximum - startValue)
            *
            inv

        if t1 > t2 then
            t1,t2 = t2,t1
        end

        tMin =
            math.max(tMin,t1)

        tMax =
            math.min(tMax,t2)

        return tMin <= tMax
    end

    if not axis(
        p1.X,
        direction.X,
        -half.X,
        half.X
    ) then
        return false
    end

    if not axis(
        p1.Y,
        direction.Y,
        -half.Y,
        half.Y
    ) then
        return false
    end

    if not axis(
        p1.Z,
        direction.Z,
        -half.Z,
        half.Z
    ) then
        return false
    end

    return true
end

--============================================================
-- GUI
--============================================================

local gui =
    Instance.new("ScreenGui")

gui.Name =
    "DRIVZX_RACE_V8"

gui.ResetOnSpawn = false
gui.IgnoreGuiInset = true

pcall(function()
    gui.Parent = gethui()
end)

if not gui.Parent then

    gui.Parent =
        player:
        WaitForChild("PlayerGui")
end

--============================================================
-- MENU
--============================================================

local menu =
    Instance.new("Frame")

menu.Size =
    UDim2.fromOffset(
        430,
        560
    )

menu.Position =
    UDim2.new(
        0.5,
        -215,
        0.5,
        -280
    )

menu.BackgroundColor3 =
    CONFIG.BG

menu.BorderSizePixel = 0

menu.Parent = gui

Instance.new(
    "UICorner",
    menu
).CornerRadius =
    UDim.new(0,16)

local stroke =
    Instance.new("UIStroke")

stroke.Color =
    Color3.fromRGB(
        65,
        70,
        90
    )

stroke.Transparency =
    0.35

stroke.Parent =
    menu

--============================================================
-- HEADER
--============================================================

local header =
    Instance.new("Frame")

header.Size =
    UDim2.new(
        1,
        0,
        0,
        56
    )

header.BackgroundColor3 =
    Color3.fromRGB(
        18,
        20,
        26
    )

header.BorderSizePixel =
    0

header.Parent =
    menu

Instance.new(
    "UICorner",
    header
).CornerRadius =
    UDim.new(0,16)

local title =
    Instance.new("TextLabel")

title.Size =
    UDim2.new(
        1,
        -70,
        0,
        32
    )

title.Position =
    UDim2.fromOffset(
        17,
        4
    )

title.BackgroundTransparency =
    1

title.Text =
    "DRIVZX • RACE BUILDER"

title.TextColor3 =
    Color3.new(1,1,1)

title.Font =
    Enum.Font.GothamBold

title.TextSize = 16

title.TextXAlignment =
    Enum.TextXAlignment.Left

title.Parent =
    header

local subtitle =
    Instance.new("TextLabel")

subtitle.Size =
    UDim2.new(
        1,
        -70,
        0,
        16
    )

subtitle.Position =
    UDim2.fromOffset(
        17,
        34
    )

subtitle.BackgroundTransparency =
    1

subtitle.Text =
    "Race Builder V8"

subtitle.TextColor3 =
    Color3.fromRGB(
        125,
        135,
        160
    )

subtitle.Font =
    Enum.Font.Gotham

subtitle.TextSize = 10

subtitle.TextXAlignment =
    Enum.TextXAlignment.Left

subtitle.Parent =
    header

local minimize =
    Instance.new("TextButton")

minimize.Size =
    UDim2.fromOffset(
        36,
        36
    )

minimize.Position =
    UDim2.new(
        1,
        -46,
        0,
        10
    )

minimize.BackgroundColor3 =
    CONFIG.BUTTON

minimize.Text =
    "—"

minimize.TextColor3 =
    Color3.new(1,1,1)

minimize.Font =
    Enum.Font.GothamBold

minimize.TextSize =
    18

minimize.Parent =
    header

Instance.new(
    "UICorner",
    minimize
).CornerRadius =
    UDim.new(0,9)

--============================================================
-- SCROLL
--============================================================

local scroll =
    Instance.new("ScrollingFrame")

scroll.Size =
    UDim2.new(
        1,
        -16,
        1,
        -70
    )

scroll.Position =
    UDim2.fromOffset(
        8,
        62
    )

scroll.BackgroundTransparency =
    1

scroll.BorderSizePixel =
    0

scroll.ScrollBarThickness =
    3

scroll.ScrollBarImageColor3 =
    Color3.fromRGB(
        90,
        95,
        115
    )

scroll.Parent =
    menu

local y = 5

--============================================================
-- GUI HELPERS
--============================================================

local function section(text)

    local label =
        Instance.new("TextLabel")

    label.Size =
        UDim2.new(
            1,
            -20,
            0,
            28
        )

    label.Position =
        UDim2.fromOffset(
            10,
            y
        )

    y += 34

    label.BackgroundTransparency =
        1

    label.Text =
        string.upper(text)

    label.TextColor3 =
        Color3.fromRGB(
            135,
            150,
            185
        )

    label.Font =
        Enum.Font.GothamBold

    label.TextSize =
        11

    label.TextXAlignment =
        Enum.TextXAlignment.Left

    label.Parent =
        scroll
end

local function button(
    text,
    callback
)

    local btn =
        Instance.new("TextButton")

    btn.Size =
        UDim2.new(
            1,
            -20,
            0,
            42
        )

    btn.Position =
        UDim2.fromOffset(
            10,
            y
        )

    y += 48

    btn.BackgroundColor3 =
        CONFIG.BUTTON

    btn.Text =
        text

    btn.TextColor3 =
        Color3.new(1,1,1)

    btn.Font =
        Enum.Font.GothamMedium

    btn.TextSize =
        13

    btn.Parent =
        scroll

    Instance.new(
        "UICorner",
        btn
    ).CornerRadius =
        UDim.new(0,9)

    btn.MouseButton1Click:
    Connect(callback)

    return btn
end

local function dual(
    leftText,
    leftCallback,
    rightText,
    rightCallback
)

    local left =
        Instance.new("TextButton")

    left.Size =
        UDim2.new(
            0.5,
            -13,
            0,
            40
        )

    left.Position =
        UDim2.fromOffset(
            10,
            y
        )

    left.BackgroundColor3 =
        CONFIG.BUTTON

    left.Text =
        leftText

    left.TextColor3 =
        Color3.new(1,1,1)

    left.Font =
        Enum.Font.GothamMedium

    left.TextSize =
        12

    left.Parent =
        scroll

    Instance.new(
        "UICorner",
        left
    ).CornerRadius =
        UDim.new(0,8)

    local right =
        Instance.new("TextButton")

    right.Size =
        UDim2.new(
            0.5,
            -13,
            0,
            40
        )

    right.Position =
        UDim2.new(
            0.5,
            3,
            0,
            y
        )

    right.BackgroundColor3 =
        CONFIG.BUTTON

    right.Text =
        rightText

    right.TextColor3 =
        Color3.new(1,1,1)

    right.Font =
        Enum.Font.GothamMedium

    right.TextSize =
        12

    right.Parent =
        scroll

    Instance.new(
        "UICorner",
        right
    ).CornerRadius =
        UDim.new(0,8)

    y += 46

    left.MouseButton1Click:
    Connect(leftCallback)

    right.MouseButton1Click:
    Connect(rightCallback)
end

local function textBox(value)

    local box =
        Instance.new("TextBox")

    box.Size =
        UDim2.new(
            1,
            -20,
            0,
            40
        )

    box.Position =
        UDim2.fromOffset(
            10,
            y
        )

    y += 46

    box.BackgroundColor3 =
        CONFIG.PANEL

    box.Text =
        tostring(value)

    box.TextColor3 =
        Color3.new(1,1,1)

    box.Font =
        Enum.Font.Gotham

    box.TextSize =
        13

    box.ClearTextOnFocus =
        true

    box.Parent =
        scroll

    Instance.new(
        "UICorner",
        box
    ).CornerRadius =
        UDim.new(0,8)

    return box
end

--============================================================
-- RGB INPUT
--============================================================

local function rgbBoxes(r,g,b)

    local values = {
        r,g,b
    }

    local names = {
        "R","G","B"
    }

    local result = {}

    for i = 1,3 do

        local box =
            Instance.new("TextBox")

        box.Size =
            UDim2.new(
                1/3,
                -10,
                0,
                40
            )

        box.Position =
            UDim2.new(
                (i-1)/3,
                10,
                0,
                y
            )

        box.BackgroundColor3 =
            CONFIG.PANEL

        box.Text =
            tostring(values[i])

        box.PlaceholderText =
            names[i]

        box.TextColor3 =
            Color3.new(1,1,1)

        box.Font =
            Enum.Font.Gotham

        box.TextSize =
            12

        box.ClearTextOnFocus =
            true

        box.Parent =
            scroll

        Instance.new(
            "UICorner",
            box
        ).CornerRadius =
            UDim.new(0,8)

        result[i] = box
    end

    y += 46

    return
        result[1],
        result[2],
        result[3]
end

local function readNumber(
    box,
    fallback
)

    local value =
        tonumber(box.Text)

    return value or fallback
end

local function readRGB(
    rBox,
    gBox,
    bBox
)

    local r =
        math.clamp(
            readNumber(rBox,255),
            0,
            255
        )

    local g =
        math.clamp(
            readNumber(gBox,255),
            0,
            255
        )

    local b =
        math.clamp(
            readNumber(bBox,255),
            0,
            255
        )

    return Color3.fromRGB(
        r,
        g,
        b
    )
end

--============================================================
-- SELEÇÃO
--============================================================

local selectedLabel = nil

local transparencyLabel = nil
local transparencyValue = 52

local cpR
local cpG
local cpB

local takenR
local takenG
local takenB

local function colorToRGB(color)

    return
        math.floor(color.R*255 + 0.5),
        math.floor(color.G*255 + 0.5),
        math.floor(color.B*255 + 0.5)
end

local function updateAppearanceControls(data)

    if
        not data
        or
        data.type ~= "Checkpoint"
    then
        return
    end

    transparencyValue =
        math.floor(
            (data.transparency or 0)
            *
            100
            +
            0.5
        )

    if transparencyLabel then

        transparencyLabel.Text =
            "Transparência: "
            ..
            tostring(transparencyValue)
            ..
            "%"
    end

    if cpR and cpG and cpB then

        local r,g,b =
            colorToRGB(
                data.color
            )

        cpR.Text = tostring(r)
        cpG.Text = tostring(g)
        cpB.Text = tostring(b)
    end

    if takenR and takenG and takenB then

        local r,g,b =
            colorToRGB(
                data.takenColor
            )

        takenR.Text = tostring(r)
        takenG.Text = tostring(g)
        takenB.Text = tostring(b)
    end
end

local function selectObject(data)

    if selectedObject then

        local old =
            selectedObject:
            FindFirstChild(
                "DZSelection"
            )

        if old then
            old.Visible = false
        end
    end

    selectedData = data

    selectedObject =
        data
        and data.part
        or nil

    if selectedObject then

        local selection =
            selectedObject:
            FindFirstChild(
                "DZSelection"
            )

        if selection then
            selection.Visible = true
        end
    end

    if selectedLabel then

        if not data then

            selectedLabel.Text =
                "Nenhum objeto selecionado"

        elseif data.type == "Finish" then

            selectedLabel.Text =
                "Selecionado: Linha de Chegada"

        else

            selectedLabel.Text =
                "Selecionado: Checkpoint "
                ..
                tostring(
                    data.number
                )
        end
    end

    updateAppearanceControls(data)
end

--============================================================
-- HUD
--============================================================

local hud =
    Instance.new("Frame")

hud.Size =
    UDim2.fromOffset(
        400,
        74
    )

hud.Position =
    UDim2.new(
        0.5,
        -200,
        0,
        20
    )

hud.BackgroundColor3 =
    CONFIG.BG

hud.BackgroundTransparency =
    0.08

hud.Parent =
    gui

Instance.new(
    "UICorner",
    hud
).CornerRadius =
    UDim.new(0,13)

local hudStatus =
    Instance.new("TextLabel")

hudStatus.Size =
    UDim2.new(
        1,
        -20,
        0,
        28
    )

hudStatus.Position =
    UDim2.fromOffset(
        10,
        5
    )

hudStatus.BackgroundTransparency =
    1

hudStatus.Text =
    "Monte o traçado e inicie a corrida"

hudStatus.TextColor3 =
    Color3.new(1,1,1)

hudStatus.Font =
    Enum.Font.GothamBold

hudStatus.TextSize =
    12

hudStatus.Parent =
    hud

local hudTimer =
    Instance.new("TextLabel")

hudTimer.Size =
    UDim2.new(
        1,
        -20,
        0,
        30
    )

hudTimer.Position =
    UDim2.fromOffset(
        10,
        35
    )

hudTimer.BackgroundTransparency =
    1

hudTimer.Text =
    "TEMPO  00:00.00"

hudTimer.TextColor3 =
    Color3.fromRGB(
        140,
        190,
        255
    )

hudTimer.Font =
    Enum.Font.GothamBold

hudTimer.TextSize =
    15

hudTimer.Parent =
    hud

--============================================================
-- COUNTDOWN
--============================================================

local countdownFrame =
    Instance.new("Frame")

countdownFrame.Size =
    UDim2.fromScale(1,1)

countdownFrame.BackgroundTransparency =
    1

countdownFrame.Visible =
    false

countdownFrame.ZIndex =
    100

countdownFrame.Parent =
    gui

local countdownText =
    Instance.new("TextLabel")

countdownText.Size =
    UDim2.fromOffset(
        500,
        230
    )

countdownText.Position =
    UDim2.new(
        0.5,
        -250,
        0.5,
        -115
    )

countdownText.BackgroundTransparency =
    1

countdownText.Text =
    "3"

countdownText.TextColor3 =
    Color3.new(1,1,1)

countdownText.TextStrokeTransparency =
    0.2

countdownText.TextStrokeColor3 =
    Color3.new(0,0,0)

countdownText.Font =
    Enum.Font.GothamBlack

countdownText.TextScaled =
    true

countdownText.ZIndex =
    101

countdownText.Parent =
    countdownFrame

--============================================================
-- SOM
--============================================================

local countdownSound =
    Instance.new("Sound")

countdownSound.Name =
    "DZCountdown"

countdownSound.SoundId =
    "rbxasset://sounds/electronicpingshort.wav"

countdownSound.Volume =
    1

countdownSound.Parent =
    SoundService

local function playCountdownSound(speed)

    countdownSound:Stop()

    countdownSound.PlaybackSpeed =
        speed or 1

    countdownSound.TimePosition =
        0

    countdownSound:Play()
end

--============================================================
-- TELA FINAL
--============================================================

local finishScreen =
    Instance.new("Frame")

finishScreen.Size =
    UDim2.fromOffset(
        400,
        270
    )

finishScreen.Position =
    UDim2.new(
        0.5,
        -200,
        0.5,
        -135
    )

finishScreen.BackgroundColor3 =
    CONFIG.BG

finishScreen.Visible =
    false

finishScreen.Parent =
    gui

Instance.new(
    "UICorner",
    finishScreen
).CornerRadius =
    UDim.new(0,17)

local finishTitle =
    Instance.new("TextLabel")

finishTitle.Size =
    UDim2.new(
        1,
        -30,
        0,
        70
    )

finishTitle.Position =
    UDim2.fromOffset(
        15,
        15
    )

finishTitle.BackgroundTransparency =
    1

finishTitle.Text =
    "🏁 CORRIDA FINALIZADA!"

finishTitle.TextColor3 =
    Color3.new(1,1,1)

finishTitle.Font =
    Enum.Font.GothamBold

finishTitle.TextSize =
    22

finishTitle.Parent =
    finishScreen

local finalTimeLabel =
    Instance.new("TextLabel")

finalTimeLabel.Size =
    UDim2.new(
        1,
        -30,
        0,
        70
    )

finalTimeLabel.Position =
    UDim2.fromOffset(
        15,
        75
    )

finalTimeLabel.BackgroundTransparency =
    1

finalTimeLabel.Text =
    "TEMPO TOTAL\n00:00.00"

finalTimeLabel.TextColor3 =
    Color3.fromRGB(
        140,
        190,
        255
    )

finalTimeLabel.Font =
    Enum.Font.GothamBold

finalTimeLabel.TextSize =
    16

finalTimeLabel.Parent =
    finishScreen

local bestLapLabel =
    Instance.new("TextLabel")

bestLapLabel.Size =
    UDim2.new(
        1,
        -30,
        0,
        30
    )

bestLapLabel.Position =
    UDim2.fromOffset(
        15,
        145
    )

bestLapLabel.BackgroundTransparency =
    1

bestLapLabel.Text =
    ""

bestLapLabel.TextColor3 =
    Color3.fromRGB(
        190,
        195,
        210
    )

bestLapLabel.Font =
    Enum.Font.GothamMedium

bestLapLabel.TextSize =
    12

bestLapLabel.Parent =
    finishScreen

local finalRestart =
    Instance.new("TextButton")

finalRestart.Size =
    UDim2.new(
        1,
        -40,
        0,
        50
    )

finalRestart.Position =
    UDim2.new(
        0,
        20,
        1,
        -70
    )

finalRestart.BackgroundColor3 =
    CONFIG.BLUE

finalRestart.Text =
    "RECOMEÇAR"

finalRestart.TextColor3 =
    Color3.new(1,1,1)

finalRestart.Font =
    Enum.Font.GothamBold

finalRestart.TextSize =
    14

finalRestart.Parent =
    finishScreen

Instance.new(
    "UICorner",
    finalRestart
).CornerRadius =
    UDim.new(0,10)

--============================================================
-- TELEPORTE LARGADA
--============================================================

local function teleportToStart()

    local first =
        checkpoints[1]

    if
        not first
        or
        not first.part
    then
        return false
    end

    local checkpoint =
        first.part

    local rawStart =
        (
            checkpoint.CFrame
            *
            CFrame.new(
                0,
                0,
                CONFIG.START_DISTANCE
            )
        ).Position

    local car =
        getCar()

    local groundPosition =
        getGroundPosition(
            rawStart,
            car
        )

    --========================================================
    -- CARRO
    --========================================================

    if car and car.PrimaryPart then

        local seat =
            getCurrentSeat()

        freezeCar(
            car,
            true
        )

        currentlyFrozenCar =
            car

        local seatPosition =
            Vector3.new(
                groundPosition.X,
                groundPosition.Y + 3,
                groundPosition.Z
            )

        local targetPosition =
            Vector3.new(
                checkpoint.Position.X,
                seatPosition.Y,
                checkpoint.Position.Z
            )

        local desiredSeatCFrame =
            CFrame.lookAt(
                seatPosition,
                targetPosition
            )

        local targetCarCF =
            desiredSeatCFrame

        if seat then

            local primaryToSeat =
                car.PrimaryPart.CFrame:
                ToObjectSpace(
                    seat.CFrame
                )

            targetCarCF =
                desiredSeatCFrame
                *
                primaryToSeat:Inverse()
        end

        pcall(function()

            car:SetPrimaryPartCFrame(
                targetCarCF
            )
        end)

        stopCarVelocity(car)

        task.wait(0.08)

        local boundingCF,
            boundingSize =
            car:GetBoundingBox()

        local bottom =
            boundingCF.Position.Y
            -
            boundingSize.Y/2

        local desiredBottom =
            groundPosition.Y
            +
            CONFIG.CAR_GROUND_CLEARANCE

        local correction =
            desiredBottom
            -
            bottom

        pcall(function()

            car:SetPrimaryPartCFrame(
                car.PrimaryPart.CFrame
                +
                Vector3.new(
                    0,
                    correction,
                    0
                )
            )
        end)

        stopCarVelocity(car)

        task.wait(0.15)

        return true
    end

    --========================================================
    -- PERSONAGEM
    --========================================================

    local root =
        getRoot()

    local humanoid =
        getHumanoid()

    if root then

        local height = 3

        if humanoid then
            height =
                humanoid.HipHeight + 2
        end

        local position =
            Vector3.new(
                groundPosition.X,
                groundPosition.Y + height,
                groundPosition.Z
            )

        root.CFrame =
            CFrame.lookAt(
                position,
                Vector3.new(
                    checkpoint.Position.X,
                    position.Y,
                    checkpoint.Position.Z
                )
            )

        root.AssemblyLinearVelocity =
            Vector3.zero

        root.AssemblyAngularVelocity =
            Vector3.zero

        root.Anchored =
            true

        currentlyFrozenRoot =
            root

        return true
    end

    return false
end

--============================================================
-- LIBERAR
--============================================================

local function releaseStart()

    if
        currentlyFrozenCar
        and
        currentlyFrozenCar.Parent
    then

        stopCarVelocity(
            currentlyFrozenCar
        )

        freezeCar(
            currentlyFrozenCar,
            false
        )
    end

    currentlyFrozenCar = nil

    if
        currentlyFrozenRoot
        and
        currentlyFrozenRoot.Parent
    then

        currentlyFrozenRoot.AssemblyLinearVelocity =
            Vector3.zero

        currentlyFrozenRoot.AssemblyAngularVelocity =
            Vector3.zero

        currentlyFrozenRoot.Anchored =
            false
    end

    currentlyFrozenRoot =
        nil
end

--============================================================
-- PEGAR CHECKPOINT
--============================================================

local function takeCheckpoint(data)

    if not data or data.taken then
        return
    end

    data.taken =
        true

    -- Apenas muda a cor.
    -- Transparência NÃO muda.

    applyCheckpointVisual(data)

    currentCheckpoint += 1
end

--============================================================
-- TEMPO ESGOTADO
--============================================================

local function timeExpired()

    if not raceActive then
        return
    end

    raceActive = false
    timerRunning = false
    raceFinished = true

    raceElapsed =
        os.clock()
        -
        raceStartTime

    finishTitle.Text =
        "⌛ TEMPO ESGOTADO!"

    finishTitle.TextColor3 =
        CONFIG.RED

    finalTimeLabel.Text =
        "TEMPO\n"
        ..
        formatTime(
            raceElapsed
        )

    bestLapLabel.Text =
        "Limite: "
        ..
        formatTime(
            timeLimitSeconds
        )

    finishScreen.Visible =
        true
end

--============================================================
-- COMPLETAR VOLTA
--============================================================

local function completeLap()

    local now = os.clock()

    local lapTime =
        now
        -
        lapStartTime

    table.insert(
        lapTimes,
        lapTime
    )

    local maximum =
        loopEnabled
        and
        totalLaps
        or
        1

    if currentLap < maximum then

        currentLap += 1

        lapStartTime =
            os.clock()

        resetCheckpoints()

    else

        raceElapsed =
            now
            -
            raceStartTime

        timerRunning =
            false

        raceActive =
            false

        raceFinished =
            true

        finishTitle.Text =
            "🏁 CORRIDA FINALIZADA!"

        finishTitle.TextColor3 =
            Color3.new(1,1,1)

        finalTimeLabel.Text =
            "TEMPO TOTAL\n"
            ..
            formatTime(
                raceElapsed
            )

        local best =
            getBestLap()

        if
            loopEnabled
            and
            #lapTimes > 1
            and
            best
        then

            bestLapLabel.Text =
                "Melhor volta: "
                ..
                formatTime(best)

        else

            bestLapLabel.Text =
                ""
        end

        finishScreen.Visible =
            true
    end
end

--============================================================
-- COMEÇAR CORRIDA
--============================================================

local function startRace()

    if countdownRunning then
        return
    end

    if #checkpoints == 0 then

        hudStatus.Text =
            "Adicione pelo menos 1 checkpoint"

        return
    end

    if not finishLine then

        hudStatus.Text =
            "Adicione a linha de chegada"

        return
    end

    releaseStart()

    countdownRunning =
        true

    raceActive =
        false

    raceFinished =
        false

    timerRunning =
        false

    raceElapsed =
        0

    lapTimes =
        {}

    currentLap =
        1

    currentCheckpoint =
        1

    finishArmed =
        true

    previousPosition =
        nil

    finishScreen.Visible =
        false

    resetCheckpoints()

    hudTimer.Text =
        "TEMPO  00:00.00"

    hudStatus.Text =
        "Posicionando na largada..."

    local success =
        teleportToStart()

    if not success then

        countdownRunning =
            false

        hudStatus.Text =
            "Não consegui teleportar"

        return
    end

    countdownFrame.Visible =
        true

    -- 3

    countdownText.Text =
        "3"

    countdownText.TextColor3 =
        Color3.new(1,1,1)

    playCountdownSound(1)

    task.wait(
        CONFIG.COUNTDOWN_TIME
    )

    -- 2

    countdownText.Text =
        "2"

    playCountdownSound(1)

    task.wait(
        CONFIG.COUNTDOWN_TIME
    )

    -- 1

    countdownText.Text =
        "1"

    playCountdownSound(1)

    task.wait(
        CONFIG.COUNTDOWN_TIME
    )

    -- VAI

    countdownText.Text =
        "VAI!"

    countdownText.TextColor3 =
        Color3.fromRGB(
            65,
            255,
            105
        )

    playCountdownSound(1.65)

    releaseStart()

    local now =
        os.clock()

    raceStartTime =
        now

    lapStartTime =
        now

    timerRunning =
        true

    previousPosition =
        getRacerPosition()

    raceActive =
        true

    task.wait(0.6)

    countdownFrame.Visible =
        false

    countdownRunning =
        false
end

--============================================================
-- MENU - CRIAÇÃO
--============================================================

section("Criar Traçado")

button(
    "＋ Adicionar Checkpoint",

    function()

        selectObject(
            createCheckpoint()
        )
    end
)

button(
    "🏁 Adicionar Linha de Chegada",

    function()

        selectObject(
            createFinish()
        )
    end
)

--============================================================
-- SELEÇÃO
--============================================================

section("Selecionado")

selectedLabel =
    Instance.new("TextLabel")

selectedLabel.Size =
    UDim2.new(
        1,
        -20,
        0,
        38
    )

selectedLabel.Position =
    UDim2.fromOffset(
        10,
        y
    )

y += 44

selectedLabel.BackgroundColor3 =
    CONFIG.PANEL

selectedLabel.Text =
    "Nenhum objeto selecionado"

selectedLabel.TextColor3 =
    Color3.fromRGB(
        215,
        220,
        230
    )

selectedLabel.Font =
    Enum.Font.GothamMedium

selectedLabel.TextSize =
    12

selectedLabel.Parent =
    scroll

Instance.new(
    "UICorner",
    selectedLabel
).CornerRadius =
    UDim.new(0,8)

dual(

    "◀ Anterior",

    function()

        if #checkpoints == 0 then
            return
        end

        local index = 1

        if
            selectedData
            and
            selectedData.number
        then

            index =
                selectedData.number - 1

            if index < 1 then
                index = #checkpoints
            end
        end

        selectObject(
            checkpoints[index]
        )
    end,

    "Próximo ▶",

    function()

        if #checkpoints == 0 then
            return
        end

        local index = 1

        if
            selectedData
            and
            selectedData.number
        then

            index =
                selectedData.number + 1

            if index > #checkpoints then
                index = 1
            end
        end

        selectObject(
            checkpoints[index]
        )
    end
)

button(
    "🏁 Selecionar Chegada",

    function()

        if finishLine then
            selectObject(finishLine)
        end
    end
)

--============================================================
-- POSIÇÃO
--============================================================

section("Posição")

dual(
    "← Esquerda",

    function()

        if selectedObject then

            selectedObject.CFrame *=
                CFrame.new(
                    -CONFIG.MOVE_STEP,
                    0,
                    0
                )

            refreshFinish()
        end
    end,

    "Direita →",

    function()

        if selectedObject then

            selectedObject.CFrame *=
                CFrame.new(
                    CONFIG.MOVE_STEP,
                    0,
                    0
                )

            refreshFinish()
        end
    end
)

dual(
    "↑ Frente",

    function()

        if selectedObject then

            selectedObject.CFrame *=
                CFrame.new(
                    0,
                    0,
                    -CONFIG.MOVE_STEP
                )

            refreshFinish()
        end
    end,

    "↓ Trás",

    function()

        if selectedObject then

            selectedObject.CFrame *=
                CFrame.new(
                    0,
                    0,
                    CONFIG.MOVE_STEP
                )

            refreshFinish()
        end
    end
)

dual(
    "▲ Subir",

    function()

        if selectedObject then

            selectedObject.CFrame +=
                Vector3.new(
                    0,
                    CONFIG.MOVE_STEP,
                    0
                )

            refreshFinish()
        end
    end,

    "▼ Descer",

    function()

        if selectedObject then

            selectedObject.CFrame -=
                Vector3.new(
                    0,
                    CONFIG.MOVE_STEP,
                    0
                )

            refreshFinish()
        end
    end
)

--============================================================
-- ROTAÇÃO
--============================================================

section("Rotação")

dual(
    "↶ Girar",

    function()

        if selectedObject then

            selectedObject.CFrame *=
                CFrame.Angles(
                    0,
                    math.rad(
                        CONFIG.ROTATE_STEP
                    ),
                    0
                )

            refreshFinish()
        end
    end,

    "Girar ↷",

    function()

        if selectedObject then

            selectedObject.CFrame *=
                CFrame.Angles(
                    0,
                    math.rad(
                        -CONFIG.ROTATE_STEP
                    ),
                    0
                )

            refreshFinish()
        end
    end
)

dual(
    "Inclinar -",

    function()

        if selectedObject then

            selectedObject.CFrame *=
                CFrame.Angles(
                    math.rad(
                        -CONFIG.ROTATE_STEP
                    ),
                    0,
                    0
                )

            refreshFinish()
        end
    end,

    "Inclinar +",

    function()

        if selectedObject then

            selectedObject.CFrame *=
                CFrame.Angles(
                    math.rad(
                        CONFIG.ROTATE_STEP
                    ),
                    0,
                    0
                )

            refreshFinish()
        end
    end
)

--============================================================
-- TAMANHO
--============================================================

section("Tamanho")

dual(
    "Largura -",

    function()

        if selectedObject then

            selectedObject.Size =
                Vector3.new(

                    math.max(
                        3,
                        selectedObject.Size.X
                        -
                        CONFIG.SIZE_STEP
                    ),

                    selectedObject.Size.Y,

                    selectedObject.Size.Z
                )

            refreshFinish()
        end
    end,

    "Largura +",

    function()

        if selectedObject then

            selectedObject.Size +=
                Vector3.new(
                    CONFIG.SIZE_STEP,
                    0,
                    0
                )

            refreshFinish()
        end
    end
)

dual(
    "Altura -",

    function()

        if selectedObject then

            selectedObject.Size =
                Vector3.new(

                    selectedObject.Size.X,

                    math.max(
                        3,
                        selectedObject.Size.Y
                        -
                        CONFIG.SIZE_STEP
                    ),

                    selectedObject.Size.Z
                )

            refreshFinish()
        end
    end,

    "Altura +",

    function()

        if selectedObject then

            selectedObject.Size +=
                Vector3.new(
                    0,
                    CONFIG.SIZE_STEP,
                    0
                )

            refreshFinish()
        end
    end
)

--============================================================
-- APARÊNCIA
--============================================================

section("Cor Normal RGB")

cpR,cpG,cpB =
    rgbBoxes(
        45,
        235,
        90
    )

section("Cor Quando Pegar")

takenR,takenG,takenB =
    rgbBoxes(
        255,
        55,
        65
    )

--============================================================
-- TRANSPARÊNCIA CORRIGIDA
--============================================================

section("Transparência")

transparencyLabel =
    Instance.new("TextLabel")

transparencyLabel.Size =
    UDim2.new(
        1,
        -20,
        0,
        38
    )

transparencyLabel.Position =
    UDim2.fromOffset(
        10,
        y
    )

y += 44

transparencyLabel.BackgroundColor3 =
    CONFIG.PANEL

transparencyLabel.Text =
    "Transparência: 52%"

transparencyLabel.TextColor3 =
    Color3.new(1,1,1)

transparencyLabel.Font =
    Enum.Font.GothamBold

transparencyLabel.TextSize =
    12

transparencyLabel.Parent =
    scroll

Instance.new(
    "UICorner",
    transparencyLabel
).CornerRadius =
    UDim.new(0,8)

local function setTransparency(value)

    transparencyValue =
        math.clamp(
            value,
            0,
            100
        )

    transparencyLabel.Text =
        "Transparência: "
        ..
        tostring(
            transparencyValue
        )
        ..
        "%"

    if
        selectedData
        and
        selectedData.type
        ==
        "Checkpoint"
    then

        selectedData.transparency =
            transparencyValue / 100

        applyCheckpointVisual(
            selectedData
        )
    end
end

dual(
    "− 10%",

    function()

        setTransparency(
            transparencyValue - 10
        )
    end,

    "+ 10%",

    function()

        setTransparency(
            transparencyValue + 10
        )
    end
)

dual(
    "0% Visível",

    function()

        setTransparency(0)
    end,

    "100% Invisível",

    function()

        setTransparency(100)
    end
)

dual(
    "− 1%",

    function()

        setTransparency(
            transparencyValue - 1
        )
    end,

    "+ 1%",

    function()

        setTransparency(
            transparencyValue + 1
        )
    end
)

--============================================================
-- APLICAR COR
--============================================================

button(
    "Aplicar Cor no Selecionado",

    function()

        if
            not selectedData
            or
            selectedData.type
            ~=
            "Checkpoint"
        then
            return
        end

        selectedData.color =
            readRGB(
                cpR,
                cpG,
                cpB
            )

        selectedData.takenColor =
            readRGB(
                takenR,
                takenG,
                takenB
            )

        selectedData.transparency =
            transparencyValue / 100

        applyCheckpointVisual(
            selectedData
        )
    end
)

button(
    "Aplicar Aparência em TODOS",

    function()

        local normalColor =
            readRGB(
                cpR,
                cpG,
                cpB
            )

        local collectedColor =
            readRGB(
                takenR,
                takenG,
                takenB
            )

        local transparency =
            transparencyValue / 100

        defaultCheckpointColor =
            normalColor

        defaultTakenColor =
            collectedColor

        defaultTransparency =
            transparency

        for _,data
            in ipairs(checkpoints)
        do

            data.color =
                normalColor

            data.takenColor =
                collectedColor

            data.transparency =
                transparency

            applyCheckpointVisual(
                data
            )
        end
    end
)

--============================================================
-- GERENCIAR
--============================================================

section("Gerenciar")

button(
    "🗑 Excluir Selecionado",

    function()

        if not selectedData then
            return
        end

        raceActive = false

        if selectedData.type == "Finish" then

            destroyFinishTiles(
                finishLine
            )

            if
                finishLine
                and
                finishLine.part
            then

                finishLine.part:
                    Destroy()
            end

            finishLine =
                nil

        else

            for index,data
                in ipairs(checkpoints)
            do

                if data == selectedData then

                    if
                        data.part
                        and
                        data.part.Parent
                    then

                        data.part:
                            Destroy()
                    end

                    table.remove(
                        checkpoints,
                        index
                    )

                    break
                end
            end

            updateNumbers()
        end

        selectObject(nil)

        resetCheckpoints()
    end
)

--============================================================
-- LOOP
--============================================================

section("Modo Loop")

local lapsBox =
    textBox("8")

local loopButton

loopButton =
    button(
        "Loop: DESATIVADO",

        function()

            loopEnabled =
                not loopEnabled

            if loopEnabled then

                loopButton.Text =
                    "Loop: ATIVADO ✓"

                loopButton.BackgroundColor3 =
                    Color3.fromRGB(
                        35,
                        105,
                        65
                    )
            else

                loopButton.Text =
                    "Loop: DESATIVADO"

                loopButton.BackgroundColor3 =
                    CONFIG.BUTTON
            end
        end
    )

button(
    "Aplicar Quantidade de Voltas",

    function()

        local value =
            tonumber(
                lapsBox.Text
            )

        if value then

            totalLaps =
                math.clamp(
                    math.floor(value),
                    1,
                    999
                )

            lapsBox.Text =
                tostring(totalLaps)
        end
    end
)

--============================================================
-- TEMPO
--============================================================

section("Tempo da Corrida")

local timeLimitBox =
    textBox("120")

local limitButton

limitButton =
    button(
        "Limite de Tempo: DESATIVADO",

        function()

            timeLimitEnabled =
                not timeLimitEnabled

            if timeLimitEnabled then

                local value =
                    tonumber(
                        timeLimitBox.Text
                    )

                if value then
                    timeLimitSeconds =
                        math.max(
                            1,
                            value
                        )
                end

                limitButton.Text =
                    "Limite de Tempo: ATIVADO ✓"

                limitButton.BackgroundColor3 =
                    Color3.fromRGB(
                        115,
                        75,
                        30
                    )
            else

                limitButton.Text =
                    "Limite de Tempo: DESATIVADO"

                limitButton.BackgroundColor3 =
                    CONFIG.BUTTON
            end
        end
    )

button(
    "Aplicar Limite em Segundos",

    function()

        local value =
            tonumber(
                timeLimitBox.Text
            )

        if value then

            timeLimitSeconds =
                math.max(
                    1,
                    value
                )

            timeLimitBox.Text =
                tostring(
                    timeLimitSeconds
                )
        end
    end
)

--============================================================
-- CORRIDA
--============================================================

section("Corrida")

local startButton =
    button(
        "▶ COMEÇAR CORRIDA",

        function()

            task.spawn(
                startRace
            )
        end
    )

startButton.BackgroundColor3 =
    CONFIG.GREEN

local restartButton =
    button(
        "↻ RECOMEÇAR",

        function()

            task.spawn(
                startRace
            )
        end
    )

restartButton.BackgroundColor3 =
    CONFIG.BLUE

scroll.CanvasSize =
    UDim2.fromOffset(
        0,
        y + 30
    )

finalRestart.MouseButton1Click:
Connect(function()

    task.spawn(
        startRace
    )
end)

--============================================================
-- BOTÃO DZ
--============================================================

local bubble =
    Instance.new("TextButton")

bubble.Size =
    UDim2.fromOffset(
        58,
        58
    )

bubble.Position =
    UDim2.fromOffset(
        18,
        180
    )

bubble.BackgroundColor3 =
    Color3.fromRGB(
        20,
        22,
        29
    )

bubble.Text =
    "DZ"

bubble.TextColor3 =
    Color3.new(1,1,1)

bubble.Font =
    Enum.Font.GothamBold

bubble.TextSize =
    18

bubble.Parent =
    gui

Instance.new(
    "UICorner",
    bubble
).CornerRadius =
    UDim.new(1,0)

local bubbleStroke =
    Instance.new("UIStroke")

bubbleStroke.Color =
    CONFIG.BLUE

bubbleStroke.Thickness =
    1.5

bubbleStroke.Parent =
    bubble

bubble.MouseButton1Click:
Connect(function()

    menu.Visible =
        not menu.Visible
end)

minimize.MouseButton1Click:
Connect(function()

    menu.Visible =
        false
end)

--============================================================
-- DRAG
--============================================================

local function makeDraggable(
    dragObject,
    target
)

    local dragging = false

    local dragStart = nil
    local startPosition = nil
    local dragInput = nil

    dragObject.InputBegan:
    Connect(function(input)

        if
            input.UserInputType
            ==
            Enum.UserInputType.MouseButton1

            or

            input.UserInputType
            ==
            Enum.UserInputType.Touch
        then

            dragging = true

            dragStart =
                input.Position

            startPosition =
                target.Position
        end
    end)

    dragObject.InputChanged:
    Connect(function(input)

        if
            input.UserInputType
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

    local moveConnection =
        UserInputService.InputChanged:
        Connect(function(input)

            if
                dragging
                and
                input
                ==
                dragInput
            then

                local delta =
                    input.Position
                    -
                    dragStart

                target.Position =
                    UDim2.new(

                        startPosition.X.Scale,

                        startPosition.X.Offset
                        +
                        delta.X,

                        startPosition.Y.Scale,

                        startPosition.Y.Offset
                        +
                        delta.Y
                    )
            end
        end)

    table.insert(
        connections,
        moveConnection
    )

    local endConnection =
        UserInputService.InputEnded:
        Connect(function(input)

            if
                input.UserInputType
                ==
                Enum.UserInputType.MouseButton1

                or

                input.UserInputType
                ==
                Enum.UserInputType.Touch
            then

                dragging = false
            end
        end)

    table.insert(
        connections,
        endConnection
    )
end

makeDraggable(
    header,
    menu
)

makeDraggable(
    bubble,
    bubble
)

--============================================================
-- LOOP PRINCIPAL
--============================================================

local heartbeat =
    RunService.Heartbeat:
    Connect(function()

        local position =
            getRacerPosition()

        if not position then
            return
        end

        --====================================================
        -- TIMER
        --====================================================

        if timerRunning then

            raceElapsed =
                os.clock()
                -
                raceStartTime

            if timeLimitEnabled then

                local remaining =
                    timeLimitSeconds
                    -
                    raceElapsed

                hudTimer.Text =
                    "TEMPO "
                    ..
                    formatTime(
                        raceElapsed
                    )
                    ..
                    "  |  RESTANTE "
                    ..
                    formatTime(
                        remaining
                    )

                if remaining <= 0 then

                    timeExpired()

                    previousPosition =
                        position

                    return
                end
            else

                hudTimer.Text =
                    "TEMPO  "
                    ..
                    formatTime(
                        raceElapsed
                    )
            end
        end

        local maxLaps =
            loopEnabled
            and
            totalLaps
            or
            1

        --====================================================
        -- PARADO
        --====================================================

        if not raceActive then

            if countdownRunning then

                hudStatus.Text =
                    "Preparando largada..."

            elseif raceFinished then

                hudStatus.Text =
                    "Corrida finalizada"

            else

                hudStatus.Text =
                    "Aguardando largada"
            end

            previousPosition =
                position

            return
        end

        --====================================================
        -- HUD
        --====================================================

        if
            currentCheckpoint
            <=
            #checkpoints
        then

            hudStatus.Text =
                string.format(
                    "VOLTA %d/%d • CHECKPOINT %d/%d",
                    currentLap,
                    maxLaps,
                    currentCheckpoint,
                    #checkpoints
                )
        else

            hudStatus.Text =
                string.format(
                    "VOLTA %d/%d • CHEGADA 🏁",
                    currentLap,
                    maxLaps
                )
        end

        if not previousPosition then

            previousPosition =
                position

            return
        end

        --====================================================
        -- CHECKPOINT
        --====================================================

        local checkpoint =
            checkpoints[
                currentCheckpoint
            ]

        if
            checkpoint
            and
            checkpoint.part
        then

            if
                segmentHitsPart(
                    checkpoint.part,
                    previousPosition,
                    position
                )
            then

                takeCheckpoint(
                    checkpoint
                )
            end
        end

        --====================================================
        -- CHEGADA
        --====================================================

        if
            finishLine
            and
            finishLine.part
        then

            local inside =
                pointInsidePart(
                    finishLine.part,
                    position
                )

            if not inside then
                finishArmed = true
            end

            local crossed =
                segmentHitsPart(
                    finishLine.part,
                    previousPosition,
                    position
                )

            if
                crossed
                and
                finishArmed
            then

                finishArmed = false

                if
                    #checkpoints > 0
                    and
                    currentCheckpoint
                    >
                    #checkpoints
                then

                    completeLap()
                end
            end
        end

        previousPosition =
            position
    end)

table.insert(
    connections,
    heartbeat
)

--============================================================
-- RESPAWN
--============================================================

local characterConnection =
    player.CharacterAdded:
    Connect(function()

        releaseStart()

        raceActive = false
        raceFinished = false
        countdownRunning = false
        timerRunning = false

        previousPosition = nil

        raceElapsed = 0

        countdownFrame.Visible =
            false

        finishScreen.Visible =
            false

        hudTimer.Text =
            "TEMPO  00:00.00"

        task.wait(1)

        resetCheckpoints()
    end)

table.insert(
    connections,
    characterConnection
)

--============================================================
-- CLEANUP
--============================================================

getgenv().DRIVZX_RACE_CLEANUP =
function()

    raceActive = false
    timerRunning = false
    countdownRunning = false

    pcall(
        releaseStart
    )

    for _,connection
        in ipairs(connections)
    do

        pcall(function()
            connection:Disconnect()
        end)
    end

    pcall(function()

        countdownSound:Stop()
        countdownSound:Destroy()
    end)

    pcall(function()
        gui:Destroy()
    end)

    pcall(function()
        raceFolder:Destroy()
    end)
end

print(
    "[DRIVZX] Race Builder V8 carregado!"
)
