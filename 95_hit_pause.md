### 95. Hit Pause / Freeze Frames

**Category:** Game Feel - Visual Feedback

**Problem It Solves:** Lack of impact on attacks. Players feel disconnected from combat actions. Hits feel weightless and unimportant. Without momentary pause, fast-paced combat becomes visually overwhelming and attacks blend together, reducing player satisfaction and making it harder to read game state.

**Technical Explanation:**
Temporarily pause or slow game time for 1-10 frames when significant events occur (hits, deaths, parries). Implemented by setting `Engine.time_scale` to 0.0 (freeze) or 0.1-0.5 (slow-motion) for a brief duration. The pause creates emphasis through temporal contrast - the sudden stop draws attention and gives the brain time to process the impact. Critical for creating "weight" and "crunch" in combat. Works by exploiting the psychological principle that temporal discontinuities capture attention more than continuous motion.

**Algorithmic Complexity:**
- O(1) to trigger pause (single variable assignment)
- O(n) to exclude UI/VFX from pause (n = excluded nodes)
- Minimal overhead: ~0.01ms per frame during pause

**Implementation Pattern:**
```gdscript
# Godot implementation - HitPauseManager
class_name HitPauseManager
extends Node

var is_paused: bool = false
var pause_timer: float = 0.0
var normal_time_scale: float = 1.0

# Different pause lengths for different events
const PAUSE_LIGHT_HIT = 0.05    # 3 frames at 60fps
const PAUSE_HEAVY_HIT = 0.12    # 7 frames at 60fps
const PAUSE_CRITICAL = 0.20     # 12 frames at 60fps
const PAUSE_DEATH = 0.25        # 15 frames at 60fps
const PAUSE_PARRY = 0.15        # 9 frames at 60fps

# Nodes excluded from time scaling (UI, particles, VFX)
var excluded_nodes: Array[Node] = []

func _ready():
	# Set to run during pause
	process_mode = Node.PROCESS_MODE_ALWAYS

func _process(delta):
	if is_paused:
		pause_timer -= delta  # Uses real delta, not scaled
		if pause_timer <= 0.0:
			_end_pause()

func trigger_pause(duration: float, slow_motion: bool = false):
	# Cancel existing pause if retriggered
	if is_paused:
		_end_pause()

	is_paused = true
	pause_timer = duration

	if slow_motion:
		Engine.time_scale = 0.1  # 10% speed
	else:
		Engine.time_scale = 0.0  # Complete freeze

	# Emit signal for VFX/audio systems
	hit_pause_started.emit(duration)

func _end_pause():
	is_paused = false
	pause_timer = 0.0
	Engine.time_scale = normal_time_scale
	hit_pause_ended.emit()

# Exclude specific nodes from time scaling
func exclude_from_pause(node: Node):
	if node not in excluded_nodes:
		excluded_nodes.append(node)
		node.process_mode = Node.PROCESS_MODE_ALWAYS

# Usage in combat system
func _on_player_hit_enemy(damage: float, is_critical: bool):
	if is_critical:
		HitPauseManager.trigger_pause(PAUSE_CRITICAL)
	elif damage > 50:
		HitPauseManager.trigger_pause(PAUSE_HEAVY_HIT)
	else:
		HitPauseManager.trigger_pause(PAUSE_LIGHT_HIT)
```

**Key Parameters:**
- **Light hits:** 2-4 frames (0.033-0.066s at 60fps)
- **Heavy hits:** 5-8 frames (0.083-0.133s at 60fps)
- **Critical hits:** 10-15 frames (0.166-0.25s at 60fps)
- **Death/Finisher:** 12-20 frames (0.2-0.333s at 60fps)
- **Parry/Counter:** 8-12 frames (0.133-0.2s at 60fps)
- **Slow-motion alternative:** 0.05-0.2 time scale for 0.1-0.3s

**Edge Cases:**
- **Rapid hits:** Limit pause frequency (max 1 per 0.1s) to prevent stuttering
- **Multiplayer:** Server authoritative - client predicts but syncs to server
- **UI interaction:** Always exclude UI nodes from time scaling
- **Particle systems:** May want to continue updating during pause for visual consistency
- **Audio:** Pause audio or apply pitch shift effect, never let audio desync
- **Online lag:** Can feel worse with network latency - reduce duration by 30-50%

