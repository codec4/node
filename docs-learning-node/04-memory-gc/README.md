# Memory Management and Garbage Collection

## Overview

Understanding V8's memory management and garbage collection is crucial for building high-performance Node.js applications. This lesson covers how V8 organizes memory, performs garbage collection, and how Node.js developers can work effectively with these systems.

### The Generational Hypothesis

V8's GC design is fundamentally based on the **generational hypothesis**, an empirical observation about object lifetimes in programs:

> **Most objects die young**

Studies of real-world programs show that:

- 80-98% of objects become unreachable shortly after allocation
- Objects that survive initial collections tend to live much longer
- Age is a strong predictor of future lifetime

This insight drives V8's entire memory management strategy: optimize for the common case (short-lived objects) while handling the exceptions (long-lived objects) differently.

### Why GC Algorithm Choice Matters

Different garbage collection strategies have vastly different performance characteristics:

| Strategy           | Throughput | Pause Time        | Memory Overhead | Complexity |
| ------------------ | ---------- | ----------------- | --------------- | ---------- |
| Reference Counting | Low        | Low (incremental) | High            | Low        |
| Mark-Sweep         | High       | High              | Medium          | Medium     |
| Copying GC         | Very High  | Medium            | High (2x space) | Low        |
| Generational GC    | Very High  | Low-Medium        | Medium          | High       |
| Concurrent GC      | High       | Very Low          | Medium-High     | Very High  |

V8 combines multiple strategies to get the best of all worlds, as we'll explore in depth.

## Table of Contents

