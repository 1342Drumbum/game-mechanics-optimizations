### 52. 2D Particle Optimization

**Category:** Performance - 2D Visual Effects & Rendering

**Problem It Solves:** Particle systems drain performance through excessive draw calls and physics calculations. 10 emitters × 1000 particles = 10,000 sprites rendered individually = 10,000 draw calls at 50μs each = 500ms GPU time, destroying 60fps. Optimized particles batch to 10-50 draw calls = 0.5-2.5ms, a 99% reduction. CPU particle simulation for 10,000 particles = 30ms, GPU-based reduces to 2ms.

**Technical Explanation:**
Particle optimization uses multiple techniques: GPU-based particles (GPUParticles2D) offload simulation to GPU, sprite batching combines particles into single draw call, object pooling reuses particle nodes, particle culling disables off-screen emitters, LOD systems reduce particle count at distance, and texture atlasing combines particle textures. Modern engines use compute shaders for particle physics - update 10,000 particles in parallel vs sequential CPU updates.

Advanced approach separates particle categories: alpha-blended (smoke, magic) use GPU particles with additive blending, opaque particles (debris, sparks) use simpler simulation, collision-enabled particles (shells, gibs) limited to <100 count, and pure visual particles (dust, ambient) use cheapest settings. Critical optimization: particle lifetime management - short-lived particles (<1s) can use aggressive pooling, long-lived particles need careful memory management.

**Algorithmic Complexity:**
- CPU Particles: O(n) sequential update for n particles = 30ms for 10k particles
- GPU Particles: O(n) parallel update on 2000+ cores = 2ms for 10k particles
- Draw Calls: Unbatched O(n), batched O(1) per emitter
- Physics: O(n × m) for particles colliding with m objects, avoid when possible

