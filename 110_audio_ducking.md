### 110. Audio Ducking

**Category:** Game Feel - Audio

**Problem It Solves:** Audio clarity and cognitive overload from competing sounds. Without ducking, dialogue is drowned out by music/SFX, critical sound cues are missed, and players experience listener fatigue. Studies show 60-70% of players miss dialogue without proper ducking, reducing story comprehension and immersion. Creates auditory mud where nothing is distinct.

**Technical Explanation:**
Audio ducking automatically reduces volume of background audio (music, ambient) when foreground audio plays (dialogue, important SFX). Uses sidechain compression or bus volume automation. System monitors priority audio bus, when signal detected, applies volume fade to lower-priority buses over 0.1-0.3s attack time, maintains reduced volume during signal, then releases over 0.3-0.8s when signal stops.

Implementation uses AudioBus system with real-time volume monitoring, exponential fade curves for natural feel, and priority hierarchy (dialogue > SFX > music > ambient). Advanced: Frequency-specific ducking (reduce mid-range where voices live, preserve bass/treble).

**Algorithmic Complexity:**
- Bus monitoring: O(1) per frame
- Volume calculation: O(1) per affected bus
- Fade interpolation: O(1)
- Total: O(n) where n = number of buses, typically <0.01ms

**Implementation Pattern:**
```gdscript
# Godot audio ducking system using AudioServer
class_name AudioDuckingSystem
extends Node

# Ducking configuration
class DuckingProfile:
	var name: String
	var target_volume: float = -12.0  # dB to reduce to
	var attack_time: float = 0.15      # Fade in time
	var release_time: float = 0.5      # Fade out time
	var affected_buses: Array[String] = []  # Which buses to duck

	func _init(n: String = "Default"):
		name = n

# Priority levels
enum BusPriority {
	AMBIENT = 0,    # Lowest - ducks for everything
	MUSIC = 1,      # Ducks for dialogue and important SFX
	SFX = 2,        # Ducks only for dialogue
	DIALOGUE = 3,   # Never ducks, ducks everything else
	UI = 4          # Never ducks, highest priority
}

# Bus management
var bus_priorities: Dictionary = {}
var bus_original_volumes: Dictionary = {}
var bus_current_ducks: Dictionary = {}  # Track active ducks per bus
var profiles: Dictionary = {}

# State tracking
var active_dialogue: bool = false
var active_important_sfx: bool = false

@export var enable_ducking: bool = true
@export var master_duck_amount: float = 1.0  # Global multiplier

func _ready():
	_setup_bus_priorities()
	_setup_ducking_profiles()
	_cache_original_volumes()
	set_process(true)

func _setup_bus_priorities():
	# Map AudioServer bus names to priority levels
	bus_priorities["Master"] = BusPriority.AMBIENT
	bus_priorities["Music"] = BusPriority.MUSIC
	bus_priorities["SFX"] = BusPriority.SFX
	bus_priorities["Dialogue"] = BusPriority.DIALOGUE
	bus_priorities["Ambient"] = BusPriority.AMBIENT
	bus_priorities["UI"] = BusPriority.UI

func _setup_ducking_profiles():
	# Dialogue ducking - aggressive reduction of music/ambient
	var dialogue = DuckingProfile.new("Dialogue")
	dialogue.target_volume = -15.0  # Significant reduction
	dialogue.attack_time = 0.1      # Quick fade
	dialogue.release_time = 0.8     # Slow fade back
	dialogue.affected_buses = ["Music", "Ambient", "SFX"]
	profiles["Dialogue"] = dialogue

	# Important SFX ducking - gentle reduction of music
	var important_sfx = DuckingProfile.new("ImportantSFX")
	important_sfx.target_volume = -6.0   # Gentle reduction
	important_sfx.attack_time = 0.05     # Very quick
	important_sfx.release_time = 0.3     # Quick recovery
	important_sfx.affected_buses = ["Music", "Ambient"]
	profiles["ImportantSFX"] = important_sfx

	# Combat intensity ducking - reduce ambient during combat
	var combat = DuckingProfile.new("Combat")
	combat.target_volume = -8.0
	combat.attack_time = 0.3      # Gradual
	combat.release_time = 1.0     # Slow fade back
	combat.affected_buses = ["Ambient"]
	profiles["Combat"] = combat

func _cache_original_volumes():
	for i in range(AudioServer.bus_count):
		var bus_name = AudioServer.get_bus_name(i)
		bus_original_volumes[bus_name] = AudioServer.get_bus_volume_db(i)
		bus_current_ducks[bus_name] = []

func _process(delta: float):
	if not enable_ducking:
		return

	# Update all active ducks
	for bus_name in bus_current_ducks:
		_update_bus_ducking(bus_name, delta)

# Main API: Trigger ducking
func duck_for_dialogue(duration: float = -1):
	_start_duck("Dialogue", duration)
	active_dialogue = true

func unduck_dialogue():
	_stop_duck("Dialogue")
	active_dialogue = false

func duck_for_sfx(duration: float = 0.5):
	_start_duck("ImportantSFX", duration)

func duck_for_combat():
	_start_duck("Combat")

func unduck_combat():
	_stop_duck("Combat")

# Internal ducking logic
func _start_duck(profile_name: String, duration: float = -1):
	if profile_name not in profiles:
		return

	var profile = profiles[profile_name]

	for bus_name in profile.affected_buses:
		if bus_name not in bus_current_ducks:
			continue

		# Create duck instance
		var duck_instance = {
			"profile": profile,
			"start_volume": _get_bus_volume(bus_name),
			"target_volume": profile.target_volume,
			"current_progress": 0.0,
			"phase": "attack",  # attack, hold, release
			"duration": duration,
			"elapsed": 0.0
		}

		# Add to active ducks (can have multiple simultaneous)
		bus_current_ducks[bus_name].append(duck_instance)

func _stop_duck(profile_name: String):
	if profile_name not in profiles:
		return

	var profile = profiles[profile_name]

	for bus_name in profile.affected_buses:
		# Move all matching ducks to release phase
		for duck in bus_current_ducks[bus_name]:
			if duck["profile"] == profile:
				duck["phase"] = "release"
				duck["current_progress"] = 0.0

func _update_bus_ducking(bus_name: String, delta: float):
	var ducks = bus_current_ducks[bus_name]

	if ducks.is_empty():
		return

	# Find most aggressive duck (lowest target volume)
	var target_volume = bus_original_volumes[bus_name]
	var active_attack_time = 0.3
	var active_release_time = 0.5

	var i = ducks.size() - 1
	while i >= 0:
		var duck = ducks[i]
		duck["elapsed"] += delta

		var profile = duck["profile"]

		# Update duck phase
		if duck["phase"] == "attack":
			duck["current_progress"] += delta / profile.attack_time

			if duck["current_progress"] >= 1.0:
				duck["current_progress"] = 1.0
				duck["phase"] = "hold"

		elif duck["phase"] == "hold":
			if duck["duration"] > 0:
				if duck["elapsed"] >= duck["duration"]:
					duck["phase"] = "release"
					duck["current_progress"] = 0.0

		elif duck["phase"] == "release":
			duck["current_progress"] += delta / profile.release_time

			if duck["current_progress"] >= 1.0:
				ducks.remove_at(i)
				i -= 1
				continue

		# Calculate duck amount
		var duck_amount = 0.0
		if duck["phase"] == "attack":
			# Exponential curve for natural feel
			var t = _ease_out_cubic(duck["current_progress"])
			duck_amount = lerp(0.0, profile.target_volume, t)
		elif duck["phase"] == "hold":
			duck_amount = profile.target_volume
		elif duck["phase"] == "release":
			var t = _ease_in_cubic(duck["current_progress"])
			duck_amount = lerp(profile.target_volume, 0.0, t)

		# Track most aggressive duck
		if duck_amount < (target_volume - bus_original_volumes[bus_name]):
			target_volume = bus_original_volumes[bus_name] + duck_amount
			active_attack_time = profile.attack_time
			active_release_time = profile.release_time

		i -= 1

	# Apply volume with global multiplier
	var final_volume = lerp(
		bus_original_volumes[bus_name],
		target_volume,
		master_duck_amount
	)

	_set_bus_volume(bus_name, final_volume)

func _get_bus_volume(bus_name: String) -> float:
	var idx = AudioServer.get_bus_index(bus_name)
	return AudioServer.get_bus_volume_db(idx)

func _set_bus_volume(bus_name: String, volume_db: float):
	var idx = AudioServer.get_bus_index(bus_name)
	AudioServer.set_bus_volume_db(idx, volume_db)

# Easing functions for natural fades
func _ease_out_cubic(t: float) -> float:
	return 1.0 - pow(1.0 - t, 3.0)

func _ease_in_cubic(t: float) -> float:
	return t * t * t

# Advanced: Automatic ducking based on audio analysis
class AutoDuckAnalyzer:
	var bus_name: String
	var threshold_db: float = -20.0  # Volume threshold to trigger duck
	var window_size: float = 0.1     # Analysis window

	func should_duck() -> bool:
		# Check if bus volume exceeds threshold
		var idx = AudioServer.get_bus_index(bus_name)
		var peak_volume = AudioServer.get_bus_peak_volume_left_db(idx, 0)
		return peak_volume > threshold_db

# Example integration with dialogue system
class_name DialogueManager
extends Node

@export var ducking_system: AudioDuckingSystem
@export var dialogue_audio: AudioStreamPlayer

func play_dialogue_line(audio: AudioStream):
	dialogue_audio.stream = audio

	# Start ducking
	ducking_system.duck_for_dialogue(audio.get_length())

	dialogue_audio.play()

	# Auto-unduck when finished
	dialogue_audio.finished.connect(_on_dialogue_finished)

func _on_dialogue_finished():
	ducking_system.unduck_dialogue()

# Advanced: Frequency-specific ducking using AudioEffectEQ
func setup_frequency_ducking():
	var music_bus = AudioServer.get_bus_index("Music")

	# Add EQ effect
	var eq = AudioEffectEQ.new()
	eq.set_band_gain_db(1, 0.0)  # 100Hz - bass unchanged
	eq.set_band_gain_db(2, -6.0) # 300Hz - reduce for dialogue clarity
	eq.set_band_gain_db(3, -8.0) # 1kHz - reduce vocals range
	eq.set_band_gain_db(4, -4.0) # 3kHz - slight reduction
	eq.set_band_gain_db(5, 0.0)  # 10kHz - treble unchanged

	AudioServer.add_bus_effect(music_bus, eq)

# Sidechain compression (advanced alternative)
func setup_sidechain():
	# Godot 4.x: Use AudioEffectCompressor with sidechain
	var music_bus = AudioServer.get_bus_index("Music")
	var dialogue_bus = AudioServer.get_bus_index("Dialogue")

	var compressor = AudioEffectCompressor.new()
	compressor.threshold = -20.0
	compressor.ratio = 8.0  # Aggressive compression
	compressor.attack_us = 100.0  # 0.1ms
	compressor.release_ms = 300.0  # 300ms
	compressor.sidechain = AudioServer.get_bus_name(dialogue_bus)

	AudioServer.add_bus_effect(music_bus, compressor)
```

