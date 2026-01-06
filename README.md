local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UIS = game:GetService("UserInputService")

local LP = Players.LocalPlayer

local ESPEnabled = false
local SilentEnabled = false
local SpeedEnabled = false
local AntiStunEnabled = false

local Prediction = 0.1
local MaxRangeSq = 400 * 400
local SpeedValue = 700

local AntiStunStrong = 1.2
local AntiStunLight = 0.4
local AntiStunPower = AntiStunStrong

local AllowedKeys = {
    [Enum.KeyCode.Z] = true,
    [Enum.KeyCode.X] = true,
    [Enum.KeyCode.C] = true,
    [Enum.KeyCode.V] = true,
    [Enum.KeyCode.F] = true
}

local TargetPos
local PlayerCache = {}
local ESPs = {}
local AllyCache = {}

local ESPFolder = game.CoreGui:FindFirstChild("PlayerESP") or Instance.new("Folder", game.CoreGui)
ESPFolder.Name = "PlayerESP"

local function SilentKey()
    for k in pairs(AllowedKeys) do
        if UIS:IsKeyDown(k) then
            return true
        end
    end
end

local function CachePlayer(plr)
    local function CharAdded(char)
        PlayerCache[plr] = {
            Char = char,
            HRP = char:WaitForChild("HumanoidRootPart", 3),
            Hum = char:WaitForChild("Humanoid", 3)
        }
    end

    if plr.Character then
        CharAdded(plr.Character)
    end

    plr.CharacterAdded:Connect(CharAdded)
    plr.CharacterRemoving:Connect(function()
        PlayerCache[plr] = nil
    end)
end

for _, p in ipairs(Players:GetPlayers()) do
    if p ~= LP then
        CachePlayer(p)
    end
end

Players.PlayerAdded:Connect(CachePlayer)
Players.PlayerRemoving:Connect(function(p)
    PlayerCache[p] = nil
    if ESPs[p] then
        ESPs[p].Gui:Destroy()
        ESPs[p] = nil
    end
end)

task.spawn(function()
    while true do
        table.clear(AllyCache)

        local gui = LP:FindFirstChild("PlayerGui")
        local sc = gui
            and gui:FindFirstChild("Main")
            and gui.Main:FindFirstChild("Allies")
            and gui.Main.Allies:FindFirstChild("Container")
            and gui.Main.Allies.Container:FindFirstChild("Allies")
            and gui.Main.Allies.Container.Allies:FindFirstChild("ScrollingFrame")

        if sc then
            for _, obj in ipairs(sc:GetDescendants()) do
                if obj:IsA("TextLabel") and obj.Text and obj.Text ~= "" then
                    for _, plr in ipairs(Players:GetPlayers()) do
                        if plr ~= LP and string.find(obj.Text, plr.Name, 1, true) then
                            AllyCache[plr.Name] = true
                        end
                    end
                end
            end
        end

        task.wait(1)
    end
end)

local function IsEnemy(plr)
    if not plr or plr == LP then return false end

    local myTeam = LP.Team
    local hisTeam = plr.Team

    if myTeam and hisTeam then
        if myTeam.Name == "Pirates" and hisTeam.Name == "Marines" then return true end
        if myTeam.Name == "Marines" and hisTeam.Name == "Pirates" then return true end
        if myTeam.Name == "Pirates" and hisTeam.Name == "Pirates" then
            return not AllyCache[plr.Name]
        end
        if myTeam.Name == "Marines" and hisTeam.Name == "Marines" then
            return false
        end
    end

    return true
end

local function ClosestTarget()
    local my = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
    if not my then return end

    local best, dist
    for plr, data in pairs(PlayerCache) do
        if data.HRP and data.Hum and data.Hum.Health > 0 and IsEnemy(plr) then
            local d = data.HRP.Position - my.Position
            local mag = d.X*d.X + d.Y*d.Y + d.Z*d.Z
            if mag < MaxRangeSq and (not dist or mag < dist) then
                dist = mag
                best = data.HRP
            end
        end
    end
    return best
end

