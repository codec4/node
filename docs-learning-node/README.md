# V8 Engine Internals for Node.js Engineers

A comprehensive learning guide to understand the V8 JavaScript engine from a Node.js perspective.

## Table of Contents

### Part I: Foundations

1. [V8 Architecture Overview](01-v8-architecture/README.md)

   - What is V8 and how it fits into Node.js
   - Isolates, Contexts, and Execution Model
   - Memory Layout and Management
   - Compilation Pipeline Overview

2. [Setting Up the Development Environment](02-development-setup/README.md)
   - Building V8 from source
   - Debugging tools and techniques
   - Profiling and performance analysis
   - Integration with Node.js development

### Part II: Core Concepts

3. [Isolates and Context Management](03-isolates-context/README.md)

   - Understanding V8 Isolates
   - Context creation and management
   - Security boundaries and realms
   - Worker threads and Isolate sharing

4. [Memory Management and Garbage Collection](04-memory-gc/README.md)

   - Heap organization (Young/Old generation)
   - Garbage collection algorithms
   - Write barriers and concurrent marking
   - Memory pressure and Node.js applications
   - Heap snapshots and memory leak detection

5. [JavaScript Compilation Pipeline](05-compilation/README.md)
   - Ignition interpreter
   - Baseline compiler
   - Maglev mid-tier compiler
   - TurboFan optimizing compiler
   - Deoptimization and feedback loops

### Part III: Performance-Critical APIs

6. [Fast API Calls](06-fast-api/README.md)

   - What are Fast API calls
   - Performance benefits over traditional callbacks
   - Implementation patterns
   - Node.js built-in usage examples
   - Creating your own Fast APIs

7. [Function and Object Templates](07-templates/README.md)

   - Creating JavaScript constructors from C++
   - Property accessors and method bindings
   - Instance templates vs prototype templates
   - Node.js built-in object implementation

8. [ArrayBuffers and Typed Arrays](08-arraybuffers/README.md)
   - Memory management for binary data
   - Zero-copy operations
   - Node.js Buffer implementation
   - Shared array buffers and worker threads

### Part IV: Asynchronous Execution

9. [Microtask Queue and Event Loop Integration](09-microtasks/README.md)

   - Microtask vs macrotask scheduling
   - Promise resolution timing
   - Integration with Node.js event loop
   - Custom microtask management

10. [Promise Implementation](10-promises/README.md)
    - V8's Promise implementation details
    - Async/await transformation
    - Error handling and stack traces
    - Performance characteristics

### Part V: Advanced Topics

11. [WebAssembly Integration](11-wasm/README.md)

    - WASM module loading and compilation
    - JavaScript/WASM interop
    - Memory management considerations
    - Node.js WASM ecosystem

12. [Profiling and Debugging](12-profiling-debugging/README.md)

    - CPU profiling APIs
    - Heap analysis and memory profiling
    - Inspector protocol integration
    - Performance debugging techniques

13. [Security and Sandboxing](13-security/README.md)
    - V8 security model
    - Context isolation
    - Embedder security considerations
    - Node.js security implications

### Part VI: Practical Applications

14. [Writing Efficient Native Addons](14-native-addons/README.md)

    - N-API vs direct V8 API usage
    - Performance optimization techniques
    - Memory management best practices
    - Error handling and exception safety

15. [Performance Optimization Strategies](15-optimization/README.md)

    - Understanding V8 optimization hints
    - Avoiding deoptimization triggers
    - Memory allocation patterns
    - Benchmarking and measurement

16. [Real-World Case Studies](16-case-studies/README.md)
    - Node.js core modules analysis
    - Popular npm packages optimization
    - Production performance debugging
    - Scaling considerations

### Part VII: Tools and Utilities

17. [V8 Inspector and DevTools](17-inspector/README.md)

    - Inspector protocol deep dive
    - Custom debugging tools
    - Remote debugging setup
    - Performance analysis workflows

18. [Command-line Tools and Utilities](18-tools/README.md)
    - d8 shell usage
    - V8 flags and options
    - Heap analysis tools
    - Code generation inspection

### Appendices

A. [V8 API Reference Quick Guide](appendix-a-api-reference/README.md)
B. [Performance Flags and Options](appendix-b-flags/README.md)
C. [Troubleshooting Common Issues](appendix-c-troubleshooting/README.md)
D. [Further Reading and Resources](appendix-d-resources/README.md)

---

## How to Use This Guide

This guide is designed to be read progressively, but you can also jump to specific sections based on your interests:

- **Beginners**: Start with Part I and II to build foundational knowledge
- **Performance Engineers**: Focus on Parts III, V, and VI
- **Addon Developers**: Emphasize Parts III, VI, and VII
- **Platform Engineers**: Pay special attention to Parts II, IV, and V

Each section includes:

- Theoretical background
- Practical examples
- Hands-on exercises
- References to Node.js source code
- Performance tips and gotchas

## Prerequisites

- Solid understanding of JavaScript and Node.js
- Basic C++ knowledge (helpful but not required)
- Familiarity with system programming concepts
- Node.js development experience

## Contributing

This guide is a living document. Contributions, corrections, and improvements are welcome!