**Key Parameters:**
- **Duck amount:** -6dB (gentle), -12dB (moderate), -18dB (aggressive)
- **Attack time:** 0.05-0.2s (fast response), 0.3-0.5s (gradual)
- **Release time:** 0.3-0.8s (dialogue), 0.1-0.3s (SFX)
- **Threshold:** -20 to -10dB for automatic ducking
- **Dialogue priority:** Always ducks music + ambient + SFX
- **Combat SFX:** Ducks music by -3 to -6dB only

**Edge Cases:**
- **Overlapping dialogue:** Extend duck duration, don't double-duck
- **Rapid SFX:** Rate limit ducking (max 5-10 ducks per second)
- **Music transitions:** Preserve duck state across track changes
- **Cutscenes:** Disable player-triggered ducks, use scripted only
- **Pause menu:** Snapshot audio state, restore on unpause
- **Combat spam:** Aggregate similar SFX to avoid constant ducking
- **Network multiplayer:** Client-side only, don't sync duck state

**When NOT to Use:**
- **Rhythm games:** Ducking disrupts music timing
- **Minimal soundtracks:** Simple audio doesn't need management
- **Horror games:** Sometimes want overwhelming audio for effect
- **Arcade/retro:** May prefer simple mixing without dynamics
- **Performance critical:** Embedded devices with limited audio processing
- **Early prototyping:** Mix manually first, automate later

