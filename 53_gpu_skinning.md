### 53. GPU Skinning (Skeletal Animation)

**Category:** Performance - 3D Rendering / Animation

**Problem It Solves:** CPU-bound skeletal animation bottleneck. Skinning 100 characters with 50 bones and 5000 vertices each on CPU requires 25M vertex transformations per frame at 60fps = 1.5B transforms/sec. At 100 cycles/transform = 150B cycles = 60ms on a 2.5GHz CPU, exceeding 16.67ms budget by 3.6x.

**Technical Explanation:**
GPU skinning offloads vertex transformation from CPU to GPU using vertex shaders. Each vertex stores bone indices and weights (typically 4 bones per vertex). Bone matrices are uploaded to GPU as uniform buffers. Vertex shader performs matrix palette skinning: final_position = Σ(bone_matrix[i] * weight[i] * vertex_position). GPU's parallel architecture processes thousands of vertices simultaneously, reducing 60ms CPU work to <1ms GPU work. Modern GPUs have dedicated skinning units for further optimization.

**Algorithmic Complexity:**
- CPU Skinning: O(vertices * bones_per_vertex) sequential
- GPU Skinning: O(vertices * bones_per_vertex) parallel
- Effective speedup: 50-100x due to GPU parallelism
- Memory bandwidth: 1-2GB/s for bone matrices and vertex data

**Implementation Pattern:**
```gdscript
# Godot implementation
extends Skeleton3D

var bone_matrices: Array = []
var skinning_shader: Shader

func _ready():
    # Godot handles GPU skinning automatically for MeshInstance3D with Skeleton
    # Custom shader for advanced skinning
    skinning_shader = preload("res://shaders/gpu_skinning.gdshader")
    _setup_bone_buffers()

func _setup_bone_buffers():
    bone_matrices.resize(get_bone_count())
    for i in range(get_bone_count()):
        bone_matrices[i] = get_bone_global_pose(i)

func _process(delta):
    # Update bone matrices (happens automatically in Godot)
    for i in range(get_bone_count()):
        bone_matrices[i] = get_bone_global_pose(i)

    # Matrices sent to GPU via RenderingServer
    # Skinning happens in vertex shader

# Custom vertex shader for GPU skinning
"""
shader_type spatial;

uniform mat4 bone_matrices[64]; // Max 64 bones

void vertex() {
    // GODOT_SKIN_MATRIX handles this automatically
    // Manual implementation:
    mat4 skin_matrix =
        bone_matrices[int(BONE_INDICES.x)] * BONE_WEIGHTS.x +
        bone_matrices[int(BONE_INDICES.y)] * BONE_WEIGHTS.y +
        bone_matrices[int(BONE_INDICES.z)] * BONE_WEIGHTS.z +
        bone_matrices[int(BONE_INDICES.w)] * BONE_WEIGHTS.w;

    VERTEX = (skin_matrix * vec4(VERTEX, 1.0)).xyz;
    NORMAL = normalize((skin_matrix * vec4(NORMAL, 0.0)).xyz);
}
"""

# Optimization: Use compute shader for skinning (Godot 4.0+)
class_name ComputeSkinning

var compute_shader: RDShaderFile
var pipeline: RID

func _setup_compute_skinning():
    var rd = RenderingServer.get_rendering_device()
    compute_shader = load("res://shaders/compute_skinning.glsl")

    # Dispatch compute shader for skinning
    # Processes vertices in parallel batches
    var vertex_count = 5000
    var workgroup_size = 64
    var dispatch_count = ceili(vertex_count / float(workgroup_size))
```

**Key Parameters:**
- Max bones per skeleton: 64-256 (hardware dependent)
- Bones per vertex: 4 (sweet spot for quality/performance)
- Vertex batch size: 1000-10000 vertices per draw call
- Bone update frequency: Every frame for animated, once for static poses
- Uniform buffer size: 64 bones × 16 floats × 4 bytes = 4KB per skeleton

