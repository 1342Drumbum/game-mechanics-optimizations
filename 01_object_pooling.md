### 1. Object Pooling for Bullets/Particles/Enemies

**Category:** Performance - Memory Management

**Problem It Solves:** Garbage collection spikes causing frame stutters. Creating/destroying 100-1000+ objects per second triggers frequent GC pauses (30-60ms spikes), breaking 60fps target (16.67ms budget).

**Technical Explanation:**
Pre-allocate arrays of objects at game start, activate/deactivate instead of instantiate/free. When object "dies", return to pool instead of freeing. When spawning, grab from pool instead of allocating new. O(1) retrieval vs O(log n) allocation. Eliminates allocation pressure and fragmentation.

**Algorithmic Complexity:** O(1) get/return vs O(log n) malloc/free

**Implementation Pattern:**
```gdscript
# Godot implementation
class_name ObjectPool

var pool: Array = []
var prefab: PackedScene
var initial_size: int

func _init(p_prefab, p_size):
    prefab = p_prefab
    initial_size = p_size
    for i in range(initial_size):
        var obj = prefab.instantiate()
        obj.set_process(false)
        obj.hide()
        pool.append(obj)

func get_object() -> Node:
    if pool.is_empty():
        return prefab.instantiate()  # Grow if needed
    var obj = pool.pop_back()
    obj.set_process(true)
    obj.show()
    return obj

func return_object(obj: Node):
    obj.set_process(false)
    obj.hide()
    pool.append(obj)
```

**Key Parameters:**
- Bullets: 500-2000 pool size
- Particles: 5000-50000
- Enemies: 50-500
- Growth: 1.5x when exhausted

**Edge Cases:**
- Pool exhaustion: Grow by 50% or reuse oldest
- Memory: Monitor total pool memory (<100MB typical)
- Cleanup: Free pools on level unload

**When NOT to Use:**
- Objects vary greatly (100+ unique types)
- Rare spawning (<10/sec)
- Prototyping phase

**Examples from Shipped Games:**

1. **Unity Standard (ObjectPool<T>):** 50-90% GC reduction in particle-heavy games, 30ms spikes → <1ms
2. **Industry Standard:** 95%+ of commercial engines use pooling for projectiles/particles
3. **Doom 2016:** Extensive pooling for projectiles, demons, particle systems
4. **Dead Cells:** All enemies/projectiles pooled, essential for 60fps on Switch
5. **Celeste:** Pooling for particles, strawberries, platforms - no GC during gameplay

**Platform Considerations:**
- **Mobile:** Critical - GC pauses 50-100ms, aim for zero allocations during gameplay
- **Desktop:** Important - reduces stutters, improves minimum frametime
- **Console:** Essential - fixed memory, can't afford GC pauses
- **Memory:** 2-5x more memory than dynamic, but worth it for performance

**Godot-Specific Notes:**
- Use `queue_free()` carefully - still triggers deferred deletion
- Consider `Object.new()` for pure data objects
- Node pooling: Keep in scene tree but invisible/disabled
- Godot 4.x: Reference counting more efficient than Godot 3.x but still pool

**Synergies:**
- Pairs with Spatial Partitioning (pooled objects in grid)
- Combines with Dirty Flags (only update active objects)
- Essential for Particle Systems optimization

**Measurement/Profiling:**
- Monitor `Monitors.OBJECT_COUNT` in Godot
- Track `Performance.get_monitor(MEMORY_STATIC)`
- Use Godot Profiler: Check allocation frequency
- Target: Zero allocations during intense gameplay sections