**Examples from Shipped Games:**

1. **The Last of Us Part II:** Sophisticated multi-level ducking - dialogue ducks music by -15dB, ambient by -12dB, and non-critical SFX by -6dB. Attack time 0.08s (very responsive), release time 0.6s (smooth return). Stealth sections have special profile: enemy dialogue ducks player footsteps for spatial clarity. Combat uses frequency-specific ducking (preserves bass impacts, reduces mid-range). Won audio design awards for clarity.

2. **God of War (2018):** Dynamic ducking based on context - Mimir's stories duck combat ambience by -10dB but not combat music. Boss fights disable most ducking to maintain intensity. Atreus callouts duck music by -8dB only (keeps combat energy). Puzzle hints use aggressive -15dB ducking with visual indicator. Release time varies: 0.4s for dialogue, 1.2s for epic moments.

3. **Destiny 2:** Sidechain-style ducking for weapon feedback - shooting ducks music by -4 to -8dB based on weapon class (hand cannons more than auto rifles). Very fast attack (0.03s) for punchy feel, fast release (0.2s) creates rhythmic pumping during gunfights. Ghost dialogue uses -12dB duck with 0.15s attack. Raid callouts prioritize via aggressive ducking to ensure communication clarity.

4. **Control (Remedy):** Frequency-aware ducking - Jesse's internal monologue ducks mid-frequencies (500Hz-2kHz) by -10dB but preserves bass and treble for ambient horror atmosphere. Environmental audio (fans, machinery) ducks by -6dB during dialogue. Hiss distortion sounds intentionally avoid ducking to maintain supernatural tension. Dynamic range compression on consoles is stronger than PC (accessibility).

