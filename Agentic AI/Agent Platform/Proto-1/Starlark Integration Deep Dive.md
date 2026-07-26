Based on my analysis of the Recipe Executor's Starlark integration, here's a comprehensive overview of how LLM-generated code is safely executed in a sandboxed environment:
Architecture Overview
Starlark serves as the sandboxed execution environment for LLM-generated code in the Recipe Executor's "LLM Code" mode. Unlike the Python exec() path in the original demo, this Go implementation uses starlark-go to provide a real sandbox with strict security guarantees.
1. Execution Environment Setup
The Starlark environment is configured with permissive options to accommodate LLM-generated code (exec.go:28-34):
var starlarkExecOptions = &syntax.FileOptions{
    Set:             true,        // Python set literals
    While:           true,        // While loops
    TopLevelControl: true,        // Top-level if/for/while
    GlobalReassign:  true,        // Reassign globals
    Recursion:       true,        // Allow recursive functions
}
These options make Starlark feel more like Python, important for LLM code generation since models are trained on Python syntax.
2. Code Execution Flow
The runCode function orchestrates execution (exec.go:35-79):
Code Preparation: Strip markdown code fences (python` or starlark`)
Environment Setup: Create predeclared variables including query string and primitive functions
Timeout Protection: Configure thread cancellation with time.AfterFunc
Execution: Run via starlark.ExecFileOptions
Result Extraction: Get the result variable from globals
Type Conversion: Convert Starlark value back to Go
3. Primitive Binding Mechanism
Search primitives are exposed as built-in Starlark functions (exec.go:81-102):
func primitiveBuiltin(name string, invoke InvokeFunc, calls *[]CallRecord) *starlark.Builtin {
    return starlark.NewBuiltin(name, func(thread, builtin, args, kwargs) {
        // Enforce keyword-only arguments
        if len(args) > 0 {
            return nil, fmt.Errorf("%s: use keyword arguments only", name)
        }
        // Convert kwargs to Go map
        // Invoke the real primitive
        // Record the call
        // Convert result to Starlark
    })
}
Key Features:
Keyword-only arguments enforced (no positional args)
All calls are recorded for replay in the UI
Results are summarized for display
Type conversion happens automatically
4. Type Conversion System
Bidirectional type conversion bridges Go and Starlark (exec.go:104-194): Go → Starlark (goToStarlark):
nil → starlark.None
bool → starlark.Bool
string → starlark.String
float64 → starlark.Float
int → starlark.Int
[]any → starlark.List
map[string]any → starlark.Dict
Starlark → Go (starlarkToGo):
Handles both small and big integers
Preserves nested structures
Dict iteration with proper ordering
5. Security and Safety Measures
The sandbox provides multiple layers of protection:
Import Blocking
No import statements allowed (exec_test.go:TestBlocksImport)
Starlark inherently blocks imports - attempting import os fails immediately
Timeout Protection
Configurable timeout via thread cancellation (exec.go:54-60)
Protects against infinite loops
Test verifies timeout works (exec_test.go:TestTimeoutCancelsLoop)
No I/O Operations
No file system access
No network operations
No subprocess execution
Only primitive calls allowed
Controlled Execution
No classes (Starlark limitation)
No try/except (Starlark limitation)
No f-strings (Starlark limitation)
Thread name llm_code for debugging
Print function disabled (no output leakage)
6. LLM Integration
The runLLMCode function shows the complete flow (modes.go:289-328):
Prompt Generation: System prompt from template with primitive catalog
LLM Call: Generate Starlark code via LLM Gateway
Code Execution: Run in sandbox with timeout
Result Processing: Extract answer and emit SSE events
7. Prompt Engineering
The code generation prompt provides Starlark-specific guidance (prompts.go:35-51):
"code_gen": "You write a short Starlark snippet (Python-like syntax)...
- Starlark is a Python subset: no imports, no classes, no try/except, no f-strings.
- Do NOT pass requestor_email, org, bot, or acl_tokens — the runtime injects the real values.
- Assign the final summarize(...) dict to a variable named `result`.
- Output ONLY one fenced code block (```python or ```starlark), no prose."
8. Call Recording and Observability
Every primitive call is recorded for the UI (exec.go:98-99):
*calls = append(*calls, CallRecord{
    Primitive: name,
    Args:      in,
    Summary:   summaryOf(out),
    Result:    callView(out)
})
These are emitted as SSE call events for real-time visibility.
9. Identity Injection
Security-sensitive parameters are injected by the runtime, not passed by LLM code (modes.go:311-314):
extra := identityArgs(rc)  // requestor_email, org, bot, acl_tokens
invoke := func(name string, kwargs map[string]any) (any, error) {
    return rc.Client.InvokePrimitive(ctx, name, mergeIdentity(kwargs, extra))
}
10. Performance Characteristics
Lightweight sandbox: starlark-go is pure Go, no external process
Fast startup: No interpreter initialization overhead
Timeout protection: Default 10 seconds, configurable
Memory safe: Go's garbage collection handles cleanup
Concurrent safe: Each execution gets its own thread
Key Insights
Starlark provides real sandboxing unlike Python's exec() which requires careful restriction
Type conversion is comprehensive supporting all JSON-like types plus None
Security is multi-layered: language restrictions + timeout + no I/O + identity injection
LLM-friendly: Permissive syntax options make it feel like Python
Observable: Every primitive call is recorded and streamed
Production-ready: Comprehensive test coverage including security scenarios
The Starlark integration represents a sophisticated solution to the problem of safely executing LLM-generated code, balancing security, flexibility, and observability in a way that's both developer-friendly and production-safe.