**Implementation Pattern:**
```gdscript
# Optimized particle system manager
class_name ParticleSystemManager
extends Node2D

# Particle pools for different effects
var particle_pools: Dictionary = {}  # effect_name -> Array[GPUParticles2D]
const MAX_POOL_SIZE = 20

# Active particle tracking
var active_particles: Array[GPUParticles2D] = []
var particle_budgets: Dictionary = {
    "high_priority": 5000,    # Player attacks, important VFX
    "medium_priority": 3000,  # Enemy effects
    "low_priority": 2000,     # Ambient, environmental
}
var current_particle_counts: Dictionary = {}

func _ready():
    setup_particle_pools()

func setup_particle_pools():
    # Pre-create particle systems for common effects
    create_pool("explosion", preload("res://particles/explosion.tres"), 10)
    create_pool("smoke", preload("res://particles/smoke.tres"), 15)
    create_pool("spark", preload("res://particles/spark.tres"), 20)
    create_pool("dust", preload("res://particles/dust.tres"), 10)

func create_pool(effect_name: String, particle_scene: Resource, pool_size: int):
    particle_pools[effect_name] = []

    for i in pool_size:
        var particles = GPUParticles2D.new()
        particles.process_material = particle_scene
        particles.emitting = false
        particles.one_shot = true
        add_child(particles)
        particle_pools[effect_name].append(particles)

func spawn_particle(effect_name: String, position: Vector2, priority: String = "medium_priority") -> GPUParticles2D:
    # Check budget
    if not can_spawn_particle(priority):
        return null

    # Get from pool
    var particles = get_pooled_particle(effect_name)
    if not particles:
        return null

    # Configure and emit
    particles.global_position = position
    particles.emitting = true
    particles.restart()

    active_particles.append(particles)
    current_particle_counts[priority] = current_particle_counts.get(priority, 0) + particles.amount

    # Auto-return to pool after lifetime
    var lifetime = particles.lifetime
    await get_tree().create_timer(lifetime).timeout
    return_particle(particles, priority)

    return particles

func can_spawn_particle(priority: String) -> bool:
    var current = current_particle_counts.get(priority, 0)
    var budget = particle_budgets.get(priority, 1000)
    return current < budget

func get_pooled_particle(effect_name: String) -> GPUParticles2D:
    if not particle_pools.has(effect_name):
        return null

    var pool = particle_pools[effect_name]
    for particles in pool:
        if not particles.emitting:
            return particles

    # Pool exhausted, reuse oldest if allowed
    if pool.size() > 0:
        return pool[0]

    return null

func return_particle(particles: GPUParticles2D, priority: String):
    particles.emitting = false
    active_particles.erase(particles)
    current_particle_counts[priority] = max(0, current_particle_counts.get(priority, 0) - particles.amount)

# GPU-optimized particle configuration
class_name OptimizedGPUParticles
extends GPUParticles2D

@export var auto_disable_when_offscreen: bool = true
@export var cull_distance: float = 1000.0

func _ready():
    # Use GPU particles for performance
    # process_material should be ParticleProcessMaterial

    # Enable batching
    texture_filter = CanvasItem.TEXTURE_FILTER_LINEAR
    visibility_rect = Rect2(-50, -50, 100, 100)  # Local bounds

    # Optimize draw passes
    draw_order = DRAW_ORDER_INDEX  # Faster than DRAW_ORDER_LIFETIME

    # Fixed FPS for particle simulation (30fps sufficient for most effects)
    fixed_fps = 30
    fract_delta = false  # Better performance

func _process(_delta):
    if not auto_disable_when_offscreen:
        return

    # Cull particles outside viewport
    var viewport_rect = get_viewport_rect()
    var screen_pos = get_global_transform_with_canvas().origin

    if not viewport_rect.has_point(screen_pos):
        emitting = false

    # Distance-based culling
    var camera = get_viewport().get_camera_2d()
    if camera:
        var dist = global_position.distance_to(camera.global_position)
        if dist > cull_distance:
            emitting = false

# Particle LOD system
class_name ParticleLODSystem
extends Node

enum LODLevel { HIGH, MEDIUM, LOW, DISABLED }

func update_particle_lod(particles: GPUParticles2D, camera_pos: Vector2):
    var distance = particles.global_position.distance_to(camera_pos)
    var lod = calculate_lod(distance)

    apply_lod(particles, lod)

func calculate_lod(distance: float) -> LODLevel:
    if distance < 300:
        return LODLevel.HIGH
    elif distance < 600:
        return LODLevel.MEDIUM
    elif distance < 1000:
        return LODLevel.LOW
    else:
        return LODLevel.DISABLED

func apply_lod(particles: GPUParticles2D, lod: LODLevel):
    match lod:
        LODLevel.HIGH:
            particles.amount = 100
            particles.process_material.set_shader_parameter("subdivision", 4)
            particles.emitting = true
        LODLevel.MEDIUM:
            particles.amount = 50
            particles.process_material.set_shader_parameter("subdivision", 2)
            particles.emitting = true
        LODLevel.LOW:
            particles.amount = 20
            particles.process_material.set_shader_parameter("subdivision", 1)
            particles.emitting = true
        LODLevel.DISABLED:
            particles.emitting = false

# Lightweight CPU particles for simple effects
class_name LightweightParticles
extends Node2D

# Custom particle struct
class Particle:
    var position: Vector2
    var velocity: Vector2
    var lifetime: float
    var max_lifetime: float
    var color: Color
    var size: float

var particles: Array[Particle] = []
var max_particles: int = 100

func _ready():
    # Pre-allocate particle array
    for i in max_particles:
        particles.append(Particle.new())

func emit(pos: Vector2, count: int):
    var emitted = 0
    for i in max_particles:
        if particles[i].lifetime <= 0 and emitted < count:
            particles[i].position = pos
            particles[i].velocity = Vector2(randf_range(-50, 50), randf_range(-100, -50))
            particles[i].lifetime = randf_range(0.5, 1.5)
            particles[i].max_lifetime = particles[i].lifetime
            particles[i].color = Color.WHITE
            particles[i].size = randf_range(2, 6)
            emitted += 1

func _process(delta):
    queue_redraw()

    for particle in particles:
        if particle.lifetime <= 0:
            continue

        # Update particle
        particle.lifetime -= delta
        particle.position += particle.velocity * delta
        particle.velocity.y += 200 * delta  # Gravity

        # Fade out
        var alpha = particle.lifetime / particle.max_lifetime
        particle.color.a = alpha

func _draw():
    for particle in particles:
        if particle.lifetime > 0:
            draw_circle(particle.position, particle.size, particle.color)

# Batched particle rendering
class_name BatchedParticleRenderer
extends Node2D

var particle_mesh: MultiMeshInstance2D
var particle_data: Array = []  # Position, rotation, scale, color per particle

const MAX_PARTICLES = 10000

func _ready():
    # Setup multimesh for batched rendering
    particle_mesh = MultiMeshInstance2D.new()
    particle_mesh.multimesh = MultiMesh.new()
    particle_mesh.multimesh.transform_format = MultiMesh.TRANSFORM_2D
    particle_mesh.multimesh.instance_count = MAX_PARTICLES

    # Set particle texture
    particle_mesh.texture = preload("res://textures/particle.png")

    add_child(particle_mesh)

func update_particles(delta: float):
    # Update all particles (on CPU or GPU compute)
    for i in particle_data.size():
        var particle = particle_data[i]

        # Physics update
        particle.position += particle.velocity * delta
        particle.velocity.y += particle.gravity * delta
        particle.lifetime -= delta

        # Set transform in multimesh
        if particle.lifetime > 0:
            var transform = Transform2D()
            transform = transform.rotated(particle.rotation)
            transform = transform.scaled(Vector2.ONE * particle.scale)
            transform.origin = particle.position

            particle_mesh.multimesh.set_instance_transform_2d(i, transform)

            # Set color/alpha
            var alpha = particle.lifetime / particle.max_lifetime
            particle_mesh.multimesh.set_instance_color(i, Color(1, 1, 1, alpha))
        else:
            # Hide dead particle
            particle_mesh.multimesh.set_instance_transform_2d(i, Transform2D(0, Vector2.ZERO))

# Collision-aware particles (expensive, use sparingly)
class_name PhysicsParticles
extends Node2D

var particle_bodies: Array[RigidBody2D] = []
const MAX_PHYSICS_PARTICLES = 50  # Keep low!

func emit_physics_particle(pos: Vector2, impulse: Vector2):
    var particle = get_pooled_physics_particle()
    if not particle:
        return

    particle.global_position = pos
    particle.linear_velocity = impulse
    particle.visible = true

func get_pooled_physics_particle() -> RigidBody2D:
    # Find inactive particle
    for particle in particle_bodies:
        if not particle.visible:
            return particle

    # Create new if under limit
    if particle_bodies.size() < MAX_PHYSICS_PARTICLES:
        var particle = RigidBody2D.new()
        var sprite = Sprite2D.new()
        sprite.texture = preload("res://textures/particle.png")
        particle.add_child(sprite)

        var collision = CollisionShape2D.new()
        var circle = CircleShape2D.new()
        circle.radius = 4
        collision.shape = circle
        particle.add_child(collision)

        add_child(particle)
        particle_bodies.append(particle)

        # Auto-hide after lifetime
        var timer = Timer.new()
        timer.wait_time = 2.0
        timer.one_shot = true
        timer.timeout.connect(func(): particle.visible = false)
        particle.add_child(timer)
        timer.start()

        return particle

    return null

# Particle texture atlas optimization
class_name ParticleAtlas
extends Node

# Combine multiple particle textures into atlas
static func create_particle_atlas(textures: Array[Texture2D]) -> Texture2D:
    var atlas_size = 512
    var particles_per_row = 8
    var particle_size = atlas_size / particles_per_row

    var image = Image.create(atlas_size, atlas_size, false, Image.FORMAT_RGBA8)

    for i in textures.size():
        var x = (i % particles_per_row) * particle_size
        var y = (i / particles_per_row) * particle_size

        var particle_image = textures[i].get_image()
        particle_image.resize(particle_size, particle_size)

        image.blit_rect(particle_image, Rect2(Vector2.ZERO, particle_image.get_size()), Vector2(x, y))

    return ImageTexture.create_from_image(image)
```

