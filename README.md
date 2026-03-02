
-- Public config
local DISCORD_LINK = "https://discord.gg/BhXZzkPGYk" -- where players get key


local _kA = {119, 105, 110}   -- 'w','i','n'
local _kB = {116, 101, 114}   -- 't','e','r'

local function _d(list)
    local s = {}
    for i = 1, #list do
        s[i] = string.char(list[i])
    end
    return table.concat(s)
end

local function _buildKey()
    return _d(_kA) .. _d(_kB)
end

local REQUIRED_KEY = _buildKey()
local SALT = "_D3S7"            -- extra salt for comparison

-- \[NEW] File path for saving key (per game)
local KEY_FILE = "destiny_key_" .. game.PlaceId .. ".txt"

-- Safe wrappers in case executor doesn't support file functions
local function canUseFiles()
    return (typeof(isfile) == "function")
        and (typeof(writefile) == "function")
        and (typeof(readfile) == "function")
end

local function saveKey(key)
    if canUseFiles() then
        writefile(KEY_FILE, key)
    end
end

local function loadSavedKey()
    if canUseFiles() and isfile(KEY_FILE) then
        local ok, data = pcall(readfile, KEY_FILE)
        if ok then
            return tostring(data)
        end
    end
    return nil
end

-- Services
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

_G.DESTINY_KEY_VERIFIED = false

-- Key helpers
local function _normalize(text)
    text = tostring(text or "")
    text = string.gsub(text, "^%s+", "")
    text = string.gsub(text, "%s+$", "")
    return text
end

local function _encodeForCheck(s)
    return s .. SALT
end

local function _expectedEncoded()
    return _encodeForCheck(REQUIRED_KEY)
end

-- \[NEW] Try to auto-verify from saved key BEFORE creating UI
local function tryAutoVerifyFromFile()
    local saved = loadSavedKey()
    if not saved then return false end

    saved = _normalize(saved)
    local candidate = _encodeForCheck(saved)

    if candidate == _expectedEncoded() then
        _G.DESTINY_KEY_VERIFIED = true
        return true
    end
    return false
end

-- If auto-verify works, skip UI entirely
if tryAutoVerifyFromFile() then
    -- Already verified, no GUI, just continue to hub
    return
end

-- UI setup (only if not already verified)
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "DestinyKeySystemV2"
ScreenGui.ResetOnSpawn = false
ScreenGui.IgnoreGuiInset = true
ScreenGui.Parent = PlayerGui

local Frame = Instance.new("Frame")
Frame.Size = UDim2.new(0, 430, 0, 220)
Frame.Position = UDim2.new(0.5, -215, 0.5, -110)
Frame.BackgroundColor3 = Color3.fromRGB(15, 15, 20)
Frame.BorderSizePixel = 0
Frame.Parent = ScreenGui

local UICorner = Instance.new("UICorner")
UICorner.CornerRadius = UDim.new(0, 8)
UICorner.Parent = Frame

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, -20, 0, 40)
Title.Position = UDim2.new(0, 10, 0, 10)
Title.BackgroundTransparency = 1
Title.Text = "Destiny Hub | Key System"
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.Font = Enum.Font.GothamBold
Title.TextSize = 20
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Parent = Frame


local KeyBox = Instance.new("TextBox")
KeyBox.Size = UDim2.new(1, -20, 0, 32)
KeyBox.Position = UDim2.new(0, 10, 0, 70)
KeyBox.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
KeyBox.TextColor3 = Color3.fromRGB(255, 255, 255)
KeyBox.Font = Enum.Font.Gotham
KeyBox.PlaceholderText = "Paste key here (from Discord)"
KeyBox.Text = ""
KeyBox.TextSize = 14
KeyBox.ClearTextOnFocus = false
KeyBox.Parent = Frame

local KeyCorner = Instance.new("UICorner")
KeyCorner.CornerRadius = UDim.new(0, 5)
KeyCorner.Parent = KeyBox

