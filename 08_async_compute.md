# Async Compute (GPU Work Overlap)

## Category
Memory & Data Management

## Problem It Solves
Traditional GPU rendering uses only graphics units, leaving compute units idle for **40-60% of frame time**. Modern GPUs have separate graphics and compute pipelines that can run simultaneously, but sequential rendering wastes this parallelism:
- **Graphics-only rendering**: GPU utilization **40-60%** (compute units idle)
- **Async compute enabled**: GPU utilization **85-95%** (both pipelines active)
- **Frame time improvement**: **15-30% faster** rendering from overlapped work

**Performance Impact**:
- **Particle simulation**: Sequential = 2.5ms graphics + 1.5ms compute = **4ms total**
- **Async compute**: Overlapped = **2.5ms total** (compute runs during graphics)
- **Lighting calculations**: 3ms saved by computing during geometry pass
- **Post-processing**: 1-2ms saved by computing during shadow rendering

NVIDIA's GDC presentations show async compute can **reduce frame time by 21.8ms to 19.2ms** (13% speedup) in optimal scenarios by eliminating GPU idle time.

## Technical Explanation

### GPU Architecture
Modern GPUs have multiple execution units:
- **Graphics pipeline**: Vertex shaders, rasterization, pixel shaders
- **Compute pipeline**: General-purpose compute units
- **Copy engines**: DMA transfers (texture uploads, etc.)

These can run **simultaneously** if work is submitted to different queues.

**Sequential Execution** (traditional):
```
Timeline:
Graphics: [Geometry][Lighting][Post-process]
Compute:  [Idle....][Particles................][Idle..........]
          └ Wasted GPU time
```

**Async Compute** (overlapped):
```
Timeline:
Graphics: [Geometry][Lighting][Post-process]
Compute:  [Particles][SSR calc][AO denoise]
          └ Work overlaps, no idle time
```

### Vulkan/DirectX 12 API Model
```cpp
// Graphics queue (rendering)
graphicsQueue.submit(geometryCommands);

// Compute queue (async, runs in parallel)
computeQueue.submit(particleSimulation);  // Overlaps with geometry

// Wait for both to finish
fence.wait();
```

In Godot, this is abstracted but the principle remains: offload expensive calculations to compute shaders that run during otherwise idle GPU time.

### Ideal Async Compute Workloads
1. **Particle simulation**: Physics-based particles while rendering geometry
2. **Post-processing**: Bloom/blur while rendering shadows
3. **Light culling**: Determine visible lights during opaque rendering
4. **Procedural generation**: Terrain/textures while rendering existing world
5. **Physics simulation**: Soft-body/cloth sim during shadow passes

**Key Requirement**: Workloads must **not conflict** (different memory regions, no dependencies).

### Performance Analysis
AMD research found:
- **Graphics-heavy frames**: Async compute provides **5-15% speedup**
- **Balanced workload**: **15-25% speedup** from perfect overlap
- **Compute-heavy frames**: **25-40% speedup** if graphics pipeline has gaps

NVIDIA Nsight Graphics shows "GPU Trace" feature identifying low throughput metrics and unused warp slots—perfect candidates for async compute injection.

## Algorithmic Complexity

| Metric | Sequential | Async Compute |
|--------|-----------|---------------|
| GPU utilization | 40-60% | 85-95% |
| Frame time (graphics+compute) | T_graphics + T_compute | max(T_graphics, T_compute) |
| Memory bandwidth | Peak in graphics pass | Smoothed across frame |
| Command buffer overhead | Low (single queue) | Medium (multiple queues + sync) |

**Example Calculation**:
- Graphics work: 10ms
- Compute work: 4ms
- Sequential total: 14ms (71 FPS)
- Async (if full overlap): 10ms (100 FPS)
- Async (50% overlap): 12ms (83 FPS)