**Key Parameters:**
- **Particle Count:** 20-50 (ambient), 100-500 (effects), 1000-5000 (explosions), >10k (rare hero moments)
- **Particle Lifetime:** 0.5-2s typical, >5s avoid (memory leak risk)
- **Fixed FPS:** 30fps (most effects), 60fps (close-up important), 15fps (distant ambient)
- **Draw Order:** INDEX (fastest), LIFETIME (quality), REVERSE_LIFETIME
- **Texture Size:** 32×32 (small), 64×64 (medium), 128×128 (large effects)
- **Collision Particles:** <50 total, >100 kills performance

**Edge Cases:**
- **Particle Explosions:** Thousands spawning simultaneously, need pooling + budgets
- **Long-Lived Particles:** >10s lifetime can leak memory, enforce max lifetime
- **Collision Particles:** Physics calculations expensive, limit to <50
- **Alpha Blending:** Overdraw kills fillrate, limit overlapping particle count
- **Off-Screen Particles:** Continue simulating unless culled, waste GPU
- **One-Shot vs Continuous:** One-shot easier to pool, continuous need different lifecycle

**When NOT to Use:**
- Simple games with minimal VFX (<10 particle emitters total)
- Retro games avoiding particle effects for aesthetic
- Already optimized and performant particle systems
- Development phase - focus on gameplay first
- Limited art resources - particles need good textures/design

