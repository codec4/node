# Fast API Calls

## Learning Objectives

- Understand the performance benefits of Fast API calls
- Learn how to implement Fast API methods
- Explore Node.js usage patterns
- Build your own Fast API bindings

## What are Fast API Calls?

Fast API calls allow C++ functions to be called directly from optimized JavaScript code, bypassing the traditional callback overhead. This is one of V8's most important performance features for embedders like Node.js.

### Traditional vs Fast API Flow

**Traditional Callback:**

```
JS optimized code → deopt → slow path → callback setup → C++ function
```

**Fast API:**

```
JS optimized code → direct call → C++ function
```

## Key Benefits

1. **Zero allocation** during the call
2. **No deoptimization** of JavaScript code
3. **Direct register-level** parameter passing
4. **Type checks done at compile time**

## Implementation Requirements

Fast API functions must follow strict rules:

- No JavaScript heap allocation
- No JavaScript execution trigger
- Limited parameter and return types
- Idempotent (safe to retry)

## Example Implementation

### Basic Fast Method

```cpp
// Fast path - called from optimized JS
void FastAdd(int a, int b, v8::FastApiCallbackOptions& options) {
  // Simple computation only
  return a + b;
}

// Slow path - fallback
void SlowAdd(const v8::FunctionCallbackInfo<v8::Value>& args) {
  // Full error checking and allocation
  v8::Isolate* isolate = args.GetIsolate();
  if (args.Length() < 2) {
    isolate->ThrowException(v8::String::NewFromUtf8Literal(isolate, "Requires 2 arguments"));
    return;
  }
  // ... type checking and conversion
}

// Setup
v8::CFunction c_func = v8::CFunction::MakeWithFallbackSupport(FastAdd);
v8::Local<v8::FunctionTemplate> tmpl = v8::FunctionTemplate::New(
    isolate, SlowAdd, {}, {}, 0, v8::ConstructorBehavior::kThrow,
    v8::SideEffectType::kHasNoSideEffect, &c_func);
```

### Fallback Support

```cpp
void FastMethodWithFallback(int param, v8::FastApiCallbackOptions& options) {
  if (param < 0) {
    // Request fallback to slow path for error handling
    options.fallback = true;
    return;
  }
  // Continue with fast path...
}
```

## Node.js Usage Examples

### File System Operations

Node.js uses Fast API for:

- `fs.readFileSync()` - direct buffer operations
- `fs.statSync()` - system call results
- Path manipulation functions

### Crypto Operations

- Hash computations
- Cipher operations
- Random number generation

### Buffer Operations

- Memory copying
- Data transformation
- Encoding/decoding

## Supported Types

### Parameters

- Primitive types: `bool`, `int32_t`, `uint32_t`, `int64_t`, `float`, `double`
- V8 objects: `v8::Local<v8::Object>`, `v8::Local<v8::ArrayBuffer>`
- Typed arrays: `v8::Local<v8::TypedArray>`
- Embedder types via internal fields

### Return Types

- `void`, `bool`, `int32_t`, `uint32_t`, `float`, `double`
- V8 objects (with care)

## Performance Considerations

### When to Use Fast API

- ✅ Computational heavy functions
- ✅ Frequent calls from hot code
- ✅ Simple parameter/return types
- ✅ No error conditions in common case

### When NOT to Use

- ❌ Functions that allocate frequently
- ❌ Complex error handling required
- ❌ Infrequent calls
- ❌ Complex object manipulation

## Debugging Fast API

### Trace Fast API Calls

```bash
node --trace-turbo-fast-api-calls script.js
```

### Verify Optimization

```bash
node --trace-opt --trace-deopt script.js
```

## Hands-on Exercise

Create a simple Fast API method:

1. Implement a fast mathematical operation
2. Add fallback support for edge cases
3. Benchmark against traditional callback
4. Trace optimization behavior

## Real-World Impact

Fast API calls can provide:

- 2-10x performance improvement for computational tasks
- Reduced GC pressure
- Better predictable performance
- Lower CPU usage

## Next Steps

- [Function Templates](../07-templates/README.md)
- [Performance Optimization](../15-optimization/README.md)
- [Native Addons](../14-native-addons/README.md)
