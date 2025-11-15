### 13. Struct-of-Arrays vs Array-of-Structs

**Category:** Memory & Data Management

**Problem It Solves:** Array-of-Structs (AoS) scatters related data across memory, causing 40-60% cache miss rate and preventing SIMD vectorization. When updating positions of 10,000 entities, AoS loads entire Enemy struct (64 bytes) per iteration, but only uses position (12 bytes), wasting 81% of cache bandwidth. L1 cache fetches 64-byte cache lines: AoS gets 1 position per line, SoA gets 5.3 positions per line (5.3x better cache utilization).

**Technical Explanation:**
Array-of-Structs stores objects sequentially with all fields together: [Enemy1{pos,vel,health,ai}, Enemy2{...}, ...]. Iterating positions loads all fields into cache, evicting useful data. Struct-of-Arrays stores same fields together: positions[10000], velocities[10000], healths[10000]. Iterating positions loads only position data into cache, achieving 90-95% cache hit rate vs 40-60% for AoS. SoA enables SIMD vectorization: SSE/AVX can process 4-8 positions simultaneously. Modern CPUs have 32KB L1 cache: AoS fills it with 500 enemies (mixed data), SoA fits 2666 positions (pure data).

**Algorithmic Complexity:**
- Memory access: O(n) for both, but SoA has 2-4x better cache efficiency
- Cache utilization: AoS ~40-60% hit rate, SoA ~90-95% hit rate
- SIMD potential: AoS none (mixed data types), SoA 4-8x (vectorized operations)
- Memory footprint: Same total size, but SoA has better locality