**Examples from Shipped Games:**

1. **Dead Cells:** Extensive particle optimization for 60fps on Switch. 100+ active emitters during combat (blood splatter, weapon trails, torches). Uses GPU particles with aggressive culling. Blood particles pooled (500 max), weapon effects use batched MultiMesh. Torch flames static animations, not simulated particles. Critical for maintaining performance during intense combat.

2. **Celeste:** Minimal but effective particles. Dash effects (30-50 particles), snow ambient (200 particles), strawberry collect (100 particles burst). Uses CPU particles for precise control. Particle budget strictly enforced - max 500 active particles. Culling disables particles >2 screens away. Enables clean 60fps even on low-end hardware.

3. **Hollow Knight:** Atmospheric particle usage. Ambient particles (dust, spores, rain) limited to 300 total. Combat effects (nail slashes, spells) 50-200 particles. Boss attacks can spawn 500+ particles, uses LOD - reduce count at distance. GPU particles for most effects. Careful budgeting prevents frame drops during intense boss fights.

4. **Enter the Gungeon:** Bullet-hell with massive particle counts. Shell casings (1000+ active), muzzle flashes (500+), explosions (2000+ bursts). Uses aggressive pooling - 5000 particle pool. GPU particles mandatory. Culling beyond 1 room away. Batching reduces 1000s of particles to 20-30 draw calls. Essential for genre defining particle density.

5. **Terraria:** Particles for ambient effects, combat, mining. Dust particles (200 max), liquid effects (300 max), combat (500 max). Simple CPU particles for most effects (performance over quality). Strict budgets prevent particle spam. Multiplayer requires aggressive culling (4 players × particle budget). Efficient particles enable smooth 60fps with 4-player chaos.

**Platform Considerations:**
- **Mobile (iOS/Android):** Critical - limit to 500-1000 total particles. Use GPU particles, disable complex shaders. Fixed FPS 30. Aggressive culling <500px from camera. Target <2ms particle time.
- **Desktop (Windows/Mac/Linux):** 5000-10000 particles comfortable. GPU particles with full features. Can afford particle collisions (limited). Target <5ms particle time.
- **Console (Switch/PS/Xbox):** Switch: 1000-2000 particles, GPU mandatory. PS5/Xbox: 10000+ fine. Fixed performance easier to optimize.
- **WebGL:** GPU particles slower in browser - limit to 500-1000. Simple shaders only. Batching critical.
- **VRAM:** Each particle emitter ~1-5MB for textures/buffers. Budget 50-100MB for all particles.

