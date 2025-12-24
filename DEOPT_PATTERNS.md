# Deoptimization Patterns - Technical Reference

## Overview

This document provides detailed technical information about common deoptimization patterns in V8, how to detect them, and how to fix them. This serves as both a reference for implementing the examples and educational content for users.

---

## Table of Contents

1. [Understanding V8 Optimization](#understanding-v8-optimization)
2. [Deoptimization Pattern #1: Hidden Class Changes](#pattern-1-hidden-class-changes)
3. [Deoptimization Pattern #2: Polymorphic Functions](#pattern-2-polymorphic-functions)
4. [Deoptimization Pattern #3: Array Operations](#pattern-3-array-operations)
5. [Deoptimization Pattern #4: Try-Catch Blocks](#pattern-4-try-catch-blocks)
6. [Deoptimization Pattern #5: Arguments Object](#pattern-5-arguments-object)
7. [Detection Techniques](#detection-techniques)
8. [V8 Flags Reference](#v8-flags-reference)
9. [Optimization Status Codes](#optimization-status-codes)

---

## Understanding V8 Optimization

### The Optimization Pipeline

```
JavaScript Code
     │
     ▼
Ignition (Interpreter) ──────┐
     │                       │ Hot code detected
     │                       │
     ▼                       ▼
Execution              TurboFan (Optimizer)
     │                       │
     │                       │ Optimized code
     │                       │
     ▼                       ▼
Slow Path ◄────────── Fast Path
     ▲                       │
     │                       │
     │  Deoptimization       │
     │  (speculation failed) │
     └───────────────────────┘
```

### Key Concepts

**Hidden Classes (Maps)**
- V8 creates hidden classes for object shapes
- Objects with the same properties in the same order share a hidden class
- Changing object shape creates new hidden classes

**Inline Caches (ICs)**
- V8 caches type information at operation sites
- Monomorphic: one type seen
- Polymorphic: 2-4 types seen
- Megamorphic: 5+ types seen (bad for performance)

**Speculation**
- TurboFan makes assumptions about types and behavior
- If assumptions are violated, code must deoptimize
- Deoptimization means going back to interpreted code

---

## Pattern #1: Hidden Class Changes

### What Causes It

1. **Inconsistent Property Initialization Order**
```javascript
// Creates different hidden classes
function Point(x, y) {
  if (Math.random() > 0.5) {
    this.x = x;
    this.y = y;
  } else {
    this.y = y;  // Different order!
    this.x = x;
  }
}
```

2. **Dynamic Property Addition**
```javascript
function User(name) {
  this.name = name;
  // Later in code:
  if (someCondition) {
    this.age = 25;  // Adds property after construction
  }
}
```

3. **Property Deletion**
```javascript
const obj = { a: 1, b: 2, c: 3 };
delete obj.b;  // Triggers hidden class change
```

### Why It's Bad

- Each hidden class change requires creating new internal structures
- Inline caches miss because object shapes don't match
- Prevents effective optimization of property access
- Memory overhead from multiple hidden classes

### How to Fix

1. **Initialize All Properties in Constructor**
```javascript
function User(name, age) {
  this.name = name;
  this.age = age || null;  // Always initialize, even if null
}
```

2. **Consistent Initialization Order**
```javascript
function Point(x, y) {
  this.x = x;  // Always same order
  this.y = y;
}
```

3. **Use Object.freeze() or Object.seal()**
```javascript
const obj = Object.freeze({ a: 1, b: 2 });
// Prevents modifications that would change shape
```

### Detection Commands

```bash
# Run with hidden class tracking
node --trace-maps your-script.js

# Check optimization status
node --allow-natives-syntax --trace-opt your-script.js
```

### Example Output
```
[creating hidden class]
[transition to new hidden class]
[property addition causes new hidden class]
```

---

## Pattern #2: Polymorphic Functions

### What Causes It

1. **Mixed Type Parameters**
```javascript
function add(a, b) {
  return a + b;
}

add(1, 2);           // Called with numbers
add("hello", " ");   // Called with strings
add({}, {});         // Called with objects
```

2. **Type-Unstable Variables**
```javascript
let value = 42;       // Initially a number
value = "string";     // Now a string
value = { obj: 1 };   // Now an object
```

3. **Mixed-Type Arrays**
```javascript
const arr = [1, 2, 3, "four", 5, null, {}];
// Contains numbers, strings, null, objects
```

### Why It's Bad

- Inline caches transition from monomorphic to polymorphic to megamorphic
- Each type requires different code path
- Prevents inlining and optimization
- Significantly slower property/method access

### IC State Transitions

```
Uninitialized
     │
     ▼
Monomorphic (1 type)        ← Fast!
     │
     ▼
Polymorphic (2-4 types)     ← OK
     │
     ▼
Megamorphic (5+ types)      ← Slow!
```

### How to Fix

1. **Type Consistency**
```javascript
function addNumbers(a, b) {
  // Only accepts numbers
  return a + b;
}

function addStrings(a, b) {
  // Only accepts strings
  return a + b;
}
```

2. **Type Guards**
```javascript
function add(a, b) {
  if (typeof a !== 'number' || typeof b !== 'number') {
    throw new TypeError('Expected numbers');
  }
  return a + b;
}
```

3. **Separate Code Paths**
```javascript
function process(value) {
  if (typeof value === 'string') {
    return processString(value);
  } else if (typeof value === 'number') {
    return processNumber(value);
  }
  // Each function handles one type
}
```

### Detection Commands

```bash
# Run with IC state tracking
node --trace-ic your-script.js

# Check for megamorphic ICs
node --trace-ic your-script.js 2>&1 | grep "MEGAMORPHIC"
```

### Example Output
```
[LoadIC in ~add at <script>:2 MONOMORPHIC]
[LoadIC in ~add at <script>:2 POLYMORPHIC]
[LoadIC in ~add at <script>:2 MEGAMORPHIC]
```

---

## Pattern #3: Array Operations

### What Causes It

1. **Holey Arrays (Sparse Arrays)**
```javascript
const arr = [];
arr[0] = 1;
arr[1000] = 2;  // Creates holes between 1 and 1000
```

2. **Mixed Element Types**
```javascript
const arr = [1, 2, 3];     // PACKED_SMI_ELEMENTS
arr.push(4.5);             // Transitions to PACKED_DOUBLE_ELEMENTS
arr.push("string");        // Transitions to PACKED_ELEMENTS
```

3. **Array Transitions**
```javascript
const arr = [1, 2, 3];     // Optimized for integers
arr[1] = undefined;        // Creates hole, transitions to slower mode
```

### Element Kinds Hierarchy

```
PACKED_SMI_ELEMENTS          ← Fastest (small integers)
     │
     ▼
PACKED_DOUBLE_ELEMENTS       ← Fast (doubles)
     │
     ▼
PACKED_ELEMENTS              ← Slower (mixed types)
     │
     ▼
HOLEY_SMI_ELEMENTS          ← Slow (sparse integers)
     │
     ▼
HOLEY_DOUBLE_ELEMENTS        ← Slow (sparse doubles)
     │
     ▼
HOLEY_ELEMENTS               ← Slowest (sparse mixed)
```

### Why It's Bad

- Holey arrays require extra checks for each access
- Type transitions invalidate previous optimizations
- Prevents use of optimized array operations
- Degrades to generic property access

### How to Fix

1. **Pre-allocate Arrays**
```javascript
// Bad
const arr = [];
for (let i = 0; i < 1000; i++) {
  arr.push(i);
}

// Good
const arr = new Array(1000);
for (let i = 0; i < 1000; i++) {
  arr[i] = i;
}
```

2. **Keep Types Consistent**
```javascript
// Bad
const mixed = [1, "two", 3, null];

// Good
const numbers = [1, 2, 3];
const strings = ["one", "two", "three"];
```

3. **Avoid Creating Holes**
```javascript
// Bad
const arr = [];
arr[5] = 'value';  // Creates holes at 0-4

// Good
const arr = [];
arr.push('value');
```

4. **Use Proper Array Methods**
```javascript
// Bad (can create holes)
const arr = [1, 2, 3];
arr.length = 1000;

// Good
const arr = new Array(1000).fill(0);
```

### Detection Commands

```bash
# Run with elements kind tracking
node --trace-elements-transitions your-script.js

# Check for deoptimizations
node --trace-deopt your-script.js
```

### Example Output
```
[elements transition: PACKED_SMI_ELEMENTS -> PACKED_DOUBLE_ELEMENTS]
[elements transition: PACKED_ELEMENTS -> HOLEY_ELEMENTS]
```

---

## Pattern #4: Try-Catch Blocks

### What Causes It

1. **Hot Code in Try Block**
```javascript
function processItems(items) {
  try {
    // Hot path code inside try
    for (let i = 0; i < items.length; i++) {
      compute(items[i]);
    }
  } catch (e) {
    handleError(e);
  }
}
```

2. **Deeply Nested Try-Catch**
```javascript
try {
  try {
    try {
      // Code here
    } catch (e3) {}
  } catch (e2) {}
} catch (e1) {}
```

3. **Try-Catch in Tight Loops**
```javascript
for (let i = 0; i < 1000000; i++) {
  try {
    // Each iteration enters try block
    process(i);
  } catch (e) {}
}
```

### Why It's Bad

- V8's TurboFan has difficulty optimizing code inside try blocks
- Exception handling overhead even when no exception occurs
- Prevents inlining
- Creates complex control flow that's hard to optimize

### How to Fix

1. **Move Try-Catch to Appropriate Boundary**
```javascript
// Bad
function processItems(items) {
  try {
    for (let i = 0; i < items.length; i++) {
      compute(items[i]);
    }
  } catch (e) {
    handleError(e);
  }
}

// Good
function processItems(items) {
  for (let i = 0; i < items.length; i++) {
    compute(items[i]);  // Hot path outside try
  }
}

function safeProcessItems(items) {
  try {
    return processItems(items);
  } catch (e) {
    handleError(e);
  }
}
```

2. **Move Try-Catch Outside Loops**
```javascript
// Bad
for (let i = 0; i < items.length; i++) {
  try {
    process(items[i]);
  } catch (e) {
    handleError(e);
  }
}

// Good
try {
  for (let i = 0; i < items.length; i++) {
    process(items[i]);
  }
} catch (e) {
  handleError(e);
}
```

3. **Use Error Codes Instead**
```javascript
// Instead of exceptions for control flow
function parse(str) {
  try {
    return JSON.parse(str);
  } catch (e) {
    return null;
  }
}

// Use validation
function parse(str) {
  if (!isValidJSON(str)) {
    return null;
  }
  return JSON.parse(str);
}
```

### Detection Commands

```bash
# Check optimization status
node --allow-natives-syntax your-script.js

# In code, check if function is optimized
%OptimizeFunctionOnNextCall(myFunction);
console.log(%GetOptimizationStatus(myFunction));
```

### Optimization Status

- Functions with try-catch often show as "never optimized" or "optimized but disabled"
- Use `--trace-opt` to see why functions aren't optimized

---

## Pattern #5: Arguments Object

### What Causes It

1. **Using Arguments Object**
```javascript
function sum() {
  let total = 0;
  for (let i = 0; i < arguments.length; i++) {
    total += arguments[i];
  }
  return total;
}
```

2. **Modifying Arguments**
```javascript
function process() {
  arguments[0] = 'modified';  // Modifies arguments
  return doSomething.apply(this, arguments);
}
```

3. **Leaking Arguments**
```javascript
function outer() {
  function inner() {
    return arguments;  // Leaks outer's arguments
  }
  return inner();
}
```

### Why It's Bad

- Arguments object is array-like but not a real array
- Prevents optimization of the function
- Creates overhead for maintaining arguments object
- Prevents inlining in many cases

### How to Fix

1. **Use Rest Parameters**
```javascript
// Bad
function sum() {
  let total = 0;
  for (let i = 0; i < arguments.length; i++) {
    total += arguments[i];
  }
  return total;
}

// Good
function sum(...numbers) {
  let total = 0;
  for (let i = 0; i < numbers.length; i++) {
    total += numbers[i];
  }
  return total;
}

// Even better
function sum(...numbers) {
  return numbers.reduce((a, b) => a + b, 0);
}
```

2. **Use Named Parameters**
```javascript
// Bad
function createUser() {
  return {
    name: arguments[0],
    age: arguments[1],
    email: arguments[2]
  };
}

// Good
function createUser(name, age, email) {
  return { name, age, email };
}
```

3. **Don't Modify Arguments**
```javascript
// Bad
function process(a, b) {
  arguments[0] = a * 2;  // Don't do this
  return a + b;
}

// Good
function process(a, b) {
  const modifiedA = a * 2;
  return modifiedA + b;
}
```

### Detection Commands

```bash
# Check if function uses arguments
node --trace-opt-verbose your-script.js 2>&1 | grep "arguments"
```

---

## Detection Techniques

### Using V8 Intrinsics

```javascript
// Enable with --allow-natives-syntax

// Check optimization status
%OptimizeFunctionOnNextCall(myFunction);
myFunction();
console.log(%GetOptimizationStatus(myFunction));

// Force optimization for testing
%NeverOptimizeFunction(slowFunction);
%OptimizeFunctionOnNextCall(fastFunction);

// Check hidden class
function getHiddenClass(obj) {
  return %HaveSameMap(obj, obj);  // Internal debugging
}
```

### Using Performance API

```javascript
const { performance } = require('perf_hooks');

function benchmark(fn, iterations = 10000) {
  // Warmup
  for (let i = 0; i < 1000; i++) fn();
  
  // Measure
  const start = performance.now();
  for (let i = 0; i < iterations; i++) {
    fn();
  }
  const end = performance.now();
  
  return end - start;
}
```

### Using dexnode/--trace-deopt

```bash
# Trace all deoptimizations
node --trace-deopt your-script.js

# Trace optimizations
node --trace-opt your-script.js

# Trace inline cache state
node --trace-ic your-script.js

# Combined
node --trace-opt --trace-deopt --trace-ic your-script.js 2>&1 | tee output.log
```

---

## V8 Flags Reference

### Essential Flags

```bash
--trace-deopt              # Show when functions deoptimize
--trace-opt                # Show when functions optimize
--trace-opt-verbose        # Detailed optimization info
--trace-ic                 # Track inline cache states
--trace-maps               # Track hidden class changes
--trace-elements-transitions  # Track array element kind changes
--allow-natives-syntax     # Enable % intrinsic functions
--print-opt-code          # Print optimized code
--code-comments           # Add comments to generated code
```

### Performance Flags

```bash
--max-opt                  # Optimize immediately (testing)
--always-opt              # Always optimize (testing)
--no-opt                  # Never optimize (comparison)
--opt                     # Enable optimization (default)
```

### Profiling Flags

```bash
--prof                    # Generate profiling output
--prof-process            # Process profiling output
--log-ic                  # Log IC events to v8.log
--log-maps                # Log map events to v8.log
```

---

## Optimization Status Codes

When using `%GetOptimizationStatus(function)`:

```
1 (0x1)    - Function is optimized
2 (0x2)    - Function is not optimized
3 (0x3)    - Function is always optimized
4 (0x4)    - Function is never optimized
6 (0x6)    - Function is maybe deoptimized
```

### Interpretation

```javascript
const status = %GetOptimizationStatus(myFunction);

if (status & 0x1) console.log('Optimized');
if (status & 0x2) console.log('Not optimized');
if (status & 0x4) console.log('Never optimized');
```

---

## Common Deoptimization Reasons

### From V8 Output

```
"Insufficient type feedback"
  → Function hasn't been called enough or with consistent types

"Wrong map"
  → Object shape changed unexpectedly

"Not a heap number"
  → Expected a number but got something else

"Not a string"
  → Expected a string but got something else

"Array length changed"
  → Array was resized unexpectedly

"Hole in array"
  → Accessed array element that doesn't exist

"Wrong instance type"
  → Expected one object type, got another

"Out of bounds"
  → Array access outside valid range

"Type mismatch"
  → General type inconsistency
```

---

## Testing for Deoptimizations

### Test Structure

```javascript
// test-hidden-class.js
const assert = require('assert');
const { execSync } = require('child_process');

function testHiddenClassDeopt() {
  // Run with deopt tracking
  const output = execSync(
    'node --trace-deopt examples/hidden-class/deopt.js',
    { encoding: 'utf-8', stderr: 'inherit' }
  );
  
  // Check that deopt occurred
  assert(output.includes('Deoptimizing'), 
    'Expected deoptimization to occur');
}

testHiddenClassDeopt();
```

### Comparative Testing

```javascript
function testPerformanceDifference() {
  const deoptTime = benchmark(deoptimizedVersion);
  const optTime = benchmark(optimizedVersion);
  
  const improvement = (deoptTime / optTime - 1) * 100;
  
  assert(improvement > 50, 
    `Expected >50% improvement, got ${improvement.toFixed(2)}%`);
}
```

---

## Best Practices Summary

### DO:
✅ Initialize all object properties in constructor
✅ Keep function parameter types consistent
✅ Use rest parameters instead of arguments
✅ Pre-allocate arrays when size is known
✅ Keep array element types consistent
✅ Move try-catch to appropriate boundaries
✅ Use named parameters for clarity
✅ Benchmark before and after optimizations

### DON'T:
❌ Add properties after object creation
❌ Delete object properties
❌ Mix types in function calls
❌ Create holey arrays
❌ Use arguments object
❌ Put hot code inside try-catch
❌ Modify function arguments
❌ Assume V8 will optimize everything

---

## Additional Resources

### V8 Blog Posts
- Understanding V8's Hidden Classes
- What's up with Monomorphism
- Elements Kinds in V8

### Tools
- Chrome DevTools Performance Panel
- Node.js --prof and --prof-process
- clinic.js suite (doctor, bubbleprof, flame)
- deoptigate

### Books
- "Understanding V8" by Franziska Hinkelmann
- "High Performance Browser Networking" by Ilya Grigorik

---

## Conclusion

Understanding these deoptimization patterns is crucial for writing high-performance Node.js applications. The key is to:

1. **Write type-stable code** - Consistent types lead to better optimization
2. **Understand V8's assumptions** - Know what V8 expects for optimization
3. **Measure actual performance** - Use profiling tools to validate improvements
4. **Profile in production** - Real-world usage patterns matter most

Use this reference when implementing examples and when teaching others about V8 optimization.
