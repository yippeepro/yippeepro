---//Variables
local Settings = {
    MobFarm = {
        Toggle = false,
        Selectedmob = "",
        Position = 0,
    
    },
    SandDebrisFarm = {
        Toggle = false,
    },
    Dailyquest = {
        Toggle = false,
        Position = 5

    },
    Chestfarm = false,
    AstralChestfarm = false,
    Shardsfarm = false,
    Autostore = false,
    AutoFarmstand = {
        Toggle = false,
        Stand = ""
    },
    Autoquest = {
        Toggle = false,
        Position = 6,
    },
    Selectednpc = "",
    BossFarm = {
        Toggle = false,
        Boss = "",
        Position = 6
    },
    }
    local player = game.Players.LocalPlayer
    local attackremote = game:GetService("ReplicatedStorage").ReplicatedModules.KnitPackage.Knit.Services.MoveInputService.RF.FireInput
    local mobTable = {0,10 ,20, 30, 40,50,60,70,80,90,100}
    local npctable = {}
    local dialogueremote = game:GetService("ReplicatedStorage").ReplicatedModules.KnitPackage.Knit.Services.DialogueService.RF.CheckDialogue
    local spawnbossremote = game:GetService("ReplicatedStorage").ReplicatedModules.KnitPackage.Knit.Services.NPCService.RF.SpawnBoss
    local bossTable = {"Professor Diavolo","Hold up, aint you Dio?","Boa H","Umbra","Puddest","Minos Prime"}


    --//ESP Settings--
--Settings--
local ESP = {
    Enabled = false,
    Boxes = false,
    BoxShift = CFrame.new(0,-1.5,0),
BoxSize = Vector3.new(4,6,0),
    Color = Color3.fromRGB(255, 0, 0),
    FaceCamera = false,
    Names = false,
    TeamColor = false,
    Thickness = 2,
    AttachShift = 1,
    TeamMates = false,
    Players = true,
    
    Objects = setmetatable({}, {__mode="kv"}),
    Overrides = {}
}

--ESP Declarations--
local cam = workspace.CurrentCamera
local plrs = game:GetService("Players")
local plr = plrs.LocalPlayer
local mouse = plr:GetMouse()

local V3new = Vector3.new
local WorldToViewportPoint = cam.WorldToViewportPoint

--ESP Functions--
local function Draw(obj, props)
local new = Drawing.new(obj)

props = props or {}
for i,v in pairs(props) do
new[i] = v
end
return new
end

function ESP:GetTeam(p)
local ov = self.Overrides.GetTeam
if ov then
return ov(p)
end

return p and p.Team
end

function ESP:IsTeamMate(p)
    local ov = self.Overrides.IsTeamMate
if ov then
return ov(p)
    end
    
    return self:GetTeam(p) == self:GetTeam(plr)
end

function ESP:GetColor(obj)
local ov = self.Overrides.GetColor
if ov then
return ov(obj)
    end
    local p = self:GetPlrFromChar(obj)
return p and self.TeamColor and p.Team and p.Team.TeamColor.Color or self.Color
end

function ESP:GetPlrFromChar(char)
local ov = self.Overrides.GetPlrFromChar
if ov then
return ov(char)
end

return plrs:GetPlayerFromCharacter(char)
end

function ESP:Toggle(bool)
    self.Enabled = bool
    if not bool then
        for i,v in pairs(self.Objects) do
            if v.Type == "Box" then --fov circle etc
                if v.Temporary then
                    v:Remove()
                else
                    for i,v in pairs(v.Components) do
                        v.Visible = false
                    end
                end
            end
        end
    end
end

function ESP:GetBox(obj)
    return self.Objects[obj]
end