**Edge Cases:**
- Bone limit exceeded: Split mesh into multiple skinned meshes
- Uniform buffer overflow: Use texture buffer for >256 bones
- Vertex weight normalization: Ensure weights sum to 1.0
- Root motion: Handle separately on CPU for gameplay logic
- Blend shapes + skinning: Apply blend shapes first, then skinning

**When NOT to Use:**
- Very simple models (<500 vertices, <10 bones) - CPU overhead may be acceptable
- Static meshes with no animation
- Platforms without programmable shaders (legacy hardware)
- When CPU is idle and GPU is bottlenecked
- Mobile devices with poor GPU but strong CPU (rare)

**Examples from Shipped Games:**

1. **Doom 2016:** GPU skinning for all demons and player weapons, 100+ animated characters on screen, 60fps locked. Reduced character animation CPU time from 45ms to <2ms.

2. **Horizon Zero Dawn:** Advanced GPU skinning for machines, up to 200 bones per machine, 30-50 machines visible simultaneously. Used compute shaders for skinning, achieving 30fps on PS4 with minimal CPU overhead.

3. **Spider-Man (PS4):** GPU skinning for crowds, 100+ pedestrians with full skeletal animation. Enabled dense crowds without CPU bottleneck, ~0.5ms CPU time for all crowd animation.

4. **The Witcher 3:** GPU skinning for Geralt, monsters, NPCs. Supported 50-80 bones per character, multiple characters in combat. CPU animation time <3ms for all characters.

5. **Assassin's Creed Unity:** GPU skinning enabled massive crowds (10,000+ NPCs), each with skeletal animation. Critical for achieving series' densest crowds without performance collapse.

**Platform Considerations:**
- **PC/Console:** Full GPU skinning support, 256+ bones, compute shader acceleration
- **Mobile:** Limited bone count (32-64), reduced precision, power consumption concerns
- **VR:** Critical for performance, 90fps requirement, use aggressive LOD with bone reduction
- **WebGL:** Limited uniform buffers, may need texture-based bone matrices
- **Memory bandwidth:** 1-2GB/s for bone data, ensure efficient upload
- **GPU generations:** Older GPUs (pre-2010) may struggle, newer GPUs have dedicated units

**Godot-Specific Notes:**
- Godot 4.x uses GPU skinning by default for all Skeleton3D nodes
- Automatic optimization: Godot batches skinned meshes when possible
- `Skeleton3D.set_bone_pose()` updates are batched and uploaded once per frame
- For extreme optimization, use custom compute shaders via RenderingDevice
- Godot's skinning supports up to 256 bones per skeleton (driver dependent)
- Use `MeshInstance3D.set_software_skinning_transform_normals(false)` to force GPU skinning
- Monitor `RenderingServer.RENDER_INFO_SURFACE_CHANGES` for batch breaking

**Synergies:**
- **Animation LOD (Technique 58):** Reduce bone count at distance, fewer GPU calculations
- **Instanced Rendering:** Combine with GPU instancing for crowd animation
- **Occlusion Culling:** Skip skinning for occluded characters entirely
- **Frustum Culling:** Don't upload bone matrices for off-screen characters
- **Animation Compression (Technique 54):** Reduces CPU→GPU data transfer

**Measurement/Profiling:**
- **Godot Profiler:** Monitor "Skeleton3D" and "VisualServer" sections
- **GPU Frame Analyzer:** RenderDoc, NSight Graphics to measure vertex shader time
- **Target metrics:** <1ms total skinning time for 50 characters
- **Bandwidth:** Monitor GPU memory bandwidth usage (should be <10% for skinning)
- **CPU time:** Bone matrix updates should be <2ms for 100 characters
- **Compare:** Profile with/without skinning to isolate impact
- **Bottleneck detection:** If GPU-bound, reduce bone count or vertex count

---
