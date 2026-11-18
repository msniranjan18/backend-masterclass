# Profiling

Profiling in Go is the process of analyzing a program's runtime behavior—such as CPU usage, memory allocation, and goroutine blocking—to identify performance bottlenecks. While benchmarking tells you _how fast_ your code runs, profiling tells you _where_ your code is spending its time and _why_ it is consuming resources.

Go has profiling built directly into its runtime via the `runtime/pprof` package (for standalone tools) and `net/http/pprof` (for web servers).

Here is a deep dive into how profiling works, using a simple but resource-heavy example.

### 1. Types of Profiles in Go

Go can capture several types of profiles, but the two most common are:

* CPU Profile: Determines where the application is spending its time. It samples the call stack every 10 milliseconds to find functions occupying the CPU.
* Heap (Memory) Profile: Tracks memory allocation to help identify memory leaks and find where heavy heap allocations are occurring.

### 2. Practical Profiling Example

Let's write a program that simulates a CPU-heavy cryptographic operation (generating a password hash) and a memory-heavy operation (building an unnecessary slice in memory).

We will embed a local CPU and Memory profiler directly inside the `main` function.

#### Step 1: Write the Code (`main.go`)

Go

{% code lineNumbers="true" %}
```go
package main

import (
	"crypto/sha256"
	"fmt"
	"os"
	"runtime"
	"runtime/pprof"
)

// HeavyCPU simulates a CPU-bound operation
func HeavyCPU() {
	for i := 0; i < 100000; i++ {
		data := []byte(fmt.Sprintf("data-string-%d", i))
		_ = sha256.Sum256(data)
	}
}

// HeavyMemory simulates a memory-bound operation by allocating to the heap
func HeavyMemory() [][]byte {
	var leak [][]byte
	for i := 0; i < 50000; i++ {
		// Allocating 1KB slices repeatedly
		token := make([]byte, 1024)
		leak = append(leak, token)
	}
	return leak
}

func main() {
	// --- 1. SET UP CPU PROFILING ---
	cpuFile, err := os.Create("cpu.prof")
	if err != nil {
		fmt.Println("Could not create CPU profile:", err)
		return
	}
	defer cpuFile.Close()

	if err := pprof.StartCPUProfile(cpuFile); err != nil {
		fmt.Println("Could not start CPU profile:", err)
		return
	}
	defer pprof.StopCPUProfile() // Crucial to flush data to disk

	// Run our workload
	fmt.Println("Running heavy workloads...")
	HeavyCPU()
	_ = HeavyMemory()
	fmt.Println("Workloads complete!")

	// --- 2. SET UP MEMORY PROFILING ---
	memFile, err := os.Create("mem.prof")
	if err != nil {
		fmt.Println("Could not create memory profile:", err)
		return
	}
	defer memFile.Close()

	// Ensure memory stats are up to date
	runtime.GC() 
	if err := pprof.WriteHeapProfile(memFile); err != nil {
		fmt.Println("Could not write memory profile:", err)
		return
	}
}
```
{% endcode %}

### 3. Generating and Analyzing the Profiles

Run the program as you normally would. It will execute the workloads and cleanly spit out two binary files: `cpu.prof` and `mem.prof`.

Bash

```
go run main.go
```

Go includes a built-in analysis tool called `go tool pprof` to read these binary files. You can interact with it via the command line or view it visually in a web browser.

#### Scenario A: Analyzing CPU in the Terminal

Run the following command to open the interactive CPU profile terminal:

Bash

```
go tool pprof cpu.prof
```

You will enter an interactive prompt (`(pprof)`). Type `top` and press Enter.

Plaintext

```
(pprof) top
Showing nodes accounting for 190ms, 95.00% of 200ms total
      flat  flat%   sum%        cum   cum%
     110ms 55.00% 55.00%      110ms 55.00%  crypto/sha256.block
      40ms 20.00% 75.00%       40ms 20.00%  runtime.kevent
      20ms 10.00% 85.00%       30ms 15.00%  fmt.Fprintf
      20ms 10.00% 95.00%      160ms 80.00%  main.HeavyCPU
```

**How to read this terminal output:**

* `flat`: The time spent explicitly inside _this_ function.
* `cum` (Cumulative): The time spent in this function _plus_ any sub-functions it called down the stack.
* The Takeaway: `main.HeavyCPU` has a cumulative time of 80%, meaning it is responsible for the overwhelming majority of the application's CPU load, specifically driven by `crypto/sha256.block`.

You can dive deeper by typing `list HeavyCPU`, which will show you the exact line of code causing the bottleneck!

#### Scenario B: The Web UI (Recommended)

Reading text charts can be tedious. Go can generate graphical call graphs and flame graphs directly in your browser.

> _Note: For the visual call graph to work, you must have Graphviz installed on your machine (`brew install graphviz` on Mac or `sudo apt install graphviz` on Ubuntu)._

Run the profile command with the `-http` flag:

Bash

```
go tool pprof -http=:8080 mem.prof
```

This will automatically launch your web browser to `localhost:8080`.

**What to look for in the Web UI:**

1. The Graph View: This shows boxes representing functions. The larger and redder the box, the more memory or CPU it consumed. The arrows show the path of execution. You will instantly see a giant box pointing to `main.HeavyMemory`.
2. The Flame Graph (View -> Flame Graph): This stacks your functions vertically. The wider the bar, the more resources that function consumed. It allows you to visually drill down from `main` all the way to the internal runtime allocations.

### 4. Profiling Production Services (`net/http/pprof`)

If you are running a long-running web application, you don't want to manually start and stop files like we did above. Instead, Go allows you to safely expose a profiling endpoint.

All you have to do is import `net/http/pprof` (the blank import `_` initializes it) and start an HTTP server:

Go

```
package main

import (
	"net/http"
	_ "net/http/pprof" // Exposes endpoints automatically
)

func main() {
	// Your application logic goes here...

	// Start a separate routing port for internal metrics and profiling
	http.ListenAndServe("localhost:6060", nil)
}
```

When your production app is running under heavy load, you can pull a live 30-second snapshot of your production server's CPU profile straight to your local machine by running:

Bash

```
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30
```
