-- Rayfield UI (Aimbot, Silent Aim, ESP, Misc 탭 완전 분리)
local Rayfield = loadstring(game:HttpGet("https://sirius.menu/rayfield"))()
print("[Rayfield] Loaded")

local Window = Rayfield:CreateWindow({
    Name = "MoonHook Rivals",
    LoadingTitle = "MoonHook",
    LoadingSubtitle = "by eszkeredzon",
    ConfigurationSaving = { Enabled = true, FolderName = "MoonHook", FileName = "Settings" },
    Discord = { Enabled = false },
    KeySystem = false
})

-- ========== 탭 생성 (4개) ==========
local AimbotTab = Window:CreateTab("Aimbot", 0)       -- 일반 에임봇
local SilentTab = Window:CreateTab("Silent Aim", 1)   -- 사일런트 에임봇
local EspTab = Window:CreateTab("ESP", 2)
local MiscTab = Window:CreateTab("Misc", 3)

-- ========== 서비스 및 변수 ==========
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInput = game:GetService("UserInputService")
local Camera = workspace.CurrentCamera
local LocalPlayer = Players.LocalPlayer
local ReplicatedStorage = game:GetService("ReplicatedStorage")

-- ========== 팀 체크 (적만 true) ==========
local function isEnemy(targetPlayer)
    if not targetPlayer or targetPlayer == LocalPlayer then return false end
    local myTeamID = LocalPlayer:GetAttribute("TeamID")
    local theirTeamID = targetPlayer:GetAttribute("TeamID")
    if myTeamID and theirTeamID then
        return myTeamID ~= theirTeamID
    end
    local myEnvID = LocalPlayer:GetAttribute("EnvironmentID")
    local theirEnvID = targetPlayer:GetAttribute("EnvironmentID")
    if myEnvID and theirEnvID then
        return myEnvID ~= theirEnvID
    end
    if LocalPlayer.Team and targetPlayer.Team then
        return LocalPlayer.Team ~= targetPlayer.Team
    end
    return true
end

-- ========== 일반 에임봇 설정 ==========
local NormalAimbot = {
    Enabled = false,
    TargetPart = "Head",
    Radius = 200,
    Smoothness = 50,
    WallCheck = true,
    TeamCheck = true
}

-- ========== 사일런트 에임봇 설정 ==========
local SilentAimbot = {
    Enabled = false,
    Chance = 100,          -- %
    TargetPart = "Head",
    Radius = 200,
    WallCheck = true,
    TeamCheck = true
}

-- ========== ESP 설정 ==========
local ESP = {
    Highlight = { Enabled = false, Color = Color3.fromRGB(255, 255, 0) },
    Box = { Enabled = false, Color = Color3.fromRGB(255, 0, 0) },
    Name = { Enabled = false, Color = Color3.fromRGB(255, 255, 255) },
    Health = { Enabled = false, Color = Color3.fromRGB(0, 255, 0) },
    Distance = { Enabled = false, Color = Color3.fromRGB(255, 255, 255) },
    Skeleton = { Enabled = false, Color = Color3.fromRGB(255, 255, 0) },
    ShowFOV = false,
    FOVColor = Color3.fromRGB(255, 255, 255),
    MaxDistance = 300
}

local ESPstore = {}
local FOVCircle = Drawing.new("Circle")
FOVCircle.Visible = false
FOVCircle.Filled = false
FOVCircle.Thickness = 2
FOVCircle.Color = ESP.FOVColor