**Implementation Pattern:**
```gdscript
# ANTI-PATTERN: Array of Structs (AoS) - Poor cache performance
class Enemy:
    var position: Vector3      # 12 bytes
    var velocity: Vector3      # 12 bytes
    var health: float          # 4 bytes
    var max_health: float      # 4 bytes
    var ai_state: int          # 4 bytes
    var team: int              # 4 bytes
    var target_id: int         # 4 bytes
    # Total: 44 bytes per enemy (padded to 64 for alignment)

var enemies_aos: Array[Enemy] = []

func update_positions_aos(delta: float) -> void:
    # BAD: Loads entire 64-byte Enemy struct per iteration
    # Only uses position (12 bytes) + velocity (12 bytes) = 24 bytes
    # Wastes 40 bytes (62.5%) of cache bandwidth per enemy
    for enemy in enemies_aos:
        enemy.position += enemy.velocity * delta
        # Loads: position, velocity, health, max_health, ai_state, team, target_id
        # Uses: position, velocity only
        # Cache efficiency: ~37.5% (24/64 bytes used)

func update_health_aos() -> void:
    # Even worse: Only needs health (4 bytes) but loads 64 bytes
    for enemy in enemies_aos:
        if enemy.health < 0:
            enemy.health = 0
    # Cache efficiency: 6.25% (4/64 bytes used) - TERRIBLE!

# OPTIMAL: Struct of Arrays (SoA) - Excellent cache performance
class EnemySystemSoA:
    var positions: PackedVector3Array       # All positions contiguous
    var velocities: PackedVector3Array      # All velocities contiguous
    var healths: PackedFloat32Array         # All healths contiguous
    var max_healths: PackedFloat32Array
    var ai_states: PackedInt32Array
    var teams: PackedInt32Array
    var target_ids: PackedInt32Array
    var count: int

    func _init(capacity: int):
        positions.resize(capacity)
        velocities.resize(capacity)
        healths.resize(capacity)
        max_healths.resize(capacity)
        ai_states.resize(capacity)
        teams.resize(capacity)
        target_ids.resize(capacity)
        count = 0

    func add_enemy(pos: Vector3, vel: Vector3, hp: float, state: int, team_id: int) -> int:
        var index = count
        positions[index] = pos
        velocities[index] = vel
        healths[index] = hp
        max_healths[index] = hp
        ai_states[index] = state
        teams[index] = team_id
        target_ids[index] = -1
        count += 1
        return index

    func remove_enemy(index: int) -> void:
        # Swap-and-pop for O(1) removal
        count -= 1
        if index < count:
            positions[index] = positions[count]
            velocities[index] = velocities[count]
            healths[index] = healths[count]
            max_healths[index] = max_healths[count]
            ai_states[index] = ai_states[count]
            teams[index] = teams[count]
            target_ids[index] = target_ids[count]

var enemies_soa: EnemySystemSoA = EnemySystemSoA.new(10000)

func update_positions_soa(delta: float) -> void:
    # GOOD: Only loads position + velocity arrays
    # Cache line (64 bytes) contains 5.3 Vector3s (12 bytes each)
    # vs AoS which contains 1 Enemy (64 bytes)
    # 5.3x better cache utilization!
    for i in range(enemies_soa.count):
        enemies_soa.positions[i] += enemies_soa.velocities[i] * delta
        # Loads: positions[i-5 to i+5] in same cache line (pre-fetched)
        # Cache efficiency: ~100% (all loaded data is used)

func update_health_soa() -> void:
    # EXCELLENT: Only loads health array
    # Cache line contains 16 floats (4 bytes each)
    # vs AoS which contains 1 Enemy (64 bytes) with only 4-byte health
    # 16x better cache utilization!
    for i in range(enemies_soa.count):
        if enemies_soa.healths[i] < 0:
            enemies_soa.healths[i] = 0
    # Cache efficiency: 100% (all 16 floats in cache line are processed)

# HYBRID: Array of Small Structs of Arrays (AoSoA) - Balanced approach
class EnemyAoSoA:
    # Bundle related fields into small groups
    const BUNDLE_SIZE = 16  # Process 16 enemies at once (fits in cache)

    class TransformBundle:
        var positions: PackedVector3Array   # 16 positions = 192 bytes
        var velocities: PackedVector3Array  # 16 velocities = 192 bytes
        # Total: 384 bytes (6 cache lines)

    class CombatBundle:
        var healths: PackedFloat32Array     # 16 healths = 64 bytes (1 cache line)
        var max_healths: PackedFloat32Array
        var teams: PackedInt32Array

    class AIBundle:
        var ai_states: PackedInt32Array
        var target_ids: PackedInt32Array

    var transform_bundles: Array[TransformBundle]
    var combat_bundles: Array[CombatBundle]
    var ai_bundles: Array[AIBundle]

# Real-world example: Particle system
class ParticleSystemSoA:
    var positions: PackedVector3Array
    var velocities: PackedVector3Array
    var colors: PackedColorArray
    var lifetimes: PackedFloat32Array
    var sizes: PackedFloat32Array
    var active_count: int
    var max_particles: int

    func _init(max_count: int):
        max_particles = max_count
        positions.resize(max_count)
        velocities.resize(max_count)
        colors.resize(max_count)
        lifetimes.resize(max_count)
        sizes.resize(max_count)
        active_count = 0

    func update(delta: float) -> void:
        # Phase 1: Update physics (only touches position, velocity)
        for i in range(active_count):
            velocities[i] += Vector3.DOWN * 9.81 * delta  # Gravity
            positions[i] += velocities[i] * delta

        # Phase 2: Update lifetime (only touches lifetimes)
        var i = 0
        while i < active_count:
            lifetimes[i] -= delta
            if lifetimes[i] <= 0:
                # Remove dead particle (swap with last active)
                _remove_particle(i)
            else:
                i += 1

        # Phase 3: Update visuals (only touches colors, sizes)
        for i in range(active_count):
            var life_ratio = lifetimes[i] / 5.0  # Fade over 5 seconds
            colors[i].a = life_ratio
            sizes[i] = life_ratio * 2.0

        # Each phase has perfect cache locality!

    func _remove_particle(index: int) -> void:
        active_count -= 1
        positions[index] = positions[active_count]
        velocities[index] = velocities[active_count]
        colors[index] = colors[active_count]
        lifetimes[index] = lifetimes[active_count]
        sizes[index] = sizes[active_count]

# Benchmark: Demonstrate performance difference
func benchmark_aos_vs_soa():
    const COUNT = 10000
    const ITERATIONS = 1000

    # Setup AoS
    var aos_enemies: Array[Enemy] = []
    for i in range(COUNT):
        var e = Enemy.new()
        e.position = Vector3(randf(), randf(), randf())
        e.velocity = Vector3(randf(), randf(), randf())
        e.health = 100.0
        aos_enemies.append(e)

    # Setup SoA
    var soa_enemies = EnemySystemSoA.new(COUNT)
    for i in range(COUNT):
        soa_enemies.add_enemy(
            Vector3(randf(), randf(), randf()),
            Vector3(randf(), randf(), randf()),
            100.0, 0, 0
        )

    # Benchmark AoS
    var aos_start = Time.get_ticks_usec()
    for _iter in range(ITERATIONS):
        for i in range(COUNT):
            aos_enemies[i].position += aos_enemies[i].velocity * 0.016
    var aos_time = Time.get_ticks_usec() - aos_start

    # Benchmark SoA
    var soa_start = Time.get_ticks_usec()
    for _iter in range(ITERATIONS):
        for i in range(COUNT):
            soa_enemies.positions[i] += soa_enemies.velocities[i] * 0.016
    var soa_time = Time.get_ticks_usec() - soa_start

    var speedup = float(aos_time) / float(soa_time)
    print("AoS time: %.2fms" % (aos_time / 1000.0))
    print("SoA time: %.2fms" % (soa_time / 1000.0))
    print("Speedup: %.2fx" % speedup)
    # Expected: 2-4x speedup on modern CPUs
```

