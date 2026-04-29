# Offline Evaluation Framework — Technical Specification

**Context:** Production-grade spec for a consumer marketplace at Poshmark/Depop scale.
Implicit feedback logs · MongoDB Atlas BM25 baseline · No existing A/B infra · ~1 year of logs.

---

## Table of Contents

1. [Log Schema Design](#1-log-schema-design)
2. [Dataset Construction](#2-dataset-construction)
3. [Metrics](#3-metrics)
4. [Counterfactual Evaluation](#4-counterfactual-evaluation)
5. [Evaluation Harness Architecture](#5-evaluation-harness-architecture)
6. [Offline vs. Online Metric Gap](#6-offline-vs-online-metric-gap)
7. [Cold Start and Edge Cases](#7-cold-start-and-edge-cases)
8. [Practical Implementation Plan](#8-practical-implementation-plan)

---

## 1. Log Schema Design

> **What naive implementations get wrong:** Logging only `item_id + timestamp + user_id`. Fatal consequences: (1) no position field means position bias is uncorrectable, (2) no session/query grouping makes joins ambiguous, (3) no `schema_version` means you cannot safely reparse old logs after field additions. All three will bite you within the first month.

### 1.1 Impression Event Schema

One row per item shown in one result set. Do **not** aggregate — keep at item granularity.

| Field | Type | Required | Purpose / Notes |
|---|---|---|---|
| `event_id` | UUID | Yes | Idempotent deduplication key. Use server-generated UUIDv7 (time-ordered). |
| `session_id` | string | Yes | Groups all queries from one user visit. Reset on 30-min inactivity. |
| `query_id` | string | Yes* | Groups items from one search query. One session → many `query_id`s. *Required for search surfaces. |
| `user_id` | string | Yes | Stable user identity. NULL for logged-out; use `anonymous_id` cookie instead. |
| `item_id` | string | Yes | The shown listing SKU. |
| `rank_position` | int | Yes | Zero-indexed. Position 0 = first result. **Critical for IPS.** |
| `page_num` | int | Yes | Paginated page number. Position 0 on page 2 ≠ position 0 on page 1. |
| `surface` | enum | Yes | `"search"` \| `"feed"` \| `"pdp_similar"` \| `"homepage"`. Separate eval harnesses per surface. |
| `timestamp_ms` | int64 | Yes | Server-side Unix epoch ms. Never client-side (clock skew corrupts joins). |
| `retrieval_model_id` | string | Yes | Which retrieval system served this batch (e.g. `"bm25_atlas_v3"`). |
| `ranking_model_id` | string | Yes | Which ranker scored this (e.g. `"heuristic_price_v2"`). |
| `retrieval_score` | float | Yes | BM25 or dense retrieval score. Required for propensity model. |
| `ranking_score` | float | Yes | Final ranking score. Log pre- and post-diversity reranking separately. |
| `query_text` | string | Soft | Raw query string. Hash if PII policy requires. |
| `num_results_returned` | int | Yes | Total items in result set before truncation. Not just items on screen. |
| `viewport_visible` | bool | Yes | Was item within viewport? **Critical:** clicks on invisible items are noise. |
| `item_price_usd` | float | No | For price-aware ranking eval. |
| `item_age_days` | int | No | Days since listing posted. Freshness covariate for debiasing. |
| `experiment_assignments` | string[] | Yes | All A/B experiment IDs this user is enrolled in at time of impression. |
| `schema_version` | int | Yes | Increment on any breaking change. Version-lock eval snapshots to this. |

### 1.2 Click Event Schema

One row per distinct user action. Capture multiple click types to enable graded relevance later.

| Field | Type | Required | Notes |
|---|---|---|---|
| `event_id` | UUID | Yes | Dedup key. |
| `impression_event_id` | UUID | Yes | Direct FK to impression event. **Primary join key.** ~80% of clicks should have this. |
| `session_id` | string | Yes | Fallback join key when `impression_event_id` is missing. |
| `query_id` | string | Yes* | Fallback join key. |
| `user_id` | string | Yes | |
| `item_id` | string | Yes | Verify matches impression `item_id` in join validation step. |
| `click_type` | enum | Yes | `"pdp_view"` \| `"add_to_cart"` \| `"purchase"` \| `"like"`. Graded relevance: purchase > cart > like > view. |
| `timestamp_ms` | int64 | Yes | Must be strictly > impression `timestamp_ms`. Enforce in pipeline. |
| `rank_position_at_click` | int | Yes | Re-record position at click time. Guard against re-rankings between impression and click. |
| `dwell_time_ms` | int | No | Time on PDP before back-navigation. Quality proxy: >30s dwell is a stronger positive signal. |
| `schema_version` | int | Yes | |

### 1.3 Join Strategy: Impression → Click

Use `impression_event_id` as the primary join key (direct FK). Fall back to `(session_id, query_id, item_id)` compound key within a 30-minute session window.

```python
import pandas as pd

CLICK_WEIGHT = {'purchase': 4, 'add_to_cart': 3, 'like': 2, 'pdp_view': 1}

def join_impressions_to_clicks(
    impressions: pd.DataFrame,
    clicks: pd.DataFrame,
    session_window_minutes: int = 30,
    dedup_strategy: str = 'max_weight',  # 'max_weight' | 'first'
) -> pd.DataFrame:
    '''
    Left join: every impression row gets a relevance label (0 or weighted int).
    Handles two join paths and deduplication of multiple clicks on same item.

    Path 1 (preferred): impression.event_id = click.impression_event_id
    Path 2 (fallback): (session_id, query_id, item_id) within time window
    '''
    clicks = clicks.copy()
    clicks['rel_weight'] = clicks['click_type'].map(CLICK_WEIGHT).fillna(1)

    # Dedup: keep max-weight click per (session, query, item)
    if dedup_strategy == 'max_weight':
        clicks_deduped = (
            clicks.sort_values('rel_weight', ascending=False)
                  .drop_duplicates(subset=['session_id', 'query_id', 'item_id'])
        )
    else:
        clicks_deduped = clicks.drop_duplicates(
            subset=['session_id', 'query_id', 'item_id'], keep='first'
        )

    # --- Path 1: direct FK join ---
    direct = impressions.merge(
        clicks_deduped[['impression_event_id', 'click_type', 'rel_weight',
                         'timestamp_ms', 'dwell_time_ms']]
                       .rename(columns={'timestamp_ms': 'click_ts_ms'}),
        left_on='event_id', right_on='impression_event_id',
        how='left', suffixes=('', '_click'),
    )
    matched_mask = direct['impression_event_id'].notna()

    # --- Path 2: compound key fallback for orphan clicks ---
    orphan_clicks = clicks_deduped[clicks_deduped['impression_event_id'].isna()].copy()
    unmatched_impressions = direct[~matched_mask].drop(
        columns=[c for c in direct.columns if c.endswith('_click')]
    )
    window_ms = session_window_minutes * 60_000
    orphan_clicks['_window'] = orphan_clicks['timestamp_ms'] // window_ms
    unmatched_impressions['_window'] = unmatched_impressions['timestamp_ms'] // window_ms

    fallback = unmatched_impressions.merge(
        orphan_clicks[['session_id', 'query_id', 'item_id', 'click_type',
                        'rel_weight', 'timestamp_ms', '_window']]
                      .rename(columns={'timestamp_ms': 'click_ts_ms'}),
        on=['session_id', 'query_id', 'item_id', '_window'], how='left',
    )
    result = pd.concat([direct[matched_mask], fallback], ignore_index=True)
    result = result.drop(columns=['_window'], errors='ignore')
    result['relevance'] = result['rel_weight'].notna().astype(int)
    result['graded_relevance'] = result['rel_weight'].fillna(0).astype(int)

    # Sanity check: click must come after impression
    if 'click_ts_ms' in result.columns:
        bad = result[
            result['click_ts_ms'].notna() &
            (result['click_ts_ms'] < result['timestamp_ms'])
        ]
        if len(bad) > 0:
            print(f'WARNING: {len(bad)} clicks precede their impression -- dropping')
            result = result[
                result['click_ts_ms'].isna() |
                (result['click_ts_ms'] >= result['timestamp_ms'])
            ]
    return result
```

### 1.4 Schema Drift Handling

| Strategy | How | Trade-off |
|---|---|---|
| `schema_version` on every event | Integer field, increment on breaking changes | Required baseline; zero cost. Lock eval snapshots to a version. |
| Additive-only schema changes | New fields always optional with NULL default | Older readers ignore unknown fields; forward compatible |
| Schema Registry (Avro/Protobuf) | Confluent Schema Registry or Buf BSR | Strong guarantees; adds operational overhead. Use at 5+ teams writing logs. |
| Null-fill partial logs | Missing fields → typed NULL, not dropped rows | Preserves denominator for rate metrics. Dropped rows silently deflate impressions count. |
| Snapshot pinning | Hash the schema at eval dataset creation time; store in metadata | Enables audit: "which schema version was this eval set built on?" |

---

## 2. Dataset Construction

> **Why random train/eval splits are wrong — the precise failure mode:** Suppose user A clicks item X at time T+1 (eval), and user B sees item X at T-1 (train). A random split may put T+1 in train and T-1 in eval. Item X's popularity is inflated in training because you have observed its future clicks. Worse: user-level embedding features leak future behavior. The model learns "this item gets clicked" because it has seen clicks that, in production, would not yet exist.

### 2.1 Temporal Split with Gap

```python
from dataclasses import dataclass, field
import hashlib, json, pandas as pd

@dataclass
class EvalDatasetMetadata:
    train_start_ms: int
    train_end_ms: int
    eval_start_ms: int
    eval_end_ms: int
    gap_days: int
    n_train_queries: int
    n_eval_queries: int
    n_eval_items: int
    item_overlap_pct: float
    schema_version: int
    fingerprint: str = field(init=False)

    def __post_init__(self):
        d = {k: v for k, v in self.__dict__.items() if k != 'fingerprint'}
        self.fingerprint = hashlib.sha256(
            json.dumps(d, sort_keys=True).encode()
        ).hexdigest()[:16]


def build_temporal_splits(
    events: pd.DataFrame,
    eval_days: int = 7,
    gap_days: int = 7,
    train_days: int = 90,
) -> tuple[pd.DataFrame, pd.DataFrame, EvalDatasetMetadata]:
    '''
    Timeline:  [train_start -- train_end] [-- gap --] [eval_start -- eval_end]

    The gap prevents leakage of items whose signals span the boundary.
    For a 7-day gap: if item X was popular in the last 7 days of train,
    that recency signal does not bleed into the eval window.

    Choose eval_days >= 7 (shorter → distribution shift from weekly patterns).
    ~40-70% item overlap between train and eval is EXPECTED and FINE.
    Temporal leakage ≠ item overlap.
    '''
    t_max = int(events['timestamp_ms'].max())
    eval_end    = t_max
    eval_start  = eval_end  - eval_days  * 86_400_000
    train_end   = eval_start - gap_days  * 86_400_000
    train_start = train_end  - train_days * 86_400_000

    train = events[
        (events['timestamp_ms'] >= train_start) &
        (events['timestamp_ms'] <  train_end)
    ].copy()
    eval_ = events[
        (events['timestamp_ms'] >= eval_start) &
        (events['timestamp_ms'] <= eval_end)
    ].copy()

    train_items = set(train['item_id'].unique())
    eval_items  = set(eval_['item_id'].unique())
    overlap_pct = 100 * len(train_items & eval_items) / max(len(eval_items), 1)

    meta = EvalDatasetMetadata(
        train_start_ms=train_start, train_end_ms=train_end,
        eval_start_ms=eval_start,   eval_end_ms=eval_end,
        gap_days=gap_days,
        n_train_queries=train['query_id'].nunique(),
        n_eval_queries=eval_['query_id'].nunique(),
        n_eval_items=len(eval_items),
        item_overlap_pct=round(overlap_pct, 1),
        schema_version=int(events['schema_version'].max()),
    )
    return train, eval_, meta
```

### 2.2 Position Bias and Propensity Estimation

Position bias follows the **examination hypothesis** (Joachims et al., 2017 WSDM): a user can only click an item if they examine it first. Propensity `p(k)` = P(examine | position=k).

The standard empirical estimator uses click-through rate by position, then fits a power law. A better approach (if you have it) is a **randomization experiment** — randomly swap items between rank positions in 1% of traffic. See: Agarwal et al. (KDD 2019, "Estimating Position Bias without Intrusive Interventions").

```python
import numpy as np
from scipy.optimize import curve_fit
import pandas as pd

def estimate_position_propensities(
    df: pd.DataFrame,
    max_rank: int = 20,
    min_impressions: int = 1000,
) -> np.ndarray:
    stats = (
        df[df['viewport_visible'] == True]   # only count viewable impressions
          .groupby('rank_position')['relevance']
          .agg(clicks='sum', impressions='count')
          .assign(ctr=lambda x: x['clicks'] / x['impressions'])
          .query(f'impressions >= {min_impressions}')
          .reindex(range(max_rank))
    )
    valid_mask = stats['ctr'].notna() & (stats['ctr'] > 0)
    ranks = stats.index[valid_mask].values.astype(float)
    ctrs  = stats['ctr'][valid_mask].values

    def power_law(k, a, b):
        return a * np.power(k + 1.0, -b)

    try:
        (a, b), _ = curve_fit(power_law, ranks, ctrs, p0=[0.1, 0.5], maxfev=10000)
    except RuntimeError:
        b, a = 1.0, ctrs[0]   # fallback: 1/rank heuristic

    raw = power_law(np.arange(max_rank), a, b)
    return raw / raw[0]   # normalize: p(rank=0) = 1.0


def snips_weighted_ndcg(
    qrels: pd.DataFrame,      # cols: query_id, item_id, rank_position, relevance
    propensities: np.ndarray,
    k: int = 10,
    clip_weight: float = 10.0,
) -> float:
    '''
    Self-Normalized IPS (SNIPS) estimator for NDCG@K.

    Standard IPS: V = (1/N) Σ_i [r_i / p_i]
      → unbiased but high variance when p_i is small (rare positions).

    SNIPS (Swaminathan & Joachims, ICML 2015):
      V_SNIPS = Σ_i [r_i / p_i] / Σ_i [1 / p_i]
      → slight bias, much lower variance. Use SNIPS in practice.
    '''
    def _query_snips_ndcg(group: pd.DataFrame) -> float:
        g = group.sort_values('rank_position').head(k).copy()
        pos     = g['rank_position'].clip(0, len(propensities) - 1).values
        props   = np.maximum(propensities[pos], 1.0 / clip_weight)
        weights = np.minimum(1.0 / props, clip_weight)
        discounts = np.log2(np.arange(2, len(g) + 2))
        dcg  = (g['relevance'].values * weights / discounts).sum()
        ideal = np.sort(g['relevance'].values)[::-1]
        idcg = (ideal / discounts[:len(ideal)]).sum()
        if idcg == 0:
            return float('nan')
        snips_norm = weights.sum() / len(g)
        return float((dcg / idcg) / snips_norm)

    per_query = qrels.groupby('query_id').apply(_query_snips_ndcg)
    return float(per_query.dropna().mean())
```

### 2.3 Selection Bias: Evaluating Retrieval Itself

> **The counterfactual retrieval problem:** Clicks are only observable on items that were retrieved. If BM25 never returns item X for query Q, you have zero click signal about X's relevance to Q. Any offline eval of retrieval recall is limited to the logging policy's candidate set. Offline Recall@K is optimistic by construction.

| Approach | How it works | Key limitation |
|---|---|---|
| Pooled test collection (TREC-style) | Run multiple retrievers; annotate the union of top-K results from all; unknowns outside pool treated as non-relevant | Pool gaps inflate metrics for systems that contributed more to the pool |
| Purchase/like as ground truth | Treat purchases or saves as strong relevance signals; compute Recall@K of new retriever against this set | Best proxy for marketplace; only works for high-intent signals |
| LLM relevance judging | Use a strong LLM (e.g. GPT-4o) to rate (query, item title+description) pairs | LLM bias toward textual match; use for cold-start item eval only |
| Human annotation | Sample (query, item) pairs; have 3+ raters grade relevance 0-4 | Gold standard; expensive; annotate 500+ queries for significance |
| Interleaving (online) | Interleave results from two systems; observe clicks on live traffic | Not offline; requires live traffic; ~5x more sensitive than A/B test |

---

## 3. Metrics

### 3.1 NDCG(K) — Full Derivation

**Step 1 — Cumulative Gain** (no position sensitivity):
```
CG(K) = Σ_{i=1}^{K}  rel_i
```

**Step 2 — Discounted CG** (logarithmic rank discount):
```
DCG(K) = Σ_{i=1}^{K}  rel_i / log2(i+1)

# Industry variant (Burges et al., 2005 — LambdaMART):
DCG(K) = Σ_{i=1}^{K}  (2^rel_i - 1) / log2(i+1)
# For binary rel ∈ {0,1}: 2^1-1=1, 2^0-1=0 → reduces to first form.
# For graded rel ∈ {0,1,2,3}: exponential reward for higher grades.
```

**Step 3 — Ideal DCG** (oracle ranking of same items):
```
IDCG(K) = DCG(K) computed on items sorted descending by rel
        = Σ_{i=1}^{min(K, |relevant|)}  rel_sorted_desc_i / log2(i+1)
```

**Step 4 — Normalize:**
```
NDCG(K) = DCG(K) / IDCG(K)  ∈ [0, 1]

# Edge case 1: IDCG=0 (no relevant items in pool) → skip query from mean
# Edge case 2: fewer than K items retrieved → pad with 0-relevance items
# Edge case 3: ties in rank → use deterministic tie-breaking (e.g. item_id)
```

### 3.2 Choosing K

| K | Surface | Rationale |
|---|---|---|
| 5 | Mobile search, above-the-fold | Median mobile user sees ≤5 items before scrolling. Primary metric for mobile-first marketplaces. |
| 10 | Desktop search (standard) | First-page proxy. Most IR literature uses K=10. Use as your headline metric. |
| 20 | Feed / infinite scroll | Users scroll further; captures more of the session intent. |
| 100 | Retrieval evaluation | Tests whether relevant items are in the candidate pool at all, not ranking quality. |

### 3.3 Full Metric Suite Implementation

```python
import numpy as np
from dataclasses import dataclass

@dataclass
class RankingMetrics:
    ndcg: float
    ndcg_5: float
    ndcg_20: float
    mrr: float
    map_score: float
    recall: float
    n_queries: int

def compute_ranking_metrics(
    qrels: dict[str, list[int]],   # query_id → ordered relevance labels (model's ranking order)
    k: int = 10,
) -> RankingMetrics:
    '''
    qrels: for each query, rel labels in the ORDER your model ranked them.
    Relevance can be binary (0/1) or graded (0-3).

    Skip queries with n_relevant=0 — do NOT count as 0, it biases the mean.
    '''
    ndcg_scores, ndcg5, ndcg20, mrr_scores, ap_scores, recall_scores = [], [], [], [], [], []
    discounts_k  = np.log2(np.arange(2, k + 2))
    discounts_5  = np.log2(np.arange(2, 7))
    discounts_20 = np.log2(np.arange(2, 22))

    for qid, rels in qrels.items():
        rels_arr = np.array(rels, dtype=float)
        n_rel = rels_arr.sum()
        if n_rel == 0:
            continue   # skip (do NOT count as 0 — biases mean)

        def _ndcg(rels_a, disc):
            r    = rels_a[:len(disc)]
            dcg  = ((2 ** r - 1) / disc[:len(r)]).sum()
            ideal = np.sort(rels_a)[::-1][:len(disc)]
            idcg = ((2 ** ideal - 1) / disc[:len(ideal)]).sum()
            return dcg / idcg if idcg > 0 else 0.0

        ndcg_scores.append(_ndcg(rels_arr, discounts_k))
        ndcg5.append(_ndcg(rels_arr, discounts_5))
        ndcg20.append(_ndcg(rels_arr, discounts_20))

        # MRR@K: reciprocal rank of FIRST relevant item in top-K
        # Use for: navigational / "find one good result" queries
        # Do NOT use when many relevant items exist per query (MRR ignores them)
        mrr = next((1.0/(i+1) for i, r in enumerate(rels_arr[:k]) if r > 0), 0.0)
        mrr_scores.append(mrr)

        # MAP@K: mean precision at each relevant item hit
        # Penalizes gaps between relevant items; better than MRR for multi-relevant queries
        # MAP ≠ NDCG: MAP treats all relevant items equally; NDCG uses graded relevance
        n_rel_seen, prec_sum = 0, 0.0
        for i, r in enumerate(rels_arr[:k]):
            if r > 0:
                n_rel_seen += 1
                prec_sum += n_rel_seen / (i + 1)
        ap_scores.append(prec_sum / n_rel)

        recall_scores.append(rels_arr[:k].clip(0, 1).sum() / max(n_rel, 1))

    def safe_mean(lst): return float(np.mean(lst)) if lst else float('nan')

    return RankingMetrics(
        ndcg=safe_mean(ndcg_scores), ndcg_5=safe_mean(ndcg5), ndcg_20=safe_mean(ndcg20),
        mrr=safe_mean(mrr_scores), map_score=safe_mean(ap_scores),
        recall=safe_mean(recall_scores), n_queries=len(ndcg_scores),
    )
```

### 3.4 When to Use Each Metric

| Metric | Best for | Avoid when | Common mistake |
|---|---|---|---|
| NDCG(K) | General ranking quality; graded relevance; multiple relevant items | Single navigational query | Using K=10 on mobile-first surfaces (should be K=5) |
| MRR(K) | Navigational search ("find listing #12345") | Multiple relevant items per query | Reporting MRR alongside NDCG without noting they measure different things |
| MAP(K) | Balanced precision+recall across the list | Graded relevance (MAP treats all hits equally) | Using MAP when n_relevant is highly variable across queries (biases mean) |
| Recall(K) | Retrieval evaluation (is the right item in the candidate pool?) | Ranking evaluation | Confusing Recall@100 (retrieval) with Recall@10 (ranking) |
| Hit Rate(K) | Binary: did any relevant item appear at all? | When ranking quality matters | Using as only metric — ignores rank order entirely |

### 3.5 Catalog Coverage and Popularity Bias Detection

Pure CTR/click metrics cause **popularity bias**: your eval set will be dominated by items that were already popular in the logged data, incentivizing models to re-rank popular items higher.

```python
import numpy as np
from collections import Counter

def gini_coefficient(item_freq: np.ndarray) -> float:
    '''
    Lorenz-curve Gini. 0 = uniform distribution, 1 = monopoly.
    High Gini in eval set → metrics dominated by popular items →
    small gains on head items look large; tail improvements are invisible.
    Reference: Brynjolfsson et al. (2003) on the long tail.
    '''
    s = np.sort(item_freq)
    n = len(s)
    idx = np.arange(1, n + 1)
    return float((2 * (idx * s).sum() - (n + 1) * s.sum()) / (n * s.sum()))


def catalog_coverage(recs: list[list[str]], catalog: set[str]) -> float:
    '''Fraction of catalog items appearing in >=1 recommendation list.'''
    shown = {item for lst in recs for item in lst}
    return len(shown & catalog) / max(len(catalog), 1)


def popularity_bias_report(eval_df, train_df) -> dict:
    '''
    Compare item popularity distribution in training vs. eval results.
    pop_bias_ratio > 1.5 suggests significant popularity bias.
    '''
    train_pop = Counter(train_df['item_id'])
    total_train = sum(train_pop.values())
    eval_df = eval_df.copy()
    eval_df['train_pop_pct'] = eval_df['item_id'].map(
        lambda x: train_pop.get(x, 0) / total_train
    )
    clicked   = eval_df[eval_df['relevance'] > 0]
    all_shown = eval_df
    return {
        'mean_pop_clicked':  float(clicked['train_pop_pct'].mean()),
        'mean_pop_shown':    float(all_shown['train_pop_pct'].mean()),
        'pop_bias_ratio':    float(
            clicked['train_pop_pct'].mean() /
            max(all_shown['train_pop_pct'].mean(), 1e-9)
        ),
        'gini_shown_items':  gini_coefficient(
            np.array(list(Counter(all_shown['item_id']).values()), dtype=float)
        ),
    }
```

> **Netflix/Airbnb on beyond-accuracy metrics:**
> - Netflix (Steck, RecSys 2018) frames diversity as a post-processing calibration constraint, not a training objective.
> - Airbnb (KDD 2018) found intra-list diversity (ILD) correlated weakly with booking lift.
>
> **Recommendation:** Track Gini + `catalog_coverage` as health metrics. Optimize diversity **only** if you have user research (NPS/surveys) showing it matters.

---

## 4. Counterfactual Evaluation

### 4.1 The Logged Policy Problem

> Your eval set was generated by the logging policy (heuristic ranker). The heuristic always puts item A at rank 1 and item B at rank 5. A is clicked more. A new ranker that promotes B gets **penalized** on this dataset even if B is genuinely better — because the eval set has no clicks on B-at-rank-1. This is called the **exposure bias problem** and is the reason offline NDCG lifts often underestimate online gains.

Formally, let `π_0` be the logging policy and `π_e` be the new policy. We observe rewards only under `π_0` but want to estimate the value of `π_e`. This is the **Off-Policy Evaluation (OPE)** problem.

Key references:
- Strehl et al. (2010 ICML)
- Joachims et al. (2017 WSDM)
- Swaminathan & Joachims (2015 ICML)

### 4.2 IPS for OPE — Formal Framing

```
Data D = {(x_i, a_i, r_i)} where:
  x_i = context (query + user features)
  a_i = action = ranked list at position level
  r_i = reward (click, purchase, etc.)
  π_0(a_i | x_i) = logging policy propensity

IPS policy value estimator:
  V_IPS(π_e) = (1/N) Σ_i  r_i * [π_e(a_i|x_i) / π_0(a_i|x_i)]

Problem: for a DETERMINISTIC logging policy (heuristic always gives same ranking),
π_0 ∈ {0, 1}, making IPS undefined when π_0(a_i) = 0 (new policy takes different action).

Practical fix for deterministic logging policy:
  → Treat POSITION as the action (not the full ranking)
  → π_0(pos=k | query) ≈ propensity[k]  (estimated from click data)
  → This is the standard "cascade model" approximation

When your logging policy is heuristic (deterministic), you CANNOT do full OPE.
Best in month 1: IPS-debiased NDCG using position-level propensities.
```

### 4.3 Doubly Robust (DR) Estimator

DR combines a direct model (DM) estimate with an IPS residual correction. It is consistent if **either** the DM is correctly specified **or** the propensity model is correct — not necessarily both. (Dudík et al., 2011 — "Doubly Robust Policy Evaluation and Learning.")

```python
import pandas as pd
import numpy as np
from lightgbm import LGBMClassifier

def fit_direct_model(train_df: pd.DataFrame, feature_cols: list[str]) -> LGBMClassifier:
    '''
    DM: fit P(click | context, item, position) on training data.
    LightGBM is sufficient for 1-year of marketplace data; no need for NN.
    '''
    X = train_df[feature_cols].fillna(0)
    y = train_df['relevance']
    model = LGBMClassifier(
        n_estimators=300, learning_rate=0.05,
        num_leaves=63, min_child_samples=50, subsample=0.8, verbose=-1,
    )
    model.fit(X, y)
    return model


def doubly_robust_ndcg(eval_df: pd.DataFrame, k: int = 10) -> float:
    '''
    DR estimator for NDCG@K.

    V_DR = (1/N) Σ_i [
        dm_pred_i                                   ← direct model baseline
        + (relevance_i - dm_pred_i) / propensity_i  ← IPS residual correction
    ]

    Unbiased if EITHER DM is correct OR propensity model is correct.

    When DR breaks: BOTH DM and propensity model are misspecified.
    Do NOT use in month 1 — you need a trained DM first.
    '''
    df = eval_df[eval_df['rank_position'] < k].copy()
    df['propensity'] = df['propensity'].clip(lower=0.05)   # prevent explosion
    df['dr_term'] = (
        df['dm_pred'] +
        (df['relevance'] - df['dm_pred']) / df['propensity']
    )

    def _dr_ndcg_per_query(group):
        g = group.sort_values('rank_position').head(k)
        discounts = np.log2(np.arange(2, len(g) + 2))
        dcg  = (g['dr_term'].values / discounts).sum()
        ideal = np.sort(g['relevance'].values)[::-1]
        idcg = (ideal / discounts[:len(ideal)]).sum()
        return dcg / idcg if idcg > 0 else float('nan')

    return float(df.groupby('query_id').apply(_dr_ndcg_per_query).dropna().mean())
```

### 4.4 Replay Evaluator

The Replay method (Li et al., 2011) keeps a logged event **only if** the new policy would have shown that item in its top-K. This gives an unbiased estimate but at the cost of very low data efficiency (typically 1–10% of logged events survive).

**Critical assumption:** The logging policy must be **stochastic** (randomized). A deterministic heuristic ranker makes replay non-unbiased. Add ε-greedy exploration to your logging policy first. Implement replay in month 2+.

---

## 5. Evaluation Harness Architecture

### 5.1 Core Abstractions

Three-layer design:
1. `Ranker` — a Protocol any model must implement
2. `EvalDataset` — an immutable, content-addressed snapshot
3. `EvalHarness` — orchestrates score → rank → metrics → persist

```python
# eval_harness/ranker.py
from typing import Protocol, runtime_checkable
import pandas as pd

@runtime_checkable
class Ranker(Protocol):
    '''
    Every model — heuristic, BM25, LightGBM, two-tower — implements this.
    The harness only knows about this interface.
    '''
    name: str       # human-readable, e.g. 'bm25_atlas_v3'
    version: str    # semver, e.g. '1.2.0'

    def score(
        self,
        queries: pd.DataFrame,     # cols: query_id, query_text, user_id, ...
        candidates: pd.DataFrame,  # cols: query_id, item_id, retrieval_score, ...
    ) -> pd.DataFrame:
        '''
        Return DataFrame with columns [query_id, item_id, score].
        score is a real-valued ranking signal; higher = more relevant.
        Must be deterministic given the same input (for reproducibility).
        '''
        ...


# eval_harness/dataset.py
import hashlib, json
from pathlib import Path
from dataclasses import dataclass, field
import pandas as pd

@dataclass
class EvalDataset:
    '''
    Immutable snapshot of an evaluation set.
    Versioned by a content hash. Never mutate — create a new one.
    '''
    name: str
    qrels: pd.DataFrame       # cols: query_id, item_id, relevance, rank_position
    queries: pd.DataFrame     # cols: query_id, query_text, user_id, surface
    candidates: pd.DataFrame  # cols: query_id, item_id, retrieval_score, ...
    metadata: dict = field(default_factory=dict)
    _fingerprint: str = field(init=False, repr=False)

    def __post_init__(self):
        h = hashlib.sha256()
        h.update(pd.util.hash_pandas_object(
            self.qrels.sort_values(['query_id', 'item_id'])
        ).values.tobytes())
        h.update(json.dumps(self.metadata, sort_keys=True).encode())
        self._fingerprint = h.hexdigest()[:16]

    def save(self, path: Path) -> None:
        path = Path(path)
        path.mkdir(parents=True, exist_ok=True)
        self.qrels.to_parquet(path / 'qrels.parquet', index=False)
        self.queries.to_parquet(path / 'queries.parquet', index=False)
        self.candidates.to_parquet(path / 'candidates.parquet', index=False)
        (path / 'metadata.json').write_text(
            json.dumps({**self.metadata, 'fingerprint': self._fingerprint,
                        'name': self.name}, indent=2)
        )
```

```python
# eval_harness/harness.py
import json, time, hashlib
from pathlib import Path
from dataclasses import dataclass, field, asdict
import pandas as pd, numpy as np

class EvalHarness:
    def __init__(self, results_dir: Path, k: int = 10):
        self.results_dir = Path(results_dir)
        self.results_dir.mkdir(parents=True, exist_ok=True)
        self.k = k

    def evaluate(self, ranker, dataset) -> dict:
        t0 = time.time()

        # 1. Score candidates
        scores_df = ranker.score(dataset.queries, dataset.candidates)
        assert set(scores_df.columns) >= {'query_id', 'item_id', 'score'}

        # 2. Rank: sort by score descending within each query
        scores_df = scores_df.sort_values(
            ['query_id', 'score'], ascending=[True, False]
        )
        scores_df['predicted_rank'] = scores_df.groupby('query_id').cumcount()

        # 3. Join with qrels
        eval_df = scores_df.merge(
            dataset.qrels[['query_id', 'item_id', 'relevance', 'rank_position']],
            on=['query_id', 'item_id'], how='left',
        )
        eval_df['relevance'] = eval_df['relevance'].fillna(0)

        # 4. Build qrels dict and compute metrics
        qrels_dict = {}
        for qid, group in eval_df.sort_values('predicted_rank').groupby('query_id'):
            qrels_dict[qid] = group['relevance'].tolist()

        metrics = compute_ranking_metrics(qrels_dict, k=self.k)
        run = {
            'ranker_name': ranker.name, 'ranker_version': ranker.version,
            'dataset_fingerprint': dataset._fingerprint, 'k': self.k,
            'metrics': asdict(metrics),
            'run_timestamp_utc': time.strftime('%Y-%m-%dT%H:%M:%SZ', time.gmtime()),
            'wall_time_sec': round(time.time() - t0, 2),
        }
        run_id = hashlib.sha256(
            f"{ranker.name}:{ranker.version}:{dataset._fingerprint}".encode()
        ).hexdigest()[:12]
        (self.results_dir / f'{run_id}.json').write_text(json.dumps(run, indent=2))
        return run
```

### 5.2 Statistical Significance: Bootstrap vs. Paired t-Test

> **Which test is correct for NDCG?** NDCG scores are **not** normally distributed (bounded in [0,1], often skewed, bimodal when many queries have NDCG=0 or NDCG=1). The paired t-test's normality assumption is violated. Use the **non-parametric paired bootstrap** instead. This is standard in IR evaluation (Sakai, 2006 — "Evaluating Evaluation Metrics Based on the Bootstrap").

```python
import numpy as np
from dataclasses import dataclass

@dataclass
class BootstrapCI:
    delta: float       # mean(metric_b) - mean(metric_a)
    ci_low: float      # 2.5th percentile of delta distribution
    ci_high: float     # 97.5th percentile
    p_value: float     # fraction of bootstrap samples where delta <= 0
    significant: bool  # CI does not cross zero at alpha=0.05

def bootstrap_metric_delta(
    scores_a: np.ndarray,   # per-query metric values for system A
    scores_b: np.ndarray,   # per-query metric values for system B
    n_bootstrap: int = 10_000,
    alpha: float = 0.05,
    seed: int = 42,
) -> BootstrapCI:
    '''
    Paired bootstrap CI for metric delta (B - A).

    Key: resample (query_a_i, query_b_i) JOINTLY — preserves pairing.
    Do NOT resample A and B independently (ignores per-query correlation,
    inflates variance).

    n_bootstrap=10,000 is standard in IR (Sakai 2006).
    Use 50,000 for publication-grade claims.

    When to use paired t-test: ONLY when >5000 queries AND normality verified.
    NDCG scores are NOT normal. The bootstrap is always safer.
    '''
    assert len(scores_a) == len(scores_b), 'Scores must be paired (same queries)'
    rng = np.random.default_rng(seed)
    n = len(scores_a)
    deltas = scores_b - scores_a
    observed_delta = deltas.mean()
    bootstrap_deltas = np.array([
        deltas[rng.integers(0, n, size=n)].mean()
        for _ in range(n_bootstrap)
    ])
    ci_low  = float(np.percentile(bootstrap_deltas, 100 * alpha / 2))
    ci_high = float(np.percentile(bootstrap_deltas, 100 * (1 - alpha / 2)))
    p_value = float((bootstrap_deltas <= 0).mean())   # one-tailed: B > A
    return BootstrapCI(
        delta=float(observed_delta), ci_low=ci_low, ci_high=ci_high,
        p_value=p_value, significant=(ci_low > 0),
    )

# Usage:
# ci = bootstrap_metric_delta(per_query_ndcg_a, per_query_ndcg_b)
# print(f'NDCG delta: {ci.delta:+.4f} [{ci.ci_low:+.4f}, {ci.ci_high:+.4f}] p={ci.p_value:.3f}')
```

### 5.3 Industry References

| Company | Paper / Blog | Key design insight |
|---|---|---|
| LinkedIn | "Towards a Better Tradeoff Between Relevance and Diversity in Search" (KDD 2021) | Separate eval harnesses for retrieval (Recall@K) and ranking (NDCG@K); results versioned in a metadata store alongside model artifacts |
| Airbnb | "Applying Deep Learning To Airbnb Search" (KDD 2018) | Offline NDCG used to triage experiments before A/B launch; calibrated against historical A/B results to set minimum detectable effect thresholds |
| Netflix | "Calibrated Recommendations" (RecSys 2018, Harald Steck) | Offline eval tracks calibration (genre distribution match) alongside NDCG; diversity treated as a constraint not an objective |
| Twitter/X | "Personalization in Search" (2021 blog) | Per-surface eval datasets (search vs. explore vs. who-to-follow); each surface has its own primary and secondary metric set |
| Criteo | "A Large Scale Benchmark for Uplift Modeling" (2021) | Full OPE pipeline with logged propensities; enables DR estimation at scale |

---

## 6. Offline vs. Online Metric Gap

> **The fundamental problem:** NDCG improvements of +2–5% offline routinely fail to replicate as CTR lifts in A/B tests. In some cases, a model with *lower* offline NDCG produces *better* online CTR. This is not random noise — it is systematic.

### 6.1 Known Causes

| Cause | Mechanism | Detection method |
|---|---|---|
| Position bias in training data | Model learns that rank-1 items are "good"; reproduces existing order; offline NDCG is high but not causal | Compare IPS-corrected NDCG vs. naive NDCG — large gap indicates position learning |
| Popularity bias in eval set | Eval set dominated by head items; improvement on tail is invisible offline but matters online | Stratify NDCG by query frequency quintile; track tail-query NDCG separately |
| Distribution shift at eval time | User behavior changed between train and eval period (seasonality, category trends) | Compute metric over rolling weekly windows; check for monotonic drift |
| Feedback loops | Items promoted online get more clicks → appear "better" in future logs → model over-ranks them | Compare item rank in model vs. item CTR conditional on the SAME position |
| Beyond-click engagement missing | Offline eval uses clicks; online metrics include dwell time, purchase, return rate | Add `dwell_time_ms` and purchase signal to graded relevance; recompute NDCG |
| Cold start items in online traffic | New items appear in online A/B that were not in offline eval set | Measure online CTR separately for items <7 days old vs. catalog items |
| Presentation effects | Image quality, price formatting, trust badges change CTR independent of ranking | Control for presentation in A/B test; report CTR normalized by item quality score |

### 6.2 Calibrating Offline Metrics to Predict Online Outcomes

You need a library of historical A/B experiments with both offline NDCG deltas and online CTR deltas. Fit a regression: `CTR_delta = f(NDCG_delta, surface, model_family)`. This requires at least 10–15 historical experiments.

```python
import pandas as pd, numpy as np
from sklearn.linear_model import LinearRegression

def calibrate_offline_metric(
    experiments: pd.DataFrame,   # cols: ndcg_delta, ctr_delta, surface, model_family
    new_ndcg_delta: float,
    surface: str,
    model_family: str,
) -> dict:
    '''
    Predict expected online CTR delta from offline NDCG delta.

    With <10 experiments: use Spearman correlation as a sanity check only.
    Do NOT report predicted CTR delta to stakeholders with <10 data points.

    With 10-30 experiments: linear regression on ndcg_delta is appropriate.
    With 30+ experiments: add interaction terms (surface * model_family).

    Key reference: Kohavi et al. (2013) "Online Controlled Experiments at
    Microsoft" — Section 4 on offline-online metric correlation.
    '''
    exp = experiments[
        (experiments['surface'] == surface) &
        (experiments['model_family'] == model_family)
    ]
    if len(exp) < 5:
        return {
            'warning': f'Only {len(exp)} experiments. Prediction unreliable.',
            'spearman_r': float(exp[['ndcg_delta','ctr_delta']].corr('spearman').iloc[0,1])
                if len(exp) >= 3 else None,
        }
    X = exp[['ndcg_delta']].values
    y = exp['ctr_delta'].values
    reg = LinearRegression().fit(X, y)
    predicted_ctr_delta = float(reg.predict([[new_ndcg_delta]])[0])
    residuals = y - reg.predict(X)
    return {
        'predicted_ctr_delta': predicted_ctr_delta,
        'uncertainty_1sigma':  float(np.std(residuals)),
        'r2_offline_online':   float(reg.score(X, y)),
        'n_experiments':       len(exp),
        'recommendation': 'Ship to A/B' if predicted_ctr_delta > 0.005 else 'Borderline',
    }
```

### 6.3 Surrogate Metric Dashboard for Leadership

| Metric tier | Metric | Owner | Update frequency |
|---|---|---|---|
| Primary (business) | GMV per query session | Product + Eng | Online A/B only |
| Primary (business) | Purchase rate per unique search session | Product + Eng | Online A/B only |
| Secondary (proxy) | CTR@K (online) | Eng | Online A/B + monitoring |
| Tertiary (offline surrogate) | NDCG@10 on held-out eval set | ML Eng | Every model version |
| Tertiary (offline surrogate) | IPS-corrected NDCG@10 | ML Eng | Every model version |
| Health (guardrail) | Recall@100 (retrieval coverage) | ML Eng | Every model version |
| Health (guardrail) | Gini coefficient of recommendations | ML Eng | Weekly monitoring |
| Health (guardrail) | p50/p95 query latency | Infra | Continuous |

---

## 7. Cold Start and Edge Cases

### 7.1 New Items (Eval Set Only)

> If an item first appears in the eval window, it has zero training signal. Your ranker may score it low (no click history → low collaborative signal), but it may be genuinely relevant. This causes you to underestimate models that generalize well to cold items.

| Scenario | Detection | Handling |
|---|---|---|
| Item in eval set, not in train | `item_id` not in train item vocab | Segment eval metrics by `item_age_days` quartile. Cold items (age <7d) get a separate `NDCG@10_cold` metric. **Do not average with warm-item NDCG.** |
| Item has no clicks in eval (zero-relevance) | `relevance=0` for all impressions | These are NOT cold-start items — they may be irrelevant. Keep in eval denominator. |
| Item has clicks only from one power user | All relevance from same `user_id` | Cap per-user relevance contribution (see §7.3) |
| Truly new item with zero impressions | Not in eval impression log at all | Cannot evaluate offline. Use LLM-based relevance judging on item metadata as proxy. |

```python
def segment_eval_by_item_age(
    eval_df: pd.DataFrame,
    items_meta: pd.DataFrame,    # cols: item_id, first_seen_ms
    eval_start_ms: int,
    age_thresholds_days: list[int] = [7, 30, 90],
) -> dict[str, float]:
    '''
    Separate NDCG@10 for cold vs. warm items.
    Report both; do NOT combine into a single number.
    '''
    eval_df = eval_df.merge(
        items_meta[['item_id', 'first_seen_ms']], on='item_id', how='left'
    )
    eval_df['item_age_at_eval_days'] = (
        (eval_start_ms - eval_df['first_seen_ms']) / 86_400_000
    ).clip(lower=0)

    results = {}
    thresholds = [0] + age_thresholds_days + [float('inf')]
    for lo, hi in zip(thresholds, thresholds[1:]):
        label = f'item_age_{int(lo)}_{int(hi) if hi != float("inf") else "inf"}d'
        subset = eval_df[
            (eval_df['item_age_at_eval_days'] >= lo) &
            (eval_df['item_age_at_eval_days'] < hi)
        ]
        results[label] = compute_ndcg_from_df(subset, k=10) if len(subset) >= 100 else None
    return results
```

### 7.2 Sparse Users (< 5 interactions)

| Approach | Trade-off |
|---|---|
| Separate eval cohort: sparse users | Lets you track cold-start user performance independently. Required for personalized ranking eval. |
| Fallback to non-personalized baseline for sparse users | Fair comparison: personalized vs. non-personalized on same users. Track `Lift@10 = NDCG_personalized / NDCG_nonpersonalized`. |
| Require minimum interactions for inclusion in personalized eval | Simple but creates survivor bias — you never measure cold-start quality. |
| Weight sparse user queries equally (no upweighting of active users) | Prevents dense-user NDCG from dominating. Each query contributes equally to the mean. |

### 7.3 Power User Dominance

Power users (top 1% by query volume) can contribute 30–50% of eval queries. This means your NDCG effectively measures quality for power users, not the median user.

```python
import numpy as np
import pandas as pd

def cap_user_query_contribution(
    eval_df: pd.DataFrame,
    max_queries_per_user: int = 20,
    seed: int = 42,
) -> pd.DataFrame:
    '''
    Cap per-user query contribution to prevent power user dominance.
    Strategy: sample max_queries_per_user queries uniformly at random per user.

    Alternative: weight each query by 1/user_query_count (changes the estimand —
    you would be estimating average per-user quality, not per-query quality).
    '''
    rng = np.random.default_rng(seed)

    def _sample_user_queries(group):
        unique_queries = group['query_id'].unique()
        if len(unique_queries) <= max_queries_per_user:
            return group
        sampled = rng.choice(unique_queries, size=max_queries_per_user, replace=False)
        return group[group['query_id'].isin(sampled)]

    result = eval_df.groupby('user_id', group_keys=False).apply(_sample_user_queries)
    print(f'Queries before cap: {eval_df["query_id"].nunique():,}')
    print(f'Queries after cap:  {result["query_id"].nunique():,}')
    return result


def user_activity_quartile_report(eval_df, train_df) -> pd.DataFrame:
    '''
    NDCG@10 broken down by user activity quartile.
    Q1 = cold start users. Q4 = power users.
    A good model improves uniformly across quartiles.
    A popularity-biased model improves Q4 but degrades Q1.
    '''
    user_activity = train_df.groupby('user_id')['query_id'].nunique().rename('n_queries')
    quartiles = pd.qcut(user_activity, q=4, labels=['Q1_cold', 'Q2', 'Q3', 'Q4_power'])
    eval_df = eval_df.merge(quartiles.reset_index(), on='user_id', how='left')
    results = []
    for q in ['Q1_cold', 'Q2', 'Q3', 'Q4_power']:
        subset = eval_df[eval_df['quartile'] == q]
        ndcg = compute_ndcg_from_df(subset, k=10) if len(subset) > 100 else None
        results.append({'quartile': q, 'ndcg_10': ndcg, 'n_queries': subset['query_id'].nunique()})
    return pd.DataFrame(results)
```

---

## 8. Practical Implementation Plan

### Week 1: Foundation

| Task | Output | Libraries |
|---|---|---|
| Audit impression + click logs for schema completeness | Schema gap report; list of missing required fields | `pandas`, `pydantic` for schema validation |
| Implement and validate impression → click join | Joined DataFrame; join rate ≥70% on direct FK, ≥85% total | `pandas` (merge), `pytest` for join unit tests |
| Temporal train/eval split | `train.parquet`, `eval.parquet` + `EvalDatasetMetadata` JSON | `pandas`, `pathlib`, `hashlib` |
| Baseline NDCG@10 for current heuristic ranker | Baseline number: "heuristic_v1 NDCG@10 = 0.XXXX" | `numpy` (fast NDCG), `pyarrow` (parquet I/O) |
| Position CTR audit by `rank_position` | CTR@rank plot; confirms position bias exists | `pandas`, `matplotlib` (one-off analysis) |

### Week 2: Debiasing + Full Metric Suite

| Task | Output | Libraries |
|---|---|---|
| Estimate position propensities (power-law fit) | `propensities.npy` + fit diagnostic plot | `scipy.optimize` (`curve_fit`), `numpy` |
| IPS-corrected and SNIPS-weighted NDCG(K) | IPS_NDCG vs. naive NDCG delta report | `numpy` (all vectorized) |
| Full metric suite: MRR, MAP, Recall(K), Hit Rate | `RankingMetrics` dataclass output per ranker | `numpy`, `dataclasses` |
| User segmentation: cold/warm/power user NDCG | NDCG by user activity quartile | `pandas` groupby + `numpy` |
| Item age segmentation: cold-item NDCG | NDCG by `item_age_days` quartile | `pandas` |
| Popularity bias report: Gini + coverage | `gini_coefficient`, `catalog_coverage` values | `numpy`, `collections.Counter` |

### Week 3: Harness CLI + Reproducibility

| Task | Output | Libraries |
|---|---|---|
| `EvalDataset` save/load with fingerprinting | `eval_dataset/` directory with parquet + `metadata.json` | `pandas`, `pyarrow`, `hashlib`, `json` |
| `EvalHarness.evaluate()` + `EvalRun` persistence | `eval_results/{run_id}.json` per experiment | `dataclasses`, `json`, `pathlib` |
| `Ranker` protocol + BM25 reference implementation | `BM25Ranker` using `pymongo` Atlas Search API | `pymongo`, `pandas` |
| Bootstrap CI for metric delta | `BootstrapCI` dataclass; use in CI/CD for regression detection | `numpy` (vectorized bootstrap loop) |
| CLI wrapper for reproducible eval runs | `python -m eval_harness.run --ranker ... --dataset ...` → JSON | `argparse`, `pathlib` |
| Regression test: new eval run must not regress NDCG by >0.005 | CI/CD check in GitHub Actions | `pytest`, `numpy` |

### What NOT to Build in the First 30 Days

> **Over-engineering traps — defer these:**

- **DR estimator:** Requires a calibrated direct model (DM) AND accurate propensity model. You will not have either in 30 days. Build plain IPS-NDCG first; add DR in month 2 when you have a trained LightGBM click model.

- **Replay evaluator:** Requires a stochastic logging policy (ε-greedy or Boltzmann exploration). Your heuristic ranker is deterministic. Replay is undefined. Add exploration to your logging policy first.

- **Neural ranker training:** You need the eval harness to validate model improvements *before* you train models. Eval first, model second.

- **Fancy diversity metrics (ILD, novelty, serendipity):** Track Gini as a health metric. Do not optimize for diversity until you have user research showing it matters.

- **Real-time eval pipeline:** Offline eval should run in batch on a snapshot. A real-time streaming eval adds infra complexity with no benefit in month 1.

- **Feature store / training pipeline:** The eval harness must be decoupled from model training. Keep them separate from day 1.

- **A/B experimentation platform:** Out of scope for 30 days. A/B comes after the eval harness is validated against at least 2 model versions.

### Python Library Choices

| Library | Use | Why this, not X |
|---|---|---|
| `pandas >= 2.0` | DataFrame joins, groupby, temporal splits | Native parquet I/O via pyarrow; copy-on-write semantics in 2.0 prevents silent mutation bugs |
| `pyarrow >= 14` | Parquet read/write for dataset snapshots | Zero-copy, columnar; 10–100× faster than CSV; cross-language compatibility |
| `numpy >= 1.26` | All metric computation (NDCG, bootstrap) | Vectorized loops; 100× faster than pure Python. Never loop over rows for NDCG. |
| `scipy >= 1.11` | Propensity curve fitting, stats tests | `curve_fit` for power-law propensity; `scipy.stats.spearmanr` for offline-online calibration |
| `lightgbm >= 4.0` | Direct model for DR estimator, click predictor | Fastest GBT for tabular click data; `learning_to_rank` objective built-in (LambdaRank) |
| `pydantic >= 2.0` | Event schema validation, metadata classes | Runtime validation catches log field changes immediately; 10× faster than v1 (Rust core) |
| `pytest >= 7` | Unit tests for join logic, NDCG correctness | Parametrize edge cases: empty queries, all-relevant, all-irrelevant, K > len(results) |
| `mlflow` or `DVC` | Eval run versioning and artifact tracking | `mlflow.log_metrics()` for `EvalRun`; DVC for dataset versioning. Do NOT use git for parquet files. |
| `rich` (CLI output) | Pretty-print `EvalRun` results in terminal | Optional but significantly improves DX for a harness run 10+ times/day |

---

## Key References

| Paper / Blog | Relevance |
|---|---|
| Joachims et al. (2017 WSDM) — "Unbiased Learning-to-Rank with Biased Feedback" | Position bias; examination hypothesis; IPS for ranking |
| Swaminathan & Joachims (2015 ICML) — "The Self-Normalized Estimator for Counterfactual Learning" | SNIPS estimator |
| Agarwal et al. (KDD 2019) — "Estimating Position Bias without Intrusive Interventions" | Propensity estimation without randomization |
| Dudík et al. (2011) — "Doubly Robust Policy Evaluation and Learning" | DR estimator |
| Li et al. (2011) — "Unbiased Offline Evaluation of Contextual-Bandit-Based News Article Recommendation" | Replay method |
| Sakai (2006) — "Evaluating Evaluation Metrics Based on the Bootstrap" | Bootstrap for IR metric significance |
| Kohavi et al. (2013) — "Online Controlled Experiments at Microsoft" | Offline-online metric correlation |
| Steck (RecSys 2018) — "Calibrated Recommendations" | Diversity as calibration constraint |
| Airbnb (KDD 2018) — "Applying Deep Learning To Airbnb Search" | Offline NDCG in production eval pipeline |
| LinkedIn (KDD 2021) — "Towards a Better Tradeoff Between Relevance and Diversity in Search" | Separate retrieval/ranking eval harnesses |
