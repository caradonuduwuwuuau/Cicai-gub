---

Código — CicaiServer (ServerScriptService)

-- CicaiServer.lua
-- Colocar en ServerScriptService
-- Maneja teleports, god-mode server-side, spawn seguro y limpieza anti-lag.

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local ServerStorage = game:GetService("ServerStorage")
local Players = game:GetService("Players")
local CollectionService = game:GetService("CollectionService")

local CICAI_EVENT = ReplicatedStorage:WaitForChild("CicaiEvent")

-- Config
local SPAWN_TAG = "CicaiSpawned"
local MAX_SPAWNS = 150
local CLEANUP_INTERVAL = 60 -- segundos
local savedLocations = {} -- savedLocations[userId] = {tp1 = CFrame, tp2 = CFrame}
local godPlayers = {}     -- godPlayers[userId] = true

-- Util: teleport server-side (seguro)
local function teleportPlayerToCFrame(player, cframe)
    if not player or not player.Character then return end
    local hrp = player.Character:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    pcall(function()
        player.Character:SetPrimaryPartCFrame(cframe + Vector3.new(0,3,0))
    end)
end

-- Spawn seguro: clona modelos de ServerStorage.CicaiSpawnables si existen, si no crea un Part
local function spawnAdminObject(player, modelName)
    local pos = Vector3.new(0,5,0)
    if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
        pos = player.Character.HumanoidRootPart.Position + Vector3.new(0,6,-8)
    end

    -- intentar clonar desde ServerStorage.CicaiSpawnables
    local folder = ServerStorage:FindFirstChild("CicaiSpawnables")
    if folder and modelName and folder:FindFirstChild(modelName) then
        local toClone = folder[modelName]:Clone()
        toClone:SetPrimaryPartCFrame(CFrame.new(pos))
        toClone.Parent = workspace
        CollectionService:AddTag(toClone, SPAWN_TAG)
        toClone:SetAttribute("SpawnedBy", player.UserId)
        return toClone
    end

    -- si no hay modelo, crear Part simple
    local p = Instance.new("Part")
    p.Name = "CicaiPart_"..tostring(os.time())
    p.Size = Vector3.new(4,1,4)
    p.Position = pos
    p.Anchored = false
    p.Parent = workspace
    CollectionService:AddTag(p, SPAWN_TAG)
    p:SetAttribute("SpawnedBy", player.UserId)
    return p
end

-- Mantener god-mode server-side
local function onCharacterAdded(player, char)
    local userId = player.UserId
    local humanoid = char:WaitForChild("Humanoid", 5)
    if not humanoid then return end

    local conn
    conn = humanoid.HealthChanged:Connect(function(health)
        if godPlayers[userId] then
            if humanoid and humanoid.Parent then
                humanoid.Health = humanoid.MaxHealth
            end
        end
    end)

    humanoid.Died:Connect(function()
        if godPlayers[userId] then
            wait(0.2)
            if player and player.Character == nil then
                player:LoadCharacter()
            end
        end
    end)

    char.AncestryChanged:Connect(function(_, parent)
        if not parent and conn then
            conn:Disconnect()
            conn = nil
        end
    end)
end

Players.PlayerAdded:Connect(function(p)
    savedLocations[p.UserId] = {tp1 = nil, tp2 = nil}
    p.CharacterAdded:Connect(function(char) onCharacterAdded(p, char) end)
end)

Players.PlayerRemoving:Connect(function(p)
    savedLocations[p.UserId] = nil
    godPlayers[p.UserId] = nil
end)

-- Limpieza anti-lag periódica
spawn(function()
    while true do
        wait(CLEANUP_INTERVAL)
        local tagged = CollectionService:GetTagged(SPAWN_TAG)
        if #tagged > MAX_SPAWNS then
            local toRemove = #tagged - math.floor(MAX_SPAWNS * 0.6)
            for i = 1, toRemove do
                local obj = tagged[i]
                if obj and obj.Parent then
                    pcall(function() obj:Destroy() end)
                end
            end
        end
    end
end)