**Godot-Specific Notes:**
- Use `GPUParticles2D` over `CPUParticles2D` for >100 particles
- `GPUParticles2D.fixed_fps` reduces simulation cost
- Set `visibility_rect` accurately for proper culling
- `draw_order = DRAW_ORDER_INDEX` fastest (vs DRAW_ORDER_LIFETIME)
- Monitor with Profiler → Visual → Particle Updates
- Godot 4.x GPU particles much faster than 3.x
- Use `ParticleProcessMaterial` for built-in effects, custom shaders for unique needs
- `one_shot = true` easier to pool than continuous emitters

**Synergies:**
- **Object Pooling:** Essential for particle system reuse
- **Sprite Batching:** Particles batch together when same texture/material
- **Canvas Layer Management:** Separate particle layers for better organization
- **Dirty Rectangles:** Particles force dirty regions, use separate layer
- **LOD Systems:** Reduce particle count/quality at distance

**Measurement/Profiling:**
- **Particle Count:** Track active particles - target <1000 mobile, <5000 desktop
- **Draw Calls:** Particles should batch to <20 draw calls total
- **GPU Time:** Monitor `TIME_RENDER` - particles should be <20% of frame
- **Memory:** Track particle texture memory + simulation buffers
- **Profiling Pattern:**
```gdscript
var total_particles = 0
var active_emitters = 0
var particle_draw_calls = 0

func profile_particles():
    total_particles = 0
    active_emitters = 0

    for node in get_tree().get_nodes_in_group("particles"):
        if node is GPUParticles2D:
            if node.emitting:
                active_emitters += 1
                total_particles += node.amount

    print("Active emitters: ", active_emitters)
    print("Total particles: ", total_particles)
    print("Particles per emitter: ", total_particles / max(1, active_emitters))

var particle_time_samples = []

func measure_particle_performance():
    var time_before = Performance.get_monitor(Performance.TIME_RENDER)
    # Particles render here
    await get_tree().process_frame
    var time_after = Performance.get_monitor(Performance.TIME_RENDER)

    var particle_time = (time_after - time_before) * 1000
    particle_time_samples.append(particle_time)

    if particle_time_samples.size() > 60:
        var avg = 0.0
        for sample in particle_time_samples:
            avg += sample
        avg /= particle_time_samples.size()
        print("Avg particle time: ", avg, " ms")
        particle_time_samples.clear()
```

**Advanced Optimization Patterns:**
```gdscript
# Pattern 1: Adaptive particle quality based on performance
var target_fps = 60.0
var current_particle_quality = 1.0  # 0.0 to 1.0

func adapt_particle_quality():
    var current_fps = Engine.get_frames_per_second()

    if current_fps < target_fps * 0.9:  # Below target
        current_particle_quality = max(0.3, current_particle_quality - 0.1)
    elif current_fps > target_fps * 1.05:  # Above target
        current_particle_quality = min(1.0, current_particle_quality + 0.05)

    # Apply quality to all active particles
    for emitter in active_particles:
        var base_amount = emitter.get_meta("base_amount", 100)
        emitter.amount = int(base_amount * current_particle_quality)

# Pattern 2: Particle pooling with priority queue
var priority_queue: Array = []  # Sorted by priority

func spawn_particle_prioritized(effect: String, pos: Vector2, priority: int):
    var particle = get_pooled_particle(effect)
    particle.global_position = pos
    particle.set_meta("priority", priority)

    # Insert into priority queue
    priority_queue.append(particle)
    priority_queue.sort_custom(func(a, b): return a.get_meta("priority") > b.get_meta("priority"))

    # Enforce budget - kill lowest priority if over limit
    if priority_queue.size() > MAX_TOTAL_PARTICLES:
        var lowest = priority_queue.pop_back()
        lowest.emitting = false

# Pattern 3: Screen-space particle density limiting
var screen_grid_size = 64  # 64x64 pixel cells
var max_particles_per_cell = 10

func enforce_particle_density(particle: GPUParticles2D):
    var screen_pos = particle.get_global_transform_with_canvas().origin
    var cell = Vector2i(screen_pos / screen_grid_size)

    var cell_key = "%d_%d" % [cell.x, cell.y]
    var count = density_grid.get(cell_key, 0)

    if count >= max_particles_per_cell:
        particle.emitting = false  # Too dense, disable
        return false

    density_grid[cell_key] = count + 1
    return true
```
