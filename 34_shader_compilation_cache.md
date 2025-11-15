### 34. Shader Compilation Cache

**Category:** Performance - Rendering Optimizations

**Problem It Solves:** Runtime shader compilation stutter. First-time shader use triggers 50-500ms compilation stall (frame drops to 2-20fps). Complex shader with 100 variants × 200ms = 20 seconds of stuttering. Shader cache pre-compiles or saves compiled shaders, eliminating runtime compilation: 200ms stall → 0.1ms cache lookup, saving 199.9ms per unique shader.

**Technical Explanation:**
GPUs require shaders compiled to native instructions. Runtime compilation causes frame hitches. Solutions: 1) Pre-compile all shader variants at build/first-run, 2) Cache compiled shaders to disk, 3) Async compilation with fallback. Modern approach: warmup frame renders all possible shader combinations offscreen, populates cache. Persistent cache stores binaries between runs. Driver-level (D3D12 PSO cache, Vulkan pipeline cache) or engine-level. Critical for stutter-free gameplay, especially with complex material systems (PBR with many permutations).

**Algorithmic Complexity:**
- No cache: O(n) compilation per unique shader, 50-500ms each
- With cache: O(n) first run (warmup), O(1) cache hits thereafter (0.1ms)
- Cache size: 10-500MB depending on shader complexity
- Hit rate: Target >95% after warmup
- Cold start: 5-30 seconds warmup, then smooth

**Implementation Pattern:**
```gdscript
# Godot Shader Compilation Cache Manager
class_name ShaderCache
extends Node

const CACHE_PATH = "user://shader_cache/"
const CACHE_VERSION = 1

var cached_shaders: Dictionary = {}  # Hash -> ShaderRID
var compilation_queue: Array = []
var warmup_in_progress: bool = false

func _ready():
    _ensure_cache_directory()
    _load_persistent_cache()

func _ensure_cache_directory():
    var dir = DirAccess.open("user://")
    if not dir.dir_exists("shader_cache"):
        dir.make_dir("shader_cache")

func cache_shader(shader: Shader, variants: Array = []) -> bool:
    var shader_hash = _get_shader_hash(shader)

    # Check if already cached
    if shader_hash in cached_shaders:
        return true

    # Compile shader
    var compiled = _compile_shader(shader, variants)

    if compiled:
        cached_shaders[shader_hash] = compiled
        _save_to_disk(shader_hash, shader)
        return true

    return false

func get_cached_shader(shader: Shader) -> RID:
    var shader_hash = _get_shader_hash(shader)

    if shader_hash in cached_shaders:
        return cached_shaders[shader_hash]

    # Not cached, compile now
    print("Cache miss: ", shader_hash)
    if cache_shader(shader):
        return cached_shaders[shader_hash]

    return RID()

func warmup_shaders(materials: Array):
    print("Starting shader warmup...")
    warmup_in_progress = true

    # Collect all unique shaders
    var shaders_to_compile = {}

    for material in materials:
        if material is ShaderMaterial:
            var shader = material.shader
            if shader:
                var hash = _get_shader_hash(shader)
                shaders_to_compile[hash] = shader

    print("Found ", shaders_to_compile.size(), " unique shaders")

    # Compile all shaders (may take time)
    for shader_hash in shaders_to_compile:
        cache_shader(shaders_to_compile[shader_hash])

    warmup_in_progress = false
    print("Shader warmup complete!")

func _compile_shader(shader: Shader, variants: Array) -> RID:
    # In Godot, compilation happens automatically
    # This simulates the concept

    # Get shader RID
    var rid = shader.get_rid()

    # Force compilation by using shader
    # (Godot compiles on first use)

    return rid

func _get_shader_hash(shader: Shader) -> String:
    # Hash shader source code
    var code = shader.code
    return code.md5_text()

func _save_to_disk(hash: String, shader: Shader):
    var file_path = CACHE_PATH + hash + ".cache"
    var file = FileAccess.open(file_path, FileAccess.WRITE)

    if file:
        # Save metadata
        file.store_32(CACHE_VERSION)
        file.store_string(shader.code)
        file.close()

func _load_persistent_cache():
    var dir = DirAccess.open(CACHE_PATH)
    if not dir:
        return

    dir.list_dir_begin()
    var file_name = dir.get_next()

    while file_name != "":
        if file_name.ends_with(".cache"):
            _load_cached_file(CACHE_PATH + file_name)

        file_name = dir.get_next()

    dir.list_dir_end()

    print("Loaded ", cached_shaders.size(), " cached shaders")

func _load_cached_file(path: String):
    var file = FileAccess.open(path, FileAccess.READ)
    if not file:
        return

    var version = file.get_32()
    if version != CACHE_VERSION:
        file.close()
        return

    var code = file.get_string()
    file.close()

    # Recreate shader
    var shader = Shader.new()
    shader.code = code

    var hash = _get_shader_hash(shader)
    cached_shaders[hash] = shader.get_rid()

# Async shader compilation
class AsyncShaderCompiler:
    signal shader_compiled(shader_hash: String)

    var compilation_queue: Array = []
    var compiling: bool = false
    var max_compilations_per_frame: int = 3

    func queue_compilation(shader: Shader):
        compilation_queue.append(shader)

        if not compiling:
            _start_compilation()

    func _start_compilation():
        compiling = true
        # Would use threading in real implementation

    func _process(_delta):
        if compilation_queue.is_empty():
            compiling = false
            return

        # Compile a few shaders per frame
        var count = min(max_compilations_per_frame, compilation_queue.size())

        for i in range(count):
            var shader = compilation_queue.pop_front()
            _compile_shader(shader)

    func _compile_shader(shader: Shader):
        # Compile shader
        var hash = shader.code.md5_text()

        # Emit signal when done
        shader_compiled.emit(hash)

# PSO (Pipeline State Object) cache for Vulkan/Metal/DX12
class PipelineStateCache:
    var cache_file: String = "user://pso_cache.bin"
    var cached_states: Dictionary = {}

    func _init():
        _load_cache()

    func get_or_create_pso(state_desc: Dictionary) -> RID:
        var hash = _hash_state_desc(state_desc)

        if hash in cached_states:
            return cached_states[hash]

        # Create new PSO
        var pso = _create_pipeline_state(state_desc)
        cached_states[hash] = pso

        _save_cache()
        return pso

    func _hash_state_desc(desc: Dictionary) -> String:
        return JSON.stringify(desc).md5_text()

    func _create_pipeline_state(desc: Dictionary) -> RID:
        # Create pipeline state object
        # Platform specific
        return RID()

    func _load_cache():
        if not FileAccess.file_exists(cache_file):
            return

        # Load PSO cache from disk
        pass

    func _save_cache():
        # Save PSO cache to disk
        pass

# Shader variant pre-compilation
class ShaderVariantCompiler:
    func precompile_variants(shader: Shader, defines: Array[Dictionary]):
        # Compile all possible shader variants

        for define_set in defines:
            _compile_with_defines(shader, define_set)

    func _compile_with_defines(shader: Shader, defines: Dictionary):
        # Apply defines and compile
        # This would modify shader code with #defines
        var modified_code = _inject_defines(shader.code, defines)

        var variant_shader = Shader.new()
        variant_shader.code = modified_code

        # Trigger compilation
        var material = ShaderMaterial.new()
        material.shader = variant_shader

    func _inject_defines(code: String, defines: Dictionary) -> String:
        var result = code

        for key in defines:
            var define_line = "#define " + key + " " + str(defines[key]) + "\n"
            result = define_line + result

        return result

# Statistics and monitoring
class ShaderCacheStats:
    var cache_hits: int = 0
    var cache_misses: int = 0
    var compilation_time_ms: float = 0.0

    func record_hit():
        cache_hits += 1

    func record_miss(compile_time: float):
        cache_misses += 1
        compilation_time_ms += compile_time

    func get_hit_rate() -> float:
        var total = cache_hits + cache_misses
        if total == 0:
            return 0.0

        return float(cache_hits) / total

    func print_stats():
        print("Shader Cache Statistics:")
        print("  Hits: ", cache_hits)
        print("  Misses: ", cache_misses)
        print("  Hit Rate: ", get_hit_rate() * 100.0, "%")
        print("  Total Compilation Time: ", compilation_time_ms, "ms")
```

