The Recipe Executor in florence-go-prototype is a sophisticated search orchestration system that executes DAG-based "recipes" for information retrieval and processing. Here's how it works:

### Core Architecture

#### 1. **Recipe Structure**

A recipe is a DAG (Directed Acyclic Graph) where:

- **Nodes**: Individual operations (primitives) like `reformulate_query`, `lexical_retrieval`, `semantic_retrieval`, `rerank`, etc.
- **Dependencies**: Edges defining data flow between operations
- **Output**: The terminal node whose result becomes the recipe output

Example recipe (hybrid search):

```json
{
  "nodes": [
    {"id": "q", "op": "reformulate_query", "deps": []},
    {"id": "lex", "op": "lexical_retrieval", "deps": ["q"]},
    {"id": "sem", "op": "semantic_retrieval", "deps": ["q"]},
    {"id": "m", "op": "merge", "deps": ["lex", "sem"]},
    {"id": "rr", "op": "rerank", "deps": ["m"]},
    {"id": "top", "op": "truncate", "deps": ["rr"], "args": {"k": 5}}
  ],
  "output": "top"
}
```

DAG Deep Dive: DAG Handling in Recipe Executor

#### 2. **Execution Engine (`searchdag/engine.go`)**

- Uses **flowsys runtime** for concurrent DAG execution
    
    Deep Dive: Flowsys Runtime
    
- **Parallel execution**: Independent nodes run concurrently
- **Dependency management**: Nodes wait for dependencies before executing
- **Error propagation**: Failed nodes cause dependent nodes to fail

Key execution flow:

1. Validate recipe structure and operations
2. Build execution nodes in topological order
3. Execute with dependency tracking
4. Return terminal node result + trace

#### 3. **Four Execution Modes**

**Fixed Mode** (`runFixed`):

- Executes predefined hybrid recipe
- No LLM planning, just execution + summarization
- Most deterministic and fastest

**LLM Recipe Mode** (`runLLMRecipe`):

- LLM generates a recipe JSON from catalog
- Validates and executes generated recipe
- Falls back to fixed recipe on generation failure

**LLM Code Mode** (`runLLMCode`):

- LLM writes Starlark code using primitives as functions
- Executes in sandboxed Starlark interpreter
    
    **Starlark Integration Deep Dive**
    
- More flexible than recipes (loops, conditionals)
- 10-second timeout for code execution

**Agent Mode** (`runAgent`): 

- Full LLM tool-use loop
- Primitives exposed as OpenAI function tools
- Iterative refinement based on results
- Most flexible but highest latency

Agent Mode Deep Dive

#### 4. **Primitives Catalog**

Core primitives include:

**Query Operations**:

- `reformulate_query`: Expands query into variants using LLM
- `extract_keywords`: Extracts key terms from query

**Retrieval**:

- `lexical_retrieval`: BM25 keyword search
- `semantic_retrieval`: Vector embedding search
- `mcp_retrieval`: Searches MCP-connected sources
- Federation across all configured sources (KB, tickets, Slack, etc.)

**Processing**:

- `merge`: Combines and deduplicates results
- `featurize`: Adds scoring features
- `rerank`: Re-scores using BGE cross-encoder
- `filter`: ACL filtering
- `truncate`: Limits result count
- `summarize`: LLM-generated answer from results

**Advanced**:

- `rank_fusion`: Reciprocal rank fusion
- `diversify`: Result diversification
- `cluster`: Groups similar results

#### 5. **Backend Integration**

The executor integrates with multiple Moveworks services:

- **ResourceRetrieverService**: Lexical/semantic search
- **LLM Gateway**: Query reformulation, summarization, planning
- **Knorah RecordSearcher**: Record-based search
- **MCP Gateway**: External tool integration
- **Recon**: Organization configuration discovery

#### 6. **Observability Features**

**Tracing**:

- Detailed DAG execution trace
- Node timing and dependencies
- Error propagation paths

**Token Metering** (`meter/` package):

- Tracks LLM token usage per purpose
- Cost attribution by backend
- Efficiency scoring via LLM judge
- SSE events: `cost` (per call) and `cost_total` (rollup)

#### 7. **Demo Server Features**

The demo server (`demo/server.go`) provides:

- `/api/primitives`: Available operations catalog
- `/api/prompts`: Default prompts for each mode
- `/api/recipe`: Current recipe JSON
- `/api/run`: SSE stream for live execution

SSE events stream includes:

- `mode`: Execution mode started
- `trace`: Real-time execution trace
- `node_result`: Individual operation results
- `calls`: LLM code primitive invocations
- `tool`: Agent tool calls
- `cost`: Token usage tracking
- `done`: Completion with final result

SSE Deep Dive in Recipe Executtor

#### 8. **Security & Sandboxing**

- **Starlark sandbox**: No imports, I/O, or system access
- **Execution timeouts**: Prevents infinite loops
- **ACL enforcement**: Results filtered by permissions
- **Identity forwarding**: Requestor identity passed through

### Key Innovation Points

1. **Multi-mode execution**: Same primitives, different orchestration strategies
2. **Federation**: Seamlessly combines multiple search backends
3. **Query expansion**: Automatic reformulation for better recall
4. **Real-time streaming**: Live execution visibility via SSE
5. **Cost transparency**: Detailed token usage attribution
6. **Graceful degradation**: Falls back when backends unavailable

The Recipe Executor represents a sophisticated approach to search orchestration, allowing experimentation with different execution strategies while maintaining a consistent primitive interface.
