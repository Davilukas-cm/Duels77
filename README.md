if _G.BloodHoundsRunning then return end
_G.BloodHoundsRunning = true

do
	local function runIntro()
		local Players          = game:GetService("Players")
		local TweenService     = game:GetService("TweenService")
		local Lighting         = game:GetService("Lighting")
		local UserInputService = game:GetService("UserInputService")
		local TextService      = game:GetService("TextService")
		local RunService       = game:GetService("RunService")

		local player    = Players.LocalPlayer
		local playerGui = player:WaitForChild("PlayerGui")

		-- ===== CONFIG =====
		local TITLE        = "BLESSED HUB"
		local SOUND_ID     = 120267378058133
		local SKIP_SECONDS = 10           
		local TOTAL        = 5.4  

		local WHITE = Color3.fromRGB(255, 255, 255)
		local GREY  = Color3.fromRGB(235, 235, 235)
		local BLACK = Color3.fromRGB(18, 4, 14)
		local TITLE_FONT = Enum.Font.FredokaOne 

		local PINK = Color3.fromRGB(255, 75, 175)
		local ICE  = Color3.fromRGB(255, 165, 215)
		local IRIS = ColorSequence.new({
			ColorSequenceKeypoint.new(0, ICE),
			ColorSequenceKeypoint.new(0.35, WHITE),
			ColorSequenceKeypoint.new(0.65, PINK),
			ColorSequenceKeypoint.new(1, ICE),
		})
		local SPARK = { WHITE, ICE, PINK }

		local camera = workspace.CurrentCamera
		local vp = (camera and camera.ViewportSize) or Vector2.new(1280, 720)

		-- ===== HELPERS =====
		local function make(class, props, parent)
			local obj = Instance.new(class)
			for k, v in pairs(props) do obj[k] = v end
			obj.Parent = parent
			return obj
		end

		local function tween(obj, time, goal, style, dir)
			local t = TweenService:Create(
				obj,
				TweenInfo.new(time, style or Enum.EasingStyle.Quad, dir or Enum.EasingDirection.Out),
				goal
			)
			t:Play()
			return t
		end

		local function measure(txt, size, font)
			local ok, b = pcall(function()
				return TextService:GetTextSize(txt, size, font, Vector2.new(10000, 10000))
			end)
			if ok and b then return b.X, b.Y end
			return size * 0.66 * #txt, size * 1.2
		end

		-- ===== STATE / CANCELLATION =====
		local alive, skipped = true, false
		local CANCEL = {}

		-- Yields for t seconds; throws CANCEL the moment the intro is skipped.
		local function sleep(t)
			local t0 = os.clock()
			while os.clock() - t0 < t do
				if skipped then error(CANCEL, 0) end
				RunService.Heartbeat:Wait()
			end
			if skipped then error(CANCEL, 0) end
		end

		-- ===== ROOT =====
		local gui = make("ScreenGui", {
			Name = "RedHubIntro",
			IgnoreGuiInset = true,
			ResetOnSpawn = false,
			ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
			DisplayOrder = 999,
		}, playerGui)

		local blur = make("BlurEffect", { Name = "RedHubIntroBlur", Size = 0 }, Lighting)

		-- Stage: one CanvasGroup so the whole scene fades as a single piece
		local stage = make("CanvasGroup", {
			AnchorPoint = Vector2.new(0.5, 0.5),
			Position = UDim2.fromScale(0.5, 0.5),
			Size = UDim2.fromScale(1, 1),
			BackgroundTransparency = 1,
			BorderSizePixel = 0,
			GroupTransparency = 0,
		}, gui)

		-- ===== BACKGROUND =====
		local bg = make("Frame", {
			Size = UDim2.fromScale(1, 1), BackgroundColor3 = BLACK,
			BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 1,
		}, stage)

		local vignette = make("ImageLabel", {
			Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1,
			Image = "rbxassetid://195611797", ImageColor3 = BLACK,
			ImageTransparency = 1, ZIndex = 2,
		}, stage)

		-- soft layered glow behind the title
		local glowScale = math.clamp(vp.Y / 720, 0.6, 1.5)
		local glows = {}
		for i, d in ipairs({ 560, 820, 1100 }) do
			local g = make("Frame", {
				AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
				Size = UDim2.fromOffset(0, 0), BackgroundColor3 = WHITE,
				BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 2,
			}, stage)
			make("UICorner", { CornerRadius = UDim.new(1, 0) }, g)
			glows[i] = { frame = g, size = math.floor(d * glowScale), alpha = 0.955 + i * 0.01 }
		end

		-- ===== BUBBLE FACTORY (glassy sphere: rainbow rim + two highlights) =====
		local function makeBubble(parent, d, fillAlpha, strokeAlpha, zindex)
			local b = make("Frame", {
				AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
				Size = UDim2.fromOffset(d, d), BackgroundColor3 = WHITE,
				BackgroundTransparency = fillAlpha, BorderSizePixel = 0, ZIndex = zindex,
			}, parent)
			make("UICorner", { CornerRadius = UDim.new(1, 0) }, b)
			local st = make("UIStroke", { Color = WHITE, Thickness = 2, Transparency = strokeAlpha }, b)
			local gr = make("UIGradient", { Color = IRIS, Rotation = 0 }, st)
			local hi1 = make("Frame", {
				Position = UDim2.fromScale(0.17, 0.14), Size = UDim2.fromScale(0.3, 0.16),
				BackgroundColor3 = WHITE, BackgroundTransparency = 0.3, BorderSizePixel = 0,
				Rotation = -35, ZIndex = zindex,
			}, b)
			make("UICorner", { CornerRadius = UDim.new(1, 0) }, hi1)
			local hi2 = make("Frame", {
				Position = UDim2.fromScale(0.68, 0.7), Size = UDim2.fromScale(0.1, 0.1),
				BackgroundColor3 = WHITE, BackgroundTransparency = 0.5, BorderSizePixel = 0,
				ZIndex = zindex,
			}, b)
			make("UICorner", { CornerRadius = UDim.new(1, 0) }, hi2)
			return { frame = b, stroke = st, grad = gr, hi1 = hi1, hi2 = hi2 }
		end

		-- ===== AMBIENT BUBBLES (drift upward behind the title) =====
		local ambient = {}
		for i = 1, 18 do
			local d = math.random(16, 72)
			local bb = makeBubble(stage, d, 1, 1, 3)
			bb.hi1.BackgroundTransparency = 1
			bb.hi2.BackgroundTransparency = 1
			bb.x0 = math.random()
			bb.y = math.random()
			bb.vy = 0.03 + math.random() * 0.06
			bb.ph = math.random() * 6.28
			bb.sway = 0.01 + math.random() * 0.02
			bb.frame.Position = UDim2.fromScale(bb.x0, bb.y)
			ambient[i] = bb
		end

		-- ===== TITLE LAYOUT =====
		local titleSize = math.clamp(math.floor(math.min(vp.X * 0.095, vp.Y * 0.17)), 34, 110)
		local rowH = math.floor(titleSize * 1.2)
		local rowY = -15
		local titlePad = math.max(1, math.floor(titleSize * 0.03))
		local shadowOff = math.max(3, math.floor(titleSize * 0.06))
		local D = math.floor(titleSize * 1.4)           -- bubble diameter

		local widths, totalW = {}, 0
		for i = 1, #TITLE do
			local ch = TITLE:sub(i, i)
			local w = measure(ch, titleSize, TITLE_FONT)
			if ch == " " then w = math.max(w, titleSize * 0.34) end
			widths[i] = w
			totalW = totalW + w
		end
		totalW = totalW + titlePad * (#TITLE - 1)

		local row = make("Frame", {
			AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.new(0.5, 0, 0.5, rowY),
			Size = UDim2.fromOffset(totalW, rowH), BackgroundTransparency = 1, ZIndex = 6,
		}, stage)
		make("UIListLayout", {
			FillDirection = Enum.FillDirection.Horizontal,
			HorizontalAlignment = Enum.HorizontalAlignment.Left,
			VerticalAlignment = Enum.VerticalAlignment.Center,
			SortOrder = Enum.SortOrder.LayoutOrder,
			Padding = UDim.new(0, titlePad),
		}, row)

		-- one holder per letter:  holder (moves/rotates)
		--                           ├─ inner (jelly scale): shadow + glow + letter
		--                           └─ bubble (soap bubble around it)
		local letters = {}
		for i = 1, #TITLE do
			local ch = TITLE:sub(i, i)
			local slot = make("Frame", {
				Size = UDim2.fromOffset(widths[i], rowH), BackgroundTransparency = 1,
				LayoutOrder = i, ZIndex = 6,
			}, row)
			if ch ~= " " then
				local holder = make("Frame", {
					AnchorPoint = Vector2.new(0.5, 0.5),
					Position = UDim2.new(0.5, math.random(-60, 60), 0.5, math.floor(vp.Y * 0.6)),
					Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, ZIndex = 6,
				}, slot)
				local inner = make("Frame", {
					AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromScale(0.5, 0.5),
					Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, ZIndex = 2,
				}, holder)
				local lsc = make("UIScale", { Scale = 0.6 }, inner)

				local function txt(props)
					props.Size = UDim2.fromScale(1, 1)
					props.BackgroundTransparency = 1
					props.Text = ch
					props.Font = TITLE_FONT
					props.TextSize = titleSize
					props.TextTransparency = 1
					return make("TextLabel", props, inner)
				end

				local shadow = txt({ Position = UDim2.fromOffset(0, shadowOff), TextColor3 = BLACK, ZIndex = 1 })
				local glow = txt({ TextColor3 = WHITE, ZIndex = 2 })
				make("UIScale", { Scale = 1.05 }, glow)
				local glowStroke = make("UIStroke", {
					Color = WHITE, Thickness = 9, Transparency = 1,
					ApplyStrokeMode = Enum.ApplyStrokeMode.Contextual,
				}, glow)
				local lbl = txt({ TextColor3 = WHITE, ZIndex = 3, TextTransparency = 0.08 })
				make("UIStroke", {
					Color = Color3.fromRGB(16, 75, 42), Thickness = 3, Transparency = 0.05,
					ApplyStrokeMode = Enum.ApplyStrokeMode.Contextual,
				}, lbl)
				make("UIGradient", {
					Rotation = 90,
					Color = ColorSequence.new({
						ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 65, 85)),
						ColorSequenceKeypoint.new(0.48, Color3.fromRGB(255, 225, 115)),
						ColorSequenceKeypoint.new(1, Color3.fromRGB(70, 235, 135)),
					}),
				}, lbl)

				-- Christmas-light bulbs frame each animated title letter.
				-- These are decorative only; the original letter animation remains unchanged.
				local bulbColors = {
					Color3.fromRGB(255, 55, 75),
					Color3.fromRGB(70, 235, 125),
					Color3.fromRGB(255, 220, 90),
					Color3.fromRGB(90, 190, 255),
				}
				for bulbIndex = 1, 6 do
					local x = (bulbIndex - 0.5) / 6
					for _, y in ipairs({ 0.06, 0.94 }) do
						local bulb = make("Frame", {
							AnchorPoint = Vector2.new(0.5, 0.5),
							Position = UDim2.fromScale(x, y),
							Size = UDim2.fromOffset(5, 5),
							BackgroundColor3 = bulbColors[(bulbIndex + (y > 0.5 and 1 or 0) - 1) % #bulbColors + 1],
							BorderSizePixel = 0, ZIndex = 8,
						}, holder)
						make("UICorner", { CornerRadius = UDim.new(1, 0) }, bulb)
						make("UIStroke", { Color = WHITE, Thickness = 0.7, Transparency = 0.15 }, bulb)
					end
				end

				local bub = makeBubble(holder, D, 0.92, 0.65, 3)

				letters[#letters + 1] = {
					holder = holder, inner = inner, lsc = lsc, shadow = shadow, glow = glow,
					glowStroke = glowStroke, lbl = lbl, bub = bub,
					state = "wait", ph = math.random() * 6.28, idleT = 0,
				}
			end
		end

		-- ===== SKIP HINT (big pill at the bottom, pulses so it is easy to see) =====
		local HINT_TEXT = "TAP ANYWHERE TO SKIP"
		local hintSize = math.clamp(math.floor(titleSize * 0.2), 15, 20)
		local hintTextW = measure(HINT_TEXT, hintSize, Enum.Font.FredokaOne)
		local hintPill = make("Frame", {
			AnchorPoint = Vector2.new(0.5, 1), Position = UDim2.new(0.5, 0, 1, -22),
			Size = UDim2.fromOffset(math.floor(hintTextW + 48), hintSize + 22),
			BackgroundColor3 = BLACK, BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 10,
		}, stage)
		make("UICorner", { CornerRadius = UDim.new(1, 0) }, hintPill)
		local hintStroke = make("UIStroke", {
			Color = WHITE, Thickness = 1.5, Transparency = 1,
			ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
		}, hintPill)
		local hintScale = make("UIScale", { Scale = 1 }, hintPill)
		local hint = make("TextLabel", {
			Size = UDim2.fromScale(1, 1), BackgroundTransparency = 1, Text = HINT_TEXT,
			TextColor3 = WHITE, Font = Enum.Font.FredokaOne, TextSize = hintSize,
			TextTransparency = 1, ZIndex = 11,
		}, hintPill)

		-- ===== FLASH =====
		local flash = make("Frame", {
			Size = UDim2.fromScale(1, 1), BackgroundColor3 = WHITE,
			BackgroundTransparency = 1, BorderSizePixel = 0, ZIndex = 20,
		}, stage)
		local function doFlash(alpha, dur)
			flash.BackgroundTransparency = 1 - alpha
			tween(flash, dur or 0.35, { BackgroundTransparency = 1 })
		end

		-- ===== AUDIO =====
		local sound = make("Sound", {
			Name = "IntroSound", SoundId = "rbxassetid://" .. SOUND_ID, Volume = 0.7,
		}, gui)
		task.spawn(function()
			local t0 = os.clock()
			while alive and not sound.IsLoaded and os.clock() - t0 < 4 do task.wait(0.05) end
			if not alive or skipped then return end
			pcall(function()
				sound.TimePosition = SKIP_SECONDS
				sound:Play()
			end)
		end)

		-- ===== PER-FRAME UPDATES =====
		local pConn = RunService.Heartbeat:Connect(function(dt)
			local now = os.clock()

			-- ambient bubbles drift up and sway; their rim colours slowly rotate
			for _, b in ipairs(ambient) do
				b.y = b.y - b.vy * dt
				if b.y < -0.12 then b.y = 1.12; b.x0 = math.random() end
				b.frame.Position = UDim2.fromScale(b.x0 + math.sin(now * 0.9 + b.ph) * b.sway, b.y)
				b.grad.Rotation = (now * 50 + b.ph * 60) % 360
			end

			-- letters: wobble while rising, gentle jelly-bob once popped
			for i, L in ipairs(letters) do
				if L.state == "rise" then
					local k = math.sin(now * 9 + L.ph) * 0.07
					L.bub.frame.Size = UDim2.fromOffset(D * (1 + k), D * (1 - k))
					L.bub.grad.Rotation = (now * 120 + L.ph * 40) % 360
					L.holder.Rotation = math.sin(now * 5 + L.ph) * 9
				elseif L.state == "idle" then
					local amp = math.clamp((now - L.idleT) / 0.8, 0, 1)
					L.holder.Position = UDim2.new(0.5, 0, 0.5, math.sin(now * 2.4 + i * 0.6) * 4 * amp)
					L.holder.Rotation = math.sin(now * 1.8 + i * 0.7) * 2.2 * amp
				end
			end
		end)

		-- ===== SKIP =====
		local skipConn = UserInputService.InputBegan:Connect(function(input, gameProcessed)
			if gameProcessed then return end
			local t = input.UserInputType
			if t == Enum.UserInputType.Touch or t == Enum.UserInputType.MouseButton1 then
				skipped = true
			end
		end)

		local function cleanup()
			alive = false
			if pConn then pConn:Disconnect() end
			if skipConn then skipConn:Disconnect() end
			pcall(function() sound:Stop() end)
			gui:Destroy()
			blur:Destroy()
		end

		-- ===== EFFECT HELPERS =====
		-- pop burst at the centre of a GUI object: ring + sparkle droplets
		local function popFX(obj, d)
			local c = obj.AbsolutePosition + obj.AbsoluteSize / 2
			local ring = make("Frame", {
				AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromOffset(c.X, c.Y),
				Size = UDim2.fromOffset(d * 0.8, d * 0.8), BackgroundTransparency = 1,
				BorderSizePixel = 0, ZIndex = 12,
			}, stage)
			make("UICorner", { CornerRadius = UDim.new(1, 0) }, ring)
			local rs = make("UIStroke", { Color = WHITE, Thickness = 2.5, Transparency = 0.15 }, ring)
			make("UIGradient", { Color = IRIS }, rs)
			tween(ring, 0.45, { Size = UDim2.fromOffset(d * 1.9, d * 1.9) }, Enum.EasingStyle.Quint)
			tween(rs, 0.45, { Transparency = 1 })
			task.delay(0.5, function() ring:Destroy() end)

			for i = 1, 9 do
				local ang = (i / 9) * math.pi * 2 + math.random() * 0.5
				local dist = d * (0.75 + math.random() * 0.55)
				local sz = math.random(4, 9)
				local drop = make("Frame", {
					AnchorPoint = Vector2.new(0.5, 0.5), Position = UDim2.fromOffset(c.X, c.Y),
					Size = UDim2.fromOffset(sz, sz), BackgroundColor3 = SPARK[math.random(#SPARK)],
					BackgroundTransparency = 0.15, BorderSizePixel = 0, ZIndex = 12,
				}, stage)
				make("UICorner", { CornerRadius = UDim.new(1, 0) }, drop)
				local dur = 0.4 + math.random() * 0.25
				tween(drop, dur, {
					Position = UDim2.fromOffset(c.X + math.cos(ang) * dist, c.Y + math.sin(ang) * dist),
					Size = UDim2.fromOffset(sz * 0.4, sz * 0.4),
					BackgroundTransparency = 1,
				})
				task.delay(dur + 0.05, function() drop:Destroy() end)
			end
		end

		-- the letter's bubble pops and the letter inflates with a jelly bounce
		local function popLetter(L)
			L.state = "pop"
			popFX(L.holder, D)
			local b = L.bub
			tween(b.frame, 0.14, { Size = UDim2.fromOffset(D * 1.3, D * 1.3), BackgroundTransparency = 1 })
			tween(b.stroke, 0.14, { Transparency = 1 })
			tween(b.hi1, 0.1, { BackgroundTransparency = 1 })
			tween(b.hi2, 0.1, { BackgroundTransparency = 1 })
			task.delay(0.18, function() b.frame:Destroy() end)

			tween(L.holder, 0.7, { Rotation = 0 }, Enum.EasingStyle.Elastic)
			tween(L.lbl, 0.15, { TextTransparency = 0 })
			tween(L.shadow, 0.3, { TextTransparency = 0.5 })
			tween(L.glow, 0.4, { TextTransparency = 0.88 })
			tween(L.glowStroke, 0.4, { Transparency = 0.94 })
			tween(L.lsc, 0.8, { Scale = 1 }, Enum.EasingStyle.Elastic)

			task.delay(0.85, function()
				if not alive then return end
				L.state = "idle"
				L.idleT = os.clock()
			end)
		end

		-- jelly wave: each letter squishes up and bounces back, one after another
		local function jelly(step)
			for i, L in ipairs(letters) do
				task.delay((i - 1) * step, function()
					if not alive then return end
					tween(L.lsc, 0.1, { Scale = 1.22 })
					task.delay(0.1, function()
						if alive then tween(L.lsc, 0.6, { Scale = 1 }, Enum.EasingStyle.Elastic) end
					end)
				end)
			end
		end

		-- a burst of fresh bubbles rushing up the screen
		local function bubbleBurst(count)
			for i = 1, count do
				local d = math.random(18, 64)
				local x = math.random()
				local bb = makeBubble(stage, d, 0.9, 0.25, 9)
				bb.frame.Position = UDim2.new(x, 0, 1.1, 0)
				local dur = 1.1 + math.random() * 0.9
				task.delay(math.random() * 0.35, function()
					if not alive then return end
					tween(bb.frame, dur, {
						Position = UDim2.new(x + (math.random() - 0.5) * 0.12, 0, -0.15, 0),
					}, Enum.EasingStyle.Sine)
					task.delay(dur + 0.05, function() bb.frame:Destroy() end)
				end)
			end
		end

		-- ===== MAIN SEQUENCE =====
		local function sequence()
			local t0 = os.clock()

			-- 1) atmosphere in
			tween(blur, 0.9, { Size = 16 })
			tween(bg, 0.7, { BackgroundTransparency = 0.22 })
			tween(vignette, 0.9, { ImageTransparency = 0.3 })
			tween(hintPill, 0.6, { BackgroundTransparency = 0.35 })
			tween(hintStroke, 0.6, { Transparency = 0.2 })
			tween(hint, 0.6, { TextTransparency = 0 })
			for _, g in ipairs(glows) do
				tween(g.frame, 1.4, {
					Size = UDim2.fromOffset(g.size, g.size),
					BackgroundTransparency = g.alpha,
				}, Enum.EasingStyle.Quint)
			end
			for _, b in ipairs(ambient) do
				tween(b.frame, 1, { BackgroundTransparency = 0.94 })
				tween(b.stroke, 1, { Transparency = 0.45 })
				tween(b.hi1, 1, { BackgroundTransparency = 0.5 })
				tween(b.hi2, 1, { BackgroundTransparency = 0.65 })
			end
			sleep(0.4)

			-- 2) each letter rises in its own bubble, then pops
			for i, L in ipairs(letters) do
				task.spawn(function()
					task.wait((i - 1) * 0.11)
					if not alive then return end
					L.state = "rise"
					local dur = 0.8 + math.random() * 0.2
					tween(L.holder, dur, { Position = UDim2.new(0.5, 0, 0.5, 0) }, Enum.EasingStyle.Sine)
					task.wait(dur)
					if not alive then return end
					popLetter(L)
				end)
			end
			sleep((#letters - 1) * 0.11 + 1.0 + 0.75)

			-- 3) everything is in: flash, jelly wave, bubbles rush up
			doFlash(0.25, 0.4)
			jelly(0.05)
			bubbleBurst(16)

			-- skip pill pulses (glow + gentle size pulse) so it stays noticeable
			local pulse = TweenInfo.new(0.8, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true)
			TweenService:Create(hintStroke, pulse, { Transparency = 0.7 }):Play()
			TweenService:Create(hintScale, pulse, { Scale = 1.07 }):Play()
			TweenService:Create(hint, pulse, { TextTransparency = 0.25 }):Play()

			-- 4) hold: letters bob, second jelly wave
			sleep(1.3)
			jelly(0.05)
			local remain = TOTAL - (os.clock() - t0)
			if remain > 0 then sleep(remain) end

			sleep(0.3)

			-- 5) outro: letters pop away one by one
			for i, L in ipairs(letters) do
				task.delay((i - 1) * 0.05, function()
					if not alive then return end
					L.state = "out"
					popFX(L.holder, D)
					tween(L.lsc, 0.25, { Scale = 1.5 })
					tween(L.lbl, 0.25, { TextTransparency = 1 })
					tween(L.shadow, 0.25, { TextTransparency = 1 })
					tween(L.glow, 0.25, { TextTransparency = 1 })
					tween(L.glowStroke, 0.25, { Transparency = 1 })
				end)
			end
			tween(sound, 0.9, { Volume = 0 }, Enum.EasingStyle.Linear)
			sleep((#letters - 1) * 0.05 + 0.4)
			tween(stage, 0.45, { GroupTransparency = 1 })
			tween(blur, 0.45, { Size = 0 })
			sleep(0.5)
		end

		local ok, err = pcall(sequence)
		if not ok and err ~= CANCEL then
			warn("[RedHub intro] " .. tostring(err))
		e
