### 112. Audio Occlusion

**Category:** Game Feel - Audio

**Problem It Solves:** Unrealistic spatial audio where sounds pass through walls at full volume, destroying immersion and spatial awareness. Players lose ability to judge enemy position/distance when audio ignores geometry. Studies show 70-80% of players unconsciously notice audio clipping through walls, reducing perceived quality and realism. Breaks suspension of disbelief in horror/stealth games where audio positioning is critical.

**Technical Explanation:**
Audio occlusion system raycasts from listener to audio source, detects geometry intersection, then applies low-pass filter and volume reduction based on material thickness/density. Direct path blocked = muffled sound. Uses physics layers to distinguish occluding geometry (walls) from non-occluding (grass, water). Advanced implementation calculates occlusion factor (0-1) from material properties, applies frequency-dependent filtering (high frequencies attenuate faster than low), and considers indirect sound paths (portals, reverb).

Implementation performs periodic raycasts (10-30Hz, not per-frame), caches results with timeout, applies AudioEffectLowPassFilter with cutoff frequency based on occlusion, and adjusts volume. Optimization: Use spatial hashing to only check nearby sound sources, skip checks for quiet/distant sounds.

**Algorithmic Complexity:**
- Raycast per sound: O(log n) with BVH spatial partitioning
- Sound source culling: O(k) where k = nearby sources
- Filter application: O(1) per source
- Total: O(k log n), typically 0.1-0.5ms for 20-30 active sources