**Key Parameters:**
- Array size: >1000 elements to see significant benefit
- Bundle size (AoSoA): 8-32 elements per bundle (fits in L1 cache)
- Access pattern: Sequential iteration benefits most (90%+ of game logic)
- Field grouping: Group frequently-accessed fields together

**Edge Cases:**
- Random access: SoA loses advantage (but rare in games)
- Small arrays (<100 elements): Overhead not worth it
- Complex queries: Joining multiple arrays requires careful indexing
- Removal: Swap-and-pop works but breaks stable iteration order
- Sparse data: Wasted memory for unused fields (use bitmask for active flags)

**When NOT to Use:**
- Objects accessed randomly (not sequentially)
- Fewer than 100 objects total
- Complex object graphs with many relationships
- Prototyping phase (harder to debug)
- Gameplay code (reserve for performance-critical systems only)

**Examples from Shipped Games:**

1. **Unity DOTS (ECS):** Pure SoA architecture, enables 10-100x speedup for data-heavy systems like crowd simulation, 100K+ entities at 60fps
2. **Unreal Mass Entity System:** SoA-based crowd AI, handles 10,000+ agents at 60fps in Fortnite/Matrix demo, 5-10x faster than traditional actors
3. **Frostbite Engine (DICE):** Hybrid AoSoA for balance, used in Battlefield for 64-player physics/networking, 3-4x cache efficiency improvement
4. **SimCity (2013):** SoA for agent simulation, 100,000+ sims in city updating at 30fps, enabled massive scale on 2013 hardware
5. **Hitman Series (IO Interactive):** SoA crowd system with 1000+ NPCs, each with full AI routines, maintains 60fps on consoles

**Platform Considerations:**
- **PC (x86/x64):** Large L1 cache (32-64KB), SoA provides 2-4x speedup, SIMD 4-8 wide
- **Mobile (ARM):** Smaller caches (16-32KB), SoA even more critical, NEON 4-wide SIMD
- **Consoles (PS5/Xbox):** Optimized for streaming data, SoA matches hardware design
- **VR:** High frame rates (90-120fps) require maximum efficiency, SoA essential

**Godot-Specific Notes:**
- Use PackedArrays (PackedVector3Array, PackedFloat32Array) for SoA - optimized in C++
- Regular Arrays are less efficient (use Packed variants)
- GDScript iteration of PackedArrays is fast (C++ backend)
- Consider GDExtension for maximum SoA performance (custom SIMD)
- Godot's internal rendering already uses SoA for mesh data (vertices, normals)

**Synergies:**
- Essential for ECS (Entity Component System) architecture
- Pairs with Job System (parallel processing of arrays)
- Enables SIMD Vectorization (process 4-8 elements simultaneously)
- Combines with Cache-Friendly Algorithms (spatial locality)
- Critical for Particle Systems (thousands of particles updated per frame)

**Measurement/Profiling:**
```gdscript
func profile_cache_performance():
    # Cache misses are invisible in simple timing
    # Need CPU performance counters or careful benchmark

    const COUNT = 10000
    var aos_time = _benchmark_aos(COUNT)
    var soa_time = _benchmark_soa(COUNT)

    print("Cache efficiency comparison:")
    print("  AoS: %.2f ns/element" % (aos_time / COUNT))
    print("  SoA: %.2f ns/element" % (soa_time / COUNT))
    print("  Speedup: %.2fx" % (aos_time / soa_time))

    # Expected results:
    # AoS: 20-40 ns/element (cache misses)
    # SoA: 5-10 ns/element (cache hits)
    # Speedup: 2-4x

# Monitor memory bandwidth
func estimate_memory_bandwidth():
    const DATA_SIZE = 10000 * 12  # 10K Vector3s = 120KB
    var iterations = 1000

    var start = Time.get_ticks_usec()
    # Memory-bound operation
    for _i in range(iterations):
        for j in range(10000):
            positions[j] = positions[j] + Vector3.ONE
    var elapsed = Time.get_ticks_usec() - start

    var bytes_processed = DATA_SIZE * iterations * 2  # Read + write
    var bandwidth_gbps = (bytes_processed / 1_000_000_000.0) / (elapsed / 1_000_000.0)

    print("Memory bandwidth: %.2f GB/s" % bandwidth_gbps)
    # Expected: 5-20 GB/s depending on cache level
```

**Target Metrics:**
- Cache hit rate: >90% for SoA vs 40-60% for AoS
- Performance: 2-4x speedup for typical update loops
- Memory bandwidth: Utilize 50-80% of theoretical L1 cache bandwidth
- SIMD efficiency: 4-8x throughput when combined with vectorization
