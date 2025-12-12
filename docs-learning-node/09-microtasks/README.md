# Microtask Queue and Event Loop Integration

## Learning Objectives

- Understand microtask vs macrotask scheduling
- Learn how V8 microtasks integrate with Node.js event loop
- Master Promise resolution timing
- Implement custom microtask management

## The JavaScript Event Loop

### Task Categories

1. **Macrotasks (Task Queue)**

   - setTimeout/setInterval callbacks
   - I/O operations
   - setImmediate (Node.js)
   - MessageChannel events

2. **Microtasks (Microtask Queue)**
   - Promise.then/catch/finally
   - queueMicrotask()
   - process.nextTick() (Node.js)
   - MutationObserver (browser)

### Execution Order

```
1. Execute current task
2. Process ALL microtasks
3. Render (browser only)
4. Execute next macrotask
5. Repeat
```

## V8 Microtask Implementation

### MicrotaskQueue Class

Located in `src/execution/microtask-queue.h`:

```cpp
class V8_EXPORT MicrotaskQueue {
 public:
  // Add a microtask to the queue
  void EnqueueMicrotask(Microtask microtask);

  // Process all pending microtasks
  void PerformCheckpoint(Isolate* isolate);

  // Check if queue has pending tasks
  bool HasPendingMicrotasks() const;

  // Get queue size
  int size() const;
};
```

### Microtask Types

1. **CallbackTask**

   ```cpp
   void EnqueueCallback(v8::Function* callback, v8::Value* data = nullptr);
   ```

2. **PromiseReactionTask**

   ```cpp
   // Created when Promise.then() is called
   class PromiseReactionJobTask {
     PromiseReaction reaction;
     Object argument;
   };
   ```

3. **PromiseResolveThenableTask**
   ```cpp
   // Created when resolving with thenable object
   class PromiseResolveThenableJobTask {
     Promise promise_to_resolve;
     Object thenable;
     Object then;
   };
   ```

## Node.js Integration

### Event Loop Phases

```
┌───────────────────────────┐
┌─>│           timers          │  ← setTimeout/setInterval
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  ← I/O callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  ← internal use
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           poll            │  ← fetch new I/O events
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │           check           │  ← setImmediate callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  ← socket.on('close', ...)
   └───────────────────────────┘
```

### Microtask Checkpoints

Microtasks are processed:

- Between each event loop phase
- After each individual callback
- Before returning to JavaScript from C++

### process.nextTick() vs Microtasks

```javascript
// Execution order demonstration
console.log('1');

setImmediate(() => console.log('2'));

Promise.resolve().then(() => console.log('3'));

process.nextTick(() => console.log('4'));

console.log('5');

// Output: 1, 5, 4, 3, 2
```

**Priority:**

1. process.nextTick() (highest)
2. Promise microtasks
3. setImmediate
4. setTimeout(fn, 0)

## Promise Implementation Details

### Promise States

```cpp
enum PromiseState {
  kPending,
  kFulfilled,
  kRejected
};
```

### Reaction Chain

When `promise.then()` is called:

```cpp
class PromiseReaction {
  Object fulfill_handler;    // Success callback
  Object reject_handler;     // Error callback
  Promise promise;           // Promise to resolve
  PromiseCapability capability;
};
```

### Resolution Process

```javascript
// This code:
Promise.resolve(42)
  .then((x) => x * 2)
  .then(console.log);

// Creates this microtask chain:
// 1. PromiseReactionJobTask(fulfill: x => x * 2, arg: 42)
// 2. PromiseReactionJobTask(fulfill: console.log, arg: 84)
```

## Async/Await Implementation

### Transformation

```javascript
// Original async function
async function fetchData() {
  const result = await fetch('/api/data');
  return result.json();
}

// Equivalent Promise chain (simplified)
function fetchData() {
  return fetch('/api/data').then((result) => {
    return result.json();
  });
}
```

### Generator-based Implementation

V8 transforms async/await using generators:

```javascript
function* fetchData() {
  const result = yield fetch('/api/data');
  return result.json();
}

// Runtime wrapper
function asyncWrapper(generator) {
  return new Promise((resolve, reject) => {
    const gen = generator();
    function step(value) {
      const { value: promiseOrValue, done } = gen.next(value);
      if (done) return resolve(promiseOrValue);
      Promise.resolve(promiseOrValue).then(step, reject);
    }
    step();
  });
}
```

