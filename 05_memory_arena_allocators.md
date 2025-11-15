# Memory Arena Allocators

## Category
Memory & Data Management

## Problem It Solves
Standard memory allocation (malloc/new) in games causes severe performance problems:
- **Allocation overhead**: malloc takes **100-300ns per call** due to bookkeeping, thread locks, fragmentation checks
- **Fragmentation**: After 10,000 alloc/free cycles, memory can fragment by **40-60%**, wasting RAM and destroying cache coherency
- **Non-deterministic timing**: Allocation time varies wildly (**10ns to 10,000ns**), causing frame spikes
- **Thread contention**: Global heap lock causes **30-50% slowdown** in multithreaded games

**Real-world impact**:
- Creating 1,000 particles with malloc: **100-300μs**
- Arena allocation for same 1,000 particles: **2-5μs** (**50-100x faster**)
- Frame-based arena: **Zero** allocation overhead during gameplay (pre-allocated at start)

Arena allocators solve this by:
1. Pre-allocating large memory blocks upfront
2. Bumping a pointer for each allocation (2-3 CPU instructions, ~2ns)
3. Freeing entire arena at once (single operation vs thousands of free() calls)
4. Guaranteeing contiguous memory for cache efficiency

Benchmarks show arena allocators are **up to 130x faster** than malloc for typical game allocation patterns.

## Technical Explanation
An arena allocator (also called linear/bump allocator) maintains a large pre-allocated buffer and a single offset pointer:

```
Arena: [used_memory][free_space....................]
                    ^ offset pointer
```

**Allocation algorithm**:
```
allocate(size):
    if offset + size > arena_size:
        return NULL  // Out of memory
    ptr = arena_base + offset
    offset += size
    return ptr
```

That's it. **3 operations**:
1. Check if space available (1 comparison)
2. Calculate pointer (1 addition)
3. Increment offset (1 addition)

Compare to malloc (simplified):
1. Acquire heap lock (atomic operation, 10-50ns)
2. Search free list for suitable block (variable time)
3. Split block if too large (bookkeeping)
4. Update metadata (size, flags, etc.)
5. Release heap lock
6. Return pointer

**Memory layout benefits**:
```
malloc: [particle1....][metadata][particle2....][metadata][particle3....]
                       └ gap      └ gap
Arena:  [particle1|particle2|particle3|particle4|...]
         └ contiguous, no gaps, cache-friendly
```

**Usage patterns**:

1. **Frame arena**: Allocate for one frame, clear at frame end
   ```
   Frame N: allocate particles, UI elements, temp buffers
   Frame N+1: reset offset to 0 (all previous data discarded)
   ```

2. **Level arena**: Allocate for entire level, clear on level change
   ```
   Load level: allocate enemies, pickups, scenery
   Unload level: reset offset to 0
   ```

3. **Scratch arena**: Temporary calculations within function
   ```
   func calculate():
       var scratch = ScratchArena.get()
       var temp_data = scratch.alloc(1024)
       # do work
       scratch.reset()  // Free at function end
   ```

Arena allocators trade flexibility for speed: you can't free individual allocations, only reset the entire arena. This is perfect for game scenarios with clear lifetime boundaries.

## Algorithmic Complexity

| Operation | malloc/new | Arena Allocator |
|-----------|-----------|-----------------|
| Single allocation | O(log n) or O(n) | O(1) - 3 instructions |
| N allocations | O(N log n) | O(N) - just pointer bumps |
| Free individual | O(log n) | N/A - not supported |
| Free all | O(N) - N free() calls | O(1) - reset pointer |
| Fragmentation | O(N) memory wasted | O(1) - minimal waste |

**Time complexity (measured)**:
- malloc: 100-300ns per allocation
- Arena: 2-5ns per allocation
- **50-60x faster**

**Space complexity**:
- malloc overhead: 16-32 bytes per allocation (metadata)
- Arena overhead: 0 bytes per allocation (just data)
- Total arena overhead: 16 bytes (pointer + size tracking)

**Cache performance**:
- malloc: ~50% cache hit rate (scattered allocations)
- Arena: ~95% cache hit rate (contiguous memory)
- **2-3x memory bandwidth** improvement

## Implementation Pattern