local StatusLabel = Instance.new("TextLabel")
StatusLabel.Size = UDim2.new(1, -20, 0, 22)
StatusLabel.Position = UDim2.new(0, 10, 0, 105)
StatusLabel.BackgroundTransparency = 1
StatusLabel.Text = "Status: Waiting for key..."
StatusLabel.TextColor3 = Color3.fromRGB(255, 255, 0)
StatusLabel.Font = Enum.Font.Gotham
StatusLabel.TextSize = 14
StatusLabel.TextXAlignment = Enum.TextXAlignment.Left
StatusLabel.Parent = Frame

local SubmitButton = Instance.new("TextButton")
SubmitButton.Size = UDim2.new(0.5, -15, 0, 30)
SubmitButton.Position = UDim2.new(0, 10, 0, 145)
SubmitButton.BackgroundColor3 = Color3.fromRGB(40, 120, 255)
SubmitButton.Text = "Submit Key"
SubmitButton.TextColor3 = Color3.fromRGB(255, 255, 255)
SubmitButton.Font = Enum.Font.GothamBold
SubmitButton.TextSize = 14
SubmitButton.Parent = Frame

local SubmitCorner = Instance.new("UICorner")
SubmitCorner.CornerRadius = UDim.new(0, 5)
SubmitCorner.Parent = SubmitButton

local CopyDiscordButton = Instance.new("TextButton")
CopyDiscordButton.Size = UDim2.new(0.5, -15, 0, 30)
CopyDiscordButton.Position = UDim2.new(0.5, 5, 0, 145)
CopyDiscordButton.BackgroundColor3 = Color3.fromRGB(50, 50, 55)
CopyDiscordButton.Text = "Copy Discord"
CopyDiscordButton.TextColor3 = Color3.fromRGB(255, 255, 255)
CopyDiscordButton.Font = Enum.Font.GothamBold
CopyDiscordButton.TextSize = 14
CopyDiscordButton.Parent = Frame

local CopyCorner = Instance.new("UICorner")
CopyCorner.CornerRadius = UDim.new(0, 5)
CopyCorner.Parent = CopyDiscordButton

-- Slight randomization of background color (optional)
local function _randColor()
    return Color3.fromRGB(20 + math.random(0, 10), 20 + math.random(0, 10), 25 + math.random(0, 10))
end

Frame.BackgroundColor3 = _randColor()

-- Key check logic
local tries = 0
local maxTries = 6

local function checkKey()
    if _G.DESTINY_KEY_VERIFIED then return end

    local raw = _normalize(KeyBox.Text)
    local candidate = _encodeForCheck(raw)

    if candidate == _expectedEncoded() then
        _G.DESTINY_KEY_VERIFIED = true

        -- \[NEW] Save valid key so next time it's auto-verified
        saveKey(raw)

        StatusLabel.Text = "Status: Key correct! Loading Destiny Hub..."
        StatusLabel.TextColor3 = Color3.fromRGB(0, 255, 0)

        KeyBox.TextEditable = false
        SubmitButton.AutoButtonColor = false
        SubmitButton.BackgroundColor3 = Color3.fromRGB(80, 180, 80)

        task.delay(0.5, function()
            if ScreenGui then
                ScreenGui:Destroy()
            end
        end)
    else
        tries += 1
        _G.DESTINY_KEY_VERIFIED = false
        StatusLabel.Text = "Status: Invalid key. Attempts: " .. tostring(tries) .. "/" .. tostring(maxTries)
        StatusLabel.TextColor3 = Color3.fromRGB(255, 0, 0)

        if tries >= maxTries then
            StatusLabel.Text = "Status: Too many attempts. Rejoin required."
            KeyBox.TextEditable = false
            SubmitButton.AutoButtonColor = false
            SubmitButton.BackgroundColor3 = Color3.fromRGB(80, 80, 80)
            SubmitButton.Text = "Locked"
        end
    end
end

SubmitButton.MouseButton1Click:Connect(checkKey)

