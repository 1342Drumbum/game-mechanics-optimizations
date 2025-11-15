### 50. 2D Physics Shape Simplification

**Category:** Performance - Physics Optimization

**Problem It Solves:** Complex collision shapes cause expensive physics calculations. A sprite with 100-vertex collision polygon requires 100² = 10,000 vertex pair checks for SAT collision detection. At 50ns per check = 0.5ms per collision test. With 100 objects checking 20 neighbors = 1000ms total, destroying 60fps target. Simplified shapes (4-8 vertices) reduce to 0.02ms per test, a 96% reduction.

**Technical Explanation:**
Physics engines use Separating Axis Theorem (SAT) for convex polygon collision, requiring O(n×m) checks for polygons with n and m vertices. Simplifying collision shapes to fewer vertices dramatically reduces computation. Techniques: use rectangles/circles instead of complex polygons, decompose concave shapes into multiple convex hulls, apply Ramer-Douglas-Peucker algorithm to reduce vertex count while preserving shape, and use approximate bounding shapes for broadphase before precise collision.

Modern approach: separate visual sprite from collision shape entirely. Visual can be 512×512 detailed sprite, collision shape 4-8 vertex approximate polygon. For static geometry, prebake simplified collision at export time. For projectiles, use circles (fastest primitive). For characters, capsules (2 circles + rectangle). Only use precise polygons when necessary for gameplay.

**Algorithmic Complexity:**
- Complex Polygon (100 vertices): O(n²) = 10,000 checks per collision pair
- Simplified Polygon (6 vertices): O(n²) = 36 checks per collision pair (278× faster)
- Circle Collision: O(1) = single distance check (10,000× faster)
- Broadphase + Narrowphase: O(log n) + O(simplified_shape) = total speedup 100-1000×