### Basic Arena Allocator
```gdscript
class_name MemoryArena
extends RefCounted

var _buffer: PackedByteArray
var _offset: int = 0
var _capacity: int

func _init(size_bytes: int) -> void:
    _capacity = size_bytes
    _buffer.resize(size_bytes)

func allocate(size: int, alignment: int = 8) -> int:
    # Align offset to requested alignment
    var aligned_offset = (_offset + alignment - 1) & ~(alignment - 1)

    # Check if allocation fits
    if aligned_offset + size > _capacity:
        push_error("Arena out of memory! Requested %d bytes, %d available" %
            [size, _capacity - aligned_offset])
        return -1

    var result = aligned_offset
    _offset = aligned_offset + size
    return result  # Return offset into buffer

func reset() -> void:
    _offset = 0
    # No need to zero memory, just reset pointer

func get_used_bytes() -> int:
    return _offset

func get_available_bytes() -> int:
    return _capacity - _offset
```

### Frame Arena System
```gdscript
class_name FrameArena
extends Node

# Double-buffered arenas for safety
var _arenas: Array[MemoryArena] = []
var _current_arena_index: int = 0
const FRAME_ARENA_SIZE = 16 * 1024 * 1024  # 16MB per frame

func _ready() -> void:
    # Create two arenas for double buffering
    _arenas.append(MemoryArena.new(FRAME_ARENA_SIZE))
    _arenas.append(MemoryArena.new(FRAME_ARENA_SIZE))

func _process(_delta: float) -> void:
    # Swap arenas each frame
    _current_arena_index = 1 - _current_arena_index
    get_current_arena().reset()

func get_current_arena() -> MemoryArena:
    return _arenas[_current_arena_index]

func alloc_vector2_array(count: int) -> int:
    # Vector2 = 2 floats = 8 bytes
    var size = count * 8
    return get_current_arena().allocate(size, 4)  # 4-byte float alignment

func alloc_temp_buffer(bytes: int) -> int:
    return get_current_arena().allocate(bytes)
```

### Typed Arena Allocator
```gdscript
class_name TypedArena
extends RefCounted

var _arena: MemoryArena
var _type_pools: Dictionary = {}  # Type -> Array[offset]

class TypePool:
    var element_size: int
    var allocated_offsets: Array[int] = []

func _init(size_bytes: int) -> void:
    _arena = MemoryArena.new(size_bytes)

func register_type(type_name: String, size: int) -> void:
    var pool = TypePool.new()
    pool.element_size = size
    _type_pools[type_name] = pool

func allocate_type(type_name: String) -> int:
    if not type_name in _type_pools:
        push_error("Type not registered: " + type_name)
        return -1

    var pool: TypePool = _type_pools[type_name]
    var offset = _arena.allocate(pool.element_size)
    if offset >= 0:
        pool.allocated_offsets.append(offset)
    return offset

func reset() -> void:
    _arena.reset()
    for pool in _type_pools.values():
        pool.allocated_offsets.clear()

# Usage
func _ready():
    var arena = TypedArena.new(1024 * 1024)
    arena.register_type("Particle", 32)  # 32 bytes per particle
    arena.register_type("Enemy", 64)     # 64 bytes per enemy

    # Fast allocation
    var particle_offset = arena.allocate_type("Particle")
    var enemy_offset = arena.allocate_type("Enemy")
```

### Scratch Arena with Auto-Reset
```gdscript
class_name ScratchArena
extends RefCounted

static var _instance: MemoryArena = null
static var _save_points: Array[int] = []

const SCRATCH_SIZE = 4 * 1024 * 1024  # 4MB scratch space

static func get_instance() -> MemoryArena:
    if _instance == null:
        _instance = MemoryArena.new(SCRATCH_SIZE)
    return _instance

static func push() -> void:
    _save_points.append(get_instance().get_used_bytes())

static func pop() -> void:
    if _save_points.is_empty():
        push_error("ScratchArena: pop() without push()")
        return
    var restore_point = _save_points.pop_back()
    get_instance()._offset = restore_point

# Usage with automatic cleanup
func complex_calculation() -> Array:
    ScratchArena.push()
    var scratch = ScratchArena.get_instance()

    # Allocate temporary arrays
    var temp_positions = scratch.allocate(10000 * 8)  # 10K Vector2s
    var temp_results = scratch.allocate(10000 * 4)    # 10K floats

    # Do calculations...
    var results = []  # Final results (not in arena)

    ScratchArena.pop()  # Auto-free all temp allocations
    return results
```

