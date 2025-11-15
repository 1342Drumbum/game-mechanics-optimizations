### 109. Footstep Material Detection

**Category:** Game Feel - Audio

**Problem It Solves:** Repetitive, non-contextual footstep audio breaks immersion and makes environments feel lifeless. Games with single footstep sound feel 40-50% less immersive according to audio design studies. Players unconsciously notice when footsteps don't match visual surfaces, creating cognitive dissonance that undermines world believability and player presence.

**Technical Explanation:**
System performs raycast from character foot position downward, retrieves material/texture information from hit surface, maps material to corresponding audio pool, then plays random sound from that pool with subtle pitch/volume variation. Uses physics layers for efficiency (terrain layer separate from props). Caches recent results to avoid per-frame raycasts.

Advanced implementation: Store material data in PhysicsMaterial metadata, use texture splatmap for terrain blending (play multiple sounds with weights), account for weather states (wet sounds after rain), and integrate with animation events for precise timing (heel strike vs toe).

**Algorithmic Complexity:**
- Raycast: O(log n) with spatial partitioning (BVH)
- Material lookup: O(1) hash map access
- Sound selection: O(1) random array access
- Total: O(log n) per footstep, typically <0.1ms

**Implementation Pattern:**
```gdscript
# Godot material-based footstep system
class_name FootstepSystem
extends Node3D

# Material type definitions
enum MaterialType {
	GRASS,
	CONCRETE,
	WOOD,
	METAL,
	WATER,
	GRAVEL,
	DIRT,
	TILE,
	CARPET,
	SNOW,
	MUD
}

# Audio pool for each material
class MaterialSounds:
	var material_type: MaterialType
	var sounds: Array[AudioStream] = []
	var volume_range: Vector2 = Vector2(0.8, 1.0)
	var pitch_range: Vector2 = Vector2(0.95, 1.05)
	var wet_sounds: Array[AudioStream] = []  # Optional wet variants

@export var character_body: CharacterBody3D
@export var left_foot_marker: Node3D  # Position marker for left foot
@export var right_foot_marker: Node3D
@export var audio_player: AudioStreamPlayer3D

# Material sound mappings
var material_sounds: Dictionary = {}

# Raycast optimization
var raycast_distance: float = 1.5
var last_material: MaterialType = MaterialType.CONCRETE
var cache_duration: float = 0.5  # Cache material for 0.5s
var last_check_time: float = 0.0

# State tracking
var is_wet: bool = false  # Weather system integration
var movement_speed: float = 0.0

func _ready():
	_setup_material_sounds()
	set_physics_process(false)  # Use animation events instead

func _setup_material_sounds():
	# Grass sounds
	var grass = MaterialSounds.new()
	grass.material_type = MaterialType.GRASS
	grass.sounds = [
		preload("res://audio/footsteps/grass_01.ogg"),
		preload("res://audio/footsteps/grass_02.ogg"),
		preload("res://audio/footsteps/grass_03.ogg"),
		preload("res://audio/footsteps/grass_04.ogg"),
	]
	grass.volume_range = Vector2(0.6, 0.8)  # Softer
	material_sounds[MaterialType.GRASS] = grass

	# Concrete sounds
	var concrete = MaterialSounds.new()
	concrete.material_type = MaterialType.CONCRETE
	concrete.sounds = [
		preload("res://audio/footsteps/concrete_01.ogg"),
		preload("res://audio/footsteps/concrete_02.ogg"),
		preload("res://audio/footsteps/concrete_03.ogg"),
	]
	concrete.volume_range = Vector2(0.8, 1.0)  # Louder, hard surface
	material_sounds[MaterialType.CONCRETE] = concrete

	# Wood sounds - high pitch variation
	var wood = MaterialSounds.new()
	wood.material_type = MaterialType.WOOD
	wood.sounds = [
		preload("res://audio/footsteps/wood_01.ogg"),
		preload("res://audio/footsteps/wood_02.ogg"),
		preload("res://audio/footsteps/wood_03.ogg"),
		preload("res://audio/footsteps/wood_04.ogg"),
	]
	wood.pitch_range = Vector2(0.9, 1.15)  # Creaky variation
	material_sounds[MaterialType.WOOD] = wood

	# Metal sounds - ringing quality
	var metal = MaterialSounds.new()
	metal.material_type = MaterialType.METAL
	metal.sounds = [
		preload("res://audio/footsteps/metal_01.ogg"),
		preload("res://audio/footsteps/metal_02.ogg"),
	]
	metal.volume_range = Vector2(0.9, 1.1)  # Can be loud
	material_sounds[MaterialType.METAL] = metal

	# Water/puddles
	var water = MaterialSounds.new()
	water.material_type = MaterialType.WATER
	water.sounds = [
		preload("res://audio/footsteps/water_01.ogg"),
		preload("res://audio/footsteps/water_02.ogg"),
		preload("res://audio/footsteps/water_03.ogg"),
	]
	material_sounds[MaterialType.WATER] = water

# Main detection function
func detect_material(foot_position: Vector3) -> MaterialType:
	var space_state = get_world_3d().direct_space_state

	# Setup raycast
	var query = PhysicsRayQueryParameters3D.create(
		foot_position,
		foot_position + Vector3.DOWN * raycast_distance
	)
	query.collision_mask = 1  # Only check ground layer

	var result = space_state.intersect_ray(query)

	if result.is_empty():
		return last_material  # Return cached if no hit

	# Get material from collider
	var collider = result.collider
	var material = _get_material_from_collider(collider, result.position)

	last_material = material
	last_check_time = Time.get_ticks_msec() / 1000.0

	return material

# Extract material type from physics object
func _get_material_from_collider(collider: Node, hit_position: Vector3) -> MaterialType:
	# Method 1: PhysicsMaterial metadata
	if collider is StaticBody3D or collider is RigidBody3D:
		var physics_material = collider.get("physics_material_override")
		if physics_material and physics_material.has_meta("footstep_material"):
			return physics_material.get_meta("footstep_material")

	# Method 2: Collider metadata
	if collider.has_meta("footstep_material"):
		return collider.get_meta("footstep_material")

	# Method 3: Terrain texture splatmap (advanced)
	if collider.has_method("get_material_at_position"):
		return collider.get_material_at_position(hit_position)

	# Method 4: Node group/tag system
	if collider.is_in_group("grass"):
		return MaterialType.GRASS
	elif collider.is_in_group("concrete"):
		return MaterialType.CONCRETE
	elif collider.is_in_group("wood"):
		return MaterialType.WOOD
	elif collider.is_in_group("metal"):
		return MaterialType.METAL

	# Default fallback
	return MaterialType.CONCRETE

# Play footstep sound (called from animation event)
func play_footstep(foot: String = "left"):
	var foot_pos = left_foot_marker.global_position if foot == "left" else right_foot_marker.global_position

	# Detect current material
	var material = detect_material(foot_pos)

	if material not in material_sounds:
		return  # No sounds defined for this material

	var mat_sounds = material_sounds[material]

	# Select sound variant
	var sound_pool = mat_sounds.wet_sounds if is_wet and mat_sounds.wet_sounds.size() > 0 else mat_sounds.sounds
	var sound = sound_pool[randi() % sound_pool.size()]

	# Apply variation
	var volume_db = randf_range(-6.0, 0.0) * (1.0 - randf_range(mat_sounds.volume_range.x, mat_sounds.volume_range.y))
	var pitch = randf_range(mat_sounds.pitch_range.x, mat_sounds.pitch_range.y)

	# Volume based on movement speed
	var speed_multiplier = clamp(movement_speed / 6.0, 0.3, 1.0)  # Running louder than walking
	volume_db += linear_to_db(speed_multiplier)

	# Play sound
	audio_player.stream = sound
	audio_player.pitch_scale = pitch
	audio_player.volume_db = volume_db
	audio_player.global_position = foot_pos
	audio_player.play()

	# Optional: Trigger particle effect based on material
	_spawn_footstep_particle(foot_pos, material)

func _spawn_footstep_particle(position: Vector3, material: MaterialType):
	# Dust puff for dirt, water splash for water, etc.
	match material:
		MaterialType.DIRT, MaterialType.GRAVEL:
			# Spawn dust particle
			pass
		MaterialType.WATER, MaterialType.MUD:
			# Spawn splash
			pass
		MaterialType.SNOW:
			# Snow puff + footprint decal
			pass

# Advanced: Terrain texture blending
func detect_terrain_blend(terrain: Node, position: Vector3) -> Dictionary:
	# Returns weights for multiple materials (e.g., 70% grass, 30% dirt)
	var weights = {}

	# Sample splatmap at position
	if terrain.has_method("get_texture_weights_at"):
		var texture_weights = terrain.get_texture_weights_at(position)

		# Map texture indices to material types
		var texture_to_material = {
			0: MaterialType.GRASS,
			1: MaterialType.DIRT,
			2: MaterialType.GRAVEL,
			3: MaterialType.ROCK,
		}

		for i in texture_weights.size():
			if texture_weights[i] > 0.1:  # Only if significant
				var mat = texture_to_material.get(i, MaterialType.DIRT)
				weights[mat] = texture_weights[i]

	return weights

# Play blended footstep for terrain transitions
func play_blended_footstep(foot_pos: Vector3):
	var terrain = _get_terrain_at(foot_pos)
	if not terrain:
		play_footstep()
		return

	var weights = detect_terrain_blend(terrain, foot_pos)

	# Play multiple sounds with volume based on weight
	for material in weights:
		if material in material_sounds:
			var weight = weights[material]
			if weight > 0.2:  # Only play if significant
				_play_material_sound(material, foot_pos, weight)

func _play_material_sound(material: MaterialType, position: Vector3, volume_mult: float):
	var mat_sounds = material_sounds[material]
	var sound = mat_sounds.sounds[randi() % mat_sounds.sounds.size()]

	# Use one-shot player for multiple simultaneous sounds
	var player = AudioStreamPlayer3D.new()
	add_child(player)
	player.stream = sound
	player.global_position = position
	player.volume_db = linear_to_db(volume_mult) - 3.0  # Reduced for blending
	player.pitch_scale = randf_range(mat_sounds.pitch_range.x, mat_sounds.pitch_range.y)
	player.play()

	# Auto-cleanup
	player.finished.connect(func(): player.queue_free())

# Animation event integration
func _on_animation_tree_footstep_left():
	movement_speed = character_body.velocity.length()
	play_footstep("left")

func _on_animation_tree_footstep_right():
	movement_speed = character_body.velocity.length()
	play_footstep("right")
```