-- ========== 일반 에임봇 함수 ==========
local function getClosestTargetForNormal()
    local bestTarget, bestDist = nil, NormalAimbot.Radius
    local mousePos = UserInput:GetMouseLocation()
    local localChar = LocalPlayer.Character
    local localRoot = localChar and localChar:FindFirstChild("HumanoidRootPart")
    if not localRoot then return nil end

    for _, targetPlayer in pairs(Players:GetPlayers()) do
        if targetPlayer == LocalPlayer then continue end
        if NormalAimbot.TeamCheck and not isEnemy(targetPlayer) then continue end
        
        local char = targetPlayer.Character
        if not char then continue end
        local targetPart = char:FindFirstChild(NormalAimbot.TargetPart)
        if not targetPart then continue end
        local hum = char:FindFirstChildOfClass("Humanoid")
        if not hum or hum.Health <= 0 then continue end

        local screenPos, onScreen = Camera:WorldToViewportPoint(targetPart.Position)
        if not onScreen then continue end

        local distToMouse = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
        if distToMouse < bestDist then
            local visible = true
            if NormalAimbot.WallCheck then
                local params = RaycastParams.new()
                params.FilterType = Enum.RaycastFilterType.Exclude
                params.FilterDescendantsInstances = {localChar, char}
                local ray = workspace:Raycast(localRoot.Position, targetPart.Position - localRoot.Position, params)
                if ray and ray.Instance ~= targetPart and not ray.Instance:IsDescendantOf(char) then
                    visible = false
                end
            end
            if visible then
                bestTarget = targetPlayer
                bestDist = distToMouse
            end
        end
    end
    return bestTarget
end

local function aimAt(part)
    local pos, onScreen = Camera:WorldToViewportPoint(part.Position)
    if onScreen then
        local mouse = UserInput:GetMouseLocation()
        local smoothFactor = NormalAimbot.Smoothness / 100
        local deltaX = (pos.X - mouse.X) * smoothFactor
        local deltaY = (pos.Y - mouse.Y) * smoothFactor
        mousemoverel(deltaX, deltaY)
    end
end

-- ========== 사일런트 에임봇 함수 (후킹 + 확률) ==========
local Utility = require(ReplicatedStorage.Modules.Utility)
local OldRaycast = Utility.Raycast
local SilentTargetPart = nil

local function getClosestTargetForSilent()
    local bestPos, bestDist = nil, SilentAimbot.Radius
    local localChar = LocalPlayer.Character
    local localRoot = localChar and localChar:FindFirstChild("HumanoidRootPart")
    if not localRoot then return nil end
    
    local center = Camera.ViewportSize / 2

    for _, targetPlayer in pairs(Players:GetPlayers()) do
        if targetPlayer == LocalPlayer then continue end
        if SilentAimbot.TeamCheck and not isEnemy(targetPlayer) then continue end
        
        local char = targetPlayer.Character
        if not char then continue end
        local targetPart = char:FindFirstChild(SilentAimbot.TargetPart)
        if not targetPart then continue end
        local hum = char:FindFirstChildOfClass("Humanoid")
        if not hum or hum.Health <= 0 then continue end

        local screenPos, onScreen = Camera:WorldToViewportPoint(targetPart.Position)
        if not onScreen then continue end

        local distToCenter = (Vector2.new(screenPos.X, screenPos.Y) - center).Magnitude
        if distToCenter < bestDist then
            local visible = true
            if SilentAimbot.WallCheck then
                local params = RaycastParams.new()
                params.FilterType = Enum.RaycastFilterType.Exclude
                params.FilterDescendantsInstances = {localChar, char}
                local ray = workspace:Raycast(localRoot.Position, targetPart.Position - localRoot.Position, params)
                if ray and ray.Instance ~= targetPart and not ray.Instance:IsDescendantOf(char) then
                    visible = false
                end
            end
            if visible then
                bestPos = targetPart.Position
                bestDist = distToCenter
            end
        end
    end
    return bestPos
end

local function hookedRaycast(...)
    local args = {...}
    if SilentAimbot.Enabled and args[4] == 999 then
        if math.random(1, 100) <= SilentAimbot.Chance then
            local aimPos = SilentTargetPart or getClosestTargetForSilent()
            if aimPos and typeof(aimPos) == "Vector3" then
                args[3] = aimPos
            end
        end
    end
    return OldRaycast(table.unpack(args))
end

-- 항상 후킹 적용 (조건은 함수 내에서 체크)
Utility.Raycast = hookedRaycast

-- ========== ESP 함수 (거리 제한) ==========
local function getBonePairs()
    return {
        {"Head", "UpperTorso"},
        {"UpperTorso", "LowerTorso"},
        {"UpperTorso", "LeftUpperArm"}, {"LeftUpperArm", "LeftLowerArm"}, {"LeftLowerArm", "LeftHand"},
        {"UpperTorso", "RightUpperArm"}, {"RightUpperArm", "RightLowerArm"}, {"RightLowerArm", "RightHand"},
        {"LowerTorso", "LeftUpperLeg"}, {"LeftUpperLeg", "LeftLowerLeg"}, {"LeftLowerLeg", "LeftFoot"},
        {"LowerTorso", "RightUpperLeg"}, {"RightUpperLeg", "RightLowerLeg"}, {"RightLowerLeg", "RightFoot"}
    }
