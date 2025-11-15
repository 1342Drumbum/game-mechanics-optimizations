### 15. Copy-on-Write Data Structures

**Category:** Memory & Data Management

**Problem It Solves:** Copying entire game state for undo/replay systems, multithreading, or time-rewind mechanics wastes 80-95% memory on unchanged data. Full-copying 10MB game state 60 times/second = 600MB/s memory bandwidth and triggers GC pressure. Copy-on-Write (CoW) shares immutable data between copies, only duplicating modified portions, reducing memory overhead from 100% to 5-20% and enabling instant O(1) "snapshots" instead of O(n) full copies.

**Technical Explanation:**
CoW delays copying until write occurs. When creating snapshot, share pointer to original data with reference count. On read, access shared data directly (zero cost). On write, check if data is shared (ref count > 1): if yes, copy data first (expensive), then modify copy; if no, modify in place (cheap). Persistent data structures use tree-based CoW: modifications create new tree nodes sharing subtrees with original, achieving O(log n) time and space for changes. Common implementations: immutable strings, copy-on-write vectors, persistent hash maps. Critical for functional programming, version control systems (Git uses CoW file trees), and undo systems.

**Algorithmic Complexity:**
- Read: O(1) - same as normal access, just follows pointer
- Write (first to unique data): O(1) - modify in place
- Write (to shared data): O(n) for array copy, O(log n) for tree-based structures
- Space: O(k) where k = modified elements, not O(n) full size
- Snapshot: O(1) - just increment reference count