**Implementation Pattern:**
```gdscript
# Physics shape simplification utility
class_name PhysicsShapeSimplifier
extends Node

# Simplify polygon using Ramer-Douglas-Peucker algorithm
static func simplify_polygon(points: PackedVector2Array, epsilon: float = 2.0) -> PackedVector2Array:
    if points.size() < 3:
        return points

    # Find point with maximum distance from line
    var max_dist = 0.0
    var max_index = 0

    for i in range(1, points.size() - 1):
        var dist = point_to_line_distance(
            points[i],
            points[0],
            points[points.size() - 1]
        )
        if dist > max_dist:
            max_dist = dist
            max_index = i

    # If max distance exceeds epsilon, split and recurse
    if max_dist > epsilon:
        var left = points.slice(0, max_index + 1)
        var right = points.slice(max_index, points.size())

        var simplified_left = simplify_polygon(left, epsilon)
        var simplified_right = simplify_polygon(right, epsilon)

        # Merge (remove duplicate middle point)
        var result = simplified_left.slice(0, simplified_left.size() - 1)
        result.append_array(simplified_right)
        return result
    else:
        # Simplify to line segment
        var result = PackedVector2Array()
        result.append(points[0])
        result.append(points[points.size() - 1])
        return result

static func point_to_line_distance(point: Vector2, line_start: Vector2, line_end: Vector2) -> float:
    var line_vec = line_end - line_start
    var line_length_sq = line_vec.length_squared()

    if line_length_sq < 0.0001:
        return point.distance_to(line_start)

    var t = clamp((point - line_start).dot(line_vec) / line_length_sq, 0.0, 1.0)
    var projection = line_start + t * line_vec
    return point.distance_to(projection)

# Create approximate collision shape from sprite
static func create_simplified_collision(sprite: Sprite2D, max_vertices: int = 8) -> CollisionPolygon2D:
    var texture = sprite.texture
    if not texture:
        return null

    # Get sprite bounds
    var image = texture.get_image()
    var size = image.get_size()

    # Trace outline of sprite (alpha channel)
    var outline = trace_sprite_outline(image)

    # Simplify to max_vertices
    var epsilon = 1.0
    var simplified = outline

    while simplified.size() > max_vertices and epsilon < 50.0:
        simplified = simplify_polygon(outline, epsilon)
        epsilon += 0.5

    # Create collision polygon
    var collision = CollisionPolygon2D.new()
    collision.polygon = simplified
    return collision

static func trace_sprite_outline(image: Image) -> PackedVector2Array:
    # Simple outline tracer (finds first non-transparent pixel per column)
    var outline = PackedVector2Array()
    var width = image.get_width()
    var height = image.get_height()

    # Top edge
    for x in width:
        for y in height:
            if image.get_pixel(x, y).a > 0.1:
                outline.append(Vector2(x, y) - Vector2(width, height) / 2)
                break

    # Right edge
    for y in height:
        for x in range(width - 1, -1, -1):
            if image.get_pixel(x, y).a > 0.1:
                outline.append(Vector2(x, y) - Vector2(width, height) / 2)
                break

    # Bottom edge
    for x in range(width - 1, -1, -1):
        for y in range(height - 1, -1, -1):
            if image.get_pixel(x, y).a > 0.1:
                outline.append(Vector2(x, y) - Vector2(width, height) / 2)
                break

    # Left edge
    for y in range(height - 1, -1, -1):
        for x in width:
            if image.get_pixel(x, y).a > 0.1:
                outline.append(Vector2(x, y) - Vector2(width, height) / 2)
                break

    return outline

# Optimized character collision (capsule approximation)
class_name CapsuleCharacter
extends CharacterBody2D

@export var capsule_radius: float = 16.0
@export var capsule_height: float = 48.0

func _ready():
    # Use capsule shape (most efficient for characters)
    var capsule = CapsuleShape2D.new()
    capsule.radius = capsule_radius
    capsule.height = capsule_height

    var collision = CollisionShape2D.new()
    collision.shape = capsule
    add_child(collision)

# Projectile with circle collision (fastest)
class_name FastProjectile
extends Area2D

@export var radius: float = 4.0

func _ready():
    # Circle is fastest collision primitive
    var circle = CircleShape2D.new()
    circle.radius = radius

    var collision = CollisionShape2D.new()
    collision.shape = circle
    add_child(collision)

# Convex decomposition for concave shapes
class_name ConvexDecomposer
extends Node

static func decompose_concave_to_convex(concave_polygon: PackedVector2Array) -> Array[PackedVector2Array]:
    # Split concave polygon into multiple convex polygons
    # This is a simplified implementation
    var convex_parts: Array[PackedVector2Array] = []

    # Check if already convex
    if is_convex(concave_polygon):
        convex_parts.append(concave_polygon)
        return convex_parts

    # Find reflex vertex (interior angle > 180°)
    var reflex_idx = find_reflex_vertex(concave_polygon)
    if reflex_idx == -1:
        convex_parts.append(concave_polygon)
        return convex_parts

    # Split at reflex vertex
    var split_idx = find_split_vertex(concave_polygon, reflex_idx)

    # Create two sub-polygons
    var poly1 = concave_polygon.slice(0, split_idx + 1)
    poly1.append(concave_polygon[reflex_idx])

    var poly2 = PackedVector2Array()
    poly2.append(concave_polygon[reflex_idx])
    poly2.append_array(concave_polygon.slice(split_idx, concave_polygon.size()))

    # Recursively decompose
    convex_parts.append_array(decompose_concave_to_convex(poly1))
    convex_parts.append_array(decompose_concave_to_convex(poly2))

    return convex_parts

static func is_convex(polygon: PackedVector2Array) -> bool:
    var n = polygon.size()
    if n < 3:
        return false

    var sign = 0
    for i in n:
        var p1 = polygon[i]
        var p2 = polygon[(i + 1) % n]
        var p3 = polygon[(i + 2) % n]

        var cross = (p2.x - p1.x) * (p3.y - p2.y) - (p2.y - p1.y) * (p3.x - p2.x)

        if cross != 0:
            if sign == 0:
                sign = 1 if cross > 0 else -1
            elif (cross > 0 and sign < 0) or (cross < 0 and sign > 0):
                return false  # Not convex

    return true

static func find_reflex_vertex(polygon: PackedVector2Array) -> int:
    var n = polygon.size()
    for i in n:
        var p1 = polygon[(i - 1 + n) % n]
        var p2 = polygon[i]
        var p3 = polygon[(i + 1) % n]

        var cross = (p2.x - p1.x) * (p3.y - p2.y) - (p2.y - p1.y) * (p3.x - p2.x)

        if cross < 0:  # Reflex angle
            return i

    return -1

static func find_split_vertex(polygon: PackedVector2Array, reflex_idx: int) -> int:
    # Find best vertex to split at (simplified heuristic)
    var best_idx = (reflex_idx + 2) % polygon.size()
    return best_idx

# Dynamic LOD for physics shapes
class_name PhysicsShapeLOD
extends Node

enum LODLevel { HIGH, MEDIUM, LOW }

var current_lod: LODLevel = LODLevel.HIGH

func update_physics_lod(body: PhysicsBody2D, camera_pos: Vector2):
    var distance = body.global_position.distance_to(camera_pos)

    var new_lod: LODLevel
    if distance < 300:
        new_lod = LODLevel.HIGH
    elif distance < 800:
        new_lod = LODLevel.MEDIUM
    else:
        new_lod = LODLevel.LOW

    if new_lod != current_lod:
        current_lod = new_lod
        apply_lod_to_body(body, new_lod)

func apply_lod_to_body(body: PhysicsBody2D, lod: LODLevel):
    # Get or create collision shapes for different LODs
    match lod:
        LODLevel.HIGH:
            # Detailed collision (8-12 vertices)
            set_collision_shape(body, "collision_high")
        LODLevel.MEDIUM:
            # Simplified collision (4-6 vertices)
            set_collision_shape(body, "collision_medium")
        LODLevel.LOW:
            # Bounding box or circle
            set_collision_shape(body, "collision_low")

func set_collision_shape(body: PhysicsBody2D, shape_name: String):
    # Disable all collision shapes
    for child in body.get_children():
        if child is CollisionShape2D:
            child.disabled = true

    # Enable target shape
    var target = body.get_node_or_null(shape_name)
    if target and target is CollisionShape2D:
        target.disabled = false
```