end

local function createESP(targetPlayer)
    if targetPlayer == LocalPlayer then return end
    if ESPstore[targetPlayer] then return end
    if NormalAimbot.TeamCheck and not isEnemy(targetPlayer) then return end
    
    local char = targetPlayer.Character
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart")
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not root or not hum or hum.Health <= 0 then return end

    local hl = Instance.new("Highlight", char)
    hl.Enabled = false

    local box = Drawing.new("Square")
    box.Visible = false; box.Filled = false; box.Thickness = 2

    local nameText = Drawing.new("Text")
    nameText.Visible = false; nameText.Center = true; nameText.Outline = true

    local healthLine = Drawing.new("Line")
    healthLine.Visible = false; healthLine.Thickness = 4

    local distText = Drawing.new("Text")
    distText.Visible = false; distText.Center = true; distText.Outline = true

    local skeletonLines = {}
    for _, bone in ipairs(getBonePairs()) do
        local line = Drawing.new("Line")
        line.Visible = false; line.Thickness = 2
        table.insert(skeletonLines, {line = line, a = bone[1], b = bone[2]})
    end

    ESPstore[targetPlayer] = {root=root, hum=hum, hl=hl, box=box, nameText=nameText, healthLine=healthLine, distText=distText, skeleton=skeletonLines}
end

local function removeESP(targetPlayer)
    local e = ESPstore[targetPlayer]
    if not e then return end
    e.hl:Destroy()
    e.box:Remove()
    e.nameText:Remove()
    e.healthLine:Remove()
    e.distText:Remove()
    for _, v in ipairs(e.skeleton) do v.line:Remove() end
    ESPstore[targetPlayer] = nil
end

local function updateESP()
    for targetPlayer, e in pairs(ESPstore) do
        if NormalAimbot.TeamCheck and not isEnemy(targetPlayer) then
            removeESP(targetPlayer)
            continue
        end
        if not targetPlayer or not targetPlayer.Character or not e.root.Parent then
            removeESP(targetPlayer)
            continue
        end
        local hum = targetPlayer.Character:FindFirstChildOfClass("Humanoid")
        if hum and hum.Health <= 0 then
            removeESP(targetPlayer)
            continue
        end

        local depth = (e.root.Position - Camera.CFrame.Position).Magnitude
        if depth > ESP.MaxDistance then
            e.hl.Enabled = false
            e.box.Visible = false
            e.nameText.Visible = false
            e.healthLine.Visible = false
            e.distText.Visible = false
            for _, boneLine in ipairs(e.skeleton) do boneLine.line.Visible = false end
            continue
        end

        local pos, vis = Camera:WorldToViewportPoint(e.root.Position)
        local size = depth > 0 and (2000/depth) or 0

        e.hl.Enabled = ESP.Highlight.Enabled
        e.hl.FillColor = ESP.Highlight.Color
        e.hl.OutlineColor = ESP.Highlight.Color

        if ESP.Box.Enabled and vis then
            e.box.Position = Vector2.new(pos.X - size/2, pos.Y - size/2)
            e.box.Size = Vector2.new(size, size)
            e.box.Color = ESP.Box.Color
            e.box.Visible = true
        else
            e.box.Visible = false
        end

        if ESP.Name.Enabled and vis then
            e.nameText.Text = targetPlayer.Name
            e.nameText.Position = Vector2.new(pos.X, pos.Y - size/2 - 20)
            e.nameText.Color = ESP.Name.Color
            e.nameText.Size = 18
            e.nameText.Visible = true
        else
            e.nameText.Visible = false
        end

        if ESP.Health.Enabled and vis then
            local percent = math.clamp(e.hum.Health / e.hum.MaxHealth, 0, 1)
            e.healthLine.From = Vector2.new(pos.X + size/2 + 6, pos.Y + size/2)
            e.healthLine.To = Vector2.new(pos.X + size/2 + 6, pos.Y + size/2 - size*percent)
            e.healthLine.Color = ESP.Health.Color
            e.healthLine.Visible = true
        else
            e.healthLine.Visible = false
        end

        if ESP.Distance.Enabled and vis then
            e.distText.Text = math.floor(depth) .. "m"
            e.distText.Position = Vector2.new(pos.X, pos.Y + size/2 + 6)
            e.distText.Color = ESP.Distance.Color
            e.distText.Size = 16
            e.distText.Visible = true
        else
            e.distText.Visible = false
        end

        if ESP.Skeleton.Enabled and vis then
            local char = targetPlayer.Character
            if char then
                for _, boneLine in ipairs(e.skeleton) do
                    local partA = char:FindFirstChild(boneLine.a)
                    local partB = char:FindFirstChild(boneLine.b)
                    if partA and partB then
                        local posA, _ = Camera:WorldToViewportPoint(partA.Position)
                        local posB, _ = Camera:WorldToViewportPoint(partB.Position)
                        boneLine.line.From = Vector2.new(posA.X, posA.Y)
                        boneLine.line.To = Vector2.new(posB.X, posB.Y)
                        boneLine.line.Color = ESP.Skeleton.Color
                        boneLine.line.Visible = true
                    else
                        boneLine.line.Visible = false
                    end
                end
            end
        else
            for _, boneLine in ipairs(e.skeleton) do boneLine.line.Visible = false end
        end
    end

    if ESP.ShowFOV then
        local mx, my = UserInput:GetMouseLocation().X, UserInput:GetMouseLocation().Y
        FOVCircle.Position = Vector2.new(mx, my)
        FOVCircle.Radius = NormalAimbot.Radius   -- FOV 원은 일반 에임봇과 공유 (시각적)
        FOVCircle.Color = ESP.FOVColor
        FOVCircle.Visible = true
    else
        FOVCircle.Visible = false
    end