**Implementation Pattern:**
```gdscript
# Basic Copy-on-Write wrapper for arrays
class_name CoWArray
extends RefCounted

var _data: Array
var _is_unique: bool = true
var _ref_count: int = 1

func _init(initial_data: Array = []) -> void:
    _data = initial_data.duplicate()

func clone() -> CoWArray:
    # Create shallow copy that shares underlying data
    var new_cow = CoWArray.new()
    new_cow._data = _data  # Share reference
    new_cow._is_unique = false
    _is_unique = false
    _ref_count += 1
    new_cow._ref_count = _ref_count
    return new_cow

func size() -> int:
    return _data.size()

func get_at(index: int) -> Variant:
    # Read is free - no copy needed
    return _data[index]

func set_at(index: int, value: Variant) -> void:
    # Write triggers copy if shared
    _ensure_unique()
    _data[index] = value

func append(value: Variant) -> void:
    _ensure_unique()
    _data.append(value)

func insert(index: int, value: Variant) -> void:
    _ensure_unique()
    _data.insert(index, value)

func remove_at(index: int) -> void:
    _ensure_unique()
    _data.remove_at(index)

func _ensure_unique() -> void:
    # Copy data if shared with other references
    if not _is_unique:
        _data = _data.duplicate()
        _is_unique = true
        _ref_count = 1

# Optimized CoW for packed arrays (better performance)
class_name CoWPackedVector3Array
extends RefCounted

var _data: PackedVector3Array
var _is_shared: bool = false

func _init(initial_data: PackedVector3Array = PackedVector3Array()) -> void:
    _data = initial_data

func clone() -> CoWPackedVector3Array:
    var new_cow = CoWPackedVector3Array.new()
    new_cow._data = _data  # Share reference
    new_cow._is_shared = true
    _is_shared = true
    return new_cow

func get_at(index: int) -> Vector3:
    return _data[index]

func set_at(index: int, value: Vector3) -> void:
    if _is_shared:
        _data = _data.duplicate()
        _is_shared = false
    _data[index] = value

func size() -> int:
    return _data.size()

# Persistent tree-based CoW for efficient modifications
class_name CoWPersistentTree
extends RefCounted

class Node:
    var value: Variant
    var left: Node = null
    var right: Node = null

    func _init(v: Variant) -> void:
        value = v

    func clone() -> Node:
        # Shallow clone - shares children
        var new_node = Node.new(value)
        new_node.left = left
        new_node.right = right
        return new_node

var _root: Node = null
var _size: int = 0

func set_value(index: int, value: Variant) -> CoWPersistentTree:
    # Returns NEW tree sharing most nodes with original
    var new_tree = CoWPersistentTree.new()
    new_tree._root = _set_recursive(_root, index, value, 0)
    new_tree._size = _size
    return new_tree

func _set_recursive(node: Node, index: int, value: Variant, depth: int) -> Node:
    if node == null:
        return Node.new(value)

    # Clone current node (path copying)
    var new_node = node.clone()

    # Recurse down appropriate branch
    var mid = _size / (1 << (depth + 1))
    if index < mid:
        new_node.left = _set_recursive(node.left, index, value, depth + 1)
    else:
        new_node.right = _set_recursive(node.right, index - mid, value, depth + 1)

    return new_node

# Real-world example: Game state snapshot for undo system
class GameState:
    var entities: CoWArray
    var player_position: Vector3
    var player_health: int
    var world_time: float

    func _init():
        entities = CoWArray.new()

    func clone() -> GameState:
        var new_state = GameState.new()
        new_state.entities = entities.clone()  # CoW: shares until modified
        new_state.player_position = player_position
        new_state.player_health = player_health
        new_state.world_time = world_time
        return new_state

class UndoSystem:
    var history: Array[GameState] = []
    var current_state: GameState
    var max_history: int = 100

    func _init(initial_state: GameState):
        current_state = initial_state

    func record_state() -> void:
        # Snapshot current state (instant with CoW)
        var snapshot = current_state.clone()
        history.append(snapshot)

        # Limit history size
        if history.size() > max_history:
            history.pop_front()

    func undo() -> GameState:
        if history.is_empty():
            return current_state

        current_state = history.pop_back()
        return current_state

    func get_memory_usage() -> int:
        # CoW means history uses far less memory than full copies
        # Rough estimate: only modified data is duplicated
        return history.size() * 1024  # Assume ~1KB per snapshot (vs 100KB for full copy)

# Replay system with CoW
class ReplaySystem:
    var frame_states: Array[GameState] = []
    var current_frame: int = 0

    func record_frame(state: GameState) -> void:
        # Record frame with CoW - only stores differences
        var snapshot = state.clone()
        frame_states.append(snapshot)

        # Memory efficient: 10,000 frames * ~1KB = 10MB
        # vs full copies: 10,000 frames * 100KB = 1GB

    func playback_frame(frame: int) -> GameState:
        if frame < 0 or frame >= frame_states.size():
            return null
        return frame_states[frame]

    func scrub_to_frame(frame: int) -> void:
        # Instant seek to any frame (no recomputation needed)
        current_frame = clampi(frame, 0, frame_states.size() - 1)

# Advanced: Persistent hash map with CoW
class CoWHashMap:
    # Simplified persistent hash map for key-value storage
    # Production version would use HAMT (Hash Array Mapped Trie)

    var _buckets: Array[Array] = []
    var _is_shared: bool = false
    var _count: int = 0
    const BUCKET_COUNT = 32

    func _init():
        for i in range(BUCKET_COUNT):
            _buckets.append([])

    func clone() -> CoWHashMap:
        var new_map = CoWHashMap.new()
        new_map._buckets = _buckets  # Share buckets
        new_map._is_shared = true
        new_map._count = _count
        _is_shared = true
        return new_map

    func set_value(key: Variant, value: Variant) -> void:
        _ensure_unique()

        var hash_val = key.hash()
        var bucket_idx = hash_val % BUCKET_COUNT
        var bucket = _buckets[bucket_idx]

        # Find or insert key
        for i in range(bucket.size()):
            if bucket[i][0] == key:
                bucket[i][1] = value
                return

        bucket.append([key, value])
        _count += 1

    func get_value(key: Variant, default: Variant = null) -> Variant:
        var hash_val = key.hash()
        var bucket_idx = hash_val % BUCKET_COUNT
        var bucket = _buckets[bucket_idx]

        for pair in bucket:
            if pair[0] == key:
                return pair[1]

        return default

    func _ensure_unique() -> void:
        if _is_shared:
            # Deep copy buckets
            var new_buckets = []
            for bucket in _buckets:
                new_buckets.append(bucket.duplicate())
            _buckets = new_buckets
            _is_shared = false

# Time-rewind mechanic (like Braid game)
class TimeRewindSystem:
    var entity_histories: Dictionary = {}  # Entity ID -> Array[EntityState]
    var max_rewind_seconds: float = 10.0
    var snapshot_rate: int = 60  # Snapshots per second

    class EntityState:
        var position: Vector3
        var velocity: Vector3
        var state_flags: int

    func record_entity(entity_id: int, position: Vector3, velocity: Vector3, flags: int) -> void:
        if not entity_histories.has(entity_id):
            entity_histories[entity_id] = []

        var state = EntityState.new()
        state.position = position
        state.velocity = velocity
        state.state_flags = flags

        var history = entity_histories[entity_id]
        history.append(state)

        # Limit history length
        var max_snapshots = int(max_rewind_seconds * snapshot_rate)
        if history.size() > max_snapshots:
            history.pop_front()

    func rewind_entity(entity_id: int, frames_back: int) -> EntityState:
        if not entity_histories.has(entity_id):
            return null

        var history = entity_histories[entity_id]
        var index = max(0, history.size() - frames_back)
        return history[index] if index < history.size() else null

# Benchmark: Compare full copy vs CoW
func benchmark_cow_vs_full_copy():
    const ARRAY_SIZE = 10000
    const MODIFICATIONS = 100
    const ITERATIONS = 100

    # Setup test data
    var test_array = []
    for i in range(ARRAY_SIZE):
        test_array.append(Vector3(randf(), randf(), randf()))

    # Benchmark full copy
    var full_copy_time = 0
    var full_copy_memory = 0

    var start = Time.get_ticks_usec()
    for iter in range(ITERATIONS):
        # Full copy
        var copy = test_array.duplicate()
        full_copy_memory += ARRAY_SIZE * 12  # 12 bytes per Vector3

        # Modify a few elements
        for i in range(MODIFICATIONS):
            copy[i] = Vector3.ZERO
    full_copy_time = Time.get_ticks_usec() - start

    # Benchmark CoW
    var cow_time = 0
    var cow_memory = 0

    start = Time.get_ticks_usec()
    var cow_array = CoWArray.new(test_array)
    for iter in range(ITERATIONS):
        # CoW clone (instant)
        var cow_copy = cow_array.clone()
        cow_memory += MODIFICATIONS * 12  # Only modified elements copied

        # Modify a few elements (triggers copy)
        for i in range(MODIFICATIONS):
            cow_copy.set_at(i, Vector3.ZERO)
    cow_time = Time.get_ticks_usec() - start

    print("=== Copy-on-Write Benchmark ===")
    print("Full Copy:")
    print("  Time: %.2f ms" % (full_copy_time / 1000.0))
    print("  Memory: %.2f MB" % (full_copy_memory / 1024.0 / 1024.0))
    print("CoW:")
    print("  Time: %.2f ms" % (cow_time / 1000.0))
    print("  Memory: %.2f MB" % (cow_memory / 1024.0 / 1024.0))
    print("Speedup: %.2fx" % (float(full_copy_time) / cow_time))
    print("Memory savings: %.2fx" % (float(full_copy_memory) / cow_memory))
    # Expected: 10-20x memory savings, 2-5x speed improvement
```