**Speedup Formula**:
```
Speedup = (T_graphics + T_compute) / max(T_graphics, T_compute_async)

Where T_compute_async ≥ T_compute due to sync overhead
```

**Best Case**: Graphics 10ms, Compute 8ms → Sequential 18ms, Async 10ms = **1.8x speedup**

**Worst Case**: Work conflicts, no overlap → Async slightly slower due to queue overhead

## Implementation Pattern

### Compute Shader for Particle System (Godot)
```gdscript
# Godot 4.x Compute Shader approach
class_name AsyncParticleSystem
extends Node3D

var rd: RenderingDevice
var shader: RID
var pipeline: RID
var particle_buffer: RID

const PARTICLE_COUNT = 100000
const WORK_GROUP_SIZE = 256

func _ready() -> void:
    rd = RenderingServer.get_rendering_device()
    if not rd:
        push_error("Compute shaders require Forward+ renderer")
        return

    _setup_compute_shader()
    _create_particle_buffer()

func _setup_compute_shader() -> void:
    # Load compute shader
    var shader_file = load("res://shaders/particles_compute.glsl")
    var shader_spirv: RDShaderSPIRV = shader_file.get_spirv()

    shader = rd.shader_create_from_spirv(shader_spirv)
    pipeline = rd.compute_pipeline_create(shader)

func _create_particle_buffer() -> void:
    # Create buffer for particle data
    var particle_data = PackedFloat32Array()
    particle_data.resize(PARTICLE_COUNT * 8)  # pos(3) + vel(3) + life(1) + padding(1)

    for i in range(PARTICLE_COUNT):
        var base = i * 8
        # Initialize particles
        particle_data[base + 0] = randf() * 100 - 50  # x
        particle_data[base + 1] = randf() * 100 - 50  # y
        particle_data[base + 2] = randf() * 100 - 50  # z
        particle_data[base + 3] = randf() * 2 - 1     # vx
        particle_data[base + 4] = randf() * 2 - 1     # vy
        particle_data[base + 5] = randf() * 2 - 1     # vz
        particle_data[base + 6] = randf() * 5.0       # life
        particle_data[base + 7] = 0.0                 # padding

    var bytes = particle_data.to_byte_array()

    particle_buffer = rd.storage_buffer_create(bytes.size(), bytes)

func _physics_process(delta: float) -> void:
    if not rd:
        return

    # Run particle simulation on GPU (async with rendering)
    _dispatch_compute_shader(delta)

func _dispatch_compute_shader(delta: float) -> void:
    # Create uniform set
    var uniform = RDUniform.new()
    uniform.uniform_type = RenderingDevice.UNIFORM_TYPE_STORAGE_BUFFER
    uniform.binding = 0
    uniform.add_id(particle_buffer)

    var uniform_set = rd.uniform_set_create([uniform], shader, 0)

    # Push constants (delta time, etc.)
    var push_constant = PackedFloat32Array([delta])

    # Create compute list
    var compute_list = rd.compute_list_begin()
    rd.compute_list_bind_compute_pipeline(compute_list, pipeline)
    rd.compute_list_bind_uniform_set(compute_list, uniform_set, 0)
    rd.compute_list_set_push_constant(compute_list, push_constant.to_byte_array(), push_constant.size() * 4)

    # Dispatch work groups
    var work_groups = (PARTICLE_COUNT + WORK_GROUP_SIZE - 1) / WORK_GROUP_SIZE
    rd.compute_list_dispatch(compute_list, work_groups, 1, 1)

    rd.compute_list_end()

    # Submit (runs async on GPU)
    rd.submit()

    # Optional: Sync if needed this frame
    # rd.sync()
```

