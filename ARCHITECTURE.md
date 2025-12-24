# Architecture Overview - Node.js Deoptimization Demo

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Developer Interface                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  CLI Tool    │  │  VSCode      │  │  Chrome      │          │
│  │  Commands    │  │  Debugger    │  │  DevTools    │          │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Core Application                            │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  HTTP Server (Express/Fastify)                          │   │
│  │  ├── /api/hidden-class                                  │   │
│  │  ├── /api/polymorphic                                   │   │
│  │  ├── /api/arrays                                        │   │
│  │  ├── /api/try-catch                                     │   │
│  │  └── /api/arguments                                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                              │                                   │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Example Modules                                        │   │
│  │  ├── Deoptimized Implementations                        │   │
│  │  └── Optimized Implementations                          │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Tool Integration Layer                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │  Autocannon  │  │   Dexnode    │  │  Performance │          │
│  │  Load Test   │  │  Deopt Track │  │    Hooks     │          │
│  └──────────────┘  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Output & Reporting                          │
│  ├── Console Reports (formatted tables, charts)                 │
│  ├── JSON Reports (machine-readable)                            │
│  ├── HTML Reports (visual dashboards)                           │
│  └── Markdown Reports (documentation-ready)                     │
└─────────────────────────────────────────────────────────────────┘
```

## Component Interaction Flow

### Scenario 1: Running a Performance Comparison

```
User runs: npm run compare:hidden-class

    │
    ▼
CLI Script
    │
    ├──▶ Start HTTP Server
    │    └── Load example modules
    │
    ├──▶ Run Autocannon (Deoptimized)
    │    ├── Send requests to /api/hidden-class?mode=deopt
    │    └── Collect metrics
    │
    ├──▶ Run Autocannon (Optimized)
    │    ├── Send requests to /api/hidden-class?mode=optimized
    │    └── Collect metrics
    │
    ├──▶ Compare Results
    │    ├── Calculate differences
    │    └── Generate report
    │
    └──▶ Display Output
         └── Formatted table with improvements
```

### Scenario 2: Detecting Deoptimizations

```
User runs: npm run deopt:check

    │
    ▼
Dexnode Wrapper
    │
    ├──▶ Start Node with V8 flags
    │    └── --trace-deopt
    │    └── --trace-opt
    │    └── --allow-natives-syntax
    │
    ├──▶ Execute All Examples
    │    ├── hidden-class/deopt.js
    │    ├── polymorphic/deopt.js
    │    ├── arrays/deopt.js
    │    ├── try-catch/deopt.js
    │    └── arguments/deopt.js
    │
    ├──▶ Capture V8 Output
    │    └── Parse deopt events
    │
    ├──▶ Process Results
    │    ├── Group by file
    │    ├── Count occurrences
    │    └── Extract reasons
    │
    └──▶ Generate Report
         ├── List all deopts found
         └── Suggest fixes
```

### Scenario 3: Interactive Debugging

```
Developer presses F5 in VSCode

    │
    ▼
Launch Configuration (.vscode/launch.json)
    │
    ├──▶ Start Node with --inspect-brk
    │
    ├──▶ Load example file
    │
    ├──▶ Pause at breakpoint
    │
    └──▶ Developer Actions:
         ├── Step through code
         ├── Inspect variables
         ├── Run V8 optimization commands
         │   └── %OptimizationStatus(func)
         │   └── %GetOptimizationStatus(func)
         └── View deopt reasons in console
```

## Data Flow Architecture

```
┌──────────────┐
│  Source Code │
└──────┬───────┘
       │
       ▼
┌──────────────────┐
│  V8 Engine       │  ◄─── V8 Flags (--trace-deopt, etc.)
│  - Parse         │
│  - Optimize      │
│  - Execute       │
│  - Deoptimize    │
└──────┬───────────┘
       │
       ├─────────────┬────────────────┬──────────────┐
       │             │                │              │
       ▼             ▼                ▼              ▼