end

-- ========== UI: Aimbot 탭 ==========
AimbotTab:CreateToggle({
    Name = "Enable Normal Aimbot",
    CurrentValue = false,
    Callback = function(v) NormalAimbot.Enabled = v end
})

AimbotTab:CreateDropdown({
    Name = "Target Part",
    Options = {"Head", "Torso", "HumanoidRootPart"},
    CurrentOption = "Head",
    Callback = function(v) NormalAimbot.TargetPart = v end
})

AimbotTab:CreateSlider({
    Name = "FOV Radius",
    Range = {50, 500},
    Increment = 10,
    CurrentValue = 200,
    Callback = function(v) NormalAimbot.Radius = v end
})

AimbotTab:CreateSlider({
    Name = "Smoothness %",
    Range = {1, 100},
    Increment = 1,
    CurrentValue = 50,
    Callback = function(v) NormalAimbot.Smoothness = v end
})

AimbotTab:CreateToggle({
    Name = "Wall Check (Visibility)",
    CurrentValue = true,
    Callback = function(v) NormalAimbot.WallCheck = v end
})

AimbotTab:CreateToggle({
    Name = "Team Check (Only Enemies)",
    CurrentValue = true,
    Callback = function(v)
        NormalAimbot.TeamCheck = v
        -- ESP에도 팀 체크 적용을 위해 재생성
        for pl,_ in pairs(ESPstore) do removeESP(pl) end
        for _, pl in pairs(Players:GetPlayers()) do createESP(pl) end
    end
})

-- ========== UI: Silent Aim 탭 ==========
SilentTab:CreateToggle({
    Name = "Enable Silent Aimbot",
    CurrentValue = false,
    Callback = function(v) SilentAimbot.Enabled = v end
})

SilentTab:CreateSlider({
    Name = "Hit Chance (%)",
    Range = {1, 100},
    Increment = 1,
    CurrentValue = 100,
    Callback = function(v) SilentAimbot.Chance = v end
})

SilentTab:CreateDropdown({
    Name = "Target Part",
    Options = {"Head", "Torso", "HumanoidRootPart"},
    CurrentOption = "Head",
    Callback = function(v) SilentAimbot.TargetPart = v end
})

SilentTab:CreateSlider({
    Name = "FOV Radius",
    Range = {50, 500},
    Increment = 10,
    CurrentValue = 200,
    Callback = function(v) SilentAimbot.Radius = v end
})

SilentTab:CreateToggle({
    Name = "Wall Check (Visibility)",
    CurrentValue = true,
    Callback = function(v) SilentAimbot.WallCheck = v end
})

