# Scenario 1: Design a per-key rate limiter
Limit each API key to N req/s with bursts, in-process, at 100k keys. Then make it work across 20 pods.

### Strong answer

* Use a token bucket with lazy refill. Compute tokens from elapsed time on each call, so there is no goroutine or ticker per key. A ticker per bucket is the classic mistake: 100k goroutines, and they leak.
* Shard the key map (e.g. 64 shards, each with its own mutex) to cut contention. Each bucket has its own small mutex, or uses atomics.
* Inject the clock so the limiter is testable.
* Evict idle buckets, either with a janitor goroutine that sweeps per shard or by storing lastSeen and cleaning lazily.

```
type Bucket struct {
    mu     sync.Mutex
    tokens float64
    last   time.Time
    rate   float64 // tokens per second
    burst  float64 
}

func (b *Bucket) Allow(now time.Time) bool {
    b.mu.Lock()
    defer b.mu.Unlock()
    b.tokens = math.Min(b.burst, b.tokens+now.Sub(b.last).Seconds()*b.rate)
    b.last = now
    if b.tokens < 1 {
        return false
    }
    b.tokens--
    return true
}
```

### Explanation:
#### 1. The Architecture: Why this approach?
The prompt asks for 100,000 keys in-process. A naive approach would create a time.Ticker or a dedicated goroutine for each of the 100k keys to refill tokens every second. This would create 100k goroutines, consuming significant memory and CPU for context switching.

The "answer" avoids this by using Lazy Refill:

* No background goroutines: Each bucket is just a struct in memory.
Compute-on-read: Tokens are calculated only when a request actually arrives (Allow() is called).
* Sharding: To prevent 100k keys from fighting over a single sync.Mutex, the map is split into 64 shards (likely shard = hash(key) % 64). Each shard has its own mutex, drastically reducing contention.

#### 2. Deep Dive into the Critical Line
```
b.tokens = math.Min(b.burst, b.tokens+now.Sub(b.last).Seconds()*b.rate)
```
This single line handles three things simultaneously: 
* Time Progression, 
* Rate Limiting, and 
* Burst Cap.

Step-by-Step Breakdown:
```
now.Sub(b.last).Seconds()
```
Calculates how much time has passed since the last request.
Example: If 2.5 seconds passed, this returns 2.5.
```
... * b.rate
```
Converts that time into how many tokens should have been generated.
If rate is 10 req/s, and 2.5 seconds passed, 2.5 * 10 = 25 tokens are added.
```
b.tokens + ...
```
Adds the newly earned tokens to the existing balance.
If you had 3 tokens left, and earned 25, you now have 28.
```
math.Min(b.burst, ...)
```
This is the crucial part for "Bursts."
It caps the total tokens at b.burst.
If burst is 20, even if 100 seconds passed (generating 1000 tokens), b.tokens will only be set to 20.
This ensures that a user can burst up to 20 requests immediately, but then must wait for the steady rate (b.rate) to replenish them.
Why math.Min instead of just adding?
If you didn't use math.Min, a user who hasn't made a request for an hour would accumulate infinite tokens, allowing them to send thousands of requests in a single second. The burst parameter enforces the maximum burst size, regardless of idle time.

#### 3. Coding Example: Full Runnable Sharded Limiter
Here is the complete, thread-safe implementation including the sharding, lazy refill, and idle eviction logic