CopyDiscordButton.MouseButton1Click:Connect(function()
    -- Copy to clipboard if executor supports it
    if setclipboard then
        setclipboard(DISCORD_LINK)
        StatusLabel.Text = "Status: Discord link copied to clipboard."
        StatusLabel.TextColor3 = Color3.fromRGB(0, 255, 0)
    else
        -- Fallback: print to console if setclipboard is not available
        print("[Destiny Hub] Join Discord for key: " .. DISCORD_LINK)
        StatusLabel.Text = "Status: Executor has no clipboard, link printed in console."
        StatusLabel.TextColor3 = Color3.fromRGB(0, 170, 255)
    end
end)

KeyBox.FocusLost:Connect(function(enterPressed)
    if enterPressed then
        checkKey()
    end
end)


repeat task.wait() until _G.DESTINY_KEY_VERIFIED

-- Services
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local StarterGui = game:GetService("StarterGui")
local Player = Players.LocalPlayer

-- Try to use CoreGui; fallback to PlayerGui if CoreGui blocked
local guiParent = game:FindFirstChildOfClass("CoreGui") or Player:WaitForChild("PlayerGui")

-- GUI Setup
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "WinterMultiFlingGUI"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = guiParent

-- Main Frame (new style)
local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 360, 0, 380)
MainFrame.Position = UDim2.new(0.5, -180, 0.5, -190)
MainFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 20)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Draggable = true
MainFrame.Parent = ScreenGui

local UICorner_Main = Instance.new("UICorner")
UICorner_Main.CornerRadius = UDim.new(0, 10)
UICorner_Main.Parent = MainFrame

-- Title Bar (gradient style)
local TitleBar = Instance.new("Frame")
TitleBar.Size = UDim2.new(1, 0, 0, 32)
TitleBar.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
TitleBar.BorderSizePixel = 0
TitleBar.Parent = MainFrame

local UICorner_Title = Instance.new("UICorner")
UICorner_Title.CornerRadius = UDim.new(0, 10)
UICorner_Title.Parent = TitleBar

-- Title
local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, -70, 1, 0)
Title.Position = UDim2.new(0, 10, 0, 0)
Title.BackgroundTransparency = 1
Title.Text = "Winter's Multi-Person Fling"
Title.TextColor3 = Color3.fromRGB(120, 200, 255)
Title.Font = Enum.Font.GothamBold
Title.TextSize = 16
Title.TextXAlignment = Enum.TextXAlignment.Left
Title.Parent = TitleBar

-- Close Button
local CloseButton = Instance.new("TextButton")
CloseButton.Position = UDim2.new(1, -34, 0, 4)
CloseButton.Size = UDim2.new(0, 28, 0, 24)
CloseButton.BackgroundColor3 = Color3.fromRGB(200, 60, 60)
CloseButton.BorderSizePixel = 0
CloseButton.Text = "X"
CloseButton.TextColor3 = Color3.fromRGB(255, 255, 255)
CloseButton.Font = Enum.Font.GothamBold
CloseButton.TextSize = 16
CloseButton.Parent = TitleBar

local UICorner_Close = Instance.new("UICorner")
UICorner_Close.CornerRadius = UDim.new(0, 6)
UICorner_Close.Parent = CloseButton

-- Status Label (top, under title)
local StatusLabel = Instance.new("TextLabel")
StatusLabel.Position = UDim2.new(0, 12, 0, 42)
StatusLabel.Size = UDim2.new(1, -24, 0, 26)
StatusLabel.BackgroundTransparency = 1
StatusLabel.Text = "Select targets to fling"
StatusLabel.TextColor3 = Color3.fromRGB(230, 230, 230)
StatusLabel.Font = Enum.Font.Gotham
StatusLabel.TextSize = 14
StatusLabel.TextXAlignment = Enum.TextXAlignment.Left
StatusLabel.Parent = MainFrame

