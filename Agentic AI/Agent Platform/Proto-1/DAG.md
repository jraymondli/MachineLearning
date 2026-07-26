Deep Dive: DAG Handling in Recipe Executor
The Recipe Executor implements a sophisticated DAG execution system with strong typing, validation, and visualization capabilities. Here's the comprehensive breakdown:
1. DAG Schema and Definition
Core Structure (searchdag/validate.go):
type Node struct {
    ID   string         // Unique identifier
    Op   string         // Operation/primitive name
    Deps []string       // Upstream dependencies
    Args map[string]any // Static configuration
}

type Plan struct {
    Nodes  []Node // All nodes in the DAG
    Output string // Terminal node ID
}
Type System (searchdag/contract.go):
4 Port Types: query, queries, hits, answer
Port Constraints:
Regular: Exactly one input required
Optional: Zero or one input allowed
Variadic: Two or more inputs (fan-in)
2. Contract-Based Validation
The system enforces strict contracts for each primitive:
type Descriptor struct {
    In       []Port      // Input port requirements
    Config   []ConfigArg // Static args with validation
    Out      PortType    // Output type produced
    Requires []string    // Required hit fields
    Provides []string    // Hit fields generated
}
Validation Steps:
Structural checks: Node existence, unique IDs, cycle detection
Type checking: Edge compatibility verification
Arity validation: Port cardinality requirements
Config validation: Required args and enum membership
Field-flow analysis: Hit field propagation and requirements
3. DAG Execution Engine
Execution Flow (searchdag/engine.go):
func Run(ctx, plan, input, registry) (result, trace, error) {
    // 1. Validate and topologically sort
    order, err := Validate(plan, registry)
    
    // 2. Build flowsys nodes
    for _, node := range order {
        spec := NodeSpec{
            Name:     node.ID,
            Kind:     node.Op,
            Deps:     dependencies,
            WaitDeps: dependencies,
        }
        
        // 3. Create DeferredNode with wait/run phases
        built[node.ID] = DeferredNode(ctx, spec,
            waitPhase,  // Block on dependencies
            runPhase)   // Execute handler
    }
    
    // 4. Start terminal node and await
    output.Start()
    result := output.Await()
}
Key Features:
Concurrent execution: Independent nodes run in parallel
Two-phase nodes: Wait phase (dependencies) + Run phase (work)
Error isolation: Failed nodes don't cancel siblings
Context propagation: Cancellation flows through runtime
4. Flowsys Runtime Integration
The DAG executor leverages the flowsys runtime for:
Futures (flowsys/future.go):
Eager (Go): Starts immediately
Deferred: Starts on first await
Resolved: Pre-computed values with trace nodes
Runtime Management (flowsys/runtime.go):
Goroutine lifecycle: WaitGroup tracking
Cancellation boundaries: Runtime owns cancellation
Trace collection: Centralized event sink
Node States:
scheduled → waiting → running → done/failed/cancelled
5. Tracing and Visualization
Trace Collection (flowsys/trace.go):
type TraceSink interface {
    OnRunStart(ctx, RunInfo)
    OnNodeRegister(ctx, NodeInfo)
    OnNodeEvent(ctx, NodeEvent)
    OnRunDone(ctx, RunResult)
}
Visualization Outputs:
Mermaid Flowchart:
flowchart LR
    node_1["q\nwait=0s run=150ms\nstate=done"]
    node_2["lex\nwait=0s run=200ms\nstate=done"]
    node_3["sem\nwait=0s run=180ms\nstate=done"]
    node_1 --> node_2
    node_1 --> node_3
Mermaid Gantt (execution timeline):
gantt
    section running
    q : 0, 150ms
    lex : 150ms, 200ms
    sem : 150ms, 180ms
Chrome Trace Format: JSON for chrome://tracing visualization
6. Primitive Catalog
The system defines 12+ primitives with full contracts: Retrieval Primitives:
reformulate_query: Query expansion (1→N variants)
lexical_retrieval: BM25 keyword search
semantic_retrieval: Vector embedding search
mcp_retrieval: MCP tool-based search
Processing Primitives:
merge: Fan-in deduplication
featurize: Add scoring features
rerank: Re-score with models
filter: ACL filtering
truncate: Limit result count
Terminal Primitives:
summarize: Generate answer from hits
7. Advanced DAG Features
Field Propagation:
Tracks which fields are available at each node
Validates field requirements are met
Intersection at fan-in points
Configuration System:
type ConfigArg struct {
    Name     string
    Type     string   // "string", "int", "enum"
    Enum     []string // Valid values for enum
    Default  string
    Required bool
}
Dependency Resolution:
Topological sorting ensures execution order
Cycle detection prevents infinite loops
Max 128 nodes prevents pathological DAGs
8. Recipe Examples
Hybrid Search (parallel retrieval + merge):
{
  "nodes": [
    {"id": "q", "op": "reformulate_query"},
    {"id": "lex", "op": "lexical_retrieval", "deps": ["q"]},
    {"id": "sem", "op": "semantic_retrieval", "deps": ["q"]},
    {"id": "m", "op": "merge", "deps": ["lex", "sem"]},
    {"id": "rr", "op": "rerank", "deps": ["m"]},
    {"id": "top", "op": "truncate", "deps": ["rr"], "args": {"k": 5}}
  ],
  "output": "top"
}
9. SSE Integration for DAG Events
The DAG execution streams real-time events:
node: Node state updates with timing
edges: DAG structure for visualization
node_result: Individual operation outputs
trace: Mermaid diagram updates
10. Performance Optimizations
Parallel execution: Max concurrency for independent paths
Lazy evaluation: Deferred nodes start only when needed
Streaming results: No buffering of full DAG state
Bounded validation: O(N²) worst case for N nodes
The Recipe Executor's DAG system provides a robust, type-safe, and observable framework for orchestrating complex search pipelines with real-time visibility into execution flow.