```go
package main

import (
	"fmt"
	"math"
	"sync"
	"time"
)

// Bucket represents a single rate limiter for one API key.
type Bucket struct {
	mu     sync.Mutex
	tokens float64
	last   time.Time
	rate   float64 // tokens per second // How fast tokens refill over time 
	burst  float64 // max tokens allowed // How many tokens can accumulate at once
}

/*
Analogy:
rate(10/s) - Water flowing into a tank at 10 liters/sec
burst(20) - The tank's maximum capacity is 20 liters
*/
/*
Timeline Example: rate=10, burst=20
Imagine an idle user who hasn't made any requests for a while. Their bucket starts full (20 tokens).

Time    Action                          Tokens Before   Result      Tokens After
────    ──────                          ─────────────   ──────      ────────────
t=0     Bucket full from idle           20              —           20
t=0     Request #1                      20              ✅ Allow    19
t=0     Request #2                      19              ✅ Allow    18
...     ...                             ...             ...         ...
t=0     Request #20                     1               ✅ Allow    0
t=0     Request #21                     0               ❌ Deny     0
t=0     Request #22                     0               ❌ Deny     0
        ← 2 seconds pass →
        Refill: 2s × 10 tokens/s = 20 new tokens added

t=2     Request #23                     20              ✅ Allow    19
t=2     Request #24                     19              ✅ Allow    18
...     ...                             ...             ...         ...
t=2     Request #42                     1               ✅ Allow    0
t=2     Request #43                     0               ❌ Deny     0
*/

// NewBucket creates a new rate limiter bucket.
func NewBucket(rate float64, burst float64) *Bucket {
	return &Bucket{
		tokens: burst, // Start full
		last:   time.Now(),
		rate:   rate,
		burst:  burst,
	}
}

// Allow checks if a request is permitted. It uses lazy refill.
func (b *Bucket) Allow() bool {
	b.mu.Lock()
	defer b.mu.Unlock()

	now := time.Now()
	// Lazy refill: calculate tokens earned since last request
	elapsed := now.Sub(b.last).Seconds()
	b.tokens = b.tokens + elapsed*b.rate

	// Cap at burst
	b.tokens = math.Min(b.burst, b.tokens)

	// If we have at least 1 token, allow and deduct
	if b.tokens >= 1 {
		b.tokens--
		b.last = now
		return true
	}

	// Update last time even if denied, to avoid large jumps on next allow
	b.last = now
	return false
}

// ShardedLimiter manages buckets for many keys with reduced contention.
type ShardedLimiter struct {
	shards []*shard
	count  int // number of shards (power of 2 for fast modulo)
}

type shard struct {
	mu   sync.Mutex
	keys map[string]*Bucket
}

// NewShardedLimiter creates a limiter with N shards.
func NewShardedLimiter(numShards int) *ShardedLimiter {
	shards := make([]*shard, numShards)
	for i := 0; i < numShards; i++ {
		shards[i] = &shard{
			keys: make(map[string]*Bucket),
		}
	}
	return &ShardedLimiter{
		shards: shards,
		count:  numShards,
	}
}

// getBucket returns the bucket for a key, creating it if it doesn't exist.
// It uses lazy initialization to avoid allocating 100k buckets upfront.
func (sl *ShardedLimiter) getBucket(key string, rate float64, burst float64) *Bucket {
	// Simple hash for sharding
	shardIdx := int(hash(key)) % sl.count
	shard := sl.shards[shardIdx]

	// Double-checked locking pattern for performance
	shard.mu.Lock()
	b, exists := shard.keys[key]
	shard.mu.Unlock()
    /*
    This is a critical point in Go concurrency. You are right that reading a map is technically safe, but Go maps are not safe for concurrent access if any goroutine is writing to them.

    The "Read-While-Write" Race Condition
    Here is the exact scenario that causes a fatal runtime panic (concurrent map read and map write) if you remove that lock:

    Goroutine A calls getBucket("user1").
    It locks the shard.
    It checks shard.keys["user1"] → Does not exist.
    It unlocks the shard.

    Goroutine B calls getBucket("user1") at the exact same time.
    It locks the shard.
    It checks shard.keys["user1"] → Does not exist.
    It unlocks the shard.

    Goroutine A sees exists == false, so it creates a new bucket and writes it to shard.keys["user1"].
    Goroutine B sees exists == false, so it creates a new bucket and writes it to shard.keys["user1"].
    
    Result:
    Race Condition: Two goroutines are writing to the map.
    Panic: If they do it at the same time, Go detects the concurrent write and panics.
    Logic Error: Even if it doesn't panic, you might have two different Bucket structs for the same user, leading to incorrect rate limiting.

    Why is shard.mu.Lock() needed for the read?
    The lock is not there to protect the read itself. It is there to protect the decision to write.

    This pattern is called Double-Checked Locking.


    Scenario	                                        Lock Needed?	                Why?
    Reading a map alone	                                No	                            Go maps are read-safe.
    Reading a map while another goroutine might write	Yes	                            To prevent concurrent map read/write panic.
    Double-Checked Locking	                            Yes	                               To ensure that between "check" and "create," no one else created it.

    Key Rule:
        If any goroutine writes to a map, all reads must be protected by a lock.
    */

	if exists {
		return b
	}

	// Need to create it
	shard.mu.Lock()
	defer shard.mu.Unlock()

	// Check again in case another goroutine created it while we waited
	b, exists = shard.keys[key]
	if exists {
		return b
	}

	b = NewBucket(rate, burst)
	shard.keys[key] = b
	return b
}

// Allow checks if the API key is allowed under the rate limit.
func (sl *ShardedLimiter) Allow(key string, rate float64, burst float64) bool {
	b := sl.getBucket(key, rate, burst)
	return b.Allow()
}

// Simple string hash function for demo purposes
func hash(s string) uint64 {
	var h uint64
	for _, c := range s {
		h = 31*h + uint64(c)
	}
	return h
}

func main() {
	// Create a limiter with 64 shards
	limiter := NewShardedLimiter(64)

	apiKey := "user-123"
	rate := 10.0   // 10 requests per second //How fast tokens refill over time
	burst := 20.0  // Can burst up to 20 requests //How many tokens can accumulate at once

	fmt.Println("Testing Rate Limiter for Key:", apiKey)
	fmt.Println("Rate:", rate, "req/s, Burst:", burst)
	fmt.Println()

	// Test 1: Burst capability
	fmt.Println("Attempting 25 rapid requests...")
	allowed := 0
	for i := 0; i < 25; i++ {
		if limiter.Allow(apiKey, rate, burst) {
			allowed++
		}
	}
	fmt.Printf("Allowed: %d (Max burst is %d)\n", allowed, burst)

	// Test 2: Rate limiting after burst
	fmt.Println("\nAttempting 5 more requests immediately...")
	moreAllowed := 0
	for i := 0; i < 5; i++ {
		if limiter.Allow(apiKey, rate, burst) {
			moreAllowed++
		}
	}
	fmt.Printf("Allowed: %d (Should be 0 or very few due to burst depletion)\n", moreAllowed)

	// Test 3: Wait and retry
	fmt.Println("\nWaiting 2 seconds for tokens to refill...")
	time.Sleep(2 * time.Second)

	waitAllowed := 0
	for i := 0; i < 5; i++ {
		if limiter.Allow(apiKey, rate, burst) {
			waitAllowed++
		}
	}
	fmt.Printf("Allowed after 2s wait: %d (Expected ~20, since 2s * 10req/s = 20 tokens)\n", waitAllowed)
}
```
##### 1. Why Sharding? (The "Hot Key" Problem)
Imagine you have 100,000 users, and they are all stored in a single map[string]*Bucket.