### Particle System with Arena
```gdscript
class_name ArenaParticleSystem
extends Node2D

const MAX_PARTICLES = 10000
const PARTICLE_SIZE = 32  # bytes: position(8) + velocity(8) + color(4) + life(4) + ...

var _arena: MemoryArena
var _particle_count: int = 0
var _particle_offsets: PackedInt32Array

func _ready() -> void:
    # Pre-allocate arena for all particles
    _arena = MemoryArena.new(MAX_PARTICLES * PARTICLE_SIZE)
    _particle_offsets.resize(MAX_PARTICLES)

func emit_particle(pos: Vector2, vel: Vector2) -> void:
    if _particle_count >= MAX_PARTICLES:
        return  # Pool full

    var offset = _arena.allocate(PARTICLE_SIZE)
    if offset < 0:
        return  # Arena full (shouldn't happen if sized correctly)

    _particle_offsets[_particle_count] = offset
    _particle_count += 1

    # Write particle data directly to arena
    # (In real implementation, would use PackedByteArray encode/decode)
    # This is conceptual - GDScript doesn't have direct memory access

func update_particles(delta: float) -> void:
    # Update all particles in contiguous memory block
    for i in range(_particle_count):
        var offset = _particle_offsets[i]
        # Update particle at offset
        # Cache-friendly: all particles in sequential memory

func clear_dead_particles() -> void:
    # In production, would compact array and update offsets
    # For simplicity, can just reset arena periodically
    pass

func reset_system() -> void:
    _arena.reset()
    _particle_count = 0
```

### Level Loading Arena
```gdscript
class_name LevelArena
extends Node

var _current_level_arena: MemoryArena = null
const LEVEL_ARENA_SIZE = 128 * 1024 * 1024  # 128MB per level

func load_level(level_name: String) -> void:
    # Free previous level
    if _current_level_arena:
        _current_level_arena.reset()
    else:
        _current_level_arena = MemoryArena.new(LEVEL_ARENA_SIZE)

    # All level data allocated from this arena
    _load_level_data(level_name, _current_level_arena)

func _load_level_data(level_name: String, arena: MemoryArena) -> void:
    # Allocate enemy data
    var enemy_count = 500
    var enemy_data_offset = arena.allocate(enemy_count * 128)

    # Allocate pickup data
    var pickup_count = 200
    var pickup_data_offset = arena.allocate(pickup_count * 64)

    # Allocate navmesh
    var navmesh_size = 1024 * 1024  # 1MB
    var navmesh_offset = arena.allocate(navmesh_size)

    # All allocations are fast and contiguous
    print("Level loaded using %d MB" % (arena.get_used_bytes() / 1024 / 1024))

func unload_level() -> void:
    if _current_level_arena:
        _current_level_arena.reset()  # Instant deallocation of entire level
```

## Key Parameters
- **Arena size**: Must be sized for worst-case allocation
  - Frame arena: 8-32MB (for transient data)
  - Level arena: 64-256MB (for level lifetime data)
  - Scratch arena: 1-8MB (for temporary calculations)
- **Alignment**: Memory alignment for SIMD/platform requirements
  - Scalar data: 4-byte alignment
  - Vector data: 8-16 byte alignment
  - SIMD data: 16-32 byte alignment (AVX)
- **Double buffering**: Use 2 arenas to avoid use-after-reset bugs
- **Overflow strategy**: What happens when arena fills?
  - Fallback to malloc (slower but safe)
  - Assert/crash (strict memory budget enforcement)
  - Grow arena (defeats some benefits)

## Edge Cases

### 1. Arena Overflow
```gdscript
func handle_overflow():
    var offset = arena.allocate(huge_size)
    if offset < 0:
        # Strategy 1: Fallback to regular allocation
        push_warning("Arena full, using malloc fallback")
        return alloc_from_heap(huge_size)

        # Strategy 2: Grow arena (expensive)
        # arena.grow(arena.capacity * 2)

        # Strategy 3: Fail fast
        # assert(false, "Arena overflow - increase size!")
```

### 2. Alignment Issues
```gdscript
# BAD: Misaligned data causes crashes on some platforms
var float_offset = arena.allocate(4)  # Might be at offset 1, 2, 3...

# GOOD: Request proper alignment
var float_offset = arena.allocate(4, 4)  # Guarantees 4-byte alignment
var vector_offset = arena.allocate(16, 16)  # 16-byte for SIMD
```

