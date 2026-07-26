

### Frontend (React Application)

- **Main Component**: /moveworks/js/packages/webbot/src/app/webbot/RecipeSpectrum/index.tsx
- **API Layer**: /moveworks/js/packages/webbot/src/app/webbot/RecipeSpectrum/api.ts
- **Route**: Mounted at `/recipe_spectrum` path (RECIPE_SPECTRUM_PATH constant)

### Key Features

The RecipeSpectrum is a side-by-side comparison tool showing 4 execution strategies:

1. **Fixed**: Hardcoded recipe execution
2. **LLM-Recipe**: LLM generates the recipe
3. **LLM-Code**: LLM generates code for execution
4. **Agent**: Full agentic tool loop

### Data Flow

1. **Frontend** (React) → Makes requests to `/recipe-demo/*` endpoints
2. **server-webbot proxy** → Routes `/recipe-demo` to florence-go-prototype backend
3. **florence-go-prototype** → Runs on port 28401 locally, serves the recipe demo HTTP/SSE API

### Backend Service

The backend is provided by **florence-go-prototype** with these key endpoints:

- `/api/prompts` - Returns default prompts for each mode
- `/api/recipe` - Returns the fixed recipe configuration
- `/api/run` - SSE endpoint that streams execution results

### Deployment

- **Production**: Served at https://cockpit.moveworks.io/recipe_spectrum
- **Service**: webbot-web-server (deployed across multiple regions)
- **Proxy**: server-webbot provides the `/recipe-demo` proxy to florence-go-prototype
- **Authentication**: Protected by JWT middleware

### Key Implementation Details by Weizhen

Based on git history, Weizhen (GitHub: weizhen) implemented:

1. **Initial Recipe Executor** (SEARCH-3950): Core recipe executor prototype in florence-go-prototype with search-DAG engine and primitives
2. **Cockpit Integration** (SEARCH-3951): Added Recipe Spectrum tab to cockpit with server-webbot streaming proxy
3. **Performance Optimizations** (SEARCH-4131): Truncate results to 50 before rerank
4. **Native Rank Primitive** (SEARCH-4441): Added ranking primitive to the recipe executor
5. **Disable Enforcement** (SEARCH-4619): Added per-run primitive disable enforcement

Recipe Executor Walkthrough

### Local Development

To run RecipeSpectrum locally:

```bash
# Start florence-go-prototype backend
bazel run //moveworks/services/florence_go_prototype:start_local -- --demo_http_port 28401

# Start webbot frontend
cd moveworks/js && npm run dev --workspace=webbot
```

The RecipeSpectrum uses Server-Sent Events (SSE) for real-time streaming of execution progress, showing DAG visualization, tool calls, search results, and generated answers for each execution mode.
