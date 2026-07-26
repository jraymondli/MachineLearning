# Deep Dive: Flowsys Runtime

The flowsys runtime is a sophisticated concurrent execution framework designed for DAG-based workflows. Using `searchdag/engine.go` as our guide, here's how it orchestrates complex execution patterns:

### 1. **Core Architecture**

#### Runtime Lifecycle (`flowsys/runtime.go`)

```go
// From searchdag/engine.go usage:
collector := flowsys.NewTraceCollector()
ctx = flowsys.WithTraceSink(ctx, collector)
rt := flowsys.NewRuntime(ctx)
fctx := rt.Context()
// ... execute DAG ...
rt.Wait()  // Block until all futures complete
```

**Key Components**:

- **Runtime**: Manages goroutine lifecycle with WaitGroup
- **Context hierarchy**: Runtime owns cancellation boundary
- **Trace sink**: Captures execution events

### 2. **Future Types and Execution Models**

#### Three Future Variants:

**1. Eager (Go/GoNode)**:

```go
// Starts immediately
flowsys.Go(ctx, "name", func(ctx) (T, error) {...})
```

**2. Deferred (DeferredNode)** - Used in searchdag:

```go
// From engine.go line 55-81
built[n.ID] = flowsys.DeferredNode(fctx, spec,
    waitFn,  // Dependency synchronization
    runFn)   // Actual work
```

**3. Resolved (Input/ResolvedNode)**:

```go
// Pre-computed value with trace node
flowsys.Resolved(ctx, "name", value)
```

### 3. **Two-Phase Execution Pattern**

The DAG engine uses a critical two-phase pattern:

```go
// From engine.go lines 56-81
flowsys.DeferredNode(fctx, spec,
    // WAIT PHASE: Synchronize dependencies
    func(ctx context.Context) error {
        for _, dv := range depVs {
            dv.Start()  // Start all deps concurrently
        }
        for i, dv := range depVs {
            r, err := dv.Await(ctx)  // Collect results
            if err != nil {
                return err
            }
            depResults[n.Deps[i]] = r
        }
        return nil
    },
    // RUN PHASE: Execute the actual work
    func(ctx context.Context) (Result, error) {
        r, err := handler(ctx, input, depResults, node)
        if err == nil {
            nodeResults[n.ID] = r  // Thread-safe storage
        }
        return r, err
    })
```

**Why Two Phases?**

- **Wait phase**: Records dependency blocking time
- **Run phase**: Records actual execution time
- **Trace separation**: Distinguishes waiting vs working in metrics

### 4. **Dependency Management**

#### Starting Strategy (line 58):

```go
for _, dv := range depVs {
    dv.Start()  // Start ALL dependencies first
}
```

This ensures maximum parallelism - all independent branches execute concurrently.

#### NodeRef System:

```go
type NodeSpec struct {
    Name     string
    Kind     string
    Deps     []NodeRef    // Graph structure
    WaitDeps []NodeRef    // Subset to await
}
```

- **Deps**: Full dependency graph for visualization
- **WaitDeps**: Only nodes this one blocks on
- **JoinNodeRefs**: Deduplicates and flattens references

### 5. **Context Propagation Magic**

#### The futureContext Split:

```go
// From future.go lines 314-340
type futureContext struct {
    values    context.Context  // Trace labels, runtime ref
    lifecycle context.Context  // Cancellation, deadline
}
```

**Critical Design**:

- **Values**: Follow caller's context chain (preserves metadata)
- **Lifecycle**: Follow runtime's context (shared cancellation)
- **Result**: Cancelled Await doesn't kill shared work

Example impact:

```go
// Multiple awaiters, one cancels
result1 := future.Await(ctx1)  // ctx1 cancels
result2 := future.Await(ctx2)  // Still gets result!
```

### 6. **Concurrency Patterns**

#### Maximum Parallelism:

```go
// All independent nodes start together
for _, n := range topologicalOrder {
    // Each node gets its own goroutine
    built[n.ID] = DeferredNode(...)
}
// Only terminal node is explicitly started
out.Start()
```

#### Error Isolation:

```go
// From engine.go line 62-64
// A failed dep fails this node, but already-started
// siblings run to completion (no cross-cancellation)
```

### 7. **Stream Abstraction**

For handling collections of futures:

```go
type Stream[T any] struct {
    items    []V[T]           // Concrete futures
    deferred V[Stream[T]]     // Or future-of-stream
}
```

**Key Methods**:

- `AwaitAll`: Wait for all items
- `Ready/ReadyOK`: Partial completion observation
- `MapStream/FilterItems`: Lazy transformations

### 8. **Trace Collection System**

#### Event Flow:

```go
// Registration
OnNodeRegister(NodeInfo{ID, Name, Kind, Deps})
    ↓
// Lifecycle events
OnNodeEvent(scheduled → waiting → running → done/failed)
    ↓
// Timing extraction
WaitDuration = RunningAt - ScheduledAt
RunDuration = DoneAt - RunningAt
```

#### Mermaid Generation:

```go
// From trace.go - Converts events to diagram
func (c *TraceCollector) Mermaid() string {
    // Generates:
    // node_1["q\nwait=0s run=150ms\nstate=done"]
    // node_1 --> node_2
}
```

### 9. **Advanced Patterns in searchdag/engine.go**

#### Thread-Safe Result Collection (lines 33-34, 76-78):

```go
var mu sync.Mutex
nodeResults := make(map[string]Result)
// Later in run phase:
mu.Lock()
nodeResults[n.ID] = r
mu.Unlock()
```

#### Topological Ordering (line 23):

```go
order, err := Validate(p, reg)  // Returns topo-sorted nodes
```

Ensures dependencies are built before dependents.

#### Runtime Ownership (lines 85-87):

```go
out.Start()           // Start terminal node
res, err := out.Await(fctx)  // Use runtime context
rt.Wait()            // Ensure all goroutines complete
```

### 10. **Performance Characteristics**

**Concurrency**:

- O(1) goroutines per node
- Maximum parallelism for independent paths
- No unnecessary synchronization

**Memory**:

- Bounded by node count
- Results stored once, shared across awaiters
- Trace events buffered in collector

**Latency**:

- Critical path determines total time
- Parallel branches don't add latency
- Two-phase split enables accurate attribution

### 11. **Error Handling Philosophy**

```go
// Panic recovery in future.go
defer func() {
    if rec := recover(); rec != nil {
        f.err = fmt.Errorf("node %q panicked: %v", f.name, rec)
        emitNodeEvent(f.ctx, f.ref, NodeEventFailed, f.err)
    }
}()
```

**Principles**:

- Panics converted to errors
- Errors propagate through dependencies
- Failed nodes don't cancel siblings
- Runtime cancellation stops everything

### 12. **Real-World Usage Pattern**

From the searchdag engine, the complete pattern:

1. **Setup**: Create runtime with trace collector
2. **Build**: Construct DAG of DeferredNodes
3. **Wire**: Connect dependencies via NodeRefs
4. **Execute**: Start terminal node
5. **Await**: Block on result
6. **Cleanup**: Wait for all goroutines
7. **Observe**: Extract trace for visualization

The flowsys runtime provides a robust foundation for complex concurrent workflows, with sophisticated context management, comprehensive observability, and efficient execution patterns perfectly suited for DAG-based search orchestration.