**Key Parameters:**
- **Simplification Epsilon:** 1.0-5.0 pixels (smaller = more detail, larger = simpler)
- **Max Vertices:** 4-6 (simple), 8-12 (detailed), 20+ (very detailed, slow)
- **Shape Type:** Circle (fastest), Capsule (characters), Rectangle (tiles), Polygon (complex)
- **LOD Distances:** High <300px, Medium 300-800px, Low >800px
- **Convex Hull Count:** 1-3 parts for most shapes, 4-8 for very complex
- **Collision Layers:** Separate simple shapes on different layers

**Edge Cases:**
- **Very Thin Objects:** Simplified shape may be too small, set minimum size
- **Holes in Sprites:** Simplification may close holes, handle separately
- **Rotating Objects:** Simple shapes may not fit well when rotated
- **Scale Changes:** Collision shapes must scale with sprite
- **One-Way Platforms:** Simplification can't break platform detection
- **Trigger Areas:** Separate trigger shapes from collision shapes

**When NOT to Use:**
- Pixel-perfect platformers requiring exact collision
- Puzzle games where precise shapes matter for gameplay
- Small objects already using simple shapes (4-6 vertices)
- Static geometry that can be preprocessed offline
- Games where physics is not performance bottleneck

**Examples from Shipped Games:**

1. **Celeste:** Character uses capsule collision (2 vertices equivalent), despite detailed sprite. Spikes use triangles (3 vertices). Platforms use rectangles. Maximum 8 vertices for any complex collision. Enables 200+ physics objects at 60fps. Critical for precise platforming feel.

2. **Dead Cells:** All enemies use 4-8 vertex simplified collision regardless of sprite complexity. Weapons use 2-4 vertex hitboxes. Background decorations have no collision (visual only). Procedural levels prebake simplified collision during generation. 100+ active physics bodies maintained.

3. **Hollow Knight:** Knight character uses 6-vertex hexagon collision. Enemies range from circles (simple) to 8-vertex polygons (complex). Environmental hazards simplified to rectangles/triangles. Boss arenas use multiple convex shapes instead of one complex shape. Maintains 60fps with 50+ physics objects.

4. **Terraria:** All tiles use rectangle collision (4 vertices), never complex. Characters use capsules. Projectiles use circles. NPCs use rectangles. Complex shapes decomposed into multiple rectangles. Enables 1000+ active physics objects for multiplayer chaos.

5. **Enter the Gungeon:** Player and enemies use 6-8 vertex collision polygons. 1000+ bullets on screen all use circles (fastest). Room geometry simplified to 4-6 rectangles per room. Furniture uses bounding boxes, not detailed shapes. Essential for bullet-hell performance.