**Key Parameters:**
- **Sound variants per material:** 3-6 samples minimum (4-5 ideal)
- **Pitch variation:** ±5% standard, ±15% for creaky surfaces (wood)
- **Volume variation:** ±2dB standard, ±4dB for natural surfaces
- **Raycast distance:** 1.0-2.0m (character height dependent)
- **Cache duration:** 0.3-0.5s to avoid redundant raycasts
- **Speed multiplier:** Walk (0.5x), Run (1.0x), Sprint (1.3x)
- **Material blend threshold:** >20% weight to trigger blended sound

**Edge Cases:**
- **Mid-air footsteps:** Check if grounded before playing
- **Ladder/climbing:** Different sound system or muted footsteps
- **Swimming:** Separate water movement sounds
- **Untagged surfaces:** Fallback to default (concrete/dirt)
- **Thin surfaces:** Raycast might hit object below (extend raycast)
- **Moving platforms:** Account for platform velocity in positioning
- **Network sync:** Client-side prediction, validate on server

**When NOT to Use:**
- **2D platformers:** Visual abstraction makes material detection unnecessary
- **Abstract/stylized games:** May prefer unified audio aesthetic
- **Very fast movement:** Sounds blend together, simpler system sufficient
- **Limited audio budget:** 50+ sound files per character significant
- **Stealth games with simple detection:** Binary sound/no-sound enough
- **Early prototyping:** Add after core mechanics proven

