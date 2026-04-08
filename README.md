# appliance-hiddendevs-scripting



-- services
local RunService    = game:GetService("RunService")
local Players       = game:GetService("Players")
local RS            = game:GetService("ReplicatedStorage")
local TweenService  = game:GetService("TweenService")

-- remotes
local Remotes           = RS:WaitForChild("Remotes"):WaitForChild("Characters"):WaitForChild("Gojo"):WaitForChild("Infinity Run")
local InfinityRunRemote = Remotes:WaitForChild("Infinity Run Remote")
local DashBegin         = Remotes:WaitForChild("DashBegin")
local DashStarted       = Remotes:WaitForChild("DashStarted")
local StartVFX          = Remotes:WaitForChild("StartVFX")
local StopVFX           = Remotes:WaitForChild("StopVFX")

-- assets
local VFXTemplate = RS
	:WaitForChild("VFX")
	:WaitForChild("Characters")
	:WaitForChild("Gojo")
	:WaitForChild("Infinity Run")
	:WaitForChild("Infinity RunVFX")

local InfinityRunAnim = RS
	:WaitForChild("Animations")
	:WaitForChild("Characters")
	:WaitForChild("Gojo")
	:WaitForChild("Infinity Run")
	:WaitForChild("Infinity Run Anim")

local Config = {
	Cooldown              = 0.6,
	StartVelocity         = 15,   -- slow windup speed during the first second before full dash kicks in
	IntervalVelocity      = 80,   -- full dash speed after IntervalStart
	IntervalStart         = 1,    -- seconds before full speed + hitbox + vfx activate
	Duration              = 5.25, -- total dash duration in seconds

	NormalWalkSpeed       = 16,
	NormalJumpHeight      = 7.2,

	HitboxSize            = Vector3.new(6.5, 7.0, 8.0),
	HitboxOffset          = CFrame.new(0, 0, -5),

	VFXPredictDelta       = 0.045, -- how far ahead to place vfx based on velocity
	VFXLocalOffset        = CFrame.new(-12, -5, 0) * CFrame.Angles(0, math.rad(180), 0),

	DefaultFOV            = 70,
	DashFOV               = 120,

	MaxHitDistance        = 30, -- server-side anti-exploit distance check

	FrontDragOffset       = 4.5, -- how far in front of the attacker the target is pinned
	DragResponsiveness    = 200,
}

local InfinityRun = {}
InfinityRun.__index = InfinityRun

-- ─── client ──────────────────────────────────────────────────────────────────