function ESP:AddObjectListener(parent, options)
    local function NewListener(c)
        if type(options.Type) == "string" and c:IsA(options.Type) or options.Type == nil then
            if type(options.Name) == "string" and c.Name == options.Name or options.Name == nil then
                if not options.Validator or options.Validator(c) then
                    local box = ESP:Add(c, {
                        PrimaryPart = type(options.PrimaryPart) == "string" and c:WaitForChild(options.PrimaryPart) or type(options.PrimaryPart) == "function" and options.PrimaryPart(c),
                        Color = type(options.Color) == "function" and options.Color(c) or options.Color,
                        ColorDynamic = options.ColorDynamic,
                        Name = type(options.CustomName) == "function" and options.CustomName(c) or options.CustomName,
                        IsEnabled = options.IsEnabled,
                        RenderInNil = options.RenderInNil
                    })
                    --TODO: add a better way of passing options
                    if options.OnAdded then
                        coroutine.wrap(options.OnAdded)(box)
                    end
                end
            end
        end
    end

    if options.Recursive then
        parent.DescendantAdded:Connect(NewListener)
        for i,v in pairs(parent:GetDescendants()) do
            coroutine.wrap(NewListener)(v)
        end
    else
        parent.ChildAdded:Connect(NewListener)
        for i,v in pairs(parent:GetChildren()) do
            coroutine.wrap(NewListener)(v)
        end
    end
end

local boxBase = {}
boxBase.__index = boxBase

function boxBase:Remove()
    ESP.Objects[self.Object] = nil
    for i,v in pairs(self.Components) do
        v.Visible = false
        v:Remove()
        self.Components[i] = nil
    end
end

function boxBase:Update()
    if not self.PrimaryPart then
        --warn("not supposed to print", self.Object)
        return self:Remove()
    end

    local color
    if ESP.Highlighted == self.Object then
       color = ESP.HighlightColor
    else
        color = self.Color or self.ColorDynamic and self:ColorDynamic() or ESP:GetColor(self.Object) or ESP.Color
    end

    local allow = true
    if ESP.Overrides.UpdateAllow and not ESP.Overrides.UpdateAllow(self) then
        allow = false
    end
    if self.Player and not ESP.TeamMates and ESP:IsTeamMate(self.Player) then
        allow = false
    end
    if self.Player and not ESP.Players then
        allow = false
    end
    if self.IsEnabled and (type(self.IsEnabled) == "string" and not ESP[self.IsEnabled] or type(self.IsEnabled) == "function" and not self:IsEnabled()) then
        allow = false
    end
    if not workspace:IsAncestorOf(self.PrimaryPart) and not self.RenderInNil then
        allow = false
    end

    if not allow then
        for i,v in pairs(self.Components) do
            v.Visible = false
        end
        return
    end

    if ESP.Highlighted == self.Object then
        color = ESP.HighlightColor
    end

    --calculations--
    local cf = self.PrimaryPart.CFrame
    if ESP.FaceCamera then
        cf = CFrame.new(cf.p, cam.CFrame.p)
    end
    local size = self.Size
    local locs = {
        TopLeft = cf * ESP.BoxShift * CFrame.new(size.X/2,size.Y/2,0),
        TopRight = cf * ESP.BoxShift * CFrame.new(-size.X/2,size.Y/2,0),
        BottomLeft = cf * ESP.BoxShift * CFrame.new(size.X/2,-size.Y/2,0),
        BottomRight = cf * ESP.BoxShift * CFrame.new(-size.X/2,-size.Y/2,0),
        TagPos = cf * ESP.BoxShift * CFrame.new(0,size.Y/2,0),
        Torso = cf * ESP.BoxShift
    }

    if ESP.Boxes then
        local TopLeft, Vis1 = WorldToViewportPoint(cam, locs.TopLeft.p)
        local TopRight, Vis2 = WorldToViewportPoint(cam, locs.TopRight.p)
        local BottomLeft, Vis3 = WorldToViewportPoint(cam, locs.BottomLeft.p)
        local BottomRight, Vis4 = WorldToViewportPoint(cam, locs.BottomRight.p)

        if self.Components.Quad then
            if Vis1 or Vis2 or Vis3 or Vis4 then
                self.Components.Quad.Visible = true
                self.Components.Quad.PointA = Vector2.new(TopRight.X, TopRight.Y)
                self.Components.Quad.PointB = Vector2.new(TopLeft.X, TopLeft.Y)
                self.Components.Quad.PointC = Vector2.new(BottomLeft.X, BottomLeft.Y)
                self.Components.Quad.PointD = Vector2.new(BottomRight.X, BottomRight.Y)
                self.Components.Quad.Color = color
            else
                self.Components.Quad.Visible = false
            end
        end
    else
        self.Components.Quad.Visible = false
    end

    if ESP.Names then
        local TagPos, Vis5 = WorldToViewportPoint(cam, locs.TagPos.p)
        
        if Vis5 then
            self.Components.Name.Visible = true
            self.Components.Name.Position = Vector2.new(TagPos.X, TagPos.Y)
            self.Components.Name.Text = self.Name
            self.Components.Name.Color = color
            
            self.Components.Distance.Visible = true
            self.Components.Distance.Position = Vector2.new(TagPos.X, TagPos.Y + 14)
            self.Components.Distance.Text = math.floor((cam.CFrame.p - cf.p).magnitude) .."m away"
            self.Components.Distance.Color = color
        else
            self.Components.Name.Visible = false
            self.Components.Distance.Visible = false
        end
    else
        self.Components.Name.Visible = false
        self.Components.Distance.Visible = false
    end
    
    if ESP.Tracers then
        local TorsoPos, Vis6 = WorldToViewportPoint(cam, locs.Torso.p)

        if Vis6 then
            self.Components.Tracer.Visible = true
            self.Components.Tracer.From = Vector2.new(TorsoPos.X, TorsoPos.Y)
            self.Components.Tracer.To = Vector2.new(cam.ViewportSize.X/2,cam.ViewportSize.Y/ESP.AttachShift)
            self.Components.Tracer.Color = color
        else
            self.Components.Tracer.Visible = false
        end
    else
        self.Components.Tracer.Visible = false
    end