**When NOT to Use:**
- **Puzzle games:** Breaks concentration and pacing
- **Racing games:** Interrupts flow and sense of speed
- **Rhythm games:** Destroys timing precision
- **Frequent weak hits:** Every bullet hitting creates nausea-inducing stutter
- **Accessibility concerns:** Some players experience motion sickness from frequent pauses
- **Multiplayer fighting games:** Only use in single-player modes unless both players freeze

**Examples from Shipped Games:**

1. **Celeste:** 3-frame freeze on dash impacts, 5-frame on death. Minimal but extremely effective. Contributes to the game's "crunchy" feel without disrupting precise platforming. Death pause signals state reset clearly.

2. **Enter the Gungeon:** 4-6 frame pause on blank activation (dodge roll special). Creates satisfying "impact" for defensive ability. No pause on normal shots (would be overwhelming with bullet density).

3. **Dead Cells:** 2-3 frame freeze on parry, 8-10 frames on critical hits, 15 frames on boss phase transitions. Heavy weapons get 5-7 frame pause, daggers get 1-2 frames. Scaled by weapon weight creates clear weapon identity.

4. **Hollow Knight:** 6-8 frame pause on nail strikes hitting enemies, 12 frames on killing blow, 20 frames on boss deaths. Combined with screen shake and particle burst. Contributes to combat satisfaction despite simple mechanics.

5. **Hyper Light Drifter:** 4-5 frame pause on sword hits, 10-12 frames on gun shots, 15-20 frames on chain dash impacts. One of the most "juicy" indie games, pause system is core to the feel.

**Platform Considerations:**
- **60fps target:** Design pauses in frame counts (3, 5, 8, 12 frames)
- **30fps games:** Reduce frame counts by 50% (3 frames → 1-2 frames)
- **Variable refresh:** Use time-based (seconds) instead of frame counts
- **Mobile:** Reduce duration by 20-30% - lower frame rates make pauses feel longer
- **Switch/Console:** Works great at locked 60fps, test at 30fps fallback
- **VR:** NEVER USE - causes severe motion sickness

**Godot-Specific Notes:**
- Use `Engine.time_scale` for global pause, affects all delta times
- Set pause-immune nodes to `PROCESS_MODE_ALWAYS`
- Physics continues at reduced rate - can cause collision issues
- Use `get_process_delta_time()` to get unscaled delta for UI
- Consider pausing audio with `AudioServer.set_bus_volume_db()` fade
- Particles: Set `process_material.time_scale` separately or use `local_coords = true`
- Godot 4.x: `Engine.time_scale` works identically to 3.x

**Synergies:**
- **Screen Shake (Technique 97):** Combine pause with shake for maximum impact
- **Particle Effects (Technique 98):** Spawn particle burst at pause start
- **Hit Confirmation (Technique 106):** Pause + hitmarker + sound + rumble = complete feedback
- **Damage Scaling (Technique 102):** Longer pause for higher damage values
- **Squash & Stretch (Technique 96):** Pause holds squashed frame longer for emphasis
- **Chromatic Aberration (Technique 99):** Flash effect during pause frame

**Measurement/Profiling:**
- **Playtest feedback:** "Does this feel impactful?" vs "Is this stuttering?"
- **Frame timing:** Monitor actual pause duration matches intended (±1 frame tolerance)
- **Frequency tracking:** Log pause events per second - should be <10/sec
- **A/B testing:** Record player sessions with/without pause, compare engagement
- **Performance:** Negligible CPU cost (<0.01ms), no GPU impact
- **User settings:** Consider toggle option for accessibility (some players dislike)
- **Metrics:** Track player retention in combat sections with/without feature

**Advanced Techniques:**
```gdscript
# Stacking pause system - longest pause wins
var pause_stack: Array = []

func trigger_pause_stacked(duration: float):
	pause_stack.append(duration)
	pause_stack.sort()
	pause_timer = pause_stack.back()  # Use longest

# Gradual resume for smoother feel
func _end_pause_smooth():
	var tween = create_tween()
	tween.tween_property(Engine, "time_scale", 1.0, 0.05)

# Per-object time scaling (more complex but selective)
class EntityTimeScale:
	var time_multiplier: float = 1.0

	func _process(delta):
		var scaled_delta = delta * time_multiplier
		# Use scaled_delta for movement/animations
```