**Examples from Shipped Games:**

1. **The Last of Us Part II:** Industry-leading implementation - 8-12 variants per material, context-aware (wet/dry, crouch/walk/run). Subtle shoe scuff sounds between steps. Material detection uses terrain splatmaps for natural blending. Running on grass->dirt transition plays blended sounds. Audio team recorded 200+ unique footstep samples. Footsteps account for 15-20% of environmental immersion.

2. **Hellblade: Senua's Sacrifice:** Binaural footstep recording creates 3D presence. Different sounds for bare feet vs boots. Detects surface angle (uphill footsteps sound different). Gravel footsteps have micro-delay between rocks settling. Water depth affects splash intensity. Footsteps integrated with psychosis mechanic (imagined footsteps behind player).

3. **Red Dead Redemption 2:** Advanced material system - 15+ surface types including mud, snow, sand, various wood types. Horse hooves have separate material system (4 feet = 4x complexity). Boot spurs add jingling layer. Snow depth affects sound (compacting vs crunching). Wet materials have 50% volume reduction + muffled EQ. Weather persistence system tracks wet surfaces after rain.

4. **Doom Eternal:** Fast-paced combat still has detailed footsteps - metal corridors, flesh floors, water. Material detection at 120+ fps with minimal overhead. Heavy emphasis on metal gratings (industrial aesthetic). Demon footsteps have similar system scaled to creature mass. Footstep volume modulates with combat intensity (ducking during action).