**Implementation Pattern:**
```gdscript
# Godot audio occlusion system
class_name AudioOcclusionSystem
extends Node3D

# Material occlusion properties
class OcclusionMaterial:
	var material_name: String
	var occlusion_factor: float  # 0 = no occlusion, 1 = complete block
	var low_freq_pass: float     # % of low freq that passes (0-1)
	var high_freq_pass: float    # % of high freq that passes (0-1)
	var density: float           # Thickness equivalent

# Sound source tracking
class OccludedSound:
	var audio_player: AudioStreamPlayer3D
	var last_check_time: float = 0.0
	var current_occlusion: float = 0.0
	var target_occlusion: float = 0.0
	var base_volume_db: float = 0.0
	var lpf_effect: AudioEffectLowPassFilter

@export var listener: Node3D  # Usually the Camera3D
@export var check_frequency: float = 20.0  # Hz
@export var max_check_distance: float = 50.0  # Don't check beyond this
@export var min_volume_to_check: float = -40.0  # Skip quiet sounds

# Material definitions
var materials: Dictionary = {}

# Active sound sources
var tracked_sounds: Array[OccludedSound] = []

var time_accumulator: float = 0.0

func _ready():
	_setup_materials()
	set_physics_process(true)

func _setup_materials():
	# Wood - moderate occlusion
	var wood = OcclusionMaterial.new()
	wood.material_name = "wood"
	wood.occlusion_factor = 0.6
	wood.low_freq_pass = 0.5      # Bass passes better
	wood.high_freq_pass = 0.2     # Highs blocked
	wood.density = 0.05           # 5cm equivalent
	materials["wood"] = wood

	# Concrete - heavy occlusion
	var concrete = OcclusionMaterial.new()
	concrete.material_name = "concrete"
	concrete.occlusion_factor = 0.85
	concrete.low_freq_pass = 0.3
	concrete.high_freq_pass = 0.05
	concrete.density = 0.3  # 30cm thick
	materials["concrete"] = concrete

	# Metal - complete occlusion
	var metal = OcclusionMaterial.new()
	metal.material_name = "metal"
	metal.occlusion_factor = 0.95
	metal.low_freq_pass = 0.2
	metal.high_freq_pass = 0.01
	metal.density = 0.1
	materials["metal"] = metal

	# Glass - minimal occlusion
	var glass = OcclusionMaterial.new()
	glass.material_name = "glass"
	glass.occlusion_factor = 0.3
	glass.low_freq_pass = 0.8
	glass.high_freq_pass = 0.7
	glass.density = 0.01
	materials["glass"] = glass

	# Fabric/curtain - frequency dependent
	var fabric = OcclusionMaterial.new()
	fabric.material_name = "fabric"
	fabric.occlusion_factor = 0.4
	fabric.low_freq_pass = 0.7
	fabric.high_freq_pass = 0.3
	fabric.density = 0.02
	materials["fabric"] = fabric

func _physics_process(delta: float):
	time_accumulator += delta

	var check_interval = 1.0 / check_frequency

	if time_accumulator >= check_interval:
		time_accumulator = 0.0
		_update_occlusion_checks()

	# Always update smooth transitions
	_update_occlusion_filters(delta)

func _update_occlusion_checks():
	for sound in tracked_sounds:
		if not is_instance_valid(sound.audio_player):
			continue

		# Skip if too quiet or too far
		if sound.audio_player.volume_db < min_volume_to_check:
			continue

		var distance = listener.global_position.distance_to(
			sound.audio_player.global_position
		)

		if distance > max_check_distance:
			continue

		# Perform occlusion check
		var occlusion = _check_occlusion(
			listener.global_position,
			sound.audio_player.global_position
		)

		sound.target_occlusion = occlusion

func _check_occlusion(from: Vector3, to: Vector3) -> float:
	var space_state = get_world_3d().direct_space_state

	# Raycast from listener to sound source
	var query = PhysicsRayQueryParameters3D.create(from, to)
	query.collision_mask = 2  # Occlusion layer (not player/items)
	query.collide_with_areas = false

	var result = space_state.intersect_ray(query)

	if result.is_empty():
		return 0.0  # No occlusion - direct path

	# Get material properties from hit object
	var material = _get_material_from_collider(result.collider)

	if material:
		# Calculate occlusion based on material and hit angle
		var hit_normal = result.normal
		var ray_dir = (to - from).normalized()
		var incident_angle = abs(hit_normal.dot(ray_dir))

		# Glancing hits have less occlusion
		var angle_factor = lerp(0.5, 1.0, incident_angle)

		return material.occlusion_factor * angle_factor
	else:
		# Default moderate occlusion
		return 0.5

func _get_material_from_collider(collider: Node) -> OcclusionMaterial:
	# Method 1: Metadata on collider
	if collider.has_meta("occlusion_material"):
		var mat_name = collider.get_meta("occlusion_material")
		return materials.get(mat_name, null)

	# Method 2: Group membership
	for mat_name in materials.keys():
		if collider.is_in_group("occlusion_" + mat_name):
			return materials[mat_name]

	# Method 3: PhysicsMaterial metadata
	if collider.has("physics_material_override"):
		var phys_mat = collider.physics_material_override
		if phys_mat and phys_mat.has_meta("occlusion_material"):
			var mat_name = phys_mat.get_meta("occlusion_material")
			return materials.get(mat_name, null)

	return null

func _update_occlusion_filters(delta: float):
	for sound in tracked_sounds:
		if not is_instance_valid(sound.audio_player):
			continue

		# Smooth transition to target occlusion
		var smooth_speed = 5.0
		sound.current_occlusion = lerp(
			sound.current_occlusion,
			sound.target_occlusion,
			smooth_speed * delta
		)

		# Apply volume reduction
		var volume_reduction = sound.current_occlusion * 15.0  # Up to -15dB
		sound.audio_player.volume_db = sound.base_volume_db - volume_reduction

		# Apply low-pass filter
		if sound.lpf_effect:
			_update_lpf(sound)

func _update_lpf(sound: OccludedSound):
	# Get material for current occlusion (approximate)
	var material = _estimate_material(sound.current_occlusion)

	if not material:
		sound.lpf_effect.cutoff_hz = 20000.0  # No filtering
		return

	# Calculate cutoff frequency based on material
	# Full occlusion: 300Hz (very muffled)
	# No occlusion: 20000Hz (full range)
	var base_cutoff = lerp(20000.0, 300.0, sound.current_occlusion)

	# Material-specific adjustment
	var material_mult = lerp(1.0, 2.0, material.high_freq_pass)
	sound.lpf_effect.cutoff_hz = base_cutoff * material_mult

	# Resonance for realistic filter character
	sound.lpf_effect.resonance = 0.5 + (sound.current_occlusion * 0.3)

func _estimate_material(occlusion: float) -> OcclusionMaterial:
	# Find closest material match
	# (In practice, cache the actual material from raycast)
	if occlusion < 0.4:
		return materials.get("glass", null)
	elif occlusion < 0.7:
		return materials.get("wood", null)
	else:
		return materials.get("concrete", null)

# Registration API
func register_sound(player: AudioStreamPlayer3D) -> OccludedSound:
	var sound = OccludedSound.new()
	sound.audio_player = player
	sound.base_volume_db = player.volume_db

	# Add low-pass filter to audio bus
	_add_lpf_to_player(sound)

	tracked_sounds.append(sound)
	return sound

func unregister_sound(player: AudioStreamPlayer3D):
	for i in range(tracked_sounds.size() - 1, -1, -1):
		if tracked_sounds[i].audio_player == player:
			tracked_sounds.remove_at(i)
			break

func _add_lpf_to_player(sound: OccludedSound):
	# Create unique audio bus for this sound
	var bus_name = "Occluded_%d" % sound.get_instance_id()
	AudioServer.add_bus()
	var bus_idx = AudioServer.bus_count - 1
	AudioServer.set_bus_name(bus_idx, bus_name)

	# Route to Master
	AudioServer.set_bus_send(bus_idx, "Master")

	# Add low-pass filter
	var lpf = AudioEffectLowPassFilter.new()
	lpf.cutoff_hz = 20000.0  # Start unfiltered
	lpf.resonance = 0.5
	AudioServer.add_bus_effect(bus_idx, lpf)

	sound.lpf_effect = lpf
	sound.audio_player.bus = bus_name

# Advanced: Portal system for indirect paths
class Portal:
	var position: Vector3
	var connected_rooms: Array[String] = []
	var attenuation: float = 0.3  # Sound loss through portal

var portals: Array[Portal] = []

func _check_occlusion_with_portals(from: Vector3, to: Vector3) -> float:
	# First check direct path
	var direct_occlusion = _check_occlusion(from, to)

	if direct_occlusion < 0.5:
		return direct_occlusion  # Direct path is good enough

	# Check for portal paths
	var best_portal_occlusion = 1.0

	for portal in portals:
		# Check if portal is between listener and source
		var to_portal = _check_occlusion(from, portal.position)
		var from_portal = _check_occlusion(portal.position, to)

		var portal_path_occlusion = max(to_portal, from_portal) + portal.attenuation

		best_portal_occlusion = min(best_portal_occlusion, portal_path_occlusion)

	# Use best path (direct or portal)
	return min(direct_occlusion, best_portal_occlusion)

# Advanced: Multi-raycast for large sources
func _check_occlusion_multi_point(from: Vector3, to: Vector3, source_radius: float) -> float:
	# Cast multiple rays to approximate large sound source
	var sample_points = [
		to,
		to + Vector3.RIGHT * source_radius,
		to + Vector3.LEFT * source_radius,
		to + Vector3.UP * source_radius,
		to + Vector3.DOWN * source_radius,
	]

	var total_occlusion = 0.0
	var valid_samples = 0

	for point in sample_points:
		var occlusion = _check_occlusion(from, point)
		total_occlusion += occlusion
		valid_samples += 1

	return total_occlusion / valid_samples if valid_samples > 0 else 1.0

# Integration with room/reverb system
func get_occlusion_reverb_mix(occlusion: float) -> float:
	# Occluded sounds have more reverb (indirect paths)
	return remap(occlusion, 0.0, 1.0, 0.2, 0.7)

# Debug visualization
func debug_draw_occlusion_rays():
	for sound in tracked_sounds:
		if not is_instance_valid(sound.audio_player):
			continue

		var from = listener.global_position
		var to = sound.audio_player.global_position

		# Draw line colored by occlusion
		var color = Color.GREEN.lerp(Color.RED, sound.current_occlusion)
		# Use ImmediateMesh or debug drawing tool

# Performance optimization: Spatial culling
func _get_nearby_sounds(max_distance: float) -> Array[OccludedSound]:
	var nearby = []

	for sound in tracked_sounds:
		if not is_instance_valid(sound.audio_player):
			continue

		var distance = listener.global_position.distance_to(
			sound.audio_player.global_position
		)

		if distance <= max_distance:
			nearby.append(sound)

	return nearby

# Example integration
class_name OccludedAudioPlayer
extends AudioStreamPlayer3D

var occlusion_system: AudioOcclusionSystem
var occlusion_handle: AudioOcclusionSystem.OccludedSound

func _ready():
	occlusion_system = get_node("/root/AudioOcclusionSystem")
	occlusion_handle = occlusion_system.register_sound(self)

func _exit_tree():
	if occlusion_system:
		occlusion_system.unregister_sound(self)
```

