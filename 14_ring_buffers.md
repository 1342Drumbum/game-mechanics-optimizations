### 14. Ring Buffers for Streaming

**Category:** Memory & Data Management

**Problem It Solves:** Dynamic allocation for streaming data (audio samples, network packets, input commands) causes 100-300ns allocation overhead per operation and memory fragmentation. At 48kHz audio with 256-sample chunks, that's 187 allocations/second causing GC pressure and potential audio glitches. Ring buffers use fixed pre-allocated circular memory with O(1) enqueue/dequeue and zero runtime allocations, eliminating stutter and ensuring consistent <1μs latency.

**Technical Explanation:**
Ring buffer is fixed-size circular array with read and write pointers. Writer advances write pointer (wraps at end), reader advances read pointer. When write catches read, buffer is full (producer must wait or drop data). When read catches write, buffer is empty (consumer waits for data). Size must be power of 2 for fast modulo using bitwise AND: `index = (index + 1) & (size - 1)`. Lock-free implementation uses atomic operations for multi-threading: producer writes without locks, consumer reads without locks, only contention is when buffer nearly full/empty. Typically sized for 100-500ms of data (audio: 4800-24000 samples at 48kHz).

**Algorithmic Complexity:**
- Enqueue/Dequeue: O(1) constant time
- Space: O(n) where n = buffer capacity
- Allocation: Zero after initialization
- Synchronization: Lock-free with atomic operations (50-100ns) vs mutex (5-10μs)

