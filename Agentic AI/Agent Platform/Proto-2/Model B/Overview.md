### **Major Pieces of modeB.ts**

#### 1. **Tool Definitions** (Lines 29-198)

- **Core Tools**: rewrite_query, search, fetch, load_skill, rerank, filter, assess, synthesize
- **Capability Tools**: Extended verbs (search, query, resolve, fetch, scan, exec)
- **Recipe Tools**: digest_doc, extract_fields for recipe-specific extractions

#### 2. **Main Orchestration Function: `runAgentic()`** (Lines 253-845)

This is the heart of Mode B - implements the ReAct agent loop:

```tsx
export async function runAgentic(
  ctx: RunContext,
  query: string,
  guidance?: string,
  prompts?: PromptOverrides,
  capability = false,
  maxTurns: number = MAX_TURNS,
  recipe?: RecipeRunSpec,
  ranker: RankerMode = "llm",
  sourceFilter?: string[],
): Promise<ModeSummary>
```

#### 3. **State Management** (Lines 268-277)

- `candidates`: All retrieved documents
- `filtered`: Top-ranked documents after filtering
- `thoughts`: Agent's reasoning trace
- `rewritten`: Query after rewriting
- `answer`/`citations`: Final synthesis output

#### 4. **Key Helper Functions**

- **`synthesisDocs()`** (Lines 282-297): Selects which docs go to synthesis
- **`investigatorNotes()`** (Lines 301-307): Formats agent thoughts for synthesis
- **`dedupePush()`** (Lines 309-317): Deduplicates retrieved docs
- **`handleSourceError()`** (Lines 327-350): Manages source failures with pause/retry

#### 5. **Agent Loop** (Lines ~400-700)

The main ReAct loop that:

1. Builds system prompt with tool catalog
2. Runs Anthropic's tool-use API
3. Processes each tool call
4. Continues until MAX_TURNS or task completion

### **Integration Points**

#### **1. Called by Runner** (runner.ts:63-70)

- Runner orchestrates **parallel execution** of Mode A and Mode B
- Or runs Mode B alone in `agenticOnly` mode
- Passes `RunContext` for event streaming and pause/resume

#### **2. Uses RunContext** for:

- **Event Streaming**: Each step emits events via `ctx.step()`
- **User Interaction**: `ctx.pauseForUser()` for auth failures
- **Progress Tracking**: Groups steps by turn

#### **3. Consumes Lower-Level Primitives**:

- `queryRewrite` - Normalizes queries
- `assess` - Evaluates sufficiency
- `rerank` - Scores by relevance
- `filter` - Deduplicates
- `synthesize` - Generates final answer
- `retrieve` - MCP tool calls

#### **4. Supports Two Tool Surfaces**:

- **Raw Mode**: Direct MCP tools from catalog
- **Capability Mode**: Semantic verbs (search/query/resolve/fetch)

### **Key Architectural Patterns**

#### **1. Document Management**

```tsx
candidates[] → rerank → filter → filtered[] → synthesize
         ↑
    dedupePush() // Maintains uniqueness
```

#### **2. Error Recovery**

- Source failures tracked per-source
- Auth errors trigger user pause/retry
- Non-auth failures retry 2x then mark source down

#### **3. Evidence Preservation**

`synthesisDocs()` ensures critical docs reach synthesis:

- Pinned (explicitly fetched)
- Full-text expansions
- Query counts (authoritative)
- Exec results

#### **4. Agent Memory**

- `thoughts[]` - Agent's reasoning trace
- `investigatorNotes()` - Passes reasoning to synthesis
- Prevents synthesis from contradicting agent's findings

### **Flow Summary**

```
WebSocket → runner.ts → runAgentic() → ReAct Loop
                                          ↓
                                    Tool Calls (1-12 turns)
                                          ↓
                                    assess() decides continue/stop
                                          ↓
                                    synthesize() with citations
                                          ↓
                                    ModeSummary result
```

### **Higher-Level Usage**

1. **Side-by-Side Comparison**: Mode B runs parallel to Mode A for comparison
2. **Recipe Execution**: Constrained agent with specific sources/fields
3. **Standalone Agent**: Direct agentic search without planning
4. **Judge Evaluation**: Results compared by LLM judge

The key innovation is Mode B's **adaptive nature** - it can recognize failures, retry with refinements, and build evidence iteratively, unlike Mode A's static plan.
