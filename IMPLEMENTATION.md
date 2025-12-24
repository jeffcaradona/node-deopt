# Implementation Guide - Step-by-Step

This guide provides concrete steps and code examples for implementing the Node.js Deoptimization Demo Application. Follow these steps in order to build the complete application.

---

## Prerequisites

Before starting implementation:
- Node.js v18+ installed
- Basic understanding of V8 optimization concepts
- Familiarity with Node.js development
- VSCode or another IDE with debugging support

---

## Phase 1: Project Initialization

### Step 1.1: Initialize npm Project

```bash
cd node-deopt
npm init -y
```

### Step 1.2: Install Dependencies

```bash
# Core dependencies
npm install fastify

# Development and tool dependencies
npm install --save-dev autocannon chalk cli-table3

# Optional but recommended
npm install --save-dev dexnode
```

### Step 1.3: Create Directory Structure

```bash
mkdir -p src/{examples/{hidden-class,polymorphic,arrays,try-catch,arguments},server,utils}
mkdir -p scripts benchmarks docs/tools .vscode test
```

---

## Phase 2: Core Examples

See DEOPT_PATTERNS.md for detailed technical information about each pattern.

### Example Structure

Each example should have:
- `deopt.js` - Intentionally deoptimized version
- `optimized.js` - Optimized version
- `index.js` - Comparison runner
- `README.md` - Documentation

---

## Phase 3: HTTP Server

Create `src/server/index.js` with Fastify endpoints for each example.

Endpoints should accept `?mode=deopt` or `?mode=optimized` query parameter.

---

## Phase 4: Tool Integration

### Autocannon Benchmarking
Create `scripts/benchmark-all.js` to:
- Start the server
- Run autocannon against each endpoint
- Compare deopt vs optimized versions
- Display results in formatted tables

### Deoptimization Checking
Create `scripts/check-deopts.js` to:
- Run examples with `--trace-deopt` flag
- Parse V8 output
- Report which examples cause deopts
- Verify optimized versions don't deopt

---

## Phase 5: VSCode Configuration

Create `.vscode/launch.json` with configurations:
- Debug current file
- Debug with deopt tracking
- Debug with V8 intrinsics
- Debug server

---

## Phase 6: Documentation

### README.md
- Quick start guide
- Example descriptions
- Tool usage instructions
- Links to other docs

### Individual Example READMEs
- What the example demonstrates
- How to run it
- Expected results
- Key takeaways

---

## Implementation Order

1. Start with Hidden Class example (simplest)
2. Test thoroughly with V8 flags
3. Verify performance difference
4. Move to next example
5. Add server after examples work standalone
6. Add benchmarking scripts
7. Complete documentation

---

## Testing Checklist

- [ ] All examples run without errors
- [ ] Deoptimized versions cause deopts (verify with --trace-deopt)
- [ ] Optimized versions show performance improvements
- [ ] Server starts and responds correctly
- [ ] Autocannon benchmarks complete successfully
- [ ] VSCode debugging configurations work
- [ ] All npm scripts execute correctly
- [ ] Documentation is clear and accurate

---

## Key npm Scripts to Add

```json
{
  "scripts": {
    "start": "node src/server/index.js",
    "example:hidden-class": "node src/examples/hidden-class/index.js",
    "example:polymorphic": "node src/examples/polymorphic/index.js",
    "example:arrays": "node src/examples/arrays/index.js",
    "example:try-catch": "node src/examples/try-catch/index.js",
    "example:arguments": "node src/examples/arguments/index.js",
    "bench:all": "node scripts/benchmark-all.js",
    "deopt:check": "node scripts/check-deopts.js"
  }
}
```

---

## Common Issues & Solutions

### No deoptimizations detected
Increase iteration count or run multiple times to trigger optimization.

### Performance improvements not significant
Ensure warmup runs, increase workload size, check Node.js version.

### Autocannon fails to connect
Verify server is running, check port availability.

### V8 flags not recognized
Check Node.js version (need v18+), verify flag spelling.

---

## Next Steps After Planning

1. Review all planning documents
2. Get stakeholder approval
3. Set up project repository
4. Begin Phase 1 implementation
5. Iterate through each phase
6. Test continuously
7. Document as you build

---

For detailed code examples and technical specifications, see:
- PLANNING.md - Overall strategy
- ARCHITECTURE.md - System design
- DEOPT_PATTERNS.md - Technical details