**Implementation Pattern:**
```gdscript
class_name RingBuffer
extends RefCounted

var _buffer: PackedByteArray
var _read_pos: int = 0
var _write_pos: int = 0
var _capacity: int
var _mask: int  # For fast modulo (capacity - 1)

func _init(capacity: int) -> void:
    # Ensure capacity is power of 2
    assert(capacity > 0 and (capacity & (capacity - 1)) == 0, "Capacity must be power of 2")
    _capacity = capacity
    _mask = capacity - 1
    _buffer.resize(capacity)

func write(data: PackedByteArray) -> bool:
    var available = get_available_write()
    if data.size() > available:
        return false  # Buffer full, cannot write

    # Write data (may wrap around end)
    for byte in data:
        _buffer[_write_pos] = byte
        _write_pos = (_write_pos + 1) & _mask  # Fast modulo using bitmask

    return true

func write_bytes(data: PackedByteArray, offset: int, length: int) -> bool:
    # Optimized: write without iteration
    var available = get_available_write()
    if length > available:
        return false

    # Calculate wrap-around
    var first_chunk = min(length, _capacity - _write_pos)
    var second_chunk = length - first_chunk

    # Copy first chunk (until end of buffer)
    for i in range(first_chunk):
        _buffer[_write_pos + i] = data[offset + i]

    # Copy second chunk (from start of buffer if wrapped)
    if second_chunk > 0:
        for i in range(second_chunk):
            _buffer[i] = data[offset + first_chunk + i]

    _write_pos = (_write_pos + length) & _mask
    return true

func read(count: int) -> PackedByteArray:
    var available = get_available_read()
    var to_read = min(count, available)

    if to_read == 0:
        return PackedByteArray()

    var result = PackedByteArray()
    result.resize(to_read)

    # Read data (may wrap around)
    for i in range(to_read):
        result[i] = _buffer[_read_pos]
        _read_pos = (_read_pos + 1) & _mask

    return result

func peek(count: int) -> PackedByteArray:
    # Read without advancing read pointer
    var available = get_available_read()
    var to_read = min(count, available)

    var result = PackedByteArray()
    result.resize(to_read)

    var pos = _read_pos
    for i in range(to_read):
        result[i] = _buffer[pos]
        pos = (pos + 1) & _mask

    return result

func skip(count: int) -> void:
    # Advance read pointer without reading data
    var available = get_available_read()
    var to_skip = min(count, available)
    _read_pos = (_read_pos + to_skip) & _mask

func get_available_read() -> int:
    # Data available to read
    return (_write_pos - _read_pos) & _mask

func get_available_write() -> int:
    # Space available to write (leave 1 byte to distinguish full/empty)
    return (_capacity - get_available_read() - 1)

func is_empty() -> bool:
    return _read_pos == _write_pos

func is_full() -> bool:
    return get_available_write() == 0

func clear() -> void:
    _read_pos = 0
    _write_pos = 0

# Specialized ring buffer for audio streaming
class AudioRingBuffer extends RingBuffer:
    const SAMPLE_SIZE = 4  # 32-bit float = 4 bytes
    var sample_rate: int
    var channels: int

    func _init(duration_seconds: float, p_sample_rate: int = 48000, p_channels: int = 2):
        sample_rate = p_sample_rate
        channels = p_channels

        # Calculate buffer size in bytes
        var sample_count = int(duration_seconds * sample_rate * channels)
        var byte_count = sample_count * SAMPLE_SIZE

        # Round up to next power of 2
        var capacity = _next_power_of_2(byte_count)
        super._init(capacity)

    func write_samples(samples: PackedFloat32Array) -> bool:
        # Convert float samples to bytes
        var bytes = PackedByteArray()
        bytes.resize(samples.size() * SAMPLE_SIZE)

        for i in range(samples.size()):
            var sample = samples[i]
            var packed = PackedFloat32Array([sample])
            var sample_bytes = packed.to_byte_array()
            for j in range(SAMPLE_SIZE):
                bytes[i * SAMPLE_SIZE + j] = sample_bytes[j]

        return write(bytes)

    func read_samples(count: int) -> PackedFloat32Array:
        # Read samples as floats
        var bytes = read(count * SAMPLE_SIZE)
        var samples = PackedFloat32Array()
        samples.resize(count)

        for i in range(count):
            var sample_bytes = bytes.slice(i * SAMPLE_SIZE, (i + 1) * SAMPLE_SIZE)
            var packed = sample_bytes.to_float32_array()
            samples[i] = packed[0] if packed.size() > 0 else 0.0

        return samples

    func get_latency_ms() -> float:
        # Calculate current buffer latency
        var samples_buffered = get_available_read() / SAMPLE_SIZE / channels
        return (samples_buffered / float(sample_rate)) * 1000.0

    static func _next_power_of_2(n: int) -> int:
        n -= 1
        n |= n >> 1
        n |= n >> 2
        n |= n >> 4
        n |= n >> 8
        n |= n >> 16
        return n + 1

# Real-world example: Audio streaming system
class AudioStreamingSystem:
    var audio_buffer: AudioRingBuffer
    var audio_player: AudioStreamPlayer
    var is_streaming: bool = false

    func _init():
        # 0.5 second buffer at 48kHz stereo
        audio_buffer = AudioRingBuffer.new(0.5, 48000, 2)

    func start_streaming(audio_source: String) -> void:
        is_streaming = true
        # Producer thread: Read audio from disk/network
        _start_producer_thread(audio_source)
        # Consumer: Audio callback pulls from buffer
        _start_audio_playback()

    func _start_producer_thread(source: String) -> void:
        var thread = Thread.new()
        thread.start(func():
            while is_streaming:
                # Read chunk from source (file/network)
                var chunk = _read_audio_chunk(source, 4096)

                # Write to ring buffer (blocks if full)
                while not audio_buffer.write_samples(chunk):
                    OS.delay_usec(1000)  # Wait 1ms for space

                # Check for underrun
                if audio_buffer.get_latency_ms() < 50:
                    push_warning("Audio buffer underrun!")
        )

    func _audio_callback() -> PackedFloat32Array:
        # Called by audio thread at regular intervals
        var samples_needed = 256  # Typical audio callback size

        if audio_buffer.get_available_read() < samples_needed * 4:
            # Buffer underrun - return silence
            push_warning("Audio glitch: buffer empty")
            var silence = PackedFloat32Array()
            silence.resize(samples_needed)
            return silence

        return audio_buffer.read_samples(samples_needed)

# Network packet buffering example
class NetworkPacketBuffer:
    var ring_buffer: RingBuffer
    const MAX_PACKET_SIZE = 1400  # MTU for internet

    func _init(buffer_packets: int = 256):
        # Allocate ring buffer for 256 packets
        ring_buffer = RingBuffer.new(_next_pow2(buffer_packets * MAX_PACKET_SIZE))

    func enqueue_packet(packet_data: PackedByteArray) -> bool:
        if packet_data.size() > MAX_PACKET_SIZE:
            push_error("Packet too large: %d bytes" % packet_data.size())
            return false

        # Write packet size (2 bytes) + packet data
        var size_bytes = PackedByteArray([
            packet_data.size() & 0xFF,
            (packet_data.size() >> 8) & 0xFF
        ])

        if not ring_buffer.write(size_bytes):
            return false  # Buffer full

        if not ring_buffer.write(packet_data):
            # Rollback size write (complicated, better to check space first)
            push_error("Failed to write packet data")
            return false

        return true

    func dequeue_packet() -> PackedByteArray:
        # Read packet size
        var size_bytes = ring_buffer.read(2)
        if size_bytes.size() < 2:
            return PackedByteArray()  # Empty buffer

        var packet_size = size_bytes[0] | (size_bytes[1] << 8)

        # Read packet data
        return ring_buffer.read(packet_size)

    func get_queued_packets() -> int:
        # Estimate number of packets (not exact due to variable sizes)
        return ring_buffer.get_available_read() / (MAX_PACKET_SIZE / 2)

    static func _next_pow2(n: int) -> int:
        var power = 1
        while power < n:
            power *= 2
        return power

# Input buffering for fighting games
class InputRingBuffer:
    var inputs: Array[int] = []  # Circular array of input states
    var timestamps: Array[int] = []  # Frame numbers
    var read_pos: int = 0
    var write_pos: int = 0
    var capacity: int

    func _init(buffer_frames: int = 60):
        capacity = buffer_frames
        inputs.resize(capacity)
        timestamps.resize(capacity)

    func record_input(input_state: int, frame: int) -> void:
        inputs[write_pos] = input_state
        timestamps[write_pos] = frame
        write_pos = (write_pos + 1) % capacity

        # Overwrite old data if buffer full
        if write_pos == read_pos:
            read_pos = (read_pos + 1) % capacity

    func get_input_at_frame(frame: int) -> int:
        # Find input at specific frame (for rollback netcode)
        var pos = read_pos
        while pos != write_pos:
            if timestamps[pos] == frame:
                return inputs[pos]
            pos = (pos + 1) % capacity
        return 0  # No input found

    func clear_old_inputs(current_frame: int, keep_frames: int = 10) -> void:
        # Remove inputs older than keep_frames
        while read_pos != write_pos:
            if current_frame - timestamps[read_pos] > keep_frames:
                read_pos = (read_pos + 1) % capacity
            else:
                break
```

