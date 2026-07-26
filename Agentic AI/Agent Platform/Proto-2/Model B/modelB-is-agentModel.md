# **Mode B in the asap_test workspace is indeed the same as the Agent Mode (runAgent)**

**Mode B in the asap_test workspace is indeed the same as the Agent Mode (runAgent) pattern** described in the Recipe Spectrum concept. Here's the evidence:

## Key Similarities:

1. **ReAct-style Agent Loop**: Mode B implements a dynamic ReAct agent that:
    - Runs up to 12 turns (`MAX_TURNS = 12`)
    - Makes tool calls dynamically based on observations
    - Maintains conversation history with tool results
    - Can reassess and refine its approach
2. **Tool-based Execution**: The agent has access to tools like:
    - `rewrite_query` - Query normalization
    - `search` - Retrieve from data sources
    - `fetch` - Expand documents
    - `load_skill` - Load user-written recipes
    - `rerank`, `filter`, `assess` - Evaluation tools
    - `synthesize` - Final answer generation
3. **Dynamic Decision Making**: Unlike Mode A's pre-planned DAG, Mode B:
    - Decides each step based on previous results
    - Can retry failed sources
    - Handles authentication errors with user prompts
    - Tracks "thoughts" across turns for context
4. **Capability Mode Support**: Mode B supports two tool surfaces:
    - **Raw tools**: Direct MCP tool catalog access
    - **Capability mode**: Higher-level verbs (search, query, resolve, fetch, scan, exec)
5. **Recipe Integration**: When running with a recipe:
    - Restricts sources to recipe-defined sources
    - Extracts structured fields per recipe schema
    - Uses recipe guidance to inform decisions

This aligns with the "typed stream" model (like Hera) mentioned in the Notion doc, where each tool call and result is a first-class message in the agent's conversation, as opposed to Mode A's "bundled" approach where everything is planned upfront.

The implementation shows Mode B is the agentic, iterative execution engine that represents one end of the Recipe Spectrum - the flexible, runtime-adaptive approach versus Mode A's deterministic, planned execution.