5. **Control (Remedy):** Concrete brutalist architecture with subtle material variation. Footsteps change in Oldest House areas (supernatural reverb/delay). Glass floor sections have distinct sound. Running on papers/debris adds layered detail. Dynamic audio mixing keeps footsteps audible during combat without overwhelming.

**Platform Considerations:**
- **PC/Console:** Can handle 8-12 variants per material, complex blending
- **Mobile:** Reduce to 2-3 variants, simpler material detection
- **VR:** Critical for presence - binaurally recorded samples essential
- **Audio memory:** 50-100MB for full system (compressed OGG)
- **Streaming:** Stream footsteps if total audio >200MB
- **Spatial audio:** Use 3D audio player, position at foot location

**Godot-Specific Notes:**
- `PhysicsRayQueryParameters3D` for efficient raycasting
- Store material type in `PhysicsMaterial.set_meta()` for easy tagging
- Use `AudioStreamPlayer3D` for proper 3D positioning
- `AnimationPlayer` tracks for footstep timing (add "footstep_L/R" events)
- `collision_mask` to limit raycasts to ground layer (huge optimization)
- `AudioStreamRandomizer` (Godot 4.2+) perfect for variant selection
- Custom `Terrain` node needs `get_material_at_position()` method
- Network: Use `MultiplayerSynchronizer` with `rpc_config` for sound triggers

**Synergies:**
- **Animation System:** Footstep timing from AnimationPlayer events
- **Particle Effects:** Dust/splash VFX matching audio material
- **Decal System:** Footprints on snow/mud coordinated with audio
- **Weather System:** Wet materials after rain, snow crunch in winter
- **Audio Ducking (Technique 110):** Footsteps duck during dialogue
- **Dynamic Music (Technique 111):** Footstep tempo can drive music intensity
- **Audio Occlusion (Technique 112):** Footsteps behind walls muffled

**Measurement/Profiling:**
- **Raycast cost:** Should be <0.1ms per footstep (use Godot profiler)
- **Audio memory:** Monitor total loaded footstep samples
- **Variant repetition:** Track how often same sample plays consecutively (should be <10%)
- **Material coverage:** Ensure 95%+ of surfaces have assigned materials
- **Player feedback:** Immersion surveys before/after implementation
- **Performance budget:** Total footstep system <0.5ms per frame
- **Network bandwidth:** Footstep triggers should be <10 bytes each

**Advanced Techniques:**
```gdscript
# Directional surface detection (uphill vs downhill)
func get_surface_angle(normal: Vector3) -> float:
	return rad_to_deg(acos(normal.dot(Vector3.UP)))

func play_footstep_with_angle():
	# Uphill = heavier, downhill = lighter
	var angle = get_surface_angle(floor_normal)
	var volume_mod = remap(angle, 0, 45, 0.0, 3.0)  # Louder on slopes
	audio_player.volume_db += volume_mod

# Shoe type variation
enum ShoeType { BOOTS, SNEAKERS, BAREFOOT, HEELS }
var current_shoes: ShoeType = ShoeType.BOOTS

func get_shoe_modifier() -> Dictionary:
	match current_shoes:
		ShoeType.BOOTS:
			return {"volume": 1.0, "pitch": 0.9}  # Lower, louder
		ShoeType.SNEAKERS:
			return {"volume": 0.7, "pitch": 1.0}  # Quieter
		ShoeType.BAREFOOT:
			return {"volume": 0.5, "pitch": 1.1}  # Soft, higher
		ShoeType.HEELS:
			return {"volume": 1.2, "pitch": 1.2}  # Loud, clicky
	return {"volume": 1.0, "pitch": 1.0}

# Footstep audio pooling (performance)
var audio_pool: Array[AudioStreamPlayer3D] = []
var pool_size: int = 8

func get_audio_player() -> AudioStreamPlayer3D:
	for player in audio_pool:
		if not player.playing:
			return player
	return audio_pool[0]  # Steal oldest
```

---
