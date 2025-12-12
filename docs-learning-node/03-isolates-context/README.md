# Chapter 3: Isolates and Context Management

## Introduction

Understanding V8 Isolates and Contexts is fundamental to working with Node.js internals. An **Isolate** represents an isolated instance of the V8 engine with its own heap and independent execution state, while a **Context** provides a sandboxed JavaScript execution environment within an Isolate. This chapter explores these core concepts and their practical implications for Node.js applications.

## Table of Contents

- [Understanding V8 Isolates](#understanding-v8-isolates)
- [Context Creation and Management](#context-creation-and-management)
- [Security Boundaries and Realms](#security-boundaries-and-realms)
- [Worker Threads and Isolate Sharing](#worker-threads-and-isolate-sharing)
- [Practical Examples](#practical-examples)
- [Performance Considerations](#performance-considerations)
- [Exercises](#exercises)

---

## Understanding V8 Isolates

### What is an Isolate?

A V8 Isolate is an isolated instance of the V8 JavaScript engine. Each Isolate:

- Has its own heap (memory allocation area)
- Maintains its own garbage collector
- Has independent execution state
- Cannot directly share JavaScript objects with other Isolates
- Represents a complete V8 VM instance

Think of an Isolate as a completely separate JavaScript virtual machine. In Node.js:

- The main thread runs in one Isolate
- Each Worker thread gets its own Isolate
- Multiple Isolates can exist in a single process, but they are isolated from each other

### Key Characteristics

```cpp
// From deps/v8/include/v8-isolate.h
class V8_EXPORT Isolate {
 public:
  struct V8_EXPORT CreateParams {
    // ArrayBuffer allocator for memory management
    ArrayBuffer::Allocator* array_buffer_allocator = nullptr;
    std::shared_ptr<ArrayBuffer::Allocator> array_buffer_allocator_shared;

    // Resource constraints (heap size, etc.)
    ResourceConstraints constraints;

    // Snapshot for faster startup
    const StartupData* snapshot_blob = nullptr;

    // External references for serialization/deserialization
    const intptr_t* external_references = nullptr;

    // Whether Atomics.wait is allowed
    bool allow_atomics_wait = true;

    // Callbacks for errors
    FatalErrorCallback fatal_error_callback = nullptr;
    OOMErrorCallback oom_error_callback = nullptr;
  };
};
```

### Isolate Lifecycle

#### 1. Creation

In Node.js, Isolates are created using the `NewIsolate()` function:

```cpp
// From src/api/environment.cc
Isolate* NewIsolate(Isolate::CreateParams* params,
                    uv_loop_t* event_loop,
                    MultiIsolatePlatform* platform,
                    const SnapshotData* snapshot_data,
                    const IsolateSettings& settings) {
  Isolate* isolate = Isolate::Allocate();
  CHECK_NOT_NULL(isolate);

  platform->RegisterIsolate(isolate, event_loop);
  Isolate::Initialize(isolate, *params);

  Isolate::Scope isolate_scope(isolate);

  if (snapshot_data == nullptr) {
    SetIsolateUpForNode(isolate, settings);
  } else {
    SetIsolateMiscHandlers(isolate, settings);
  }

  return isolate;
}
```

**Key steps:**

1. **Allocate** - Reserve memory for the Isolate structure
2. **Register** - Register with the platform for task scheduling
3. **Initialize** - Set up the Isolate with provided parameters
4. **Configure** - Apply Node.js-specific settings

#### 2. Entering and Exiting

You must "enter" an Isolate before executing any V8 operations:

```cpp
// Isolate::Scope automatically enters/exits
{
  Isolate::Scope isolate_scope(isolate);
  // V8 operations here
} // Isolate is exited when scope ends
```

#### 3. Disposal

```cpp
// From src/node_worker.cc - WorkerThreadData destructor
platform->AddIsolateFinishedCallback(isolate, [](void* data) {
  *static_cast<bool*>(data) = true;
}, &platform_finished);

platform->DisposeIsolate(isolate);

// Wait until platform cleanup is complete
while (!platform_finished) {
  uv_run(&loop, UV_RUN_ONCE);
}
```

### Resource Constraints

Isolates can be configured with memory and execution limits:

```cpp
v8::ResourceConstraints constraints;

// Configure heap size (MB)
constraints.ConfigureDefaultsFromHeapSize(
  128 * 1024 * 1024,  // Initial: 128 MB
  512 * 1024 * 1024   // Maximum: 512 MB
);

Isolate::CreateParams params;
params.constraints = constraints;
```

**Important constraints:**

- `max_old_generation_size_in_bytes` - Maximum old generation heap size
- `max_young_generation_size_in_bytes` - Maximum young generation size
- Stack size limits (set separately per thread)

### Isolate Data

Node.js attaches metadata to each Isolate using `IsolateData`:

```cpp
// From src/node.h
IsolateData* CreateIsolateData(
    v8::Isolate* isolate,
    uv_loop_t* loop,
    MultiIsolatePlatform* platform = nullptr,
    ArrayBufferAllocator* allocator = nullptr,
    const EmbedderSnapshotData* snapshot_data = nullptr);
```

`IsolateData` stores:

- Reference to the event loop
- Platform for task scheduling
- Array buffer allocator
- Cached strings and templates
- Worker context (if applicable)

---

## Context Creation and Management

### What is a Context?

A **Context** is a sandboxed JavaScript execution environment within an Isolate. Each Context:

- Has its own global object
- Has its own set of built-in objects (Object, Array, etc.)
- Is isolated from other Contexts (security boundary)
- Multiple Contexts can coexist in a single Isolate

```cpp
// From deps/v8/include/v8-context.h
/**
 * A sandboxed execution context with its own set of built-in objects
 * and functions.
 */
class V8_EXPORT Context : public Data {
 public:
  /**
   * Returns the global proxy object.
   */
  Local<Object> Global();

  /**
   * Creates a new context.
   */
  static Local<Context> New(
      Isolate* isolate,
      ExtensionConfiguration* extensions = nullptr,
      MaybeLocal<ObjectTemplate> global_template = MaybeLocal<ObjectTemplate>(),
      MaybeLocal<Value> global_object = MaybeLocal<Value>(),
      DeserializeInternalFieldsCallback internal_fields_deserializer =
          DeserializeInternalFieldsCallback(),
      MicrotaskQueue* microtask_queue = nullptr);
};
```

### Creating a Context

Node.js provides a convenience function for creating Contexts:

```cpp
// From src/node.h
v8::Local<v8::Context> NewContext(
    v8::Isolate* isolate,
    v8::Local<v8::ObjectTemplate> object_template =
        v8::Local<v8::ObjectTemplate>());
```

**Example:**

```cpp
Isolate* isolate = Isolate::GetCurrent();
HandleScope handle_scope(isolate);

// Create a new context
Local<Context> context = Context::New(isolate);

// Enter the context to execute code
Context::Scope context_scope(context);

// Now you can execute JavaScript in this context
Local<String> source = String::NewFromUtf8Literal(isolate,
    "const x = 42; x * 2");
Local<Script> script = Script::Compile(context, source).ToLocalChecked();
Local<Value> result = script->Run(context).ToLocalChecked();
// result is 84
```

### Context Initialization

Node.js applies specific tweaks to new Contexts:

```cpp
// From src/api/environment.cc
v8::Maybe<bool> InitializeContext(v8::Local<v8::Context> context) {
  // Sets up:
  // - Error stack trace customization
  // - Promise hooks
  // - Module resolution
  // - Other Node.js-specific behavior
}
```

### Global Object vs Global Proxy

Contexts use a **Global Proxy** pattern for security:

```
User Code → Global Proxy → Actual Global Object
```

**Why?**

- The proxy can intercept property access
- Enables security checks before accessing globals
- Allows context switching without copying objects

```cpp
Local<Context> context = Context::New(isolate);
Local<Object> global_proxy = context->Global();  // The proxy
// The real global object is hidden behind the proxy
```

### Multiple Contexts in One Isolate

```cpp
Isolate* isolate = Isolate::GetCurrent();
HandleScope handle_scope(isolate);

// Create two separate contexts
Local<Context> context1 = Context::New(isolate);
Local<Context> context2 = Context::New(isolate);

// Each has its own global object and built-ins
{
  Context::Scope scope1(context1);
  // Modify Array.prototype in context1
  // This does NOT affect context2
}

{
  Context::Scope scope2(context2);
  // Array.prototype is pristine here
}
```

**Use cases:**

- Running untrusted code (VM module)
- REPL sessions
- Testing environments
- Custom JavaScript environments

---

## Security Boundaries and Realms

### Contexts as Security Boundaries

Contexts provide **security isolation** through separate global objects:

```javascript
// In Node.js using the 'vm' module
const vm = require('vm');

// Create a sandboxed context
const sandbox = { x: 5 };
const context = vm.createContext(sandbox);

// Run code in sandbox
vm.runInContext('x = 10; y = 20;', context);

console.log(sandbox.x); // 10
console.log(sandbox.y); // 20
console.log(typeof y); // undefined - y is NOT in global scope
```

### What's Isolated?

✅ **Isolated between Contexts:**

- Global variables
- Built-in prototypes (Array.prototype, Object.prototype, etc.)
- Global functions (setTimeout in browser, but NOT in Node.js - it's on `global`)

❌ **NOT Isolated:**

- Native objects shared between contexts (can leak references)
- CPU and memory resources (shared within Isolate)
- The Isolate's heap (objects are in the same heap)

### Security Considerations

```cpp
// From Node.js VM module implementation
Local<Context> context = Context::New(
    isolate,
    nullptr,  // No extensions
    global_template);  // Custom global

// IMPORTANT: Must initialize properly
if (InitializeContext(context).IsNothing()) {
  return;  // Failed to initialize
}
```

**Security pitfalls:**

1. **Prototype pollution** - Modifying Object.prototype affects new objects
2. **Constructor access** - `obj.constructor` can escape sandbox
3. **Shared native objects** - Buffers, TypedArrays can leak
4. **Side channels** - Timing attacks, resource exhaustion

### Realms and ShadowRealms

The ShadowRealm proposal (Stage 3) provides a more robust isolation mechanism:

```javascript
// Future API (ShadowRealm proposal)
const realm = new ShadowRealm();

// Completely isolated execution environment
const result = realm.evaluate('1 + 1'); // 2

// Only primitives can cross the boundary
realm.evaluate('globalThis.x = { data: 42 }');
const x = realm.evaluate('globalThis.x'); // TypeError: not a primitive
```

**Key differences from vm.Context:**

- Stronger isolation guarantees
- No direct object sharing
- Only primitives cross boundaries
- Built-in to the language (no C++ needed)

---

## Worker Threads and Isolate Sharing

### One Isolate Per Worker

Each Node.js Worker thread gets its own Isolate:

```javascript
// main.js
const { Worker } = require('worker_threads');

// Creates a new thread with its own Isolate
const worker = new Worker('./worker.js');
```

```cpp
// From src/node_worker.cc
class WorkerThreadData {
  explicit WorkerThreadData(Worker* w) {
    // Create new event loop
    uv_loop_init(&loop_);

    // Create array buffer allocator
    std::shared_ptr<ArrayBufferAllocator> allocator =
        ArrayBufferAllocator::Create();

    // Set up Isolate parameters
    Isolate::CreateParams params;
    SetIsolateCreateParamsForNode(&params);
    w->UpdateResourceConstraints(&params.constraints);
    params.array_buffer_allocator_shared = allocator;

    // Create the new Isolate for this worker
    Isolate* isolate = NewIsolate(
        &params,
        &loop_,
        w->platform_,
        w->snapshot_data());

    // Store isolate reference
    w->isolate_ = isolate;
  }
};
```

### Worker Lifecycle

```cpp
// From src/node_worker.cc
void Worker::Run() {
  // Create worker-specific data (including Isolate)
  WorkerThreadData data(this);
  if (isolate_ == nullptr) return;

  {
    Locker locker(isolate_);
    Isolate::Scope isolate_scope(isolate_);

    // Create IsolateData
    // Create Environment
    // Run worker script
    // Event loop processing
  }

  // Cleanup when worker exits
}
```

**Lifecycle stages:**

1. **Thread creation** - OS thread spawned
2. **Isolate creation** - New V8 Isolate allocated
3. **Environment setup** - Node.js environment initialized
4. **Execution** - Worker script runs
5. **Event loop** - Process async operations
6. **Cleanup** - Dispose Isolate and free resources

### Communication Between Workers

Workers cannot share JavaScript objects directly. Instead, they use:

#### 1. Message Passing (Structured Clone)

```javascript
// main.js
const { Worker } = require('worker_threads');
const worker = new Worker('./worker.js');

worker.postMessage({ data: [1, 2, 3] }); // Structured clone
worker.on('message', (msg) => {
  console.log('Received:', msg);
});
```

```javascript
// worker.js
const { parentPort } = require('worker_threads');

parentPort.on('message', (msg) => {
  // msg is a COPY of the original data
  console.log('Worker received:', msg);
  parentPort.postMessage({ result: 'done' });
});
```

**Structured Clone Algorithm:**

- Creates a deep copy of the data
- Supports most JavaScript types
- Cannot clone functions, symbols, or certain objects

#### 2. SharedArrayBuffer (Zero-Copy Sharing)

```javascript
// main.js
const { Worker } = require('worker_threads');

// Create shared memory
const sharedBuffer = new SharedArrayBuffer(1024);
const sharedArray = new Int32Array(sharedBuffer);

const worker = new Worker('./worker.js', {
  workerData: { sharedBuffer },
});

// Both main and worker can access same memory
sharedArray[0] = 42;
```

```javascript
// worker.js
const { workerData } = require('worker_threads');
const sharedArray = new Int32Array(workerData.sharedBuffer);

console.log(sharedArray[0]); // 42 - same memory!
```

**Important:**

- Requires `Atomics` for thread-safe operations
- Risk of data races
- Memory is shared, but each Isolate still has its own heap

#### 3. Transfer List (Ownership Transfer)

```javascript
const buffer = new ArrayBuffer(1024);
worker.postMessage({ buffer }, [buffer]);
// buffer is now neutered (length = 0) in main thread
// worker owns it exclusively
```

### Isolate Sharing: What's NOT Possible

❌ Cannot share:

- JavaScript objects across Isolates
- Function references
- Class instances
- Direct memory pointers (safely)

✅ Can share:

- SharedArrayBuffer (with care)
- Transferable objects (ownership transfer)
- Primitive data via messaging

### Resource Constraints Per Worker

```javascript
const { Worker } = require('worker_threads');

const worker = new Worker('./worker.js', {
  resourceLimits: {
    maxOldGenerationSizeMb: 512,
    maxYoungGenerationSizeMb: 64,
    codeRangeSizeMb: 128,
    stackSizeMb: 4,
  },
});
```

```cpp
// From src/node_worker.cc
void Worker::UpdateResourceConstraints(ResourceConstraints* constraints) {
  if (resource_limits_[kMaxYoungGenerationSizeMb] > 0) {
    constraints->set_max_young_generation_size_in_bytes(
        static_cast<size_t>(resource_limits_[kMaxYoungGenerationSizeMb])
        * kMB);
  }

  if (resource_limits_[kMaxOldGenerationSizeMb] > 0) {
    constraints->set_max_old_generation_size_in_bytes(
        static_cast<size_t>(resource_limits_[kMaxOldGenerationSizeMb])
        * kMB);
  }
  // ... more constraints
}
```

---

## Practical Examples

### Example 1: Creating a Custom Context

```cpp
#include <node.h>
#include <v8.h>

using v8::Context;
using v8::FunctionCallbackInfo;
using v8::HandleScope;
using v8::Isolate;
using v8::Local;
using v8::Object;
using v8::ObjectTemplate;
using v8::String;
using v8::Value;

void CustomLog(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();
  HandleScope scope(isolate);

  if (args.Length() > 0) {
    String::Utf8Value str(isolate, args[0]);
    printf("Custom Log: %s\n", *str);
  }
}

void CreateCustomContext(const FunctionCallbackInfo<Value>& args) {
  Isolate* isolate = args.GetIsolate();
  HandleScope scope(isolate);

  // Create a template for the global object
  Local<ObjectTemplate> global = ObjectTemplate::New(isolate);

  // Add custom 'log' function
  global->Set(isolate, "log",
              v8::FunctionTemplate::New(isolate, CustomLog));

  // Create context with custom global
  Local<Context> context = Context::New(isolate, nullptr, global);

  // Enter the context and run some code
  Context::Scope context_scope(context);

  Local<String> source = String::NewFromUtf8Literal(isolate,
      "log('Hello from custom context!'); 42;");

  Local<v8::Script> script =
      v8::Script::Compile(context, source).ToLocalChecked();

  Local<Value> result = script->Run(context).ToLocalChecked();

  args.GetReturnValue().Set(result);
}

void Initialize(Local<Object> exports) {
  NODE_SET_METHOD(exports, "createCustomContext", CreateCustomContext);
}

NODE_MODULE(NODE_GYP_MODULE_NAME, Initialize)
```

### Example 2: Worker Thread Communication

```javascript
// main.js
const { Worker } = require('worker_threads');
const path = require('path');

function runWorker(data) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(path.join(__dirname, 'worker.js'), {
      workerData: data,
    });

    worker.on('message', resolve);
    worker.on('error', reject);
    worker.on('exit', (code) => {
      if (code !== 0) {
        reject(new Error(`Worker stopped with exit code ${code}`));
      }
    });
  });
}

// Run multiple workers in parallel
async function main() {
  const results = await Promise.all([
    runWorker({ task: 'heavy-compute', input: 1000000 }),
    runWorker({ task: 'heavy-compute', input: 2000000 }),
    runWorker({ task: 'heavy-compute', input: 3000000 }),
  ]);

  console.log('All workers completed:', results);
}

main().catch(console.error);
```

```javascript
// worker.js
const { workerData, parentPort } = require('worker_threads');

function heavyCompute(n) {
  let result = 0;
  for (let i = 0; i < n; i++) {
    result += Math.sqrt(i);
  }
  return result;
}

const result = heavyCompute(workerData.input);
parentPort.postMessage({ result });
```

### Example 3: SharedArrayBuffer for Real-Time Updates

```javascript
// main.js
const { Worker } = require('worker_threads');

// Shared memory for counter
const sharedBuffer = new SharedArrayBuffer(Int32Array.BYTES_PER_ELEMENT);
const sharedArray = new Int32Array(sharedBuffer);

const worker = new Worker(
  `
  const { workerData } = require('worker_threads');
  const sharedArray = new Int32Array(workerData.buffer);

  // Increment counter atomically
  setInterval(() => {
    Atomics.add(sharedArray, 0, 1);
  }, 100);
`,
  {
    eval: true,
    workerData: { buffer: sharedBuffer },
  }
);

// Monitor counter from main thread
setInterval(() => {
  const count = Atomics.load(sharedArray, 0);
  console.log('Counter:', count);
}, 500);
```

---

## Performance Considerations

### Isolate Creation Overhead

Creating a new Isolate is expensive:

- Memory allocation (~10-20 MB minimum)
- Heap initialization
- Built-in objects creation
- Snapshot deserialization

```javascript
// DON'T: Create worker for every request
app.get('/api', (req, res) => {
  const worker = new Worker('./handler.js'); // Too expensive!
  // ...
});

// DO: Use worker pool
const { WorkerPool } = require('worker-pool-library');
const pool = new WorkerPool({ size: 4 });

app.get('/api', async (req, res) => {
  const result = await pool.exec('./handler.js', data);
  res.json(result);
});
```

### Context Creation Performance

Contexts are lighter than Isolates but still have overhead:

```cpp
// Benchmark: Context creation time
auto start = std::chrono::high_resolution_clock::now();

for (int i = 0; i < 1000; i++) {
  Local<Context> context = Context::New(isolate);
}

auto end = std::chrono::high_resolution_clock::now();
// Typically: 0.05-0.2 ms per context
```

**Best practices:**

- Reuse contexts when possible
- Use context snapshots for faster creation
- Consider pooling for frequently-created contexts

### Memory Implications

**Per Isolate:**

- Minimum heap: ~2-10 MB
- Typical heap: 10-100 MB
- Maximum heap: Configurable (default ~1.4 GB on 64-bit)

**Per Context:**

- Global object: ~100 KB
- Built-in objects: ~500 KB - 1 MB
- Additional overhead: Varies with usage

### Worker Thread Best Practices

1. **Limit worker count**

   ```javascript
   const os = require('os');
   const maxWorkers = os.cpus().length;
   ```

2. **Use worker pools**

   - Reuse workers across tasks
   - Avoid creation/destruction overhead

3. **Batch work**

   ```javascript
   // Better: Send batch of work
   worker.postMessage({ items: [1, 2, 3, 4, 5] });

   // Worse: Multiple messages
   items.forEach((item) => worker.postMessage(item));
   ```

4. **Monitor memory**
   ```javascript
   const used = process.memoryUsage();
   console.log(`Heap used: ${used.heapUsed / 1024 / 1024} MB`);
   ```

---

## Exercises

### Exercise 1: Context Isolation Verification

Create a program that demonstrates context isolation:

```javascript
const vm = require('vm');

// 1. Create two contexts
// 2. Modify Array.prototype in one context
// 3. Verify the other context is unaffected
// 4. Try to access variables across contexts

// Your code here
```

**Expected behavior:**

- Changes in one context don't affect the other
- Variables are completely isolated
- Built-in prototypes are independent

<details>
<summary>Solution</summary>

```javascript
const vm = require('vm');

// Context 1
const sandbox1 = {};
const context1 = vm.createContext(sandbox1);

vm.runInContext(
  `
  Array.prototype.customMethod = function() {
    return 'Context 1';
  };
  var x = 42;
`,
  context1
);

// Context 2
const sandbox2 = {};
const context2 = vm.createContext(sandbox2);

vm.runInContext(
  `
  var x = 100;
  var testArray = [1, 2, 3];
`,
  context2
);

// Test isolation
const result1 = vm.runInContext('[].customMethod()', context1);
console.log('Context 1 custom method:', result1); // 'Context 1'

try {
  vm.runInContext('[].customMethod()', context2);
} catch (e) {
  console.log('Context 2 error:', e.message); // customMethod is not a function
}

const x1 = vm.runInContext('x', context1);
const x2 = vm.runInContext('x', context2);
console.log('Context 1 x:', x1); // 42
console.log('Context 2 x:', x2); // 100
```

</details>

### Exercise 2: Worker Thread Pool

Implement a simple worker pool:

```javascript
class WorkerPool {
  constructor(scriptPath, poolSize) {
    // Initialize pool
  }

  async execute(data) {
    // Get available worker
    // Send task
    // Return result
  }

  destroy() {
    // Clean up all workers
  }
}

// Use it
const pool = new WorkerPool('./worker.js', 4);
const result = await pool.execute({ task: 'process', data: 100 });
```

<details>
<summary>Solution</summary>

```javascript
const { Worker } = require('worker_threads');
const EventEmitter = require('events');

class WorkerPool extends EventEmitter {
  constructor(scriptPath, poolSize = 4) {
    super();
    this.scriptPath = scriptPath;
    this.poolSize = poolSize;
    this.workers = [];
    this.queue = [];

    for (let i = 0; i < poolSize; i++) {
      this.addWorker();
    }
  }

  addWorker() {
    const worker = new Worker(this.scriptPath);

    worker.on('message', (result) => {
      worker.currentTask.resolve(result);
      worker.currentTask = null;
      this.processQueue();
    });

    worker.on('error', (err) => {
      if (worker.currentTask) {
        worker.currentTask.reject(err);
        worker.currentTask = null;
      }
    });

    this.workers.push(worker);
  }

  execute(data) {
    return new Promise((resolve, reject) => {
      this.queue.push({ data, resolve, reject });
      this.processQueue();
    });
  }

  processQueue() {
    if (this.queue.length === 0) return;

    const availableWorker = this.workers.find((w) => !w.currentTask);
    if (!availableWorker) return;

    const task = this.queue.shift();
    availableWorker.currentTask = task;
    availableWorker.postMessage(task.data);
  }

  async destroy() {
    await Promise.all(this.workers.map((w) => w.terminate()));
  }
}

module.exports = WorkerPool;
```

```javascript
// worker.js
const { parentPort } = require('worker_threads');

parentPort.on('message', (data) => {
  // Process data
  const result = processData(data);
  parentPort.postMessage(result);
});

function processData(data) {
  // Your processing logic
  return { result: data.value * 2 };
}
```

</details>

### Exercise 3: SharedArrayBuffer Synchronization

Implement a producer-consumer pattern using SharedArrayBuffer:

```javascript
// Create a shared buffer with:
// - Index 0: Counter for produced items
// - Index 1: Counter for consumed items
// - Index 2-101: Ring buffer for data

// Producer worker: Generates numbers
// Consumer worker: Processes numbers
// Use Atomics for synchronization
```

<details>
<summary>Solution</summary>

```javascript
// main.js
const { Worker } = require('worker_threads');

const BUFFER_SIZE = 100;
const sharedBuffer = new SharedArrayBuffer(
  Int32Array.BYTES_PER_ELEMENT * (BUFFER_SIZE + 2)
);
const sharedArray = new Int32Array(sharedBuffer);

// Index 0: produced count
// Index 1: consumed count
// Index 2+: ring buffer

const producer = new Worker(
  `
  const { workerData } = require('worker_threads');
  const array = new Int32Array(workerData.buffer);
  const BUFFER_SIZE = ${BUFFER_SIZE};

  for (let i = 0; i < 1000; i++) {
    // Wait if buffer is full
    while (Atomics.load(array, 0) - Atomics.load(array, 1) >= BUFFER_SIZE) {
      Atomics.wait(array, 0, Atomics.load(array, 0), 10);
    }

    const produced = Atomics.load(array, 0);
    const index = (produced % BUFFER_SIZE) + 2;

    // Store value
    Atomics.store(array, index, i);

    // Increment produced count
    Atomics.add(array, 0, 1);

    // Notify consumer
    Atomics.notify(array, 0, 1);
  }
`,
  {
    eval: true,
    workerData: { buffer: sharedBuffer },
  }
);

const consumer = new Worker(
  `
  const { workerData, parentPort } = require('worker_threads');
  const array = new Int32Array(workerData.buffer);
  const BUFFER_SIZE = ${BUFFER_SIZE};

  let sum = 0;

  for (let i = 0; i < 1000; i++) {
    // Wait if buffer is empty
    while (Atomics.load(array, 1) >= Atomics.load(array, 0)) {
      Atomics.wait(array, 1, Atomics.load(array, 1), 10);
    }

    const consumed = Atomics.load(array, 1);
    const index = (consumed % BUFFER_SIZE) + 2;

    // Read value
    const value = Atomics.load(array, index);
    sum += value;

    // Increment consumed count
    Atomics.add(array, 1, 1);

    // Notify producer
    Atomics.notify(array, 1, 1);
  }

  parentPort.postMessage({ sum });
`,
  {
    eval: true,
    workerData: { buffer: sharedBuffer },
  }
);

consumer.on('message', (msg) => {
  console.log('Sum of all values:', msg.sum);
  producer.terminate();
  consumer.terminate();
});
```

</details>

---

## Summary

Key takeaways:

1. **Isolates** are independent V8 instances with separate heaps
2. **Contexts** provide sandboxed execution within an Isolate
3. **Security** requires careful context management and awareness of shared references
4. **Workers** each get their own Isolate for true parallelism
5. **Communication** between Isolates requires message passing or shared memory
6. **Performance** considerations include creation overhead and memory usage

## Further Reading

- [V8 Embedder's Guide](https://v8.dev/docs/embed)
- [Node.js Worker Threads Documentation](https://nodejs.org/api/worker_threads.html)
- [VM Module Documentation](https://nodejs.org/api/vm.html)
- [SharedArrayBuffer and Atomics](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)

## Next Chapter

Proceed to [Chapter 4: Memory Management and Garbage Collection](../04-memory-gc/README.md) to learn how V8 manages memory within Isolates and Contexts.