-- Player Selection Frame (rounded, left side)
local SelectionFrame = Instance.new("Frame")
SelectionFrame.Position = UDim2.new(0, 12, 0, 74)
SelectionFrame.Size = UDim2.new(0, 220, 0, 230)
SelectionFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 35)
SelectionFrame.BorderSizePixel = 0
SelectionFrame.Parent = MainFrame

local UICorner_Selection = Instance.new("UICorner")
UICorner_Selection.CornerRadius = UDim.new(0, 8)
UICorner_Selection.Parent = SelectionFrame

-- Selection title
local SelectionTitle = Instance.new("TextLabel")
SelectionTitle.Size = UDim2.new(1, -10, 0, 20)
SelectionTitle.Position = UDim2.new(0, 6, 0, 4)
SelectionTitle.BackgroundTransparency = 1
SelectionTitle.Text = "Target Players"
SelectionTitle.TextColor3 = Color3.fromRGB(180, 200, 255)
SelectionTitle.Font = Enum.Font.GothamSemibold
SelectionTitle.TextSize = 13
SelectionTitle.TextXAlignment = Enum.TextXAlignment.Left
SelectionTitle.Parent = SelectionFrame

-- Player List ScrollFrame
local PlayerScrollFrame = Instance.new("ScrollingFrame")
PlayerScrollFrame.Position = UDim2.new(0, 6, 0, 26)
PlayerScrollFrame.Size = UDim2.new(1, -12, 1, -32)
PlayerScrollFrame.BackgroundTransparency = 1
PlayerScrollFrame.BorderSizePixel = 0
PlayerScrollFrame.ScrollBarThickness = 4
PlayerScrollFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
PlayerScrollFrame.Parent = SelectionFrame

-- Right-side control panel
local ControlFrame = Instance.new("Frame")
ControlFrame.Position = UDim2.new(0, 240, 0, 74)
ControlFrame.Size = UDim2.new(1, -252, 0, 230)
ControlFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 35)
ControlFrame.BorderSizePixel = 0
ControlFrame.Parent = MainFrame

local UICorner_Control = Instance.new("UICorner")
UICorner_Control.CornerRadius = UDim.new(0, 8)
UICorner_Control.Parent = ControlFrame

local ControlTitle = Instance.new("TextLabel")
ControlTitle.Size = UDim2.new(1, -10, 0, 20)
ControlTitle.Position = UDim2.new(0, 6, 0, 4)
ControlTitle.BackgroundTransparency = 1
ControlTitle.Text = "Controls"
ControlTitle.TextColor3 = Color3.fromRGB(180, 200, 255)
ControlTitle.Font = Enum.Font.GothamSemibold
ControlTitle.TextSize = 13
ControlTitle.TextXAlignment = Enum.TextXAlignment.Left
ControlTitle.Parent = ControlFrame

-- Buttons (Start/Stop/Select/Deselect) inside ControlFrame
local StartButton = Instance.new("TextButton")
StartButton.Position = UDim2.new(0, 10, 0, 34)
StartButton.Size = UDim2.new(1, -20, 0, 36)
StartButton.BackgroundColor3 = Color3.fromRGB(65, 180, 110)
StartButton.BorderSizePixel = 0
StartButton.Text = "Start Fling"
StartButton.TextColor3 = Color3.fromRGB(255, 255, 255)
StartButton.Font = Enum.Font.GothamBold
StartButton.TextSize = 16
StartButton.Parent = ControlFrame

local UICorner_Start = Instance.new("UICorner")
UICorner_Start.CornerRadius = UDim.new(0, 8)
UICorner_Start.Parent = StartButton

local StopButton = Instance.new("TextButton")
StopButton.Position = UDim2.new(0, 10, 0, 80)
StopButton.Size = UDim2.new(1, -20, 0, 36)
StopButton.BackgroundColor3 = Color3.fromRGB(200, 80, 80)
StopButton.BorderSizePixel = 0
StopButton.Text = "Stop Fling"
StopButton.TextColor3 = Color3.fromRGB(255, 255, 255)
StopButton.Font = Enum.Font.GothamBold
StopButton.TextSize = 16
StopButton.Parent = ControlFrame

