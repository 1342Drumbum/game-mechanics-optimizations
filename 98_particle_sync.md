### 98. Particle Effects Sync

**Category:** Game Feel - Visual Feedback

**Problem It Solves:** Disconnected visual feedback where particles spawn before/after the actual event. Unsynchronized particle timing breaks immersion and reduces impact. Players perceive delayed particles as "laggy" even at 60fps. Generic particle effects that don't match action intensity feel procedural and cheap.

**Technical Explanation:**
Spawn particles on exact game event frame (hit, jump, death) synchronized with other feedback systems. Use object pooling for zero-allocation spawning. Scale particle count/velocity/lifetime based on event magnitude (heavy hit = more particles). Particles inherit parent velocity for momentum conservation. Uses emission signals, not polling, to guarantee frame-perfect spawning. Critical for "juice" - the particle burst must coincide exactly with hit pause, screen shake, and sound effect.

**Algorithmic Complexity:**
- O(1) particle emission trigger
- O(n) particle update where n = active particles
- O(1) pooled particle retrieval
- CPU: 0.1-2ms per frame for 1000-5000 active particles
- GPU: Particle rendering scales linearly with count

**Implementation Pattern:**
```gdscript
# Godot implementation - Synced Particle System
class_name SyncedParticleEmitter
extends Node2D

@export var particle_scene: PackedScene
@export var pool_size: int = 100
@export var base_particle_count: int = 10
@export var max_particle_count: int = 50
@export var velocity_inheritance: float = 0.5  # 0-1, how much parent velocity to inherit
@export var intensity_scaling: bool = true

var particle_pool: Array[GPUParticles2D] = []
var active_particles: Array[GPUParticles2D] = []

func _ready():
	_initialize_pool()

func _initialize_pool():
	for i in range(pool_size):
		var particle = particle_scene.instantiate() as GPUParticles2D
		particle.emitting = false
		particle.one_shot = true
		add_child(particle)
		particle_pool.append(particle)

# Emit particles synchronized with event
func emit_particles(pos: Vector2, intensity: float = 1.0, velocity: Vector2 = Vector2.ZERO):
	# Calculate particle count based on intensity
	var particle_count = base_particle_count
	if intensity_scaling:
		particle_count = int(lerp(base_particle_count, max_particle_count, intensity))

	for i in range(particle_count):
		var particle = _get_pooled_particle()
		if particle == null:
			break

		# Position particle
		particle.global_position = pos

		# Inherit velocity (for moving objects)
		if velocity.length() > 0.1:
			var inherited_vel = velocity * velocity_inheritance
			particle.process_material.initial_velocity_min = inherited_vel.length() * 0.8
			particle.process_material.initial_velocity_max = inherited_vel.length() * 1.2
			particle.process_material.direction = Vector3(velocity.normalized().x, velocity.normalized().y, 0)

		# Scale by intensity
		particle.amount = int(particle.amount * intensity)
		particle.lifetime = particle.lifetime * clamp(intensity, 0.5, 1.5)

		# Start emission
		particle.emitting = true
		active_particles.append(particle)

	# Cleanup finished particles
	_cleanup_finished_particles()

func _get_pooled_particle() -> GPUParticles2D:
	if particle_pool.is_empty():
		# Pool exhausted - reuse oldest active or skip
		if active_particles.size() > 0:
			var oldest = active_particles.pop_front()
			oldest.emitting = false
			return oldest
		return null

	return particle_pool.pop_back()

func _cleanup_finished_particles():
	# Return finished particles to pool
	var i = 0
	while i < active_particles.size():
		var particle = active_particles[i]
		if not particle.emitting:
			particle_pool.append(particle)
			active_particles.remove_at(i)
		else:
			i += 1

# Combat integration example
class_name CombatSystem extends Node

signal hit_landed(position: Vector2, damage: float, velocity: Vector2)

@onready var hit_particles: SyncedParticleEmitter = $HitParticles
@onready var camera: CameraTrauma = $Camera
@onready var hit_pause: HitPauseManager = $HitPause

func _on_weapon_hit(target_pos: Vector2, damage: float, weapon_velocity: Vector2):
	# Calculate intensity from damage (0-1 range)
	var intensity = clamp(damage / 100.0, 0.0, 1.0)

	# SYNCHRONIZED FEEDBACK - all happen same frame
	hit_particles.emit_particles(target_pos, intensity, weapon_velocity)
	camera.add_trauma(0.2 + intensity * 0.4)
	hit_pause.trigger_pause(0.05 + intensity * 0.1)
	_play_hit_sound(intensity)

	hit_landed.emit(target_pos, damage, weapon_velocity)

# Advanced: Direction-based particle emission
func emit_directional_particles(pos: Vector2, direction: Vector2, intensity: float = 1.0):
	var particle = _get_pooled_particle()
	if particle == null:
		return

	particle.global_position = pos

	# Configure particle material for directional emission
	var mat = particle.process_material as ParticleProcessMaterial
	mat.direction = Vector3(direction.x, direction.y, 0)
	mat.spread = 30.0  # Cone angle
	mat.initial_velocity_min = 100.0 * intensity
	mat.initial_velocity_max = 200.0 * intensity

	particle.emitting = true
	active_particles.append(particle)

# Blood splatter / impact effect
func emit_impact_spray(pos: Vector2, surface_normal: Vector2, intensity: float = 1.0):
	# Particles spray away from surface
	var spray_direction = surface_normal.normalized()

	for i in range(int(10 * intensity)):
		var particle = _get_pooled_particle()
		if particle == null:
			break

		particle.global_position = pos

		# Randomize within cone
		var spread_angle = randf_range(-PI/4, PI/4)
		var final_direction = spray_direction.rotated(spread_angle)

		var mat = particle.process_material as ParticleProcessMaterial
		mat.direction = Vector3(final_direction.x, final_direction.y, 0)
		mat.initial_velocity_min = 50.0 * intensity
		mat.initial_velocity_max = 150.0 * intensity

		particle.emitting = true
		active_particles.append(particle)
```