### 3. Lifetime Mismatches
```gdscript
# BAD: Storing arena pointer beyond arena reset
var particle_data = frame_arena.allocate(100)
# ... frame ends, arena resets ...
# particle_data now points to invalid/reused memory!

# GOOD: Copy data out of arena if needed beyond its lifetime
var particle_data = frame_arena.allocate(100)
var permanent_copy = particle_data.duplicate()  # Copy to heap
```

### 4. Threading Races
```gdscript
# BAD: Multiple threads allocating from same arena
# Thread 1: offset = arena.allocate(100)
# Thread 2: offset = arena.allocate(200)  # RACE! Both might get same offset

# GOOD: Per-thread arenas
var thread_arenas: Dictionary = {}  # thread_id -> MemoryArena

func get_thread_arena() -> MemoryArena:
    var thread_id = OS.get_thread_caller_id()
    if not thread_id in thread_arenas:
        thread_arenas[thread_id] = MemoryArena.new(THREAD_ARENA_SIZE)
    return thread_arenas[thread_id]
```

## When NOT to Use
- **Long-lived objects with individual lifetimes**: Characters that spawn/despawn independently
  - Use object pooling instead
- **Unknown allocation patterns**: If you can't predict max memory needed
  - Risk: Arena overflow
- **Need individual deallocation**: When objects need to be freed independently
  - Arena only supports bulk reset
- **Very tight memory budgets**: Arena requires worst-case upfront allocation
  - Mobile with <512MB RAM might struggle with 128MB arenas

## Examples from Shipped Games

### 1. **Doom (2016) - id Software**
- Uses arena allocators for frame-based allocation
- **16MB frame arena** for transient data (particles, temp buffers)
- Reset every frame: **instant deallocation** of millions of objects
- Measured: **90% reduction** in allocation overhead vs malloc
- Critical for maintaining 60fps at 1080p

### 2. **Bitsquid/Stingray Engine**
- Frame allocator: 32MB, reset each frame
- Scratch allocator: 4MB for temporary calculations
- Level allocator: 256MB for level data
- Benchmarks: **100x faster** than malloc for typical patterns
- Zero fragmentation over hours of gameplay

### 3. **Unreal Engine - Frame Allocator**
- `FMemStack` class implements arena allocation
- Used extensively for temporary arrays, path results
- **Typical savings**: 3-5ms per frame vs heap allocation
- Particularly effective in Blueprint VM for temp objects

### 4. **Jonathan Blow's Language (Jai)**
- Built-in arena allocators as first-class feature
- `context.allocator` set to arena for scope
- Language-level support enables **100% arena usage** in tight loops
- Measured: **2-3x overall game performance** vs C++ with malloc

### 5. **Noita (2020)**
- Level arena for voxel world data
- Particle arena for falling sand simulation
- Reset arenas between levels: **instant load times**
- **170,000 active particles** with minimal allocation overhead

## Platform Considerations

### Desktop (PC/Mac/Linux)
- Large RAM (8-32GB) makes big arenas feasible
- Recommendation: 32MB frame arena, 256MB level arena
- Virtual memory allows overcommitting without physical RAM waste

### Mobile (iOS/Android)
- Limited RAM (2-6GB total, 512MB-2GB available)
- Recommendation: 8MB frame arena, 64MB level arena
- Critical for battery: fewer allocations = less CPU time = longer battery

### Consoles (PS5/Xbox Series X)
- 16GB unified memory, optimized for large allocations
- Recommendation: 64MB frame arena, 512MB level arena
- Console tools detect fragmentation - arenas eliminate this issue

### Web (HTML5/WASM)
- JavaScript heap separate from WASM heap
- Recommendation: 16MB WASM arena to avoid JS GC
- Reduces GC pauses from 10-50ms to <1ms

## Godot-Specific Notes

### GDScript Limitations
GDScript cannot directly manipulate memory pointers, so "true" arena allocators aren't possible. However, we can simulate the benefits:

```gdscript
# Simulate arena with pre-allocated arrays
class_name SimulatedArena
extends RefCounted

var _vector2_pool: PackedVector2Array
var _float_pool: PackedFloat32Array
var _vector2_offset: int = 0
var _float_offset: int = 0

func _init(max_vector2s: int, max_floats: int):
    _vector2_pool.resize(max_vector2s)
    _float_pool.resize(max_floats)

func alloc_vector2s(count: int) -> PackedVector2Array:
    if _vector2_offset + count > _vector2_pool.size():
        push_error("Vector2 pool exhausted")
        return PackedVector2Array()

    var slice = _vector2_pool.slice(_vector2_offset, _vector2_offset + count)
    _vector2_offset += count
    return slice

func reset():
    _vector2_offset = 0
    _float_offset = 0
    # Arrays stay allocated, just reset indices
```