end

function ESP:Add(obj, options)
    if not obj.Parent and not options.RenderInNil then
        return warn(obj, "has no parent")
    end

    local box = setmetatable({
        Name = options.Name or obj.Name,
        Type = "Box",
        Color = options.Color --[[or self:GetColor(obj)]],
        Size = options.Size or self.BoxSize,
        Object = obj,
        Player = options.Player or plrs:GetPlayerFromCharacter(obj),
        PrimaryPart = options.PrimaryPart or obj.ClassName == "Model" and (obj.PrimaryPart or obj:FindFirstChild("HumanoidRootPart") or obj:FindFirstChildWhichIsA("BasePart")) or obj:IsA("BasePart") and obj,
        Components = {},
        IsEnabled = options.IsEnabled,
        Temporary = options.Temporary,
        ColorDynamic = options.ColorDynamic,
        RenderInNil = options.RenderInNil
    }, boxBase)

    if self:GetBox(obj) then
        self:GetBox(obj):Remove()
    end

    box.Components["Quad"] = Draw("Quad", {
        Thickness = self.Thickness,
        Color = color,
        Transparency = 1,
        Filled = false,
        Visible = self.Enabled and self.Boxes
    })
    box.Components["Name"] = Draw("Text", {
Text = box.Name,
Color = box.Color,
Center = true,
Outline = true,
        Size = 19,
        Visible = self.Enabled and self.Names
})
box.Components["Distance"] = Draw("Text", {
Color = box.Color,
Center = true,
Outline = true,
        Size = 19,
        Visible = self.Enabled and self.Names
})

box.Components["Tracer"] = Draw("Line", {
Thickness = ESP.Thickness,
Color = box.Color,
        Transparency = 1,
        Visible = self.Enabled and self.Tracers
    })
    self.Objects[obj] = box
    
    obj.AncestryChanged:Connect(function(_, parent)
        if parent == nil and ESP.AutoRemove ~= false then
            box:Remove()
        end
    end)
    obj:GetPropertyChangedSignal("Parent"):Connect(function()
        if obj.Parent == nil and ESP.AutoRemove ~= false then
            box:Remove()
        end
    end)

    local hum = obj:FindFirstChildOfClass("Humanoid")
if hum then
        hum.Died:Connect(function()
            if ESP.AutoRemove ~= false then
                box:Remove()
            end
end)
    end

    return box
end