1. [Heap Organization](#heap-organization)
2. [Garbage Collection Algorithms](#garbage-collection-algorithms)
3. [Write Barriers and Concurrent Marking](#write-barriers-and-concurrent-marking)
4. [Memory Pressure in Node.js](#memory-pressure-in-nodejs)
5. [Heap Snapshots and Memory Leak Detection](#heap-snapshots-and-memory-leak-detection)
6. [Practical Examples](#practical-examples)
7. [Best Practices](#best-practices)

---

## Why V8 Uses These Specific GC Algorithms

### The Problem Space

Before diving into V8's solutions, let's understand the constraints:

1. **JavaScript is single-threaded**: Long GC pauses block execution
2. **Dynamic allocation patterns**: No compile-time lifetime analysis
3. **High allocation rate**: Modern JS frameworks create millions of objects
4. **Unpredictable object graphs**: Deep, interconnected object references
5. **Performance expectations**: Users expect 60fps (16.6ms per frame)

### Alternative GC Strategies (Not Used by V8)

#### 1. Reference Counting

**How it works**: Track how many pointers reference each object. When count reaches zero, free immediately.

```javascript
// Conceptual example
let obj = { data: 'value' }; // ref_count = 1
let ref2 = obj; // ref_count = 2
ref2 = null; // ref_count = 1
obj = null; // ref_count = 0 → immediate deallocation
```

**Why V8 doesn't use it**:

- ❌ **Circular references**: Cannot collect cycles
  ```javascript
  let a = {},
    b = {};
  a.ref = b;
  b.ref = a; // Both have ref_count ≥ 1 forever
  ```
- ❌ **High overhead**: Every assignment requires count updates
- ❌ **Poor cache locality**: Reference counts scattered across memory
- ✅ **Advantage**: Deterministic, immediate reclamation (used by Python, Swift for this reason)

#### 2. Mark-Compact Only (No Generational Collection)

**How it works**: Single heap, mark live objects, compact memory.

**Why V8 doesn't use only this**:

- ❌ **Poor for short-lived objects**: Wastes time marking objects that will die soon
- ❌ **Long pause times**: Must scan entire heap each collection
- ❌ **Inefficient for high allocation rates**: Modern JS creates ~1GB objects/sec

```javascript
// Example: React component rendering
function render() {
  // Each render creates thousands of temporary objects
  return (
    <div>
      {items.map((item) => (
        <Component key={item.id} data={item} />
      ))}
    </div>
  );
  // Most of these objects die immediately after render
}
```

#### 3. Stop-the-World Only (No Concurrent/Incremental)

**Why V8 doesn't use only this**:

- ❌ **Unacceptable pauses**: 100ms+ pauses on large heaps
- ❌ **Janky user experience**: Visible stuttering in animations
- ❌ **Doesn't scale**: Pause time grows with heap size

### V8's Hybrid Approach: Why It's Optimal

V8 combines **multiple algorithms**, each optimized for its specific use case:

```
┌─────────────────────────────────────────────────────────────┐
│                    V8's Multi-Algorithm GC                  │
├─────────────────────────────────────────────────────────────┤
│ Young Gen │ Scavenger (Copying)      │ Fast, assumes most │
│           │ + Parallel collection     │ objects die        │
├───────────┼──────────────────────────┼────────────────────┤
│ Old Gen   │ Mark-Sweep-Compact       │ Handles long-lived │
│ (Initial) │ + Concurrent marking      │ objects efficiently│
├───────────┼──────────────────────────┼────────────────────┤
│ Old Gen   │ Incremental marking      │ Splits work into   │
│ (Ongoing) │ + Write barriers          │ small chunks       │
├───────────┼──────────────────────────┼────────────────────┤
│ All Gens  │ Parallel evacuation      │ Uses multiple CPU  │
│           │ (Orinoco project)         │ cores              │
└─────────────────────────────────────────────────────────────┘
```

**Key Design Decisions**:

1. **Generational Collection**: Exploits generational hypothesis

   - Young gen: ~98% of objects die → optimize for this
   - Old gen: Rare collections for long-lived data

2. **Copying for Young Gen**: Accepts memory overhead for speed

   - Only live objects are touched
   - Automatic compaction (no fragmentation)
   - Cache-friendly sequential allocation

3. **Mark-Sweep for Old Gen**: Minimizes memory overhead

   - Old gen can be GB in size (2x overhead unacceptable)
   - Objects move less → better stability for native pointers

4. **Concurrent & Incremental**: Trades CPU for latency
   - Uses background threads for marking
   - Splits work into <5ms increments
   - Write barriers add ~5% overhead but enable concurrency

### Performance Impact of Algorithm Choice

```javascript
// Benchmark: Different GC strategies (hypothetical)
const iterations = 1000000;

// Scenario 1: High allocation rate (favors generational)
console.time('generational-gc');
for (let i = 0; i < iterations; i++) {
  const temp = { id: i, data: new Array(100).fill(i) };
  // 98% die immediately → young gen handles efficiently
}
console.timeEnd('generational-gc');
// Typical: 150ms with V8's generational GC
// Hypothetical with mark-sweep only: 800ms (5x slower)

// Scenario 2: Long-lived objects (favors mark-sweep)
const longLived = new Map();
console.time('mark-sweep-friendly');
for (let i = 0; i < iterations; i++) {
  longLived.set(i, { id: i, data: new Array(100).fill(i) });
  // All survive → eventually promoted to old gen
}
console.timeEnd('mark-sweep-friendly');
// Copying GC would be inefficient here (copying millions of objects)
// Mark-sweep only touches live objects during marking phase
```

### Evolution of V8's GC

V8's garbage collector has evolved significantly:

**2008-2010: Initial Design**

- Basic generational collection
- Stop-the-world for all operations
- Pause times: 50-100ms on large heaps

**2011-2015: Incremental Marking**

- Introduced incremental marking (Larry + Emmy)
- Reduced pause times to 10-20ms
- Write barriers for correctness

**2016-2018: Orinoco Project**

- **Concurrent marking**: Background threads mark objects
- **Parallel scavenging**: Multiple threads collect young gen
- **Parallel compaction**: Multiple threads compact old gen
- Pause times reduced to 1-5ms

**2019-Present: Continuous Optimization**

- Improved write barriers
- Better heuristics for triggering GC
- Pointer compression (saves 50% heap memory on 64-bit)
- Scavenger optimization for large young gen

```javascript
// Example: Pause time improvement over the years
// (Same application, 1GB heap)
const pauseTimes = {
  v8_2010: '100ms', // Stop-the-world
  v8_2015: '20ms', // Incremental marking
  v8_2018: '5ms', // Orinoco (concurrent)
  v8_2023: '2ms', // Latest optimizations
};

// For 60fps, we have 16.6ms per frame
// Modern V8 GC fits comfortably within frame budget
```

---

## Heap Organization

### V8 Heap Structure

V8 divides its heap into several spaces to optimize garbage collection:

```
┌─────────────────────────────────────────────┐
│              V8 Heap Layout                 │
├─────────────────────────────────────────────┤
│  New Space (Young Generation)               │
│  ├── Nursery (Semi-space 0)                 │
│  └── Intermediate (Semi-space 1)            │
├─────────────────────────────────────────────┤
│  Old Space (Old Generation)                 │
│  ├── Old Pointer Space                      │
│  └── Old Data Space                         │
├─────────────────────────────────────────────┤
│  Large Object Space                         │
├─────────────────────────────────────────────┤
│  Code Space                                 │
├─────────────────────────────────────────────┤
│  Map Space                                  │
└─────────────────────────────────────────────┘
```

### Young Generation (New Space)

**Size**: Typically 1-8MB (default ~16MB max on 64-bit)
**Purpose**: Short-lived objects
**Algorithm**: Scavenger (Cheney's algorithm)

New objects are allocated in the young generation. Most objects die young (the generational hypothesis), so this space is collected frequently.

```cpp
// V8 source: src/heap/heap.h
class Heap {
  // New space is divided into two semi-spaces
  NewSpace* new_space_;

  // Each semi-space uses bump-pointer allocation
  // Fast allocation: just increment a pointer
};
```

**Key Characteristics**:

- Fast allocation using bump-pointer technique
- Small size for quick collection
- Two semi-spaces for copying collection
- Objects surviving multiple collections are promoted to old generation

### Old Generation

**Size**: Significantly larger (can grow to heap limit)
**Purpose**: Long-lived objects
**Algorithm**: Mark-Sweep-Compact

Objects that survive multiple young generation collections are promoted here.

```cpp
// V8 source: src/heap/spaces.h
class OldSpace : public PagedSpace {
  // Contains objects that have survived GC cycles
  // Uses mark-sweep and mark-compact algorithms
};
```

**Spaces within Old Generation**:

1. **Old Pointer Space**: Objects containing pointers to other objects
2. **Old Data Space**: Objects containing only data (strings, numbers)
3. **Large Object Space**: Objects larger than the page size (~256KB)
4. **Code Space**: Compiled code objects
5. **Map Space**: Hidden class (Map) objects

### Node.js Memory Limits

```javascript
// Check current heap statistics
const v8 = require('v8');
const heapStats = v8.getHeapStatistics();

console.log('Heap Limit:', heapStats.heap_size_limit / (1024 * 1024), 'MB');
console.log(
  'Total Heap Size:',
  heapStats.total_heap_size / (1024 * 1024),
  'MB'
);
console.log('Used Heap Size:', heapStats.used_heap_size / (1024 * 1024), 'MB');
console.log(
  'Total Available:',
  heapStats.total_available_size / (1024 * 1024),
  'MB'
);
```

Default limits (64-bit):

- **Old Generation**: ~1.4GB (can be increased with `--max-old-space-size`)
- **Young Generation**: ~16MB (adjustable with `--max-semi-space-size`)

---

## Garbage Collection Algorithms

### 1. Scavenger (Minor GC)

Used for the young generation, implements Cheney's semi-space copying algorithm.

**Why Scavenger for Young Gen?**

1. **Optimal for high mortality rate**: Only touches live objects (2-20% of allocations)
2. **Fast allocation**: Bump-pointer allocation takes ~3 CPU cycles
3. **Automatic compaction**: No fragmentation, perfect for continuous allocation
4. **Cache-friendly**: Sequential memory access pattern

**Trade-off**: Uses 2x memory (two semi-spaces), but young gen is small (~16MB) so acceptable.

**Compared to alternatives**:

- **Mark-Sweep**: Would need to scan ALL objects (wasteful when 98% are dead)
- **Reference Counting**: Would add overhead to every object creation (billions/sec)

**How it works**:

```
Initial State:
┌──────────────┬──────────────┐
│  From Space  │   To Space   │
│  (active)    │   (empty)    │
│              │              │
│  Obj A ──┐   │              │
│  Obj B   │   │              │
│  Obj C ──┤   │              │
│  (dead)  │   │              │
└──────────┼───┴──────────────┘
           │
After Scavenge:
┌──────────────┬──────────────┐
│  From Space  │   To Space   │
│  (empty)     │  (active)    │
│              │              │
│              │  Obj A       │
│              │  Obj C       │
│              │              │
│              │              │
└──────────────┴──────────────┘
```

**Process**:

1. Start from roots (stack, global objects)
2. Copy live objects from "from-space" to "to-space"
3. Update all references
4. Swap the roles of the two spaces

**Advantages**:

- Very fast (typically < 1ms)
- No fragmentation
- Compacts memory automatically

**Implementation Note** (from V8 source):

```cpp
// deps/v8/src/heap/scavenger.cc
void Scavenger::ScavengePage(MemoryChunk* page) {
  // Iterate over all objects in the page
  // Copy live objects to to-space
  // Update pointers
}
```

### 2. Mark-Sweep (Major GC)

Used for the old generation when it reaches capacity.

**Why Mark-Sweep-Compact for Old Gen?**

1. **Memory efficient**: No 2x space overhead (old gen can be GB in size)
2. **Handles sparse liveness**: Many dead objects interspersed with live ones
3. **Stable addresses**: Objects don't move during marking (important for native code)
4. **Scales to large heaps**: Only touches reachable objects

**Why NOT copying for old gen?**

- Copying 1GB of long-lived objects would take seconds
- Would waste 1GB for to-space
- Most old gen objects survive (opposite of young gen)

**Mathematical justification**:

```
Let:
  L = number of live objects
  D = number of dead objects
  T = total objects = L + D

Copying GC cost: O(L)           - only copy live objects
Mark-Sweep cost: O(L) + O(T)    - mark live, sweep all

Young gen: L << D → Copying wins (L is tiny)
Old gen: L ≈ T → Mark-Sweep comparable, no 2x memory cost
```

**Three Phases**:

**Phase 1: Marking**

```
┌─────────────────────────────┐
│     Mark Live Objects       │
│                             │
│  Root → Obj A (marked)      │
│         └→ Obj B (marked)   │
│                             │
│  Obj C (unmarked = dead)    │
└─────────────────────────────┘
```

```cpp
// V8 marking process (simplified)
void MarkingVisitor::VisitPointers(Object** start, Object** end) {
  for (Object** p = start; p < end; p++) {
    Object* object = *p;
    if (!marking_state_.IsMarked(object)) {
      marking_state_.Mark(object);
      worklist_.Push(object);
    }
  }
}
```

**Phase 2: Sweeping**

```
┌─────────────────────────────┐
│    Sweep Dead Objects       │
│                             │
│  Obj A (kept)               │
│  Obj B (kept)               │
│  [free space where C was]   │
└─────────────────────────────┘
```

**Phase 3: Compaction** (when needed)

```
Before:
┌──────┬─────┬──────┬─────┐
│ Obj A│ free│ Obj B│ free│
└──────┴─────┴──────┴─────┘

After:
┌──────┬──────┬────────────┐
│ Obj A│ Obj B│    free    │
└──────┴──────┴────────────┘
```

### 3. Incremental Marking

To avoid long pause times, V8 performs marking incrementally.

```javascript
// Demonstration of GC pause impact
const start = Date.now();
const arr = [];

// Allocate lots of objects
for (let i = 0; i < 10000000; i++) {
  arr.push({ id: i, data: 'x'.repeat(100) });
}

console.log('Allocation time:', Date.now() - start, 'ms');
// GC will happen in small increments during allocation
```

**How Incremental Marking Works**:

1. **Black**: Marked and scanned
2. **Grey**: Marked but not yet scanned
3. **White**: Not marked (potentially dead)

```
Step 1:        Step 2:        Step 3:
┌─────┐       ┌─────┐       ┌─────┐
│Black│       │Black│       │Black│
│Grey │  →    │Black│  →    │Black│
│White│       │Grey │       │Black│
│White│       │White│       │     │
└─────┘       └─────┘       └─────┘
```

### 4. Concurrent Marking

V8 performs marking on background threads while JavaScript executes.

**Why Concurrent Marking?**

The problem:

- Marking a 1GB heap takes ~50ms
- This exceeds 60fps frame budget (16.6ms)
- Users experience visible jank

The solution:

- Do marking on background threads
- Main thread only pauses briefly to start/finish
- Requires write barriers for correctness

```cpp
// V8 source: deps/v8/src/heap/concurrent-marking.cc
void ConcurrentMarking::Run(JobDelegate* delegate) {
  // Runs on background thread
  while (!marking_worklist_.IsEmpty()) {
    Object obj = marking_worklist_.Pop();
    MarkObject(obj);
  }
}
```

**Performance Benefits**:

- Main thread pause time reduced by 60-70%
- Throughput increase of 5-15%
- Scales with CPU cores (4-8 cores commonly used)

**Cost**: Write barriers add 3-5% overhead to all pointer writes

### 5. Parallel Collection

**Why Parallel vs Concurrent?**

These terms are often confused:

- **Parallel**: Multiple threads do GC work simultaneously (app paused)
- **Concurrent**: GC threads work while app runs (app continues)

```
Parallel Scavenge:
┌─────────────────────────────────────┐
│ Main Thread: PAUSED                 │
├─────────────────────────────────────┤
│ GC Thread 1: ████████ (scanning)    │
│ GC Thread 2: ████████ (scanning)    │
│ GC Thread 3: ████████ (scanning)    │
│ GC Thread 4: ████████ (scanning)    │
└─────────────────────────────────────┘
Pause: 2ms (4x faster than single-threaded)

Concurrent Marking:
┌─────────────────────────────────────┐
│ Main Thread: ████████ (running JS)  │
├─────────────────────────────────────┤
│ GC Thread 1: ████████ (marking)     │
│ GC Thread 2: ████████ (marking)     │
└─────────────────────────────────────┘
Pause: ~0.5ms (just to start/finish)
```

**V8 uses both**:

1. **Parallel Scavenge**: Young gen collection uses all cores

   - Reduces pause from 8ms to 2ms on 4 cores
   - Limited by Amdahl's law (some work must be sequential)

2. **Concurrent Marking**: Old gen marking uses background threads

   - Reduces pause from 50ms to 5ms
   - Requires write barriers

3. **Parallel Compaction**: Old gen compaction uses all cores
   - Reduces compaction from 20ms to 5ms
   - Complex synchronization required

**Trade-offs**:

```javascript
// Parallel GC trade-offs
const tradeoffs = {
  advantages: [
    'Shorter pause times (better UX)',
    'Scales with CPU cores',
    'Better for large heaps',
  ],
  costs: [
    'Higher CPU usage (uses all cores)',
    'Complex implementation (synchronization)',
    'Memory overhead (per-thread structures)',
  ],
};

// V8's decision: Worth it for 60fps applications
// 8ms → 2ms pause means 60fps instead of 30fps
```

### 6. Comparison: V8 vs Other JavaScript Engines

Different JavaScript engines make different GC trade-offs:

#### SpiderMonkey (Firefox)

**Strategy**: Generational + Incremental + Concurrent

**Differences from V8**:

- **Nursery size**: Configurable, can be larger (32MB+)
- **Compacting**: Uses mostly non-moving GC (less copying)
- **Parallel GC**: More aggressive parallelization
- **Incremental slices**: Smaller, more frequent slices

```javascript
// SpiderMonkey GC API (not available in browsers)
// gc();                    // Trigger major GC
// gcslice(budget);         // Incremental GC slice
// schedulegc(n);           // Schedule GC in n allocations
```

**Philosophy**: Optimize for Firefox's workload (long-running tabs)

#### JavaScriptCore (Safari)

**Strategy**: Generational + Concurrent + DFG compiler integration

**Unique features**:

- **Eden + Survivor spaces**: More generations than V8
- **Concurrent JIT**: Compiles code while GC runs
- **SIMD scanning**: Uses vector instructions for marking
- **Butterfly allocation**: Optimized for property storage

**Philosophy**: Optimize for memory-constrained devices (iOS)

#### Comparison Table

| Feature                  | V8                 | SpiderMonkey                     | JavaScriptCore          |
| ------------------------ | ------------------ | -------------------------------- | ----------------------- |
| Generations              | 2 (young, old)     | 2-3 (nursery, tenured, optional) | 3 (eden, survivor, old) |
| Young gen algorithm      | Scavenger          | Copying                          | Copying                 |
| Old gen algorithm        | Mark-Sweep-Compact | Mostly Mark-Sweep                | Mark-Sweep              |
| Concurrent marking       | ✅ Yes             | ✅ Yes                           | ✅ Yes                  |
| Parallel evacuation      | ✅ Yes             | ✅ Yes                           | ⚠️ Limited              |
| Write barrier            | Standard           | Optimized for large objects      | Optimized for iOS       |
| Typical pause (1GB heap) | 2-5ms              | 3-7ms                            | 1-3ms                   |
| Memory overhead          | Medium             | Medium-High                      | Low                     |
| Throughput               | Very High          | High                             | High                    |

**Why these differences?**

1. **Target platform**:

   - V8: Server (Node.js) + Chrome (desktop/mobile)
   - SpiderMonkey: Desktop browser (long sessions)
   - JavaScriptCore: iOS (memory-constrained)

2. **Workload assumptions**:

   - V8: High allocation rate, performance-critical
   - SpiderMonkey: Multiple tabs, each with moderate allocation
   - JavaScriptCore: Battery life, memory footprint critical

3. **Engineering resources**:
   - V8: Large team, can afford complexity
   - SpiderMonkey: Focus on Firefox-specific optimizations
   - JavaScriptCore: Tight integration with Apple ecosystem

```javascript
// Example: Memory footprint differences
// Same application on different engines

const largeArray = new Array(1000000).fill({
  id: 0,
  data: new Array(100).fill('x'),
});

// Approximate memory usage:
// V8:             ~180MB (pointer compression, efficient layout)
// SpiderMonkey:   ~200MB (more metadata per object)
// JavaScriptCore: ~160MB (aggressive compression for iOS)
```

---

## Write Barriers and Concurrent Marking

### What are Write Barriers?

Write barriers are small pieces of code executed whenever a pointer is written to track object graph changes during concurrent/incremental GC.

```cpp
// Simplified write barrier concept
void WriteBarrier(Object** slot, Object* value) {
  *slot = value;  // Actual write

  // If we're in incremental marking and writing a white object
  // reference into a black object, we need to mark it grey
  if (InIncrementalMarking() &&
      IsBlack(*slot) &&
      IsWhite(value)) {
    MarkGrey(value);
  }
}
```

### Why Write Barriers are Needed

**Problem without write barriers**:

```
During incremental marking:

1. Object A is marked black (scanned)
2. JavaScript code: A.child = B (B is white/unmarked)
3. Continue marking...
4. B is never reached and incorrectly collected!
```

**Solution with write barriers**:

```
1. Object A is marked black
2. JavaScript code: A.child = B
   → Write barrier marks B as grey
3. B will be scanned before marking completes
4. B is correctly kept
```

### Types of Write Barriers in V8

**1. Generational Barrier** (for young → old references):

```cpp
// V8 source reference
if (object_is_in_old_generation &&
    value_is_in_young_generation) {
  RememberedSet::Insert(object_address);
}
```

When an old object points to a young object, record it in the "remembered set" so young generation GC can find it.

**2. Incremental Marking Barrier**:

```cpp
if (incremental_marking_in_progress &&
    object_is_black &&
    value_is_white) {
  MarkGrey(value);
}
```

### Performance Impact

Write barriers add overhead to every object property write:

```javascript
// Each property assignment triggers a write barrier
const obj = { child: null };
obj.child = { data: 'value' }; // Write barrier executed here

// Measure the overhead
const iterations = 10000000;

// Without many writes
console.time('read-heavy');
for (let i = 0; i < iterations; i++) {
  const x = obj.child;
}
console.timeEnd('read-heavy');

// With many writes
console.time('write-heavy');
for (let i = 0; i < iterations; i++) {
  obj.child = { data: i };
}
console.timeEnd('write-heavy');
// write-heavy will be noticeably slower
```

---

## Allocation Strategies and Fast Paths

### Why Allocation Speed Matters

Modern JavaScript applications allocate objects at extraordinary rates:

```javascript
// React rendering example
function App() {
  const [items, setItems] = useState(Array.from({ length: 1000 }));

  return (
    <div>
      {items.map((item, i) => (
        <Card key={i} data={item} /> // Creates ~1000 objects per render
      ))}
    </div>
  );
}

// At 60fps: 1000 objects × 60 frames = 60,000 objects/second
// In reality: 10-100x more (virtual DOM, closures, etc.)
// Total: 1-10 million allocations/second is common!
```

### V8's Allocation Fast Path

V8 optimizes allocation to be extremely fast:

```cpp
// Simplified V8 fast allocation (pseudocode)
Object* Allocate(size_t size) {
  // Fast path: bump pointer allocation (3-5 CPU cycles)
  if (new_space_top + size <= new_space_limit) {
    Object* result = new_space_top;
    new_space_top += size;
    return result;  // No locks, no complex logic
  }

  // Slow path: need to GC or switch spaces
  return SlowAllocate(size);
}
```

**Why bump-pointer is fast**:

1. **No search**: Don't need to find free space
2. **No fragmentation**: All space is contiguous
3. **Cache-friendly**: Sequential memory access
4. **No locks**: Thread-local allocation buffers

**Benchmark**:

```javascript
// Allocation speed comparison
const iterations = 10000000;

console.time('allocate');
for (let i = 0; i < iterations; i++) {
  const obj = { id: i }; // ~10-15ns per allocation on modern CPU
}
console.timeEnd('allocate');

// 10M allocations in ~100-150ms
// = 66-100 million allocations/second per core!
```

### Allocation in Different Spaces

```javascript
// V8 allocation decision tree
function allocate(object) {
  const size = sizeOf(object);

  if (size > 256 * 1024) {
    // Large Object Space (LOS)
    // - Direct OS memory allocation
    // - Not moved during GC
    // - Individually freed
    return allocateLarge(object);
  }

  if (isCode(object)) {
    // Code Space
    // - Executable pages
    // - Special permissions
    return allocateCode(object);
  }

  if (isMap(object)) {
    // Map Space (hidden classes)
    // - Frequently accessed
    // - Separate for cache optimization
    return allocateMap(object);
  }

  // Normal allocation
  if (youngGenHasSpace()) {
    return allocateYoung(object); // Fast path
  } else {
    collectYoungGen(); // Minor GC
    return allocateYoung(object);
  }
}
```

### Inline Caches and Hidden Classes

V8's allocation is intertwined with optimization:

```javascript
// Hidden classes (Maps) make property access fast
class Point {
  constructor(x, y) {
    this.x = x; // Assigns hidden class "Map1"
    this.y = y; // Transitions to "Map2"
  }
}

// All Point instances share the same hidden class
const p1 = new Point(1, 2); // Uses Map2
const p2 = new Point(3, 4); // Uses Map2 (cached)

// Property access becomes:
// obj.x → load from offset 0 (Map2 tells us)
// vs hash table lookup in dynamic languages
```

**Why this matters for GC**:

- Hidden classes (Maps) stored in separate Map Space
- Frequently accessed, kept in old gen
- Sharing maps reduces memory pressure

### Optimization: Object Pooling

For hot paths, you can beat V8's allocator:

```javascript
// V8's allocator is fast, but not free
// Pooling can help in extreme cases

class Vector3Pool {
  constructor(size = 1000) {
    this.pool = [];
    for (let i = 0; i < size; i++) {
      this.pool.push({ x: 0, y: 0, z: 0 });
    }
  }

  acquire() {
    return this.pool.pop() || { x: 0, y: 0, z: 0 };
  }

  release(v) {
    v.x = v.y = v.z = 0;
    this.pool.push(v);
  }
}

// Benchmark: 3D math library
const pool = new Vector3Pool();

console.time('with-pooling');
for (let i = 0; i < 1000000; i++) {
  const v = pool.acquire();
  v.x = i;
  v.y = i * 2;
  v.z = i * 3;
  // Use vector...
  pool.release(v);
}
console.timeEnd('with-pooling');
// ~50-70ms

console.time('without-pooling');
for (let i = 0; i < 1000000; i++) {
  const v = { x: i, y: i * 2, z: i * 3 };
  // Use vector...
  // GC will collect later
}
console.timeEnd('without-pooling');
// ~100-150ms (includes GC time)

// Pooling wins for: game loops, real-time graphics, high-frequency trading
// V8 allocator wins for: normal applications (simpler code)
```

---

## Memory Pressure in Node.js

### Understanding Memory Pressure

Memory pressure occurs when the application approaches the heap limit, triggering more frequent garbage collection.

### Monitoring Memory Pressure

```javascript
const v8 = require('v8');

function checkMemoryPressure() {
  const stats = v8.getHeapStatistics();
  const usedPercent = (stats.used_heap_size / stats.heap_size_limit) * 100;

  console.log(`Heap Usage: ${usedPercent.toFixed(2)}%`);

  if (usedPercent > 90) {
    console.warn('⚠️  High memory pressure!');
  } else if (usedPercent > 75) {
    console.warn('⚠️  Moderate memory pressure');
  }

  return usedPercent;
}

// Monitor periodically
setInterval(checkMemoryPressure, 5000);
```

### GC Events and Performance

```javascript
const { performance, PerformanceObserver } = require('perf_hooks');

// Observe GC events
const obs = new PerformanceObserver((list) => {
  const entries = list.getEntries();
  entries.forEach((entry) => {
    console.log(`GC ${entry.kind}: ${entry.duration.toFixed(2)}ms`);
    // kind can be: 'major', 'minor', 'incremental', 'weakcb'
  });
});

obs.observe({ entryTypes: ['gc'], buffered: true });

// Trigger some allocations
const arrays = [];
for (let i = 0; i < 100; i++) {
  arrays.push(new Array(100000).fill('data'));
}
```

### Memory Pressure Callbacks

Node.js doesn't expose direct memory pressure callbacks, but you can monitor:

```javascript
const v8 = require('v8');

class MemoryMonitor {
  constructor(threshold = 0.8) {
    this.threshold = threshold;
    this.lastCheck = Date.now();
  }

  check() {
    const stats = v8.getHeapStatistics();
    const ratio = stats.used_heap_size / stats.heap_size_limit;

    if (ratio > this.threshold) {
      this.onHighMemory(ratio, stats);
    }

    return ratio;
  }

  onHighMemory(ratio, stats) {
    console.warn(`High memory usage: ${(ratio * 100).toFixed(2)}%`);

    // Potential actions:
    // 1. Clear caches
    // 2. Pause non-critical work
    // 3. Alert monitoring systems
    // 4. Trigger manual GC if appropriate

    if (global.gc) {
      console.log('Triggering manual GC...');
      global.gc();
    }
  }

  startMonitoring(interval = 10000) {
    this.timer = setInterval(() => this.check(), interval);
  }

  stopMonitoring() {
    if (this.timer) clearInterval(this.timer);
  }
}

// Usage
const monitor = new MemoryMonitor(0.75);
monitor.startMonitoring(5000);
```

### Increasing Heap Limits

```bash
# Increase old space size to 4GB
node --max-old-space-size=4096 app.js

# Increase young generation size
node --max-semi-space-size=16 app.js

# Expose GC to JavaScript (for testing only!)
node --expose-gc app.js
```

---

## Heap Snapshots and Memory Leak Detection

### Taking Heap Snapshots

#### Method 1: Using v8.writeHeapSnapshot()

```javascript
const v8 = require('v8');
const fs = require('fs');

function takeSnapshot(filename) {
  const snapshotPath = v8.writeHeapSnapshot(filename);
  console.log(`Snapshot written to: ${snapshotPath}`);
  return snapshotPath;
}

// Take snapshot at specific points
takeSnapshot('./before-operation.heapsnapshot');

// Perform memory-intensive operation
const leakyArray = [];
for (let i = 0; i < 1000000; i++) {
  leakyArray.push({ id: i, data: 'x'.repeat(100) });
}

takeSnapshot('./after-operation.heapsnapshot');
```

#### Method 2: Using Inspector Protocol

```javascript
const inspector = require('inspector');
const fs = require('fs');

function takeHeapSnapshot(path) {
  const session = new inspector.Session();
  session.connect();

  return new Promise((resolve, reject) => {
    const stream = fs.createWriteStream(path);

    session.on('HeapProfiler.addHeapSnapshotChunk', (m) => {
      stream.write(m.params.chunk);
    });

    session.post('HeapProfiler.takeHeapSnapshot', null, (err) => {
      session.disconnect();
      stream.end();

      if (err) reject(err);
      else resolve(path);
    });
  });
}

// Usage
takeHeapSnapshot('./heap.heapsnapshot').then((path) =>
  console.log('Snapshot saved:', path)
);
```

### Analyzing Heap Snapshots

Load snapshots in Chrome DevTools:

1. Open Chrome DevTools
2. Go to Memory tab
3. Load snapshot file
4. Analyze:
   - **Summary**: View by constructor
   - **Comparison**: Compare two snapshots
   - **Containment**: View object retention tree
   - **Statistics**: Memory distribution

### Common Memory Leak Patterns

#### 1. Forgotten Timers

```javascript
// BAD: Memory leak
class LeakyTimer {
  constructor() {
    this.data = new Array(10000).fill('data');

    setInterval(() => {
      console.log(this.data.length);
    }, 1000);
    // Timer keeps reference to 'this', preventing GC
  }
}

const instances = [];
for (let i = 0; i < 100; i++) {
  instances[i] = new LeakyTimer();
  instances[i] = null; // Instance can't be collected!
}

// GOOD: Clean up timers
class CleanTimer {
  constructor() {
    this.data = new Array(10000).fill('data');

    this.timerId = setInterval(() => {
      console.log(this.data.length);
    }, 1000);
  }

  cleanup() {
    clearInterval(this.timerId);
  }
}
```

#### 2. Event Listener Leaks

```javascript
const EventEmitter = require('events');

// BAD: Listeners not removed
class LeakySubscriber {
  constructor(emitter) {
    this.data = new Array(10000).fill('data');

    emitter.on('event', () => {
      console.log(this.data.length);
    });
    // Listener keeps reference to 'this'
  }
}

// GOOD: Remove listeners
class CleanSubscriber {
  constructor(emitter) {
    this.emitter = emitter;
    this.data = new Array(10000).fill('data');

    this.handler = () => {
      console.log(this.data.length);
    };

    emitter.on('event', this.handler);
  }

  cleanup() {
    this.emitter.off('event', this.handler);
  }
}
```

#### 3. Closure Leaks

```javascript
// BAD: Closure captures large context
function createHandlers() {
  const hugeArray = new Array(1000000).fill('data');

  return {
    small: () => 'small',
    // Even though this doesn't use hugeArray,
    // the closure captures the entire scope
  };
}

const handlers = [];
for (let i = 0; i < 100; i++) {
  handlers.push(createHandlers());
}

// GOOD: Minimize closure scope
function createHandlersFixed() {
  const hugeArray = new Array(1000000).fill('data');

  // Process data and release reference
  const result = hugeArray.length;

  return {
    small: () => result, // Only captures 'result', not 'hugeArray'
  };
}
```

#### 4. Global Variables

```javascript
// BAD: Accidental globals
function processData() {
  leakyGlobal = new Array(10000).fill('data'); // No var/let/const!
}

// GOOD: Explicit scoping
function processDataFixed() {
  const localData = new Array(10000).fill('data');
  // Will be collected when function returns
}
```

### Memory Leak Detection Tool

```javascript
const v8 = require('v8');

class MemoryLeakDetector {
  constructor() {
    this.snapshots = [];
  }

  async takeSnapshot(label) {
    const stats = v8.getHeapStatistics();

    this.snapshots.push({
      label,
      timestamp: Date.now(),
      heapUsed: stats.used_heap_size,
      heapTotal: stats.total_heap_size,
      external: stats.external_memory,
    });

    console.log(
      `Snapshot '${label}': ${(stats.used_heap_size / 1024 / 1024).toFixed(
        2
      )} MB`
    );
  }

  analyze() {
    if (this.snapshots.length < 2) {
      console.log('Need at least 2 snapshots to compare');
      return;
    }

    console.log('\n=== Memory Growth Analysis ===');

    for (let i = 1; i < this.snapshots.length; i++) {
      const prev = this.snapshots[i - 1];
      const curr = this.snapshots[i];

      const growth = curr.heapUsed - prev.heapUsed;
      const growthMB = growth / 1024 / 1024;
      const growthPercent = (growth / prev.heapUsed) * 100;

      console.log(`\n${prev.label} → ${curr.label}:`);
      console.log(
        `  Growth: ${growthMB.toFixed(2)} MB (${growthPercent.toFixed(2)}%)`
      );

      if (growthPercent > 10) {
        console.warn('  ⚠️  Significant memory increase detected!');
      }
    }
  }

  reset() {
    this.snapshots = [];
  }
}

// Usage example
async function detectLeaks() {
  const detector = new MemoryLeakDetector();

  detector.takeSnapshot('baseline');

  // Simulate workload
  for (let i = 0; i < 5; i++) {
    // Your application logic here
    const temp = new Array(100000).fill('data');
    await new Promise((resolve) => setTimeout(resolve, 1000));

    detector.takeSnapshot(`iteration-${i + 1}`);
  }

  detector.analyze();
}

// Run with: node --expose-gc script.js
if (global.gc) {
  detectLeaks();
}
```

---

## Practical Examples

### Example 1: Optimizing for Young Generation

```javascript
// INEFFICIENT: Objects promoted to old generation
class BadCache {
  constructor() {
    this.cache = new Map();
  }

  set(key, value) {
    // These objects live forever, end up in old gen
    this.cache.set(key, value);
  }
}

// EFFICIENT: Short-lived objects stay in young gen
class GoodCache {
  constructor(maxAge = 60000) {
    this.cache = new Map();
    this.maxAge = maxAge;
  }

  set(key, value) {
    this.cache.set(key, {
      value,
      timestamp: Date.now(),
    });
  }

  get(key) {
    const entry = this.cache.get(key);
    if (!entry) return undefined;

    if (Date.now() - entry.timestamp > this.maxAge) {
      this.cache.delete(key); // Allows GC
      return undefined;
    }

    return entry.value;
  }

  cleanup() {
    const now = Date.now();
    for (const [key, entry] of this.cache.entries()) {
      if (now - entry.timestamp > this.maxAge) {
        this.cache.delete(key);
      }
    }
  }
}

// Periodic cleanup to prevent promotion
const cache = new GoodCache();
setInterval(() => cache.cleanup(), 30000);
```

### Example 2: Reducing GC Pressure

```javascript
// BAD: Creates many temporary objects
function processDataBad(items) {
  return items
    .map((item) => ({ ...item, processed: true })) // New object
    .filter((item) => item.value > 0) // Array copy
    .map((item) => item.value * 2); // Array copy
}

// GOOD: Minimize allocations
function processDataGood(items) {
  const result = [];
  for (let i = 0; i < items.length; i++) {
    const item = items[i];
    if (item.value > 0) {
      result.push(item.value * 2); // Direct push, no intermediate objects
    }
  }
  return result;
}

// Benchmark
const data = Array.from({ length: 100000 }, (_, i) => ({
  id: i,
  value: i % 2 === 0 ? i : -i,
}));

console.time('bad');
processDataBad(data);
console.timeEnd('bad');

console.time('good');
processDataGood(data);
console.timeEnd('good');
```

### Example 3: Object Pooling

```javascript
// Reuse objects instead of creating new ones
class ObjectPool {
  constructor(factory, reset, initialSize = 10) {
    this.factory = factory;
    this.reset = reset;
    this.pool = [];

    // Pre-allocate
    for (let i = 0; i < initialSize; i++) {
      this.pool.push(factory());
    }
  }

  acquire() {
    return this.pool.pop() || this.factory();
  }

  release(obj) {
    this.reset(obj);
    this.pool.push(obj);
  }
}

// Example: Buffer pool
const bufferPool = new ObjectPool(
  () => Buffer.allocUnsafe(1024),
  (buf) => buf.fill(0),
  100
);

function processWithPool(data) {
  const buffer = bufferPool.acquire();
  try {
    // Use buffer
    buffer.write(data);
    return buffer.toString('base64');
  } finally {
    bufferPool.release(buffer);
  }
}

// vs creating new buffers each time
function processWithoutPool(data) {
  const buffer = Buffer.from(data); // New allocation every time
  return buffer.toString('base64');
}
```

### Example 4: Streaming for Large Data

```javascript
const fs = require('fs');
const { Transform } = require('stream');

// BAD: Loads entire file into memory
function processFileBad(inputPath, outputPath) {
  const data = fs.readFileSync(inputPath, 'utf8');
  const processed = data.toUpperCase();
  fs.writeFileSync(outputPath, processed);
  // Huge memory spike for large files
}

// GOOD: Stream processing
function processFileGood(inputPath, outputPath) {
  const transform = new Transform({
    transform(chunk, encoding, callback) {
      this.push(chunk.toString().toUpperCase());
      callback();
    },
  });

  fs.createReadStream(inputPath)
    .pipe(transform)
    .pipe(fs.createWriteStream(outputPath));

  // Memory usage stays constant
}
```

---

## Best Practices

### 1. Understand Your Memory Budget

```javascript
const v8 = require('v8');

// Calculate safe memory usage
const heapStats = v8.getHeapStatistics();
const maxHeap = heapStats.heap_size_limit;
const safeLimit = maxHeap * 0.75; // Leave 25% buffer

console.log(`Maximum safe memory usage: ${safeLimit / 1024 / 1024} MB`);
```

### 2. Monitor in Production

```javascript
// Simple production memory monitoring
function setupMemoryMonitoring() {
  const interval = setInterval(() => {
    const usage = process.memoryUsage();

    console.log({
      rss: `${(usage.rss / 1024 / 1024).toFixed(2)} MB`,
      heapTotal: `${(usage.heapTotal / 1024 / 1024).toFixed(2)} MB`,
      heapUsed: `${(usage.heapUsed / 1024 / 1024).toFixed(2)} MB`,
      external: `${(usage.external / 1024 / 1024).toFixed(2)} MB`,
    });

    // Alert if heap usage exceeds threshold
    if (usage.heapUsed / usage.heapTotal > 0.9) {
      console.error('ALERT: High memory usage detected!');
      // Send to monitoring service
    }
  }, 60000); // Every minute

  return () => clearInterval(interval);
}
```

### 3. Avoid Memory Leaks

**Checklist**:

- ✅ Remove event listeners when done
- ✅ Clear timers and intervals
- ✅ Null out large data structures
- ✅ Use WeakMap/WeakSet for caches
- ✅ Implement proper cleanup in classes

```javascript
class ProperCleanup {
  constructor() {
    this.listeners = [];
    this.timers = [];
    this.data = new Map();
  }

  addListener(emitter, event, handler) {
    emitter.on(event, handler);
    this.listeners.push({ emitter, event, handler });
  }

  addTimer(fn, delay) {
    const id = setInterval(fn, delay);
    this.timers.push(id);
    return id;
  }

  cleanup() {
    // Remove all listeners
    for (const { emitter, event, handler } of this.listeners) {
      emitter.off(event, handler);
    }
    this.listeners = [];

    // Clear all timers
    for (const id of this.timers) {
      clearInterval(id);
    }
    this.timers = [];

    // Clear data
    this.data.clear();
  }
}
```

### 4. Use WeakMap for Metadata

```javascript
// BAD: Prevents GC of objects used as keys
const metadata = new Map();

function attachMetadata(obj, data) {
  metadata.set(obj, data); // Keeps obj alive!
}

// GOOD: Allows GC of objects
const weakMetadata = new WeakMap();

function attachMetadataWeak(obj, data) {
  weakMetadata.set(obj, data); // Doesn't prevent GC
}

// When obj is no longer referenced elsewhere, it can be collected
// and the WeakMap entry is automatically removed
```

### 5. Benchmark Memory Usage

```javascript
function benchmarkMemory(fn, label) {
  if (global.gc) global.gc(); // Clear memory

  const before = process.memoryUsage();
  const startTime = Date.now();

  fn();

  if (global.gc) global.gc(); // Force GC

  const after = process.memoryUsage();
  const duration = Date.now() - startTime;

  console.log(`\n=== ${label} ===`);
  console.log(`Duration: ${duration}ms`);
  console.log(
    `Heap Delta: ${((after.heapUsed - before.heapUsed) / 1024 / 1024).toFixed(
      2
    )} MB`
  );
  console.log(
    `RSS Delta: ${((after.rss - before.rss) / 1024 / 1024).toFixed(2)} MB`
  );
}

// Usage (run with --expose-gc)
benchmarkMemory(() => {
  const arr = new Array(1000000).fill('test');
}, 'Large Array Allocation');
```

---

## Advanced: GC Tuning and Configuration

### V8 GC Flags

V8 exposes many flags for tuning garbage collection:

```bash
# Memory limits
node --max-old-space-size=4096 app.js       # 4GB old gen
node --max-semi-space-size=16 app.js        # 16MB young gen per semi-space
node --max-heap-size=8192 app.js            # 8GB total heap

# GC behavior
node --expose-gc app.js                      # Expose global.gc()
node --gc-interval=100 app.js                # GC after N allocations
node --optimize-for-size app.js              # Minimize memory over speed
node --optimize-for-latency app.js           # Minimize pause times

# Debugging
node --trace-gc app.js                       # Log all GC events
node --trace-gc-verbose app.js               # Detailed GC logs
node --trace-gc-nvp app.js                   # Name-value pair format
```

### Interpreting GC Traces

```bash
# Run with tracing
node --trace-gc app.js
```

Output:

```
[12844:0x5600a7f00000]       63 ms: Scavenge 2.8 (3.9) -> 2.1 (4.9) MB, 1.2 / 0.0 ms  (average mu = 0.994, current mu = 0.994) allocation failure
[12844:0x5600a7f00000]      891 ms: Mark-sweep 18.2 (23.4) -> 16.8 (24.9) MB, 12.3 / 0.0 ms  (average mu = 0.891, current mu = 0.756) allocation failure scavenge might not succeed
```

**Decoding**:

```
[PID:isolate]    time: GC_Type  before (reserved) -> after (reserved) MB,
                       pause / concurrent ms  (stats) reason
```

- **Scavenge**: Young gen GC (should be <5ms)
- **Mark-sweep**: Old gen GC (should be <20ms)
- **mu**: Mutator utilization (% time running app vs GC)
  - mu = 0.994 means 99.4% app, 0.6% GC (excellent)
  - mu < 0.9 means GC is taking >10% of time (bad)

### When to Tune GC

**Symptoms that need tuning**:

1. **Frequent old gen GC**:

   ```bash
   # Too many Mark-sweep events
   [12844]  100ms: Mark-sweep ...
   [12844]  250ms: Mark-sweep ...
   [12844]  380ms: Mark-sweep ...
   ```

   **Solution**: Increase `--max-old-space-size`

2. **Long young gen pauses**:

   ```bash
   [12844]  100ms: Scavenge ... 15.3ms  # Too long!
   ```

   **Solution**: Decrease `--max-semi-space-size` (smaller = faster GC)

3. **Low mutator utilization**:
   ```bash
   [12844]  ... (average mu = 0.750, current mu = 0.680)
   ```
   **Solution**: Increase heap size or reduce allocation rate

### Real-World Tuning Examples

#### Example 1: High-Throughput Server

```javascript
// Problem: Frequent GC pauses under load
// Symptom: 99th percentile latency spikes every few seconds

// Before:
// node app.js
// → GC every 2 seconds, 10ms pause

// Solution: Increase heap, accept higher memory usage
// node --max-old-space-size=8192 app.js
// → GC every 30 seconds, 15ms pause (but less frequent)
```

#### Example 2: Memory-Constrained Container

```javascript
// Problem: Running in 512MB container, OOM crashes
// Symptom: Heap grows to 450MB, then crashes

// Solution: Set explicit limit with buffer for safety
// node --max-old-space-size=400 app.js
// → V8 knows to GC more aggressively before OOM
```

#### Example 3: Real-Time Game Server

```javascript
// Problem: GC pauses cause frame drops
// Requirement: Never pause >5ms

// Solution: Small young gen + incremental old gen
// node --max-semi-space-size=2 \
//      --max-old-space-size=2048 \
//      app.js
// → Young gen GC <2ms, old gen incremental
```

### GC Tuning Decision Matrix

```javascript
const tuningGuide = {
  highThroughput: {
    priority: 'Maximize ops/sec',
    flags: '--max-old-space-size=8192',
    tradeoff: 'Higher memory usage, less frequent GC',
  },

  lowLatency: {
    priority: 'Minimize pause times',
    flags: '--max-semi-space-size=2',
    tradeoff: 'More frequent (but shorter) GC',
  },

  memoryConstrained: {
    priority: 'Minimize memory footprint',
    flags: '--max-old-space-size=512 --optimize-for-size',
    tradeoff: 'More frequent GC, lower throughput',
  },

  default: {
    priority: 'Balanced',
    flags: '(no flags, use defaults)',
    tradeoff: 'V8 defaults are good for most apps',
  },
};
```

### Monitoring GC in Production

```javascript
const v8 = require('v8');
const { PerformanceObserver } = require('perf_hooks');

class ProductionGCMonitor {
  constructor(alertThresholds = {}) {
    this.thresholds = {
      pauseTime: alertThresholds.pauseTime || 50, // ms
      frequency: alertThresholds.frequency || 10, // per minute
      heapUsage: alertThresholds.heapUsage || 0.9, // ratio
    };

    this.gcEvents = [];
    this.setupObserver();
  }

  setupObserver() {
    const obs = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        this.recordGC(entry);
      }
    });
    obs.observe({ entryTypes: ['gc'], buffered: true });
  }

  recordGC(entry) {
    const event = {
      kind: entry.kind,
      duration: entry.duration,
      timestamp: Date.now(),
    };

    this.gcEvents.push(event);

    // Keep last 1000 events
    if (this.gcEvents.length > 1000) {
      this.gcEvents.shift();
    }

    // Alert on long pauses
    if (entry.duration > this.thresholds.pauseTime) {
      this.alert('LONG_GC_PAUSE', {
        duration: entry.duration,
        kind: entry.kind,
      });
    }

    // Check frequency
    const recentEvents = this.gcEvents.filter(
      (e) => Date.now() - e.timestamp < 60000
    );
    if (recentEvents.length > this.thresholds.frequency) {
      this.alert('HIGH_GC_FREQUENCY', {
        count: recentEvents.length,
      });
    }
  }

  alert(type, data) {
    console.error(`[GC ALERT] ${type}:`, data);
    // Send to monitoring service (DataDog, New Relic, etc.)
  }

  getStats() {
    const stats = v8.getHeapStatistics();
    const recent = this.gcEvents.slice(-100);

    return {
      heap: {
        used: stats.used_heap_size,
        total: stats.total_heap_size,
        limit: stats.heap_size_limit,
        usage: stats.used_heap_size / stats.heap_size_limit,
      },
      gc: {
        avgPause:
          recent.reduce((sum, e) => sum + e.duration, 0) / recent.length,
        maxPause: Math.max(...recent.map((e) => e.duration)),
        frequency: recent.length, // per last 100 events
      },
    };
  }
}

// Usage
const monitor = new ProductionGCMonitor({
  pauseTime: 20, // Alert if GC pause >20ms
  frequency: 20, // Alert if >20 GC/min
  heapUsage: 0.85, // Alert if heap >85% full
});

setInterval(() => {
  const stats = monitor.getStats();
  console.log('GC Stats:', stats);
}, 60000);
```

---

## Exercises

### Exercise 1: Identify Memory Leaks

Find and fix the memory leaks in this code:

```javascript
const EventEmitter = require('events');

class DataProcessor extends EventEmitter {
  constructor() {
    super();
    this.cache = new Map();

    setInterval(() => {
      this.emit('tick');
    }, 1000);
  }

  process(data) {
    const id = Math.random();
    this.cache.set(id, data);

    setTimeout(() => {
      this.emit('processed', this.cache.get(id));
    }, 5000);
  }
}

// Create many processors
for (let i = 0; i < 100; i++) {
  const processor = new DataProcessor();
  processor.on('tick', () => {
    processor.process({ data: 'x'.repeat(1000) });
  });
}
```

<details>
<summary>Solution</summary>

```javascript
class DataProcessor extends EventEmitter {
  constructor() {
    super();
    this.cache = new Map();
    this.timerId = null;
  }

  start() {
    this.timerId = setInterval(() => {
      this.emit('tick');
    }, 1000);
  }

  process(data) {
    const id = Math.random();
    this.cache.set(id, data);

    setTimeout(() => {
      this.emit('processed', this.cache.get(id));
      this.cache.delete(id); // FIX: Clean up cache
    }, 5000);
  }

  cleanup() {
    // FIX: Clear timer
    if (this.timerId) {
      clearInterval(this.timerId);
      this.timerId = null;
    }

    // FIX: Clear cache
    this.cache.clear();

    // FIX: Remove all listeners
    this.removeAllListeners();
  }
}

// Create processors with cleanup
const processors = [];
for (let i = 0; i < 100; i++) {
  const processor = new DataProcessor();
  processor.on('tick', () => {
    processor.process({ data: 'x'.repeat(1000) });
  });
  processor.start();
  processors.push(processor);
}

// Clean up when done
process.on('SIGTERM', () => {
  processors.forEach((p) => p.cleanup());
});
```

</details>

### Exercise 2: Optimize GC Performance

Optimize this code to reduce GC pressure:

```javascript
function processLargeDataset(records) {
  return records
    .filter((r) => r.active)
    .map((r) => ({
      id: r.id,
      name: r.name.toUpperCase(),
      value: r.value * 1.1,
      timestamp: Date.now(),
    }))
    .sort((a, b) => b.value - a.value)
    .slice(0, 100);
}

const records = Array.from({ length: 1000000 }, (_, i) => ({
  id: i,
  name: `Record ${i}`,
  value: Math.random() * 1000,
  active: i % 2 === 0,
}));

console.time('process');
const result = processLargeDataset(records);
console.timeEnd('process');
```

<details>
<summary>Solution</summary>

```javascript
function processLargeDatasetOptimized(records) {
  // Single pass: filter, transform, and track top 100
  const topN = 100;
  const results = [];

  for (let i = 0; i < records.length; i++) {
    const r = records[i];
    if (!r.active) continue;

    const transformed = {
      id: r.id,
      name: r.name.toUpperCase(),
      value: r.value * 1.1,
      timestamp: Date.now(),
    };

    // Maintain sorted top-N array
    let insertIdx = results.length;
    for (let j = 0; j < results.length; j++) {
      if (transformed.value > results[j].value) {
        insertIdx = j;
        break;
      }
    }

    results.splice(insertIdx, 0, transformed);

    // Keep only top N
    if (results.length > topN) {
      results.pop();
    }
  }

  return results;
}
```

</details>

### Exercise 3: Memory Profiling

Create a tool to profile memory usage of different operations:

```javascript
// Implement a memory profiler that:
// 1. Takes snapshots before and after an operation
// 2. Calculates memory delta
// 3. Identifies which object types increased
// 4. Generates a report

class MemoryProfiler {
  // Your implementation here
}

// Usage:
const profiler = new MemoryProfiler();
await profiler.profile('Large Array', () => {
  const arr = new Array(1000000).fill({ data: 'test' });
});
```

---

## Further Reading

- [V8 Blog: Orinoco - Concurrent Marking](https://v8.dev/blog/concurrent-marking)
- [V8 Blog: Trash talk (Garbage Collection)](https://v8.dev/blog/trash-talk)
- [Node.js Memory Management](https://nodejs.org/en/docs/guides/simple-profiling/)
- [Chrome DevTools Memory Profiling](https://developer.chrome.com/docs/devtools/memory-problems/)

---

## Summary

Key takeaways:

1. **Heap Organization**: V8 divides memory into young and old generations for efficient GC
2. **GC Algorithms**: Different algorithms (Scavenger, Mark-Sweep, Incremental/Concurrent) optimize for different scenarios
3. **Write Barriers**: Enable concurrent GC but add overhead to writes
4. **Memory Monitoring**: Use heap statistics and snapshots to detect issues early
5. **Leak Prevention**: Clean up timers, listeners, and references appropriately
6. **Optimization**: Minimize allocations, reuse objects, and stream large data

Understanding V8's memory management helps you write more efficient Node.js applications and debug memory issues effectively.