local UICorner_Stop = Instance.new("UICorner")
UICorner_Stop.CornerRadius = UDim.new(0, 8)
UICorner_Stop.Parent = StopButton

local SelectAllButton = Instance.new("TextButton")
SelectAllButton.Position = UDim2.new(0, 10, 0, 130)
SelectAllButton.Size = UDim2.new(1, -20, 0, 32)
SelectAllButton.BackgroundColor3 = Color3.fromRGB(60, 100, 170)
SelectAllButton.BorderSizePixel = 0
SelectAllButton.Text = "Select All"
SelectAllButton.TextColor3 = Color3.fromRGB(255, 255, 255)
SelectAllButton.Font = Enum.Font.Gotham
SelectAllButton.TextSize = 14
SelectAllButton.Parent = ControlFrame

local UICorner_SAll = Instance.new("UICorner")
UICorner_SAll.CornerRadius = UDim.new(0, 8)
UICorner_SAll.Parent = SelectAllButton

local DeselectAllButton = Instance.new("TextButton")
DeselectAllButton.Position = UDim2.new(0, 10, 0, 170)
DeselectAllButton.Size = UDim2.new(1, -20, 0, 32)
DeselectAllButton.BackgroundColor3 = Color3.fromRGB(80, 80, 110)
DeselectAllButton.BorderSizePixel = 0
DeselectAllButton.Text = "Deselect All"
DeselectAllButton.TextColor3 = Color3.fromRGB(255, 255, 255)
DeselectAllButton.Font = Enum.Font.Gotham
DeselectAllButton.TextSize = 14
DeselectAllButton.Parent = ControlFrame

local UICorner_DSAll = Instance.new("UICorner")
UICorner_DSAll.CornerRadius = UDim.new(0, 8)
UICorner_DSAll.Parent = DeselectAllButton

-- Variables
local SelectedTargets = {}
local PlayerCheckboxes = {}
local FlingActive = false
local FlingConnection = nil

getgenv().OldPos = nil
getgenv().FPDH = workspace.FallenPartsDestroyHeight

-- Notification wrapper
local function Message(Title, Text, Time)
    pcall(function()
        StarterGui:SetCore("SendNotification", {
            Title = Title,
            Text = Text,
            Duration = Time or 5
        })
    end)
end

-- Count selected targets
local function CountSelectedTargets()
    local count = 0
    for _ in pairs(SelectedTargets) do
        count = count + 1
    end
    return count
end

-- Update status display
local function UpdateStatus()
    local count = CountSelectedTargets()
    if FlingActive then
        StatusLabel.Text = "Flinging " .. count .. " target(s)"
        StatusLabel.TextColor3 = Color3.fromRGB(255, 140, 140)
    else
        StatusLabel.Text = count .. " target(s) selected"
        StatusLabel.TextColor3 = Color3.fromRGB(230, 230, 230)
    end
end