### Compute Shader (GLSL)
```glsl
#[compute]
#version 450

layout(local_size_x = 256, local_size_y = 1, local_size_z = 1) in;

struct Particle {
    vec3 position;
    float padding1;
    vec3 velocity;
    float life;
};

layout(set = 0, binding = 0, std430) buffer ParticleBuffer {
    Particle particles[];
};

layout(push_constant, std430) uniform Params {
    float delta_time;
};

void main() {
    uint index = gl_GlobalInvocationID.x;

    if (index >= particles.length()) {
        return;
    }

    Particle p = particles[index];

    // Simple gravity + motion
    p.velocity.y -= 9.8 * delta_time;
    p.position += p.velocity * delta_time;
    p.life -= delta_time;

    // Reset dead particles
    if (p.life <= 0.0) {
        p.position = vec3(0.0, 100.0, 0.0);
        p.velocity = vec3(
            (float(index) / float(particles.length()) - 0.5) * 10.0,
            0.0,
            (float(index * 7) / float(particles.length()) - 0.5) * 10.0
        );
        p.life = 5.0;
    }

    particles[index] = p;
}
```

### Async Post-Processing Pipeline
```gdscript
class_name AsyncPostProcessing
extends Node

var rd: RenderingDevice
var bloom_shader: RID
var ao_shader: RID

func _process(_delta: float) -> void:
    if not rd:
        return

    # While main rendering happens on graphics queue,
    # run post-processing prep on compute queue

    # Example: Downsample for bloom while rendering geometry
    _dispatch_bloom_downsample()

    # Compute ambient occlusion while rendering shadows
    _dispatch_ssao_compute()

func _dispatch_bloom_downsample() -> void:
    # Compute shader downsamples HDR buffer for bloom
    # Runs async while geometry renders
    var compute_list = rd.compute_list_begin()
    # ... dispatch bloom compute ...
    rd.compute_list_end()
    rd.submit()  # Async

func _dispatch_ssao_compute() -> void:
    # Compute SSAO from depth buffer
    # Overlaps with shadow map rendering
    var compute_list = rd.compute_list_begin()
    # ... dispatch SSAO compute ...
    rd.compute_list_end()
    rd.submit()  # Async
```

### Light Culling with Async Compute
```gdscript
class_name AsyncLightCulling
extends Node3D

var rd: RenderingDevice
var culling_shader: RID
var light_buffer: RID
var visible_lights_buffer: RID

const MAX_LIGHTS = 1024

func _process(_delta: float) -> void:
    # Cull lights on GPU while rendering opaque geometry
    _dispatch_light_culling()

func _dispatch_light_culling() -> void:
    # Compute shader determines which lights affect each tile
    # Runs async during opaque rendering pass

    var compute_list = rd.compute_list_begin()
    rd.compute_list_bind_compute_pipeline(compute_list, culling_shader)

    # Dispatch 16x16 tile grid
    var tiles_x = (get_viewport().size.x + 15) / 16
    var tiles_y = (get_viewport().size.y + 15) / 16
    rd.compute_list_dispatch(compute_list, tiles_x, tiles_y, 1)

    rd.compute_list_end()
    rd.submit()

    # Graphics pass can now use culled light list for shading
```

## Key Parameters
- **Work group size**: 64-256 threads per group (GPU-dependent)
  - AMD: 64 optimal (wavefront size)
  - NVIDIA: 32 optimal (warp size)
  - Intel: 8-16 optimal (SIMD width)
- **Queue priority**: Some GPUs support priority queues
  - Graphics: High priority
  - Async compute: Normal priority
- **Sync points**: Minimize barriers between queues
  - Insert fence/semaphore only when graphics needs compute results
- **Memory allocation**: Ensure no conflicts
  - Graphics writes texture A → Compute reads texture A = **BAD**
  - Graphics writes texture A → Compute writes texture B = **GOOD**

## Edge Cases

### 1. Resource Conflicts
```gdscript
# BAD: Both queues accessing same buffer
# Graphics: Rendering shadows to shadow_map
# Compute: Reading shadow_map for SSAO
# → Race condition!

# GOOD: Explicit synchronization
func _render_frame():
    render_shadows()        # Graphics queue
    rd.barrier()            # Wait for shadows to finish
    compute_ssao()          # Now safe to read shadow_map
```