task.spawn(function()
    while true do
        if SilentEnabled and SilentKey() then
            local hrp = ClosestTarget()
            TargetPos = hrp and (hrp.Position + hrp.Velocity * Prediction) or nil
        else
            TargetPos = nil
        end
        task.wait(0.05)
    end
end)

local mt = getrawmetatable(game)
setreadonly(mt, false)
local old = mt.__namecall

mt.__namecall = newcclosure(function(self, ...)
    if getnamecallmethod() == "FireServer" and SilentEnabled and TargetPos then
        local a = { ... }
        if typeof(a[1]) == "Vector3" then
            a[1] = TargetPos
            return old(self, unpack(a))
        end
    end
    return old(self, ...)
end)

setreadonly(mt, true)

local function CreateESP(plr)
    if ESPs[plr] then return end

    local gui = Instance.new("BillboardGui", ESPFolder)
    gui.Size = UDim2.fromOffset(240, 50)
    gui.AlwaysOnTop = true
    gui.Enabled = false

    local lvl = Instance.new("TextLabel", gui)
    lvl.Size = UDim2.new(1, 0, 0.45, 0)
    lvl.BackgroundTransparency = 1
    lvl.Font = Enum.Font.SourceSansBold
    lvl.TextSize = 13
    lvl.TextStrokeTransparency = 0.2
    lvl.TextColor3 = Color3.fromRGB(0,170,255)

    local main = Instance.new("TextLabel", gui)
    main.Position = UDim2.new(0, 0, 0.45, 0)
    main.Size = UDim2.new(1, 0, 0.55, 0)
    main.BackgroundTransparency = 1
    main.Font = Enum.Font.SourceSansBold
    main.TextSize = 14
    main.TextStrokeTransparency = 0.2

    ESPs[plr] = {
        Gui = gui,
        Level = lvl,
        Main = main
    }
end

task.spawn(function()
    while true do
        if ESPEnabled then
            local my = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
            for plr, data in pairs(PlayerCache) do
                if data.HRP and data.Hum then
                    CreateESP(plr)
                    local esp = ESPs[plr]

                    esp.Gui.Enabled = true
                    esp.Gui.Adornee = data.Char:FindFirstChild("Head")

                    local d = my and math.floor((data.HRP.Position - my.Position).Magnitude) or 0
                    local lvlValue = "?"
                    local df = plr:FindFirstChild("Data")
                    if df and df:FindFirstChild("Level") then
                        lvlValue = df.Level.Value
                    end

                    esp.Level.Text = "Lv. "..lvlValue
                    esp.Main.Text =
                        "["..math.floor(data.Hum.Health).."] "..plr.DisplayName.." ("..d.."m)"
                    esp.Main.TextColor3 =
                        IsEnemy(plr) and Color3.fromRGB(255,255,0) or Color3.fromRGB(0,255,0)
                end
            end
        else
            for _, e in pairs(ESPs) do
                e.Gui.Enabled = false
            end
        end
        task.wait(0.25)
    end
end)

RunService.RenderStepped:Connect(function()
    local c = LP.Character
    local h = c and c:FindFirstChild("Humanoid")
    local r = c and c:FindFirstChild("HumanoidRootPart")
    if not h or not r then return end

    if SpeedEnabled then
        h.WalkSpeed = SpeedValue
    end
    if AntiStunEnabled and h.MoveDirection.Magnitude > 0 then
        r.CFrame = r.CFrame + h.MoveDirection.Unit * AntiStunPower
    end
end)

UIS.InputBegan:Connect(function(i, gp)
    if gp then return end
    if i.KeyCode == Enum.KeyCode.L then
        ESPEnabled = not ESPEnabled
    elseif i.KeyCode == Enum.KeyCode.B then
        SilentEnabled = not SilentEnabled
    elseif i.KeyCode == Enum.KeyCode.K then
        SpeedEnabled = not SpeedEnabled
        AntiStunEnabled = SpeedEnabled
        AntiStunPower = AntiStunStrong
    elseif i.KeyCode == Enum.KeyCode.P then
        AntiStunEnabled = not AntiStunEnabled
        AntiStunPower = AntiStunLight
    end
end)