-- Manejo de eventos desde cliente
CICAI_EVENT.OnServerEvent:Connect(function(player, data)
    if type(data) ~= "table" or not data.action then return end

    if data.action == "SaveTP" then
        local slot = data.slot -- "tp1" o "tp2"
        if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
            savedLocations[player.UserId] = savedLocations[player.UserId] or {}
            savedLocations[player.UserId][slot] = player.Character.HumanoidRootPart.CFrame
            CICAI_EVENT:FireClient(player, {action = "Notify", text = ("Guardado %s"):format(slot)})
        end

    elseif data.action == "GoTP" then
        local slot = data.slot
        local saved = savedLocations[player.UserId] and savedLocations[player.UserId][slot]
        if saved then
            teleportPlayerToCFrame(player, saved)
            CICAI_EVENT:FireClient(player, {action = "Notify", text = ("Teletransportado a %s"):format(slot)})
        else
            CICAI_EVENT:FireClient(player, {action = "Notify", text = ("No hay ubicación guardada en %s"):format(slot)})
        end

    elseif data.action == "ToggleGod" then
        local on = data.on
        if on then
            godPlayers[player.UserId] = true
            if player.Character and player.Character:FindFirstChildOfClass("Humanoid") then
                player.Character:FindFirstChildOfClass("Humanoid").Health = player.Character:FindFirstChildOfClass("Humanoid").MaxHealth
            end
            CICAI_EVENT:FireClient(player, {action = "Notify", text = "God Mode ON"})
        else
            godPlayers[player.UserId] = nil
            CICAI_EVENT:FireClient(player, {action = "Notify", text = "God Mode OFF"})
        end

    elseif data.action == "Spawn" then
        local modelName = data.modelName
        spawnAdminObject(player, modelName)
        CICAI_EVENT:FireClient(player, {action = "Notify", text = "Objeto creado."})

    elseif data.action == "RequestSavedLocations" then
        local s = savedLocations[player.UserId] or {}
        CICAI_EVENT:FireClient(player, {action = "SavedLocations", data = s})
    end
end)


---

Código — CicaiClient (StarterPlayerScripts / LocalScript)

-- CicaiClient.lua
-- Colocar en StarterPlayerScripts (LocalScript)
-- Crea GUI automáticamente, maneja teclas, vuelo cliente-side, hitbox y envía comandos al server.

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local ServerStorage = game:GetService("ServerStorage")

local LOCAL_PLAYER = Players.LocalPlayer
local CICAI_EVENT = ReplicatedStorage:WaitForChild("CicaiEvent")

-- Keybinds (puedes editar)
local KEY_TOGGLE_FLY = Enum.KeyCode.F
local KEY_TOGGLE_GOD = Enum.KeyCode.G
local KEY_SAVE_TP1 = Enum.KeyCode.Z
local KEY_GO_TP1 = Enum.KeyCode.X
local KEY_SAVE_TP2 = Enum.KeyCode.C
local KEY_GO_TP2 = Enum.KeyCode.V
local KEY_SPAWN_MENU = Enum.KeyCode.P
local KEY_HITBOX = Enum.KeyCode.H
local KEY_TOGGLE_HUB = Enum.KeyCode.H -- la misma H puede abrir/ocultar GUI

-- Estados
local flying = false
local god = false
local showHitbox = false
local flightForce = nil
local speedFly = 80

-- Crear GUI simple (botones y notificaciones)
local playerGui = LOCAL_PLAYER:WaitForChild("PlayerGui")
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "CicaiHubGui"
screenGui.ResetOnSpawn = false
screenGui.Parent = playerGui

-- Panel
local frame = Instance.new("Frame")
frame.Size = UDim2.new(0, 360, 0, 240)
frame.Position = UDim2.new(0.02, 0, 0.12, 0)
frame.BackgroundTransparency = 0.25
frame.BackgroundColor3 = Color3.fromRGB(20,20,20)
frame.BorderSizePixel = 0
frame.Parent = screenGui

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, 0, 0, 28)
title.Position = UDim2.new(0,0,0,0)
title.BackgroundTransparency = 1
title.Text = "Cicai Hub"
title.Font = Enum.Font.GothamBold
title.TextSize = 18
title.TextColor3 = Color3.new(1,1,1)
title.Parent = frame

-- Notifier
local notif = Instance.new("TextLabel")
notif.Size = UDim2.new(1, 0, 0, 26)
notif.Position = UDim2.new(0,0,1, -26)
notif.BackgroundTransparency = 0.4
notif.Text = ""
notif.TextScaled = true
notif.Visible = false
notif.Parent = frame