**Platform Considerations:**
- **Mobile (iOS/Android):** Essential - limit to 4-6 vertices max, prefer circles/rectangles. Target <50 active physics bodies. Simplify aggressively.
- **Desktop (Windows/Mac/Linux):** Can afford 8-12 vertex polygons. 200+ physics bodies okay. Still simplify for best performance.
- **Console (Switch/PS/Xbox):** Switch: 6-8 vertices max, <100 bodies. PS5/Xbox: 12+ vertices fine, 300+ bodies.
- **WebGL:** JavaScript physics slower - 4-6 vertices max, aggressive simplification critical.
- **Physics Engine:** Box2D (Godot's engine) handles convex polygons well, but simpler = faster.

**Godot-Specific Notes:**
- Godot uses Box2D-derived physics engine, optimized for convex shapes
- `CollisionShape2D` supports: CircleShape2D (fastest), CapsuleShape2D, RectangleShape2D, ConvexPolygonShape2D, ConcavePolygonShape2D (slowest)
- Never use ConcavePolygonShape2D for dynamic objects (decompose to convex)
- Monitor physics time: Profiler → Physics → 2D Physics Time
- Godot automatically decomposes concave to convex for StaticBody2D
- Use `collision_layer` and `collision_mask` to limit which objects collide
- Godot 4.x physics more optimized than 3.x but simplification still critical

**Synergies:**
- **Spatial Partitioning:** Simplified shapes in quadtree nodes
- **Object Pooling:** Pooled objects share simplified collision shapes
- **Dirty Rectangles:** Physics only in dirty regions
- **Canvas Layer Management:** Separate physics layers for different object types
- **Broadphase Culling:** Simplified bounding boxes for broadphase

**Measurement/Profiling:**
- **Physics Time:** Monitor `Performance.TIME_PHYSICS_PROCESS` - should be <3ms
- **Collision Checks:** Track collision pair count - simplified shapes reduce by 50-90%
- **Vertex Count:** Sum all collision polygon vertices - target <1000 total
- **Active Bodies:** Count PhysicsBody2D nodes - target <100 mobile, <200 desktop
- **Profiling Pattern:**
```gdscript
var total_vertices = 0
var total_bodies = 0
var shape_types = { "circle": 0, "rectangle": 0, "polygon": 0, "capsule": 0 }

func profile_physics_shapes():
    for body in get_tree().get_nodes_in_group("physics_bodies"):
        if body is PhysicsBody2D:
            total_bodies += 1

            for child in body.get_children():
                if child is CollisionShape2D:
                    var shape = child.shape

                    if shape is CircleShape2D:
                        shape_types["circle"] += 1
                    elif shape is RectangleShape2D:
                        shape_types["rectangle"] += 1
                    elif shape is CapsuleShape2D:
                        shape_types["capsule"] += 1
                    elif shape is ConvexPolygonShape2D:
                        shape_types["polygon"] += 1
                        total_vertices += shape.points.size()

    print("Total bodies: ", total_bodies)
    print("Total vertices: ", total_vertices)
    print("Avg vertices/body: ", total_vertices / max(1, total_bodies))
    print("Shape distribution: ", shape_types)

var physics_time_samples = []

func measure_physics_performance():
    var physics_time = Performance.get_monitor(Performance.TIME_PHYSICS_PROCESS) * 1000
    physics_time_samples.append(physics_time)

    if physics_time_samples.size() > 60:  # 1 second of samples
        var avg = 0.0
        for sample in physics_time_samples:
            avg += sample
        avg /= physics_time_samples.size()
        print("Avg physics time: ", avg, " ms")
        physics_time_samples.clear()
```

**Advanced Optimization Patterns:**
```gdscript
# Pattern 1: Automatic shape simplification on spawn
func spawn_enemy_with_simple_collision(scene: PackedScene) -> Node2D:
    var enemy = scene.instantiate()

    # Find complex collision shapes and simplify
    for child in enemy.get_children():
        if child is CollisionPolygon2D:
            if child.polygon.size() > 8:
                # Too complex, simplify
                var simplified = PhysicsShapeSimplifier.simplify_polygon(child.polygon, 3.0)
                child.polygon = simplified

    return enemy

# Pattern 2: Shared collision shapes (instance optimization)
var shared_shapes: Dictionary = {}  # shape_hash -> Shape2D

func get_or_create_shared_shape(points: PackedVector2Array) -> ConvexPolygonShape2D:
    var hash_val = points.hash()

    if not shared_shapes.has(hash_val):
        var shape = ConvexPolygonShape2D.new()
        shape.points = points
        shared_shapes[hash_val] = shape

    return shared_shapes[hash_val]

# Pattern 3: Collision shape caching for animated sprites
var frame_collision_cache: Dictionary = {}  # frame_idx -> simplified_polygon

func get_collision_for_animation_frame(animation: String, frame: int) -> PackedVector2Array:
    var cache_key = "%s_%d" % [animation, frame]

    if not frame_collision_cache.has(cache_key):
        # Generate and cache simplified collision for this frame
        var sprite_frame = get_sprite_frame_texture(animation, frame)
        var collision = generate_simplified_collision(sprite_frame)
        frame_collision_cache[cache_key] = collision

    return frame_collision_cache[cache_key]
```
