# Benchmarking & Profiling

### Benchmarking

Benchmarking is the practice of measuring your code's performance metrics (speed, throughput, memory consumption) under tightly controlled, repeatable laboratory conditions.

* The Goal: To answer the question: _"How fast is my code, and how much overall memory does it consume?"_
* The Loop Mechanics (`b.N`): Go manages the benchmark execution loop dynamically. It increases the loop iteration count exponentially ($$ $1 \rightarrow 100 \rightarrow 10,000 \dots$ $$) until the code runs continuously for a stable time frame (usually 1 second) to filter out background operating system noise.
* The Timing Rules: A strategic benchmark isolates the targeted logic by explicitly halting or resetting execution timers during heavy mock-data generation steps.

#### Essential Benchmarking Commands

Bash

```
# Run all benchmarks in the current directory (CPU execution metrics only)
go test -bench=.

# Run benchmarks and expose runtime memory tracking (B/op and allocs/op columns)
go test -bench=. -benchmem

# Filter execution to run ONLY benchmarks matching a specific string or regex pattern
go test -bench=ThreeSum -benchmem

# Skip standard TestXxx unit tests to jump straight into benchmark evaluations
go test -run=^$ -bench=. -benchmem

# Increase execution duration to gather a more stable, prolonged statistical sample
go test -bench=. -benchtime=5s -benchmem

# Force the loop to execute an exact block count instead of calculating across a time window
go test -bench=. -benchtime=5000x -benchmem

# Repeat the entire benchmarking sequence multiple times to monitor performance variance
go test -bench=. -count=5 -benchmem
```

### Profiling

Profiling is the practice of inspected debugging. It records exactly _where_ and _how_ your program is spending its clock cycles and resource allocations while it is running.

* The Goal: To answer the question: _"Which specific lines of code or internal functions are causing my performance bottlenecks?"_
* The Data Dumps (.pprof): Profiling works by generating compressed binary runtime logs (`.pprof` files) that capture thread stacks, CPU sampling intervals, and heap space allocation allocations.
* The Visual Layout Options: It leverages external rendering tools (like Graphviz) to draw comprehensive box-diagram call trees, horizontal flame charts, and line-by-line code cost annotations.

#### Essential Profiling Commands

Bash

```
# Execute benchmarks and dump binary CPU and Memory resource traces to files
go test -run=^$ -bench=. -cpuprofile=cpu.pprof -memprofile=mem.pprof

# Launch the interactive local browser interface to visualize and inspect your CPU profile
go tool pprof -http=:8080 cpu.pprof

# Launch the interactive local browser interface to visualize and inspect your Memory profile
go tool pprof -http=:8080 mem.pprof

# Launch pprof inside your native terminal window for text-based command-line query routing
go tool pprof cpu.pprof
```

### High-Frequency Core Metrics Summary

| **Metric**  | **Full Meaning**               | **Optimization Target**                                                                                     |
| ----------- | ------------------------------ | ----------------------------------------------------------------------------------------------------------- |
| `ns/op`     | Nanoseconds per Operation      | Lower is better. Measures raw CPU speed.                                                                    |
| `B/op`      | Bytes allocated per Operation  | Lower is better. Tracks your structural data memory weight.                                                 |
| `allocs/op` | Heap Allocations per Operation | Lower is better. High counts reveal heap-thrashing that triggers heavy background Garbage Collection pauses |