local function notify(text, t)
    notif.Text = text
    notif.Visible = true
    delay(t or 2, function() notif.Visible = false end)
end

-- Buttons helper
local function makeButton(name, posY, callback)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(0, 160, 0, 36)
    b.Position = UDim2.new(0, 12, 0, posY)
    b.Text = name
    b.Font = Enum.Font.Gotham
    b.TextSize = 14
    b.Parent = frame
    b.MouseButton1Click:Connect(callback)
    return b
end

-- Botones
local btnFly = makeButton("Toggle Fly (F)", 36, function()
    if flying then
        flying = false
        if flightForce then
            if flightForce.bv and flightForce.bv.Parent then flightForce.bv:Destroy() end
            if flightForce.bg and flightForce.bg.Parent then flightForce.bg:Destroy() end
            flightForce = nil
        end
        notify("Vuelo OFF")
    else
        flying = true
        local char = LOCAL_PLAYER.Character
        if char and char:FindFirstChild("HumanoidRootPart") then
            local hrp = char.HumanoidRootPart
            local bv = Instance.new("BodyVelocity")
            bv.MaxForce = Vector3.new(1e5,1e5,1e5)
            bv.Velocity = Vector3.new(0,0,0)
            bv.Parent = hrp
            local bg = Instance.new("BodyGyro")
            bg.MaxTorque = Vector3.new(4e5,4e5,4e5)
            bg.CFrame = hrp.CFrame
            bg.Parent = hrp
            flightForce = {bv = bv, bg = bg}
            notify("Vuelo ON")
        end
    end
end)

local btnGod = makeButton("Toggle God (G)", 36, function()
    god = not god
    CICAI_EVENT:FireServer({action = "ToggleGod", on = god})
    notify("God: "..tostring(god))
end)

local btnSaveTP1 = makeButton("Guardar TP1 (Z)", 80, function()
    CICAI_EVENT:FireServer({action = "SaveTP", slot = "tp1"})
end)
local btnGoTP1 = makeButton("Ir TP1 (X)", 80, function()
    CICAI_EVENT:FireServer({action = "GoTP", slot = "tp1"})
end)
local btnSaveTP2 = makeButton("Guardar TP2 (C)", 124, function()
    CICAI_EVENT:FireServer({action = "SaveTP", slot = "tp2"})
end)
local btnGoTP2 = makeButton("Ir TP2 (V)", 124, function()
    CICAI_EVENT:FireServer({action = "GoTP", slot = "tp2"})
end)

local btnSpawn = makeButton("Spawner (P) - abrir", 168, function()
    -- abrir menu simple de spawn basado en ServerStorage.CicaiSpawnables
    local folder = ServerStorage:FindFirstChild("CicaiSpawnables")
    if not folder then
        -- si no hay folder, mandar comando para crear Part
        CICAI_EVENT:FireServer({action = "Spawn", modelName = nil})
        notify("Spawn: Part creado")
        return
    end

    -- crear menu temporal con modelos
    local menu = Instance.new("Frame")
    menu.Size = UDim2.new(0, 180, 0, 160)
    menu.Position = UDim2.new(0, 190, 0, 36)
    menu.BackgroundTransparency = 0.25
    menu.Parent = frame

    local y = 6
    for _, m in ipairs(folder:GetChildren()) do
        local nm = m.Name
        local b = Instance.new("TextButton")
        b.Size = UDim2.new(0, 160, 0, 28)
        b.Position = UDim2.new(0, 10, 0, y)
        b.Text = nm
        b.Font = Enum.Font.Gotham
        b.TextSize = 12
        b.Parent = menu
        b.MouseButton1Click:Connect(function()
            CICAI_EVENT:FireServer({action = "Spawn", modelName = nm})
            menu:Destroy()
            notify("Spawn pedido: "..nm)
        end)
        y = y + 34
    end

    -- botón cerrar
    local close = Instance.new("TextButton")
    close.Size = UDim2.new(0, 80, 0, 28)
    close.Position = UDim2.new(0, 90, 1, -34)
    close.Text = "Cerrar"
    close.Parent = menu
    close.MouseButton1Click:Connect(function() menu:Destroy() end)
end)

