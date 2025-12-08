# KATANA Benchmark Methodology

This document describes the unified methodology for measuring and reporting performance benchmarks in the KATANA framework.

## Table of Contents

- [Overview](#overview)
- [Measurement Modes](#measurement-modes)
- [System Configuration](#system-configuration)
- [Benchmark Structure](#benchmark-structure)
- [Naming Conventions](#naming-conventions)
- [Metrics and Reporting](#metrics-and-reporting)
- [Reproducibility Guidelines](#reproducibility-guidelines)
- [Benchmark Suite](#benchmark-suite)

---

## Overview

KATANA benchmarks are designed to provide:
- **Predictable**: Consistent results across runs on the same hardware
- **Reproducible**: Clear methodology for others to validate results
- **Comprehensive**: Cover all critical framework components
- **Actionable**: Identify performance bottlenecks and regressions

All benchmarks follow a three-phase execution pattern:
1. **Warmup** (2-5 seconds): Stabilize CPU frequency, warm caches, establish baseline
2. **Measurement** (5-10 seconds): Collect performance data under stable conditions
3. **Cooldown** (1-2 seconds): Allow system to stabilize before next benchmark

---

## Measurement Modes

KATANA benchmarks support three primary modes, each targeting different performance characteristics:

### 1. Latency Mode

**Objective**: Minimize response time (p99/p999) with low concurrency.

**Configuration**:
```bash
export KATANA_REACTORS=1
export KATANA_LOG_LEVEL=ERROR
export KATANA_ARENA_SIZE=65536

# Pin reactor to dedicated core
taskset -c 0 ./benchmark --mode latency --threads 1 --connections 1
```

**Use Case**: Interactive applications, user-facing APIs, low-latency requirements

**Expected Metrics**:
- p50: < 0.5ms (hello world)
- p95: < 1.0ms
- p99: < 2.0ms
- p999: < 10ms

**Key Factors**:
- Single reactor eliminates inter-core communication
- Low connection count reduces scheduler overhead
- CPU pinning prevents migrations and cache thrashing

### 2. Throughput Mode

**Objective**: Maximize requests per second (RPS) across all cores.

**Configuration**:
```bash
export KATANA_REACTORS=$(nproc)
export KATANA_CPU_PINNING=1
export KATANA_LOG_LEVEL=WARN

# Optionally use NUMA-aware binding
numactl --cpunodebind=0 --membind=0 ./benchmark --mode throughput \
    --threads $(nproc) --connections 1000
```

**Use Case**: Batch processing, high-volume APIs, maximum resource utilization

**Expected Metrics**:
- RPS: 100k-500k (hello world, depends on hardware)
- p99: < 10ms
- CPU utilization: 80-95% across all cores
- Error rate: < 0.01%

**Key Factors**:
- All cores utilized via reactor-per-core model
- High connection count saturates all reactors
- CPU pinning maintains cache locality per reactor

### 3. Fan-out Mode

**Objective**: Handle many concurrent connections with single reactor.

**Configuration**:
```bash
export KATANA_REACTORS=1
export KATANA_MAX_CONNECTIONS=10000
ulimit -n 100000

./benchmark --mode fanout --threads 1 --connections 500
```

**Use Case**: WebSocket servers, long-polling, connection-heavy workloads

**Expected Metrics**:
- Connections: 500-1000+ handled by single reactor
- p99: < 20ms (acceptable degradation with high concurrency)
- Memory: Linear growth with connection count
- No connection failures or timeouts

**Key Factors**:
- Reactor event loop efficiency tested
- Epoll/io_uring scalability validated
- Memory management under high FD count

---

## System Configuration

###  Hardware Requirements

**Minimum**:
- CPU: 2+ cores, x86_64 or ARM64
- RAM: 4GB
- OS: Linux 4.14+ (for io_uring: 5.1+)

**Recommended**:
- CPU: 8+ cores, x86_64 with AVX2
- RAM: 16GB
- OS: Linux 5.10+
- Network: 10Gbps or loopback

### Linux Kernel Parameters

Add to `/etc/sysctl.conf` for optimal performance:

```bash
# Network stack
net.core.somaxconn = 4096
net.core.netdev_max_backlog = 8192
net.ipv4.tcp_max_syn_backlog = 4096

# Connection management
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_tw_reuse = 1

# Buffer sizes
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# Apply with: sudo sysctl -p
```

### File Descriptor Limits

Add to `/etc/security/limits.conf`:

```
* soft nofile 65536
* hard nofile 1048576
```

Verify with: `ulimit -n`

### CPU Frequency Scaling

Disable dynamic frequency scaling for consistent benchmarks:

```bash
# Check current governor
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Set performance mode
sudo cpupower frequency-set -g performance

# Verify
cpupower frequency-info
```

### CPU Pinning (Optional)

For latency-critical benchmarks, pin reactor threads to dedicated cores:

```bash
# Pin single reactor to core 0
taskset -c 0 ./my_benchmark

# Pin multi-reactor (8 threads) to cores 0-7
taskset -c 0-7 ./my_benchmark

# NUMA-aware (2-socket system)
numactl --cpunodebind=0 --membind=0 ./my_benchmark  # Socket 0
```

Check NUMA topology:
```bash
numactl --hardware
lscpu | grep NUMA
```

---

## Benchmark Structure

All KATANA benchmarks follow this standard structure:

### Code Pattern

```cpp
benchmark_result benchmark_component_scenario() {
    // Configuration
    const size_t num_operations = 100000;
    const size_t sample_rate = 100;  // Sample every Nth operation

    // Setup
    std::vector<double> latencies;
    latencies.reserve(num_operations / sample_rate + 2);

    // Warmup (optional but recommended)
    for (size_t i = 0; i < num_operations / 10; ++i) {
        perform_operation();
    }

    // Measurement
    auto start = std::chrono::steady_clock::now();

    for (size_t i = 0; i < num_operations;) {
        size_t batch = std::min(sample_rate, num_operations - i);
        auto op_start = std::chrono::steady_clock::now();

        for (size_t j = 0; j < batch; ++j, ++i) {
            perform_operation();
        }

        auto op_end = std::chrono::steady_clock::now();
        double latency_us = std::chrono::duration_cast<std::chrono::nanoseconds>(
            op_end - op_start).count() / (1000.0 * batch);
        latencies.push_back(latency_us);
    }

    auto end = std::chrono::steady_clock::now();
    auto duration_ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        end - start).count();

    // Calculate percentiles
    std::sort(latencies.begin(), latencies.end());

    return benchmark_result{
        .name = "Component Scenario",
        .throughput = (num_operations * 1000.0) / duration_ms,
        .latency_p50 = percentile(latencies, 0.50),
        .latency_p99 = percentile(latencies, 0.99),
        .latency_p999 = percentile(latencies, 0.999),
        .operations = num_operations,
        .duration_ms = duration_ms
    };
}
```

### Percentile Calculation

Use linear interpolation for smooth percentile curves:

```cpp
double percentile(const std::vector<double>& sorted_values, double pct) {
    if (sorted_values.empty()) return 0.0;

    double clamped = std::min(1.0, std::max(0.0, pct));
    double index = clamped * (sorted_values.size() - 1);
    size_t lower = static_cast<size_t>(index);
    size_t upper = std::min(lower + 1, sorted_values.size() - 1);
    double fraction = index - lower;

    return sorted_values[lower] * (1.0 - fraction) +
           sorted_values[upper] * fraction;
}
```

---

## Naming Conventions

Consistent naming enables automated parsing and reporting.

### Metric Names

Format: `<component>_<operation>_<metric>`

Examples:
- `arena_alloc_p99_us` - Arena allocation p99 latency in microseconds
- `http_parse_throughput_ops_sec` - HTTP parser throughput in operations per second
- `router_dispatch_p50_ms` - Router dispatch p50 latency in milliseconds

### Scenario Names

Format: `<threads>t_<connections>c_<payload_size>`

Examples:
- `1t_1c_64b` - Single thread, single connection, 64-byte payload
- `8t_128c_1kb` - 8 threads, 128 connections, 1KB payload
- `16t_256c_mixed` - 16 threads, 256 connections, mixed payloads

### Mode Suffixes

- `_latency` - Latency-optimized configuration
- `_throughput` - Throughput-optimized configuration
- `_fanout` - Connection fan-out configuration
- `_stress` - Stress/stability test

Examples:
- `compute_api_1t_1c_latency` - Compute API in latency mode
- `validation_api_8t_128c_throughput` - Validation API in throughput mode

---

## Metrics and Reporting

### Primary Metrics

| Metric | Description | Unit | Target (hello world) |
|--------|-------------|------|---------------------|
| **Throughput** | Operations per second | ops/sec or req/s | 100k-500k |
| **Latency p50** | Median response time | ms or μs | < 0.5ms |
| **Latency p95** | 95th percentile | ms or μs | < 1.0ms |
| **Latency p99** | 99th percentile | ms or μs | < 2.0ms |
| **Latency p999** | 99.9th percentile | ms or μs | < 10ms |
| **Duration** | Measurement window | ms or sec | 5-10s |
| **Operations** | Total ops completed | count | 100k-500k |

### Secondary Metrics

| Metric | Description | Use Case |
|--------|-------------|----------|
| **IQR** | Interquartile range (p75-p25) | Measure spread/variance |
| **Error Rate** | Failed operations / total | Validate correctness |
| **CPU Usage** | Per-core utilization % | Identify bottlenecks |
| **Memory RSS** | Resident set size | Detect leaks |
| **Syscalls** | System call count | Optimize I/O paths |

### Report Format

Benchmarks output to `BENCHMARK_RESULTS.md` in structured format:

```markdown
## Component Name

| Benchmark | Value | Unit |
|-----------|-------|------|
| Scenario p50 | X.XXX | ms |
| Scenario p99 | X.XXX | ms |
| Scenario throughput | XXXXX.XX | req/s |
| Scenario duration | XXX | ms |
| Scenario operations | XXXXX | ops |
```

Auto-generated via `generate_benchmark_report.py`.

---

## Reproducibility Guidelines

### Environment Documentation

For every benchmark run, document:

**Hardware**:
- CPU model: `lscpu | grep "Model name"`
- Core count: `nproc`
- CPU frequency: `lscpu | grep MHz`
- RAM: `free -h`
- NUMA topology: `numactl --hardware`

**Software**:
- OS: `uname -a`
- Kernel: `uname -r`
- Compiler: `g++ --version` or `clang++ --version`
- CMake: `cmake --version`
- Build type: `Release` / `Debug` / `RelWithDebInfo`

**Example**:
```
CPU: Intel Xeon E5-2680 v4 @ 2.40GHz (28 cores)
RAM: 128GB DDR4
OS: Ubuntu 22.04 LTS
Kernel: 5.15.0-60-generic
Compiler: GCC 11.3.0
Build: Release (-O3 -march=native -DNDEBUG)
```

### Baseline Establishment

Run benchmarks 3-5 times and compute statistics:

```bash
for i in {1..5}; do
  ./run_benchmarks.sh > results_run${i}.txt
done

# Analyze variance
python analyze_variance.py results_run*.txt
```

**Acceptance Criteria**:
- Standard deviation < 5% of mean for throughput
- p99 variance < 10% across runs
- No outlier runs (±2 standard deviations)

### Environmental Factors

Control for:
- **CPU temperature**: Monitor with `sensors`, ensure < 80°C
- **Background load**: `top` / `htop` - ensure idle system
- **Network conditions**: Use loopback for consistent results
- **Time of day**: Some systems have thermal throttling patterns

### Comparison Methodology

When comparing performance:
1. **Same hardware**: Always compare on identical hardware
2. **Same configuration**: Use identical system settings
3. **Same build**: Same compiler, flags, and dependencies
4. **Same environmental conditions**: Temperature, load, time
5. **Statistical significance**: Use t-test or Mann-Whitney U test

---

## Benchmark Suite

Current KATANA benchmarks organized by component:

### Core Benchmarks (`performance_benchmark.cpp`)

| Benchmark | Operations | Scenario |
|-----------|-----------|----------|
| Arena Allocations | 500k | 64-byte objects |
| Circular Buffer | 500k | Read/write 64-byte chunks |
| Ring Buffer (Single Thread) | 1M | Push/pop pattern |
| Ring Buffer (Concurrent 4x4) | 1M | 4 producers, 4 consumers |
| Ring Buffer (High Contention 8x8) | 1M | 8 producers, 8 consumers |
| Memory Allocations (String Queue) | 100k | std::string allocations |
| HTTP Parser (Complete) | 50k | Full request parsing |
| HTTP Parser (Fragmented) | 50k | Incremental parsing |
| SIMD CRLF Search (1.5KB) | 100k | Small buffer scan |
| SIMD CRLF Search (16KB) | 50k | Large buffer scan |

### HTTP Benchmarks

**Headers** (`headers_benchmark.cpp`):
- Case-insensitive comparison (500k ops)
- Headers set (5 standard fields, 100k ops)
- Headers set (4 custom fields, 50k ops)
- Headers get (3 lookups, 200k ops)
- Headers iteration (5 fields, 100k ops)

**Parsing** (`headers_benchmark.cpp`):
- Minimal request (1500 samples, keep-alive)
- Medium request (1500 samples, multiple headers)
- Large headers (1500 samples, many headers)

### IO Buffer (`io_buffer_benchmark.cpp`)

- Append (64 bytes, 100k ops)
- Append (4KB, 50k ops)
- Read/Write (256B, 100k ops)
- Writable/Commit (128B, 100k ops)
- Scatter/Gather Write (3 buffers, 100k ops)

### Router (`router_benchmark.cpp`)

- Router dispatch (hits, 200k ops)
- Router dispatch (404 not found, 200k ops)
- Router dispatch (405 method not allowed, 200k ops)

### MPSC Queue (`mpsc_benchmark.cpp`)

- Single producer (1M ops)
- 2 producers (1M ops)
- 4 producers (1M ops)
- 8 producers (1M ops)
- Bounded queue (1024 capacity, 500k ops)

### Timer System (`timer_benchmark.cpp`)

- Add operations (100k ops)
- Cancel operations (50k ops)
- Tick operations (50k ops)
- Execution (10k timers)

### OpenAPI (`openapi_benchmark.cpp`)

- Small spec parsing
- Medium spec parsing
- Large spec parsing
- $ref resolution
- allOf merging

### Generated API (`generated_api_benchmark.cpp`)

- Dispatch + parse integration (200k ops)

### Scalability (`simple_benchmark.cpp`)

**Latency**:
- Single connection, keep-alive (10s run)
- Measures: p50, p95, p99, p999, throughput

**Thread Scaling**:
- 1, 4, 8 threads (2s each)
- Fixed connection count per thread

**Fan-out**:
- 32, 64, 128, 256 concurrent connections (2.5s each)

**Connection Churn**:
- Close-after-each-request (4 threads, 5s)

**Stability**:
- Sustained load (8 threads, 5s)

### Codegen Examples

**Compute API** (`compute_api`):
- POST /compute/sum with array sizes: 1, 8, 64, 256, 1024
- Thread scaling: 1, 4, 8, 16 threads
- Payload sizes tested for performance

**Validation API** (`validation_api`):
- POST /user/register with valid/invalid payloads
- Success rate tracking
- Thread scaling: 4, 8 threads

---

## Benchmark Execution

### Running Individual Benchmarks

```bash
# Build benchmarks
cmake --preset bench
cmake --build --preset bench

# Run specific benchmark
./build/bench/benchmark/performance_benchmark
./build/bench/benchmark/simple_benchmark
./build/bench/benchmark/headers_benchmark
```

### Running Full Benchmark Suite

```bash
# Automated suite
./run_benchmarks.sh

# With specific profile
./run_benchmarks.sh --profile latency
./run_benchmarks.sh --profile throughput

# Generate report
python generate_benchmark_report.py
```

### Docker Benchmarks

```bash
# Build and run in isolated container
docker build -t katana-bench -f Dockerfile.bench .
docker run --rm katana-bench
```

---

## Interpreting Results

### Latency Analysis

**Good**:
- Low p50-p99 gap (< 2x): Consistent performance
- Low p99-p999 gap (< 5x): No extreme outliers
- Predictable IQR: Stable under load

**Concerning**:
- High p99-p999 gap (> 10x): Outliers indicate GC pauses, context switches, or resource contention
- Bimodal distribution: Cache misses, NUMA migration, or thermal throttling
- Increasing latency over time: Memory leaks or resource exhaustion

### Throughput Analysis

**Good**:
- Linear scaling with cores (up to hardware thread count)
- CPU utilization 80-95% (not saturated, but efficient)
- Low error rate (< 0.01%)

**Concerning**:
- Sublinear scaling (< 0.7x per core): Lock contention or shared state
- Low CPU utilization (< 60%) at saturation: I/O bound or blocking operations
- High error rate (> 1%): Connection failures, timeouts, or resource exhaustion

### Scalability Analysis

**Good**:
- Graceful degradation with connection count
- No cliff drop-off (sudden performance collapse)
- Memory growth proportional to connections

**Concerning**:
- Sharp performance drop at specific connection count: Hitting system limits (FD limit, buffer sizes)
- Exponential memory growth: Memory leaks
- High error rate with many connections: Backpressure not working

---

## Best Practices

### Do's

✅ **Always warmup**: Stabilize CPU frequency and caches
✅ **Use steady_clock**: Monotonic time source for measurements
✅ **Sample strategically**: Balance detail vs overhead
✅ **Sort before percentiles**: Required for correct calculation
✅ **Document environment**: Enable reproducibility
✅ **Run multiple times**: Establish baseline variance
✅ **Monitor system**: Watch CPU, memory, temperature

### Don'ts

❌ **Don't use system_clock**: Affected by NTP adjustments
❌ **Don't sample every operation**: Measurement overhead skews results
❌ **Don't ignore outliers**: They indicate real issues
❌ **Don't compare across hardware**: Results not meaningful
❌ **Don't benchmark debug builds**: Not representative of production
❌ **Don't run with other workloads**: Interference affects results
❌ **Don't trust single runs**: Variance matters

---

## Future Enhancements

Planned improvements to benchmark methodology:

- [ ] Automated variance detection and outlier rejection
- [ ] Flamegraph generation for bottleneck identification
- [ ] Continuous benchmarking with historical tracking
- [ ] Automated regression detection (> 10% degradation fails CI)
- [ ] Per-component performance budgets
- [ ] Benchmark profile presets (JSON configuration files)
- [ ] Comparison mode (before/after analysis)
- [ ] Distribution plotting (latency histograms)
- [ ] Integration with external benchmark tools (Criterion, Google Benchmark)

---

## References

- [ARCHITECTURE.md](../ARCHITECTURE.md) - Framework design principles
- [PERFORMANCE_GUIDE.md](PERFORMANCE_GUIDE.md) - Tuning guide
- [BENCHMARK_RESULTS.md](../BENCHMARK_RESULTS.md) - Latest benchmark results

---

**Last Updated**: 2025-12-08
**Version**: 1.0 (Stage 2 Revision)