-- Function to update player list
local function RefreshPlayerList()
    for _, child in pairs(PlayerScrollFrame:GetChildren()) do
        if not child:IsA("UIListLayout") then
            child:Destroy()
        end
    end
    PlayerCheckboxes = {}

    local PlayerList = Players:GetPlayers()
    table.sort(PlayerList, function(a, b)
        return a.Name:lower() < b.Name:lower()
    end)

    local yPosition = 4
    for _, plr in ipairs(PlayerList) do
        if plr ~= Player then
            local PlayerEntry = Instance.new("Frame")
            PlayerEntry.Size = UDim2.new(1, -4, 0, 26)
            PlayerEntry.Position = UDim2.new(0, 2, 0, yPosition)
            PlayerEntry.BackgroundColor3 = Color3.fromRGB(35, 35, 50)
            PlayerEntry.BorderSizePixel = 0
            PlayerEntry.Parent = PlayerScrollFrame

            local UICorner_Entry = Instance.new("UICorner")
            UICorner_Entry.CornerRadius = UDim.new(0, 6)
            UICorner_Entry.Parent = PlayerEntry

            local Checkbox = Instance.new("TextButton")
            Checkbox.Size = UDim2.new(0, 20, 0, 20)
            Checkbox.Position = UDim2.new(0, 4, 0.5, -10)
            Checkbox.BackgroundColor3 = Color3.fromRGB(55, 55, 80)
            Checkbox.BorderSizePixel = 0
            Checkbox.Text = ""
            Checkbox.Parent = PlayerEntry

            local UICorner_Check = Instance.new("UICorner")
            UICorner_Check.CornerRadius = UDim.new(0, 4)
            UICorner_Check.Parent = Checkbox

            local Checkmark = Instance.new("TextLabel")
            Checkmark.Size = UDim2.new(1, 0, 1, 0)
            Checkmark.BackgroundTransparency = 1
            Checkmark.Text = "✓"
            Checkmark.TextColor3 = Color3.fromRGB(120, 220, 140)
            Checkmark.TextSize = 16
            Checkmark.Font = Enum.Font.GothamBold
            Checkmark.Visible = SelectedTargets[plr.Name] ~= nil
            Checkmark.Parent = Checkbox

            local NameLabel = Instance.new("TextLabel")
            NameLabel.Size = UDim2.new(1, -34, 1, 0)
            NameLabel.Position = UDim2.new(0, 30, 0, 0)
            NameLabel.BackgroundTransparency = 1
            NameLabel.Text = plr.Name
            NameLabel.TextColor3 = Color3.fromRGB(230, 230, 240)
            NameLabel.TextSize = 14
            NameLabel.Font = Enum.Font.Gotham
            NameLabel.TextXAlignment = Enum.TextXAlignment.Left
            NameLabel.Parent = PlayerEntry

            local ClickArea = Instance.new("TextButton")
            ClickArea.Size = UDim2.new(1, 0, 1, 0)
            ClickArea.BackgroundTransparency = 1
            ClickArea.Text = ""
            ClickArea.ZIndex = 2
            ClickArea.Parent = PlayerEntry

            ClickArea.MouseButton1Click:Connect(function()
                if SelectedTargets[plr.Name] then
                    SelectedTargets[plr.Name] = nil
                    Checkmark.Visible = false
                else
                    SelectedTargets[plr.Name] = plr
                    Checkmark.Visible = true
                end
                UpdateStatus()
            end)

            PlayerCheckboxes[plr.Name] = {
                Entry = PlayerEntry,
                Checkmark = Checkmark
            }

            yPosition = yPosition + 30
        end
    end

    PlayerScrollFrame.CanvasSize = UDim2.new(0, 0, 0, yPosition + 4)
end

-- Function to select/deselect all players
local function ToggleAllPlayers(select)
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= Player then
            local checkboxData = PlayerCheckboxes[plr.Name]
            if checkboxData then
                if select then
                    SelectedTargets[plr.Name] = plr
                    checkboxData.Checkmark.Visible = true
                else
                    SelectedTargets[plr.Name] = nil
                    checkboxData.Checkmark.Visible = false
                end
            end
        end
    end
    UpdateStatus()
end

