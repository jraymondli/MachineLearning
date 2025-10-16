Here’s a quick, practical intro to DCG—exactly the metric you’d use to judge how good a ranked list is (e.g., your KNN vs keyword results in RAG).

# What is DCG?

**Discounted Cumulative Gain** measures the usefulness (“gain”) of items in a ranking, **discounted** by their position so that higher-ranked items count more.

* **Gain** comes from the item’s **relevance grade** (e.g., 0,1,2,3…).
* **Discount** reduces the contribution of lower ranks (because users rarely read far down the list).

# Formula

Two common variants for **DCG@k**:

1. Linear gain
   [
   \text{DCG@k}=\sum_{i=1}^{k}\frac{\text{rel}_i}{\log_2(i+1)}
   ]

2. Exponential (popular; spreads out higher grades more)
   [
   \text{DCG@k}=\sum_{i=1}^{k}\frac{2^{\text{rel}_i}-1}{\log_2(i+1)}
   ]

* ( \text{rel}_i ) is the graded relevance of the item at rank ( i ) (rank starts at 1).
* The ( \log_2(i+1) ) term is the discount (rank 1 has denominator (\log_2 2 = 1), no discount).

# nDCG (normalized)

Different queries have different “best possible” gains, so we normalize:

[
\text{nDCG@k} = \frac{\text{DCG@k}}{\text{IDCG@k}}
]

* **IDCG@k** is the DCG of the **ideal** ranking (items sorted by relevance).
* Range: 0–1. If all relevances are 0, define nDCG as 0.

# Tiny example

Suppose at ranks 1..5 you have graded relevances: `[3, 2, 3, 0, 1]`.

Exponential DCG@5:

* r1: ((2^3-1)/\log_2(2)=7/1=7)
* r2: ((2^2-1)/\log_2(3)=3/1.585=1.892)
* r3: ((2^3-1)/\log_2(4)=7/2=3.5)
* r4: ((2^0-1)/\log_2(5)=0)
* r5: ((2^1-1)/\log_2(6)=1/2.585=0.387)

DCG@5 ≈ **12.78**.
Compute IDCG@5 by sorting labels descending (3,3,2,1,0) and repeat; then nDCG@5 = DCG/IDCG.

# Why DCG/nDCG for RAG?

* **Graded relevance** suits retrieval tasks (exact match vs partially relevant vs off-topic).
* **Position-sensitive**: rewards bringing the best contexts to the top (crucial for generators that read only top-k).
* **Comparable**: nDCG gives you a 0–1 score to compare keyword vs KNN, reranker variants, or different top-K settings.

# Practical tips

* Use **exponential gain** if you care a lot about separating “great” from “good”.
* Always report **@k** explicitly (e.g., nDCG@3, nDCG@10).
* When the **ideal list is empty** (all zeros), set nDCG to 0 for that query and average over queries.
* For A/Bs, average nDCG across queries; also look at **early-k** (e.g., @3) for UX.

# Minimal Python (drop-in)

```python
import math

def dcg(rels, k, exp=True):
    s = 0.0
    for i, rel in enumerate(rels[:k], start=1):
        gain = (2**rel - 1) if exp else rel
        s += gain / math.log2(i + 1)
    return s

def ndcg(rels, k, exp=True):
    ideal = sorted(rels, reverse=True)
    idcg = dcg(ideal, k, exp=exp)
    return 0.0 if idcg == 0 else dcg(rels, k, exp=exp) / idcg
```

If you want, I can wire this into your evaluation loop to compare **KNN-only**, **Keyword-only**, and **combined** rankings.