┌──────────┐  ┌──────────┐  ┌──────────────┐  ┌──────────┐
│Performance│  │  Deopt   │  │  Execution   │  │  Debug   │
│  Metrics  │  │  Events  │  │    Logs      │  │  Info    │
└──────┬────┘  └────┬─────┘  └──────┬───────┘  └────┬─────┘
       │            │               │               │
       └────────────┴───────┬───────┴───────────────┘
                            │
                            ▼
                ┌──────────────────────┐
                │  Analysis Engine     │
                │  - Aggregate         │
                │  - Compare           │
                │  - Visualize         │
                └──────────┬───────────┘
                           │
                           ▼
                ┌──────────────────┐
                │  Reports         │
                │  - Console       │
                │  - Files         │
                │  - Dashboard     │
                └──────────────────┘
```

## Module Structure

### Example Module Pattern

Each deoptimization example follows this structure:

```
examples/
├── hidden-class/
│   ├── index.js              # Exports both versions
│   ├── deopt.js              # Deoptimized implementation
│   ├── optimized.js          # Optimized implementation
│   ├── README.md             # Explanation and guide
│   └── test.js               # Validation tests
│
└── [other examples follow same pattern]
```

### Server Module Structure

```
server/
├── index.js                  # Main server entry point
├── routes.js                 # Route definitions
├── middleware/
│   ├── logging.js           # Request logging
│   ├── timing.js            # Performance timing
│   └── error.js             # Error handling
└── utils/
    ├── response.js          # Response formatting
    └── metrics.js           # Metrics collection
```

### Tool Integration Structure

```
tools/
├── autocannon/
│   ├── config.js            # Default configurations
│   ├── runner.js            # Autocannon wrapper
│   └── reporter.js          # Results formatting
│
├── dexnode/
│   ├── wrapper.js           # Dexnode execution wrapper
│   ├── parser.js            # Output parser
│   └── analyzer.js          # Deopt analyzer
│
└── profiler/
    ├── cpu.js               # CPU profiling utilities
    ├── memory.js            # Memory profiling utilities
    └── flame-graph.js       # Flame graph generator
```

## Configuration Architecture

### Environment-Based Configuration

```
config/
├── default.json             # Default settings
├── development.json         # Dev overrides
├── production.json          # Prod settings
└── test.json                # Test settings

Configuration Schema:
{
  "server": {
    "port": 3000,
    "host": "localhost"
  },
  "autocannon": {
    "connections": 10,
    "duration": 10,
    "pipelining": 1
  },
  "dexnode": {
    "flags": ["--trace-deopt", "--trace-opt"],
    "outputFormat": "json"
  },
  "examples": {
    "iterations": 10000,
    "warmup": 1000
  }
}
```

## Execution Modes

### 1. Standalone Mode
Run individual examples without server:
```bash
node src/examples/hidden-class/deopt.js
```

### 2. Server Mode
Run HTTP server for load testing:
```bash
node src/server/index.js
# Then use autocannon to test endpoints
```

### 3. CLI Mode
Use CLI tool for interactive experience:
```bash
node src/cli/index.js
# Presents menu of options
```

### 4. Debug Mode
Run with debugger attached:
```bash
node --inspect-brk src/examples/hidden-class/deopt.js
# Connect with Chrome DevTools or VSCode
```

### 5. Profile Mode
Run with profiling enabled:
```bash
node --prof src/examples/hidden-class/deopt.js
# Generates v8.log for analysis
```

## Tool Integration Points

### Autocannon Integration
```javascript
// Simplified integration pseudocode
const autocannon = require('autocannon');

async function benchmarkEndpoint(url, options) {
  const result = await autocannon({
    url: url,
    connections: options.connections || 10,
    duration: options.duration || 10,
    pipelining: options.pipelining || 1
  });
  
  return {
    requestsPerSec: result.requests.average,
    latency: result.latency,
    throughput: result.throughput
  };
}
```

### Dexnode Integration
```javascript
// Simplified integration pseudocode
const { spawn } = require('child_process');