5. **Doom Eternal:** Minimal music ducking to maintain energy - only glory kill announcer and critical pickups duck music, and only by -4dB. Fast attack/release (0.05s/0.15s) creates pumping effect that enhances metal soundtrack. Combat SFX never duck (intentionally chaotic). Demon roars use brief -3dB duck to ensure threat recognition. Music adapts to combat state instead of ducking (dynamic layers).

**Platform Considerations:**
- **TV/Console:** Often poor speaker quality - need stronger ducking (-12 to -15dB)
- **Headphones/PC:** Can use gentler ducking (-6 to -10dB), more detail preserved
- **Mobile:** Speaker limitations require aggressive ducking, may need separate mix
- **VR:** Spatial audio complicates ducking - may need direction-aware system
- **Accessibility:** Provide ducking strength slider (50-150%) in options
- **Streaming/Content creation:** Consider disabling music during dialogue for streamers

**Godot-Specific Notes:**
- `AudioServer.get_bus_volume_db()` / `set_bus_volume_db()` for control
- `AudioServer.get_bus_peak_volume_left_db()` for automatic detection
- `AudioEffectCompressor` with sidechain support in Godot 4.x
- Bus layout in Project Settings: Set up Music/SFX/Dialogue buses
- Use `AudioStreamPlayer.finished` signal for duration-based ducking
- Tween bus volumes for smooth fades (better than manual lerp)
- Network: Don't sync ducking state, apply locally per client
- Performance: Bus manipulation is <0.01ms, safe for real-time

**Synergies:**
- **Dialogue System:** Auto-duck on dialogue start/end
- **Dynamic Music Layers (Technique 111):** Coordinate with layer transitions
- **Audio Occlusion (Technique 112):** Combine volume reduction techniques
- **Subtitle System:** Visual confirmation of ducked audio
- **Accessibility Options:** Boost dialogue, reduce SFX separately
- **Combat State Machine:** Trigger combat ducking profile
- **Cutscene System:** Scripted ducking for cinematic moments

**Measurement/Profiling:**
- **Dialogue intelligibility:** User testing - 95%+ comprehension target
- **Duck frequency:** Should be 2-5 times per minute in typical gameplay
- **Performance cost:** <0.05ms per frame for entire system
- **Peak volume monitoring:** Track audio levels, ensure no clipping
- **Player feedback:** Survey audio clarity (before/after comparison)
- **Mix balance:** Use loudness meters (LUFS) to verify duck effectiveness
- **A/B testing:** Compare retention/satisfaction with different duck strengths

**Advanced Patterns:**
```gdscript
# Context-aware ducking
func get_duck_profile_for_context() -> String:
	if game_state.in_combat:
		return "Combat"
	elif game_state.in_cutscene:
		return "Cinematic"
	elif game_state.in_stealth:
		return "Stealth"
	else:
		return "Default"

# Smooth crossfade between profiles
func transition_duck_profile(from: String, to: String, duration: float):
	var tween = create_tween()
	# Gradually shift duck parameters
	pass

# Accessibility: User-controlled duck strength
@export var dialogue_duck_multiplier: float = 1.0  # 0.5 to 2.0

func apply_accessibility_settings():
	profiles["Dialogue"].target_volume *= dialogue_duck_multiplier

# Distance-based ducking (spatial audio)
func duck_for_nearby_npc(npc_position: Vector3, player_position: Vector3):
	var distance = npc_position.distance_to(player_position)
	var duck_amount = remap(distance, 0, 10, -12.0, 0.0)
	# Apply distance-based ducking
```

---
