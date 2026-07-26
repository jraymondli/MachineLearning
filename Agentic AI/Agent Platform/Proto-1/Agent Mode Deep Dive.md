Based on my analysis of the Recipe Executor's Agent mode, here's a comprehensive overview of how it implements a full LLM tool-use loop with search primitives exposed as OpenAI function tools:
Architecture Overview
Agent mode represents the most flexible execution pattern, implementing a conversational search agent that iteratively calls primitives based on prior results. Unlike Fixed (DAG execution), LLM-Recipe (generates DAG), or LLM-Code (generates Starlark), Agent mode enables dynamic decision-making through a tool-use loop.
1. Core Flow Implementation
The runAgent function orchestrates the agent loop (modes.go:357-409):
func runAgent(ctx context.Context, query string, rc *RunContext, emit func(Event)) {
    // 1. Setup system prompt and tools
    system := render(template, promptVars(ctx, rc))
    tools := toolDefs(rc.Primitives)
    extra := identityArgs(rc)
    
    // 2. Create tool runner with security checks
    runTool := func(name string, input map[string]any) (any, string, error) {
        if rc.Disabled[name] {
            return nil, "", fmt.Errorf("primitive %q is disabled", name)
        }
        out, err := rc.Client.InvokePrimitive(ctx, name, mergeIdentity(input, extra))
        return out, summaryOf(out), nil
    }
    
    // 3. Run tool loop with event streaming
    err := rc.RunToolLoop(ctx, system, "Query: "+query, tools, runTool, func(ev ToolEvent) {
        // Process and emit events
    })
}
2. Tool Definition Generation
Primitives are converted to OpenAI function tool definitions (catalog.go:40-61):
func toolDefs(prims []PrimitiveInfo) []*openai.OpenAIChatCompletionsTool {
    tools := make([]*openai.OpenAIChatCompletionsTool, 0, len(prims))
    for _, p := range prims {
        params, _ := structpb.NewStruct(p.InputSchema)
        tools = append(tools, &openai.OpenAIChatCompletionsTool{
            Type: "function",
            Function: &openai.OpenAIChatCompletionsFunction{
                Name:        p.Name,
                Description: p.Description,
                Parameters:  params,  // JSON Schema
            },
        })
    }
    return tools
}
Each primitive includes:
Name: Function identifier (e.g., lexical_retrieval)
Description: Natural language explanation
Parameters: JSON Schema for input validation
3. LLM Tool-Use Loop
The ToolLoop implements OpenAI-style tool calling (llm.go:69-119):
func (l *LLM) ToolLoop(ctx, system, user, tools, runTool, emit) error {
    messages := []*openai.OpenAIChatCompletionsMessage{
        {Role: "system", Content: system},
        {Role: "user", Content: user},
    }
    
    for turn := 0; turn < toolLoopMaxTurns; turn++ {  // Max 12 turns
        // 1. Get LLM response
        msg, usage, err := l.chat(ctx, l.model, messages, tools, llmMaxTokens, l.org)
        
        // 2. Emit narration
        if t := msg.GetContent(); t != "" {
            emit(ToolEvent{Kind: "text", Text: t})
        }
        
        // 3. Process tool calls
        toolCalls := msg.GetToolCalls()
        if len(toolCalls) == 0 {
            break  // No more tools to call
        }
        
        // 4. Execute each tool
        for _, tc := range toolCalls {
            result, summary, err := runTool(name, input)
            emit(ToolEvent{Kind: "tool", Tool: {...}})
            
            // 5. Add tool result to conversation
            messages = append(messages, &openai.OpenAIChatCompletionsMessage{
                Role: "tool", 
                ToolCallId: tc.GetId(), 
                Content: asToolResultText(result),  // Truncated to 6000 chars
            })
        }
    }
    emit(ToolEvent{Kind: "final", Text: finalText})
}
4. Event Processing and Streaming
Agent mode emits three types of events via SSE:
Thoughts (text events): LLM reasoning before tool calls
Tool Calls (tool events): Primitive invocations with results
Final Answer (final event): Conclusive response
Event processing (modes.go:384-398):
err := rc.RunToolLoop(ctx, system, "Query: "+query, tools, runTool, func(ev ToolEvent) {
    switch ev.Kind {
    case "text":
        buf += ev.Text  // Accumulate narration
    case "tool":
        if t := strings.TrimSpace(buf); t != "" {
            emit(thoughtEvent(t))  // Emit accumulated thoughts
            buf = ""
        }
        emit(callEvent(name, input, summary, callView(result)))
    case "final":
        emit(answerEvent(ev.Text, nil))
    }
})
5. Security and Identity Injection
Critical security measures:
Identity Injection: Security-sensitive parameters are forced by the runtime (modes.go:335-342):
func identityArgs(rc *RunContext) map[string]any {
    return map[string]any{
        "requestor_email": rc.RequestorEmail,
        "org":             rc.Org,
        "acl_tokens":      rc.ACLTokens,
        "bot":             rc.Bot,
    }
}
Primitive Disabling: Per-request primitive filtering (modes.go:371-373):
if rc.Disabled[name] {
    return nil, "", fmt.Errorf("primitive %q is disabled for this run", name)
}
Hallucination Protection: Guards against LLM inventing non-existent tools
6. Agent System Prompt
The agent receives detailed instructions (prompts.go:52-64):
"agent_system": "You are a search agent. Answer the user's query by calling the available " +
    "primitive tools one at a time. A typical flow is reformulate -> retrieve " +
    "(lexical and/or semantic) -> merge -> featurize -> truncate (k=50) -> " +
    "rerank (method \"bge\") -> filter -> truncate -> summarize, but decide each " +
    "step from the prior results. The rerank model has an input-size limit: whenever you call rerank, " +
    "first truncate the hits with k=50 so rerank receives no more than 50 items. " +
    "Before each tool call, write one short sentence of reasoning. " +
    "Never pass requestor_email, org, bot, or acl_tokens — the runtime injects " +
    "the real values. This org's searchable sources, usable as the retrieval " +
    "`corpus` arg: {{available_sources}}. Scope `corpus` to a subset when the query " +
    "clearly targets one source, or omit it to search every source. " +
    "When you have a grounded, cited answer, stop calling tools and state it."