SilentTab:CreateToggle({
    Name = "Team Check (Only Enemies)",
    CurrentValue = true,
    Callback = function(v)
        SilentAimbot.TeamCheck = v
    end
})

-- ========== UI: ESP 탭 ==========
EspTab:CreateToggle({
    Name = "Highlight ESP",
    CurrentValue = false,
    Callback = function(v) ESP.Highlight.Enabled = v end
})
EspTab:CreateColorPicker({
    Name = "Highlight Color",
    Color = ESP.Highlight.Color,
    Callback = function(c) ESP.Highlight.Color = c end
})

EspTab:CreateToggle({
    Name = "Box ESP",
    CurrentValue = false,
    Callback = function(v) ESP.Box.Enabled = v end
})
EspTab:CreateColorPicker({
    Name = "Box Color",
    Color = ESP.Box.Color,
    Callback = function(c) ESP.Box.Color = c end
})

EspTab:CreateToggle({
    Name = "Name ESP",
    CurrentValue = false,
    Callback = function(v) ESP.Name.Enabled = v end
})
EspTab:CreateColorPicker({
    Name = "Name Color",
    Color = ESP.Name.Color,
    Callback = function(c) ESP.Name.Color = c end
})

EspTab:CreateToggle({
    Name = "Health Bar",
    CurrentValue = false,
    Callback = function(v) ESP.Health.Enabled = v end
})
EspTab:CreateColorPicker({
    Name = "Health Color",
    Color = ESP.Health.Color,
    Callback = function(c) ESP.Health.Color = c end
})

EspTab:CreateToggle({
    Name = "Distance ESP",
    CurrentValue = false,
    Callback = function(v) ESP.Distance.Enabled = v end
})
EspTab:CreateColorPicker({
    Name = "Distance Color",
    Color = ESP.Distance.Color,
    Callback = function(c) ESP.Distance.Color = c end
})

EspTab:CreateToggle({
    Name = "Skeleton ESP",
    CurrentValue = false,
    Callback = function(v) ESP.Skeleton.Enabled = v end
})
EspTab:CreateColorPicker({
    Name = "Skeleton Color",
    Color = ESP.Skeleton.Color,
    Callback = function(c) ESP.Skeleton.Color = c end
})

EspTab:CreateToggle({
    Name = "Show FOV Circle",
    CurrentValue = false,
    Callback = function(v) ESP.ShowFOV = v end
})
EspTab:CreateColorPicker({
    Name = "FOV Color",
    Color = ESP.FOVColor,
    Callback = function(c) ESP.FOVColor = c; FOVCircle.Color = c end
})

EspTab:CreateSlider({
    Name = "Max ESP Distance (studs)",
    Range = {100, 800},
    Increment = 10,
    CurrentValue = 300,
    Callback = function(v) ESP.MaxDistance = v end
})