## Performance Considerations

### Microtask Queue Overflow

```javascript
// BAD: Creates infinite microtask loop
function infiniteMicrotasks() {
  Promise.resolve().then(() => {
    infiniteMicrotasks(); // Starves event loop
  });
}

// BETTER: Allow other tasks to execute
function batchedMicrotasks() {
  setImmediate(() => {
    // Process batch
    for (let i = 0; i < 1000; i++) {
      // Do work
    }
    batchedMicrotasks();
  });
}
```

### Memory Usage

Microtasks hold references to:

- Callback functions
- Captured variables (closures)
- Promise objects
- Reaction chains

```javascript
// Memory-efficient pattern
function processItems(items) {
  let index = 0;

  function processNext() {
    if (index >= items.length) return;

    const item = items[index++];
    return processItem(item)
      .then(() => {
        // Use setImmediate to avoid microtask buildup
        return new Promise((resolve) => setImmediate(resolve));
      })
      .then(processNext);
  }

  return processNext();
}
```

## Custom Microtask API

### queueMicrotask()

```javascript
// Standard API
queueMicrotask(() => {
  console.log('Custom microtask executed');
});

// Equivalent to:
Promise.resolve().then(() => {
  console.log('Promise microtask executed');
});
```

### V8 C++ API

```cpp
// Queue a microtask from C++
isolate->EnqueueMicrotask(Local<Function>::New(isolate, callback));

// Or using MicrotaskQueue directly
MicrotaskQueue* queue = isolate->GetCurrentContext()->GetMicrotaskQueue();
queue->EnqueueMicrotask(Microtask::New(isolate, callback));
```

## Debugging Microtasks

### Tracing

```bash
# Trace microtask execution
node --trace-promises script.js

# Detailed Promise debugging
node --trace-promises-verbose script.js
```

### Performance Monitoring

```javascript
const { performance, PerformanceObserver } = require('perf_hooks');

const obs = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    if (entry.entryType === 'measure') {
      console.log(`${entry.name}: ${entry.duration}ms`);
    }
  }
});
obs.observe({ entryTypes: ['measure'] });

// Measure microtask batch processing
performance.mark('microtask-start');
for (let i = 0; i < 1000; i++) {
  Promise.resolve().then(() => {});
}
setImmediate(() => {
  performance.mark('microtask-end');
  performance.measure('microtask-batch', 'microtask-start', 'microtask-end');
});
```

## Common Patterns

### Yielding to Event Loop

```javascript
// Heavy computation with yielding
async function heavyComputation(data) {
  const results = [];

  for (let i = 0; i < data.length; i++) {
    results.push(processItem(data[i]));

    // Yield every 100 items
    if (i % 100 === 0) {
      await new Promise((resolve) => setImmediate(resolve));
    }
  }

  return results;
}
```

### Error Handling

```javascript
// Proper error propagation
function robustAsyncOperation() {
  return Promise.resolve()
    .then(() => riskyOperation())
    .catch((error) => {
      // Log error but don't break the chain
      console.error('Operation failed:', error);
      return null; // Return fallback value
    });
}
```

## Testing Microtasks

```javascript
const assert = require('assert');

// Test microtask execution order
function testMicrotaskOrder() {
  const events = [];

  events.push('start');

  process.nextTick(() => events.push('nextTick'));
  Promise.resolve().then(() => events.push('promise'));
  setImmediate(() => {
    events.push('setImmediate');

    assert.deepEqual(events, [
      'start',
      'sync',
      'nextTick',
      'promise',
      'setImmediate',
    ]);
  });

  events.push('sync');
}
```

## Hands-on Exercises

1. **Execution Order Analysis**

   - Create complex microtask/macrotask scenarios
   - Predict and verify execution order
   - Understand timing implications

2. **Performance Optimization**

   - Identify microtask bottlenecks
   - Implement batching strategies
   - Measure performance improvements

3. **Custom Scheduler**
   - Build a priority-based task scheduler
   - Use microtasks for high-priority work
   - Integrate with Node.js event loop

## Next Steps

- [Promise Implementation](../10-promises/README.md)
- [Performance Optimization](../15-optimization/README.md)
- [Profiling and Debugging](../12-profiling-debugging/README.md)
