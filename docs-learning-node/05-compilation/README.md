# JavaScript Compilation Pipeline

## Overview

V8's compilation pipeline is a sophisticated multi-tier system designed to balance fast startup times with peak performance. Understanding how JavaScript code transforms from source text into optimized machine code is crucial for writing high-performance Node.js applications.

This lesson explores the complete journey from source code to optimized execution, covering all compilation tiers and the feedback-driven optimization system.

## Table of Contents

1. [Pipeline Architecture](#pipeline-architecture)
2. [Ignition Interpreter](#ignition-interpreter)
3. [Sparkplug Baseline Compiler](#sparkplug-baseline-compiler)
4. [Maglev Mid-Tier Compiler](#maglev-mid-tier-compiler)
5. [TurboFan Optimizing Compiler](#turbofan-optimizing-compiler)
6. [Deoptimization and Feedback Loops](#deoptimization-and-feedback-loops)
7. [Practical Examples](#practical-examples)
8. [Best Practices](#best-practices)

---

## Pipeline Architecture

### Multi-Tier Compilation Strategy

V8 uses a tiered compilation approach that progressively optimizes hot code while keeping startup fast:

```
┌──────────────────────────────────────────────────────────────┐
│                   V8 Compilation Pipeline                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  Source Code                                                 │
│       ↓                                                     │
│  ┌─────────┐                                                 │
│  │ Parser  │ → AST (Abstract Syntax Tree)                   │
│  └─────────┘                                                 │
│       ↓                                                     │
│  ┌──────────────┐                                            │
│  │   Ignition   │ → Bytecode + Feedback Vector              |
│  │ (Interpreter)│   ⟨Tier 0⟩                                 │
│  └──────────────┘                                            │
│       ↓ (after ~50-100 calls)                               │
│  ┌──────────────┐                                            │
│  │  Sparkplug   │ → Baseline Machine Code                   │
│  │  (Baseline)  │   ⟨Tier 1⟩                                 │
│  └──────────────┘                                            │
│       ↓ (if still hot)                                      │
│  ┌──────────────┐                                            │
│  │   Maglev     │ → Optimized Code (fast compile)           │
│  │  (Mid-Tier)  │   ⟨Tier 2⟩                                 │
│  └──────────────┘                                            │
│       ↓ (if very hot)                                       │
│  ┌──────────────┐                                            │
│  │  TurboFan    │ → Highly Optimized Machine Code           │
│  │ (Optimizing) │   ⟨Tier 3⟩                                 │
│  └──────────────┘                                            │
│       ↓                                                     │
│  Native Execution                                            │
│                                                              │
│  ← Deoptimization (if assumptions break)                    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Design Philosophy

**The Generational Hypothesis of Code:**

- Most code runs only a few times (cold code)
- Small percentage runs frequently (hot code)
- Different code needs different optimization strategies

**Tier Selection Criteria:**

| Tier      | Compilation Speed | Execution Speed | Use Case                        |
| --------- | ----------------- | --------------- | ------------------------------- |
| Ignition  | Instant           | Slow            | One-time code, startup          |
| Sparkplug | Very Fast         | Medium          | Warm code, quick boost          |
| Maglev    | Fast              | Fast            | Hot code, balanced optimization |
| TurboFan  | Slow              | Very Fast       | Very hot code, peak performance |

### V8 Source References

```cpp
// deps/v8/src/common/globals.h - Tiering states
enum class TieringState : uint8_t {
  kNone = 0b000,
  kInProgress = 0b001,
  kRequestMaglev_Synchronous = 0b010,
  kRequestMaglev_Concurrent = 0b011,
  kRequestTurbofan_Synchronous = 0b100,
  kRequestTurbofan_Concurrent = 0b101,
};
```

---

## Ignition Interpreter

### What is Ignition?

Ignition is V8's bytecode interpreter, introduced to replace the previous full-codegen compiler. It generates compact bytecode that can be executed immediately, enabling fast startup.

### Architecture

```
JavaScript Source → Parser → AST → Bytecode Generator → Bytecode
                                          ↓
                                   Feedback Vector
```

**Key Components:**

1. **Parser**: Converts source text to Abstract Syntax Tree (AST)
2. **Bytecode Generator**: Transforms AST into bytecode instructions
3. **Bytecode Dispatcher**: Executes bytecode in the interpreter loop
4. **Feedback Vector**: Collects runtime type information

### Bytecode Format

V8's bytecode is a register-based instruction set (unlike stack-based Java bytecode):

```javascript
// JavaScript function
function add(a, b) {
  return a + b;
}

// Bytecode representation (simplified)
/*
CreateFunctionContext [0]   // Create execution context
Ldar a0                      // Load argument 0 (a) into accumulator
Add a1, [0]                  // Add argument 1 (b) to accumulator
                             // [0] is feedback slot for type info
Return                       // Return accumulator value
*/
```

### Complete Example with Node.js

```javascript
// examine-bytecode.js
const v8 = require('v8');
const vm = require('vm');

// Enable bytecode printing (requires --print-bytecode flag)
function examineBytecode(code, functionName) {
  // Compile the function
  const script = new vm.Script(code);
  script.runInThisContext();

  // Note: Actual bytecode printing requires V8 flags
  console.log('Function compiled to bytecode');
  console.log('Run with: node --print-bytecode examine-bytecode.js');
}

const code = `
function multiply(x, y) {
  const result = x * y;
  return result;
}
multiply;
`;

examineBytecode(code, 'multiply');
```

Run with bytecode printing:

```bash
node --print-bytecode examine-bytecode.js
```

**Actual bytecode output:**

```
[generated bytecode for function multiply (0x...)]
Parameter count 3
Register count 1
Frame size 8
   0x... @    0 : 25 02             Ldar a1         // Load y into accumulator
   0x... @    2 : 34 03 00          Mul a0, [0]     // Multiply by x, feedback slot 0
   0x... @    5 : 26 fb             Star r0         // Store to local variable 'result'
   0x... @    7 : 25 fb             Ldar r0         // Load result
   0x... @    9 : a9                Return          // Return
```

### Bytecode Instruction Categories

**1. Load/Store Operations:**

```
Ldar  - Load accumulator from register
Star  - Store accumulator to register
LdaSmi - Load small integer
LdaConstant - Load constant from pool
```

**2. Arithmetic Operations:**

```
Add   - Addition
Sub   - Subtraction
Mul   - Multiplication
Div   - Division
```

**3. Control Flow:**

```
Jump         - Unconditional jump
JumpIfTrue   - Conditional jump
Return       - Return from function
```

**4. Object Operations:**

```
CreateObjectLiteral - Create object
GetNamedProperty    - Property access
SetNamedProperty    - Property assignment
```

### Feedback Vector

The feedback vector is crucial for future optimization:

```javascript
function processUser(user) {
  return user.name; // Feedback collected here
}

// First call - polymorphic
processUser({ name: 'Alice', age: 30 });

// Second call - learns object shape (hidden class)
processUser({ name: 'Bob', age: 25 });

// V8 records: "users have {name, age} shape"
// This feedback guides Sparkplug and TurboFan optimizations
```

### V8 Implementation Details

```cpp
// deps/v8/src/interpreter/interpreter.h
class Interpreter {
 public:
  // Creates a compilation job which will generate bytecode
  static std::unique_ptr<UnoptimizedCompilationJob> NewCompilationJob(
      ParseInfo* parse_info, FunctionLiteral* literal,
      Handle<Script> script,
      AccountingAllocator* allocator,
      std::vector<FunctionLiteral*>* eager_inner_literals,
      LocalIsolate* local_isolate);

  // Get handler for specific bytecode
  Tagged<Code> GetBytecodeHandler(Bytecode bytecode,
                                 OperandScale operand_scale);
};

// deps/v8/src/interpreter/bytecode-generator.cc
// Generates bytecode from AST nodes
```

### Performance Characteristics

**Advantages:**

- **Instant compilation**: No warm-up delay
- **Compact representation**: ~25-30% smaller than machine code
- **Memory efficient**: Critical for resource-constrained environments
- **Portable**: Same bytecode on all architectures

**Limitations:**

- **Slower execution**: ~10-100x slower than optimized code
- **Interpretation overhead**: Each instruction needs dispatch
- **Limited optimizations**: No inlining, loop optimizations, etc.

---

## Sparkplug Baseline Compiler

### What is Sparkplug?

Introduced in V8 v9.0 (Node.js 16+), Sparkplug is a super-fast baseline compiler that bridges the gap between Ignition and Maglev/TurboFan.

### Key Innovation

**Bytecode to Machine Code Translation:**

Sparkplug doesn't analyze or optimize - it performs a nearly 1:1 translation of bytecode to machine code.

```
Ignition Bytecode:              Sparkplug Machine Code:
Ldar a0          →             mov rax, [rbp + a0_offset]
Add a1, [0]      →             add rax, [rbp + a1_offset]
Return           →             ret
```

### Why Sparkplug Exists

**The Performance Gap:**

- Ignition: Interpreted (~100x slower than native)
- TurboFan: Optimized but slow to compile (~100ms)
- Sparkplug: Fast compilation (~1ms) + decent performance (~2-5x faster than Ignition)

### Architecture

```
Bytecode Array → Sparkplug Compiler → Machine Code
                        ↓
                  (Direct translation, minimal work)
```

### Compilation Trigger

```javascript
// Sparkplug compiles after ~50-100 function calls
function calculateTotal(items) {
  let total = 0;
  for (const item of items) {
    total += item.price;
  }
  return total;
}

// Calls 1-50: Ignition interpreter
for (let i = 0; i < 50; i++) {
  calculateTotal([{ price: 10 }, { price: 20 }]);
}

// Call 51+: Sparkplug baseline code (automatic)
// Much faster, no changes to your code needed
```

### Implementation Reference

```cpp
// deps/v8/src/baseline/baseline.h
bool CanCompileWithBaseline(Isolate* isolate,
                           Tagged<SharedFunctionInfo> shared);

MaybeDirectHandle<Code> GenerateBaselineCode(
    Isolate* isolate,
    Handle<SharedFunctionInfo> shared);
```

### Performance Benefits

**Real-World Impact:**

```javascript
// Benchmark: Array operations
const { performance } = require('perf_hooks');

function sumArray(arr) {
  let sum = 0;
  for (let i = 0; i < arr.length; i++) {
    sum += arr[i];
  }
  return sum;
}

const data = Array.from({ length: 10000 }, (_, i) => i);

// Warm up for Sparkplug
for (let i = 0; i < 100; i++) {
  sumArray(data);
}

// Measure with Sparkplug baseline code
const start = performance.now();
for (let i = 0; i < 1000; i++) {
  sumArray(data);
}
const end = performance.now();

console.log(`Time: ${end - start}ms`);
// Sparkplug: ~2-3x faster than pure Ignition
```

### Characteristics

**Pros:**

- ✅ Very fast compilation (~1ms for typical function)
- ✅ Significant speedup over Ignition (2-5x)
- ✅ Low memory overhead
- ✅ Doesn't block main thread

**Cons:**

- ❌ No optimizations (inlining, escape analysis, etc.)
- ❌ Still uses feedback vector (runtime type checks)
- ❌ Not as fast as optimized code

### Disabling Sparkplug (for testing)

```bash
# Disable baseline compiler
node --no-sparkplug app.js

# Enable with explicit flag
node --sparkplug app.js
```

---

## Maglev Mid-Tier Compiler

### What is Maglev?

Maglev is V8's newest tier, introduced in V8 v11.0 (Node.js 20+). It's a fast optimizing compiler that sits between Sparkplug and TurboFan.

### Design Goals

1. **Fast compilation**: ~5-10x faster than TurboFan
2. **Good performance**: ~80-90% of TurboFan performance
3. **Reduce compilation latency**: Less jank in applications

```
┌──────────────────────────────────────────────┐
│  Compilation Time vs Execution Speed         │
├──────────────────────────────────────────────┤
│                                              │
│  Fast Compile                                │
│  Ignition    ●                               │
│  Sparkplug      ●                            │
│  Maglev            ●                         │
│  TurboFan                         ●          │
│  Slow Compile                                │
│                                              │
│              Slow ← Speed → Fast             │
│                                              │
└──────────────────────────────────────────────┘
```

### Key Features

**1. Graph-Based IR (Intermediate Representation):**

Maglev builds a simplified SSA (Static Single Assignment) graph:

```javascript
function compute(x, y) {
  const a = x + 1;
  const b = y + 2;
  return a * b;
}

// Maglev IR (conceptual):
// v1 = Parameter(0)  // x
// v2 = Parameter(1)  // y
// v3 = Int32Add(v1, #1)
// v4 = Int32Add(v2, #2)
// v5 = Int32Mul(v3, v4)
// Return(v5)
```

**2. Speculative Optimizations:**

Maglev uses feedback but less aggressively than TurboFan:

```javascript
function processValue(val) {
  // Maglev observes: val is always a number
  return val * 2;
}

// After profiling:
for (let i = 0; i < 1000; i++) {
  processValue(i); // Always numbers
}

// Maglev generates specialized code for numbers
// With deoptimization if assumption breaks
```

**3. Fast Optimizations:**

- Type specialization (numbers, strings, objects)
- Redundant load elimination
- Basic inlining (small functions only)
- Dead code elimination

### When Maglev Compiles

```javascript
const v8 = require('v8');

// Maglev typically kicks in after:
// 1. Function is "hot" (called many times)
// 2. Has stable feedback (consistent types)
// 3. Sparkplug code is already running

function hotFunction(n) {
  let sum = 0;
  for (let i = 0; i < n; i++) {
    sum += Math.sqrt(i);
  }
  return sum;
}

// Warm up through tiers
for (let i = 0; i < 100; i++) {
  // Ignition
  hotFunction(100);
}

for (let i = 0; i < 1000; i++) {
  // Sparkplug
  hotFunction(100);
}

// Eventually Maglev (if enabled and hot enough)
for (let i = 0; i < 10000; i++) {
  hotFunction(100);
}
```

### V8 Implementation

```cpp
// deps/v8/src/maglev/maglev-compiler.h
class MaglevCompiler : public AllStatic {
 public:
  // May be called from any thread (concurrent compilation)
  static bool Compile(LocalIsolate* local_isolate,
                     MaglevCompilationInfo* compilation_info);

  // Generate executable code
  static std::pair<MaybeHandle<Code>, BailoutReason> GenerateCode(
      Isolate* isolate,
      MaglevCompilationInfo* compilation_info);
};
```

### Configuration Flags

```bash
# Enable Maglev (default in Node.js 20+)
node --maglev app.js

# Disable Maglev
node --no-maglev app.js

# Force early Maglev compilation (testing only)
node --maglev --stress-maglev app.js
```

### Performance Impact

**Benchmark Example:**

```javascript
// maglev-benchmark.js
const { performance } = require('perf_hooks');

function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

// Warm up to Maglev
for (let i = 0; i < 1000; i++) {
  fibonacci(10);
}

// Measure
const iterations = 1000;
const start = performance.now();
for (let i = 0; i < iterations; i++) {
  fibonacci(15);
}
const elapsed = performance.now() - start;

console.log(`Maglev time: ${elapsed.toFixed(2)}ms`);
// Compare with --no-maglev flag
```

**Typical Results:**

- ~2-3x faster than Sparkplug
- ~10-20x faster than Ignition
- ~10-20% slower than TurboFan
- ~10x faster compilation than TurboFan

---

## TurboFan Optimizing Compiler

### What is TurboFan?

TurboFan is V8's top-tier optimizing compiler, capable of generating highly optimized machine code comparable to C++ compilers.

### Architecture

```
Bytecode + Feedback → Graph Builder → Optimization Passes → Code Generator
                           ↓                  ↓                    ↓
                        Sea of Nodes      Inlining           Machine Code
                        (SSA Graph)       Type Spec.
                                         Escape Analysis
                                         Loop Opts.
```

### Compilation Pipeline

**1. Graph Construction:**

```javascript
function example(arr) {
  let sum = 0;
  for (let i = 0; i < arr.length; i++) {
    sum += arr[i];
  }
  return sum;
}

// TurboFan builds a "Sea of Nodes" graph
// Nodes represent operations, edges represent data flow
```

**Conceptual Graph:**

```
Parameter(arr) ──→ LoadLength ──→ Phi(loop counter)
                        ↓              ↓
                   Compare ──→ Branch(loop condition)
                        ↓              ↓
                  LoadElement    Add(accumulator)
```

**2. Optimization Phases:**

TurboFan applies dozens of optimization passes:

```
Type Inference → Inlining → Escape Analysis →
Loop Optimizations → Redundancy Elimination →
Register Allocation → Code Generation
```

### Key Optimizations

#### 1. Function Inlining

```javascript
// Original code
function square(x) {
  return x * x;
}

function sumOfSquares(arr) {
  let total = 0;
  for (let i = 0; i < arr.length; i++) {
    total += square(arr[i]); // Function call
  }
  return total;
}

// TurboFan optimized version (conceptual):
function sumOfSquares_optimized(arr) {
  let total = 0;
  for (let i = 0; i < arr.length; i++) {
    // Inlined: no function call overhead
    const x = arr[i];
    total += x * x;
  }
  return total;
}
```

#### 2. Escape Analysis

```javascript
function createPoint(x, y) {
  const point = { x, y }; // Object allocation
  return point.x + point.y;
}

// TurboFan optimization:
// Realizes 'point' doesn't escape the function
// Eliminates allocation, uses scalar values instead
function createPoint_optimized(x, y) {
  // No object allocated!
  return x + y;
}
```

#### 3. Type Specialization

```javascript
function add(a, b) {
  return a + b;
}

// After profiling with numbers only:
add(5, 10);
add(20, 30);

// TurboFan generates specialized code:
// - Direct integer/float addition
// - No type checks
// - Fast machine code
```

#### 4. Loop Optimizations

```javascript
function processArray(arr) {
  const results = [];
  for (let i = 0; i < arr.length; i++) {
    results[i] = arr[i] * 2 + 1;
  }
  return results;
}

// TurboFan optimizations:
// - Loop unrolling
// - Bounds check elimination (if safe)
// - SIMD vectorization (on supported platforms)
// - Loop-invariant code motion
```

### V8 Implementation

```cpp
// deps/v8/src/compiler/turbofan.h
namespace compiler {

V8_EXPORT_PRIVATE std::unique_ptr<TurbofanCompilationJob>
NewCompilationJob(
    Isolate* isolate,
    Handle<JSFunction> function,
    IsScriptAvailable has_script,
    BytecodeOffset osr_offset = BytecodeOffset::None());

} // namespace compiler
```

### OSR (On-Stack Replacement)

TurboFan can optimize already-running functions:

```javascript
function longLoop() {
  let sum = 0;
  for (let i = 0; i < 1000000; i++) {
    // First iterations: Ignition/Sparkplug
    // Mid-loop: TurboFan compiles in background
    // Later iterations: Replaces to optimized code
    sum += Math.sqrt(i);
  }
  return sum;
}

// OSR happens mid-execution!
longLoop();
```

### Measuring TurboFan Impact

```javascript
// turbofan-test.js
const { performance } = require('perf_hooks');

function computeIntensive(n) {
  let result = 0;
  for (let i = 0; i < n; i++) {
    result += Math.sin(i) * Math.cos(i);
  }
  return result;
}

// Cold run (Ignition)
const cold = performance.now();
computeIntensive(10000);
console.log('Cold:', performance.now() - cold, 'ms');

// Warm up to TurboFan
for (let i = 0; i < 10000; i++) {
  computeIntensive(100);
}

// Hot run (TurboFan)
const hot = performance.now();
computeIntensive(10000);
console.log('Hot:', performance.now() - hot, 'ms');

// Typical: 10-100x speedup
```

### Forcing Optimization (Testing Only)

```javascript
// Requires --allow-natives-syntax flag
function testFunction(x) {
  return x * x + x;
}

// Prepare for optimization
%PrepareFunctionForOptimization(testFunction);

// Call once to gather feedback
testFunction(5);

// Force optimization
%OptimizeFunctionOnNextCall(testFunction);

// Now optimized
testFunction(10);

// Check optimization status
console.log(%GetOptimizationStatus(testFunction));
```

Run with:

```bash
node --allow-natives-syntax turbofan-test.js
```

### TurboFan Compilation Time

```javascript
// Compilation can take 10-500ms for complex functions
// V8 compiles in background (concurrent compilation)

function complexFunction(data) {
  // Lots of operations, branches, calls...
  return data
    .filter((x) => x > 0)
    .map((x) => x * 2)
    .reduce((a, b) => a + b, 0);
}

// First calls: Ignition/Sparkplug (instant)
// Background: TurboFan compiles (50-100ms)
// Later calls: Optimized code (very fast)
```

---

## Deoptimization and Feedback Loops

### What is Deoptimization?

Deoptimization is the process of reverting optimized code back to bytecode when assumptions made during optimization are violated.

### Why Deoptimization Happens

```
Optimization = Assumptions + Fast Code

If Assumptions Break → Must Deoptimize
```

### Common Deoptimization Triggers

#### 1. Type Changes

```javascript
function add(a, b) {
  return a + b;
}

// Phase 1: Training
for (let i = 0; i < 10000; i++) {
  add(i, i + 1); // Always numbers
}
// TurboFan optimizes for numbers

// Phase 2: Deopt trigger
add('hello', 'world'); // String! Assumptions violated
// Deoptimizes back to Ignition

// Phase 3: Re-optimization
// V8 may recompile with polymorphic handling
```

#### 2. Hidden Class Changes

```javascript
function processUser(user) {
  return user.name + ' ' + user.email;
}

// Consistent shape
const user1 = { name: 'Alice', email: 'alice@example.com' };
const user2 = { name: 'Bob', email: 'bob@example.com' };

for (let i = 0; i < 10000; i++) {
  processUser(i % 2 ? user1 : user2);
}
// Optimized for this exact object shape

// Different shape: deopt!
const user3 = { email: 'charlie@example.com', name: 'Charlie' };
processUser(user3); // Properties in different order
```

#### 3. Prototype Chain Changes

```javascript
function getValue(obj) {
  return obj.value;
}

const proto = { value: 42 };
const obj = Object.create(proto);

// Optimize assuming stable prototype chain
for (let i = 0; i < 10000; i++) {
  getValue(obj);
}

// Modify prototype: deopt!
proto.value = 100; // Prototype changed
getValue(obj);
```

### Deoptimization Process

```
┌────────────────────────────────────────────┐
│  Running Optimized Code (TurboFan/Maglev) │
├────────────────────────────────────────────┤
│  ↓                                         │
│  Check fails (type guard, map check, etc.) │
│  ↓                                         │
│  Trigger Deoptimization                    │
│  ↓                                         │
│  1. Save current state (registers, stack)  │
│  2. Find corresponding bytecode position   │
│  3. Reconstruct interpreter frame          │
│  4. Transfer execution to Ignition         │
│  5. Mark function for re-compilation       │
│  ↓                                         │
│  Continue in Ignition                      │
└────────────────────────────────────────────┘
```

### V8 Implementation

```cpp
// deps/v8/src/builtins/builtins-definitions.h
// Deoptimization entry points
ASM(DeoptimizationEntry_Eager, DeoptimizationEntry)
ASM(DeoptimizationEntry_Lazy, DeoptimizationEntry)
```

### Deoptimization Types

**1. Eager Deoptimization:**

```javascript
function strictCheck(x) {
  // Assumes x is always a Smi (small integer)
  return x + 1;
}

// Optimized for Smi
strictCheck(42);

// Not a Smi: immediate deopt!
strictCheck(2.5);
```

**2. Lazy Deoptimization:**

```javascript
function callHelper(callback) {
  // Assumes callback is always the same function
  return callback();
}

// Optimized assuming callback = foo
function foo() {
  return 1;
}
callHelper(foo);

// Different callback: lazy deopt at call site
function bar() {
  return 2;
}
callHelper(bar);
```

### Feedback Loops

The feedback vector is the communication channel between tiers:

```
┌───────────────────────────────────────────────────┐
│              Feedback Collection                  │
├───────────────────────────────────────────────────┤
│                                                   │
│  Ignition/Sparkplug Execution                     │
│         ↓                                         │
│  Collect type info, call counts, IC states        │
│         ↓                                         │
│  Store in Feedback Vector                         │
│         ↓                                         │
│  ┌─────────────────────┐                          │
│  │  Feedback Vector    │                          │
│  │  - Call counts      │                          │
│  │  - Type feedback    │                          │
│  │  - IC states        │                          │
│  │  - Profiler data    │                          │
│  └─────────────────────┘                          │
│         ↓                                         │
│  Maglev/TurboFan read feedback                    │
│         ↓                                         │
│  Make optimization decisions                      │
│         ↓                                         │
│  Generate optimized code                          │
│         ↓                                         │
│  Runtime checks validate assumptions              │
│         ↓                                         │
│  ✓ Correct → Fast execution                       │
│  ✗ Wrong → Deoptimization → Update feedback       │
│         ↓                                         │
│  Potential re-optimization with new feedback      │
│                                                   │
└───────────────────────────────────────────────────┘
```

### Inline Caches (ICs)

Inline caches are a key component of the feedback system:

```javascript
function getProperty(obj) {
  return obj.name; // IC here
}

// First call: Uninitialized IC
const obj1 = { name: 'Alice', age: 30 };
getProperty(obj1); // IC: Monomorphic (one shape seen)

// Second call: Same shape
const obj2 = { name: 'Bob', age: 25 };
getProperty(obj2); // IC: Still monomorphic

// Third call: Different shape
const obj3 = { name: 'Charlie', role: 'Admin' };
getProperty(obj3); // IC: Polymorphic (2 shapes)

// Many different shapes
getProperty({ name: 'Dave' });
getProperty({ age: 40, name: 'Eve' });
// IC: Megamorphic (too many shapes)
```

**IC States:**

```
Uninitialized → Monomorphic → Polymorphic → Megamorphic
   (0 shapes)    (1 shape)     (2-4 shapes)   (5+ shapes)
      ↓             ↓              ↓              ↓
   Slowest       Fastest        Fast          Slow
```

### Avoiding Deoptimization

```javascript
// ❌ Bad: Type instability
function process(value) {
  return value * 2;
}
process(10); // Number
process('20'); // String - deopt!

// ✅ Good: Consistent types
function processNumber(value) {
  if (typeof value !== 'number') {
    throw new TypeError('Expected number');
  }
  return value * 2;
}

// ❌ Bad: Object shape changes
function createUser(name, email) {
  const user = { name };
  user.email = email; // Shape changes
  return user;
}

// ✅ Good: Consistent shape
function createUser(name, email) {
  return { name, email }; // All properties defined at once
}

// ❌ Bad: Polymorphic access
function getName(obj) {
  return obj.name;
}
getName({ name: 'A', age: 1 });
getName({ age: 2, name: 'B' }); // Different shape
getName({ name: 'C', role: 'X' }); // Another shape

// ✅ Good: Monomorphic access
class User {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }
}
function getUserName(user) {
  return user.name;
}
getUserName(new User('A', 1));
getUserName(new User('B', 2)); // Same shape
```

### Monitoring Deoptimizations

```javascript
// Run with trace flags
// node --trace-deopt --trace-opt app.js

function mayDeopt(x) {
  return x + 1;
}

// Train with numbers
for (let i = 0; i < 10000; i++) {
  mayDeopt(i);
}

// Trigger deopt
mayDeopt('string');

// Console output:
// [marking mayDeopt for optimized recompilation...]
// [compiling method mayDeopt using TurboFan]
// [deoptimizing (DEOPT soft): begin ...]
// [deoptimizing (DEOPT soft): end ...]
```

### Re-optimization

V8 can re-optimize after deoptimization:

```javascript
function flexible(val) {
  return val * 2;
}

// Phase 1: Numbers only
for (let i = 0; i < 10000; i++) {
  flexible(i); // Optimized for numbers
}

// Phase 2: Mixed types
for (let i = 0; i < 10000; i++) {
  flexible(i % 2 ? i : i.toString()); // Deopt, then re-optimize
}

// V8 may generate polymorphic optimized code
// or give up and stay in Ignition
```

---

## Practical Examples

### Example 1: Observing Compilation Tiers

```javascript
// compilation-tiers.js
const { performance } = require('perf_hooks');

function compute(n) {
  let sum = 0;
  for (let i = 0; i < n; i++) {
    sum += Math.sqrt(i);
  }
  return sum;
}

function measureTier(name, iterations, workSize) {
  // Warm up
  for (let i = 0; i < iterations; i++) {
    compute(workSize);
  }

  // Measure
  const start = performance.now();
  for (let i = 0; i < 100; i++) {
    compute(workSize);
  }
  const elapsed = performance.now() - start;

  console.log(`${name}: ${elapsed.toFixed(2)}ms`);
}

console.log('=== Compilation Tier Performance ===\n');

// Tier 0: Ignition (cold)
measureTier('Ignition (cold)', 1, 1000);

// Tier 1: Sparkplug (warm)
measureTier('Sparkplug (warm)', 100, 1000);

// Tier 2: Maglev (hot)
measureTier('Maglev (hot)', 1000, 1000);

// Tier 3: TurboFan (very hot)
measureTier('TurboFan (very hot)', 10000, 1000);
```

### Example 2: Feedback-Driven Optimization

```javascript
// feedback-demo.js

class Point {
  constructor(x, y) {
    this.x = x;
    this.y = y;
  }
}

function distance(p1, p2) {
  const dx = p1.x - p2.x;
  const dy = p1.y - p2.y;
  return Math.sqrt(dx * dx + dy * dy);
}

// Scenario 1: Monomorphic (best case)
console.log('=== Monomorphic ===');
const points = [];
for (let i = 0; i < 10000; i++) {
  points.push(new Point(i, i));
}

let start = performance.now();
for (let i = 0; i < points.length - 1; i++) {
  distance(points[i], points[i + 1]);
}
console.log('Time:', performance.now() - start, 'ms');

// Scenario 2: Polymorphic (worse)
console.log('\n=== Polymorphic ===');

function distance2(p1, p2) {
  const dx = p1.x - p2.x;
  const dy = p1.y - p2.y;
  return Math.sqrt(dx * dx + dy * dy);
}

class Point3D extends Point {
  constructor(x, y, z) {
    super(x, y);
    this.z = z;
  }
}

const mixed = [];
for (let i = 0; i < 5000; i++) {
  mixed.push(new Point(i, i));
  mixed.push(new Point3D(i, i, i));
}

start = performance.now();
for (let i = 0; i < mixed.length - 1; i++) {
  distance2(mixed[i], mixed[i + 1]);
}
console.log('Time:', performance.now() - start, 'ms');
// Typically 2-3x slower due to polymorphism
```

### Example 3: Deoptimization Detection

```javascript
// deopt-detector.js
const v8 = require('v8');
const { performance } = require('perf_hooks');

function sensitiveFunction(x) {
  return x * x + x;
}

// Warm up with numbers
console.log('Warming up...');
for (let i = 0; i < 10000; i++) {
  sensitiveFunction(i);
}

// Measure optimized performance
const samples = [];
for (let i = 0; i < 100; i++) {
  const start = performance.now();
  for (let j = 0; j < 1000; j++) {
    sensitiveFunction(j);
  }
  samples.push(performance.now() - start);
}

const avgBefore = samples.reduce((a, b) => a + b) / samples.length;
console.log(`Optimized avg: ${avgBefore.toFixed(3)}ms`);

// Trigger deoptimization
console.log('\nTriggering deoptimization...');
sensitiveFunction('not a number');

// Measure after deopt
const samplesAfter = [];
for (let i = 0; i < 100; i++) {
  const start = performance.now();
  for (let j = 0; j < 1000; j++) {
    sensitiveFunction(j);
  }
  samplesAfter.push(performance.now() - start);
}

const avgAfter = samplesAfter.reduce((a, b) => a + b) / samplesAfter.length;
console.log(`Deoptimized avg: ${avgAfter.toFixed(3)}ms`);
console.log(`Slowdown: ${(avgAfter / avgBefore).toFixed(2)}x`);
```

Run with:

```bash
node --trace-deopt deopt-detector.js
```

### Example 4: Inline Cache States

```javascript
// ic-states.js

function getProperty(obj) {
  return obj.value;
}

// Monomorphic: One object shape
console.log('=== Monomorphic ===');
const shape1 = { value: 1, name: 'test' };
for (let i = 0; i < 10000; i++) {
  getProperty({ value: i, name: 'test' });
}

// Polymorphic: 2-4 shapes
console.log('=== Polymorphic ===');
function getProp2(obj) {
  return obj.value;
}

for (let i = 0; i < 10000; i++) {
  if (i % 2 === 0) {
    getProp2({ value: i, name: 'test' });
  } else {
    getProp2({ value: i, age: 30 });
  }
}

// Megamorphic: Many shapes
console.log('=== Megamorphic ===');
function getProp3(obj) {
  return obj.value;
}

for (let i = 0; i < 10000; i++) {
  const obj = { value: i };
  // Add random properties to create different shapes
  for (let j = 0; j < i % 10; j++) {
    obj[`prop${j}`] = j;
  }
  getProp3(obj);
}

console.log('Run with --trace-ic to see IC state transitions');
```

---

## Best Practices

### 1. Write Optimization-Friendly Code

```javascript
// ✅ Good: Predictable types
function calculateTotal(items) {
  let total = 0;
  for (let i = 0; i < items.length; i++) {
    total += items[i].price; // Always number
  }
  return total;
}

// ❌ Bad: Type confusion
function flexibleAdd(a, b) {
  return a + b; // Could be number or string concatenation
}
```

### 2. Maintain Object Shape Consistency

```javascript
// ✅ Good: Consistent initialization
class User {
  constructor(name, email, age = null) {
    this.name = name;
    this.email = email;
    this.age = age; // Always present, even if null
  }
}

// ❌ Bad: Shape changes
class BadUser {
  constructor(name, email) {
    this.name = name;
    this.email = email;
    // age added later in some cases
  }
}
```

### 3. Avoid Polymorphism in Hot Paths

```javascript
// ✅ Good: Separate monomorphic functions
function processNumber(n) {
  return n * 2;
}

function processString(s) {
  return s.repeat(2);
}

function process(value) {
  if (typeof value === 'number') {
    return processNumber(value);
  } else {
    return processString(value);
  }
}

// ❌ Bad: One polymorphic function
function badProcess(value) {
  if (typeof value === 'number') {
    return value * 2;
  } else {
    return value.repeat(2);
  }
}
```

### 4. Use Classes for Object Creation

```javascript
// ✅ Good: Class with fixed shape
class Point {
  constructor(x, y) {
    this.x = x;
    this.y = y;
  }
}

const p1 = new Point(1, 2);
const p2 = new Point(3, 4);
// Same hidden class

// ⚠️ Okay but less optimal: Object literals
const p3 = { x: 5, y: 6 };
const p4 = { x: 7, y: 8 };
// May have different hidden classes
```

### 5. Warm Up Critical Paths

```javascript
// For performance-critical code, ensure it's hot before measurement

function criticalOperation(data) {
  // Complex operation
  return data.reduce((acc, val) => acc + val, 0);
}

// ✅ Good: Warm up before critical use
const testData = Array.from({ length: 1000 }, (_, i) => i);

// Warm up to TurboFan
for (let i = 0; i < 10000; i++) {
  criticalOperation(testData);
}

// Now it's optimized for production use
const result = criticalOperation(largeProductionData);
```

### 6. Profile Before Optimizing

```javascript
// Use built-in profiling tools

// CPU profiling
const { Session } = require('inspector');
const fs = require('fs');

const session = new Session();
session.connect();

session.post('Profiler.enable');
session.post('Profiler.start');

// Run your code
yourFunction();

session.post('Profiler.stop', (err, { profile }) => {
  if (!err) {
    fs.writeFileSync('profile.cpuprofile', JSON.stringify(profile));
  }
  session.disconnect();
});
```

### 7. Understand Optimization Limits

```javascript
// Some patterns can't be optimized well:

// ❌ Try-catch in hot code
function withTryCatch(x) {
  try {
    return x * 2;
  } catch (e) {
    return 0;
  }
}
// TurboFan has limited optimization for try-catch

// ❌ eval/with statements
function withEval(code) {
  return eval(code); // Prevents optimization
}

// ❌ Excessive property access patterns
function deepAccess(obj) {
  return obj.a.b.c.d.e.f.g; // Hard to optimize
}
```

### 8. V8 Flags for Development

```bash
# See optimization/deoptimization events
node --trace-opt --trace-deopt app.js

# See IC state changes
node --trace-ic app.js

# Print bytecode
node --print-bytecode app.js

# Disable specific tiers (for testing)
node --no-sparkplug --no-maglev app.js

# Enable native syntax (for testing only)
node --allow-natives-syntax app.js
```

### 9. Benchmarking Best Practices

```javascript
const { performance, PerformanceObserver } = require('perf_hooks');

function benchmark(name, fn, iterations = 1000) {
  // Warm up
  for (let i = 0; i < iterations; i++) {
    fn();
  }

  // Force garbage collection (if --expose-gc)
  if (global.gc) global.gc();

  // Measure
  const start = performance.now();
  for (let i = 0; i < iterations; i++) {
    fn();
  }
  const elapsed = performance.now() - start;

  console.log(`${name}: ${elapsed.toFixed(2)}ms (${iterations} iterations)`);
  console.log(`  Average: ${(elapsed / iterations).toFixed(4)}ms per call`);
}

// Usage
benchmark('Array.map', () => {
  const arr = [1, 2, 3, 4, 5];
  arr.map((x) => x * 2);
});
```

---

## Summary

### Key Takeaways

1. **Multi-Tier Compilation**: V8 uses multiple compilation tiers (Ignition, Sparkplug, Maglev, TurboFan) to balance startup performance and peak execution speed.

2. **Feedback-Driven**: Optimization decisions are based on runtime feedback collected during execution.

3. **Speculative Optimization**: Compilers make assumptions based on observed behavior and can deoptimize if those assumptions break.

4. **Deoptimization is Normal**: It's part of the adaptive optimization strategy, not an error.

5. **Write Predictable Code**: Consistent types, stable object shapes, and monomorphic call sites lead to better optimization.

### Compilation Tier Selection

| Scenario      | Best Tier | Why                            |
| ------------- | --------- | ------------------------------ |
| One-time code | Ignition  | No compilation overhead        |
| Warm code     | Sparkplug | Fast compilation, good speedup |
| Hot code      | Maglev    | Balanced optimization          |
| Very hot code | TurboFan  | Maximum performance            |

### Performance Tips

- **Keep types stable** in hot functions
- **Initialize objects completely** to maintain hidden class stability
- **Avoid polymorphism** in performance-critical code
- **Use classes** for consistent object shapes
- **Warm up** critical paths before measurement
- **Profile** to understand actual bottlenecks

### Further Reading

- [V8 Blog: Ignition Interpreter](https://v8.dev/blog/ignition-interpreter)
- [V8 Blog: Sparkplug Baseline Compiler](https://v8.dev/blog/sparkplug)
- [V8 Blog: Maglev](https://v8.dev/blog/maglev)
- [V8 Blog: TurboFan](https://v8.dev/blog/turbofan-jit)
- [Node.js V8 Options](https://nodejs.org/api/cli.html#cli_v8_options)

---

**Next Lesson**: [Fast API Calls](../06-fast-api/README.md)