**Key Parameters:**
- **Base particle count:** 5-15 for subtle, 20-50 for explosive
- **Pool size:** 50-200 (depends on max simultaneous emissions)
- **Velocity inheritance:** 0.3-0.7 (too high = particles lag behind)
- **Lifetime:** 0.3-1.5 seconds (longer = more GPU load)
- **Intensity scaling:** Linear (damage/100) or exponential (damage^1.5/100)
- **Emission rate (continuous):** 30-100 particles/second
- **Burst count (one-shot):** 10-50 particles per burst

**Edge Cases:**
- **Pool exhaustion:** Reuse oldest particles or skip emission
- **Simultaneous emissions:** Multiple hits same frame can exhaust pool
- **Particle inheritance on moving platforms:** Parent to platform temporarily
- **Screen-space particles vs world-space:** UI damage numbers use screen-space
- **Particle limit:** Console hardware often caps at 5000-10000 active particles
- **Z-ordering:** Ensure particles render above/below correct layers
- **Cleanup on scene change:** Return all particles to pool on level unload

**When NOT to Use:**
- **Low-end mobile:** Reduce particle count aggressively or disable
- **Target 30fps:** Halve particle counts, reduces GPU load
- **Minimalist aesthetic:** Contradicts design goals
- **Network games:** Don't sync particles over network, purely cosmetic
- **Distant objects:** LOD system should disable particles beyond 50-100m
- **Menu screens:** Unnecessary CPU/GPU load

**Examples from Shipped Games:**

1. **Dead Cells:** Every weapon swing spawns 5-20 particles synced with damage. Critical hits spawn 3x particles with different color. Particles inherit weapon velocity creating motion trails. Essential to the game's visceral combat feel.

2. **Hyper Light Drifter:** Sword slashes spawn geometric particle bursts (10-15 particles), enemy damage spawns pixel blood (5-10 particles). Dash creates persistent trail (30 particles/sec). Particles match the game's pixel art aesthetic perfectly.

3. **Enter the Gungeon:** Each gun has unique particle signature - shell casings for realistic guns (2-5 per shot), magical sparks for magic weapons (10-20 per shot). Dodge roll spawns dust cloud (15 particles). Particle differentiation aids weapon identification.

4. **Celeste:** Dash spawns hair-colored particles (20-30 particles), death spawns feather explosion (50+ particles), landing spawns dust puff (5-10 particles). Conservative counts maintain 60fps on Switch. Particles are critical to readability.