function InfinityRun.initClient()
	if not RunService:IsClient() then return end

	local player    = Players.LocalPlayer
	local char      = player.Character or player.CharacterAdded:Wait()
	local canDash   = true
	local isDashing = false

	-- tracks active vfx models per player so we can clean them up properly
	local activeClientVFX = {}

	local function enableVFX(model)
		for _, v in pairs(model:GetDescendants()) do
			if v:IsA("Beam") or v:IsA("ParticleEmitter") or v:IsA("PointLight") then
				v.Enabled = true
			end
		end
	end

	local function disableVFX(model)
		for _, v in pairs(model:GetDescendants()) do
			if v:IsA("Beam") or v:IsA("ParticleEmitter") or v:IsA("PointLight") then
				v.Enabled = false
				if v:IsA("ParticleEmitter") then v:Clear() end
			end
		end
	end

	-- slightly ahead of the hrp to avoid the vfx lagging behind at high speed
	local function getVFXCFrame(hrp)
		local vel       = hrp.AssemblyLinearVelocity
		local predicted = hrp.Position + vel * Config.VFXPredictDelta
		local lookDir   = hrp.CFrame.LookVector
		local baseCF    = CFrame.new(predicted, predicted + lookDir)
		return baseCF * Config.VFXLocalOffset
	end

	local function startClientVFX(targetPlayer)
		-- tear down any existing vfx for this player before spawning a new one
		local existing = activeClientVFX[targetPlayer]
		if existing then
			if existing.conn then existing.conn:Disconnect() end
			pcall(function() disableVFX(existing.model) end)
			pcall(function() existing.model:Destroy() end)
			activeClientVFX[targetPlayer] = nil
		end

		local targetChar = targetPlayer.Character
		if not targetChar then
			local loaded = false
			local cx = targetPlayer.CharacterAdded:Connect(function(c)
				targetChar = c
				loaded     = true
			end)
			local t = tick()
			repeat task.wait() until loaded or (tick() - t > 2)
			cx:Disconnect()
			if not targetChar then return end
		end

		local hrp = targetChar:FindFirstChild("HumanoidRootPart")
		if not hrp then return end

		local clone = VFXTemplate:Clone()
		clone.Parent = workspace
		clone:PivotTo(getVFXCFrame(hrp))

		for _, v in pairs(clone:GetDescendants()) do
			if v:IsA("BasePart") then
				v.Anchored   = true
				v.CanCollide = false
			end
		end
		enableVFX(clone)

		-- update the vfx position every frame so it tracks the player smoothly
		local conn
		conn = RunService.RenderStepped:Connect(function()
			local currentChar = targetPlayer.Character
			if not currentChar then return end
			local currentHRP = currentChar:FindFirstChild("HumanoidRootPart")
			if not currentHRP then return end
			if not clone or not clone.Parent then
				conn:Disconnect()
				activeClientVFX[targetPlayer] = nil
				return
			end
			clone:PivotTo(getVFXCFrame(currentHRP))
		end)

		activeClientVFX[targetPlayer] = { model = clone, conn = conn }
	end

	local function stopClientVFX(targetPlayer)
		local entry = activeClientVFX[targetPlayer]
		if not entry then return end

		if entry.conn then entry.conn:Disconnect() end
		activeClientVFX[targetPlayer] = nil

		local model = entry.model
		pcall(function()
			for _, v in pairs(model:GetDescendants()) do
				if v:IsA("Beam") or v:IsA("ParticleEmitter") or v:IsA("PointLight") then
					-- disable first so particles fade out naturally instead of popping
					v.Enabled = false
				end
			end
		end)
		task.delay(1, function()
			pcall(function() model:Destroy() end)
		end)
	end

	-- vfx is triggered by the server once phase 2 begins (after IntervalStart)
	StartVFX.OnClientEvent:Connect(function(targetPlayer)
		startClientVFX(targetPlayer)
	end)

	StopVFX.OnClientEvent:Connect(function(targetPlayer)
		stopClientVFX(targetPlayer)
	end)

	Players.PlayerRemoving:Connect(function(leavingPlayer)
		stopClientVFX(leavingPlayer)
	end)

	local function tweenFOV(targetFOV, tweenTime)
		local cam  = workspace.CurrentCamera
		local info = TweenInfo.new(tweenTime, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
		TweenService:Create(cam, info, { FieldOfView = targetFOV }):Play()
	end

	-- only blocks the dash if the player is actively mid-aerial (aerialBP present)
	-- allows dashing during freefall after an aerial combo has already ended
	local function isServerLocked()
		local hrp = char:FindFirstChild("HumanoidRootPart")
		local inAerial = hrp and hrp:FindFirstChild("AerialBP") ~= nil
		if inAerial then
			if char:GetAttribute("Stunned")    then return true end
			if char:GetAttribute("ActionLock") then return true end
		end
		return false
	end

	local function getAnimator()
		local hum = char:FindFirstChildOfClass("Humanoid")
		if not hum then return nil, nil end
		return hum:FindFirstChildOfClass("Animator"), hum
	end

	local ACTION_PRIORITIES = {
		[Enum.AnimationPriority.Action]  = true,
		[Enum.AnimationPriority.Action2] = true,
		[Enum.AnimationPriority.Action3] = true,
		[Enum.AnimationPriority.Action4] = true,
	}

	-- clears any currently playing action-priority animations before playing the dash anim
	local function stopAllActionAnimations(fadeTime)
		local animator = getAnimator()
		if not animator then return end
		for _, track in ipairs(animator:GetPlayingAnimationTracks()) do
			if ACTION_PRIORITIES[track.Priority] then
				track:Stop(fadeTime or 0)
			end
		end
	end

	-- flattens camera look to a horizontal direction, falls back to hrp or world forward
	local function getCameraForward()
		local cam     = workspace.CurrentCamera
		local forward = Vector3.new(cam.CFrame.LookVector.X, 0, cam.CFrame.LookVector.Z)
		if forward.Magnitude > 0.001 then return forward.Unit end
		local hrp = char:FindFirstChild("HumanoidRootPart")
		if hrp then
			local look = Vector3.new(hrp.CFrame.LookVector.X, 0, hrp.CFrame.LookVector.Z)
			if look.Magnitude > 0.001 then return look.Unit end
		end
		return Vector3.new(0, 0, -1)
	end

	local function makeBodyVelocity(hrp, vel)
		local old = hrp:FindFirstChild("DashBV")
		if old then old:Destroy() end
		local bv    = Instance.new("BodyVelocity")
		bv.Name     = "DashBV"
		bv.Velocity = vel
		bv.MaxForce = Vector3.new(math.huge, 0, math.huge)
		bv.P        = 1e5
		bv.Parent   = hrp
		return bv
	end

	local function cleanBodyVelocity(hrp)
		local bv = hrp:FindFirstChild("DashBV")
		if bv then bv:Destroy() end
	end

	local function resetMovement(humanoid, oldSpeed, oldJump)
		if not humanoid or not humanoid.Parent then return end
		humanoid.WalkSpeed = oldSpeed or Config.NormalWalkSpeed
		humanoid.JumpPower = oldJump  or Config.NormalJumpHeight
		humanoid:ChangeState(Enum.HumanoidStateType.Running)
		-- bounce animate script to force idle/walk blend tree to reset cleanly
		local animate = char:FindFirstChild("Animate")
		if animate then
			animate.Disabled = true
			task.defer(function()
				if animate and animate.Parent then animate.Disabled = false end
			end)
		end
	end

	local function getCharacterFromPart(part)
		local model = part:FindFirstAncestorOfClass("Model")
		if model and model:FindFirstChildOfClass("Humanoid") then return model end
	end

	-- welded invisible part in front of the hrp used to detect targets
	local function buildFrontHitbox(hrp)
		local old = char:FindFirstChild("DashHitbox_Front")
		if old then old:Destroy() end

		local hitbox        = Instance.new("Part")
		hitbox.Name         = "DashHitbox_Front"
		hitbox.Anchored     = false
		hitbox.CanCollide   = false
		hitbox.CanQuery     = true
		hitbox.Transparency = 1
		hitbox.BrickColor   = BrickColor.new("Bright blue")
		hitbox.Size         = Config.HitboxSize
		hitbox.Massless     = true
		hitbox.Parent       = char

		local weld  = Instance.new("Weld")
		weld.Part0  = hrp
		weld.Part1  = hitbox
		weld.C0     = Config.HitboxOffset
		weld.Parent = hitbox

		return hitbox
	end

	-- polls the hitbox every heartbeat and fires the server when a new target is inside
	local function startFrontHitboxQuery(hitbox)
		local alreadyHit = {}
		local conn
		conn = RunService.Heartbeat:Connect(function()
			if not hitbox or not hitbox.Parent then
				conn:Disconnect()
				return
			end
			local params = OverlapParams.new()
			params.FilterDescendantsInstances = { char }
			params.FilterType                 = Enum.RaycastFilterType.Blacklist

			for _, part in ipairs(workspace:GetPartsInPart(hitbox, params)) do
				local target = getCharacterFromPart(part)
				if target and target ~= char then
					local hum = target:FindFirstChildOfClass("Humanoid")
					if hum and hum.Health > 0 and not alreadyHit[target] then
						alreadyHit[target] = true
						InfinityRunRemote:FireServer("DashHitbox_Front", target)
					end
				end
			end
		end)
		return conn
	end

	local function destroyFrontHitbox(hitboxPart, hitboxConn)
		if hitboxConn  then hitboxConn:Disconnect() end
		if hitboxPart and hitboxPart.Parent then hitboxPart:Destroy() end
		local leftover = char:FindFirstChild("DashHitbox_Front")
		if leftover then leftover:Destroy() end
	end

	local function startDash()
		if not canDash      then return end
		if isDashing        then return end
		if isServerLocked() then return end

		local animator, humanoid = getAnimator()
		local hrp = char:FindFirstChild("HumanoidRootPart")
		if not humanoid or not hrp or not animator then return end

		-- clear any hover force left over from a previous aerial move
		local hover = hrp:FindFirstChild("AirTPHoverBV")
		if hover then hover:Destroy() end

		-- destroy aerialBP locally so the dash starts cleanly even if server hasn't cleared it yet
		local oldAerial = hrp:FindFirstChild("AerialBP")
		if oldAerial then oldAerial:Destroy() end

		canDash   = false
		isDashing = true

		DashBegin:FireServer()

		-- fov ramps up immediately, then returns near the end of the dash
		tweenFOV(Config.DashFOV, 0.35)
		task.delay(Config.Duration - 0.5, function()
			tweenFOV(Config.DefaultFOV, 0.5)
		end)

		local oldSpeed = humanoid.WalkSpeed
		local oldJump  = humanoid.JumpPower
		humanoid.WalkSpeed = 0
		humanoid.JumpPower = 0

		stopAllActionAnimations(0.05)
		local track = animator:LoadAnimation(InfinityRunAnim)
		track.Priority = Enum.AnimationPriority.Action2
		track:Play(0.05)

		local initialDir = getCameraForward()
		hrp.CFrame       = CFrame.new(hrp.Position, hrp.Position + initialDir)
		local bv         = makeBodyVelocity(hrp, initialDir * Config.StartVelocity)

		local hitboxPart    = nil
		local hitboxConn    = nil
		local dashStartTime = tick()
		local phase         = 1 -- phase 1 = slow windup, phase 2 = full speed with hitbox and vfx

		local steerConn
		steerConn = RunService.Heartbeat:Connect(function()
			if not isDashing then
				steerConn:Disconnect()
				return
			end
			if not bv or not bv.Parent then
				steerConn:Disconnect()
				return
			end

			local elapsed = tick() - dashStartTime
			local dir     = getCameraForward()
			hrp.CFrame    = CFrame.new(hrp.Position, hrp.Position + dir)

			-- transition to phase 2: enable hitbox and notify server (which fires vfx)
			if phase == 1 and elapsed >= Config.IntervalStart then
				phase = 2
				DashStarted:FireServer()
				hitboxPart = buildFrontHitbox(hrp)
				hitboxConn = startFrontHitboxQuery(hitboxPart)
			end

			-- velocity is always applied — never stopped mid-dash
			bv.Velocity = dir * (phase == 1 and Config.StartVelocity or Config.IntervalVelocity)
		end)

		task.delay(Config.Duration, function()
			isDashing = false

			if steerConn then steerConn:Disconnect() end
			destroyFrontHitbox(hitboxPart, hitboxConn)
			cleanBodyVelocity(hrp)

			track:Stop(0.2)
			resetMovement(humanoid, oldSpeed, oldJump)

			task.delay(Config.Cooldown, function()
				canDash = true
			end)
		end)
	end

	-- reassign char reference on respawn so all closures stay valid
	player.CharacterAdded:Connect(function(newChar)
		char      = newChar
		canDash   = true
		isDashing = false
	end)

	return {
		startDash = startDash,
	}
end

-- ─── server ──────────────────────────────────────────────────────────────────

function InfinityRun.initServer()
	if not RunService:IsServer() then return end

	local dashEndTimes = {} -- player → absolute tick() when their dash ends
	local hitRegistry  = {} -- player → set of characters already hit this dash

	-- returns how many seconds remain in the attacker's current dash
	local function getRemainingDash(player)
		local endTime = dashEndTimes[player]
		if not endTime then return 0 end
		return math.max(endTime - tick(), 0)
	end

	-- pins the target in front of the attacker for the full remaining dash duration
	local function startServerDrag(attackerChar, targetChar, duration, attackerPlayer)
		local attackerHRP = attackerChar:FindFirstChild("HumanoidRootPart")
		local targetHRP   = targetChar:FindFirstChild("HumanoidRootPart")
		if not attackerHRP or not targetHRP then return end

		-- give attacker network ownership so the drag position is accurate on their client
		pcall(function()
			targetHRP:SetNetworkOwner(attackerPlayer)
		end)

		local att0  = Instance.new("Attachment")
		att0.Name   = "DragAtt0"
		att0.Parent = targetHRP

		local att1          = Instance.new("Attachment")
		att1.Name           = "DragAtt1"
		att1.Position       = Vector3.new(0, 0, -Config.FrontDragOffset)
		att1.Parent         = attackerHRP

		local alignPos              = Instance.new("AlignPosition")
		alignPos.Name               = "DragAlignPos"
		alignPos.Attachment0        = att0
		alignPos.Attachment1        = att1
		alignPos.MaxForce           = math.huge
		alignPos.Responsiveness     = Config.DragResponsiveness
		alignPos.RigidityEnabled    = false
		alignPos.Parent             = targetHRP

		local alignOri              = Instance.new("AlignOrientation")
		alignOri.Name               = "DragAlignOri"
		alignOri.Attachment0        = att0
		alignOri.Attachment1        = att1
		alignOri.MaxTorque          = math.huge
		alignOri.Responsiveness     = Config.DragResponsiveness
		alignOri.Parent             = targetHRP

		local hum = targetChar:FindFirstChildOfClass("Humanoid")
		if hum then
			hum.WalkSpeed = 0
			hum.JumpPower = 0
		end

		-- clean up all drag constraints and restore movement at the end of the drag
		task.delay(duration, function()
			for _, obj in ipairs({ alignPos, alignOri, att0, att1 }) do
				if obj and obj.Parent then obj:Destroy() end
			end
			pcall(function()
				if targetHRP and targetHRP.Parent then
					targetHRP:SetNetworkOwner(nil)
				end
			end)
			if hum and hum.Parent then
				hum.WalkSpeed = Config.NormalWalkSpeed
				hum.JumpPower = Config.NormalJumpHeight
			end
		end)
	end

	-- client fired this when the dash input was accepted
	-- clears any stale locks from previous moves so the dash always starts cleanly
	DashBegin.OnServerEvent:Connect(function(player)
		local char = player.Character
		if not char then return end

		char:SetAttribute("Stunned",    false)
		char:SetAttribute("ActionLock", false)
		char:SetAttribute("CancelDash", false)
		char:SetAttribute("ActionLock", true)

		-- record the exact tick when this dash ends so getRemainingDash stays accurate
		dashEndTimes[player] = tick() + Config.Duration

		task.delay(Config.Duration, function()
			if char and char.Parent then
				char:SetAttribute("ActionLock", false)
			end
			StopVFX:FireAllClients(player)
			dashEndTimes[player] = nil
		end)
	end)

	-- client entered phase 2 (full speed) — open hit registry and fire vfx for all clients
	DashStarted.OnServerEvent:Connect(function(player)
		hitRegistry[player] = {}
		StartVFX:FireAllClients(player)
	end)

	-- client hitbox detected a character — server validates and starts the drag
	InfinityRunRemote.OnServerEvent:Connect(function(player, hitboxName, targetCharacter)
		if player.Character:GetAttribute("Stunned")    then return end
		if hitboxName ~= "DashHitbox_Front"            then return end
		if typeof(targetCharacter) ~= "Instance"       then return end
		if not targetCharacter:IsA("Model")            then return end

		local attackerChar = player.Character
		if not attackerChar then return end

		local attackerHRP = attackerChar:FindFirstChild("HumanoidRootPart")
		local targetHRP   = targetCharacter:FindFirstChild("HumanoidRootPart")
		local targetHum   = targetCharacter:FindFirstChildOfClass("Humanoid")

		if not attackerHRP or not targetHRP or not targetHum then return end
		if targetHum.Health <= 0                             then return end

		local isRagdoll = targetCharacter:FindFirstChild("IsRagdoll")
		if isRagdoll and isRagdoll.Value then return end

		-- basic distance sanity check to reject teleport exploits
		if (attackerHRP.Position - targetHRP.Position).Magnitude > Config.MaxHitDistance then return end

		if not dashEndTimes[player] then return end

		local registry = hitRegistry[player]
		if not registry              then return end
		if registry[targetCharacter] then return end
		registry[targetCharacter] = true

		targetHum:TakeDamage(0)

		-- drag lasts for the full remaining duration no matter when during the dash the hit lands
		local remaining = getRemainingDash(player)

		targetCharacter:SetAttribute("Stunned", true)
		task.delay(remaining, function()
			if targetCharacter and targetCharacter.Parent then
				targetCharacter:SetAttribute("Stunned", false)
			end
		end)

		startServerDrag(attackerChar, targetCharacter, remaining, player)
	end)

	local function cleanupPlayer(player)
		dashEndTimes[player] = nil
		hitRegistry[player]  = nil
		StopVFX:FireAllClients(player)
	end

	Players.PlayerRemoving:Connect(cleanupPlayer)

	Players.PlayerAdded:Connect(function(player)
		player.CharacterRemoving:Connect(function()
			cleanupPlayer(player)
		end)
	end)
end

return InfinityRun