### 2. GPU Starvation
```gdscript
# If compute work is too heavy, graphics queue stalls

# Monitor GPU utilization
func _process(_delta):
    var graphics_load = Performance.get_monitor(Performance.RENDER_PROCESS_TIME)
    var compute_load = _get_compute_time()

    if compute_load > graphics_load:
        push_warning("Compute queue blocking graphics - reduce compute work")
```

### 3. Mobile GPU Limitations
```gdscript
# Not all mobile GPUs support async compute
func _ready():
    if not rd or not rd.supports_async_compute():  # Hypothetical API
        push_warning("Async compute not supported - using CPU fallback")
        use_cpu_particles = true
```

### 4. Command Buffer Overhead
```gdscript
# Creating too many small compute dispatches wastes CPU time
# BAD: 100 tiny compute dispatches
for i in range(100):
    dispatch_tiny_compute(i)  # Overhead > benefit

# GOOD: One large dispatch
dispatch_batched_compute(100)  # Amortize overhead
```

## When NOT to Use

- **Integrated GPUs**: Often share resources, less benefit from async
  - Intel UHD: Minimal async benefit
  - AMD Vega (mobile): Some benefit but limited
- **Very short frames**: Sync overhead > benefit
  - If frame time <5ms, async compute overhead comparable to savings
- **Simple rendering**: Not enough graphics work to overlap
  - 2D games with minimal effects
- **Old APIs**: OpenGL doesn't support true async compute
  - Requires Vulkan, DirectX 12, or Metal

**Decision Matrix**:
```
Use Async Compute if:
  is_discrete_gpu AND frame_time > 8ms AND has_compute_work > 2ms
```

## Examples from Shipped Games

### 1. **Doom (2016) - id Software**
- Async compute for particle simulation and light culling
- **15-20% frame time reduction** on AMD GPUs (excellent async support)
- Particles simulated during geometry rendering
- Light culling during shadow map generation
- Measured: 60 FPS → 72 FPS on same hardware

### 2. **Assassin's Creed Unity**
- Async compute for SSAO and volumetric lighting
- **2-3ms saved** per frame on async-capable GPUs
- Compute shaders run during opaque geometry pass
- Critical for maintaining 30 FPS on consoles

### 3. **Rise of the Tomb Raider**
- Async compute for various post-processing effects
- **10-15% performance boost** on AMD GPUs
- Bloom, depth of field computed during shadow rendering
- DirectX 12 explicit multi-queue implementation

### 4. **Forza Horizon 4**
- Async compute for physics simulation and particles
- Weather simulation overlaps with vehicle rendering
- **Consistent 60 FPS** at 4K on Xbox One X
- Heavy use of async to maximize GPU utilization

### 5. **Helldivers 2 (2024)**
- Async compute option in graphics settings
- Some players report performance issues (implementation quality matters)
- Shows async compute is not automatic win—needs careful tuning
- Best on AMD RDNA2/3 GPUs, mixed results on older hardware

## Platform Considerations

### Desktop (PC)
- **AMD RDNA/GCN**: Excellent async compute (hardware queues)
  - Speedup: 15-30% typical
- **NVIDIA Pascal+**: Good async compute (preemption)
  - Speedup: 10-20% typical
- **Intel Arc**: Moderate async compute support
  - Speedup: 5-15% typical

### Mobile (iOS/Android)
- **Apple GPU**: Limited async compute (unified architecture)
  - Speedup: 5-10% typical
- **Qualcomm Adreno**: Decent async support
  - Speedup: 10-15% typical
- **ARM Mali**: Variable support by generation
  - Speedup: 0-10% typical

### Consoles
- **PS5 (RDNA2)**: Excellent async compute
  - AMD hardware queues, up to 30% speedup
- **Xbox Series X (RDNA2)**: Excellent async compute
  - Similar to PS5, 25-30% potential speedup