-- The fling function from zqyDSUWX (logic unchanged)
local function SkidFling(TargetPlayer)
    local Character = Player.Character
    local Humanoid = Character and Character:FindFirstChildOfClass("Humanoid")
    local RootPart = Humanoid and Humanoid.RootPart
    local TCharacter = TargetPlayer.Character
    if not TCharacter then return end

    local THumanoid
    local TRootPart
    local THead
    local Accessory
    local Handle

    if TCharacter:FindFirstChildOfClass("Humanoid") then
        THumanoid = TCharacter:FindFirstChildOfClass("Humanoid")
    end
    if THumanoid and THumanoid.RootPart then
        TRootPart = THumanoid.RootPart
    end
    if TCharacter:FindFirstChild("Head") then
        THead = TCharacter.Head
    end
    if TCharacter:FindFirstChildOfClass("Accessory") then
        Accessory = TCharacter:FindFirstChildOfClass("Accessory")
    end
    if Accessory and Accessory:FindFirstChild("Handle") then
        Handle = Accessory.Handle
    end

    if Character and Humanoid and RootPart then
        if RootPart.Velocity.Magnitude < 50 then
            getgenv().OldPos = RootPart.CFrame
        end

        if THumanoid and THumanoid.Sit then
            return Message("Error", TargetPlayer.Name .. " is sitting", 2)
        end

        if THead then
            workspace.CurrentCamera.CameraSubject = THead
        elseif Handle then
            workspace.CurrentCamera.CameraSubject = Handle
        elseif THumanoid and TRootPart then
            workspace.CurrentCamera.CameraSubject = THumanoid
        end

        if not TCharacter:FindFirstChildWhichIsA("BasePart") then
            return
        end

        local function FPos(BasePart, Pos, Ang)
            RootPart.CFrame = CFrame.new(BasePart.Position) * Pos * Ang
            Character:SetPrimaryPartCFrame(CFrame.new(BasePart.Position) * Pos * Ang)
            RootPart.Velocity = Vector3.new(9e7, 9e7 * 10, 9e7)
            RootPart.RotVelocity = Vector3.new(9e8, 9e8, 9e8)
        end

        local function SFBasePart(BasePart)
            local TimeToWait = 2
            local TimeNow = tick()
            local Angle = 0
            repeat
                if RootPart and THumanoid then
                    if BasePart.Velocity.Magnitude < 50 then
                        Angle = Angle + 100
                        FPos(BasePart, CFrame.new(0, 1.5, 0) + THumanoid.MoveDirection * BasePart.Velocity.Magnitude / 1.25, CFrame.Angles(math.rad(Angle), 0, 0))
                        task.wait()
                        FPos(BasePart, CFrame.new(0, -1.5, 0) + THumanoid.MoveDirection * BasePart.Velocity.Magnitude / 1.25, CFrame.Angles(math.rad(Angle), 0, 0))
                        task.wait()
                        FPos(BasePart, CFrame.new(0, 1.5, 0) + THumanoid.MoveDirection * BasePart.Velocity.Magnitude / 1.25, CFrame.Angles(math.rad(Angle), 0, 0))
                        task.wait()
                        FPos(BasePart, CFrame.new(0, -1.5, 0) + THumanoid.MoveDirection * BasePart.Velocity.Magnitude / 1.25, CFrame.Angles(math.rad(Angle), 0, 0))
                        task.wait()
                        FPos(BasePart, CFrame.new(0, 1.5, 0) + THumanoid.MoveDirection, CFrame.Angles(math.rad(Angle), 0, 0))
                        task.wait()
                        FPos(BasePart, CFrame.new(0, -1.5, 0) + THumanoid.MoveDirection, CFrame.Angles(math.rad(Angle), 0, 0))
                        task.wait()
                    else
                        FPos(BasePart, CFrame.new(0, 1.5, THumanoid.WalkSpeed), CFrame.Angles(math.rad(90), 0, 0))
                        task.wait()
                        FPos(BasePart, CFrame.new(0, -1.5, -THumanoid.WalkSpeed), CFrame.Angles(0, 0, 0))
                        task.wait()
                        FPos(BasePart, CFrame.new(0, 1.5, THumanoid.WalkSpeed), CFrame.Angles(math.rad(90), 0, 0))
                        task.wait()

                        FPos(BasePart, CFrame.new(0, -1.5, 0), CFrame.Angles(math.rad(90), 0, 0))
                        task.wait()
                        FPos(BasePart, CFrame.new(0, -1.5, 0), CFrame.Angles(0, 0, 0))
                        task.wait()
                        FPos(BasePart, CFrame.new(0, -1.5, 0), CFrame.Angles(math.rad(90), 0, 0))
                        task.wait()
                        FPos(BasePart, CFrame.new(0, -1.5, 0), CFrame.Angles(0, 0, 0))
                        task.wait()
                    end
                end
            until TimeNow + TimeToWait < tick() or not FlingActive
        end

        workspace.FallenPartsDestroyHeight = 0/0

        local BV = Instance.new("BodyVelocity")
        BV.Parent = RootPart
        BV.Velocity = Vector3.new(0, 0, 0)
        BV.MaxForce = Vector3.new(9e9, 9e9, 9e9)

        Humanoid:SetStateEnabled(Enum.HumanoidStateType.Seated, false)

        if TRootPart then
            SFBasePart(TRootPart)
        elseif THead then
            SFBasePart(THead)
        elseif Handle then
            SFBasePart(Handle)
        else
            return Message("Error", TargetPlayer.Name .. " has no valid parts", 2)
        end

        BV:Destroy()
        Humanoid:SetStateEnabled(Enum.HumanoidStateType.Seated, true)
        workspace.CurrentCamera.CameraSubject = Humanoid

        if getgenv().OldPos then
            repeat
                RootPart.CFrame = getgenv().OldPos * CFrame.new(0, 0.5, 0)
                Player.Character:SetPrimaryPartCFrame(getgenv().OldPos * CFrame.new(0, 0.5, 0))
                Humanoid:ChangeState("GettingUp")
                for _, part in pairs(Player.Character:GetChildren()) do
                    if part:IsA("BasePart") then
                        part.Velocity, part.RotVelocity = Vector3.new(), Vector3.new()
                    end
                end
                task.wait()
            until (RootPart.Position - getgenv().OldPos.Position).Magnitude < 25
            workspace.FallenPartsDestroyHeight = getgenv().FPDH
        end
    else
        return Message("Error", "Your character is not ready", 2)
    end
