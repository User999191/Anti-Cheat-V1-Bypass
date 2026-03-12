-- added cframe / Excessive Lag Bypass

local function createAntiKick()
    local Players = game:GetService("Players")
    local lp = Players.LocalPlayer
    local RS = game:GetService("ReplicatedStorage")
    local SS = game:GetService("ServerScriptService")
    
    local config = {
        cooldownBase = 0.35,
        cooldownJitter = 0.03,
        adonisNames = {"Adonis", "Adonis_Main", "Adonis_Client", "AdonisData", "Adonis_"},
        maxCFrameDistPerSec = 150,
        checkInterval = 0.4
    }
    
    workspace:WaitForChild("Terrain", 15)
    
    local hasAdonis = _G.Adonis
    if not hasAdonis then
        local services = {SS, RS, game:GetService("ServerStorage")}
        for _, serv in services do
            for _, name in config.adonisNames do
                if serv:FindFirstChild(name) then
                    hasAdonis = true
                    break
                end
            end
            if hasAdonis then break end
        end
    end
    
    if hasAdonis then
        return function() end
    end
    
    local state = {
        lastKickAttempt = 0,
        lastPos = nil,
        lastPosTime = 0
    }
    
    local function isOnCooldown()
        local now = tick()
        local effective = config.cooldownBase + (math.random() * 2 - 1) * config.cooldownJitter
        if now - state.lastKickAttempt < effective then
            return true
        end
        state.lastKickAttempt = now
        return false
    end
    
    local function checkSuspiciousHRP()
        if lp.Character then
            local hrp = lp.Character:FindFirstChild("HumanoidRootPart")
            if hrp and (hrp.Name == "Anti Cheat" or hrp.Name == "Anti_Cheat") then
                return true
            end
        end
        return false
    end
    
    local function checkSpeedJumpCFrame()
        if not lp.Character then return false end
        local hrp = lp.Character:FindFirstChild("HumanoidRootPart")
        local hum = lp.Character:FindFirstChildOfClass("Humanoid")
        if not hrp or not hum then return false end
        
        local now = tick()
        
        if hum.WalkSpeed > 500 or hum.JumpPower > 500 then
            return true
        end
        
        if state.lastPos and now - state.lastPosTime < 1.5 then
            local dist = (hrp.Position - state.lastPos).Magnitude
            local timeDiff = now - state.lastPosTime
            local speed = dist / timeDiff
            if speed > config.maxCFrameDistPerSec then
                return true
            end
        end
        
        state.lastPos = hrp.Position
        state.lastPosTime = now
        return false
    end
    
    spawn(function()
        while true do
            task.wait(config.checkInterval)
            if checkSuspiciousHRP() or checkSpeedJumpCFrame() then
                if not isOnCooldown() then
                end
            end
        end
    end)
    
    local oldNamecall
    oldNamecall = hookmetamethod(game, "__namecall", newcclosure(function(self, ...)
        local method = getnamecallmethod()
        
        if self == lp and method == "Kick" then
            if isOnCooldown() then return end
            return
        end
        
        if method == "FireServer" or method == "InvokeServer" then
            if typeof(self) == "Instance" and (self:IsA("RemoteEvent") or self:IsA("RemoteFunction")) then
                if self.Parent == RS or self:IsDescendantOf(RS) then
                    if isOnCooldown() then return end
                    return
                end
            end
        end
        
        return oldNamecall(self, ...)
    end))
    
    local oldIndex
    oldIndex = hookmetamethod(game, "__index", newcclosure(function(self, key)
        if self == lp and key:lower() == "kick" then
            return function(...)
                if isOnCooldown() then return end
            end
        end
        return oldIndex(self, key)
    end))
    
    return function()
        hookmetamethod(game, "__namecall", oldNamecall)
        hookmetamethod(game, "__index", oldIndex)
    end
end
