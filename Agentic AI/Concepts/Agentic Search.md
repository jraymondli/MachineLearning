

1. Federated Search
2. Multi-hop Reasoning
3. Seamless mode switch


---
# Seamless Mode Switch

In **agentic search**, a *seamless mode switch* means the agent can **change how it searches or reasons—on the fly—without the user experiencing a reset, delay spike, or loss of context**. The answer keeps streaming, state/citations persist, and the agent quietly pivots under the hood to a better strategy.

# What “modes” are we switching between?

* **Answering strategy:** LLM-only ⇄ RAG (retrieve-then-read) ⇄ multi-hop planning.
* **Retrieval:** BM25/keyword ⇄ dense/semantic ⇄ **hybrid** ⇄ tool-specific APIs.
* **Scope:** private corpus ⇄ public web ⇄ both (with ACLs).
* **Depth/quality tiers:** fast/cheap (shallow) ⇄ slower/deeper (breadth, multi-agent) when needed.
* **Fallbacks:** primary index down ⇄ backup index/web; extractive QA ⇄ generative summary; strict safety mode on risky inputs.

# When should the agent switch?

Policy triggers commonly include:

* **Low confidence / high uncertainty** (answer entropy, contradicting sources, missing citation).
* **Intent cues** (“latest”, “price”, “compare”, “near me”) → browse/tools.
* **Recall/coverage gaps** (few/low-scored hits, topic shift).
* **Runtime signals** (timeouts, rate limits, 5xx, quota).
* **Safety/guardrails** (content category → stricter tools/prompts).

# How to make it *seamless* (practical)

* **Single orchestrator + policy**: decide the next mode based on confidence/telemetry; never throw away chat state.
* **Atomic handoff**: cancel stale tasks; keep last good result in memory; continue streaming while the new mode works.
* **Unified doc schema**: normalize results from BM25/semantic/web; dedupe by ID/URL; re-rank together.
* **Hedged execution**: kick off two cheap alternatives; pick the best by final confidence/grounding.
* **Stale-while-revalidate**: return a grounded draft from local index; quietly upgrade with web/deeper pass if it improves confidence.
* **Idempotent tools** + **deadline budgets**: prevent loops/tool spam.

# Tiny orchestrator sketch (async Python; fits your stack)

```python
async def agentic_search(q, deadline):
    # Fire fast strategies immediately
    bm25 = asyncio.create_task(bm25_retrieve(q))
    dense = asyncio.create_task(dense_retrieve(q))

    # Pick best initial evidence
    initial = await pick_best([bm25, dense], by="coverage@k, diversity")

    draft = llm_answer(q, initial.docs, stream=True)  # start streaming to user

    # Decide if we must switch modes (e.g., browse or multi-hop)
    if low_confidence(draft.metrics) or needs_freshness(q):
        web = asyncio.create_task(web_retrieve(q))
        multi = asyncio.create_task(multi_hop_plan(q, initial.docs))
        better = await pick_best([web, multi], within=deadline.remaining())

        if improves(draft, better):
            cancel_others([bm25, dense, web, multi])
            draft.upgrade_with(better.docs)  # continue the same stream seamlessly

    return draft.finalize()
```

# Acceptance checklist (what “seamless” looks like)

* ✅ No UI flicker; the answer stream doesn’t restart.
* ✅ Citations/state survive the switch; added evidence appears as *additional* citations.
* ✅ Latency is capped (budgeted); timeouts trigger graceful fallbacks.
* ✅ Duplicates removed; rankings stable when new docs arrive.
* ✅ Clear but unobtrusive UX (“updated with web sources”).

# Metrics to track

* **Switch rate** & win rate (how often switching improved confidence/accuracy).
* **Final latency p95** vs single-mode baseline.
* **Coverage\@k / answerable\@k** before vs after switch.
* **Contradiction/consistency** checks across sources.
* **Cost per query** with/without deep modes.

# Common pitfalls (and fixes)

* **Answer flicker/contradictions** → freeze finalized spans; add reconciliation step before upgrading.
* **Tool storms** → per-query caps, exponential backoff, dedup keys.
* **Leaking private data to web prompts** → strict context partitioning + redaction.
* **Hotspot modes** (e.g., dense only) → hybrid first, then specialize.

If you tell me your current pipeline (e.g., OpenSearch BM25 + KNN, web browse, reranker), I can sketch a concrete policy table and drop-in orchestrator code that plugs into your asyncio/gRPC services.