**Key Parameters:**
- Buffer size: 2-10x typical chunk size (audio: 0.2-1.0s, network: 100-500 packets)
- Must be power of 2 for fast modulo (1024, 2048, 4096, 8192 bytes typical)
- Chunk size: Audio 256-2048 samples, network 512-1400 bytes (MTU)
- Watermark thresholds: Underrun <20% full, overflow >80% full

**Edge Cases:**
- Buffer overflow: Drop oldest data or block producer (depends on use case)
- Buffer underrun: Return silence (audio) or wait (network)
- Wraparound: Handle correctly when read/write cross buffer boundary
- Thread safety: Lock-free requires atomic operations (GDScript limitations)
- Power-of-2 sizing: Non-power-of-2 size requires slower modulo operation

**When NOT to Use:**
- Random access needed (use array instead)
- Unpredictable data sizes (variable packet sizes may waste space)
- Single producer/consumer with predictable timing (simple queue sufficient)
- Debugging (harder to inspect than linear buffer)

**Examples from Shipped Games:**

1. **Fmod/Wwise (Audio Middleware):** All game audio engines use ring buffers for streaming, prevents audio glitches from GC pauses, handles 100+ simultaneous streams
2. **Source Engine (Valve):** Network packet buffering via ring buffers, enabled smooth multiplayer in Half-Life 2/CS:GO with 64 players
3. **Fighting Games (GGPO Rollback Netcode):** Input ring buffer stores 60+ frames of input history for rollback, enables competitive online play with <100ms latency
4. **VoIP Applications (Discord, Teamspeak):** Ring buffers for microphone → network streaming, maintains <50ms latency for real-time voice
5. **Video Streaming (YouTube/Netflix):** Decoder ring buffers smooth playback despite network jitter, typical 5-30 second buffer

