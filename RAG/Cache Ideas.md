Great question. In RAG systems there are **multiple cache layers**, each speeding up a different stage of the pipeline. Here’s a practical menu of what teams actually cache, how they key it, and how they keep it fresh.

# 1) Embedding cache (text → vector)

**What:** Cache model outputs for `embed(text)` so you don’t re-embed the same query/chunk.
**Key:** `hash(model_ver, text_norm)`
**Value:** embedding vector (+ optional l2-norm).
**Notes:**

* Normalize `text` (lowercase, trim, collapse spaces) before hashing.
* Invalidate on **model version** change only.
* Store in a fast KV (Redis/RocksDB); large vectors compress well with FP16.

# 2) Retriever-result cache (query → top-K candidates)

**What:** Cache the **IDs+scores** returned by your retrievers (keyword / ANN / hybrid), before post-filtering.
**Key (important):**

```
tenant_id |
retriever_id |                 # e.g., "kb_keyword", "kb_knn"
query_norm_hash |              # or semantic hash, see §9
corpus_epoch |                 # increments when your index changes
retriever_params_hash          # k, filters, fields, RRF weights...
```

**Value:** `[ (doc_id, score, shard, index_ts) x K ]`
**Freshness:** bump `corpus_epoch` when you reindex or cross a watermark (`last_updated_at >= W`).
**TTL:** 15–120 minutes (short), plus epoch invalidation.

# 3) ACL/post-filter cache (candidate IDs → permitted IDs)

**What:** Cache the result of **permissions filtering** over a candidate set.
**Key:**

```
tenant_id | user_id_or_rolehash | candidate_set_hash | acl_snapshot_id
```

**Value:** permitted IDs (keep order).
**Freshness:** tie to an **ACL snapshot/version**; invalidate on token set change or after short TTL (e.g., 5–15 min).
**Why:** Huge latency win when many users share similar candidate sets but different perms.

# 4) Reranker feature cache (pairwise/query-doc features)

**What:** Inputs to your cross-encoder/reranker (tokenized pairs, BM25, freshness features).
**Key:** `hash(model_ver, query_norm_hash, doc_id, doc_feature_ver)`
**Value:** precomputed features or even **reranker scores** if model version is stable.
**TTL:** long (days) unless doc or model changes.

# 5) Snippet/summarization cache (doc → chunk summary or query-focused snippet)

**What:** Pre- or on-demand summaries for chunks.
**Key:**

* Query-agnostic: `doc_id | chunk_id | summary_ver`
* Query-focused: add `query_sem_hash` (see §9).
  **Invalidate:** on doc update or summarizer version bump.

# 6) Generation cache (“prompt → answer”)

**What:** Cache LLM final answers when safe.
**Key:** `hash(model_id, prompt_template_ver, grounded_citations_hash, user_personalization_hash)`
**Use when:** The answer is **non-time-sensitive** and depends on **stable sources** (e.g., internal HR policy).
**TTL:** hours–days; tag with source digests so updating a cited doc invalidates prior answers.

# 7) Tool/API result cache (web/db calls during grounding)

**What:** Results of slow API calls (SharePoint, Confluence, Jira, etc.).
**Key:** `service | request_params_hash | auth_scope_hash`
**TTL:** service-appropriate (minutes to hours). Respect data freshness SLAs.

# 8) Vector search “index-side” cache

**What:** Many ANN stacks benefit from caching hot **inverted lists / graph neighbors** or **top-K** for common queries.
**How:** Engine-level LRU caches + server “warm lists” for frequent queries; pin hot shards in RAM.

# 9) Semantic cache keys (handling paraphrases)

Beyond exact hashes, use a **semantic key** so “reset SSO” ≈ “how to reset single sign-on” hits the same cache:

* Compute the **query embedding**, then derive a short key:

  * **Quantized vector** (e.g., product quantization buckets), or
  * **Locality-Sensitive Hash** (SimHash/LSH) over the embedding.
* Combine with stricter disambiguators: `tenant | retriever_id | sem_key | filters_hash | corpus_epoch`.

# 10) Invalidation & freshness patterns

* **Epoch bump:** Maintain a `corpus_epoch` per index; increment on reindex or when `last_updated_at` crosses a watermark.
* **Doc digests:** Store `doc_content_hash` alongside cached scores/snippets; mismatch → invalidate just those entries.
* **Versioning:** Any model/config change (embedder, reranker, prompt template) is a **versioned dimension in the key**.
* **TTL tiers:** Shorter TTL for retriever caches, longer for embedding/feature caches.
* **Permissions:** Never share user-scoped caches across users; prefer **role/permission-set hashes** where possible.

# 11) Multi-tenant & security

* Scope every key by `tenant_id`.
* For ACL caches, prefer hashing the **set of permission tokens** rather than storing tokens raw.
* Encrypt at rest for caches that hold content/snippets.

# 12) Eviction, sizing, and metrics

* **Stores:** Redis (hot path), RocksDB/SQLite/DuckDB (warm), S3/GCS (cold summaries).
* **Policies:** LFU or TinyLFU (better than LRU for skewed traffic).
* **Metrics to watch:** hit rate by layer, p50/p90 latency delta with/without cache, stale-read rate, memory pressure, key cardinality, and per-tenant quotas.

# 13) Example key schemes (copy/paste)

**Retriever-result cache (Redis key):**

```
rag:retres:v1:
  t:{tenant}:
  r:{retriever_id}:
  q:{sha1(query_norm)}:
  e:{corpus_epoch}:
  p:{sha1(params_json)}
```

**ACL post-filter cache:**

```
rag:acl:v1:
  t:{tenant}:
  u:{sha1(user_perm_tokens)}:
  c:{sha1(candidate_ids_sorted)}:
  a:{acl_snapshot_id}
```

**Semantic query key (SimHash over embedding):**

```
rag:qsem:v2:t:{tenant}:h:{simhash64(emb)}:m:{embed_model_ver}
```

# 14) Control-flow (typical fast path)

1. **Embedding cache** get(query) → miss → embed → set.
2. **Retriever cache** get(tenant, retriever, sem_key, epoch, params) → miss → search → set.
3. **ACL cache** get(user_perm_hash, candidate_set) → miss → filter → set.
4. **Rerank feature/score cache** get(...) → compute if needed → set.
5. **Snippet cache** (optional) → set.
6. **Generation cache** (if eligible) → set.

# 15) Gotchas / when *not* to cache

* Highly time-sensitive queries (“today”, “latest price”, incidents).
* User-private content that changes often (draft docs).
* Mixed-filter queries where filters explode key cardinality—hash filters separately to avoid cache bloat.

---

If you share a bit about your current stack (OpenSearch-only vs. hybrid vector DB, what you use for ACLs, and where your latency hotspot is), I can sketch a drop-in cache plan with concrete keys, TTLs, and invalidation hooks tailored to your services.