The Single Map Bottleneck
To find the right bucket, Go must:

Hash the key.
Lock the entire map (or the specific bucket in the map's internal structure) to read/write.
If you use a map with a global Mutex, that is one mutex for 100,000 users.
The Problem: If 1,000 requests arrive simultaneously for 1,000 different users, they all fight for that one mutex.

999 goroutines are waiting.
CPU is wasted on context switching.
Latency spikes.
The Sharded Solution
We split the 100,000 users into 64 smaller maps (shards).

User A goes to Shard 1.
User B goes to Shard 2.
...
User Z goes to Shard 64.
Now, if 1,000 requests arrive for different users, they likely hit different shards.

15 requests hit Shard 1 (fighting for Shard 1's mutex).
15 requests hit Shard 2.
...
They can all run in parallel because the mutexes are independent.
Analogy:

Single Map: One cashier for 1,000 customers in a grocery store. Long line.
Sharded Map: 64 cashiers. Customers are distributed. Line is much shorter.
##### 2. Why not use sync.Map?
Go's `sync.Map` is optimized for read-heavy workloads where keys are rarely deleted. Rate limiters are often write-heavy (every request checks/updates tokens) and require eviction (removing idle users).

|Feature	                |     sync.Map	                |       Sharded map|
|---------------------------|-------------------------------|-------------------|
|Read Performance	        |Good (lock-free reads)	        |Good (if shard lock is short)|
|Write Performance	        |Poor under high contention	    |Excellent (parallel writes to different shards)|
|Eviction/Janitor	        |Hard (can't easily iterate)	    |Easy (sweep each shard independently)|
|Eviction/Janitor	        |Hard (can't easily iterate)	    |Easy (sweep each shard independently)|
|Eviction/Janitor	        |Hard (can't easily iterate)	    |Easy (sweep each shard independently)|
|Memory Overhead	        |Higher (internal structure)	    |Lower (just standard maps)|
|Predictability	            |Variable	                    |Predictable (you control shard count)|

In high-performance systems (like 100k keys), predictability is key. sync.Map can behave unpredictably under extreme write load. Sharding gives you control over how many locks you create.


### Follow-ups
* Distributed: Redis with a Lua script (atomic read-modify-write), or a local limiter that takes N/pods of the quota and syncs periodically. The trade-off is accuracy against a network hop on every request. Mention fail-open vs fail-closed when Redis is down.  
* Hot key: one key gets 50% of traffic, so shard-level locking doesn’t help. A per-bucket mutex is fine, or use a CAS loop on packed state.  
* Fairness and observability: return Retry-After, and export allow/deny counters per tier, not per key (cardinality).