**Platform Considerations:**
- **PC:** Large buffers acceptable (1-10MB), fast memory access
- **Mobile:** Smaller buffers (100KB-1MB) due to memory constraints
- **Consoles:** Optimized for streaming, 1-5MB audio buffers typical
- **VR:** Low latency critical, smaller buffers (50-200ms) to reduce audio lag
- **Web:** SharedArrayBuffer required for lock-free multi-threading

**Godot-Specific Notes:**
- Use PackedByteArray for maximum efficiency (C++ backend)
- GDScript lacks atomic operations (use Mutex for thread safety)
- Consider GDExtension for true lock-free implementation
- AudioStreamGenerator uses internal ring buffer for custom audio
- Godot's networking has built-in buffering (custom ring buffer rarely needed)

**Synergies:**
- Essential for Job System (lock-free communication between threads)
- Pairs with Fixed Timestep (buffer inputs for deterministic playback)
- Enables Audio Streaming without stutter
- Critical for Network Buffering (smooth multiplayer)
- Supports Replay Systems (input recording)

**Measurement/Profiling:**
```gdscript
func profile_ring_buffer():
    var buffer = RingBuffer.new(4096)
    const ITERATIONS = 100000

    # Benchmark write performance
    var test_data = PackedByteArray([1, 2, 3, 4, 5, 6, 7, 8])

    var start = Time.get_ticks_usec()
    for i in range(ITERATIONS):
        buffer.write(test_data)
        if buffer.get_available_write() < 100:
            buffer.clear()  # Reset when nearly full
    var write_time = Time.get_ticks_usec() - start

    print("Write performance: %.2f ns/write" % (float(write_time) * 1000.0 / ITERATIONS))
    # Expected: 50-200 ns per write (depending on GDScript overhead)

    # Benchmark read performance
    buffer.clear()
    for i in range(512):  # Fill buffer
        buffer.write(test_data)

    start = Time.get_ticks_usec()
    for i in range(ITERATIONS):
        var data = buffer.read(8)
        if buffer.is_empty():
            # Refill
            for j in range(512):
                buffer.write(test_data)
    var read_time = Time.get_ticks_usec() - start

    print("Read performance: %.2f ns/read" % (float(read_time) * 1000.0 / ITERATIONS))

# Monitor audio buffer health
func monitor_audio_buffer(buffer: AudioRingBuffer):
    var latency = buffer.get_latency_ms()
    var fill_percent = float(buffer.get_available_read()) / buffer._capacity * 100.0

    if latency < 50:
        push_warning("Audio buffer low: %.1fms (risk of underrun)" % latency)
    elif latency > 300:
        push_warning("Audio buffer high: %.1fms (excessive latency)" % latency)

    # Ideal: 100-200ms latency (2-4 frames of audio buffered)
```

**Target Metrics:**
- Latency: Audio 50-200ms, Network <100ms, Input <16ms (1 frame)
- Underruns: <0.1% of the time (imperceptible)
- CPU overhead: <1% of frame time for buffering operations
- Memory: 0.5-10MB depending on use case