async function runWithDeoptTracking(scriptPath) {
  return new Promise((resolve) => {
    const proc = spawn('node', [
      '--trace-deopt',
      '--trace-opt',
      '--allow-natives-syntax',
      scriptPath
    ]);
    
    let output = '';
    proc.stderr.on('data', (data) => {
      output += data.toString();
    });
    
    proc.on('close', () => {
      const deopts = parseDeoptOutput(output);
      resolve(deopts);
    });
  });
}
```

### VSCode Configuration
```jsonc
// .vscode/launch.json structure
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug Current Example",
      "type": "node",
      "request": "launch",
      "program": "${file}",
      "args": ["--trace-deopt"],
      "console": "integratedTerminal"
    },
    {
      "name": "Debug with Optimization Status",
      "type": "node",
      "request": "launch",
      "program": "${file}",
      "args": ["--allow-natives-syntax"],
      "console": "integratedTerminal"
    }
  ]
}
```

## Performance Metrics Collection

### Key Metrics to Track

1. **Throughput Metrics**
   - Requests per second
   - Operations per second
   - Data processed per second

2. **Latency Metrics**
   - Average latency
   - p50, p95, p99 percentiles
   - Max latency

3. **Optimization Metrics**
   - Number of optimizations
   - Number of deoptimizations
   - Optimization status distribution

4. **Resource Metrics**
   - Memory usage
   - CPU usage
   - Garbage collection pauses

### Metrics Storage Format

```json
{
  "timestamp": "2024-12-24T00:00:00.000Z",
  "example": "hidden-class",
  "mode": "deoptimized",
  "metrics": {
    "throughput": {
      "requestsPerSec": 5234
    },
    "latency": {
      "avg": 19.2,
      "p50": 15,
      "p95": 45,
      "p99": 78
    },
    "deopts": {
      "count": 5,
      "reasons": [
        "Insufficient type feedback for binary operation"
      ]
    }
  }
}
```

## Security Considerations

### Input Validation
- All HTTP endpoints validate input
- Rate limiting on server endpoints
- Sanitize user input in interactive mode

### Execution Safety
- Examples run in isolated contexts
- No eval() or Function() with user input
- Resource limits on benchmarks

### Dependencies
- Regular security audits with npm audit
- Minimal dependency tree
- Lock file for reproducible builds

## Scalability Considerations

### For Different Use Cases

1. **Workshop Setting** (10-50 users)
   - Each user runs locally
   - No shared infrastructure needed

2. **CI/CD Integration**
   - Headless execution
   - JSON output for parsing
   - Exit codes for pass/fail

3. **Production Monitoring**
   - Reusable modules as library
   - Integration with existing monitoring
   - Custom metric collection

## Future Architecture Extensions

### Potential Additions

1. **Web Dashboard**
   ```
   Frontend (React/Vue)
        │
        ▼
   Backend API (Express)
        │
        ▼
   Metrics Database (SQLite/PostgreSQL)
   ```

2. **Real-time Monitoring**
   ```
   Application Code
        │
   WebSocket Connection
        │
   Dashboard (Live Updates)
   ```

3. **Cloud Integration**
   ```
   Local Examples
        │
   Upload to Cloud Service
        │
   Aggregate Cross-Platform Results
   ```

## Conclusion

This architecture provides:
- **Modularity**: Each component is independent
- **Extensibility**: Easy to add new examples and tools
- **Testability**: Each module can be tested in isolation
- **Usability**: Multiple interfaces for different use cases
- **Maintainability**: Clear separation of concerns

The design supports both learning use cases (individual examples) and professional use cases (CI/CD integration, production monitoring) while maintaining simplicity and clarity.