local function CharAdded(char)
    local p = plrs:GetPlayerFromCharacter(char)
    if not char:FindFirstChild("HumanoidRootPart") then
        local ev
        ev = char.ChildAdded:Connect(function(c)
            if c.Name == "HumanoidRootPart" then
                ev:Disconnect()
                ESP:Add(char, {
                    Name = p.Name,
                    Player = p,
                    PrimaryPart = c
                })
            end
        end)
    else
        ESP:Add(char, {
            Name = p.Name,
            Player = p,
            PrimaryPart = char.HumanoidRootPart
        })
    end
end
local function PlayerAdded(p)
    p.CharacterAdded:Connect(CharAdded)
    if p.Character then
        coroutine.wrap(CharAdded)(p.Character)
    end
end
plrs.PlayerAdded:Connect(PlayerAdded)
for i,v in pairs(plrs:GetPlayers()) do
    if v ~= plr then
        PlayerAdded(v)
    end
end
---//----


    ---//Functions
    for i,v in pairs(workspace.NPCS:GetChildren()) do 
        if v:IsA("Model") then
            table.insert(npctable,v.Name)
        end
    end
    
    local function npctp()
        for i,v in pairs(workspace.NPCS:GetChildren()) do 
            if v.Name == Settings.Selectednpc then
                player.Character:PivotTo(v:GetPivot())
            end
        end
    end



    local function bossfarm()
        if not Settings.BossFarm.Toggle then
            return
        end

        for i,v in pairs(workspace.Living:GetChildren()) do
            if v.Name == Settings.BossFarm.Boss and v:IsA("Model") then
                    if Settings.BossFarm.Position >= 0 then
                        player.Character:PivotTo(v:GetPivot()* CFrame.new(0,Settings.BossFarm.Position ,0)* CFrame.Angles(math.rad(-90), 0, 0))
                        attackremote:InvokeServer("MouseButton1")
                        attackremote:InvokeServer("R")
                    else
                        player.Character:PivotTo(v:GetPivot()* CFrame.new(0,Settings.BossFarm.Position ,0)* CFrame.Angles(math.rad(90), 0, 0))
                        attackremote:InvokeServer("MouseButton1")
                        attackremote:InvokeServer("R")
                    end
            else
                print("No mob?")
            end
        end
    end





    local function MobCheck(mob, toggle)
        if mob == nil or mob:FindFirstChild("HumanoidRootPart") == nil  or toggle ~= true or mob.Humanoid.Health <= 0 or mob.Name ~= "Star Platinum!?!?" or mob:GetAttribute("Level") < Settings.MobFarm.Selectedmob  then
            return nil
        end
        return true
    end
    
    
    local function mobfarm()
        if not Settings.MobFarm.Toggle then
            return
        end
        for i,v in pairs(workspace.Living:GetChildren()) do
            if MobCheck(v, Settings.MobFarm.Toggle) then
                if Settings.MobFarm.Position >= 0 then
                    player.Character.HumanoidRootPart.CFrame = v.HumanoidRootPart.CFrame * CFrame.new(0,Settings.MobFarm.Position ,0)* CFrame.Angles(math.rad(-90), 0, 0)
                    attackremote:InvokeServer("MouseButton1")
                    attackremote:InvokeServer("R")
                else
                    player.Character.HumanoidRootPart.CFrame = v.HumanoidRootPart.CFrame * CFrame.new(0,Settings.MobFarmPosition ,0)* CFrame.Angles(math.rad(90), 0, 0)
                    attackremote:InvokeServer("MouseButton1")
                    attackremote:InvokeServer("R")
                end
            end
        end
    end
    

    local function debrisfarm()
        if not Settings.SandDebrisFarm.Toggle then
            return
        end

        for i,v in pairs(workspace.ItemSpawns["Sand Debris"]:GetDescendants()) do
            if v.Name == "SandDebris" and v:FindFirstChild("ProximityAttachment") then
                player.Character:PivotTo(v:GetPivot())
                task.wait()
                local Magnitude = (player.Character.PrimaryPart.Position - v.Position).Magnitude
                if Magnitude <= 15 then
                    fireproximityprompt(v.ProximityAttachment.Interaction)
                end
            else
                print('No debris found')
            end
        end
    end


    local function chestfarm()
        if not Settings.Chestfarm then
            return
        end
        for i,v in pairs(workspace.ItemSpawns.Chests:GetDescendants()) do
            if v:IsA("Model") and v.Name == "Chest" and v.RootPart:FindFirstChild("ProximityAttachment") then
                player.Character:PivotTo(v:GetPivot())
                local Magnitude = (player.Character.PrimaryPart.Position - v.RootPart.Position).Magnitude
                if Magnitude <= 15 then
                    fireproximityprompt(v.RootPart.ProximityAttachment.Interaction)
                end 
            else
                print("No chest found")
            end
        end
    end

    local function astralchest()
        if not Settings.AstralChestfarm then 
            return
        end
    end

    local function shardsfarm()
        if not Settings.Shardsfarm then 
            return
        end

    end


    local function autodaily()
        if not Settings.Dailyquest.Toggle then
            return
        end
        if player.QuestLines:FindFirstChild("DailyQuest") then
            
        else
            dialogueremote:InvokeServer("Daily Quest")
        end
    end



    local function autostore()
        if not Settings.Autostore then
            return
        end
        for i,v in pairs(player.Backpack:GetChildren()) do
            if v:IsA("Tool") and v.Parent ~= player.Character then
                v.Parent = player.Character
                task.wait(0.2)
                if v.Parent == player.Character then
                        local args = {
                            [1] = {
                                ["AddItems"] = true
                            }   
                        }
                    game:GetService("ReplicatedStorage").ReplicatedModules.KnitPackage.Knit.Services.InventoryService.RE.ItemInventory:FireServer(unpack(args))
                end
            end
        end
    end


    local function autostand()
        if not Settings.AutoFarmstand.Toggle then
            return
        end
        --game:GetService("ReplicatedStorage").ReplicatedModules.KnitPackage.Knit.Services.DialogueService.RF.CheckDialogue:InvokeServer("Reset Stand") -- to reset stand
    end



    local function autoquest()
        if not Settings.Autoquest.Toggle then
            return
        end
        for i,v in pairs(workspace.Living:GetChildren()) do
            if player.QuestLines:FindFirstChild("Sakuya")  then
                if v.Name == "Star Platinum!?!?" and v.Humanoid.Health > 0  then
                    if Settings.MobFarm.Position >= 0 then
                        player.Character.HumanoidRootPart.CFrame = v.HumanoidRootPart.CFrame * CFrame.new(0,Settings.Autoquest.Position,0)* CFrame.Angles(math.rad(-90), 0, 0)
                        attackremote:InvokeServer("MouseButton1")
                        attackremote:InvokeServer("R")
                    else
                        player.Character.HumanoidRootPart.CFrame = v.HumanoidRootPart.CFrame * CFrame.new(0,Settings.Autoquest.Position,0)* CFrame.Angles(math.rad(90), 0, 0)
                        attackremote:InvokeServer("MouseButton1")
                        attackremote:InvokeServer("R")
                    end
                end
            else
                dialogueremote:InvokeServer("Sakuya")
            end
        end
    end

    local function chatlogger()
        loadstring(game:HttpGet(('https://raw.githubusercontent.com/mac2115/Cool-private/main/ESP'),true))()
    end

