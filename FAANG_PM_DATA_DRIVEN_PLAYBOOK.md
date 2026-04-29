# The FAANG PM Data-Driven Playbook
### A Practical Guide for Product Managers at E-Commerce & Customization Businesses

> Written for a PM at Framebridge — a company that sells custom framing online. All examples, case studies, and frameworks are grounded in e-commerce, customization, and logistics contexts. The goal is to help you think and operate like a senior data-driven PM at Meta, Amazon, or Google.

---

## Table of Contents

1. [Foundations of Data-Driven Product Thinking](#1-foundations-of-data-driven-product-thinking)
2. [Core Metrics & Instrumentation](#2-core-metrics--instrumentation)
3. [Experimentation & Causal Inference](#3-experimentation--causal-inference)
4. [Analytical Thinking & SQL](#4-analytical-thinking--sql)
5. [Data Science Collaboration](#5-data-science-collaboration)
6. [Personalization & Recommendation Systems](#6-personalization--recommendation-systems)
7. [Real-World Case Studies](#7-real-world-case-studies)
8. [Mental Models & Heuristics](#8-mental-models--heuristics)
9. [Tools & Stack](#9-tools--stack)
10. [30-60-90 Day Learning Plan](#10-30-60-90-day-learning-plan)

---

## 1. Foundations of Data-Driven Product Thinking

### 1.1 How FAANG PMs Frame Problems Using Data

The single biggest difference between a junior PM and a Staff-level PM isn't tool knowledge — it's **how precisely they frame the question before touching any data**.

At FAANG, a PM encountering "conversion is down" does not immediately go to dashboards. They first ask:

> **"What specific claim am I trying to validate or refute, and what evidence would change my mind?"**

This forces a falsifiable hypothesis before analysis begins. It prevents HiPPO-driven (Highest Paid Person's Opinion) analysis and the post-hoc rationalization trap where you find data to support what you already believe.

**The FAANG PM Problem-Framing Sequence:**

```
1. Observe: "Checkout conversion dropped 4% week-over-week"
2. Bound it: When did it start? Which segment? Which device? Which step?
3. Hypothesize: Generate 3–5 plausible explanations ranked by prior probability
4. Identify evidence: For each hypothesis, what data would confirm or falsify it?
5. Collect evidence: Pull only the data needed to confirm/falsify each hypothesis
6. Synthesize: What's the most likely explanation given the evidence?
7. Act: What's the recommended action, and how will we measure if it worked?
```

**Framebridge example:**

> Someone says "our frame recommendation widget isn't working." A junior PM pulls click-through rate. A FAANG PM asks: Not working *how*? Users aren't clicking it? They're clicking but not converting? They're converting but at a lower AOV? Or the recommendations are technically surfacing but are irrelevant? Each interpretation points to a completely different fix.

---

### 1.2 Types of Product Questions

Understanding what *kind* of question you're asking prevents using the wrong analytical tool.

| Question Type | What You're Asking | Tooling |
|---|---|---|
| **Descriptive** | What happened? | Dashboards, SQL aggregations |
| **Diagnostic** | Why did it happen? | Funnel analysis, segmentation, drill-downs |
| **Exploratory** | What patterns exist? | Cohort analysis, clustering, EDA |
| **Causal** | Did X cause Y? | A/B tests, difference-in-differences, synthetic control |
| **Predictive** | What will happen? | ML models, regression, forecasting |
| **Prescriptive** | What should we do? | Optimization models, decision frameworks |

**The most common PM mistake:** Treating a *diagnostic* question as if it were *causal*. You segment users by who used the frame recommendation widget and see they convert 2x higher — but that's selection bias. Those users were already more engaged. Causality requires an experiment.

**Framebridge examples by type:**

- **Descriptive:** What percentage of customers complete the mat selection step in our custom framing flow?
- **Diagnostic:** Why did our repeat purchase rate decline in Q3?
- **Exploratory:** Are there distinct customer segments by the type of art they frame (e.g., photos vs. fine art)?
- **Causal:** Does showing a "how it looks on your wall" room visualization increase conversion?
- **Predictive:** Which customers are most likely to churn after their first order?

---

### 1.3 Decision-Making Under Uncertainty

FAANG PMs don't wait for perfect data. They develop calibrated confidence thresholds.

**The confidence-to-action ladder:**

| Confidence Level | Appropriate Action |
|---|---|
| ~90%+ | Ship / invest fully |
| ~70–90% | Run a limited experiment or staged rollout |
| ~50–70% | Run a low-cost test (survey, prototype, 5-user study) |
| <50% | Do more discovery; don't build yet |

**Key principle — reversibility changes the bar:**
- Reversible decisions (change a CTA label, adjust pricing on one SKU): Act with lower confidence. Bias toward learning.
- Irreversible decisions (change the core framing configurator UX, re-platform checkout, change refund policy): Require higher confidence. Slow down.

**Jeff Bezos's Type 1 / Type 2 distinction:** Type 1 decisions are one-way doors (hard to undo). Type 2 are two-way doors. Most product decisions are Type 2. PMs who treat all decisions as Type 1 are too slow. PMs who treat all decisions as Type 2 create technical debt and customer trust problems.

**Working with uncertainty at Framebridge:**

Custom framing has unusually long feedback loops — a customer orders, production takes 5–7 days, shipping takes another 3–5 days, and the emotional payoff (seeing it on their wall) happens later. This means your metrics lag your decisions by weeks. You'll need to identify *leading indicators* (early signals like photo upload completion, preview engagement, time spent in the configurator) that predict downstream satisfaction and repeat purchase.

---

## 2. Core Metrics & Instrumentation

### 2.1 Defining North Star Metrics for an E-Commerce / Custom Product Business

A **North Star Metric (NSM)** is the single number that best captures the core value your product delivers to customers, and that best predicts long-term business health. It is *not* revenue — revenue is an output.

**NSM Evaluation Criteria:**
1. Does it reflect value delivered to the customer?
2. Does it lead to long-term retention and revenue?
3. Can your team's work actually move it?
4. Is it hard to game?

**Candidate NSMs for Framebridge:**

| Candidate | Pros | Cons |
|---|---|---|
| # of frames shipped per month | Reflects real value delivered, tied to production | Doesn't capture quality or customer satisfaction |
| Completed orders (first purchase) | Acquisition focus, clear funnel | Ignores retention; one-time purchasers are low LTV |
| **Repeat purchase rate (% customers with 2+ orders in 12 months)** | Strong LTV signal, reflects product quality and trust | Lags 3–6 months; hard to run experiments on |
| Net Promoter Score (NPS) | Customer love signal | Survey-based, biased sample |
| Weekly Active Framers (people actively browsing/designing) | Engagement NSM (like DAU for apps) | Doesn't capture transactional value |

**Recommended NSM approach for Framebridge:** `% of customers who place a 2nd order within 6 months of their 1st order`

Why: Framebridge's moat is quality + experience. If customers come back, you've earned trust. If they don't, something went wrong in design, production, packaging, or customer service. This metric also aligns marketing, product, and ops around the same goal.

**Secondary metric to track weekly:** `Configurator-to-checkout conversion rate` — this is fast-moving, experimentable, and predictive of near-term revenue.

---

### 2.2 Input vs. Output Metrics, Leading vs. Lagging Indicators

**Output metrics** measure business results (revenue, orders, NPS). They are what you care about but can't directly control.

**Input metrics** measure actions your team takes that *cause* the output (# experiments launched, upload completion rate, recommendation click-through rate). These are actionable.

**Lagging indicators** confirm past performance (repeat purchase rate, annual revenue, LTV). They're reliable but slow.

**Leading indicators** predict future outcomes (photo upload success rate, time-in-configurator, support ticket volume, cart abandonment at mat step). They're faster but noisier.

**FAANG PMs build metric trees.** For every output metric, they decompose it into controllable inputs:

```
Repeat Purchase Rate (Output / Lagging)
├── 1st-order NPS / satisfaction score (Leading)
├── Post-purchase email open + click rate (Leading)
├── % customers who redeem a "welcome back" promo (Leading)
├── Time between 1st order delivery and account login (Leading)
└── Support ticket rate for 1st orders (Leading — negatively correlated)
```

**The PM's job is to own the inputs, not obsess over the output.** You can't will people to reorder. You can improve their satisfaction, send better email sequences, surface reorder triggers, and fix quality issues.

---

### 2.3 Designing Event Tracking and Data Schemas

**What to log and why.** Most companies under-instrument early and pay for it later. The rule of thumb:

> Log every user action that *could* be meaningful to a product decision, even if you don't know why today.

**Event Taxonomy for Framebridge:**

```
Event Name: <object>_<action>
Examples:
  - photo_uploaded
  - photo_crop_adjusted
  - frame_style_selected
  - mat_color_selected
  - size_changed
  - preview_viewed
  - room_visualizer_opened
  - cart_item_added
  - checkout_started
  - payment_info_entered
  - order_placed
  - order_delivered (server-side)
  - reorder_cta_clicked
  - recommendation_widget_viewed
  - recommendation_clicked
  - recommendation_added_to_cart
```

**Every event should carry these standard properties:**

```json
{
  "event": "frame_style_selected",
  "timestamp": "2026-04-29T14:32:00Z",
  "user_id": "u_abc123",           // anonymous if not logged in → use device_id
  "session_id": "s_xyz456",
  "device_type": "mobile",
  "platform": "web",
  "experiment_variants": ["exp_001_v2", "exp_003_control"],  // always carry active experiments
  "order_id": null,                 // null until order is placed
  "item_id": "frame_sku_0042",
  "properties": {
    "frame_style": "classic_black",
    "selection_method": "recommended"  // vs "browsed"
  }
}
```

**The biggest instrumentation mistakes:**

1. **Not logging the "non-event"** — e.g., logging `frame_style_selected` but not logging when a user *views* the style selector and leaves without selecting. You can't compute abandonment without the view event.

2. **Not carrying experiment context** — if you don't stamp which experiment variant a user was in at the time of an event, you can't do post-hoc analysis.

3. **Logging too late in the funnel** — if you only log `order_placed`, you'll never know where users dropped off.

4. **Schema drift** — `frame_style_selected` fires with `{style: "black"}` in one version and `{frame_style: "classic_black"}` in the next. Always version your schemas.

---

## 3. Experimentation & Causal Inference

### 3.1 How to Design and Evaluate A/B Tests

**The lifecycle of a good experiment:**

```
1. Hypothesis: "Showing a 'room visualizer' increases checkout conversion because
   customers are uncertain how the finished frame will look in their space."

2. Primary metric: Checkout conversion rate (order / configurator_started)

3. Secondary metrics: AOV, time-in-configurator, photo_upload rate, NPS

4. Guardrail metrics: Page load time, support ticket rate, production defect rate
   (you don't want a win on conversion + a loss on production quality)

5. Unit of randomization: user_id (logged in) or device_id (anonymous)
   — randomize at the user level, not session level, to avoid within-user contamination

6. Sample size calculation:
   - Current conversion rate: 32%
   - Minimum detectable effect (MDE): 2% absolute (≈6.25% relative)
   - Power: 80%, alpha: 0.05
   - Required sample: ~3,800 users per variant
   — At Framebridge's traffic volume, this tells you how many weeks to run

7. Randomization check: Are variants balanced on age, device, acquisition channel?
   Run a pre-experiment AA test if possible.

8. Experiment duration: At least 2 full business cycles (usually 2 weeks minimum)
   to account for day-of-week effects and novelty.
```

**Evaluating Results:**

Don't just ask "is p < 0.05?" Ask:

- Is the effect size *practically* meaningful (not just statistically significant)?
- Did any guardrail metrics move adversely?
- Is the effect consistent across device types and customer segments?
- Is there a "day 1 effect" that fades over time? (novelty effect)
- Does the confidence interval include effects that would change our decision?

**A/B Test Decision Matrix:**

| Stat Sig? | Practically Meaningful? | Decision |
|---|---|---|
| Yes | Yes | Ship it |
| Yes | No | Don't ship — not worth maintenance cost |
| No | Large potential effect | Run longer / increase traffic |
| No | Small effect | Accept null; move on |

---

### 3.2 Common Experimentation Pitfalls

**Selection Bias:**
If you segment experiment results *after* the experiment by, say, "users who clicked the room visualizer," you'll see inflated results. Users who clicked were already more engaged. Only analyze by the randomization unit (user_id), not by downstream behavior.

**Novelty Effect:**
When you ship a new UI element at Framebridge (e.g., a new mat preview carousel), users explore it out of curiosity in the first few days regardless of quality. Performance often degrades after the novelty wears off. Mitigation: Run experiments for at least 2 weeks and analyze early vs. late cohorts separately.

**P-Hacking / Peeking:**
If you check results every day and stop when p < 0.05, you'll false-positive 30–40% of the time. Use one of:
- **Fixed horizon testing:** Commit to a sample size before starting. Don't peek until you hit it.
- **Sequential / always-valid testing:** Use methods like mSPRT or Bayesian sequential testing that allow early stopping without inflating false-positive rates (Optimizely, Statsig, and Eppo support this).

**Network Effects / Spillover:**
Less relevant for Framebridge since customers are independent, but if you ever run social features (e.g., "share your frame"), treat-and-control cross-contamination becomes a real issue.

**SUTVA Violation (Stable Unit Treatment Value Assumption):**
Every unit must receive exactly one treatment and the treatment of one unit must not affect others. At Framebridge, this is mostly safe unless you test pricing (users may share promo codes) or inventory (a treatment that drives more orders might cause production delays that affect control group delivery times too).

**Multiple Comparisons:**
If you run 20 experiments simultaneously and look for any p < 0.05, you'll get ~1 false positive by chance. Use Bonferroni correction or a false discovery rate (FDR) threshold when running many tests simultaneously.

---

### 3.3 When NOT to Run Experiments

A/B testing is overused. Not everything should be an experiment.

**Don't experiment when:**

1. **The baseline is too small.** If you get 50 orders/week, you'd need months to see a 5% lift. Run a qualitative study instead.

2. **The change is obviously net-positive and low-risk.** Fixing a broken link, correcting a typo, removing a dead-end 404. Just ship it.

3. **You're asking the wrong question.** If you don't know *what* to test or *why* it might work, do discovery work (user research, session recordings, interviews) first. Testing a bad hypothesis wastes traffic and time.

4. **There's a legal, ethical, or brand risk.** Don't A/B test pricing that could be perceived as discriminatory. Don't test checkout friction in ways that could erode trust.

5. **You need to learn, not confirm.** If you want to understand *why* customers abandon at the mat selection step, a user test or survey will give you richer insight faster than an A/B test.

**Alternatives to A/B testing:**
- User interviews / usability studies (qualitative, fast)
- Fake door tests (measure intent without building the full feature)
- Holdout/holdback (measure the overall impact of a program, not individual features)
- Difference-in-differences (compare before/after changes in a natural experiment)
- Regression discontinuity (compare users just above/below a threshold, e.g., free shipping cutoff)

---

### 3.4 Interpreting Ambiguous or Conflicting Results

Real experiments are messy. Here's how to handle it:

**Scenario 1: Primary metric neutral, secondary metric moves**
- The new room visualizer didn't lift conversion, but time-in-configurator increased 15% and AOV increased $8.
- Interpretation: Customers are more engaged and buying more expensive frames, but the total number converting didn't change. Might be a win on LTV; run a follow-on analysis on repeat purchase rate in 90 days.

**Scenario 2: Positive on desktop, negative on mobile**
- Ship for desktop, keep control for mobile, and investigate *why* mobile is different (performance? UX? visualizer doesn't render well on small screens?).

**Scenario 3: Statistically significant but directionally weird**
- The "simpler checkout" experiment increased conversion *and* increased support ticket rate. Dig in: Were customers confused about something? Did you accidentally remove something important? Don't celebrate until you understand the mechanism.

**Scenario 4: Strong early signal that fades**
- Classic novelty effect. Discount the first 3–5 days of data and re-evaluate the remaining period. If the effect disappears, it's not a real improvement.

---

## 4. Analytical Thinking & SQL

### 4.1 Key Analyses Every PM Should Know

#### Funnel Analysis

The most fundamental analysis in e-commerce. Define sequential steps and calculate drop-off at each stage.

**Framebridge configurator funnel:**
```
1. Landing on product page
2. Photo upload started
3. Photo upload completed
4. Frame style selected
5. Mat color selected
6. Size confirmed
7. Added to cart
8. Checkout started
9. Payment entered
10. Order placed
```

For each step, you want:
- `Step conversion rate`: % who reach step N of those who reached step N-1
- `Overall funnel conversion rate`: % who completed step 10 of those who entered step 1
- Breakdown by: device, acquisition channel, new vs. returning, experiment variant

**What to look for:** A single step with a dramatically lower conversion rate than adjacent steps is your biggest opportunity. A drop from 70% at "frame style selected" to 40% at "mat color selected" screams that mat selection is confusing or overwhelming.

#### Retention & Cohort Analysis

Retention answers: Of the customers who first ordered in Month X, what % are still active (ordered again) in Month X+1, X+2, ... X+12?

**Framebridge retention curve shape:**
Because Framebridge is a considered purchase (not a daily habit), you'd expect:
- Very low 30-day retention (most people don't need another frame in a month)
- Moderate 6-month retention (seasonal gifting: holidays, graduations, new homes)
- The goal is to *steepen the curve* — get more people back within 6 months

If your retention curve flattens at >0%, you have a retained core. If it keeps declining to zero, you're not building a habit loop.

#### Segmentation

Divide users into meaningful groups and compare behavior. The goal is to find groups with *different product needs* that your one-size-fits-all product is serving unequally.

**Framebridge segmentation cuts to run:**
- By photo type (personal photo framers vs. art/print framers vs. gifters)
- By acquisition channel (paid social vs. organic vs. referral vs. direct — do different channels produce different LTV customers?)
- By device type at first visit
- By number of photos uploaded before purchasing
- By geographic region (are there regional preferences in frame styles?)
- By order value tier (budget customers vs. premium customers)

---

### 4.2 How to Debug a Drop in Conversion or Revenue

This is one of the most common PM interview questions and one of the most common real-world scenarios. Here is the systematic approach:

**Step 1: Confirm the drop is real**
- Is this a data pipeline issue? Check if event logging is healthy (events per day plot).
- Is the denominator changing? Revenue can drop if traffic drops even if conversion is stable.
- Is this in a specific date range, or is it a gradual trend?

**Step 2: Decompose the metric**
```
Revenue = Visitors × Conversion Rate × Average Order Value
```
Which component dropped? Each has different root causes.

**Step 3: Segment to isolate**
- Is it all devices or one? (new web deploy broke mobile)
- Is it all acquisition channels or one? (paid campaign targeting changed)
- Is it a specific product category? (frame style went out of stock)
- Is it new users or returning users?
- Is it a specific geo?

**Step 4: Look at funnel drop-offs**
- Is there a specific step where abandonment increased? Compare this week to baseline.
- Overlay any recent product changes, marketing campaigns, or infrastructure deployments.

**Step 5: Check for external factors**
- Is there a competitor sale happening?
- Did a major news event reduce consumer spending?
- Is it a seasonal/holiday timing effect?

**Step 6: Form and test hypotheses**
By Step 3 and 4, you should have a primary suspect. Now validate: Look for corroborating evidence, session recordings, or error logs.

---

### 4.3 Example SQL Queries

These use standard SQL syntax and should translate to Snowflake, BigQuery, or Redshift.

**Query 1: Funnel conversion rates by device type**

```sql
WITH funnel AS (
  SELECT
    session_id,
    user_id,
    device_type,
    MAX(CASE WHEN event_name = 'photo_upload_started' THEN 1 ELSE 0 END)   AS started_upload,
    MAX(CASE WHEN event_name = 'photo_upload_completed' THEN 1 ELSE 0 END) AS completed_upload,
    MAX(CASE WHEN event_name = 'frame_style_selected' THEN 1 ELSE 0 END)   AS selected_frame,
    MAX(CASE WHEN event_name = 'cart_item_added' THEN 1 ELSE 0 END)        AS added_to_cart,
    MAX(CASE WHEN event_name = 'order_placed' THEN 1 ELSE 0 END)           AS placed_order
  FROM events
  WHERE event_date >= CURRENT_DATE - 30
  GROUP BY session_id, user_id, device_type
)
SELECT
  device_type,
  COUNT(*) AS sessions,
  ROUND(AVG(started_upload) * 100, 1)    AS pct_started_upload,
  ROUND(AVG(completed_upload) * 100, 1)  AS pct_completed_upload,
  ROUND(AVG(selected_frame) * 100, 1)    AS pct_selected_frame,
  ROUND(AVG(added_to_cart) * 100, 1)     AS pct_added_to_cart,
  ROUND(AVG(placed_order) * 100, 1)      AS pct_placed_order
FROM funnel
GROUP BY device_type
ORDER BY sessions DESC;
```

**Query 2: 6-month cohort retention**

```sql
WITH first_orders AS (
  SELECT
    user_id,
    DATE_TRUNC('month', MIN(order_date)) AS cohort_month
  FROM orders
  GROUP BY user_id
),
subsequent_orders AS (
  SELECT
    o.user_id,
    f.cohort_month,
    DATEDIFF('month', f.cohort_month, DATE_TRUNC('month', o.order_date)) AS months_since_first
  FROM orders o
  JOIN first_orders f ON o.user_id = f.user_id
  WHERE DATEDIFF('month', f.cohort_month, DATE_TRUNC('month', o.order_date)) > 0
)
SELECT
  cohort_month,
  months_since_first,
  COUNT(DISTINCT user_id) AS retained_users
FROM subsequent_orders
GROUP BY cohort_month, months_since_first
ORDER BY cohort_month, months_since_first;
-- Join back to cohort size from first_orders to get retention %
```

**Query 3: Revenue attribution by acquisition channel with LTV proxy**

```sql
SELECT
  u.acquisition_channel,
  COUNT(DISTINCT u.user_id)                                    AS total_customers,
  COUNT(DISTINCT o.order_id)                                   AS total_orders,
  ROUND(COUNT(DISTINCT o.order_id) / COUNT(DISTINCT u.user_id)::FLOAT, 2) AS orders_per_customer,
  ROUND(SUM(o.revenue_usd), 2)                                 AS total_revenue,
  ROUND(SUM(o.revenue_usd) / COUNT(DISTINCT u.user_id), 2)    AS ltv_proxy,
  ROUND(AVG(o.revenue_usd), 2)                                 AS avg_order_value
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
  AND o.order_date BETWEEN u.signup_date AND DATEADD('month', 12, u.signup_date)
GROUP BY u.acquisition_channel
ORDER BY ltv_proxy DESC;
```

**Query 4: Identify customers at risk of churn (no reorder within 90 days post-delivery)**

```sql
WITH delivered_orders AS (
  SELECT
    o.user_id,
    MAX(o.delivered_date) AS last_delivery_date,
    COUNT(o.order_id) AS total_orders
  FROM orders o
  WHERE o.delivered_date IS NOT NULL
  GROUP BY o.user_id
)
SELECT
  user_id,
  last_delivery_date,
  total_orders,
  DATEDIFF('day', last_delivery_date, CURRENT_DATE) AS days_since_last_delivery
FROM delivered_orders
WHERE
  DATEDIFF('day', last_delivery_date, CURRENT_DATE) BETWEEN 60 AND 120
  AND total_orders = 1  -- first-time buyers
ORDER BY last_delivery_date;
-- These are your churn-risk targets for re-engagement campaigns
```

**Query 5: A/B test result summary**

```sql
SELECT
  variant,
  COUNT(DISTINCT user_id)                                 AS users,
  SUM(placed_order)                                       AS conversions,
  ROUND(AVG(placed_order) * 100, 2)                      AS conversion_rate_pct,
  ROUND(AVG(CASE WHEN placed_order = 1 THEN order_value END), 2) AS avg_order_value,
  ROUND(SUM(order_value) / COUNT(DISTINCT user_id), 2)  AS revenue_per_user
FROM experiment_assignments ea
LEFT JOIN funnel f ON ea.user_id = f.user_id
WHERE ea.experiment_id = 'exp_room_visualizer_2026q2'
GROUP BY variant;
```

---

## 5. Data Science Collaboration

### 5.1 How to Work Effectively with Data Scientists and ML Engineers

The most common failure mode: PMs hand DS teams a business problem ("increase repeat purchase") without any structure, and the DS team spends 6 weeks building a beautiful churn model that nobody uses because nobody defined how it would plug into any product surface.

**The PM's job in a DS collaboration:**

1. **Define the decision, not the model.** Before any DS work starts, answer: "If a data science model told us X, what exactly would we do differently?" If the answer is unclear, the model will collect dust.

2. **Write a crisp problem statement:**
   ```
   Current state: We send the same re-engagement email to all customers 30 days post-delivery.
   Open rate: 18%, click rate: 3%, reorder rate: 0.8%
   
   Desired state: Send personalized re-engagement emails that surface the most relevant
   upsell product for each customer based on their previous order and browsing history.
   
   If a model could predict which product category a customer is most likely to order next
   with >60% accuracy, we believe email click rate could reach 8% and reorder rate 2%.
   
   Success metric: 90-day reorder rate for treated cohort vs. holdout.
   Guardrail: Unsubscribe rate must not increase.
   ```

3. **Agree on a human baseline first.** Before building an ML model, establish what a rule-based heuristic achieves. "Customers who bought a photo frame are most likely to buy another photo frame" is a simple rule. The ML model should beat this. If it doesn't, the simple rule is your answer.

4. **Ask for confidence intervals, not point estimates.** A DS who says "the model predicts a 14% lift" without uncertainty bounds is giving you false confidence. Demand: "What's the 80% confidence interval on that lift?"

5. **Plan for the model going stale.** ML models degrade. Seasonality, product changes, and customer shifts all move the input distribution. Before shipping, agree on: What's the monitoring plan? What triggers a retrain? Who owns it?

---

### 5.2 When to Use Heuristics vs. ML Models

**Use heuristics (rule-based logic) when:**
- The pattern is obvious and stable (customers who bought a vertical print should see vertical frame recommendations)
- You have too little data to train a model (<10K examples)
- The decision needs to be explainable to customers or regulators
- The cost of being wrong is high and you need interpretability
- You need to ship in 2 weeks, not 2 months

**Use ML models when:**
- There are too many variables for a human to manually tune rules
- The pattern is nonlinear or changes over time (e.g., seasonal trends in art styles)
- You have sufficient labeled training data (>50K+ examples for most tasks)
- The model can be evaluated and monitored in a controlled way
- You have DS capacity and the ROI justifies the investment

**Framebridge rule of thumb:** Start with rules. Productionize rules. Measure. Then justify an ML investment with *evidence that rules are leaving money on the table*, not just because ML sounds impressive.

---

### 5.3 Translating Business Problems into DS Problems

**Template for problem translation:**

```
Business Problem: [Describe in plain English]

Input: [What data does the model see at inference time?]
Output: [What does the model predict / score / rank?]
Label / ground truth: [What defines a correct answer?]
Objective function: [What are we optimizing?]
Constraints: [Latency, explainability, fairness, cost]
Evaluation metric: [How do we measure model quality offline? Precision@K? AUC? RMSE?]
Business metric: [How do we measure success in production?]
```

**Framebridge example — frame style recommendation:**

```
Business Problem: Customers who land on the configurator are overwhelmed by 80+ frame
styles and often pick the first one or abandon. We want to surface the 5 most
relevant styles for each customer.

Input: Customer's photo (image embedding), past orders (if any), browse history,
       session context (device, referral source), demographic signals

Output: Ranked list of top 5 frame style IDs for this customer

Label: Did the customer purchase one of the recommended styles? (implicit positive)
       Did the customer skip recommendations and pick something else? (implicit negative)

Objective: Maximize CTR on top recommendation × downstream conversion

Constraints: Must return recommendations in <100ms (cache allowed); must be
             explainable to merchandising team; must not recommend out-of-stock styles

Evaluation (offline): Precision@5, NDCG@5 on held-out purchase data
Evaluation (online): A/B test: conversion rate, AOV, rec click-through rate
```

---

## 6. Personalization & Recommendation Systems

### 6.1 How Personalization Applies to Framebridge

Framebridge sits at a unique intersection: customers bring their own creative asset (a photo or art print) but need guidance on how to showcase it. This is a perfect environment for personalization because:

1. **The input varies widely** — a family portrait, a vintage concert poster, and an abstract watercolor all have different optimal frame pairings.
2. **Customers lack expertise** — most people don't know what a "float mount" is or when to use a mat vs. no mat.
3. **The stakes feel high** — customers are framing something meaningful. Confidence in their choice matters.

**Personalization Surfaces at Framebridge:**

| Surface | What to Personalize | Signal Available |
|---|---|---|
| Frame style recommendations | Top 5 frame styles for this photo | Photo content, past orders, browse |
| Mat color suggestions | Color palette suggestions based on photo | Image color analysis |
| Size recommendations | Suggested size for the photo's resolution | Image dimensions + common room contexts |
| Bundle recommendations | "Customers who ordered this also added..." | Collaborative filtering on order data |
| Reorder email content | Product shown in re-engagement email | Last order category |
| Pricing / promotions | Targeted discount to price-sensitive customers | Browsing behavior, cart abandonment |
| Homepage hero | Photo category featured on homepage | Referral source, past behavior |

---

### 6.2 Tradeoffs: Rule-Based vs. ML-Based Personalization

**Rule-Based Personalization:**
```
IF photo has warm tones → recommend warm wood frames (walnut, maple)
IF photo is black-and-white → recommend classic black or white frames
IF customer has ordered before → show their previously used frame style first
IF photo dimensions are square → suggest square or slight landscape sizes
```

**Pros:** Fast to build, easy to explain, easy to debug, no training data needed.
**Cons:** Doesn't scale, misses complex patterns, requires manual curation, can't learn.

**ML-Based Personalization (Collaborative Filtering):**
"Customers who uploaded similar photos to yours also liked these 5 frame styles."

**Pros:** Scales to thousands of products, learns from real behavior, improves over time.
**Cons:** Cold start problem (new customers, new products), requires sufficient data, black box.

**Hybrid approach (recommended for Framebridge at current scale):**
1. Use image analysis (color, content detection) as features for content-based filtering.
2. Layer collaborative filtering for returning customers with order history.
3. Fall back to rule-based logic for new customers or edge cases.
4. Use a re-ranking layer to apply business rules (e.g., don't recommend discontinued styles).

---

### 6.3 Measuring Personalization Success

**Don't use "recommendation click-through rate" as your sole success metric.** Users can click recommendations that don't convert. You need the full funnel:

```
Metric pyramid for recommendation systems:

[Business Impact]      Repeat purchase rate, revenue, LTV
        ↑
[Conversion Quality]   Recommendation → cart → order conversion rate
        ↑
[Engagement]           Click-through rate on recommendations, time-in-configurator
        ↑
[Coverage]             % of users who receive personalized recommendations
        ↑
[Relevance]            Offline: Precision@K, NDCG; Online: A/B vs. random
```

**Holdout experiment design for personalization:**
Because personalization touches every session, you can't just A/B test a button. You need a *holdout group* — a % of users (typically 5–10%) who permanently receive no personalization. The difference in LTV between treated and holdout over 6–12 months is the true business value of your personalization system.

**Beware self-fulfilling recommendation loops:** If you only recommend popular items and only measure click-through rate, you'll keep recommending the same popular items forever. Over time, you've reduced your effective catalog to 10 items and trained customers to think Framebridge only has a few options. Diversity metrics (catalog coverage, intra-list diversity) matter.

---

## 7. Real-World Case Studies

### Case Study 1: Drop in Checkout Conversion

**Scenario:** Your weekly checkout conversion rate (orders / configurator starts) dropped from 32% to 26% over 2 weeks. Your VP asks why and what to do about it.

---

**Step 1: Confirm the drop is real**

Pull the raw event counts:
```sql
SELECT event_date, COUNT(DISTINCT session_id) AS configurator_starts,
       COUNT(DISTINCT CASE WHEN event_name = 'order_placed' THEN session_id END) AS orders,
       ROUND(COUNT(DISTINCT CASE WHEN event_name = 'order_placed' THEN session_id END) /
             COUNT(DISTINCT session_id)::FLOAT * 100, 2) AS conversion_pct
FROM events
WHERE event_name IN ('configurator_started', 'order_placed')
  AND event_date >= CURRENT_DATE - 30
GROUP BY event_date
ORDER BY event_date;
```

Check: Is traffic up (numerator looks flat because denominator grew)? Is there a data pipeline issue (events spiked or dropped suspiciously)? Confirm it's real.

---

**Step 2: Decompose — which of the three levers moved?**

Revenue = Traffic × Conversion × AOV

Traffic grew 10% from a new paid campaign. Conversion dropped from 32% to 26%. AOV was flat. So this is a pure conversion problem, likely linked to the new traffic being lower quality or hitting a broken experience on the device/channel those users use.

---

**Step 3: Segment the drop**

```sql
-- Compare conversion by acquisition channel, before vs. after the drop
SELECT
  acquisition_channel,
  CASE WHEN event_date < '2026-04-15' THEN 'before' ELSE 'after' END AS period,
  COUNT(DISTINCT session_id) AS sessions,
  ROUND(AVG(placed_order) * 100, 2) AS conversion_pct
FROM session_summary
WHERE event_date BETWEEN '2026-04-01' AND '2026-04-29'
GROUP BY acquisition_channel, period
ORDER BY acquisition_channel, period;
```

**Findings:** Instagram paid traffic went from 8% → 28% of all sessions after the new campaign launched. Instagram users convert at 18% (vs. 36% for organic). The new campaign brought in traffic that's proportionally pulling the blended rate down.

---

**Step 4: Diagnose further — is the Instagram traffic *also* converting worse than it used to?**

Yes: Instagram users converted at 22% before the campaign, now 18%. Why? The ad creative shows a specific frame style (black modern) but the landing page shows a general homepage. There's a creative-to-landing-page mismatch.

---

**Step 5: Recommendation**

1. **Immediate (this week):** Create dedicated landing pages that match each ad creative. Black modern ad → lands on black modern configurator start. This is a "message match" fix.
2. **Medium-term (next sprint):** A/B test a simplified configurator entry for paid social traffic — one click to start with the advertised frame pre-selected.
3. **Ongoing:** Separate your headline conversion metric into "organic conversion" and "paid conversion" so a campaign surge doesn't mask product problems.

---

### Case Study 2: Low Repeat Purchase Rate

**Scenario:** Only 18% of customers who placed their first order in Q4 (holiday season) placed a second order within 6 months. The company benchmark is 28%. You need to understand why and build a plan to improve it.

---

**Step 1: Characterize the 18% who did reorder**

```sql
SELECT
  -- Demographics
  u.acquisition_channel,
  -- 1st order characteristics
  o1.frame_category AS first_order_category,
  o1.revenue AS first_order_value,
  -- Time to reorder
  DATEDIFF('day', o1.order_date, o2.order_date) AS days_to_reorder,
  COUNT(*) AS count
FROM users u
JOIN orders o1 ON u.user_id = o1.user_id AND o1.order_rank = 1
JOIN orders o2 ON u.user_id = o2.user_id AND o2.order_rank = 2
WHERE DATE_TRUNC('quarter', o1.order_date) = '2025-10-01'
GROUP BY 1, 2, 3, 4
ORDER BY count DESC;
```

**Pattern found:** Customers who ordered a photo frame (vs. art/print frame) had 34% repeat rate. Customers who gave as a gift (inferred by shipping-to-different-address) had 9% repeat rate. Holiday gifters are a fundamentally different segment — they may have no need to reorder for themselves.

---

**Step 2: Check post-purchase experience for non-repeaters**

- Support ticket analysis: Did first-time buyers file more support tickets in this cohort? Yes — delivery delays due to holiday peak caused 12% of Q4 first-time buyers to contact support vs. 4% in Q2.
- NPS: Q4 cohort NPS was 42 vs. Q2 NPS of 61.
- Post-purchase email engagement: Q4 cohort opened re-engagement emails at 14% vs. 22% for Q2.

**The core problem:** Holiday production delays damaged the first impression for a disproportionate share of this cohort.

---

**Step 3: Build the intervention plan**

| Lever | Action | Expected Impact | Measurement |
|---|---|---|---|
| Address root cause | Improve holiday production capacity planning | Reduces satisfaction damage at source | On-time delivery rate, NPS |
| Recovery email sequence | Personalized apology + offer for delayed Q4 customers | +3–5% reorder rate on this cohort | Reorder rate for email recipients vs. holdout |
| Product-first retention | Re-engage self-purchase segment (non-gifters) with "What will you frame next?" prompt | +4–8% reorder rate | Reorder rate for segment |
| Gifter program | Build a "gift again" flow for gifters near Mother's Day, Valentine's Day | Addresses the gifter segment separately | Seasonal reorder rate |

---

### Case Study 3: High Abandonment After Photo Upload Step

**Scenario:** You notice that 65% of users who successfully upload a photo abandon the configurator before selecting a frame style. This is a major drop in your funnel.

---

**Step 1: Quantify and confirm**

```sql
SELECT
  COUNT(DISTINCT CASE WHEN event_name = 'photo_upload_completed' THEN session_id END) AS upload_completions,
  COUNT(DISTINCT CASE WHEN event_name = 'frame_style_selected' THEN session_id END)   AS frame_selections,
  ROUND(
    COUNT(DISTINCT CASE WHEN event_name = 'frame_style_selected' THEN session_id END)::FLOAT /
    COUNT(DISTINCT CASE WHEN event_name = 'photo_upload_completed' THEN session_id END) * 100,
  2) AS pct_who_select_frame
FROM events
WHERE event_date >= CURRENT_DATE - 14;
```

Confirmed: 35% move from upload-complete to frame_style_selected.

---

**Step 2: Time-based analysis**

Look at time between `photo_upload_completed` and next event:
- 40% of abandoners leave within 10 seconds of upload completing.
- This suggests the problem is *immediately* after upload — not a long deliberation followed by exit.

---

**Step 3: Qualitative investigation**

Watch 20 session recordings (FullStory, Hotjar, or Amplitude's session replay). Key observations:
1. After upload, customers see all 80+ frame styles simultaneously with no guidance. Many scroll briefly and leave.
2. On mobile, the style grid is 2-column and the frame images are too small to evaluate.
3. There's no visual connection between their photo and the frame options — they can't see how their photo would look in each frame.

---

**Step 4: Hypotheses**

1. **Decision paralysis** — 80+ options with no guidance is overwhelming (Barry Schwartz paradox of choice).
2. **No immediate visual feedback** — customers can't see a preview of their photo in the frame until they click through to a detail view.
3. **Mobile rendering** — frames are too small to evaluate on a phone screen.

---

**Step 5: Proposed experiments (run in sequence)**

| Experiment | Hypothesis | Expected lift | Risk |
|---|---|---|---|
| Show photo-in-frame preview on hover/tap | Reduces uncertainty by giving instant visual feedback | +8–12% on mobile | Medium — requires product work |
| Reduce initial choice set to "Top 10 for your photo" (personalized) | Reduces decision paralysis | +5–10% on mobile | High — needs recommendation model |
| Add style filter chips ("Modern", "Classic", "Minimal") at top | Helps customers self-select a category | +4–8% | Low — easy to ship |
| Show "Staff picks for this photo type" curated row | Gives expert guidance, reduces decision anxiety | +3–6% | Low — editorial curation |

**Decision:** Start with the filter chips (low risk, fast to ship, directly addresses the overwhelming-choice problem) and the photo preview on tap (medium risk but high potential, test on mobile only first).

---

## 8. Mental Models & Heuristics

### 8.1 The Metric Tree (Input Metrics Tree)

The metric tree decomposes any outcome metric into controllable inputs. It's the primary tool for translating strategy into measurement.

**Framebridge Revenue Metric Tree:**

```
Annual Revenue
├── New customer revenue
│   ├── Website visits
│   │   ├── Paid channel spend × CPC
│   │   └── Organic / SEO / referral
│   ├── Configurator start rate (visits → config starts)
│   │   ├── Homepage → product page CTR
│   │   └── Bounce rate
│   ├── Upload completion rate (config starts → upload complete)
│   ├── Configuration completion rate (upload → frame selected + size confirmed)
│   ├── Checkout conversion rate (config complete → order)
│   └── Average Order Value
│       ├── Frame style tier (premium vs. budget)
│       ├── Mat add-on attach rate
│       └── Multi-item order rate
└── Repeat customer revenue
    ├── Repeat purchase rate (1st order → 2nd within 12 months)
    ├── Orders per repeat customer per year
    └── AOV of repeat orders
```

Every PM-owned initiative should connect to at least one node in this tree. If it doesn't, ask why you're doing it.

---

### 8.2 Opportunity Sizing

Before building anything, size the opportunity. This prevents spending 3 sprints on a feature that could move revenue 0.1%.

**Opportunity Sizing Formula:**

```
Opportunity Value = (Problem Frequency) × (% You Can Fix) × (Value Per Fix)

Example: Improve upload abandonment (20% of upload-starters abandon)
  - Problem Frequency: 10,000 users start uploads per month, 2,000 abandon
  - % You Can Fix: Conservatively fix 30% of abandonments
  - Value Per Fix: If a completer orders at $150 AOV with 35% overall conversion
    → ~600 more converters × 35% × $150 = ~$31,500/month = ~$375K/year
  - This justifies significant engineering investment.
```

---

### 8.3 ROI Thinking

FAANG PMs think in terms of expected value:

```
Expected ROI = (Probability of Success × Gain) - (Probability of Failure × Cost)

A feature that:
  - Has 60% chance of +$500K annual revenue
  - Has 40% chance of $0 lift (no cost to reverse)
  - Costs $200K in engineering time

EV = (0.6 × $500K) - $200K = $100K → Worth doing

A feature that:
  - Has 30% chance of +$500K annual revenue
  - Has 70% chance of $0 lift
  - Costs $300K in engineering time

EV = (0.3 × $500K) - $300K = -$150K → Not worth doing without more validation
```

Always ask: What's the cheapest way to increase my confidence in the probability of success before committing full engineering resources?

---

### 8.4 The ICE / RICE Framework for Prioritization

**RICE Score = (Reach × Impact × Confidence) / Effort**

| Initiative | Reach (users/mo) | Impact (1–10) | Confidence % | Effort (weeks) | RICE Score |
|---|---|---|---|---|---|
| Style filter chips | 8,000 | 6 | 80% | 1 | 38,400 |
| Photo preview on tap | 8,000 | 9 | 60% | 3 | 14,400 |
| Personalized top 5 recs | 10,000 | 9 | 40% | 12 | 3,000 |
| Reorder email personalization | 5,000 | 7 | 70% | 4 | 6,125 |

**Style filter chips wins on RICE** even though photo preview has higher potential impact — it's high-confidence and requires only 1 week of work.

---

### 8.5 Other Key FAANG Mental Models

**North Star + Counter-metrics:** Every improvement metric should have a paired counter-metric to prevent gaming. Faster checkout → counter-metric is order error rate. More recommendations clicked → counter-metric is customer-reported irrelevance.

**The jobs-to-be-done lens:** Ask not "what feature do customers want?" but "what job is the customer hiring this product to do?" Someone who frames a family photo is hiring Framebridge to "preserve a memory with dignity." Someone framing a concert poster is hiring it to "express my identity." These different jobs imply different design priorities.

**Pre-mortem:** Before launching a feature, imagine it's 6 months later and the feature failed catastrophically. Write down the most likely reasons why. Then design to prevent those outcomes.

**Counterfactual thinking:** When analyzing results, always ask "what would have happened if we *hadn't* done this?" This is exactly what an A/B test's control group measures.

---

## 9. Tools & Stack

### 9.1 The Modern Data Stack

```
┌─────────────────────────────────────────────────────────┐
│                    Data Sources                          │
│  Web/App Events   Orders DB   CRM   Ads APIs   Surveys  │
└─────────────────┬───────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────┐
│              Ingestion / ETL                             │
│  Segment (event collection)   Fivetran (DB → warehouse) │
│  Kafka / Kinesis (streaming)                            │
└─────────────────┬───────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────┐
│              Data Warehouse                              │
│  Snowflake / BigQuery / Redshift                        │
└─────────────────┬───────────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────┐
│           Data Transformation (dbt)                      │
│  Raw events → cleaned events → metrics → marts          │
│  Version-controlled SQL models                          │
└──────┬─────────────────────────────────┬────────────────┘
       │                                 │
       ▼                                 ▼
┌──────────────┐                ┌────────────────────────┐
│  BI / Viz    │                │   Product Analytics    │
│  Looker      │                │   Amplitude / Mixpanel │
│  Tableau     │                │   PostHog (open source)│
│  Mode        │                └────────────────────────┘
└──────────────┘
                  
┌─────────────────────────────────────────────────────────┐
│              Experimentation Platform                    │
│  Statsig / Optimizely / Eppo / LaunchDarkly            │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│              ML / DS Platform                            │
│  Notebooks: Jupyter / Databricks / SageMaker           │
│  Feature store: Feast / Tecton / Vertex AI              │
│  ML serving: SageMaker / Vertex / custom FastAPI        │
└─────────────────────────────────────────────────────────┘
```

---

### 9.2 What to Learn vs. What to Delegate

**Learn deeply (PM differentiators):**
- SQL: You should be able to write your own exploratory queries. Non-negotiable.
- Metric definition: You own what metrics mean and how they're defined.
- A/B test design: Hypothesis, sample size, duration, success criteria. Own this.
- Data interpretation: Reading charts, spotting anomalies, asking "is this real?"
- Funnel analysis and cohort analysis in your product analytics tool (Amplitude, Mixpanel).

**Learn conceptually (enough to collaborate):**
- Machine learning fundamentals: What's a classification model vs. regression? What's overfitting? What's a feature?
- Statistics: p-values, confidence intervals, statistical power. Enough to catch bad analysis.
- Data pipelines: High-level understanding of how data flows from app events to your dashboard so you know when something is broken.

**Delegate but review:**
- Complex statistical modeling (survival analysis, causal inference beyond A/B)
- ML model training and evaluation
- Data pipeline engineering
- Infrastructure and schema design

**Never fully delegate:**
- What the metric *means* and whether it's measuring the right thing
- Whether an experiment's success criteria align with business goals
- Whether a model's output is being used correctly in a product surface

---

### 9.3 Tool Recommendations for Framebridge

| Job to be Done | Recommended Tool | Why |
|---|---|---|
| Product analytics / funnels | Amplitude | Best-in-class funnel + cohort analysis; PMs can self-serve |
| A/B experimentation | Statsig or Eppo | Modern, stats-sound, sequential testing; integrates with most stacks |
| Data warehouse | Snowflake | Standard; great for SQL-first teams |
| Data transformation | dbt | Version-controlled SQL models; every PM should understand dbt basics |
| BI dashboards | Looker or Mode | SQL-based, PM-accessible; Looker LookML is worth learning basics |
| Session replay | FullStory or Hotjar | Critical for debugging funnel drop-offs qualitatively |
| Customer surveys | Sprig (in-product) or Typeform | For NPS, CSAT, and product-specific surveys |
| ML platform | SageMaker or Vertex AI | When you're ready to productionize recommendation models |

---

## 10. 30-60-90 Day Learning Plan

### Month 1 (Days 1–30): Build the Foundation

**Goal:** Get fluent with your own data and define a clear measurement baseline.

**Week 1: Instrument and Map**
- Audit your current event tracking. Which events are firing? Which are missing? Write down the top 5 gaps.
- Draw Framebridge's full user journey on paper, from ad click to repeat order.
- Map every step to an existing tracked event (or note where tracking is missing).
- Identify your current North Star metric candidate and pull its 90-day trend.

**Week 2: Funnel and Retention Baseline**
- Run the funnel analysis SQL from Section 4.3 against your real data.
- Which step has the biggest drop-off? Save this as your "opportunity map."
- Run a basic 6-month cohort retention query. What does your retention curve look like?

**Week 3: Metric Tree**
- Build a complete metric tree for Framebridge revenue (like Section 8.1).
- For each leaf node, pull the current value and the YoY trend.
- Identify which nodes are most below benchmark or declining.

**Week 4: First A/B Test Design**
- Pick the highest-RICE opportunity from your analysis.
- Write a complete experiment brief: hypothesis, primary metric, secondary metrics, guardrails, sample size calculation, and duration.
- Get it reviewed by your DS or engineering counterpart.

**Project:** Write a 1-page "State of the Funnel" doc that maps every step from traffic to reorder, with current conversion rates and the top 3 opportunities. Share with your team.

---

### Month 2 (Days 31–60): Go Deeper on Causality

**Goal:** Run your first experiment and develop comfort with causal questions.

**Week 5: A/B Testing Execution**
- Launch the experiment you designed in Week 4. Set up monitoring (daily check of sample size accumulation, guardrail metric health).
- Study Evan Miller's "How Not To Run an A/B Test" — the classic p-hacking article.

**Week 6: Segmentation & Personalization Discovery**
- Run a segmentation analysis on Framebridge customers. Try to identify 3–4 meaningfully different customer segments (by photo type, by acquisition channel, by LTV tier).
- For each segment, answer: Does our current product serve this segment well? What's different about how they behave?

**Week 7: Read and Reflect on DS Fundamentals**
- Read "Trustworthy Online Controlled Experiments" (Kohavi et al.) — chapters 1–6. This is the canonical A/B testing book from Microsoft/Google.
- Read "Thinking in Bets" (Annie Duke) — reframes decision-making under uncertainty in an immediately applicable way.

**Week 8: Evaluate Your Experiment**
- Your experiment should have enough data by now. Analyze results with the framework from Section 3.4.
- Write a decision doc: What did you learn? What are you shipping? What's the follow-on question?

**Project:** Write a "Cohort Deep Dive" doc on your Q4 customer cohort. Characterize them, explain their retention patterns, and recommend 2 specific interventions to improve their 6-month repeat rate.

---

### Month 3 (Days 61–90): Develop DS Collaboration and Prediction

**Goal:** Move from reactive analytics to proactive, predictive product thinking.

**Week 9: Churn and Lifetime Value Modeling**
- Work with your DS team (or on your own in SQL) to identify customers who are "at risk" of never reordering. What behavioral signals predict this?
- Build a simple rule-based early warning system: e.g., "customers who received their order 30+ days ago, haven't opened any email, and have no repeat order" → flag for re-engagement.

**Week 10: Recommendation System Discovery**
- Audit your current recommendation surfaces (if any). What's the CTR and downstream conversion of each?
- Interview 5 customers about how they chose their frame style. What was confusing? What would have helped?
- Write a crisp DS problem statement (as in Section 5.3) for a frame style recommendation system.

**Week 11: Build a Dashboard**
- In Amplitude, Looker, or Mode, build a dashboard that you will review every Monday. It should include: funnel conversion rates, weekly retention, experiment status, and top acquisition channel performance.
- The goal is to develop a weekly data ritual that makes anomalies visible immediately.

**Week 12: Present to Leadership**
- Synthesize your 90 days of work into a "Data-Driven Product Roadmap Proposal."
- For each recommendation, include: the data that motivated it, the experiment or measurement plan, the expected impact, and the investment required.

**Project:** Build and present a prioritized backlog of 5 product opportunities, each sized with RICE, grounded in data from your own analysis, with an experiment design for the top opportunity.

---

### Weekly Exercises (Ongoing)

**Every Monday:** Review your core dashboard. Write 1–3 sentences: "What's different from last week? Is it concerning or expected?"

**Every Wednesday:** Pull one ad-hoc SQL query you've never run before. Ask a question you don't know the answer to. Practice curiosity.

**Every Friday:** Read one article from one of these sources:
- [Towards Data Science](https://towardsdatascience.com) (practical ML/analytics)
- [Reforge](https://www.reforge.com/blog) (product growth frameworks)
- [Mode Analytics Blog](https://mode.com/blog/) (SQL and analytics)
- [Eugene Wei's blog](https://www.eugenewei.com) (FAANG-level product thinking)
- [Experimentation Works](https://experimentationworks.substack.com) (A/B testing depth)

---

### Recommended Books

| Book | Why Read It |
|---|---|
| *Trustworthy Online Controlled Experiments* — Kohavi et al. | The definitive A/B testing book. Chapters 1–8 are mandatory. |
| *Thinking in Bets* — Annie Duke | Decision-making under uncertainty; calibrated confidence |
| *Competing on Analytics* — Davenport & Harris | How leading companies build analytics as a core competency |
| *The Lean Startup* — Eric Ries | Build-measure-learn loop; treating every feature as an experiment |
| *Data-Driven* — D.J. Patil | Short read from former US Chief Data Scientist on data culture |
| *Inspired* — Marty Cagan | The PM bible; grounds everything above in product craft |

---

## Quick Reference: The FAANG PM Data Checklist

Before any product decision, ask:

- [ ] Have I framed a falsifiable hypothesis?
- [ ] Do I know what *type* of question I'm asking (descriptive/causal/predictive)?
- [ ] Is my success metric an *input* metric I can move, or an *output* I'm hoping follows?
- [ ] Have I sized the opportunity before investing engineering time?
- [ ] If I'm running an experiment, have I calculated required sample size and duration?
- [ ] Have I defined guardrail metrics to protect against unintended harms?
- [ ] Have I segmented the data to understand who is affected and how?
- [ ] Am I interpreting correlation as causation anywhere?
- [ ] If I'm working with DS, have I written a crisp problem statement with a defined decision?
- [ ] Can I explain this decision to a skeptical VP using only data?

---

*This guide is a living document. As you level up, you'll find places where the frameworks apply imperfectly to Framebridge's specific context — that gap is where the best product thinking happens. The goal is not to follow these frameworks rigidly but to internalize the underlying logic well enough to adapt them.*