end

-- Start flinging selected targets
local function StartFling()
    if FlingActive then return end

    local count = CountSelectedTargets()
    if count == 0 then
        StatusLabel.Text = "No targets selected!"
        wait(1)
        StatusLabel.Text = "Select targets to fling"
        return
    end

    FlingActive = true
    UpdateStatus()
    Message("Started", "Winter's Multi-Person Fling: flinging " .. count .. " target(s)", 2)

    spawn(function()
        while FlingActive do
            local validTargets = {}

            for name, plr in pairs(SelectedTargets) do
                if plr and plr.Parent then
                    validTargets[name] = plr
                else
                    SelectedTargets[name] = nil
                    local checkbox = PlayerCheckboxes[name]
                    if checkbox then
                        checkbox.Checkmark.Visible = false
                    end
                end
            end

            for _, plr in pairs(validTargets) do
                if not FlingActive then break end
                SkidFling(plr)
                wait(0.1)
            end

            UpdateStatus()
            wait(0.5)
        end
    end)
end

-- Stop flinging
local function StopFling()
    if not FlingActive then return end

    FlingActive = false
    UpdateStatus()
    Message("Stopped", "Winter's Multi-Person Fling has been stopped", 2)
end

-- Set up button connections
StartButton.MouseButton1Click:Connect(StartFling)
StopButton.MouseButton1Click:Connect(StopFling)
SelectAllButton.MouseButton1Click:Connect(function()
    ToggleAllPlayers(true)
end)
DeselectAllButton.MouseButton1Click:Connect(function()
    ToggleAllPlayers(false)
end)
CloseButton.MouseButton1Click:Connect(function()
    StopFling()
    ScreenGui:Destroy()
end)

-- Handle player joining/leaving
Players.PlayerAdded:Connect(function()
    RefreshPlayerList()
    UpdateStatus()
end)

Players.PlayerRemoving:Connect(function(plr)
    if SelectedTargets[plr.Name] then
        SelectedTargets[plr.Name] = nil
    end
    RefreshPlayerList()
    UpdateStatus()
end)

-- Initialize
RefreshPlayerList()
UpdateStatus()
Message("Loaded", "Winter's Multi-Person Fling GUI loaded!", 3)