-- dialogueremote:InvokeServer("Sakuya") -- to get quest ofc

    ---//Ui
    
    local repo = 'https://raw.githubusercontent.com/violin-suzutsuki/LinoriaLib/main/'
    local Library = loadstring(game:HttpGet(repo .. 'Library.lua'))()
    local ThemeManager = loadstring(game:HttpGet(repo .. 'addons/ThemeManager.lua'))()
    local SaveManager = loadstring(game:HttpGet(repo .. 'addons/SaveManager.lua'))()
    local Window = Library:CreateWindow({
        -- Set Center to true if you want the menu to appear in the center
        -- Set AutoShow to true if you want the menu to appear when it is created
        -- Position and Size are also valid options here
        -- but you do not need to define them unless you are changing them :)
        Title = 'Asteria AUT',
        Center = true,
        AutoShow = true,
        TabPadding = 8,
        MenuFadeTime = 0.2
    })
    
    local Tabs = {
        -- Creates a new tab titled Main
        Main = Window:AddTab('Main'),
        ['UI Settings'] = Window:AddTab('UI Settings'),
    }
    
    
    local LeftGroupBox = Tabs.Main:AddLeftGroupbox('MobFarm')
    LeftGroupBox:AddToggle('MyToggle', {
        Text = 'Mob farm toggle',
        Default = false, -- Default value (true / false)
        Tooltip = 'Mobfarm', -- Information shown when you hover over the toggle
    
        Callback = function(Value)
            Settings.MobFarm.Toggle = Value
            mobfarm()
        end
    })
    
    LeftGroupBox:AddDropdown('MyMultiDropdown', {
        Values = mobTable,
        Default = 1,
        Multi = false, -- true / false, allows multiple choices to be selected
        Text = 'Choose a mob (indicates lvl+)',
        Tooltip = 'Mob selector', -- Information shown when you hover over the dropdown
        Callback = function(Value)
            Settings.MobFarm.Selectedmob = Value
        end
    })
    
    
    
    LeftGroupBox:AddSlider('MySlider', {
        Text = 'Farm position',
        Default = 6,
        Min = -15,
        Max = 15,
        Rounding = 1,
        Compact = false,
    
        Callback = function(Value)
            Settings.MobFarm.Position = Value
        end
    })

    LeftGroupBox:AddToggle('MyToggle', {
        Text = 'Bossfarm toggle',
        Default = false, -- Default value (true / false)
        Tooltip = 'Bossfarm', -- Information shown when you hover over the toggle
    
        Callback = function(Value)
            Settings.BossFarm.Toggle = Value
            bossfarm()
        end
    })
    
    LeftGroupBox:AddDropdown('MyMultiDropdown', {
        Values = bossTable,
        Default = 1,
        Multi = false, -- true / false, allows multiple choices to be selected
        Text = 'Choose a boss',
        Tooltip = 'Boss selector', -- Information shown when you hover over the dropdown
        Callback = function(Value)
            Settings.BossFarm.Boss = Value
        end
    })
    
    
    LeftGroupBox:AddSlider('MySlider', {
        Text = 'Farm position',
        Default = 6,
        Min = -15,
        Max = 15,
        Rounding = 1,
        Compact = false,
    
        Callback = function(Value)
            Settings.BossFarm.Position = Value
        end
    })


    local LeftGroupBox = Tabs.Main:AddLeftGroupbox('Autoquests')

    LeftGroupBox:AddToggle('MyToggle', {
        Text = 'Sakuya autoquest toggle',
        Default = false, -- Default value (true / false)
        Tooltip = 'Autoquest', -- Information shown when you hover over the toggle
    
        Callback = function(Value)
            Settings.Autoquest.Toggle= Value
            autoquest()
        end
    })


    LeftGroupBox:AddSlider('MySlider', {
        Text = 'Farm position',
        Default = 6,
        Min = -15,
        Max = 15,
        Rounding = 1,
        Compact = false,
    
        Callback = function(Value)
            Settings.Autoquest.Position = Value
        end
    })

    local LeftGroupBox = Tabs.Main:AddLeftGroupbox('ItemFarms')
    LeftGroupBox:AddToggle('MyToggle', {
        Text = 'Sand Debris farm',
        Default = false, -- Default value (true / false)
        Tooltip = 'Debris farm', -- Information shown when you hover over the toggle
    
        Callback = function(Value)
            Settings.SandDebrisFarm.Toggle = Value
            debrisfarm()
        end
    })

    LeftGroupBox:AddToggle('MyToggle', {
        Text = 'Chest farm',
        Default = false, -- Default value (true / false)
        Tooltip = 'Chest farm', -- Information shown when you hover over the toggle
    
        Callback = function(Value)
            Settings.Chestfarm = Value
            chestfarm()
        end
    })

    LeftGroupBox:AddToggle('MyToggle', {
        Text = 'Autostore items',
        Default = false, -- Default value (true / false)
        Tooltip = 'Stores items for u', -- Information shown when you hover over the toggle
    
        Callback = function(Value)
            Settings.Autostore = Value
            autostore()
        end
    })
    
    local LeftGroupBox = Tabs.Main:AddRightGroupbox('Teleports')
    LeftGroupBox:AddDropdown('MyMultiDropdown', {
        Values = npctable,
        Default = 1,
        Multi = false, -- true / false, allows multiple choices to be selected
        Text = 'Npc Teleports',
        Tooltip = 'Npc teleporter', -- Information shown when you hover over the dropdown
        Callback = function(Value)
            Settings.Selectednpc = Value
            npctp()
        end
    })

    local LeftGroupBox = Tabs.Main:AddRightGroupbox('Misc')
    
    local MyButton = LeftGroupBox:AddButton({
        Text = 'Chatlogger',
        Func = function()
            chatlogger()
        end,
        DoubleClick = false,
        Tooltip = 'Chatlogger'
    })


    local LeftGroupBox = Tabs.Main:AddRightGroupbox('Visuals')
    LeftGroupBox:AddToggle('MyToggle', {
        Text = 'ESP toggle',
        Default = false, -- Default value (true / false)
        Tooltip = 'Esp', -- Information shown when you hover over the toggle
    
        Callback = function(Value)
            ESP.Enabled = Value
        end
    })

    LeftGroupBox:AddLabel('Color'):AddColorPicker('ColorPicker', {
        Default = Color3.fromRGB(255, 0, 0), -- Bright green
        Title = 'ESP color', -- Optional. Allows you to have a custom color picker title (when you open it)
        Transparency = 0, -- Optional. Enables transparency changing for this color picker (leave as nil to disable)
    
        Callback = function(Value)
           ESP.Color = Value
        end
    })
    

    LeftGroupBox:AddToggle('MyToggle', {
        Text = 'Teammates ',
        Default = false, -- Default value (true / false)
        Tooltip = 'Teammates', -- Information shown when you hover over the toggle
    
        Callback = function(Value)
            ESP.TeamMates = Value
        end
    })
   

    LeftGroupBox:AddToggle('MyToggle', {
        Text = 'Nametags toggle',
        Default = false, -- Default value (true / false)
        Tooltip = 'Nametags', -- Information shown when you hover over the toggle
    
        Callback = function(Value)
            ESP.Names = Value
        end
    })
 

    LeftGroupBox:AddToggle('MyToggle', {
        Text = 'Boxes toggle',
        Default = false, -- Default value (true / false)
        Tooltip = 'Boxes', -- Information shown when you hover over the toggle
    
        Callback = function(Value)
            ESP.Boxes = Value
        end
    })
    
    
    --//Configs
    local MenuGroup = Tabs['UI Settings']:AddLeftGroupbox('Menu')
    MenuGroup:AddLabel('Menu bind'):AddKeyPicker('MenuKeybind', { Default = 'End', NoUI = true, Text = 'Menu keybind' })
    MenuGroup:AddButton('Unload', function() Library:Unload() end)
    MenuGroup:AddButton('Copy discord invite', function() setclipboard("https://discord.gg/t2cXFpkGBh") end)
    Library.ToggleKeybind = Options.MenuKeybind
    ThemeManager:SetLibrary(Library)
    SaveManager:SetLibrary(Library)
    SaveManager:IgnoreThemeSettings()
    SaveManager:BuildConfigSection(Tabs['UI Settings'])
    ThemeManager:SetFolder('Asteria/AUT')
    SaveManager:SetFolder('Asteria/AUT')
    ThemeManager:ApplyToTab(Tabs['UI Settings'])
    SaveManager:LoadAutoloadConfig()
    
    
    
    
    
    --//Loops
    game:GetService("RunService").Heartbeat:connect(function()
        bossfarm()
        mobfarm()
        chestfarm()
        debrisfarm()
        autostore()
        autoquest()
        astralchest()
        shardsfarm()
    end)
    


    game:GetService("RunService").RenderStepped:Connect(function()
        cam = workspace.CurrentCamera
        for i,v in (ESP.Enabled and pairs or ipairs)(ESP.Objects) do
            if v.Update then
                local s,e = pcall(v.Update, v)
                if not s then warn("[EU]", e, v.Object:GetFullName()) end
            end
        end
    end)
    return ESP