-- ========== UI: Misc 탭 (Unlock All) ==========
MiscTab:CreateButton({
    Name = "Unlock All (Cosmetics & Weapons)",
    Callback = function()
        local unlockScript = [[
-- Unlock All Script (provided by user)
local plrs = game:GetService("Players")
local reps = game:GetService("ReplicatedStorage")
local http = game:GetService("HttpService")

local lp = plrs.LocalPlayer
local lps = lp.PlayerScripts
local ctrls = lps.Controllers
local rmods = reps.Modules

local elib = require(rmods:WaitForChild("EnumLibrary", 10))
if elib then elib:WaitForEnumBuilder() end
local clib = require(rmods:WaitForChild("CosmeticLibrary", 10))
local ilib = require(rmods:WaitForChild("ItemLibrary", 10))
local dctrl = require(ctrls:WaitForChild("PlayerDataController", 10))
local coss = clib.Cosmetics

local cdata = dctrl.CurrentData
if not cdata then
    task.spawn(function()
        repeat task.wait() until dctrl.CurrentData
        cdata = dctrl.CurrentData
    end)
end

local equip, favs = {}, {}
local cwep, vprof, lwep
local lpname = lp.Name
local fcache = {}

local function banned(n)
    if type(n) ~= "string" then return true end
    return n:find("MISSING_") or n:find("Bubblegum") or n:find("Ragdoll") or n:find("Fall Apart") or n:find("Every Finisher Ever")
end

local function toenum(n)
    if not elib then return nil end
    local ok, id = pcall(elib.ToEnum, elib, n)
    return ok and id or nil
end

local function clonecos(name, ctype, inv, favonly)
    if banned(name) then return nil end
    local base = coss[name]
    if not base then return nil end
    local d = table.clone(base)
    d.Name = name
    d.Type = d.Type or ctype
    d.Seed = d.Seed or math.random(1, 1000000)
    local eid = toenum(name)
    if eid then d.Enum = eid d.ObjectID = d.ObjectID or eid end
    if inv ~= nil then d.Inverted = inv end
    if favonly ~= nil then d.OnlyUseFavorites = favonly end
    return d
end

local savef = "unlockall/config.json"

local function savecfg()
    if not writefile then return end
    pcall(function()
        local cfg = { equipped = {}, favorites = favs }
        for wep, cos in equip do
            local slot = {}
            cfg.equipped[wep] = slot
            for ct, cd in cos do
                if cd and cd.Name and not banned(cd.Name) then
                    slot[ct] = { name = cd.Name, seed = cd.Seed, inverted = cd.Inverted }
                end
            end
        end
        makefolder("unlockall")
        writefile(savef, http:JSONEncode(cfg))
    end)
end

local function loadcfg()
    if not readfile or not isfile or not isfile(savef) then return end
    pcall(function()
        local cfg = http:JSONDecode(readfile(savef))
        if cfg.equipped then
            for wep, cos in cfg.equipped do
                equip[wep] = {}
                for ct, cd in cos do
                    if not banned(cd.name) then
                        local cl = clonecos(cd.name, ct, cd.inverted)
                        if cl then
                            cl.Seed = cd.seed
                            equip[wep][ct] = cl
                        end
                    end
                end
            end
        end
        favs = cfg.favorites or {}
    end)
end

local finv = {}
local function rebuildinv()
    table.clear(finv)
    for name in coss do
        if not banned(name) then finv[name] = true end
    end
    for _, cos in equip do
        for _, cd in cos do
            if cd and cd.Name and not banned(cd.Name) then finv[cd.Name] = true end
        end
    end
end

rebuildinv()

local oget = dctrl.Get
dctrl.Get = function(self, key)
    local data = oget(self, key)
    if key == "CosmeticInventory" then
        local proxy = {}
        if data then for k, v in data do if not banned(k) then proxy[k] = v end end end
        for name in finv do proxy[name] = true end
        return proxy
    end
    if key == "FavoritedCosmetics" then
        local res = data and table.clone(data) or {}
        for wep, fv in favs do
            local slot = res[wep] or {}
            res[wep] = slot
            for name, isfav in fv do
                if not banned(name) then slot[name] = isfav end
            end
        end
        return res
    end
    return data
end

local ogetwep = dctrl.GetWeaponData
dctrl.GetWeaponData = function(self, wname)
    local data = ogetwep(self, wname)
    if not data then return nil end
    local merged = table.clone(data)
    merged.Name = wname
    local weq = equip[wname]
    if weq then for ct, cd in weq do merged[ct] = cd end end
    return merged
end

local function saferep(key)
    pcall(function()
        if not cdata and dctrl.CurrentData then cdata = dctrl.CurrentData end
        if cdata then cdata:Replicate(key) end
    end)
end

local fctrl
pcall(function() fctrl = require(ctrls:WaitForChild("FighterController", 10)) end)

local function getewep()
    if not fctrl then return nil end
    local fighter = fctrl:GetFighter(lp)
    if not fighter or not fighter.Items then return nil end
    for _, item in fighter.Items do
        if item.IsEquipped then return item.Name end
    end
    return nil
end

if hookmetamethod then
    local rems = reps:FindFirstChild("Remotes")
    local drems = rems and rems:FindFirstChild("Data")
    local eqrem = drems and drems:FindFirstChild("EquipCosmetic")
    local favrem = drems and drems:FindFirstChild("FavoriteCosmetic")
    local rrems = rems and rems:FindFirstChild("Replication")
    local frems = rrems and rrems:FindFirstChild("Fighter")
    local uirem = frems and frems:FindFirstChild("UseItem")

    if eqrem then
        local oncall
        oncall = hookmetamethod(game, "__namecall", function(self, ...)
            if getnamecallmethod() ~= "FireServer" then return oncall(self, ...) end
            local args = { ... }

            if uirem and self == uirem and fctrl then
                pcall(function()
                    local fighter = fctrl:GetFighter(lp)
                    if fighter and fighter.Items then
                        local oid = args[1]
                        for _, item in fighter.Items do
                            if item:Get("ObjectID") == oid then
                                lwep = item.Name
                                break
                            end
                        end
                    end
                end)
            end

            if self == eqrem then
                local wname, ctype, cname, opts = args[1], args[2], args[3], args[4] or {}
                if not cname or cname == "None" or cname == "" then
                    equip[wname] = equip[wname] or {}
                    equip[wname][ctype] = nil
                    if not next(equip[wname]) then equip[wname] = nil end
                    rebuildinv()
                    task.defer(function()
                        saferep("WeaponInventory")
                        task.wait(0.2)
                        savecfg()
                    end)
                    return oncall(self, ...)
                end
                if banned(cname) then return oncall(self, ...) end
                local rdata = oget(dctrl, "CosmeticInventory")
                if rdata and type(rdata[cname]) ~= "boolean" and rdata[cname] ~= nil then
                    return oncall(self, ...)
                end
                equip[wname] = equip[wname] or {}
                local cloned = clonecos(cname, ctype, opts.IsInverted, opts.OnlyUseFavorites)
                if cloned then equip[wname][ctype] = cloned end
                if ctype == "Finisher" then fcache[wname] = cname end
                rebuildinv()
                task.defer(function()
                    saferep("WeaponInventory")
                    task.wait(0.2)
                    savecfg()
                end)
                return
            end

            if self == favrem then
                local fwep, fname, fstate = args[1], args[2], args[3]
                if not fname or fname == "None" or fname == "" then return oncall(self, ...) end
                if banned(fname) then return oncall(self, ...) end
                favs[fwep] = favs[fwep] or {}
                favs[fwep][fname] = fstate or nil
                savecfg()
                task.spawn(saferep, "FavoritedCosmetics")
                return
            end

            return oncall(self, ...)
        end)
    end
end

local citem
pcall(function() citem = require(lps.Modules.ClientReplicatedClasses.ClientFighter.ClientItem) end)

if citem and citem._CreateViewModel then
    local ocvm = citem._CreateViewModel
    citem._CreateViewModel = function(self, vmref)
        local wname = self.Name
        local wplr = self.ClientFighter and self.ClientFighter.Player
        cwep = (wplr == lp) and wname or nil
        if wplr == lp and equip[wname] and equip[wname].Skin and vmref then
            local skin = equip[wname].Skin
            local dk = self:ToEnum("Data")
            if vmref[dk] then
                vmref[dk][self:ToEnum("Skin")] = skin
                vmref[dk][self:ToEnum("Name")] = skin.Name
            elseif vmref.Data then
                vmref.Data.Skin = skin
                vmref.Data.Name = skin.Name
            end
        end
        local res = ocvm(self, vmref)
        cwep = nil
        return res
    end
end

local vmmod = lps.Modules.ClientReplicatedClasses.ClientFighter.ClientItem:FindFirstChild("ClientViewModel")
if vmmod then
    local cvm = require(vmmod)
    if cvm.GetWrap then
        local ogw = cvm.GetWrap
        cvm.GetWrap = function(self)
            local ci = self.ClientItem
            local wname = ci and ci.Name
            local wplr = ci and ci.ClientFighter and ci.ClientFighter.Player
            local weq = wname and wplr == lp and equip[wname]
            return (weq and weq.Wrap) or ogw(self)
        end
    end
    local onew = cvm.new
    cvm.new = function(rdata, cliitm)
        local wplr = cliitm.ClientFighter and cliitm.ClientFighter.Player
        local wname = cwep or cliitm.Name
        if wplr == lp and equip[wname] then
            local rcls = require(reps.Modules.ReplicatedClass)
            local dk = rcls:ToEnum("Data")
            rdata[dk] = rdata[dk] or {}
            local cos = equip[wname]
            local slot = rdata[dk]
            if cos.Skin then slot[rcls:ToEnum("Skin")] = cos.Skin end
            if cos.Wrap then slot[rcls:ToEnum("Wrap")] = cos.Wrap end
            if cos.Charm then slot[rcls:ToEnum("Charm")] = cos.Charm end
        end
        local res = onew(rdata, cliitm)
        if wplr == lp and equip[wname] and equip[wname].Wrap and res._UpdateWrap then
            res:_UpdateWrap()
            task.delay(0.1, function() if not res._destroyed then res:_UpdateWrap() end end)
        end
        return res
    end
end

local ogvi = ilib.GetViewModelImageFromWeaponData
ilib.GetViewModelImageFromWeaponData = function(self, wdata, hires)
    if not wdata then return ogvi(self, wdata, hires) end
    local wname = wdata.Name
    local weq = equip[wname]
    if weq and weq.Skin and (wdata.Skin == weq.Skin or vprof == lp) then
        local sinfo = self.ViewModels[weq.Skin.Name]
        if sinfo then return sinfo[hires and "ImageHighResolution" or "Image"] or sinfo.Image end
    end
    return ogvi(self, wdata, hires)
end

pcall(function()
    local vpmod = require(lps.Modules.Pages.ViewProfile)
    if vpmod and vpmod.Fetch then
        local ofetch = vpmod.Fetch
        vpmod.Fetch = function(self, tplr)
            vprof = tplr
            return ofetch(self, tplr)
        end
    end
end)

local cent
pcall(function() cent = require(lps.Modules.ClientReplicatedClasses.ClientEntity) end)

if cent and cent._PlayFinisher then
    local ofin = cent._PlayFinisher
    cent._PlayFinisher = function(self, fname, ...)
        local ewep = getewep()
        local tfin = ewep and (fcache[ewep] or (equip[ewep] and equip[ewep].Finisher and equip[ewep].Finisher.Name))
        return ofin(self, tfin or fname, ...)
    end
end

loadcfg()
rebuildinv()

for wname, wdata in equip do
    if wdata.Finisher and wdata.Finisher.Name then
        fcache[wname] = wdata.Finisher.Name
    end
end
print("Unlock All executed successfully!")
        ]]
        loadstring(unlockScript)()
        Rayfield:Notify({
            Title = "Unlock All",
            Content = "Executed! Cosmetics and weapons unlocked.",
            Duration = 3
        })
    end
})

