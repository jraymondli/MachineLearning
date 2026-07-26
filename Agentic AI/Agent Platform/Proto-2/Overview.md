# Summary of ASAP Test Workspace Analysis

Based on my exploration of the `/Users/rayli/code/asap_test` workspace, here are the key findings:

### Architecture Overview

The ASAP (Agentic Search · Side-by-Side) workspace implements a dual-execution pattern that aligns with the **Recipe Spectrum** concept mentioned in the Notion documentation:

1. **Mode A - Planned Execution** (`modeA.ts`):
    - Pre-planned DAG-based execution
    - The model designs the full pipeline upfront
    - Steps: rewrite → plan → retrieve → rerank → filter → synthesize
    - Deterministic execution with dataflow dependencies
2. **Mode B - Agentic Execution** (`modeB.ts`):
    - ReAct-style iterative execution
    - Model uses tools dynamically in a loop
    - Supports up to 12 turns with assessment and refinement
    - Can load skills on-demand and make runtime decisions

### Similar to Recipe Spectrum Pattern

The codebase exhibits the **"two core engines at opposite ends of the spectrum"** pattern described in the Notion doc:

1. **Bundled Model (like Minerva)**: Mode A bundles reasoning into a planned execution with a final synthesis
2. **Typed Stream Model (like Hera)**: Mode B streams typed messages (tool calls, results, reasoning)

### Key Components

#### Execution Engine (`/server/src/run/`)

- `runner.ts`: Orchestrates side-by-side execution of both modes
- `context.ts`: Manages run context and event emission
- Both modes run in parallel for comparison

#### Recipe System (`/server/src/recipes.ts`)

- Recipes bundle sources, guidance, fields, and templates
- JSON Schema-driven field extraction
- Structured data collection with text and list field types

#### Capability Layer (`/server/src/capability/`)

- `executor.ts`: Deterministic verb → MCP-call expansion
- Verbs: search, query, resolve, fetch, scan, exec
- Abstraction over raw MCP tools for higher-level operations

#### Primitives (`/server/src/primitives/`)

- Reusable building blocks: rewrite, retrieve, rerank, filter, synthesize, assess
- Both modes compose these same primitives differently

### MCP Integration

- Multiple data sources (Notion, Linear, Slack, GitHub, etc.)
- Tools are filtered to read-only operations
- Capability manifests provide source-specific adaptations

### Recipe Execution Flow

1. Recipe defines structured fields and guidance
2. Mode A plans the entire retrieval pipeline
3. Mode B iteratively retrieves and assesses
4. Both modes extract structured data per recipe schema
5. Judge compares results from both modes

This architecture demonstrates a sophisticated approach to comparing planned vs. agentic execution strategies, with recipes providing structured templates for specific search use cases.
