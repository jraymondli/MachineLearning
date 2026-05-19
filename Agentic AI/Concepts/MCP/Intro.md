MCP usually refers to the **Model Context Protocol** — an open protocol for connecting AI models/agents to external tools, data sources, and applications in a standardized way.

It has become popular because people want LLMs to interact with:

* databases
* APIs
* local files
* IDEs
* browsers
* Slack/Gmail/Notion
* internal company systems

without every AI app inventing its own custom integration layer.

A good mental model is:

> MCP is somewhat like “USB-C for AI tools.”

It defines:

* how an AI discovers tools
* how tools describe themselves
* how requests/responses are exchanged
* how permissions/context are handled

---

# Core Idea

Without MCP:

```text
AI App A  -> custom integration -> GitHub
AI App B  -> different integration -> GitHub
AI App C  -> another integration -> GitHub
```

With MCP:

```text
AI Apps <-> MCP <-> Tool Servers
```

A single MCP-compatible tool server can work with many AI systems.

---

# Main Components

## 1. MCP Host

The AI application itself.

Examples:

* Claude Desktop
* IDE assistants
* AI agents
* terminal copilots

The host:

* talks to the user
* decides which tool to use
* sends requests to MCP servers

---

## 2. MCP Server

A process exposing capabilities/tools/resources.

Examples:

* GitHub MCP server
* PostgreSQL MCP server
* filesystem MCP server
* Slack MCP server

The server advertises:

* available tools
* schemas
* inputs/outputs
* resources

---

## 3. Tools

Functions the model can call.

Example:

```json
{
  "name": "search_code",
  "description": "Search repository source code",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": { "type": "string" }
    }
  }
}
```

The model can then invoke:

```json
{
  "tool": "search_code",
  "arguments": {
    "query": "asyncio timeout"
  }
}
```

---

## 4. Resources

Structured data the model can read.

Examples:

* files
* documents
* logs
* database rows
* wiki pages

Think of resources as “readable context.”

---

## 5. Prompts

Reusable prompt templates exposed by the server.

Example:

* “summarize this PR”
* “review this SQL migration”

---

# Why MCP Matters

## Before MCP

AI integrations were:

* fragile
* vendor-specific
* hard to reuse
* inconsistent

Every tool integration needed:

* custom auth
* custom schemas
* custom discovery

---

## With MCP

You get:

* interoperability
* reusable integrations
* standardized tool calling
* easier agent development

This is especially important for enterprise AI systems.

---

# Typical Architecture

```text
+-------------------+
| AI Client / Host  |
| (Claude/Desktop)  |
+---------+---------+
          |
          | MCP Protocol
          |
+---------v---------+
| MCP Server        |
| GitHub / DB / FS  |
+-------------------+
```

Communication is usually over:

* stdio
* WebSocket
* HTTP streams

JSON-RPC style messaging is common.

---

# Example Workflow

Suppose you ask:

> “Find all timeout-related bugs in our repo.”

The AI host may:

1. Discover GitHub MCP tools
2. Call:

   * search_code
   * list_pull_requests
   * read_issue
3. Gather context
4. Produce a response

All through standardized MCP interfaces.

---

# Common MCP Use Cases

## Engineering

* code search
* PR review
* log analysis
* deployment tooling
* database inspection

## Enterprise Knowledge

* Notion
* Confluence
* Google Drive
* Slack

## Data/Analytics

* SQL querying
* dashboards
* metrics retrieval

## Local Desktop Automation

* filesystem access
* terminal commands
* local applications

---

# MCP vs Traditional APIs

Traditional API:

```text
Your code directly calls API
```

MCP:

```text
AI model dynamically discovers and uses tools
```

MCP adds:

* discoverability
* schemas
* AI-oriented interaction patterns
* context management

---

# MCP vs Function Calling

They are related but different.

## Function Calling

Usually:

* model-specific
* embedded in one platform

Example:

* OpenAI tool calling
* Anthropic tools

---

## MCP

A broader interoperability layer:

* externalized
* reusable
* standardized across hosts

Think:

```text
Function calling = capability
MCP = ecosystem/protocol around it
```

---

# Simple Example MCP Server (Python)

Very simplified conceptual example:

```python
from mcp import Server

server = Server("demo")

@server.tool()
def add(a: int, b: int) -> int:
    return a + b

server.run()
```

Then an MCP-compatible AI client can discover and use `add`.

---

# Why Engineers Care

MCP is becoming important because it enables:

* AI-native tooling ecosystems
* agent frameworks
* reusable enterprise integrations
* safer tool access patterns
* multi-agent systems

Especially for someone working on:

* distributed systems
* retrieval systems
* AI infrastructure
* developer tooling

you’ll likely see MCP increasingly used as a standard integration layer.

---

# Relationship to AI Agents

MCP is often the “tool layer” underneath agents.

An agent:

* plans
* reasons
* chooses actions

MCP:

* provides standardized access to actions/tools/data

---

# Limitations / Open Challenges

MCP does not magically solve:

* security
* permissions
* hallucinations
* bad tool selection
* cost/latency

Large-scale production deployments still need:

* auth controls
* observability
* rate limiting
* sandboxing
* audit logging

---

# Where It’s Headed

The ecosystem is moving toward:

* shared tool registries
* enterprise MCP gateways
* agent-to-agent interoperability
* multimodal tools
* streaming tool outputs
* long-running workflows

Many people view MCP as an early foundational layer for practical AI agents.