-- ========== 플레이어 초기화 (ESP) ==========
local function onCharacterAdded(targetPlayer)
    task.wait(0.5)
    if NormalAimbot.TeamCheck and not isEnemy(targetPlayer) then return end
    createESP(targetPlayer)
end

Players.PlayerAdded:Connect(function(targetPlayer)
    targetPlayer.CharacterAdded:Connect(function() onCharacterAdded(targetPlayer) end)
    if targetPlayer.Character then onCharacterAdded(targetPlayer) end
end)

task.spawn(function()
    for _, pl in pairs(Players:GetPlayers()) do
        if pl ~= LocalPlayer then
            pl.CharacterAdded:Connect(function() onCharacterAdded(pl) end)
            if pl.Character then onCharacterAdded(pl) end
            task.wait(0.05)
        end
    end
end)

Players.PlayerRemoving:Connect(removeESP)

-- ========== 메인 루프 (일반 에임봇만 실행, 사일런트는 후킹으로 별도) ==========
local lastTargetUpdate = 0
local TARGET_UPDATE_INTERVAL = 0.03
local normalTargetPlayer = nil

RunService.Heartbeat:Connect(function()
    updateESP()

    local now = tick()
    if now - lastTargetUpdate >= TARGET_UPDATE_INTERVAL then
        lastTargetUpdate = now
        if NormalAimbot.Enabled then
            normalTargetPlayer = getClosestTargetForNormal()
        else
            normalTargetPlayer = nil
        end
        if SilentAimbot.Enabled then
            SilentTargetPart = getClosestTargetForSilent()
        else
            SilentTargetPart = nil
        end
    end

    if NormalAimbot.Enabled and normalTargetPlayer and normalTargetPlayer.Character then
        local part = normalTargetPlayer.Character:FindFirstChild(NormalAimbot.TargetPart)
        if part then aimAt(part) end
    end
end)

print("[MoonHook] Loaded - Separate tabs: Aimbot, Silent Aim, ESP, Misc")