**Key Parameters:**
- Cache size: 50-500MB (depends on shader complexity)
- Warmup time: 5-30 seconds on first run
- Async compilation: 2-5 shaders per frame
- Cache expiry: Version changes invalidate cache
- Variants: 10-1000 per shader (materials, lighting configs)

**Edge Cases:**
- Cache corruption: Validate and rebuild if needed
- Driver updates: May invalidate cache
- Cross-platform: Separate caches per GPU vendor
- Dynamic shaders: Can't pre-cache, use async compilation
- Memory limits: Evict least-used shaders

**When NOT to Use:**
- Simple games with <10 shaders
- Shaders compile very fast (<10ms)
- Build-time pre-compilation sufficient
- Platform with instant compilation (rare)

**Examples from Shipped Games:**
1. **Destiny 2**: PSO cache, loads 2GB shader cache on startup. Eliminates stutter after first run.
2. **Horizon Zero Dawn PC**: Shader pre-compilation at startup (3-5 mins). Smooth gameplay after warmup.
3. **Monster Hunter World PC**: Infamous shader compilation stutter on launch. Later patched with cache system.
4. **Gears 5**: Extensive PSO cache, <1% cache misses after warmup. Critical for 60fps+.
5. **Death Stranding**: Pre-compile on first launch, persistent cache. Minimal hitching after setup.

**Platform Considerations:**
- **PC**: Essential (many GPU configs), 100-500MB cache
- **Console**: Less critical (fixed hardware), still beneficial
- **Mobile**: Smaller cache (50-100MB), careful memory use
- **Vulkan/DX12**: PSO cache critical (explicit API)
- **OpenGL**: Less sophisticated, basic cache

**Godot-Specific Notes:**
- Godot compiles shaders on first use automatically
- No explicit cache API exposed
- Can warm up by rendering offscreen
- Mobile exports pre-compile shaders
- Godot 4.x: Better shader compilation handling

**Synergies:**
- Pairs with **Async Resource Loading**
- Works with **Streaming Systems**
- Enables **Material Variants**
- Supports **Hot Reload** in editor

**Measurement:**
- Track compilation stalls: Should be 0 after warmup
- Monitor cache hit rate: >95% target
- Measure warmup time: 5-30 seconds acceptable
- Profile first-time use: <10ms with cache vs 50-500ms without

**Additional Resources:**
- GDC 2018: "Shader Compilation in Destiny 2"
- Khronos: Vulkan Pipeline Cache Guide