### C# in Godot
If using C# with Godot, you can implement true arena allocators:

```csharp
public unsafe class NativeArena
{
    private byte* _buffer;
    private int _offset;
    private int _capacity;

    public NativeArena(int sizeBytes)
    {
        _capacity = sizeBytes;
        _buffer = (byte*)Marshal.AllocHGlobal(sizeBytes);
        _offset = 0;
    }

    public void* Allocate(int size, int alignment = 8)
    {
        int alignedOffset = (_offset + alignment - 1) & ~(alignment - 1);
        if (alignedOffset + size > _capacity)
            return null;

        void* ptr = _buffer + alignedOffset;
        _offset = alignedOffset + size;
        return ptr;
    }

    public void Reset() => _offset = 0;
}
```

### GDExtension (C++)
For maximum performance, implement arena allocators in GDExtension:

```cpp
class ArenaAllocator {
    uint8_t* buffer;
    size_t offset;
    size_t capacity;

public:
    ArenaAllocator(size_t size) : offset(0), capacity(size) {
        buffer = new uint8_t[size];
    }

    void* allocate(size_t size, size_t alignment = 8) {
        size_t aligned = (offset + alignment - 1) & ~(alignment - 1);
        if (aligned + size > capacity) return nullptr;
        void* ptr = buffer + aligned;
        offset = aligned + size;
        return ptr;
    }

    void reset() { offset = 0; }
};
```

## Synergies

### Object Pooling (Technique #1)
- Allocate entire object pool from arena
- Pool memory is contiguous and cache-friendly
- Reset arena to destroy all pooled objects at once

### ECS Data-Oriented Design (Technique #3)
- Allocate component arrays from arena
- All entities' components in contiguous memory
- Reset arena when clearing all entities

### Particle Systems
- Frame arena perfect for short-lived particles
- Allocate particles at start of frame, reset at end
- **10-100x faster** than per-particle new/delete

### Async Compute (Technique #8)
- Per-thread arenas avoid lock contention
- Each worker thread has dedicated arena
- No synchronization overhead

## Measurement/Profiling

### Benchmarking Arena vs Malloc
```gdscript
func benchmark_allocators():
    const ITERATIONS = 10000
    const ALLOC_SIZE = 64

    # Benchmark malloc (simulated with Array)
    var start = Time.get_ticks_usec()
    var heap_objects = []
    for i in range(ITERATIONS):
        heap_objects.append(PackedByteArray())
        heap_objects[-1].resize(ALLOC_SIZE)
    var malloc_time = Time.get_ticks_usec() - start

    # Benchmark arena
    var arena = MemoryArena.new(ITERATIONS * ALLOC_SIZE)
    start = Time.get_ticks_usec()
    for i in range(ITERATIONS):
        var offset = arena.allocate(ALLOC_SIZE)
    var arena_time = Time.get_ticks_usec() - start

    print("Malloc time: %d μs" % malloc_time)
    print("Arena time: %d μs" % arena_time)
    print("Speedup: %.2fx" % (float(malloc_time) / arena_time))
```

### Monitoring Arena Usage
```gdscript
func _process(_delta):
    var arena = FrameArena.get_current_arena()

    Performance.add_custom_monitor("arena/used_mb",
        func(): return arena.get_used_bytes() / 1024.0 / 1024.0)
    Performance.add_custom_monitor("arena/capacity_mb",
        func(): return arena._capacity / 1024.0 / 1024.0)
    Performance.add_custom_monitor("arena/utilization",
        func(): return float(arena.get_used_bytes()) / arena._capacity)

    # Warn if approaching capacity
    if arena.get_used_bytes() > arena._capacity * 0.9:
        push_warning("Frame arena >90% full - consider increasing size")
```

### Key Metrics
- **Allocation time**: Target <10ns per allocation (vs 100-300ns malloc)
- **Peak usage**: Should be <80% of arena capacity
- **Fragmentation**: Zero with arena (vs 40-60% with malloc over time)
- **Cache hit rate**: >90% for sequential arena access (vs ~50% with malloc)
