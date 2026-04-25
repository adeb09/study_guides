# Recommendation & Ranking Systems – Staff ML Engineer Interview Prep Guide

> Modeling-first. No infra fluff. Dense, concrete, real trade-offs across retrieval, ranking, bias, and evaluation. Written for Senior / Staff ML engineers targeting FAANG-level recommendation system roles.

---

## Table of Contents

1. [Problem Formulation for Ranking Systems](#1-problem-formulation-for-ranking-systems)
2. [Core Model Families Used in FAANG-Level Systems](#2-core-model-families-used-in-faang-level-systems)
3. [Feature Engineering for Ranking Models](#3-feature-engineering-for-ranking-models)
4. [Biases in Recommendation Systems](#4-biases-in-recommendation-systems)
5. [Offline Evaluation of Ranking Models](#5-offline-evaluation-of-ranking-models)
6. [Training Strategies & Practical Considerations](#6-training-strategies--practical-considerations)
7. [How FAANG Systems Actually Combine Models](#7-how-faang-systems-actually-combine-models)
8. [Common Interview Questions – Senior Level](#8-common-interview-questions--senior-level)

---

## 1. Problem Formulation for Ranking Systems

### The Core Decision

How you frame the ranking problem determines your loss function, your training data format, your evaluation metric, and what biases you'll introduce. This decision is often made poorly at FAANG companies and corrected painfully later. Interviewers want to see you reason about this trade-off space explicitly.

---

### 1.1 Pointwise vs Pairwise vs Listwise

#### Pointwise

**Framing:** Treat each (query, item) pair independently. Predict a score or probability for each item. Rank by sorting those predictions.

**Loss functions:** MSE (regression on ratings), binary cross-entropy (click prediction), ordinal regression.

**Intuition:** The model never sees the list. It just predicts "how good is this item for this user?" independently. Ranking emerges implicitly from sorting predicted scores.

**Example:**
```
Input: (user_id=42, item_id=1337, context features)
Target: y=1 (clicked), y=0 (not clicked)
Loss: -[y·log(p) + (1-y)·log(1-p)]
```

**When it works well:**
- Abundant labeled data (clicks, ratings, purchases)
- You need calibrated probabilities (ads CTR prediction, where bid × CTR is the key product)
- Serving requires scoring each item independently (most FAANG-scale systems — you can't afford O(N²) comparisons)

**When it fails:**
- Ignores inter-item relationships. Two items with predicted scores 0.9 and 0.85 get the same relative ranking as 0.51 and 0.50, even though the confidence in the first pair is much higher.
- The loss does not penalize getting the *relative order* wrong — only getting the absolute score wrong.
- Position bias corrupts labels directly (items shown at position 1 get more clicks regardless of quality).

---

#### Pairwise

**Framing:** Given a query and two items (i, j), predict which item should rank higher. Train the model to satisfy: score(i) > score(j) for all (i, j) pairs where i is known to be better than j.

**Loss functions:**

- **RankNet loss (Bradley-Terry model):**

  $$
  P(i \succ j) = \sigma(s_i - s_j)
  $$

  $$
  \mathcal{L} = -\sum_{(i,j): i \succ j} \log \sigma(s_i - s_j)
  $$

- **Hinge loss (SVMRank):** $\max(0, 1 - (s_i - s_j))$

- **BPR (Bayesian Personalized Ranking):** Implicit-feedback-specific. Assumes observed interactions are better than unobserved ones. Maximizes posterior over user-item preferences:

  $$
  \mathcal{L}_{BPR} = -\sum_{u, i, j} \log \sigma(\hat{r}_{ui} - \hat{r}_{uj})
  $$

  where item $i$ was interacted with and item $j$ was not.

**When it works well:**
- You have strong preference signal (e.g., "user watched A but skipped B")
- You want to directly optimize relative ordering
- BPR is the go-to for implicit feedback matrix factorization

**When it fails:**
- Scales poorly: O(N²) pairs per query. Requires careful sampling.
- Doesn't account for the magnitude of relevance difference — treating (5-star vs 4-star) the same as (5-star vs 1-star).
- Doesn't account for position in the final list (optimizes local pairwise order, not the global list quality).

---

#### Listwise

**Framing:** The entire ranked list is the training unit. Directly optimize a list-quality metric.

**Loss functions:**

- **ListNet (SoftMax-based):** Defines a probability distribution over permutations and minimizes KL divergence between predicted and ground-truth distributions.

  $$
  P_s(\pi_i) = \frac{\exp(s_i)}{\sum_j \exp(s_j)}
  $$

- **ListMLE:** Maximizes likelihood of the ground-truth permutation given predicted scores.

- **LambdaRank / LambdaMART:** Does not define an explicit loss. Instead, computes *pseudo-gradients* that directly approximate the gradient of NDCG (or other non-differentiable metrics). This is the most important one in practice.

  $$
  \lambda_{ij} = \frac{\partial \mathcal{L}}{\partial s_i} \approx -\frac{|\Delta NDCG_{ij}|}{1 + e^{s_i - s_j}}
  $$

  The key insight: weight the pairwise gradient by how much swapping items i and j would change NDCG. Swapping items near the top of the list (where NDCG is more sensitive) gets a larger gradient.

**When it works well:**
- You want to directly optimize NDCG or MAP
- Search and ads ranking where list-level quality is the explicit product metric
- You have dense relevance judgments (human-labeled data)

**When it fails:**
- Requires the full list at training time — expensive
- NDCG gradients are dense and computationally heavier
- In implicit-feedback settings (most consumer rec systems), true relevance ordering is unknown

**Senior signal:** Most FAANG production systems use pointwise losses with carefully corrected labels (debiased) because serving requires independent per-item scoring. Pairwise/listwise are more common in search ranking (web search, ads) where query-level training data is structured.

---

### 1.2 Explicit vs Implicit Feedback

| | Explicit | Implicit |
|---|---|---|
| **Signal** | Star ratings, thumbs up/down, review scores | Clicks, watch time, dwell time, purchases, shares |
| **Availability** | Rare, expensive to collect | Abundant, cheap |
| **Noise** | Low — user stated preference directly | High — a click ≠ satisfaction |
| **Coverage** | Sparse (few users rate anything) | Dense |
| **Modeling challenge** | Missing-at-random assumption often wrong | Positive-unlabeled (PU) learning; need negative sampling |

**The fundamental problem with implicit feedback:** You observe only positive interactions. Non-interactions are ambiguous — did the user *not want* the item, or were they *never exposed* to it? This is the exposure bias problem (Section 4).

**Key modeling choices for implicit feedback:**
- BPR / pairwise losses treat unobserved as negatives
- Point-wise models use sampled negatives (random, popular items, or hard negatives)
- Some systems model "not clicked after exposure" differently from "never exposed"

---

### 1.3 Multi-Objective Ranking

Real FAANG systems never optimize a single objective. A typical ads ranking system optimizes simultaneously:
- **CTR** (engagement, short-term)
- **CVR** (conversion rate, revenue)
- **User satisfaction** (post-click dwell time, explicit ratings)
- **Advertiser fairness** (budget pacing, diversity)
- **Long-term retention** (avoiding clickbait)

**Common approaches:**

**Linear scalarization:**
$$
s = w_1 \cdot \hat{p}_{click} + w_2 \cdot \hat{p}_{purchase} \cdot \text{value} + \ldots
$$
Simple, interpretable, but weights require constant tuning and don't handle Pareto fronts.

**Multi-task learning (MTL):** Train a single model with shared lower layers and task-specific heads. Gradients from all objectives train shared representations. (Covered in depth in Section 6.)

**Constrained optimization:** Treat one objective as the primary metric, others as constraints. "Maximize engagement subject to advertiser fairness ≥ threshold." Lagrangian relaxation enables this in gradient-based training.

**Pareto / MOER (Multi-Objective Expected Reward):** More research-grade. Popular internally at some orgs for long-horizon optimization.

**Practical reality:** Most teams use a linear combination with hand-tuned weights validated via A/B testing. MTL gains favor as teams mature because it improves data efficiency — a user's purchase signal helps the click head too.

---

### 1.4 Exploration vs Exploitation (Modeling Perspective)

This is a tradeoff about *data collection* as much as ranking. A pure exploitation model will starve unshown items of signal, creating feedback loops (Section 4).

**Epsilon-greedy:** With probability ε, rank randomly. Simple but wastes exploration on obviously bad items.

**UCB (Upper Confidence Bound):** Score items by $\hat{\mu}_i + c\sqrt{\ln t / n_i}$. Prefer items where the confidence interval is wide. Natural for bandit-style retrieval.

**Thompson Sampling:** Maintain a posterior over each item's quality. Sample from posterior to rank. Well-calibrated exploration.

**Contextual bandits:** The most realistic framing for large-scale rec systems. Policy maps context → action (ranked list). LinUCB and neural contextual bandits bridge this with DNN rankers.

**Boltzmann / softmax exploration:** Score items probabilistically using softmax temperature. High temperature = exploration, low = exploitation.

**Senior signal:** At FAANG scale, full exploration is expensive. Most systems do *offline exploration* (randomization in data collection runs) or *diversification* at the re-ranking stage, not pure bandit serving. The key insight is that exploration serves *data quality* as much as *coverage*.

---

## 2. Core Model Families Used in FAANG-Level Systems

### The Ranking Stack Mental Model

Before diving into models, internalize this: FAANG systems never use a single model. The architecture is typically:

```
[Corpus: Billions of items]
        ↓
[Retrieval / ANN: ~100-1000 candidates]
        ↓
[Coarse Ranking: Fast, ~100-500 features]
        ↓
[Fine Ranking: Rich features, ~hundreds of features]
        ↓
[Re-ranking: Business rules, diversity, freshness]
```

Each stage has different model requirements. This section covers what goes in each stage.

---

### 2.1 Linear & Logistic Regression

**Intuition:** Score is a linear combination of features. For binary outcomes (click/no-click), pass through sigmoid.

$$
s = \mathbf{w}^T \mathbf{x}, \quad p = \sigma(\mathbf{w}^T \mathbf{x})
$$

**When it works well:**
- Calibration: LR naturally produces calibrated probabilities. Critical for ads systems (bid × CTR needs to be on the same scale).
- Interpretability: easy to debug and audit
- Fast inference: dot product is O(d). Serves at microsecond latency.
- Still used as the *calibration layer* on top of tree / DNN models at Google and Meta.

**When it fails:**
- Cannot learn feature interactions without manual engineering
- Saturates quickly as feature dimensionality grows
- Cannot model non-monotonic relationships

**Tradeoff at scale:** LR with hashed, crossed features (like FTRL-Proximal at Google) was the industry-standard ads model for years. It's not dead — it's the calibration head in many stacks. FTRL-Proximal optimizer is designed for sparse features: $\mathbf{w}_{t+1} = \arg\min_\mathbf{w} \sum_s g_s \cdot w + \frac{1}{2}\sum_s \frac{1}{\eta_s}(w - w_s)^2 + \lambda_1 |w| + \lambda_2 w^2$.

---

### 2.2 Factorization Machines (FM) and Matrix Factorization

#### Matrix Factorization (MF)

**Intuition:** Factorize the user-item interaction matrix $R \in \mathbb{R}^{|U| \times |I|}$ into user embeddings $P \in \mathbb{R}^{|U| \times k}$ and item embeddings $Q \in \mathbb{R}^{|I| \times k}$.

$$
\hat{r}_{ui} = \mathbf{p}_u^T \mathbf{q}_i
$$

**ALS (Alternating Least Squares):** Alternate between fixing Q and solving for P (closed-form least squares) and vice versa. Parallelizable. Used in Spark MLlib. Good for explicit ratings.

**BPR (Bayesian Personalized Ranking):** Pairwise loss on implicit feedback. Minimizes:
$$
\mathcal{L} = -\sum_{u, i \in \Omega_u, j \notin \Omega_u} \log \sigma(\mathbf{p}_u^T \mathbf{q}_i - \mathbf{p}_u^T \mathbf{q}_j) + \lambda \|\Theta\|^2
$$

**Limits of pure MF:**
- No cold-start: new users and items have no embeddings
- Cannot incorporate side information (context, item attributes)
- Linear interaction only — no higher-order features

#### Factorization Machines (FM)

**Intuition:** Generalize MF to arbitrary feature vectors. Every feature gets an embedding. Feature interaction is computed as dot product of embeddings.

$$
\hat{y} = w_0 + \sum_i w_i x_i + \sum_{i < j} \langle \mathbf{v}_i, \mathbf{v}_j \rangle x_i x_j
$$

The key trick: second-order term is O(kd) not O(d²) via the identity:
$$
\sum_{i<j} \langle \mathbf{v}_i, \mathbf{v}_j \rangle x_i x_j = \frac{1}{2}\left[\left\|\sum_i \mathbf{v}_i x_i\right\|^2 - \sum_i \|\mathbf{v}_i\|^2 x_i^2\right]
$$

**When FM works well:**
- Sparse, high-cardinality categorical features (user_id, item_id, category)
- Feature interactions matter but you don't have the data density for deep networks
- Good baseline for cold-start: item attributes and user demographics provide signal even without history

**Field-aware FM (FFM):** Each feature has a different embedding *per field* it interacts with. More expressive, more memory. Used heavily in CTR prediction at Yahoo and Criteo.

---

### 2.3 Gradient Boosted Decision Trees (GBDT): XGBoost, LightGBM

**Intuition:** Additive ensemble of shallow trees. Each tree corrects the residuals of the previous ensemble. Score is sum of leaf values across all trees.

$$
\hat{y} = \sum_{t=1}^{T} f_t(\mathbf{x}), \quad f_t \in \mathcal{F} \text{ (decision trees)}
$$

Objective: $\mathcal{L} = \sum_i \ell(y_i, \hat{y}_i) + \sum_t \Omega(f_t)$ where $\Omega$ regularizes tree complexity.

**Why GBDT dominates tabular ranking:**
- Handles mixed feature types natively (no normalization needed)
- Robust to outliers, missing values, non-monotonic relationships
- Naturally captures high-order feature interactions through tree splits
- Excellent out-of-box performance on structured data

**LightGBM vs XGBoost:**
- LightGBM: Leaf-wise tree growth (faster, lower memory), histogram binning, GOSS sampling — preferred for large datasets
- XGBoost: Level-wise growth (more stable), more tunable — preferred when you need precise regularization

**When GBDT fails:**
- Cannot generalize to unseen feature combinations (no generalization via shared embeddings)
- Expensive to update with fresh data — you need to retrain or do incremental boosting
- Doesn't handle sequence data, image/text features natively

**At FAANG scale:** GBDT is the workhorse of coarse and fine ranking stages when features are hand-engineered. Meta's news feed, LinkedIn's feed, and Airbnb's search ranking have all used GBDT or GBDT+DNN hybrids. LambdaMART (GBDT optimized with LambdaRank gradients) is the standard search ranking baseline.

---

### 2.4 Deep Neural Networks

#### Wide & Deep (Google, 2016)

**Intuition:** Combine a *wide* linear component (memorization via cross-product features) with a *deep* MLP component (generalization via dense embeddings). Addresses the memorization-generalization tension.

```
Wide Component: y_wide = w^T [x, φ(x)]   // raw + crossed features
Deep Component: y_deep = MLP(embed(x_sparse), x_dense)
Output: σ(y_wide + y_deep + bias)
```

**Key insight:** The wide part memorizes specific co-occurrences (e.g., "users who installed apps X and Y install Z"). The deep part generalizes to users who haven't been seen with those specific combos.

**When it works:** Apps with clear categorical feature importance + need for generalization. Google Play store recommendations. Large e-commerce.

**Limitation:** Wide component still requires manual feature engineering (choosing what to cross). Doesn't automatically learn all pairwise interactions.

---

#### Two-Tower / Dual Encoder

**Intuition:** Encode query (user) and item into the same embedding space using separate neural networks. Similarity score is the dot product (or cosine) of the two towers.

```
User Tower: u = f_θ(user_features)  →  ℝ^d
Item Tower: v = g_φ(item_features)  →  ℝ^d
Score: s = u^T v
```

**Training:**
- In-batch negatives (efficient): treat other items in the same batch as negatives
- Sampled softmax: sample k negatives per positive

**Why two-tower is the retrieval workhorse:**
- Item tower can be precomputed offline and indexed in an ANN index (FAISS, ScaNN)
- Serves at sub-millisecond latency for billion-scale retrieval
- Separating towers allows independent updates (update user tower without reindexing items)

**Critical failure modes:**
- Dot product is a *limited* interaction model — it can't model complex feature interactions between user and item
- Hard negative mining is required to get good results beyond random negatives
- In-batch negatives introduce *popularity bias* (frequent items appear more as negatives → model learns to demote popular items)

**Asymmetric two-tower:** User tower is more expensive (rich features, fresh context). Item tower is simpler (static attributes). Used in YouTube, Pinterest, Google Search.

---

#### DeepFM / xDeepFM

**DeepFM:** Replaces the wide component in Wide & Deep with FM. No manual feature crossing needed.

```
FM Component: captures pairwise interactions via embeddings
DNN Component: captures higher-order interactions
Shared embedding layer between both components
```

$$
\hat{y} = \sigma(y_{FM} + y_{DNN})
$$

**xDeepFM (Extreme Deep FM):** Introduces the **Compressed Interaction Network (CIN)**, which explicitly models feature interactions at the vector level (as opposed to bit-level in standard DNNs).

At each CIN layer, the interaction is:
$$
X_k^h = \sum_{i=1}^{H_{k-1}} \sum_{j=1}^{m} W_{ij}^{k,h} (X_{k-1}^i \odot X_0^j)
$$

This is an explicit, polynomial-style interaction that's more interpretable and controllable than a black-box DNN.

**When DeepFM/xDeepFM shine:** CTR prediction on sparse, high-cardinality categorical features. Criteo, Avazu ad benchmarks. Performs better than Wide & Deep when feature engineering resources are limited.

---

#### DLRM (Deep Learning Recommendation Model, Meta, 2019)

**Intuition:** Purpose-built for recommendation at Meta's scale. Key design: treats dense and sparse features asymmetrically.

```
Dense Features: MLP bottom → dense embedding
Sparse Features: direct embedding lookup → sparse embeddings
Interaction Layer: dot product of all pairs of embeddings + dense
Top MLP: final scoring
```

$$
\hat{y} = \text{MLP}_{top}(\text{concat}[\text{interactions}, \mathbf{x}_{dense}])
$$

where interactions = all pairwise dot products among embeddings.

**Key engineering insight:** Embedding tables for sparse features are the bottleneck — at Meta scale, they're TB-sized, sharded across machines. DLRM's architecture is co-designed with this constraint. The interaction is explicit (not a black box DNN), which makes it debuggable.

**When DLRM works well:** Very high-cardinality categorical features with hundreds of billions of training examples. Full pipeline from retrieval to fine ranking at social media scale.

**Limitations:** Pairwise dot-product interaction may miss complex, higher-order relationships. Subsequent work (DCN V2, MaskNet) addresses this.

---

### 2.5 Sequence-Based Models

#### RNN-Based Recommenders (GRU4Rec)

**Intuition:** Model the user's sequential interaction history as a time series. GRU captures temporal dynamics: recent items matter more, and the model can learn "reading → buying" patterns.

$$
\mathbf{h}_t = \text{GRU}(\mathbf{e}_{i_t}, \mathbf{h}_{t-1})
$$

Predict next item from $\mathbf{h}_t$.

**Limitations:** Sequential dependency means batching is harder. Long sequences cause vanishing gradient. Fixed context window. Transformers have largely superseded RNNs for sequence modeling.

---

#### Transformer-Based Models: SASRec, BERT4Rec

**SASRec (Self-Attentive Sequential Recommendation):**

Auto-regressive transformer for recommendation. Left-to-right attention (like GPT). At each position, attends to all previous interactions.

$$
\mathbf{S} = \text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d}}\right)V
$$

Each attention head learns a different type of user-item relationship (recency, category affinity, etc.). Position encodings capture temporal ordering.

**BERT4Rec:**

Uses bidirectional attention (like BERT). Randomly masks items in the sequence and trains the model to predict the masked items. Better at capturing long-range dependencies.

**SASRec vs BERT4Rec:**
- SASRec: better for next-item prediction (causal, auto-regressive — natural fit)
- BERT4Rec: better for fill-in-the-blank or session completion tasks
- SASRec is more commonly used in production (causal masking = easier serving)

**Challenges at FAANG scale:**
- O(L²) attention with long sequences — use linear attention or truncation
- User sequence can be thousands of interactions long — need careful truncation or hierarchical attention
- Serving: user sequence must be encoded fresh at inference, or cached with incremental updates
- Positional encoding choice matters: learned vs sinusoidal vs relative (ALiBi for very long sequences)

**Senior signal:** Know the difference between *session-level* sequences (short, dense, same session) vs *lifetime history* sequences (long, sparse, days/weeks). Transformers handle both but require different positional encoding schemes and context lengths.

---

### 2.6 Graph-Based Approaches

#### Node2Vec / DeepWalk (Shallow Graph Embeddings)

**Intuition:** Random walk on the user-item graph to generate "sentences." Train Word2Vec-style skip-gram on these walks. Items that appear together in walks get similar embeddings.

**DeepWalk:** Uniform random walks.

**Node2Vec:** Biased walks with parameters p (return probability) and q (exploration probability). Interpolates between BFS (community structure) and DFS (structural roles).

**When they work:** You have a rich interaction graph with clear structural patterns. Social networks, co-purchase graphs. Good for generating item embeddings when content features are weak.

**Limitations:** Transductive — can't generalize to unseen nodes without retraining. No feature propagation. Shallow model.

---

#### Graph Neural Networks (GNNs) for Recommendation

**Intuition:** Propagate information across the user-item graph. A user's embedding is influenced by their interacted items, and an item's embedding is influenced by users who interacted with it.

**LightGCN (most common in production):**

Removes nonlinear transformations from standard GCN. Pure graph convolution:

$$
\mathbf{e}_u^{(k+1)} = \sum_{i \in \mathcal{N}_u} \frac{1}{\sqrt{|\mathcal{N}_u||\mathcal{N}_i|}} \mathbf{e}_i^{(k)}
$$

Final embedding is the weighted sum across all layers:
$$
\mathbf{e}_u = \sum_{k=0}^{K} \alpha_k \mathbf{e}_u^{(k)}
$$

**Why remove nonlinearity?** Empirically, nonlinear activations in GCNs hurt for collaborative filtering because they add noise without benefit — the interaction graph already provides the inductive signal.

**PinSage (Pinterest):** Graphsage applied to Pinterest's image-item graph. Samples neighborhoods at inference time (unlike full-graph methods). Scales to billions of nodes. Uses importance-based random walks for neighborhood sampling.

**When GNNs win over MF:**
- High-order connectivity matters (user A → item X → user B patterns)
- You have rich side information on nodes (item text/image, user demographics)
- Cold-start: attribute-rich nodes can bootstrap embeddings even without interaction history

**GNN challenges at scale:**
- Neighborhood explosion: a 3-hop subgraph can involve millions of nodes
- Stale embeddings: the graph changes; re-running full GCN is expensive
- Training requires mini-batch sampling of subgraphs, not standard batches

---

### 2.7 Retrieval vs Ranking: The Critical Distinction

| | Retrieval (Candidate Generation) | Ranking (Scoring) |
|---|---|---|
| **Scale** | Billions → hundreds | Hundreds → tens |
| **Latency** | Must be fast (ANN) | Can be slower (MLP) |
| **Model** | Two-tower, MF, ANN | GBDT, Wide&Deep, DLRM |
| **Features** | Limited (must be pre-indexed) | Rich (computed at query time) |
| **Interaction** | Dot product (decomposable) | Arbitrary (non-decomposable) |
| **Training signal** | Recall-focused | Precision-focused |
| **Metric** | Recall@K | NDCG, AUC, Log loss |

**The non-decomposable problem:** A ranker can use features like `user_history ✕ item_freshness` that depend on both user and item together. A retrieval model cannot — it must decompose into user-side and item-side because items are pre-indexed before the query arrives.

**Senior signal:** "Why can't we just use one model?" Because retrieval requires ANN search over a static index. ANN requires that the score be decomposable as a dot product or L2 distance. Any model that uses cross-features between user and item at retrieval time breaks ANN indexability. This is the fundamental constraint driving the two-stage architecture.

---

## 3. Feature Engineering for Ranking Models

### 3.1 Feature Taxonomy

**User features:**
- **Static:** demographics, account age, location
- **Dynamic:** recent interactions, session activity, real-time context
- **Aggregated:** historical CTR, avg. session length, category affinities

**Item features:**
- **Static:** content attributes (genre, price, category), creation date
- **Engagement aggregates:** global CTR, like rate, share rate (beware: popularity bias source)
- **Freshness:** time since publication, trending score

**Context features:**
- Time of day, day of week, device type, app version, query (for search)
- These are underutilized but often high signal. User behavior on mobile at 11pm differs from desktop at 9am.

**Cross features (user × item):**
- User's historical CTR on this category
- Number of times user interacted with this author
- Semantic similarity between user's recent watches and this item's embedding

Cross features are where most of the signal lives — and where most junior engineers fall short. Don't just feed user features and item features independently.

---

### 3.2 Crossed Features and Feature Interactions

**Manual crossing (Wide & Deep, LR+FM era):**

```
feature_cross = hash(user_gender + item_genre)
```

Explicit, interpretable, but requires domain knowledge and misses long-tail combinations.

**Learned crossing (FM, DeepFM, DCN):**

FM: pairwise interactions via embedding dot products.

**DCN V2 (Deep & Cross Network V2):** Explicit cross layer:
$$
\mathbf{x}_{l+1} = \mathbf{x}_0 \mathbf{x}_l^T \mathbf{w}_l + \mathbf{b}_l + \mathbf{x}_l
$$
At each layer, the original input $\mathbf{x}_0$ is crossed with the current representation. Polynomial interaction of increasing degree across layers.

---

### 3.3 Temporal Dynamics and Recency

**The stale feature problem:** A user's historical average watch time is a fine feature. But *recent* behavior (last 30 min) is often more predictive than lifetime averages. Systems that don't model recency serve stale personalization.

**Techniques:**
- **Exponential decay weighting:** Weight recent interactions more: $w_t = e^{-\lambda (T - t)}$
- **Time-aware embeddings:** Concatenate time delta as a feature to sequence models
- **Separate recency buckets:** "Last 1 hour", "Last day", "Last week" as separate feature groups
- **Sequential models (SASRec, GRU):** Naturally handle recency through the sequence ordering

**Item freshness:** New content needs signal before it can be ranked well. This is the cold-start × freshness problem. Solutions:
- Exploration bonuses for new items (UCB-style)
- Fallback to content-based features for new items
- "Bootstrapping" embeddings from similar known items

---

### 3.4 Handling Sparse vs Dense Features

| | Sparse | Dense |
|---|---|---|
| **Examples** | user_id, item_id, category | watch time, price, embedding |
| **Cardinality** | Millions–billions | Continuous or low-dimensional |
| **Representation** | Embedding lookup | Direct or normalized |
| **Challenge** | Memory, cold-start | Normalization, outliers |

**Embedding table management at FAANG scale:**
- Embedding tables can be TB-sized. Partitioned across machines.
- Feature hashing: hash feature value to a fixed vocabulary size. Collisions are a real concern (consider feature-value prefixing to reduce collision rate).
- Subword hashing: for text features, hash n-grams instead of full strings (used for user query embeddings).

**Normalization for dense features:**
- Log transform for heavy-tailed distributions (watch counts, price)
- Quantile normalization for stable training
- Per-feature z-scoring — but watch for distribution shift at serving time

---

### 3.5 Embedding Strategies and Representation Learning

**Pre-trained embeddings:**
- Item2Vec: Train Word2Vec on sequences of user-item interactions. Items that appear in similar contexts get similar embeddings.
- BERT/LLM-derived embeddings for item text. Frozen or fine-tuned.
- Graph embeddings (Node2Vec, LightGCN) as input features to rankers.

**Multi-modal embeddings:**
- Combine text, image, audio embeddings for rich item representations
- Late fusion (concatenate then MLP) vs early fusion (cross-attention) — late fusion is cheaper, early fusion is more expressive

**Embedding alignment:**
- User and item embeddings must be in the same space for two-tower retrieval
- Use contrastive learning (InfoNCE loss) to align them

**Dimensionality choice:**
- Larger dimensions = more expressive but more memory, more prone to overfitting on sparse features
- Common choices: 32–512d for item embeddings; 64–256d for user embeddings
- Compressed embeddings: product quantization (PQ) for ANN index efficiency

---

## 4. Biases in Recommendation Systems

This is the section where staff-level candidates separate from senior candidates. Understanding bias isn't academic — it determines whether your offline metrics predict online performance, and whether your model actually optimizes what you think it does.

---

### 4.1 Position Bias

**Why it occurs:** Items shown at the top of a ranked list receive more clicks than lower-ranked items, *regardless of their actual quality*. Users have a declining probability of examining items as position increases.

**Examination model:** A click occurs only if:
1. The user *examines* the item (position-dependent)
2. The user finds it *relevant* (item-dependent)

$$
P(\text{click} | u, i, k) = P(\text{examine} | k) \cdot P(\text{relevant} | u, i)
$$

where k is position. This is the *examination hypothesis*.

**How it corrupts training:**
- Items at position 1 accumulate more clicks → model learns "this item is great" → model continues to rank it at position 1 → self-reinforcing loop.
- Items never shown at position 1 are starved of click signal → model underestimates their quality.

**Measuring position bias:**
- **Randomization experiments:** Randomly swap item positions and measure click rate differences. Attribute the difference to position, not item quality.
- **Propensity estimation:** Train a model to predict P(examine | k). Can be done with click-through rate analysis across positions.

**Mitigation:**

**Inverse Propensity Scoring (IPS):**
$$
\hat{\mathcal{L}}_{IPS} = \frac{1}{n}\sum_i \frac{\mathbb{1}[\text{click}_i]}{P(\text{examine} | k_i)} \cdot \ell(\hat{y}_i, y_i)
$$
Reweight the loss by the inverse of the probability of being examined. Items shown rarely (low positions) get upweighted.

**Position as a feature (PAL - Position-Aware Learning):**
- Include position as a training feature, but set position = 0 (or the top position) at inference time.
- Simple and effective. Widely deployed at Alibaba, Facebook.

**Propensity-weighted training:**
- Estimate propensity scores from randomized traffic or from position-click rate models.
- Apply propensity weights to the loss.

---

### 4.2 Selection Bias / Exposure Bias

**Why it occurs:** The model sees only items that were *shown* to users. Items never surfaced have no click data. The training distribution is therefore biased toward the current ranking policy.

**Formally:** If the current policy exposes item set $\mathcal{E}_u$ to user u, then:
$$
P(\text{observed interaction} | u, i) = P(\text{exposure} | u, i, \pi) \cdot P(\text{interaction} | u, i)
$$

Training on observed data without correcting for exposure conflates "not shown" with "not relevant."

**How it corrupts training:**
- Niche items are never shown → never get clicks → model never learns to show them → perpetual omission.
- The model's implicit universe is only the items the *previous* model ranked highly.

**Mitigation:**
- **Exploration traffic:** Inject random or UCB-driven item exposure to collect unbiased signal.
- **IPS:** Weight interactions by inverse of exposure probability.
- **Missing Not at Random (MNAR) modeling:** Explicitly model the exposure mechanism and account for it during training.
- **Unbiased matrix factorization:** Decompose the interaction matrix with explicit exposure probability correction.

---

### 4.3 Popularity Bias

**Why it occurs:** Popular items have more data, so the model learns to represent them better. Sparse collaborative filtering signals for long-tail items → their embeddings are poor → model ranks them lower → they get fewer interactions → cycle continues.

**Compounded with exposure bias:** Popular items are shown more → they get more clicks → model over-indexes on them → less popular (but potentially more relevant) items disappear.

**How it corrupts training:**
- In-batch negative sampling for two-tower training samples items proportional to popularity → popular items appear frequently as negatives → model learns to *suppress* them → hurts retrieval of popular-but-relevant items.
- Embedding quality is inversely correlated with item rarity.

**Mitigation:**
- **Popularity-corrected softmax:** In sampled softmax training, correct for sampling frequency:
  $$
  P(i|u) = \frac{\exp(s_{ui} / \tau - \log q_i)}{\sum_j \exp(s_{uj} / \tau - \log q_j)}
  $$
  where $q_i$ is item sampling probability (proportional to popularity). This is the standard correction in YouTube's two-tower training.

- **IPS debiasing:** Downweight popular items by inverse of exposure probability.
- **Regularization toward uniform distribution:** Add a diversity or entropy term to the loss.
- **Diversity constraints at re-ranking:** Limit the fraction of top-K items from dominant categories.

---

### 4.4 Feedback Loops and Self-Reinforcement

**Why it occurs:** The model's outputs become the training data for the next model. This creates a closed loop:

```
Policy π_t → Exposures → Interactions → Training data → Policy π_{t+1}
```

If π_t has any bias (which all real policies do), that bias compounds over training iterations.

**Concrete example (TikTok-style feed):** Model overweights "funny videos" → users click them → dataset becomes rich in funny video interactions → next model is even more bullish on funny videos → gradual collapse toward a single content type.

**Types of feedback loops:**
- **Recommendation → Consumption → Training:** Items recommended are the items learned from
- **Ranking → Exposure → Click → Reinforcement:** High-ranked items get more clicks regardless of quality
- **Creator feedback:** If certain content types are promoted, creators produce more of them → supply shifts toward what the model already favors

**Mitigation:**
- **Counterfactual training:** Train on data generated by diverse policies, not just the current policy.
- **Off-policy correction:** Use importance sampling to reweight data from old policies:
  $$
  w_i = \frac{\pi_{new}(a_i | s_i)}{\pi_{old}(a_i | s_i)}
  $$
- **Exploration diversity:** Intentionally serve a diverse mix to break the feedback loop.
- **Long-horizon metrics:** Track content diversity and user satisfaction over weeks/months, not just session CTR.

---

### 4.5 Survivorship Bias

**Why it occurs:** Models trained on successful items (those that got clicked, purchased, or went viral) learn only from outcomes of things that survived selection. Failed items—those shown briefly and ignored—leave little trace.

**Example:** In e-commerce, your training data is rich with interactions on products that converted. Products shown but not purchased are in the data but labeled negative. Products *never shown* don't exist in your dataset at all. The model can't learn about latent demand for unseen products.

**A subtler form:** You measure "quality" of a new feature via A/B test. The test runs for 2 weeks. But effects on retention take 3 months to manifest. You declare success based on surviving the 2-week window.

**Mitigation:**
- **Exposure-aware datasets:** Track all shown items, not just interacted-with items.
- **Holdout negatives:** Deliberately include items that were shown but not interacted with.
- **Causal inference framing:** Model the full potential outcomes, not just observed ones.

---

### 4.6 Presentation Bias

**Why it occurs:** How an item is presented (thumbnail quality, title text, rich snippet availability, display size) affects engagement independent of item quality.

**Example:** Two equally relevant videos. One has a high-quality thumbnail with faces; one has a blurry screenshot. The first gets 3× higher CTR. Model learns to favor "high-CTR thumbnails," which conflates presentation quality with content quality.

**How it corrupts training:**
- Model learns content associated with good presentation rather than content quality.
- Creators learn to optimize thumbnails over content → engagement becomes deceptive.

**Mitigation:**
- **Feature inclusion:** Include presentation features (thumbnail score, title length, rich snippet present) explicitly, so the model can separate presentation from content signal.
- **Controlled experiments:** Randomly vary presentation to estimate its causal effect.
- **Normalization by presentation opportunity:** If you can estimate the expected CTR for a given presentation context, normalize engagement by that baseline.

---

### 4.7 Counterfactual Learning & Doubly Robust Methods

These are the unifying framework for most bias correction above.

**IPS (Inverse Propensity Scoring):**
$$
\hat{\mathcal{L}}_{IPS}(\pi) = \frac{1}{n}\sum_i \frac{\pi(a_i|x_i)}{\pi_0(a_i|x_i)} \ell(y_i, \hat{y}(x_i, a_i))
$$
Unbiased if propensities are correctly estimated. High variance when propensities are small (some actions are very rare under the logging policy).

**SNIPS (Self-Normalized IPS):**
$$
\hat{\mathcal{L}}_{SNIPS} = \frac{\sum_i w_i \ell_i}{\sum_i w_i}
$$
Normalizing reduces variance at the cost of introducing slight bias. Usually worth it in practice.

**Doubly Robust (DR) Estimator:**
$$
\hat{\mathcal{L}}_{DR} = \frac{1}{n}\sum_i \hat{\ell}(x_i, a_i) + \frac{1}{n}\sum_i \frac{\pi(a_i|x_i)}{\pi_0(a_i|x_i)}\left(\ell_i - \hat{\ell}(x_i, a_i)\right)
$$

Uses a *direct model* $\hat{\ell}$ as a baseline and corrects it with IPS on the residuals. **Doubly robust** means it's consistent if *either* the direct model or the propensity model is correctly specified (not necessarily both). Lower variance than pure IPS, unbiased if either component is right.

**When to use what:**
- **Direct model:** Sufficient if your model captures confounders perfectly (rarely true)
- **IPS:** When you have good propensity estimates and don't trust your direct model
- **DR:** Default choice when you have both a model and propensity scores — best of both worlds
- **SNIPS:** When IPS weights are highly variable (rare actions, narrow policies)

---

## 5. Offline Evaluation of Ranking Models

The dirty secret of recommendation systems: offline metrics frequently don't predict online performance. Understanding why, and how to build better offline evaluation, is a core staff-level competency.

---

### 5.1 Core Metrics

#### Precision@K and Recall@K

$$
\text{Precision@K} = \frac{|\text{Relevant items in top K}|}{K}
$$
$$
\text{Recall@K} = \frac{|\text{Relevant items in top K}|}{|\text{Total relevant items}|}
$$

**When to use:**
- Precision@K: When you have a fixed-size list and care about quality of that specific slice (e.g., top 5 product recommendations on homepage)
- Recall@K: When missing relevant items is costly (e.g., safety-critical content filtering, information retrieval)

**Limitations:**
- Binary relevance — doesn't account for graded relevance
- Precision@K doesn't penalize a model for returning the 5 relevant items in positions 1–5 the same as returning them in positions K-4 to K
- Recall@K requires knowing the full set of relevant items — hard in implicit feedback settings

---

#### MAP (Mean Average Precision)

$$
\text{AP} = \frac{1}{|\text{Relevant}|} \sum_{k=1}^{K} \text{Precision@k} \cdot \mathbb{1}[\text{item}_k \text{ is relevant}]
$$
$$
\text{MAP} = \frac{1}{|Q|}\sum_{q} \text{AP}(q)
$$

**Why MAP is better than Precision@K:**
- Rewards models that rank relevant items earlier
- Averages across queries

**Limitation:** Treats all relevant items as equally relevant. No graded relevance.

---

#### NDCG (Normalized Discounted Cumulative Gain)

The most important ranking metric for production recommendation systems.

$$
\text{DCG@K} = \sum_{k=1}^{K} \frac{2^{r_k} - 1}{\log_2(k+1)}
$$
$$
\text{NDCG@K} = \frac{\text{DCG@K}}{\text{IDCG@K}}
$$

where IDCG is the DCG of the ideal ranking, and $r_k$ is the graded relevance of item at position k (e.g., 0–4 scale from human raters, or binary click/no-click).

**Key properties:**
- Handles graded relevance (unlike MAP)
- Position-sensitive: relevant items at position 1 contribute more than at position 10 (logarithmic discount)
- Normalized: comparable across queries with different numbers of relevant items

**Interpretation:** NDCG@10 = 0.82 means your ranked list achieves 82% of the quality of the ideal ranking (if the best 10 items were ranked perfectly).

**Limitation:** Requires relevance judgments. In implicit feedback settings, clicks ≠ relevance. Highly clicked items at the top inflate NDCG even if they were only clicked due to position bias.

---

#### AUC (Area Under the ROC Curve)

$$
\text{AUC} = P(\hat{s}_{i^+} > \hat{s}_{i^-})
$$

Probability that a randomly chosen positive item is scored higher than a randomly chosen negative item.

**When to use:** Binary prediction tasks (click/no-click, conversion/no-conversion). Measures ranking ability rather than calibration.

**Key limitation:** AUC is a global metric — a model can have high AUC but be poor at ranking items *within a user's session*. Use per-query AUC (gauc) for session-level evaluation.

**GAUC (Group AUC):** Average AUC computed per user:
$$
\text{GAUC} = \frac{\sum_u |\mathcal{D}_u| \cdot \text{AUC}_u}{\sum_u |\mathcal{D}_u|}
$$
This is the metric that actually matters for ranking — it measures personalization quality, not just global discrimination.

---

#### Log Loss (Binary Cross-Entropy)

$$
\mathcal{L} = -\frac{1}{N}\sum_i [y_i \log \hat{p}_i + (1-y_i)\log(1-\hat{p}_i)]
$$

**When to use:** When calibrated probabilities matter (ads CTR → bid × CTR is a revenue formula). A model with lower log loss has better-calibrated probability estimates.

**Critical distinction:** **Ranking vs calibration:**
- A model can rank items perfectly (high AUC/NDCG) but have poorly calibrated probabilities (high log loss)
- A model can have well-calibrated probabilities (low log loss) but poor ranking (low AUC)
- For ads: you need both. For pure recommendation: ranking quality dominates.

---

### 5.2 Calibration

**What calibration means:** A model is calibrated if, among all items it gives probability p, roughly fraction p of them are actually clicked.

**Platt scaling:** Post-hoc calibration. Fit a logistic regression on model scores to recalibrate probabilities.

**Isotonic regression:** More flexible post-hoc calibration. Fits a non-decreasing step function.

**Why calibration matters separately from ranking:**
- Ads: CTR × bid determines revenue. Miscalibrated CTR predictions → incorrect bid outcomes → revenue loss.
- Multi-objective scalarization: If engagement probability and conversion probability are on different scales, the linear combination is meaningless without calibration.

**Calibration plots:** Plot predicted probability buckets vs observed click rate. Well-calibrated model lies on the diagonal.

---

### 5.3 Offline vs Online Metric Mismatch

This is one of the most important problems in applied ML and frequently comes up in staff interviews.

**Why offline metrics fail to predict online metrics:**

1. **Dataset construction bias:** Offline test sets are built from historically shown items. Items that were never shown are absent — so offline metrics measure "can we re-rank items that were already shown well?" not "can we surface better items?"

2. **Label noise:** Clicks ≠ satisfaction. High offline NDCG with click labels can correspond to a clickbait-heavy model that degrades online satisfaction.

3. **Position bias in test labels:** If test data carries position bias (items at position 1 have inflated click labels), the model that learns to predict those inflated labels does well offline but just re-creates the biased ranking online.

4. **Feedback loops not modeled:** The online system's behavior changes user preferences over time. Offline data captures a snapshot of behavior under the previous policy.

5. **Coverage gap:** Offline test contains only items present in the historical log. A new model might surface never-before-seen items that perform well online but can't be evaluated offline.

**What actually predicts online performance:**
- Debiased offline metrics (IPS-corrected NDCG or GAUC)
- Counterfactual evaluation with randomization data
- Holdout temporal evaluation (predict next-day/next-week behavior)
- Interleaving experiments (faster proxy for A/B tests)

---

### 5.4 Advanced Offline Evaluation

#### Counterfactual Evaluation (Off-Policy Evaluation, OPE)

Evaluate policy $\pi_{new}$ using data collected under policy $\pi_{old}$.

**IPS estimator:**
$$
\hat{V}^{IPS}(\pi) = \frac{1}{n}\sum_i \frac{\pi(a_i|x_i)}{\pi_0(a_i|x_i)} r_i
$$

**Doubly robust estimator:** (See Section 4.7 for formulation.)

**Practical challenge:** When $\pi_{new}$ deviates significantly from $\pi_{old}$, importance weights become extreme. Clip weights at a threshold M:
$$
w_i = \min\left(\frac{\pi(a_i|x_i)}{\pi_0(a_i|x_i)}, M\right)
$$
Clipping introduces bias but dramatically reduces variance.

---

#### Negative Sampling Strategies and Their Biases

How you construct negatives for training and evaluation matters enormously.

**Random negatives:** Sample uniformly from the corpus. Easy, but ignores hard cases. Leads to over-optimistic offline metrics (the model doesn't see the hard negatives it will face online).

**Popularity-based negatives (in-batch):** Sample items proportional to their frequency. Introduces popularity bias — items that are globally popular get over-sampled as negatives, causing the model to demote them. Must apply popularity correction (see Section 4.3).

**Hard negatives:** Items that are semantically similar to positives but weren't interacted with. Dramatically improves model discriminability. Risk: if "hard negative" is actually a *relevant* item the user just wasn't shown, you're training the model to demote relevant items.

**Mixed negatives:** Combination of random and hard negatives. Common in production (e.g., Google's REALM, DPR).

**Uniform random negatives for evaluation:** Standard choice. **Bias:** If the corpus is large and positives are sparse, random evaluation dramatically underestimates model quality vs a real-world re-ranking scenario. Popularity-weighted sampling is more realistic.

---

#### Temporal Validation vs Random Splits

**Random splits:** Split data randomly into train/val/test. Problem: you're predicting *past behavior from the future*. Data leakage — features derived from future interactions bleed into training labels.

**Temporal splits:** Train on data before time T, validate/test on data after time T. No temporal leakage. Realistic — mirrors deployment where the model was trained on past data and evaluated on future behavior.

**Why temporal splits give lower metrics:** Distribution shift between training and evaluation periods. Items, users, and trends change. This is actually a *more honest* estimate of online performance.

**Sliding window evaluation:** Evaluate at multiple time horizons (T+1 day, T+7 days, T+30 days). Reveals how quickly model performance degrades — key input to training frequency decisions.

**Senior signal:** Always use temporal splits for recommendation evaluation. Any paper or internal evaluation using random splits on interaction data has inflated numbers. Know this and flag it.

---

## 6. Training Strategies & Practical Considerations

### 6.1 Negative Sampling Techniques

**(See Section 5.4 for evaluation-side negative sampling. This section covers training-side.)**

**Random negatives (uniform):** Simple baseline. Works when the corpus is small relative to the user's taste space. Poor at teaching the model to distinguish subtly similar items.

**In-batch negatives:** Items from other examples in the same batch are treated as negatives. Efficient — no extra forward passes. Standard for two-tower training. Introduces popularity bias: popular items appear more often in batches and are thus over-represented as negatives.

**Popularity-corrected in-batch:**
$$
\text{adjusted logit}_{ij} = \text{logit}_{ij} - \log q_j
$$
where $q_j$ is item j's sampling probability. This is the correction YouTube uses.

**Hard negatives:** Mine items that score highly under the current model but were not interacted with. Two sources:
1. **ANN-mined hard negatives:** Query the retrieval index with the user vector, retrieve top-K items, remove positives — the rest are hard negatives.
2. **BM25/text-hard negatives (for search):** Items that are textually similar but semantically different.

**Staged hard negative curriculum:** Start with easy negatives, gradually introduce harder ones. Prevents model collapse early in training when the model can't yet distinguish hard negatives from true positives.

**Senior signal:** Hard negatives improve recall of the retrieval model but can hurt precision if not carefully managed. The optimal mix of easy/hard negatives is a hyperparameter that requires offline-to-online validation.

---

### 6.2 Handling Class Imbalance

**The reality:** In recommendation systems, click rates are typically 0.5–5%. Conversion rates 0.1–2%. The negative class dominates.

**Downsampling negatives:**
- Train on a balanced or near-balanced subsample
- **Critical:** You must correct calibration at inference time. If you downsample negatives by factor r, your model outputs $\hat{p}_{biased}$. True probability:
  $$
  \hat{p}_{true} = \frac{\hat{p}_{biased}}{\hat{p}_{biased} + (1 - \hat{p}_{biased}) / r}
  $$
  This correction is essential for ads systems. Many teams skip it and wonder why their CTR predictions are wrong.

**Focal loss:** Down-weights easy negatives during training:
$$
\mathcal{L}_{focal} = -\alpha_t (1 - p_t)^\gamma \log(p_t)
$$
Focuses training on hard, misclassified examples. Originally from object detection but applies directly to imbalanced ranking.

**Cost-sensitive learning:** Assign different per-class weights in the loss. Simple and effective.

---

### 6.3 Regularization Strategies

**L2 regularization (weight decay):** Standard. Critical for embedding tables — without regularization, embeddings for rare items overfit to training noise.

**Dropout:** Applied to DNN layers. Not typically applied to embedding lookups (different semantics). Standard values: 0.1–0.5.

**Embedding regularization strategies:**
- L2 on embedding norms: prevent embeddings from growing unbounded
- Frequency-scaled regularization: apply stronger L2 to low-frequency items (they have fewer gradient updates, so regularization toward zero is appropriate)

**Gradient clipping:** Prevent exploding gradients, especially in sequence models. Clip gradient norm to a threshold (1.0–5.0).

**Early stopping:** Monitor validation loss and stop when it plateaus. Most important form of regularization in deep recommendation models.

---

### 6.4 Cold Start Problem (Modeling Approaches)

**User cold start:** New user with no interaction history.

- **Content-based fallback:** Use only demographic and context features (device type, location, time of day) until interaction history accumulates.
- **Onboarding prompts:** Actively solicit preferences to bootstrap the user model.
- **Meta-learning (MAML-style):** Train the model to be quickly adaptable to new users from few interactions. Expensive to train but powerful.
- **Transfer learning:** Pre-train on large user population, fine-tune rapidly on new user's few interactions.

**Item cold start:** New item with no interaction history.

- **Content embedding:** Use item's text, image, category as a surrogate for interaction-derived embedding.
- **Exploration bonus:** Rank new items higher than their predicted score to collect signal (UCB-style).
- **Warm-start via similar items:** Initialize a new item's embedding as the average of similar known items.
- **Graph propagation:** If the new item is connected to known items in a taxonomy or knowledge graph, propagate embeddings through the graph.

**Senior signal:** Cold start is not a single problem — user cold start, item cold start, and new context (new category, new feature) require different solutions. Know all three.

---

### 6.5 Multi-Task Learning (MTL) in Ranking Models

**Why MTL:** A single model trained jointly on CTR, CVR, and watch time shares a representation. The combined gradient signal from multiple tasks produces a better-generalized embedding than training each task separately with less data.

**Standard MTL architecture:**

```
Shared Bottom: shared embedding + MLP layers
        ↓
Task Towers: separate MLP per task (CTR head, CVR head, rating head)
```

**ESMM (Entire Space Multi-task Model, Alibaba):** Addresses the CVR prediction problem. CVR is only observed for *clicked* items, creating selection bias. ESMM models:
$$
P(\text{conversion}) = P(\text{click}) \cdot P(\text{conversion} | \text{click})
$$
Both are trained jointly. The click model sees unbiased data (all impressions); the CVR model is trained over the click subspace, but gradients flow back through the product, allowing it to learn on the entire impression space.

**MMoE (Multi-gate Mixture of Experts, Google):** Uses multiple expert networks with soft gating. Different tasks activate different combinations of experts:
$$
y_k = h^k(f^k(\mathbf{x})), \quad f^k(\mathbf{x}) = \sum_i g^k(\mathbf{x})_i \cdot e_i(\mathbf{x})
$$
Each task has its own gating network $g^k$ that determines which experts to use.

**PLE (Progressive Layered Extraction, Tencent):** Extends MMoE with task-specific experts + shared experts at each layer. Addresses "negative transfer" — when one task's gradients hurt another task.

**Negative transfer:** The central risk in MTL. Occurs when tasks have conflicting gradients. Mitigation:
- Gradient surgery: project out conflicting gradient components
- Task weighting: uncertainty weighting (Kendall et al.) to balance task gradients
- Architecture choices: PLE, task-specific experts

---

## 7. How FAANG Systems Actually Combine Models

### 7.1 Multi-Stage Ranking Architecture

This is the most important systems-level knowledge for a staff-level modeling interview. You need to know not just that stages exist but *why*, and what model tradeoffs drive each stage.

```
Corpus (10B+ items)
       ↓
[Stage 1: Retrieval]
  Multiple retrieval sources:
  - Collaborative filtering (Two-tower, ANN)
  - Content-based (text/image embedding ANN)
  - Graph-based (GNN embeddings)
  - Query-item (search, BM25 + dense)
  → ~1000 candidates
       ↓
[Stage 2: Coarse Ranking]
  Lightweight model (GBDT or shallow DNN)
  ~100-200 features
  Must process 1000 candidates in <50ms
  → ~200 candidates
       ↓
[Stage 3: Fine Ranking]
  Rich model (DLRM, DeepFM, Wide&Deep)
  Hundreds of features
  Cross-features, user history, context
  → Top 100 items
       ↓
[Stage 4: Re-ranking]
  Business logic, diversity, freshness, safety
  Sequential scoring (e.g., MMR, DPP)
  → Final displayed list
```

**Why multiple stages?**

The cost-quality tradeoff. A fine-ranker with 500 features applied to 1 billion items would require tera-FLOPS per query. By restricting the fine ranker to 200 candidates, you apply your most expensive model only where it matters, while using cheap retrieval to ensure you don't miss great candidates.

**Cascade error:** If retrieval misses the best item, fine ranking can't recover it. The recall of the retrieval stage is a hard ceiling on the full pipeline's quality. This is why retrieval evaluation focuses on Recall@K rather than NDCG.

---

### 7.2 Candidate Generation

**Multiple retrieval sources (important):** No single retrieval model has high recall across all intents. Production systems blend:

- **Collaborative filtering (two-tower):** Best for "users like you watched X" type retrieval
- **Content/semantic (dense retrieval):** Best for semantic matches (search, cold items)
- **Recency signals:** New/trending items that CF hasn't accumulated signal on yet
- **Pinned / business logic sources:** Sponsored content, contractual items

Sources are merged with deduplication, then fed to coarse ranking. The mixing ratio between sources is a key product/engineering decision.

**ANN algorithms:**
- **HNSW:** Hierarchical Navigable Small World graph. Best recall-latency tradeoff for in-memory indexes.
- **FAISS (Facebook):** IVF (inverted file index) + PQ compression for large-scale. Used for billion-scale item indexes.
- **ScaNN (Google):** Anisotropic quantization, optimized for Google's hardware. Outperforms FAISS in many benchmarks.

---

### 7.3 Ensembling Strategies

**GBDT + DNN (the Meta/LinkedIn approach):**
1. Train DNN to produce embedding or intermediate representations
2. Feed DNN outputs (embeddings, intermediate activations) as features to GBDT
3. GBDT exploits the learned representations + raw features + interactions

Why this works: GBDT is excellent at feature interaction discovery on tabular data; DNN is excellent at representation learning from raw features. Combined: best of both.

**Stacking / blending:**
- Train base models on the same data (or cross-validated folds)
- Train a meta-model on predictions of base models
- Risk: data leakage if not careful with fold setup

**Score averaging / rank averaging:**
- Simplest ensemble. Average predicted scores from multiple models.
- Rank averaging (average rank rather than score) is more robust to calibration differences between models.

---

### 7.4 Knowledge Distillation and Model Compression

**Why distillation in recommendation:**
- Fine rankers are expensive. You want a cheaper model that approximates the rich model.
- Teacher: Large model with rich features (hundreds of features, slow)
- Student: Smaller model with subset of features (fast enough for serving)

**Knowledge distillation loss:**
$$
\mathcal{L} = \alpha \mathcal{L}_{hard} + (1-\alpha)\mathcal{L}_{soft}
$$
$$
\mathcal{L}_{soft} = \text{KL}\left(\text{softmax}\left(\frac{z_{teacher}}{T}\right) \| \text{softmax}\left(\frac{z_{student}}{T}\right)\right)
$$

Temperature T > 1 produces softer, more informative probability distributions from the teacher.

**List-wise distillation:** Rather than distilling on individual item scores, distill on the *ranking distribution* produced by the teacher. The student learns to reproduce the teacher's relative ordering, not just its absolute scores.

**Ranking distillation (RankDistil):** Student learns to match the teacher's top-K ranking directly, with heavier weight on higher-ranked items.

**Embedding compression:** Product quantization (PQ) reduces embedding dimensions from e.g., 256 float32 → 8 bytes via 8×8-bit codebooks. 32× compression with ~2–5% recall drop for ANN.

---

## 8. Common Interview Questions – Senior Level

These questions are frequently asked at Google, Meta, Netflix, Amazon, and similar companies for Senior/Staff MLE roles focused on recommendation systems. I've included the key angles interviewers explore.

---

### Q1: How would you design a ranking model for a new video recommendation feed starting from scratch?

**What they're testing:** Can you structure a multi-stage system, identify the right modeling choices at each stage, and think about cold-start?

**Strong answer components:**
- Two-tower retrieval → GBDT coarse rank → deep fine rank → re-ranking
- Identify that you need a cold-start strategy before you have data
- Start with pointwise classification (CTR prediction) before introducing multi-objective complexity
- Choose training data with temporal splits, not random
- Metric choice: NDCG for ranking quality, log loss for CTR calibration, GAUC for personalization

**Follow-up:** "What if we have only 1 week of data?" → Content-based fallback, pre-trained text/image embeddings for cold-start, limit model complexity.

---

### Q2: Your offline NDCG went up after a model change but online engagement went down. Why?

**What they're testing:** Understanding of offline-online metric gap. This is a critical judgment question.

**Strong answer components:**
- Position bias: test data labels are biased by previous model's ranking. New model scores differently from previous model → labels don't match.
- Label quality: if NDCG is computed on click labels, it may be optimizing for clicks not satisfaction. The new model may produce clickbait that degrades long-term engagement.
- Distribution shift: new model may optimize for historical users, but online population has shifted.
- The new model may be a better *re-ranker* but a worse *full pipeline* model — the interaction with retrieval changed.

**Follow-up:** "How would you detect this?" → Compare IPS-corrected NDCG with raw NDCG. Run interleaving. Track diversity metrics alongside NDCG.

---

### Q3: Walk me through how you'd handle position bias in a news feed ranking model.

**What they're testing:** Depth of debiasing knowledge.

**Strong answer:**
1. Recognize the problem: items shown at top positions accumulate more clicks regardless of quality.
2. Use PAL (position as training feature, zeroed at inference) as the simplest fix.
3. If more rigorous: estimate propensity scores P(examine | position) using randomized traffic or position-click rate curves.
4. Apply IPS to reweight the training loss.
5. Evaluate offline using IPS-corrected NDCG to verify the debiasing works.

**Follow-up:** "What if you don't have randomized traffic?" → Estimate propensity from click-through rates across positions, with assumptions about examination probability decay.

---

### Q4: When would you use a two-tower model vs a GBDT for ranking?

**What they're testing:** Knowing which model is appropriate for which stage.

**Strong answer:**
- Two-tower for retrieval: scalable to billions of items via ANN, but limited to dot product interaction
- GBDT for ranking: rich feature interactions on the candidate set, interpretable, fast to train
- Two-tower can't use cross-features (non-decomposable) — this is the core reason for the stage separation
- Combine them: use DNN embeddings as GBDT features (Section 7.3 ensembling)

**Follow-up:** "What if latency allows for more complex models at retrieval?" → You can use approximate cross-attention with learned quantization, or a distilled version of a cross-encoder.

---

### Q5: How would you handle the cold start problem for items in a two-tower model?

**What they're testing:** Concrete solutions to cold start.

**Strong answer:**
- Initialize new item embeddings from content: text encoder (BERT), image encoder (ViT/ResNet) → project to item embedding space
- Warm-start via similar items: find K nearest neighbors in content embedding space, average their collaborative embeddings
- Exploration bonus: UCB-style upranking for new items until they accumulate interaction signal
- Hybrid tower: item tower takes both content features (always available) and ID embedding (available after warm-start). Item ID embedding starts at zero/random and learns as interactions accumulate.

---

### Q6: What biases exist in implicit feedback data and how do they affect your model?

**What they're testing:** Holistic understanding of bias taxonomy.

**Strong answer:** Cover position bias (inflated labels at top positions), selection/exposure bias (model only trains on shown items), popularity bias (frequent items dominate training signal), and feedback loops (current policy shapes future data). For each, explain the mechanism and the correction. End with: "The hardest part is that these biases are compounding — a popularity-biased retrieval model sends biased candidates to the ranker which then trains on biased labels from a position-biased display." Senior signal: articulate that each debiasing technique assumes a model of the bias mechanism, and if that model is wrong, debiasing can hurt.

---

### Q7: How does LambdaMART work and why might you prefer it over training with binary cross-entropy?

**What they're testing:** Mathematical depth in learning-to-rank.

**Strong answer:**
- LambdaMART uses pseudo-gradients that approximate the gradient of NDCG
- NDCG is non-differentiable; LambdaMART avoids differentiating NDCG directly
- The lambda gradient $\lambda_{ij}$ weights the pairwise gradient by |ΔNDCG| — items whose relative swap would most improve NDCG get the largest gradient signal
- Combined with MART (gradient boosted trees): LambdaMART = LambdaRank gradients + GBDT training
- When to prefer over BCE: you have dense relevance judgments (human labels or multi-level engagement labels), and you care directly about NDCG rather than probability calibration

**Follow-up:** "What are its limitations?" → Requires full query-item lists at training time, computationally heavier than pointwise, doesn't produce calibrated probabilities.

---

### Q8: How would you design a multi-task ranking model for an e-commerce platform optimizing CTR, CVR, and basket size?

**What they're testing:** MTL architecture knowledge + handling conflicting objectives.

**Strong answer:**
- Use ESMM for the CTR/CVR relationship: model P(conversion) = P(click) × P(conversion|click) to address CVR selection bias
- Add a third head for basket size (regression)
- Use MMoE or PLE to handle negative transfer between tasks — basket size optimization may conflict with CTR
- Final score: weighted linear combination calibrated via offline experiments and A/B tests
- Gradient monitoring: watch for one task's gradients dominating; use uncertainty weighting or gradient surgery

---

### Q9: What's the difference between NDCG and AUC, and when would you pick each?

**What they're testing:** Precision about metric semantics.

**Strong answer:**
- AUC: global pairwise ranking quality. Does the model score positives above negatives? Doesn't care about *position* within the list. Best for binary prediction quality.
- NDCG: list quality at a specific cutoff K. Position-sensitive, handles graded relevance. Best for evaluating ranked lists where position matters.
- Use AUC when: evaluating a scoring model, calibration matters, no natural list cutoff
- Use NDCG when: a specific list is presented to the user, positions matter, you have graded relevance
- Use GAUC over global AUC for recommendation: global AUC can be gamed by a model that's good on the most common users but bad on personalization

---

### Q10: How do you evaluate a new retrieval model when you only have data from the old model?

**What they're testing:** Off-policy evaluation and understanding of retrieval evaluation challenges.

**Strong answer:**
- Direct evaluation is biased: test set only contains items the old model retrieved. New model may retrieve better items that are absent from the test set.
- IPS/OPE: weight recall metrics by inverse of old model's retrieval probability — but old model usually uses ANN (not a probabilistic policy), so propensity estimation is non-trivial.
- Randomized evaluation: run a fraction of traffic with randomly sampled candidates. Evaluate new model's ranking within the random candidate set.
- Human evaluation: sample new model's candidates that are *not* in old model's candidates. Human judges assess relevance. Expensive but unbiased.
- Counterfactual retrieval recall: if you have groundtruth relevance labels from search, compare retrieval recall against that groundtruth.

---

### Q11: How would you use knowledge distillation to speed up a ranking model?

**What they're testing:** Practical model compression knowledge.

**Strong answer:**
- Teacher: large fine-ranker with full feature set, serving at 100ms
- Student: smaller model with subset of features, targeting 10ms
- Training the student: minimize KL divergence between teacher's softmax distribution and student's, in addition to ground-truth labels
- List-wise distillation: distill the ranking, not just individual scores — student matches teacher's top-K ordering via rank distillation loss
- Key tradeoff: which features to drop for the student. Features with high marginal value but high serving cost (e.g., real-time user signals requiring fresh lookups) are prime candidates for approximation or removal.

---

### Q12: Explain feedback loops in recommendation systems and how you'd mitigate them.

**What they're testing:** Systems-level thinking about data quality.

**Strong answer:**
- The loop: recommendations → user behavior → training data → next recommendations
- Effects: content diversity collapse, filter bubbles, popularity amplification
- Mitigation at training level: off-policy correction (importance sampling), counterfactual training data
- Mitigation at serving level: exploration (epsilon-greedy, Thompson sampling), diversity constraints (MMR, DPP)
- Mitigation via metrics: track long-term diversity and satisfaction, not just short-term CTR
- The fundamental challenge: any single correction only breaks the loop locally. Full mitigation requires system-level thinking: diverse retrieval sources, calibrated exploration, long-term user value metrics.

---

### Q13: What is doubly robust estimation and when would you use it in recommendation evaluation?

**What they're testing:** Advanced OPE knowledge.

**Strong answer:**
- DR combines a direct model (DM) with an IPS correction on the residuals
- If the DM is wrong, the IPS correction fixes it. If the IPS weights are noisy, the DM provides a baseline.
- "Doubly robust" = consistent if either the outcome model or propensity model is correctly specified
- Use it when: evaluating a new policy offline using logged data, and you have access to both a propensity model and a reward model
- Vs IPS alone: lower variance, comparable bias
- Vs DM alone: unbiased when DM is misspecified

---

### Q14: How would you handle a scenario where your ads ranking model is performing well on CTR but you're getting complaints about ad relevance from users?

**What they're testing:** Multi-objective thinking + label quality awareness.

**Strong answer:**
- CTR is a noisy proxy for relevance. Users click on deceptive or curiosity-bait ads even if they're unsatisfied post-click.
- Add post-click satisfaction signals: dwell time, conversion, explicit feedback (thumbs down)
- Retrain as a multi-task model: CTR + post-click satisfaction
- Add negative engagement features: skip rates, ad hide/block rates, complaint rates
- Consider long-term user value: track whether users who see more of the "high CTR" ads reduce session length over time → survivorship/feedback loop signal
- The core insight: optimizing a single short-term metric (CTR) in isolation is a known failure mode. Staff engineers push for multi-objective, long-term-aware objectives.

---

### Q15: How does in-batch negative sampling introduce popularity bias in two-tower training, and how do you correct for it?

**What they're testing:** Deep understanding of two-tower training dynamics.

**Strong answer:**
- In-batch negatives: for each (user, positive item) pair in a batch, all other items in the batch serve as negatives.
- The batch is sampled proportional to item frequency in the training data → popular items appear more often → more frequently as negatives → model learns that popular items should be demoted.
- At serving time, popular items are over-suppressed → retrieval quality degrades for popular-but-relevant items.
- Correction: log-Q correction. Subtract $\log q_j$ from the logit for item j, where $q_j$ is its sampling probability. This accounts for the fact that item j was not a "true negative" — it was just sampled because it was popular.
- Formula (see Section 6.1) — must know this cold for a staff-level interview.
- Additional fix: use a mixed strategy with some uniformly sampled negatives to reduce the proportion of popularity-biased negatives.

---

## Appendix: What Separates Senior from Staff Answers

| Topic | Senior Answer | Staff Answer |
|---|---|---|
| Bias | Can name position bias and describe IPS | Explains the compounding of multiple biases, knows when debiasing can hurt, designs experiments to measure propensity |
| Multi-stage ranking | Knows retrieval → ranking exists | Explains the non-decomposability constraint, evaluates each stage independently, designs the cascade with appropriate recall/precision tradeoffs |
| MTL | Knows shared bottom + task heads | Designs with negative transfer in mind, uses MMoE/PLE, monitors gradient conflicts, knows ESMM for CVR |
| Offline evaluation | Knows NDCG, AUC, log loss | Explains offline-online gap mechanistically, uses IPS-corrected metrics, advocates temporal splits, uses DR estimation for OPE |
| Feedback loops | Can describe the loop conceptually | Proposes system-level interventions, tracks long-term diversity metrics, understands importance sampling corrections |
| Model choice | Can describe each model family | Selects based on latency budget, feature type, interaction complexity, and deployment constraints. Knows when not to use deep learning. |
| Negative sampling | Uses random negatives | Uses hard negatives with curriculum, applies popularity correction, understands the evaluation bias introduced by each strategy |
| Cold start | Content-based fallback | Distinguishes user/item/context cold start, designs a hybrid tower that degrades gracefully, considers meta-learning |

---

*Study guide built for Senior / Staff ML Engineer interview prep at Google, Meta, Netflix, Amazon, TikTok-level recommendation system roles. Covers modeling-first topics: problem formulation, model families, biases, offline evaluation, training strategies, and multi-stage system design.*