5. **Hollow Knight:** Nail hits spawn soul particles (3-5 white particles), spell casts spawn directional bursts (20-40 particles), enemy deaths spawn distinctive particles per enemy type. Particles communicate game state clearly.

**Platform Considerations:**
- **Desktop GPU:** 10,000-50,000 active particles feasible
- **Mobile GPU:** 500-2000 particle limit for 60fps
- **Switch:** 2000-5000 particles recommended
- **WebGL:** Limit to 1000-3000, browser overhead is significant
- **VR:** Minimize particles, each eye renders separately (2x cost)
- **Particle size vs distance:** Mobile needs larger particles for small screens

**Godot-Specific Notes:**
- `GPUParticles2D/3D` for performance (GPU-accelerated)
- `CPUParticles2D/3D` for compatibility (mobile/web fallback)
- `one_shot = true` for burst emissions
- `explosiveness = 1.0` for simultaneous particle spawn
- Pool `GPUParticles2D` nodes, not individual particles
- `amount_ratio` in Godot 4.x for dynamic particle count scaling
- `process_material` must be unique per particle instance if modifying
- Use `local_coords = false` for world-space particles

**Synergies:**
- **Hit Pause (Technique 95):** Spawn particles on pause start frame
- **Screen Shake (Technique 97):** Particle burst + shake = maximum impact
- **Squash & Stretch (Technique 96):** Particles spawn during squash frame
- **Hit Confirmation (Technique 106):** Particles + sound + rumble = complete feedback
- **Damage Scaling (Technique 102):** More damage = more particles
- **Chromatic Aberration (Technique 99):** Flash effect when particles spawn

**Measurement/Profiling:**
- **GPU profiler:** Particle rendering cost (should be <2ms at 60fps)
- **Particle count:** Monitor active particles, cap at platform limit
- **Pool efficiency:** Track pool exhaustion events (should be rare)
- **Frame timing:** Verify particles spawn on exact event frame (use frame debugger)
- **Visual testing:** Record at 60fps, frame-by-frame verify sync
- **Performance regression:** Test with 100+ simultaneous particles
- **Memory:** Each particle node ~1-5KB, monitor total pool memory

**Advanced Techniques:**
```gdscript
# Particle color based on damage type
func emit_typed_particles(pos: Vector2, damage_type: DamageType, intensity: float):
	var particle = _get_pooled_particle()
	if particle == null:
		return

	# Set color based on type
	var mat = particle.process_material as ParticleProcessMaterial
	match damage_type:
		DamageType.FIRE:
			mat.color_ramp.gradient.set_color(0, Color.ORANGE_RED)
		DamageType.ICE:
			mat.color_ramp.gradient.set_color(0, Color.CYAN)
		DamageType.POISON:
			mat.color_ramp.gradient.set_color(0, Color.GREEN)

	particle.global_position = pos
	particle.emitting = true

# Particle trails for projectiles
class_name ProjectileTrail extends Node2D

var trail_particles: GPUParticles2D
var emit_timer: float = 0.0
var emit_interval: float = 0.016  # ~60 fps

func _process(delta):
	emit_timer += delta
	if emit_timer >= emit_interval:
		emit_timer = 0.0
		_emit_trail_particle()

func _emit_trail_particle():
	# Spawn particle at current position
	trail_particles.global_position = global_position
	# Particle system handles lifetime/fadeout

# Synchronized multi-emitter (explosion)
func emit_explosion(pos: Vector2, radius: float, intensity: float = 1.0):
	# Core explosion particles
	emit_particles(pos, intensity)

	# Shockwave ring (delayed)
	await get_tree().create_timer(0.05).timeout
	emit_shockwave(pos, radius)

	# Debris particles (delayed more)
	await get_tree().create_timer(0.1).timeout
	emit_debris(pos, radius, intensity)

# Particle sub-emitters (particles spawning particles)
func setup_sub_emitter(parent_particle: GPUParticles2D):
	var sub_particle = particle_scene.instantiate() as GPUParticles2D
	sub_particle.amount = 5
	sub_particle.lifetime = 0.3
	parent_particle.add_child(sub_particle)
	# Sub-particle spawns when parent particle emits
```
