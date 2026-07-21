Yes—**3 minutes and $0.765 per query is expensive for a production people-search system**, unless this is a rare, high-value research workflow where each correct result is worth tens or hundreds of dollars.

For a query like:

> “Experts on Python backend services whose last name is Wang”

most of the work is symbolic and deterministic:

* `last_name = Wang`
* expertise includes Python
* experience relates to backend services
* rank candidates by evidence strength

An LLM should not need to reason for three minutes over the full corpus.

## How I would judge the current performance

**Latency**

* **3 minutes:** acceptable for an early prototype or asynchronous deep research.
* **30–60 seconds:** still slow, but potentially acceptable for a complex enterprise search.
* **5–15 seconds:** reasonable interactive target.
* **Under 5 seconds:** ideal when employee profiles and expertise signals are already indexed.

**Cost**

* **$0.765/query:** high for ordinary search.
* At 1,000 queries/day: **$765/day**, roughly **$23,000/month**.
* At 10,000 queries/day: approximately **$230,000/month**.

A plausible production target would be:

* **$0.01–$0.05** for straightforward structured queries.
* **$0.05–$0.15** for ambiguous or multi-step expert discovery.
* Higher cost only when the user explicitly requests deep research or detailed justification.

## How neuro-symbolic architecture can help

For this example, neuro-symbolic methods could probably improve:

* **Latency by 10–30×**
* **LLM cost by 10–50×**
* Potentially more, depending on how much raw data the current MCP agents inspect repeatedly.

That could move the query from:

| Metric                  |       Current |   Realistic target |
| ----------------------- | ------------: | -----------------: |
| Latency                 |   180 seconds |       5–20 seconds |
| LLM cost                |        $0.765 |       $0.015–$0.08 |
| LLM calls               | Possibly many |                1–3 |
| Corpus inspected by LLM |         Broad | Tens of candidates |

The biggest gain does not come from making the LLM reason more cleverly. It comes from ensuring the LLM **does not do work that a query engine, ontology, rules engine, or graph traversal can do more cheaply and reliably**.

## Recommended execution flow

### 1. Convert natural language into a symbolic plan

Use a small model or constrained decoding to produce something like:

```json
{
  "entity_type": "person",
  "filters": [
    {"field": "last_name", "operator": "equals", "value": "Wang"},
    {
      "field": "expertise",
      "operator": "semantic_match",
      "value": "Python backend services"
    }
  ],
  "ranking": {
    "signals": [
      "project_experience",
      "code_contributions",
      "service_ownership",
      "profile_skills"
    ]
  }
}
```

This should take one quick model call, or even be handled by a fine-tuned classifier/parser.

### 2. Execute hard constraints symbolically

Run `last_name = Wang` directly through:

* SQL
* OpenSearch filters
* a knowledge graph
* an employee directory API
* a precomputed entity index

Do not ask an LLM to inspect names or determine whether someone’s last name is Wang.

### 3. Retrieve expertise evidence

Search only within the filtered candidate set using:

* BM25
* embeddings
* skill taxonomy expansion
* graph relationships
* code ownership and project metadata

For example:

```text
Python backend services
→ Python, Django, Flask, FastAPI
→ API development, microservices, service ownership
→ backend engineer, platform engineer
```

This is where the “neural” component is valuable: query expansion, semantic matching and evidence interpretation.

### 4. Apply symbolic evidence aggregation

Create explicit scoring rules, such as:

```text
0.30 × profile skill match
0.25 × relevant repository contributions
0.20 × backend service ownership
0.15 × relevant project participation
0.10 × recency
```

The weights can later be learned from feedback, but the execution remains cheap and explainable.

### 5. Let the LLM verify and summarize only the top candidates

Give the LLM perhaps 10–30 candidates and their supporting evidence—not the entire organization.

The final call should answer:

* Who matches?
* Why does each person match?
* How confident are we?
* What evidence is missing or contradictory?

## Where the improvement will really come from

I would separate the anticipated savings as follows:

* **Deterministic filtering:** 2–5× latency and cost reduction.
* **Candidate-set restriction before LLM processing:** 3–10×.
* **Parallel rather than sequential MCP calls:** 2–5× latency reduction.
* **Caching entity resolution and expertise extraction:** 2–10× for repeated searches.
* **Smaller model for planning/routing:** 3–20× cost reduction for that stage.
* **Precomputed expertise graph or index:** potentially another 5–20× improvement.

These factors are not fully multiplicative, but together they make a **10–50× cost reduction and 10–30× latency reduction credible**.

## One important caveat

This query is unusually favorable to neuro-symbolic execution because it contains a clear hard constraint. Harder questions such as:

> “Who could help rescue a Python service suffering from intermittent event-loop stalls?”

require more semantic reasoning, inference from indirect evidence, and perhaps multiple data sources. Those might still take **15–45 seconds** and cost **$0.10–$0.30**, depending on how much evidence synthesis is required.

So I would classify queries first:

* **Structured lookup:** mostly symbolic.
* **Semantic expert matching:** retrieval plus lightweight LLM reranking.
* **Exploratory investigation:** agentic workflow with a larger budget.

## My assessment

Your current LLM + MCP system is probably functioning more like a **general-purpose research agent** than a purpose-built expert-search engine. That is useful for validating the product, but its current economics are unlikely to scale.

For the Wang query, I would set an initial engineering goal of:

* **P50 latency under 10 seconds**
* **P95 under 25 seconds**
* **Average LLM cost under $0.05**
* **One planning call and one final reranking/explanation call**
* **No more than 20–50 candidate profiles exposed to the LLM**

Neuro-symbolic design can help substantially, but the practical architecture is less about classical theorem proving and more about **typed query planning, deterministic constraint execution, ontology-backed retrieval, graph or index operations, and narrowly scoped LLM judgment**.