- **Nintendo Switch (Maxwell)**: Limited async compute
  - 0-5% speedup, not worth complexity

### Web (WebGPU)
- **WebGPU**: Supports compute shaders
  - Async compute depends on browser + GPU
  - Generally 5-15% benefit on supported hardware

## Godot-Specific Notes

### Godot 4.x Compute Shader Support
Godot 4 added compute shader support via RenderingDevice API:

```gdscript
# Check if compute shaders available
func _ready():
    var rd = RenderingServer.get_rendering_device()
    if rd:
        print("Compute shaders supported!")
    else:
        print("Compute shaders require Forward+ renderer")
```

### Renderer Requirements
- **Forward+**: Full compute shader support
- **Mobile**: Limited compute support
- **Compatibility (OpenGL)**: No compute shaders

### Performance Monitoring
```gdscript
func _ready():
    # Monitor GPU time
    Performance.add_custom_monitor("gpu/frame_time_ms",
        func(): return Performance.get_monitor(Performance.RENDER_PROCESS_TIME))
    Performance.add_custom_monitor("gpu/draw_calls",
        func(): return Performance.get_monitor(Performance.RENDER_DRAW_CALLS_IN_FRAME))
```

### GDExtension for Advanced Control
For fine-grained async compute control, use GDExtension (C++):

```cpp
// GDExtension can directly use Vulkan queues
VkQueue graphicsQueue;
VkQueue computeQueue;

// Submit graphics work
vkQueueSubmit(graphicsQueue, ...);

// Submit async compute work
vkQueueSubmit(computeQueue, ...);

// Synchronize with semaphores
```

## Synergies

### Job System Multithreading (Technique #12)
- CPU job system prepares compute shader inputs
- GPU async compute processes data
- CPU/GPU parallelism maximized

### Particle Systems
- CPU updates emitters via job system
- GPU simulates particles via async compute
- 10-100x more particles possible

### ECS Data-Oriented Design (Technique #3)
- Pack entity component data for GPU upload
- Compute shaders process entity systems
- Results read back to CPU ECS

### Texture Streaming (Technique #10)
- Upload textures via DMA (copy engine) async
- Render with existing textures (graphics queue)
- Decompress textures (compute queue)
- All three pipelines working simultaneously

## Measurement/Profiling

### GPU Profiling Tools
```gdscript
# Use platform-specific tools for detailed async compute profiling:
# - NVIDIA Nsight Graphics (Windows/Linux)
# - AMD Radeon GPU Profiler (Windows/Linux)
# - RenderDoc (cross-platform, basic compute support)
# - PIX (Xbox/Windows DirectX 12)
```

### Custom Performance Metrics
```gdscript
class_name AsyncComputeProfiler
extends Node

var compute_dispatch_count: int = 0
var compute_time_us: int = 0

func profile_compute_dispatch(callable: Callable) -> void:
    var start = Time.get_ticks_usec()
    callable.call()
    var duration = Time.get_ticks_usec() - start

    compute_dispatch_count += 1
    compute_time_us += duration

func _process(_delta):
    Performance.add_custom_monitor("async/compute_dispatches",
        func(): return compute_dispatch_count)
    Performance.add_custom_monitor("async/compute_time_ms",
        func(): return compute_time_us / 1000.0)

    # Reset per frame
    compute_dispatch_count = 0
    compute_time_us = 0
```

### Key Metrics
- **GPU utilization**: Target 85-95% (both graphics and compute busy)
  - <70%: Not enough async work or poor overlap
  - >95%: Excellent utilization
- **Frame time improvement**: Measure with/without async
  - Enable/disable async compute, compare FPS
- **Queue stalls**: Graphics queue should not wait for compute
  - Use GPU profiler to detect stalls
- **Memory bandwidth**: Should be smoothed across frame
  - Spiky bandwidth indicates poor async overlap