local btnHitbox = makeButton("Toggle Hitbox (H)", 168, function()
    showHitbox = not showHitbox
    local char = LOCAL_PLAYER.Character
    if showHitbox and char and char:FindFirstChild("HumanoidRootPart") then
        local hrp = char.HumanoidRootPart
        local adorn = Instance.new("BoxHandleAdornment")
        adorn.Name = "CicaiHitbox"
        adorn.AlwaysOnTop = true
        adorn.Adornee = hrp
        adorn.Size = Vector3.new(2,3,1)
        adorn.Parent = hrp
        notify("Hitbox ON")
    else
        if char and char:FindFirstChild("HumanoidRootPart") then
            local hrp = char.HumanoidRootPart
            local existing = hrp:FindFirstChild("CicaiHitbox")
            if existing then existing:Destroy() end
        end
        notify("Hitbox OFF")
    end
end)

-- Notificaciones desde el servidor
CICAI_EVENT.OnClientEvent:Connect(function(data)
    if not data or type(data) ~= "table" or not data.action then return end
    if data.action == "Notify" then
        notify(tostring(data.text or ""), 2.5)
    elseif data.action == "SavedLocations" then
        -- opción: mostrar data
    end
end)

-- Input handling (teclas)
local keysHeld = {}
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.UserInputType == Enum.UserInputType.Keyboard then
        local kc = input.KeyCode
        keysHeld[kc] = true

        if kc == KEY_TOGGLE_FLY then
            btnFly:MouseButton1Click()
        elseif kc == KEY_TOGGLE_GOD then
            btnGod:MouseButton1Click()
        elseif kc == KEY_SAVE_TP1 then
            btnSaveTP1:MouseButton1Click()
        elseif kc == KEY_GO_TP1 then
            btnGoTP1:MouseButton1Click()
        elseif kc == KEY_SAVE_TP2 then
            btnSaveTP2:MouseButton1Click()
        elseif kc == KEY_GO_TP2 then
            btnGoTP2:MouseButton1Click()
        elseif kc == KEY_SPAWN_MENU then
            btnSpawn:MouseButton1Click()
        elseif kc == KEY_HITBOX then
            btnHitbox:MouseButton1Click()
        end
    end
end)
UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.Keyboard then
        keysHeld[input.KeyCode] = nil
    end
end)

-- Flight loop (RenderStepped)
RunService.RenderStepped:Connect(function(dt)
    if flying and flightForce and LOCAL_PLAYER.Character and LOCAL_PLAYER.Character:FindFirstChild("HumanoidRootPart") then
        local hrp = LOCAL_PLAYER.Character.HumanoidRootPart
        local cam = workspace.CurrentCamera
        local forward = cam.CFrame.LookVector
        local right = cam.CFrame.RightVector
        local up = Vector3.new(0,1,0)

        local moveVec = Vector3.new(0,0,0)
        if keysHeld[Enum.KeyCode.W] then moveVec = moveVec + Vector3.new(forward.X, 0, forward.Z) end
        if keysHeld[Enum.KeyCode.S] then moveVec = moveVec - Vector3.new(forward.X, 0, forward.Z) end
        if keysHeld[Enum.KeyCode.A] then moveVec = moveVec - Vector3.new(right.X, 0, right.Z) end
        if keysHeld[Enum.KeyCode.D] then moveVec = moveVec + Vector3.new(right.X, 0, right.Z) end
        if keysHeld[Enum.KeyCode.Space] then moveVec = moveVec + up end
        if keysHeld[Enum.KeyCode.LeftShift] then moveVec = moveVec - up end

        if moveVec.Magnitude > 0 then moveVec = moveVec.Unit * speedFly end
        flightForce.bv.Velocity = moveVec
        flightForce.bg.CFrame = CFrame.new(hrp.Position, hrp.Position + cam.CFrame.LookVector)
    end
end)

-- Mantener estado al respawnear
LOCAL_PLAYER.CharacterAdded:Connect(function(char)
    -- quitar adornos viejos
    local hrp = char:WaitForChild("HumanoidRootPart", 5)
    if hrp then
        local e = hrp:FindFirstChild("CicaiHitbox")
        if e then e:Destroy() end
    end
    -- re-aplicar god si estaba activo
    if god then
        CICAI_EVENT:FireServer({action = "ToggleGod", on = true})
    end
end)

-- Por defecto, mostrar GUI al entrar
notify("Cicai Hub activo - usa botones o atajos (F,G,Z/X,C/V,P,H)", 4)


---# Cicai-gub
Script