7. Result Formatting
Tool results are formatted for display:
summaryOf: One-line label (e.g., "5 hits", answer preview)
callView: Structured view for UI expansion
Hits: Up to 12 items with metadata
Answer: Text with citations
Query variants: Labeled fields
Scalars: Simple values
8. Error Handling and Recovery
Multi-level error handling:
Tool Execution Errors: Propagated to LLM for recovery
Mid-Loop Failures: Flush accumulated narration before error (modes.go:401-406)
Turn Limit: Maximum 12 turns prevents infinite loops
Result Truncation: Tool outputs limited to 6000 chars
9. Cost Tracking
Each turn is metered separately (llm.go:84-87):
meter.FromContext(ctx).Add(meter.CostRecord{
    Purpose: meter.PurposeAgentTurn, 
    Label: fmt.Sprintf("turn_%d", turn+1),
    Model: l.model, 
    Backend: meter.BackendLLMGateway, 
    Usage: usage,
})
10. Performance Characteristics
Latency: Highest among all modes (multiple LLM calls)
Flexibility: Can adapt strategy based on results
Token Usage: ~200-500 tokens per turn
Turn Limit: 12 turns maximum
Result Size: 6KB per tool result
Timeout: Inherited from context
Key Insights
Agent mode trades latency for flexibility - multiple LLM calls enable dynamic reasoning
Tool definitions use JSON Schema for structured parameter validation
Security is enforced at runtime - identity injection and primitive disabling
Event streaming provides real-time visibility into the agent's reasoning process
Error recovery is graceful - narration preserved even on failure
Cost attribution is granular - each turn tracked separately
The Agent mode represents a production-ready implementation of tool-use agents, balancing flexibility, security, and observability in a way that enables sophisticated search orchestration while maintaining control over primitive execution.