**Key Parameters:**
- **Check frequency:** 10-30Hz (20Hz typical, lower = CPU savings)
- **Max distance:** 30-50m (don't check distant sounds)
- **Occlusion factors:** Wood 0.6, Concrete 0.85, Metal 0.95
- **LPF cutoff range:** 300Hz (full occlude) to 20kHz (clear)
- **Volume reduction:** -10 to -20dB at full occlusion
- **Transition speed:** 3-8 smooth speed (higher = faster)
- **Min volume threshold:** -40dB (skip quiet sounds)

**Edge Cases:**
- **Thin walls:** Multiple raycasts may hit same wall (accumulate)
- **Open doors:** Mark as non-occluding or use area triggers
- **Moving geometry:** Elevators, doors need dynamic updates
- **Large sound sources:** Use multi-point raycasting
- **Listener moving fast:** Increase check frequency temporarily
- **Underwater:** Separate occlusion profile for water
- **Network multiplayer:** Apply occlusion per-client locally

**When NOT to Use:**
- **2D games:** No 3D spatial audio to occlude
- **Arcade/abstract:** Realism not part of aesthetic
- **Open world outdoor:** Few occluding surfaces, minimal benefit
- **Mobile low-end:** Raycast overhead may be too high
- **Very fast gameplay:** Changes happen too fast to perceive
- **Minimal audio design:** Simple soundscape doesn't justify complexity

**Examples from Shipped Games:**

1. **Rainbow Six Siege:** Industry-leading implementation - full material-based propagation. Gunshots through wooden walls audible with muffled low-pass, concrete nearly silent. Bullet holes become audio portals (reduced occlusion). Different materials have measured real-world absorption coefficients. Players use audio to track enemies through walls. Competitive edge dependent on audio awareness. Technical: 100+ raycasts per frame, optimized with spatial grid.

2. **Hellblade: Senua's Sacrifice:** Binaural audio with occlusion for psychological horror. Voices behind Senua properly occluded by environment. Stone structures heavily muffle, open areas clear. Occlusion contributes to disorientation and paranoia. Combined with reverb for spatial depth. Won multiple audio awards for realistic spatial sound. Uses HRTF (Head-Related Transfer Function) with occlusion.

3. **The Last of Us Part II:** Detailed occlusion for stealth gameplay - player must listen for enemy footsteps through walls. Different building materials affect sound propagation realistically. Open windows/doors become audio weak points. Occlusion combined with room acoustics (reverb). Critical for gameplay - players rely on audio to avoid detection. Performance: 40-60 dynamic sounds checked per frame.

4. **Escape from Tarkov:** Realistic military sim - audio occlusion determines combat awareness. Indoor vs outdoor fighting sounds completely different. Gunshots outside building heavily muffled inside. Glass windows provide minimal occlusion (tactical decision). Audio propagation through doorways and corridors. Occlusion affects gameplay balance (indoor vs outdoor loadouts).

5. **Half-Life: Alyx (VR):** VR-specific occlusion - critical for presence and immersion. Headcrabs behind walls have characteristic muffled skitter. Combine soldiers easily located by weapon sounds through walls. Environmental storytelling via occluded audio (distant battles). VR amplifies importance of spatial audio realism. Uses Valve's Steam Audio SDK (ray-traced acoustics).

**Platform Considerations:**
- **PC/Console:** Can afford 50-100 raycasts per frame (20-30 sounds)
- **Mobile:** Limit to 10-20 sounds, reduce check frequency to 10Hz
- **VR:** Critical for presence, prioritize even on lower-end hardware
- **Raycast cost:** 0.01-0.02ms per cast with BVH, budget accordingly
- **Audio DSP:** LPF is cheap (<0.01ms per sound), can use liberally
- **Memory:** Negligible overhead, just filter state per sound

**Godot-Specific Notes:**
- `PhysicsRayQueryParameters3D` for occlusion checks
- Set `collision_mask` to dedicated occlusion layer (layer 2)
- `AudioEffectLowPassFilter` for frequency filtering
- Create unique bus per occluded sound or use bus pooling
- `AudioStreamPlayer3D.volume_db` for occlusion volume reduction
- Store material data in `PhysicsMaterial.set_meta("occlusion_material")`
- Use `_physics_process()` for consistent timing
- Performance: Profile with many active sounds, target <0.5ms total

**Synergies:**
- **Footstep System (Technique 109):** Occluded footsteps for stealth
- **Dynamic Music (Technique 111):** Music less affected by occlusion
- **Audio Ducking (Technique 110):** Combine volume reductions carefully
- **Reverb System:** Occluded sounds have more reverb (indirect)
- **Enemy AI:** AI hears occluded sounds at reduced volume too
- **Material System:** Shared material properties with footsteps
- **Bullet Penetration:** Audio occlusion matches damage penetration

**Measurement/Profiling:**
- **Raycast count:** Monitor with Godot profiler (target <100/frame)
- **CPU cost:** Total occlusion system <0.5ms per frame
- **Check frequency:** Verify actual Hz matches target (timing)
- **Occlusion accuracy:** Compare to player expectation (QA testing)
- **Transition smoothness:** No audible pops/clicks during transitions
- **Player feedback:** Survey realism perception before/after
- **Performance budget:** Reserve 5-10% audio CPU for occlusion

**Advanced Techniques:**
```gdscript
# Diffraction around corners
func check_diffraction(from: Vector3, to: Vector3, blocked: bool) -> float:
	if not blocked:
		return 0.0

	# Cast rays around obstacle
	var offsets = [Vector3.UP, Vector3.DOWN, Vector3.LEFT, Vector3.RIGHT]
	var min_path_occlusion = 1.0

	for offset in offsets:
		var mid_point = (from + to) / 2.0 + offset * 2.0
		var path1 = _check_occlusion(from, mid_point)
		var path2 = _check_occlusion(mid_point, to)
		var path_occlusion = max(path1, path2) + 0.3  # Diffraction loss
		min_path_occlusion = min(min_path_occlusion, path_occlusion)

	return min_path_occlusion

# Material thickness accumulation
func check_occlusion_with_thickness(from: Vector3, to: Vector3) -> float:
	var space_state = get_world_3d().direct_space_state
	var accumulated_occlusion = 0.0
	var current_pos = from

	# Cast multiple times to accumulate layers
	for i in range(3):  # Up to 3 wall layers
		var query = PhysicsRayQueryParameters3D.create(current_pos, to)
		var result = space_state.intersect_ray(query)

		if result.is_empty():
			break

		var material = _get_material_from_collider(result.collider)
		if material:
			accumulated_occlusion += material.occlusion_factor

		current_pos = result.position + (to - from).normalized() * 0.1

	return clamp(accumulated_occlusion, 0.0, 1.0)

# Frequency-dependent distance attenuation
func apply_realistic_distance_lpf(distance: float):
	# High frequencies attenuate faster with distance
	var cutoff = remap(distance, 0, 50, 20000, 5000)
	# Apply to LPF
```

---