**Key Parameters:**
- Snapshot frequency: 60-120 per second for smooth rewind
- History depth: 5-30 seconds (300-3600 snapshots at 60fps)
- Modification ratio: CoW beneficial when <20% of data changes per snapshot
- Reference counting: Track when data can be freed

**Edge Cases:**
- Write-heavy workloads: CoW overhead exceeds benefit (many copies triggered)
- Large single modifications: Full array copy negates CoW advantage
- Reference cycles: Careful memory management needed (use weak refs)
- Thread safety: CoW with multiple writers requires locks (loses benefit)

**When NOT to Use:**
- Data modified frequently (>50% changes per snapshot)
- Single user (no need for multiple versions)
- Real-time requirements (copy-on-write latency unacceptable)
- Simple undo with linear history (array of full copies simpler)

**Examples from Shipped Games:**

1. **Braid (Jonathan Blow):** Time-rewind mechanic uses CoW-like state snapshots, enables rewinding 10+ seconds of gameplay with minimal memory (<50MB for full history)
2. **Factorio:** Replay system efficiently stores thousands of frames using differential encoding (CoW principle), enables watching 8-hour megabase construction
3. **Git Version Control:** CoW file trees power all version control, only stores diffs between commits, billions of files versioned efficiently
4. **Redux (React/Web):** Immutable state trees for time-travel debugging, inspired functional game architectures
5. **Clojure Persistent Data Structures:** O(log n) modifications with structural sharing, inspired CoW in game engines

**Platform Considerations:**
- **PC:** Large memory allows extensive history (10-60 seconds)
- **Mobile:** Limited memory requires shorter history (2-10 seconds)
- **Consoles:** Fixed memory budget, balance history vs gameplay features
- **VR:** Careful with allocations (causes hitches), CoW reduces GC pressure

**Godot-Specific Notes:**
- Godot lacks built-in CoW data structures (implement manually)
- Reference counting in Godot handles automatic cleanup
- Use PackedArrays for better memory efficiency than regular Arrays
- GDScript's duplicate() is deep copy (expensive), use reference sharing
- Consider GDExtension for optimized CoW with proper reference counting

**Synergies:**
- Essential for Time-Rewind Mechanics (store game state history)
- Enables Replay Systems (record gameplay efficiently)
- Pairs with Fixed Timestep (deterministic state snapshots)
- Supports Undo Systems in editors
- Reduces GC Pressure (fewer allocations than full copies)

**Measurement/Profiling:**
```gdscript
func profile_cow_memory():
    var initial_memory = Performance.get_monitor(Performance.MEMORY_STATIC)

    # Create CoW snapshots
    var base_state = GameState.new()
    var snapshots = []

    for i in range(100):
        var snapshot = base_state.clone()
        # Modify 10% of data
        for j in range(10):
            snapshot.entities.set_at(j, "modified")
        snapshots.append(snapshot)

    var final_memory = Performance.get_monitor(Performance.MEMORY_STATIC)
    var memory_used = final_memory - initial_memory

    print("Memory for 100 CoW snapshots: %.2f MB" % (memory_used / 1024.0 / 1024.0))
    # Expected: 1-5 MB vs 100-500 MB for full copies

# Profile snapshot time
func profile_snapshot_speed():
    var state = GameState.new()
    const ITERATIONS = 10000

    var start = Time.get_ticks_usec()
    for i in range(ITERATIONS):
        var snapshot = state.clone()  # CoW: instant
    var elapsed = Time.get_ticks_usec() - start

    print("Time per CoW snapshot: %.2f ns" % (float(elapsed) * 1000.0 / ITERATIONS))
    # Expected: 50-500 ns (reference copy + ref count increment)
```

**Target Metrics:**
- Memory overhead: 5-20% of full copy (10-20x savings)
- Snapshot time: <1μs per snapshot (vs >1ms for full copy)
- History capacity: 10-60 seconds at 60fps (600-3600 snapshots)
- Modification latency: <100μs for small changes (copy-on-write trigger)
