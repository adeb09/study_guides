# Adonis ML Engineer – Onsite Interview Prep Guide

> High-signal internal prep doc for senior ML candidate. Dense, concrete, no fluff.

---

## Table of Contents

1. [RCM 101](#1-rcm-101)
2. [Claims Lifecycle Deep Dive](#2-claims-lifecycle-deep-dive)
3. [Mapping Adonis to the Workflow](#3-mapping-adonis-to-the-workflow)
4. [Autonomous Agents in RCM](#4-autonomous-agents-in-rcm)
5. [System Design Prep](#5-system-design-prep)
6. [MLOps + Infra Talking Points](#6-mlops--infra-talking-points)
7. [Coding Interview Prep](#7-coding-interview-prep)
8. [Behavioral + Cultural Prep](#8-behavioral--cultural-prep)
9. [Hiring Manager + Leadership Prep](#9-hiring-manager--leadership-prep)
10. [Smart Questions To Ask](#10-smart-questions-to-ask)

---

## 1. RCM 101

### What RCM Is

**Revenue Cycle Management** is the full financial lifecycle of a patient encounter — from the moment a patient schedules an appointment to the moment the provider receives final payment. It bridges clinical operations and financial operations.

Key phases:
```
Patient Registration → Eligibility Verification → Service Delivery →
Charge Capture → Claim Creation (837) → Submission → Adjudication →
Payment (835/ERA) → Posting → Denial Management → Appeals → Collections
```

### Why It Matters Financially

- US healthcare providers collectively leave **~$262B/year** in denied or underpaid claims
- Denial rates average **5–10%** across providers; top quartile gets it to ~3%
- **Cost to rework a denial: $25–118 per claim** depending on complexity
- ~50–65% of initial denials are **never reworked** — pure revenue leakage
- Hospitals run on **1–4% net operating margins** — denials are existential

### Key Stakeholders

| Role | Concern |
|---|---|
| Provider (hospital/clinic) | Get paid for services rendered |
| Payer (insurer) | Adjudicate claims correctly, control spend |
| Clearinghouse (Availity, Change) | Translate + route EDI transactions |
| Patient | Understand liability, pay balance |
| Billing team | Submit clean claims, work denials |
| Compliance/HIM | ICD/CPT accuracy, audit readiness |

### Where Money Is Lost

1. **Front-end failures**: wrong eligibility, missing auth → claim denied before adjudication
2. **Coding errors**: wrong CPT, ICD mismatch, missing modifiers → technical denials
3. **Timely filing**: missed deadlines (payers enforce 90–365 days) → unrecoverable
4. **Underpayments**: payer pays less than contracted rate, nobody catches it
5. **Write-offs**: billing team gives up on reworking — often 35–50% of denials
6. **Duplicate claims**: confusing re-submissions vs. duplicates → more denials
7. **Payer-specific rules**: each payer has unique quirks; teams can't track all of them

---

## 2. Claims Lifecycle Deep Dive

### Step-by-Step Flow

```
[EHR/PMS]
    │
    ▼
[Charge Capture]         ← CPT/ICD codes assigned by coder or CDI
    │
    ▼
[Claim Creation: 837P/I] ← EDI X12 format (professional=837P, institutional=837I)
    │
    ▼
[Clearinghouse]          ← Validates format, checks basic payer rules
    │  ← Returns 999 (functional ack) or 277CA (claim status)
    ▼
[Payer Front-end]        ← Identity, eligibility, duplicate check
    │  ← Returns 277 if rejected here (not yet adjudicated)
    ▼
[Adjudication Engine]    ← Medical necessity, coverage rules, prior auth, fee schedule
    │
    ▼
[835 / ERA]              ← Electronic Remittance Advice — adjudication result
    │  Contains: CAS segments (adjustment reasons), CARC, RARC codes
    ▼
[Payment Posting]        ← Match 835 to 837, post to ledger
    │
    ▼
[Denial Mgmt / Appeals]  ← If CO-16, CO-4, PR-204, etc. → work it
    │
    ▼
[Secondary Billing / Collections / Write-off]
```

### Key Data at Each Step

| Step | Data |
|---|---|
| Claim creation | CPT, ICD-10, NPI, place of service, date of service, subscriber ID |
| Clearinghouse | 999 ack, 277CA status, payer ID routing |
| Payer front-end | Eligibility response (270/271), prior auth (278) |
| 835/ERA | CARC (claim adjustment reason), RARC (remark), paid amount, allowed amount |
| Denial management | Denial reason text, previous submission history, payer correspondence |

### Rejections vs. Denials

| | Rejection | Denial |
|---|---|---|
| When | Before adjudication | After adjudication |
| Why | Format/data error | Medical/policy reason |
| Transaction | 277CA or clearinghouse | 835 with CARC code |
| Fixable | Usually fast, resubmit | Requires appeal or correction |
| Example | Missing NPI, invalid date | Not medically necessary, no auth |

**This distinction matters for Adonis** — rejection signals a data quality problem; denial signals a coverage/policy problem. Different ML models, different interventions.

### Common CARC Denial Codes (Know These)

| CARC | Meaning | What to Do |
|---|---|---|
| CO-4 | Service inconsistent with modifier | Check modifier, recode |
| CO-11 | Diagnosis inconsistent with procedure | CDI review |
| CO-16 | Claim lacks info to adjudicate | Submit requested info |
| CO-50 | Non-covered service | Verify coverage, appeal or write off |
| CO-97 | Benefit included in allowance for another service | Bundling issue, check NCCI edits |
| CO-109 | Claim not covered by this payer/contractor | Wrong payer routing |
| CO-119 | Benefit maximum reached | Exhaust benefits, patient liability |
| PR-1 | Deductible not met | Patient liability, bill patient |
| PR-204 | Service not covered (patient's plan) | Appeal or patient bill |
| OA-23 | Timely filing limit exceeded | Unrecoverable, internal audit needed |

### Revenue Leakage Points

1. **Clearinghouse rejection loop**: claim bounces, sits in queue, timely filing clock ticking
2. **Incorrect posting**: 835 misread, claim looks paid when it's underpaid
3. **No-touch write-offs**: billing team marks as "written off" without appeal because ROI seems low
4. **Underpayment tolerance**: payer pays $480 vs. contracted $500 — $20 ignored per claim × 100K claims = $2M/year
5. **Coordination of benefits (COB)**: wrong primary/secondary payer sequencing
6. **Lag time in denial identification**: batch 835 processing means denial discovered 2 weeks late

### Why This Process Is Still Manual

- **Payer heterogeneity**: 900+ payers, each with unique portals, rules, appeal formats
- **Unstructured documents**: denial letters, EOBs, clinical notes, payer policies as PDFs
- **Legacy systems**: EMRs (Epic, Meditech) have rigid APIs; billing systems (Waystar, Availity) have limited programmatic access
- **Trust/compliance**: healthcare is risk-averse; automating appeals means AI is generating clinical justifications
- **No ground truth**: payer decisions are opaque; what "worked" on appeal is not well-labeled
- **Data silos**: clinical data in EHR, billing in PMS, payer data in clearinghouse — rarely unified

---

## 3. Mapping Adonis to the Workflow

### Where Adonis Sits

```
[Clearinghouse] → [Payer Adjudication] → [835 ingest]
                                              │
                                         [Adonis Platform]
                                         ┌──────────────────────────────┐
                                         │  Denial Pattern Detection    │ ← statistical anomaly
                                         │  Payer Behavior Modeling     │ ← payer-specific rules/patterns
                                         │  Revenue Leakage Alerts      │ ← underpayment detection
                                         │  Recommendation Engine       │ ← [BUILDING NOW]
                                         │  Workflow Automation         │ ← [BUILDING NOW]
                                         └──────────────────────────────┘
                                              │
                                         [Billing Team Actions]
```

Adonis is a **post-adjudication intelligence layer** — ingesting 835s, 837s, CARC/RARC codes, payment data, and EHR context to tell billers what to do next.

### What "Denial Intelligence" Actually Means

It's not just "you had a CO-16 denial." It's:
- **Pattern**: This payer (Cigna) denies CO-16 on CPT 99215 when diagnosis is Z-code 90% of the time
- **Prediction**: This claim has 78% probability of denial before it's submitted
- **Recommendation**: Submit with clinical note excerpt AND modifier 25 → historical approval rate 84%
- **Automation**: Draft the appeal letter using payer-specific language, queue for biller review

### What a Recommendation Engine Looks Like

**Input features** (at claim level):
- CARC/RARC codes from 835
- CPT/ICD-10/modifier combination
- Payer ID + plan type
- Prior authorization status
- Billing provider NPI
- Days since original submission
- Historical denial rate for this CPT × payer × diagnosis combination
- Claim dollar amount
- Prior attempts on this claim

**Output**:
```json
{
  "claim_id": "CLM-9823741",
  "recommended_action": "appeal",
  "confidence": 0.84,
  "rationale": "CO-16 on CPT 99215 with Z23 dx — Cigna requires clinical documentation. Similar claims approved 81% of the time with narrative + medical records attached.",
  "suggested_next_steps": [
    "Attach encounter note dated 2024-03-12",
    "Include modifier 25",
    "Submit via Cigna portal (not clearinghouse) — 12% higher approval rate"
  ],
  "appeal_deadline": "2024-04-28",
  "estimated_recovery": 420.00
}
```

**Concrete scenario examples**:
- `CO-4 on CPT 27447 (knee replacement) → modifier missing → auto-correct and resubmit`
- `PR-204 on CPT 90837 (therapy) → out-of-network → check for gap exception → if patient has referral, appeal`
- `OA-23 (timely filing) → escalate to root cause analysis → which intake step failed?`
- `CO-50 (non-covered) × 200 claims × same payer → payer policy change detected → alert billing director`

---

## 4. Autonomous Agents in RCM

### Where Agents Fit

#### Pre-Claim vs. Post-Claim — The Adonis Reality Check

Pre-claim automation is real but mostly **not Adonis's primary surface area**. Here's why:

- Adonis ingests 835s, CARC/RARC codes, and remittance data — all post-submission artifacts
- Their data moat is in **what payers actually do**, not what payer policy documents say
- Pre-claim systems (eligibility, prior auth, charge capture) live inside EHR and PMS systems — Epic, Cerner, Meditech — that Adonis doesn't own
- Their customers already have dedicated tools for eligibility (Waystar, Availity) and prior auth (Cohere, payer-native portals)

The natural pre-claim extension for Adonis is narrow but high-value:

**Denial risk scoring at submission time** — reuses existing 835/CARC pattern data:
```
[837 about to be submitted]
       │
       ▼
[Adonis denial risk scorer]
  Input: CPT × ICD × payer × modifier × prior auth status
  Output: P(denial) = 0.73
  Alert: "73% denial probability — Cigna typically denies CPT 99215
          with Z23 dx. Attach clinical documentation before submitting."
```

**Payer-specific pre-edit recommendations** — learned payer behavior as a smart rule checker:
> "Aetna requires modifier 25 on 99215 when billed same-day as a procedure. You're missing it."

Full pre-claim agents (eligibility, prior auth initiation) require integrating into clinical systems Adonis doesn't control — that's a different product bet entirely.

**If asked about pre-claim in the interview:**
> "Pre-claim is adjacent to Adonis's core but not where their data advantage sits today. The most natural extension is a denial risk scorer at submission time — that directly reuses their 835/CARC pattern data. Full pre-claim agents require EHR and scheduling system integrations that Adonis doesn't own, which is a different product bet."

---

**Pre-claim (where Adonis can play):**
- Denial risk scorer at submission time (natural extension of existing 835 data)
- Payer-specific pre-edit checker (learned payer behavior patterns)

**Pre-claim (outside Adonis's current lane):**
- Eligibility verification agent (lives in clearinghouse/PMS layer)
- Prior auth initiation agent (lives in EHR layer)
- Charge capture audit agent (lives in clinical/CDI layer)

**Post-claim (Adonis's primary surface — where agents matter most):**
- Denial triage agent (classify denial type, prioritize by dollar + recoverability)
- Appeal generation agent (draft appeal with clinical justification)
- Payer portal interaction agent (navigate payer portals, submit appeals)
- Underpayment detection agent (compare 835 paid amount vs. contracted rate)
- Follow-up agent (track outstanding claims, trigger escalations)

### Denial Triage Agent — Detailed Architecture

```
[835 arrives / denial event]
        │
        ▼
[Classifier]
  Input: CARC, RARC, CPT, ICD, payer, claim history
  Output: denial type (technical | medical necessity | coverage | administrative)
        │
        ▼
[Recoverability Scorer]
  Input: denial type, dollar amount, payer appeal rate, days to deadline
  Output: priority score (high/medium/low) + estimated recovery probability
        │
        ▼
[Action Router]
  - High + technical → auto-correct and resubmit (no human needed)
  - High + medical necessity → draft appeal, queue for clinical reviewer
  - Low + timely filing expired → auto write-off with root cause tag
  - Medium + payer portal required → queue for biller with portal instructions
        │
        ▼
[Human Review Queue] ←→ [Audit Trail / Feedback Loop]
```

#### Deep Dive: Is the Classifier ML or LLM?

**Start with a deterministic lookup table. Add ML for edge cases. Add LLM only when unstructured text is present.**

CARC codes are specific enough that 60–70% of denial types can be mapped without any model:

```python
CARC_TO_DENIAL_TYPE = {
    "CO-4":   "technical_coding",
    "CO-11":  "technical_coding",
    "CO-16":  "missing_documentation",
    "CO-50":  "coverage",
    "CO-97":  "bundling",
    "OA-23":  "timely_filing",
    "PR-1":   "patient_liability",
    "PR-204": "coverage",
}

def classify_denial(carc_code, features, fallback_model):
    if carc_code in CARC_TO_DENIAL_TYPE:
        return CARC_TO_DENIAL_TYPE[carc_code]  # deterministic — no model needed
    return fallback_model.predict(features)    # ML for edge cases + combinations
```

The three-layer classifier:

```
LAYER 1: Deterministic lookup (CARC → denial_type)
  Handles ~60–70% of cases
  Zero latency, zero cost, fully auditable

         │ unrecognized CARC or multi-code combination
         ▼

LAYER 2: ML Classifier (XGBoost on CARC+RARC+CPT+ICD+payer)
  Handles edge cases and stacked denial codes
  Fast, interpretable, no PHI risk

         │ unstructured text present (denial letter, billing note)
         ▼

LAYER 3: LLM (Azure OpenAI / Bedrock — BAA-covered)
  Parses free-text denial reasons, billing team notes
  Always outputs structured JSON — never free text
  Example output:
    {
      "denial_type":      "missing_documentation",
      "root_cause":       "op report not attached",
      "required_action":  "resubmit_with_attachment",
      "required_docs":    ["operative_report"],
      "payer_ref_number": "8823771",
      "confidence":       0.91
    }
```

**Why not just use an LLM for everything?** LLMs add latency, cost, and hallucination risk for a task that doesn't need language understanding. CARC codes are structured by design — don't bring a language model to a lookup table fight.

**Interview answer:**
> "I'd start with a deterministic CARC lookup — no model at all for 60–70% of cases. XGBoost handles the edge cases. The LLM only enters when there's unstructured text like denial letters or billing notes, and even then it outputs structured JSON, not free text. Simplest tool that solves the problem."

---

#### Deep Dive: Is the Recoverability Scorer ML or LLM?

**It's a calibrated ML model — not an LLM. Always.**

The recoverability scorer answers a precise numerical question: what is the probability this denied claim gets paid, and how much will we recover? This is a regression/classification problem on structured data. LLMs are not designed for this:

- LLMs can't output well-calibrated probabilities from tabular features
- LLM inference is 10–100x slower than a tabular model
- "The LLM said 0.81" is not an acceptable audit trail for a financial decision
- Scoring thousands of claims daily with an LLM is prohibitively expensive

**Two-model architecture:**

```python
import xgboost as xgb
from sklearn.calibration import CalibratedClassifierCV

# Model 1: Will it recover? (binary classifier → calibrated)
recovery_clf = xgb.XGBClassifier(n_estimators=500, max_depth=6)
recovery_clf.fit(X_train, y_recovered_binary)

calibrated_clf = CalibratedClassifierCV(recovery_clf, method="isotonic", cv="prefit")
calibrated_clf.fit(X_cal, y_cal)

# Model 2: How much? (regressor on recovered claims only)
recovered_mask = y_recovered_binary == 1
recovery_reg = xgb.XGBRegressor(n_estimators=500, max_depth=6)
recovery_reg.fit(X_train[recovered_mask], y_amount[recovered_mask])

# At inference:
p_recovery   = calibrated_clf.predict_proba(X)[0][1]   # e.g. 0.81
expected_amt = recovery_reg.predict(X)[0]               # e.g. $387
expected_val = p_recovery * expected_amt                 # e.g. $313.47
```

**Where LLM touches the scorer — as a feature extractor only:**

```
[Billing team note (unstructured)]
  "Called Cigna, rep confirmed clinical review needed,
   op report will resolve. Call ref# 8823771"
          │
          ▼
  [LLM extracts structured features]
    payer_confirmed_fixable = True
    required_doc_available  = True
    prior_payer_contact     = True
          │
          ▼ (combined with structured claim features)
  [XGBoost Recoverability Scorer]
    → P(recovery) = 0.88  (enriched by LLM-extracted signal)
```

The LLM is a **feature extractor upstream**, not a scorer. The ML model does the scoring.

**SHAP for explainability — no LLM needed:**

```python
import shap

explainer   = shap.TreeExplainer(recovery_clf)
shap_values = explainer.shap_values(X_single_claim)

# Per-claim output shown to biller:
# carc_cpt_payer_appeal_success_rate: +0.31  (strong positive signal)
# prior_submission_count:             -0.18  (already tried twice)
# submitted_amount:                   +0.12  ($450 — worth working)
# denial_type=technical:              +0.09  (fixable)
```

SHAP output becomes the biller rationale — no language generation needed.

**Top features that drive recoverability scores:**

| Feature | Direction | Why |
|---|---|---|
| `carc_cpt_payer_appeal_success_rate` | ↑ higher = more recoverable | Most direct historical signal |
| `days_to_filing_deadline` | ↑ more time = more recoverable | Optionality + urgency |
| `submitted_amount` | ↑ higher = more recoverable | Worth working harder |
| `prior_submission_count` | ↓ more attempts = less recoverable | Diminishing returns |
| `denial_type = technical` | ↑ = more recoverable | Modifier fix vs. clinical judgment |
| `payer_id` | varies | Payer behavior is strong signal |
| `prior_auth_matches` | ↑ match = more recoverable | Auth issues are hard to reverse |
| `claim_age_days` | ↓ older = less recoverable | Timely filing clock |

**Interview answer:**
> "The recoverability scorer is a calibrated XGBoost model — not an LLM. It's a structured tabular problem: well-defined features, calibrated probability output, needs to be fast and auditable. I'd use SHAP for per-claim explainability, which gives billers a reason for the score without generating any language. The one place an LLM touches this is upstream as a feature extractor — converting unstructured billing notes into structured binary features that feed the scorer. But scoring itself is always the ML model. LLMs output text — I need a calibrated probability."

---

#### Deep Dive: Is the Scorer a Workflow Node or an LLM Tool?

**It's both — depending on who's calling it and why.**

**Tool** = something an agent calls on demand mid-reasoning when it decides it needs a score

**Workflow node** = something that always runs at a fixed point in a predetermined sequence

---

**Pattern 1: Workflow Node (bulk processing pipeline)**

For nightly 835 ingestion where thousands of claims need triage, the scorer runs as a fixed LangGraph node — no LLM decides whether to call it:

```python
from langgraph.graph import StateGraph
from typing import TypedDict

class ClaimState(TypedDict):
    claim_id:           str
    carc_code:          str
    claim_features:     dict
    denial_type:        str
    p_recovery:         float
    expected_value:     float
    recommended_action: str

def classify_denial(state: ClaimState) -> ClaimState:
    state["denial_type"] = carc_lookup(state["carc_code"])
    return state

def score_recoverability(state: ClaimState) -> ClaimState:
    features = build_feature_vector(state["claim_features"])
    state["p_recovery"]    = calibrated_clf.predict_proba(features)[0][1]
    state["expected_value"] = state["p_recovery"] * recovery_reg.predict(features)[0]
    return state

def route_action(state: ClaimState) -> str:
    if state["p_recovery"] > 0.7 and state["denial_type"] == "technical_coding":
        return "auto_resubmit"
    elif state["p_recovery"] > 0.5:
        return "appeals_agent"
    else:
        return "write_off"

graph = StateGraph(ClaimState)
graph.add_node("classify", classify_denial)
graph.add_node("score",    score_recoverability)
graph.add_conditional_edges("score", route_action)
graph.set_entry_point("classify")
graph.add_edge("classify", "score")
```

The scorer is a plain Python function wrapped in a node. The LLM isn't in this part of the flow at all.

---

**Pattern 2: LLM Tool (complex cases, appeals agent)**

For the appeals agent reasoning through a multi-step claim, the scorer is wrapped as a callable tool:

```python
from langchain.tools import tool

@tool
def get_recoverability_score(claim_id: str, proposed_action: str) -> dict:
    """
    Returns recovery probability and expected dollar value for a given
    claim and proposed action. Call this before recommending any action
    to verify it is worth pursuing.
    """
    features = load_claim_features(claim_id)
    features["action"] = proposed_action
    X = build_feature_vector(features)

    p_recovery   = calibrated_clf.predict_proba(X)[0][1]
    expected_val = p_recovery * recovery_reg.predict(X)[0]
    shap_vals    = get_top_shap_features(X, n=3)

    return {
        "p_recovery":     round(p_recovery, 3),
        "expected_value": round(expected_val, 2),
        "top_factors":    shap_vals,  # fed back to LLM for rationale generation
    }
```

The LLM agent reasons with it:

```
Agent: "CO-16 denial on CPT 99215. Should I appeal or resubmit?
        Let me check expected value for each action."

→ calls get_recoverability_score("CLM-001", "appeal_with_clinical_docs")
← {p_recovery: 0.81, expected_value: 364.50}

→ calls get_recoverability_score("CLM-001", "resubmit_with_correction")
← {p_recovery: 0.41, expected_value: 184.50}

Agent: "Appeal has 2x higher expected value. I'll recommend appeal
        and use the SHAP top factors to generate the rationale."
```

---

**When to use which pattern:**

| Scenario | Pattern | Why |
|---|---|---|
| Nightly bulk 835 processing | Workflow node | Deterministic, fast, no LLM needed |
| Simple denial with known CARC | Workflow node | Rules + scorer is sufficient |
| Complex multi-reason denial | LLM tool | Agent needs to reason before scoring |
| Appeals agent deciding whether to draft | LLM tool | LLM needs score to decide next step |
| Comparing multiple action paths | LLM tool | Agent evaluates options dynamically |

---

**The full architecture with both patterns:**

```
┌──────────────────────────────────────────────────────┐
│  BATCH PIPELINE (LangGraph workflow)                  │
│                                                       │
│  [Classify] → [Score] → [Route] → [Execute]          │
│                  ↑                                    │
│          XGBoost scorer (workflow node, runs always)  │
└───────────────────────┬──────────────────────────────┘
                        │ complex claims escalated
┌───────────────────────▼──────────────────────────────┐
│  APPEALS AGENT (LLM with tools)                       │
│                                                       │
│  LLM reasons about claim context                      │
│     ├── Tool: get_recoverability_score()  ← scorer    │
│     ├── Tool: retrieve_clinical_notes()               │
│     ├── Tool: lookup_payer_policy()                   │
│     └── Tool: draft_appeal_letter()                   │
│                                                       │
│  LLM: "score is 0.81, worth appealing"               │
│  → drafts appeal grounded in retrieved docs           │
└──────────────────────────────────────────────────────┘
```

The same scorer model is used in both places — exposed as a node or a tool depending on the caller.

**Interview answer:**
> "The recoverability scorer fits both patterns in the same system. In the bulk pipeline it's a LangGraph workflow node — runs deterministically on every claim as part of a fixed sequence, no LLM involved. For complex cases escalated to the appeals agent, the same scorer is wrapped as a tool the LLM calls mid-reasoning to compare expected value across actions before deciding which to pursue. The underlying model is identical either way — it's just exposed differently depending on who's calling it and why."

### Appeals Generation Agent — Detailed Architecture

```
[Denial context: CARC, claim data, EHR notes, payer policy]
        │
        ▼
[Document Retrieval Tool]
  - Pull relevant clinical notes from EHR (via FHIR API or Snowflake)
  - Pull payer-specific appeal requirements (policy docs, prompt engineering)
  - Pull historical successful appeal templates for this CARC × payer
        │
        ▼
[LLM Appeal Drafter]
  - Structured prompt: denial reason + clinical context + required elements
  - Output: draft appeal letter in payer-specific format
  - Hallucination guard: clinical facts grounded to retrieved documents only
        │
        ▼
[Compliance Checker]
  - Verify all clinical statements are sourced (RAG with citations)
  - Check for PHI handling, payer-specific formatting rules
        │
        ▼
[Human Review → Submit]
```

**Critical**: LLM must cite source documents (specific note date, author, encounter ID). Never generate clinical facts. RAG is not optional here.

### Payer Interaction Agent

Most payer APIs are limited. Real-world interaction means:
- **Portal automation** (Selenium/Playwright with LLM-guided navigation)
- **Phone IVR automation** (voice AI for payer auth status checks)
- **Fax automation** (yes, fax — ~40% of payer communication is still fax)
- **API integration** where available (Availity, Change Healthcare APIs)

Architecture:
```
[Agent Controller]
    ├── Payer Portal Tool (Playwright-based RPA)
    ├── API Client Tool (Availity/Change APIs)
    ├── Fax Submission Tool
    ├── Status Check Tool
    └── Memory: payer-specific interaction history, portal credentials, success patterns
```

### Agent Architecture Patterns for RCM

**Tools** (what agents can call):
- `check_eligibility(member_id, payer_id, date_of_service)`
- `get_claim_status(claim_id)`
- `retrieve_clinical_note(encounter_id)`
- `lookup_payer_policy(payer_id, cpt_code)`
- `draft_appeal(denial_context, clinical_docs)`
- `submit_appeal(appeal_doc, payer_id, submission_channel)`
- `calculate_appeal_deadline(denial_date, payer_id)`
- `get_historical_appeal_outcomes(payer_id, carc_code, cpt_code)`

**Memory types**:
- **Episodic**: this claim's full history (submissions, denials, appeals, outcomes)
- **Semantic**: payer-specific rules, appeal templates, contract rates
- **Procedural**: step-by-step workflows per denial type

**Orchestration**:
- LangGraph or similar for stateful multi-step workflows
- Each denial is a "case" with state: `[new → triaged → in_appeal → resolved]`
- Human checkpoints at high-risk steps (clinical fact assertion, dollar thresholds)

### Human-in-the-Loop vs. Full Autonomy

| Scenario | Autonomy Level | Rationale |
|---|---|---|
| Resubmit with fixed modifier | Full auto | Deterministic fix, no clinical judgment |
| Eligibility check | Full auto | API call, no interpretation |
| Draft appeal letter | Human review before send | LLM output needs clinical sign-off |
| Write off claim | Human approval | Financial decision, compliance audit |
| Navigate payer portal | Supervised auto | High failure rate, legal risk |
| Prior auth initiation | Human initiates, AI prepares | Payer relationship risk |

**Rule of thumb**: automate data retrieval and document preparation; require human approval for decisions with financial or clinical consequence.

### Risks and Mitigations

| Risk | Mitigation |
|---|---|
| **Hallucinated clinical facts in appeals** | RAG with strict grounding; no generation without citation |
| **HIPAA/PHI exposure** | Data never leaves VPC; audit logs for all LLM calls |
| **Wrong action on high-value claim** | Dollar threshold gates; human review for claims >$X |
| **Payer portal ToS violation** | Legal review; prefer API where available; RPA as fallback |
| **Model confidence miscalibration** | Calibration metrics; uncertainty-aware recommendations |
| **Feedback loop poisoning** | Separate train/eval; don't retrain on auto-approved actions only |
| **Audit trail gaps** | Every agent action logged with timestamp, inputs, outputs, model version |

---

## 5. System Design Prep

### Problem: "Design a system to reduce claim denials"

#### Architecture (Text Diagram)

```
┌─────────────────────────────────────────────────────────────────┐
│                        DATA INGESTION LAYER                      │
│  [EHR FHIR API] [837 EDI] [835 ERA] [Payer APIs] [Billing Notes]│
│           │            │         │          │            │       │
│           └────────────┴─────────┴──────────┴────────────┘       │
│                              │                                    │
│                    [Kafka / Event Stream]                         │
│                              │                                    │
└──────────────────────────────┼──────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│                      DATA PLATFORM (Snowflake)                   │
│  Claims Table │ Remittance Table │ Denial History │ Payer Master │
│  Encounter Notes (vector store) │ Appeal Outcomes │ Contracts    │
└──────────────────────────────┬──────────────────────────────────┘
                               │
            ┌──────────────────┼──────────────────┐
            │                  │                  │
┌───────────▼──────┐  ┌────────▼───────┐  ┌──────▼──────────────┐
│  Pre-Claim Layer │  │ Post-Claim AI  │  │  Analytics Layer     │
│                  │  │                │  │                      │
│ Denial Risk      │  │ Denial Triage  │  │ Payer Behavior       │
│ Scorer           │  │ Classifier     │  │ Dashboard            │
│                  │  │                │  │                      │
│ Prior Auth       │  │ Recoverability │  │ Revenue Leakage      │
│ Predictor        │  │ Scorer         │  │ Detection            │
│                  │  │                │  │                      │
│ Clean Claim      │  │ Action Router  │  │ Anomaly Alerts       │
│ Validator        │  │                │  │                      │
└───────────┬──────┘  └────────┬───────┘  └──────────────────────┘
            │                  │
            └──────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────────┐
│                    AGENT LAYER (LangGraph)                       │
│  Denial Triage Agent │ Appeals Gen Agent │ Payer Interaction     │
│                      │                   │ Agent                 │
│  [Tool: EHR Lookup] [Tool: Policy Fetch] [Tool: Portal RPA]     │
│  [Tool: Draft Appeal][Tool: Submit]      [Tool: Status Check]   │
└──────────────────────┬──────────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────────┐
│                    HUMAN REVIEW QUEUE (UI)                       │
│  Priority queue │ Appeal drafts │ Override interface │ Feedback  │
└──────────────────────┬──────────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────────────┐
│                  FEEDBACK + RETRAINING LOOP                      │
│  Outcome labels (approved/denied/partial) → Feature store →     │
│  Model retraining pipeline (weekly/monthly cadence)             │
└─────────────────────────────────────────────────────────────────┘
```

#### Data Pipelines

**Batch (nightly)**:
- Ingest 835 files from clearinghouse → parse CARC/RARC → load to Snowflake
- Recompute denial rates by CPT × ICD × payer combinations
- Update payer behavior models
- Generate daily revenue leakage report

**Real-time (event-driven)**:
- 837 submitted → immediate denial risk score
- 835 received → real-time denial triage, alert if high priority
- Appeal submitted → track status, trigger follow-up on no response

#### Model Choices

| Task | Approach | Why |
|---|---|---|
| Denial risk pre-submission | XGBoost / LightGBM on structured features | Tabular, well-labeled, interpretable |
| Denial type classification | Fine-tuned BERT or XGBoost on CARC+narrative | Mix of codes + text |
| Recoverability scoring | Gradient boosting + historical outcomes | Needs calibration |
| Appeal generation | GPT-4 / Claude with RAG | Requires language quality + grounding |
| Payer behavior anomaly | Isolation Forest / statistical baselines | Unsupervised, interpretable |
| CPT/ICD mismatch | Rule-based NCCI edits + ML override | Deterministic rules first |

**Guideline**: use rules first (NCCI edits, LCD/NCD policies), ML second (pattern capture), LLM last (language generation and unstructured reasoning). Never use LLM where a rule suffices.

#### Feedback Loops

1. **Explicit**: biller overrides recommendation → label action taken + outcome
2. **Implicit**: claim ultimately paid → trace back to which intervention worked
3. **Appeal outcomes**: payer approves/denies appeal → label the appeal strategy
4. **A/B testing**: route % of claims to alternative recommendation → measure net recovery

#### Evaluation Metrics

**Business metrics (primary)**:
- Net collection rate (collected / expected)
- Denial rate (%)
- Appeal success rate (%)
- Days in AR (days outstanding)
- Cost per denial worked ($)
- Revenue recovered per recommendation followed ($)

**Model metrics (secondary)**:
- Denial prediction: AUC-ROC, precision@k (top K claims to work)
- Classification: F1 per denial category (especially high-value categories)
- Appeal recommendation: approval rate when recommendation followed vs. not
- Calibration: predicted probability vs. actual outcome rate

---

## 6. MLOps + Infra Talking Points

### Deploying ML in Healthcare

**Compliance non-negotiables**:
- HIPAA: PHI must stay within BAA-covered infrastructure (not raw OpenAI API without BAA)
- SOC 2 Type II: audit logs, access controls, encryption at rest + in transit
- Data lineage: know exactly what data trained which model (for audits)
- Model versioning: every prediction traceable to model version + input features

**Infra stack that works well**:
```
Training: Snowflake feature views → Spark/dbt → SageMaker or Vertex AI
Serving:  FastAPI + Docker on ECS/EKS (low-latency scoring endpoint)
Feature store: Feast or Snowflake feature store (batch + online)
Orchestration: Airflow (batch), Kafka (real-time triggers)
LLM: Azure OpenAI or Bedrock (BAA-covered) → never raw OpenAI in prod
Monitoring: Evidently AI, Arize, or custom Snowflake-based drift detection
```

### Monitoring, Drift, Retraining

**Types of drift in RCM**:
- **Payer behavior drift**: Cigna changes adjudication rules → denial patterns shift
- **Coding drift**: ICD-11 transition, CMS annual code updates → feature distribution shift
- **Seasonal drift**: deductible resets in January → PR-1 denials spike
- **Provider drift**: new physician joins → coding patterns change

**Monitoring approach**:
- Track feature distributions weekly (PSI for numerical, chi-square for categorical)
- Track prediction distribution (sudden shift in denial probability scores = drift signal)
- Track business outcomes (denial rate spike = real-world signal)
- Shadow mode: new model runs alongside old, compare predictions before promotion

**Retraining triggers**:
- Calendar-based (monthly — consistent, predictable)
- Performance-based (business metric drops 5% below baseline)
- Event-based (CMS annual code update in October)

### Handling Structured + Unstructured Data

**Structured (Snowflake)**:
- Claims: `claim_id, cpt_code, icd_codes[], payer_id, dos, submitted_amount, ...`
- Remittance: `claim_id, carc_code, rarc_code, paid_amount, adjustment_amount, ...`
- Feature engineering: denial rate by CPT×payer last 90 days, rolling appeal success rate

**Unstructured (vector store)**:
- Clinical notes, payer policy PDFs, appeal letters, billing team notes
- Chunked → embedded → stored in pgvector or Pinecone
- Retrieved at inference time for RAG-augmented LLM decisions
- Index refreshed nightly (new notes) + on demand (new payer policies)

**Integration pattern**:
- At scoring time: pull structured features from online feature store + retrieve top-K relevant documents from vector store → combine as LLM context

### Working with Data Platform Teams

Talking points that land well:
- "I treat the feature store as a contract — I define what I need, data eng owns delivery"
- "I instrument my own data quality checks so I'm not blocked by upstream issues"
- "I prefer dbt models for features that billing can also use in dashboards — single source of truth"
- "When data changes, I want automatic alerting before it hits model performance"
- Schema evolution strategy: versioned feature tables, model pinned to feature version

---

## 7. Coding Interview Prep

### Core Pattern

Based on the constraint (string processing + hashmap, no nested loops), the dominant pattern is:

**Single-pass frequency counting / sliding window / prefix tracking using dicts.**

Time complexity target: O(n) or O(n log n). No O(n²).

---

### Problem 1: Denial Code Frequency Counter

> Given a list of CARC denial codes from 835 remittance records, return the top K most frequent denial codes and their counts.

```python
from collections import Counter
import heapq

def top_k_denial_codes(denial_codes: list[str], k: int) -> list[tuple[str, int]]:
    # O(n) count, O(n log k) heap selection
    counts = Counter(denial_codes)
    return heapq.nlargest(k, counts.items(), key=lambda x: x[1])

# Example:
codes = ["CO-16", "CO-4", "CO-16", "PR-1", "CO-16", "CO-4", "OA-23"]
print(top_k_denial_codes(codes, 2))
# [('CO-16', 3), ('CO-4', 2)]
```

**Pattern**: `Counter` + `heapq.nlargest` — O(n + n log k). Never sort the full list.

---

### Problem 2: First Non-Repeated Denial Code

> Given a sequence of denial codes processed in order, find the first denial code that appeared exactly once.

```python
def first_unique_denial(codes: list[str]) -> str | None:
    # O(n) time, O(n) space
    counts = {}
    order = []
    for code in codes:
        counts[code] = counts.get(code, 0) + 1
        if code not in order:  # use set for O(1) lookup
            order.append(code)
    
    for code in order:
        if counts[code] == 1:
            return code
    return None

# Optimized: use OrderedDict for O(1) ordered traversal
from collections import OrderedDict

def first_unique_denial_v2(codes: list[str]) -> str | None:
    counts = OrderedDict()
    for code in codes:
        counts[code] = counts.get(code, 0) + 1
    for code, count in counts.items():
        if count == 1:
            return code
    return None
```

---

### Problem 3: Claim Grouping by Payer + Denial Type

> Given a list of claim records as strings in format `"claim_id|payer_id|carc_code"`, group claims by payer and return the count of each denial type per payer.

```python
from collections import defaultdict

def group_denials_by_payer(records: list[str]) -> dict[str, dict[str, int]]:
    # O(n) — single pass, no nested loops
    result = defaultdict(lambda: defaultdict(int))
    
    for record in records:
        parts = record.split("|")
        if len(parts) != 3:
            continue
        claim_id, payer_id, carc_code = parts
        result[payer_id][carc_code] += 1
    
    return dict(result)

records = [
    "CLM001|CIGNA|CO-16",
    "CLM002|CIGNA|CO-4",
    "CLM003|AETNA|CO-16",
    "CLM004|CIGNA|CO-16",
]
print(group_denials_by_payer(records))
# {'CIGNA': {'CO-16': 2, 'CO-4': 1}, 'AETNA': {'CO-16': 1}}
```

**Pattern**: `defaultdict(lambda: defaultdict(int))` for nested grouping without nested loops.

---

### Problem 4: Sliding Window — Denial Spike Detection

> Given a time-ordered list of daily denial counts, find any window of K consecutive days where total denials exceeded a threshold. Return the starting indices.

```python
def find_denial_spikes(daily_counts: list[int], k: int, threshold: int) -> list[int]:
    # Sliding window — O(n), single pass
    if len(daily_counts) < k:
        return []
    
    window_sum = sum(daily_counts[:k])
    result = []
    
    if window_sum > threshold:
        result.append(0)
    
    for i in range(k, len(daily_counts)):
        window_sum += daily_counts[i] - daily_counts[i - k]
        if window_sum > threshold:
            result.append(i - k + 1)
    
    return result

daily = [10, 15, 30, 45, 20, 10, 5, 50, 60]
print(find_denial_spikes(daily, 3, 80))
# Windows: [55, 90, 95, 75, 35, 65, 115] → indices where >80
```

---

### Problem 5: Anagram / Substring in Claim Codes (Sliding Window + Hashmap)

> Given a string of concatenated claim actions and a pattern, find all starting positions in the action string where a permutation of the pattern occurs. (Classic: find all anagrams.)

```python
from collections import Counter

def find_code_permutations(action_log: str, pattern: str) -> list[int]:
    # O(n) sliding window with frequency dict comparison
    p_count = Counter(pattern)
    w_count = Counter(action_log[:len(pattern)])
    result = []
    
    if w_count == p_count:
        result.append(0)
    
    for i in range(len(pattern), len(action_log)):
        # Add new char
        new_char = action_log[i]
        w_count[new_char] += 1
        
        # Remove old char
        old_char = action_log[i - len(pattern)]
        w_count[old_char] -= 1
        if w_count[old_char] == 0:
            del w_count[old_char]
        
        if w_count == p_count:
            result.append(i - len(pattern) + 1)
    
    return result
```

---

### Optimization Strategies to Know Cold

1. **Replace O(n²) with hashmap**: if you're searching for something in a list inside a loop, precompute it into a dict
2. **Replace sort with Counter+heap**: for top-K, never sort full array — use `heapq.nlargest`
3. **Sliding window**: for any contiguous subarray problem, maintain a running sum/count
4. **Two pointers**: for sorted array problems (avoid nested loops)
5. **Prefix sums**: for range sum queries — precompute cumulative sums, query in O(1)
6. **Early termination**: once you find what you need, return immediately
7. **Generator over list**: if you only need to iterate once, use generators to save memory

---

## 8. Behavioral + Cultural Prep

Use the **STAR-I format**: Situation → Task → Action → Result → **Impact** (what changed long-term).

### Story Bank (7 Core Stories)

---

**Story 1: Ambiguity → Shipped ML System**
*"Tell me about a time you had to build something from scratch with unclear requirements."*

Frame: You inherited a vague mandate ("improve X metric"). You drove requirements gathering with stakeholders, defined success metrics before writing code, built iteratively with checkpoints, shipped v1 with caveats, iterated.

Key beats:
- How you converted ambiguity to a concrete problem statement
- How you communicated tradeoffs (precision vs. recall, latency vs. accuracy)
- What you shipped, what you deferred, why

#### Polished Answer

**1 — Situation:** At fuboTV, I was asked to be the tech lead for a greenfield project to build out our ML infrastructure. We wanted to replace our legacy recommendation service — which was doing live cosine-similarity compute for live-TV recommendations — with a fully precomputed model-based system. The goal was a service that would sit in front of all the models the DS team was building and serve live recommendations to any user on the platform.

**2 — Task / Ambiguity:** The project was intentionally open-ended. The main requirement was that the service needed to serve live predictions, and that latency wasn't the primary constraint — the service would never be hit directly by our main recommendation service. It would pull predictions and write them to BigTable, so we targeted ~1 second at most. We didn't have an API contract defined yet, didn't know exactly how the service would be consumed, and we just knew it would be internal. My manager gave me an aggressive timeline and told me to start creating tickets and get to work.

**3 — Action:** I worked backwards from what an MVP in production would look like. Even before touching business logic, I asked: what does this service need just to exist in production? We needed it deployable to our Kubernetes cluster. Since this was going to be the first Python microservice at the company, I didn't have internal tooling to fall back on. So my first focus was getting a basic `/predict` endpoint running end-to-end — Docker setup, CircleCI for K8s deployments, service discovery, logging standards, and Datadog for observability — before writing any business logic. The reasoning: if you build the best endpoint but it only runs on your laptop, it almost doesn't matter. One of the first real decisions I had to make was the language choice. The rest of the company ran on Go, but I made a deliberate call to go with Python — it scales slightly worse, but Python is what both I and the DS team were comfortable with, and I wanted the data scientists to be able to contribute directly to the service. Developer efficiency over raw performance. Once the infra foundation was solid, I brought in the DS team to scope the V0 use case. Together we narrowed it down to acquisition-context cold-start recommendations — something our legacy system couldn't do at all.

**4 — Result:** We had an MVP in production in about 3 months. The first use case ended up being rules-based, not ML-based — which wasn't even on the roadmap at the start. It handled cold-start recommendations for new trial users, and during the AB test we saw a 5% lift in live-TV carousel engagement for first-time users, which is a leading indicator for trial-to-subscriber conversion.

**5 — Impact:** That service became the foundational platform for all ML-based recommendations going forward. The most important early decision wasn't the code itself — it was choosing a tech stack that the full team could contribute to, and building the production scaffolding before the business logic so we could iterate fast once the models were ready.

---

**Story 2: Cross-functional Collaboration**
*"Tell me about a time you worked with non-technical stakeholders."*

Frame: ML recommendation that business stakeholders initially didn't trust. How you built trust — explainability, pilot program, business metrics framing (not just AUC).

Key beats:
- Translating model output to business language
- Building trust through transparency (show your work)
- Aligning on what "success" means before you build

#### Polished Answer

**1 — Situation:** Once the sp-ranking-service was ready for recommending live-TV content, our team received a lot of pushback during rollout and pre-rollout. Stakeholders were concerned the live-TV carousel would look drastically different from the legacy one and throw off users. Product caught real issues during the internal employee rollout — I remember Chetan flagging that he suddenly lost all of his Cricket programs due to a metadata issue from the KG team. Issues like that eroded confidence fast.

**2 — Task:** My task was to make the model's reasoning legible to non-technical stakeholders — product, leadership, and content teams — before we could get sign-off on a full rollout. The only way to build that trust was to give people direct visibility into why certain programs were being ranked higher than others.

**3 — Action:** We built a Streamlit dashboard that let anyone in the company look up their own live-TV carousel, see which features were driving each program's ranking and their relative weights, and compare their carousel across different models and training dates. It gave product and QA the ability to catch issues before anything was promoted to users in production — which was exactly the kind of safety net stakeholders needed to feel comfortable moving forward.

**4 — Result:** This gave product and leadership the confidence to approve a full rollout. They knew that even if something went wrong, they'd be able to catch it quickly with the tool rather than waiting for user complaints or metric drops.

**5 — Impact:** Now 100% of live-TV carousels are powered by ML recommendations from the sp-ranking-service. Early AB tests showed a 10% increase in viewership genre diversity while overall viewership stayed constant — meaning users watched the same amount but discovered content they wouldn't have found with our old heuristics-based approach.

---

**Story 3: Production Incident / Model Failure**
*"Tell me about a time something you built failed in production."*

Frame: Concept drift, silent failure, or wrong metric. How you detected it, diagnosed it, fixed it, and prevented recurrence.

Key beats:
- What monitoring was (or wasn't) in place
- Speed of response + communication to stakeholders
- Systemic fix (not just retraining once)

---

**Story 4: Ownership Under Pressure**
*"Tell me about a time you took ownership of something outside your job description."*

Frame: Gap in ownership (data quality issue, missing infrastructure, no one else picking it up). You stepped in, fixed it, documented it, handed it off properly.

Key beats:
- Proactive identification (not just reactive)
- Execution without waiting for permission
- Ensuring knowledge transfer (you don't create single points of failure)

---

**Story 5: Technical Disagreement**
*"Tell me about a time you disagreed with a colleague's technical decision."*

Frame: You advocated for a different approach (simpler model vs. complex, different infra choice). How you made your case, what happened, what you learned.

Key beats:
- Evidence-based argument (not opinion-based)
- Willingness to defer when you lost (disagree and commit)
- What the outcome was and what you'd do the same/differently

---

**Story 6: Shipping Fast vs. Getting It Right**
*"Tell me about a time you had to balance speed and quality."*

Frame: Hard deadline, incomplete data, uncertain model. How you shipped something useful while managing technical debt and stakeholder expectations.

Key beats:
- Explicit tradeoff articulation
- What you deferred and why (not what you ignored)
- How you tracked and eventually paid down the debt

---

**Story 7: Learning a New Domain Quickly**
*"Tell me about a time you had to get up to speed on an unfamiliar domain."*

Frame: You joined a project in a domain you didn't know. How you ramped: who you talked to, what you read, how you built mental models, how domain knowledge changed your ML approach.

Key beats:
- Learning velocity (specific things you did to learn fast)
- How domain knowledge improved the technical work
- What you'd do differently

---

### Cultural Fit Signals Adonis Likely Values

- **Bias for action**: ship, learn, iterate — not perfect upfront
- **Customer obsession**: billing teams are the customer — their workflow drives design
- **Data-driven**: opinions backed by data, hypotheses validated not assumed
- **Ownership**: you don't wait for someone else to fix it
- **Intellectual humility**: willing to be wrong, willing to learn from domain experts

---

## 9. Hiring Manager + Leadership Prep

### Talking to Chirag (HM — ML Systems + MLOps)

What he likely cares about:
- Can you build production ML systems (not just research prototypes)?
- Do you understand the full lifecycle — data → model → serving → monitoring → retraining?
- Can you work across teams (data eng, product, clinical)?
- Are you pragmatic about model choices (simple when simple is right)?

**Frame your experience** around:
- "I've shipped X model to production, handling Y scale, monitored by Z metrics"
- "When the model degraded because of [drift/data change], I [detected it via / fixed it by]"
- "I chose [XGBoost over deep learning] because [latency, interpretability, data volume]"

**Relevant talking points**:
- Feature store design and the trade-offs of batch vs. online features
- How you handle data quality issues upstream of model training
- How you make deploy/no-deploy decisions (shadow mode, canary releases)
- How you work with data platform teams (schema contracts, SLAs)

---

### Talking to Manoj (Head of Eng — Impact, Ownership, Fit)

What he cares about:
- Will you move fast and own outcomes, not just tasks?
- Do you think in terms of business impact, not model metrics?
- Can you operate at a senior level — make decisions, manage ambiguity, communicate up?
- Are you a force multiplier for the team, not just an individual contributor?

**How to frame your ML experience for RCM**:

"My background is in [your domain]. What connects to what Adonis is doing: I've built recommendation and prediction systems on top of complex transactional data where the cost of a wrong recommendation is high — which maps directly to claim adjudication. The challenge is always the same: you have noisy labels, distribution shift, and domain experts who need to trust the system before they'll use it. That's the problem I find most interesting."

**Impact framing**:
- Don't say: "I improved AUC from 0.82 to 0.87"
- Say: "That model improvement translated to $X in recovered revenue / Y hours saved per biller / Z% reduction in manual review"

**Tradeoffs to demonstrate**:
- "We could have used a more complex model but chose XGBoost because billers needed to understand why a claim was flagged — interpretability was a feature, not a limitation"
- "We delayed the v2 feature to instrument better monitoring for v1 — technical debt would have compounded otherwise"

**System thinking**:
- Show you think end-to-end: "The model is 20% of the problem. The bigger challenge is the feedback loop — getting outcome labels back into training, handling selection bias because billers only rework claims they think are recoverable"

---

### Talking to Gerardo/Qian/Tudor (System Design — MLOps + Collaboration)

What they care about:
- Can you design at scale with real constraints?
- Do you think about failure modes proactively?
- Can you collaborate on design — accept input, push back with reasoning?

**Tactics**:
- Start with requirements clarification: "Before I dive in, can I ask — are we optimizing for latency, accuracy, or coverage? What's the scale in claims/day?"
- Explicitly call out tradeoffs: "We could process in real-time but batch would be 10x cheaper and we only need next-day turnaround for most denial types"
- Name failure modes: "The risk here is feedback loop bias — if we only label claims that billers rework, we have selection bias in our training set"

---

## 10. Smart Questions To Ask

### For Chirag (Hiring Manager) — ML Systems + Roadmap

1. "What does the current ML stack look like end-to-end — from feature computation to model serving? What's the biggest gap you're trying to fill?"
2. "You mentioned moving from statistical anomaly detection to an active recommendation engine. What's the biggest technical blocker to that transition today — is it data quality, labels, latency, or trust?"
3. "How do you handle payer behavior drift? When a payer changes their adjudication rules, how quickly does the system adapt?"

### For Gerardo/Qian/Tudor (System Design Peers)

4. "What does your feature pipeline look like — are you computing claim-level features at submission time, or only after the 835 comes back?"
5. "How are you handling the feedback loop problem — specifically the selection bias that comes from billers only reworking claims they think are worth it?"
6. "Do you have a vector store for clinical notes and payer policy documents, or is the LLM work primarily on structured claim data today?"

### For Maddie/Joanna (Cultural — Team + Growth)

7. "What does a great first 90 days look like for this role? How does the team measure ramp-up success?"
8. "How does the ML team interact with the billing domain experts — is there an embedded feedback loop, or is it more arm's-length?"

### For Manoj (Head of Eng — Strategy + Fit)

9. "Where do you see the biggest unlock for Adonis over the next 12–18 months — is it improving the intelligence layer, expanding data partnerships, or going deeper into autonomous workflows?"
10. "How do you think about the human-in-the-loop question for autonomous agent workflows — what's the threshold for full automation vs. human review in your customers' minds?"
11. "What's the hardest data quality problem you've encountered — is it the structured claims data or the unstructured payer policy / clinical note side?"

### For Anyone — Product + Market

12. "Are most of your customers mid-size regional health systems, or do you have a different target segment today?"
13. "How are customers thinking about the compliance and auditability requirements for AI-generated appeal letters — is that a blocker or a solved problem?"

---

## Quick Reference: Vocab to Use Naturally

| Domain Term | Use it when |
|---|---|
| 837/835 | Discussing claim submission / remittance |
| CARC / RARC | Denial reasons from 835 |
| Adjudication | Payer's process of evaluating a claim |
| Prior auth / PA | Pre-approval required before service |
| Timely filing | Deadline to submit a claim to a payer |
| COB | Coordination of benefits (primary/secondary payer) |
| LCD/NCD | Coverage rules (Local/National Coverage Determinations) |
| NCCI edits | CMS bundling rules (Correct Coding Initiative) |
| EOB / ERA | Explanation of Benefits / Electronic Remittance Advice |
| Net collection rate | % of expected revenue actually collected |
| Days in AR | How long claims sit outstanding |
| Clearinghouse | Intermediary that routes/validates EDI transactions |
| Clean claim rate | % of claims that pass without rejection/denial |
| Denial rate | % of claims denied after adjudication |
| Write-off | Revenue abandoned without collection attempt |

---

## Day-of Checklist

**Before each interview**:
- [ ] Know the interviewer's role (look them up on LinkedIn if possible)
- [ ] Have 2–3 specific questions ready for each person
- [ ] Have your 3 highest-impact stories ready (Ambiguity, Failure, Ownership)
- [ ] Be ready to sketch the system design on a whiteboard/doc

**During system design**:
- [ ] Clarify requirements first (always)
- [ ] State assumptions out loud
- [ ] Name tradeoffs explicitly
- [ ] Call out failure modes proactively
- [ ] Connect every design decision to a business outcome

**Key phrases that land well**:
- "The model is the easy part — the hard part is the feedback loop"
- "I'd start with the simplest thing that works, then add complexity with evidence"
- "Interpretability isn't a nice-to-have here — billers need to trust and override the system"
- "We need to be careful about selection bias in our labels — we only see outcomes for claims that were worked"

---

*Good luck. You know this stuff. Own the room.*

---

## 11. Production-Grade Recommender System for Adonis

> Grounded in their actual current state from the candidate brief.

### What They Have Today (The Real Starting Point)

```
✅ Strong structured data in Snowflake (837 + 835)
✅ Statistical anomaly detection (denial alerts)
✅ Some rule-based recommendations
🔬 LLM experimentation (not in production)
❌ No ML recommendation engine
```

This is a good foundation. The mistake most teams make is throwing away the rules and anomaly detection when they "go ML." The right architecture **layers on top of what exists** — it doesn't replace it.

---

### The Core Problem

A **ranked action recommendation** task:

> Given a claim in a known state (just denied, pending, resubmitted), what is the best action to take next, and why?

**Possible actions the system recommends:**
- Resubmit with specific correction (modifier, diagnosis, field)
- Appeal with clinical documentation
- Appeal with specific attachment type
- Escalate to clinical reviewer
- Correct coordination of benefits (COB)
- Bill patient (move to patient liability)
- Write off (low recoverability, not worth working)
- Escalate for root cause analysis (e.g., timely filing expired — why?)

---

### Three-Layer Architecture (Builds on Current State)

```
┌──────────────────────────────────────────────────────────┐
│  LAYER 1: RULES ENGINE  (they already have this)         │
│                                                          │
│  Hard deterministic cases — fast, interpretable, safe    │
│                                                          │
│  OA-23 → cannot recover → route to write-off + RCA      │
│  CO-4 + missing modifier → auto-correct → resubmit      │
│  CO-109 → wrong payer → re-route claim                  │
│  PR-1 → deductible → bill patient                       │
│                                                          │
│  ~30-40% of denials handled here, no ML needed          │
└──────────────────────────┬───────────────────────────────┘
                           │ remaining ~60-70%
┌──────────────────────────▼───────────────────────────────┐
│  LAYER 2: ML RECOMMENDATION ENGINE  (what to build)      │
│                                                          │
│  Input: structured features from Snowflake               │
│    - CARC / RARC codes                                   │
│    - CPT × ICD × payer combination                       │
│    - Historical denial rate for this combination         │
│    - Historical appeal success rate for this payer       │
│    - Days since submission / days to deadline            │
│    - Claim dollar amount                                 │
│    - Prior submission attempts                           │
│    - Provider NPI-level denial patterns                  │
│                                                          │
│  Model: XGBoost / LightGBM                               │
│    → P(recovery | action) for each possible action       │
│    → Rank actions by expected recovery value             │
│    → Calibrated probabilities (not just labels)          │
│                                                          │
│  Output: top recommended action + confidence + deadline  │
└──────────────────────────┬───────────────────────────────┘
                           │ for actions requiring language
┌──────────────────────────▼───────────────────────────────┐
│  LAYER 3: LLM AUGMENTATION  (they're experimenting)      │
│                                                          │
│  Only activates when action = "appeal" or "escalate"     │
│                                                          │
│  RAG pipeline:                                           │
│    - Retrieve: relevant clinical notes (EHR via FHIR)   │
│    - Retrieve: payer-specific appeal requirements        │
│    - Retrieve: successful appeal templates (same CARC)   │
│                                                          │
│  LLM (Azure OpenAI — BAA-covered):                      │
│    - Generate appeal letter draft                        │
│    - Generate human-readable rationale                   │
│    - Ground all clinical facts to retrieved documents    │
│                                                          │
│  Human review gate before any LLM output is sent        │
└──────────────────────────────────────────────────────────┘
```

---

### The Feature Pipeline (Snowflake-Native)

Since their data is already in Snowflake, this is straightforward:

```sql
-- Denial rate by CPT × payer × CARC (last 90 days)
-- Becomes a core feature fed into the ML model

SELECT
    cpt_code,
    payer_id,
    carc_code,
    COUNT(*) AS denial_count,
    AVG(CASE WHEN appeal_outcome = 'approved' THEN 1 ELSE 0 END) AS appeal_success_rate,
    AVG(paid_amount) AS avg_recovery,
    AVG(DATEDIFF(day, denial_date, resolution_date)) AS avg_resolution_days
FROM claims_denials
WHERE denial_date >= DATEADD(day, -90, CURRENT_DATE)
GROUP BY 1, 2, 3
```

These aggregated features become the **payer behavior signal** — the core IP Adonis is building. Over time this is a defensible data moat: they know Cigna's approval rate for CO-16 × CPT 99215 better than Cigna's own billing team does.

---

### What "Production Grade" Actually Requires (Beyond the Model)

**1. Calibrated probabilities, not just rankings**
Don't output "high/medium/low priority." Output `P(recovery) = 0.81, estimated recovery = $420`. This lets billers make an explicit ROI decision: "20 min of work for $420 at 81% odds — yes." Calibration is a first-class concern, not an afterthought.

**2. Explainability as a feature**
Billers won't follow a black-box recommendation. Every output needs a plain-English rationale:
> *"Cigna approves 81% of CO-16 appeals on CPT 99215 when clinical documentation is attached. 47 similar claims resolved this way in the last 90 days. Average recovery: $420."*

**3. The feedback loop — the hardest part**
This is where most RCM recommendation systems fail. The challenge:
- Billers only follow recommendations on claims they *already think are recoverable* — **selection bias**
- You only observe outcomes for claims that were worked — unworked claims are unlabeled
- If you retrain on only biller-approved actions, the model learns biller behavior, not optimal behavior

Design to capture:
- What was recommended
- Whether the biller followed it, overrode it, or ignored it
- What action was ultimately taken
- What the outcome was (paid / denied / partial / written off)

Use **counterfactual reasoning** and **exploration** — route a small % of claims to alternative actions to get unbiased outcome labels.

**4. Phased rollout — don't eat the whole problem at once**
Focus the first version on the top 5 CARC codes by `volume × avg_dollar × recoverability_rate`. Probably CO-16, CO-4, CO-11, CO-50, PR-204. Get those right before expanding. This is also how you A/B test: measure net recovery on claims where the recommendation was followed vs. not.

**5. HITRUST readiness (they mentioned it explicitly in the brief)**
Since they're working toward HITRUST certification, the ML system needs:
- Full audit trail: every prediction logged with model version, input features, output, timestamp
- Data lineage: which training data produced which model
- Access controls: who can see which recommendations, who can override
- No PHI in model logs — log feature hashes or aggregated values, not raw patient data

---

### Phased Build Plan

| Phase | Timeline | What to Build | Success Metric |
|---|---|---|---|
| **0: Foundation** | Weeks 1–4 | Formalize rules engine, build dbt feature models in Snowflake, define action taxonomy | Feature pipeline running, rules covering 30%+ of denial volume |
| **1: Triage + Prioritization** | Months 1–3 | Recoverability scorer, prioritized work queue | Billers work higher-value claims first, net recovery ↑ |
| **2: Action Recommendation** | Months 3–6 | Action classification model, A/B test vs. rules baseline | Appeal success rate ↑ vs. control group |
| **3: LLM Augmentation** | Months 6–12 | RAG rationale generation, appeal draft with human review | Time-to-appeal ↓, biller satisfaction ↑ |
| **4: Feedback Loop** | Ongoing | Outcome capture, model retraining pipeline, drift monitoring | Model doesn't degrade on payer behavior changes |

---

### The One-Paragraph Answer for the Interview

> "I'd build it in three layers on top of what Adonis already has. Layer one keeps the rules engine — it handles the deterministic cases fast and cheap, no ML needed. Layer two is an XGBoost-based recoverability scorer trained on Snowflake's claims and remittance history — it ranks possible actions by expected recovery value with calibrated probabilities, not just labels. Layer three is LLM augmentation for when the recommended action is 'appeal' — RAG over clinical notes and payer policies to generate the rationale and draft, with a human review gate. The hardest part isn't the model, it's the feedback loop — you have to design carefully around the selection bias that comes from billers only working claims they already think are recoverable."

---

### Deep Dive: P(recovery | action) — How It Actually Works

For a single denied claim, you estimate the probability of getting paid **given each possible action**. You compute it for every action, then rank by expected dollar value.

```
Claim CLM-9823741 (CO-16, CPT 99215, Cigna, $450):

P(recovery | resubmit_with_correction)    = 0.41
P(recovery | appeal_with_clinical_docs)   = 0.81  ← best action
P(recovery | appeal_without_docs)         = 0.23
P(recovery | escalate_to_reviewer)        = 0.61
P(recovery | write_off)                   = 0.00  ← by definition

Expected value of each action:
  appeal_with_clinical_docs  → 0.81 × $450 = $364.50  ✅ recommend this
  escalate_to_reviewer       → 0.61 × $450 = $274.50
  resubmit_with_correction   → 0.41 × $450 = $184.50
  appeal_without_docs        → 0.23 × $450 = $103.50
  write_off                  → $0.00
```

#### What the Feature Vector Looks Like

```python
claim_features = {

    # --- Denial context ---
    "carc_code":              "CO-16",      # claim lacks info to adjudicate
    "rarc_code":              "N56",        # missing/incomplete clinical info
    "cpt_code":               "99215",      # office visit, high complexity
    "icd_primary":            "Z23",        # immunization encounter
    "modifier_present":       False,
    "place_of_service":       11,           # office setting
    "claim_type":             "837P",

    # --- Payer context ---
    "payer_id":               "CIGNA_001",
    "plan_type":              "PPO",
    "has_prior_auth":         True,
    "prior_auth_matches":     True,

    # --- Financial context ---
    "submitted_amount":       450.00,
    "expected_contracted":    387.00,       # from fee schedule
    "claim_age_days":         14,
    "days_to_filing_deadline": 76,

    # --- History: this specific claim ---
    "prior_submission_count": 0,
    "prior_appeal_count":     0,

    # --- History: CPT × payer (last 90 days, from Snowflake) ---
    "cpt_payer_denial_rate":          0.31,
    "cpt_payer_appeal_success_rate":  0.81,
    "cpt_payer_avg_resolution_days":  18.4,

    # --- History: CARC × payer (last 90 days) ---
    "carc_payer_denial_count":        142,
    "carc_payer_appeal_success_rate": 0.74,

    # --- History: CARC × CPT × payer (most specific signal) ---
    "carc_cpt_payer_approval_rate":   0.81,
    "carc_cpt_payer_sample_size":     47,

    # --- Provider context ---
    "provider_npi_denial_rate":       0.08,
    "provider_specialty":             "internal_medicine",

    # --- Action being evaluated (varies at inference) ---
    "action":                         "appeal_with_clinical_docs"
}
```

#### How the Model Is Trained

One row per (claim, action taken, outcome) in historical data:

```python
training_data = [
    # claim_id | carc  | cpt   | payer  | ... | action_taken              | recovered
    ("CLM001",  "CO16", "99215","CIGNA", ...,   "appeal_with_clinical_docs", 1),
    ("CLM002",  "CO16", "99215","CIGNA", ...,   "resubmit_with_correction",  1),
    ("CLM003",  "CO16", "99213","AETNA", ...,   "appeal_with_clinical_docs", 0),
    ("CLM004",  "CO4",  "27447","BCBS",  ...,   "resubmit_with_correction",  1),
    ("CLM005",  "CO16", "99215","CIGNA", ...,   "write_off",                 0),
]
```

Action is just another feature. Train one model across all actions:

```python
import xgboost as xgb
from sklearn.calibration import CalibratedClassifierCV

model = xgb.XGBClassifier(n_estimators=500, max_depth=6, learning_rate=0.05)
model.fit(X_train, y_train)

# Calibrate so P=0.81 actually means ~81% of those claims recover
calibrated_model = CalibratedClassifierCV(model, method="isotonic", cv="prefit")
calibrated_model.fit(X_cal, y_cal)
```

#### How Inference Works — One Claim, All Actions

```python
POSSIBLE_ACTIONS = [
    "resubmit_with_correction",
    "appeal_with_clinical_docs",
    "appeal_without_docs",
    "escalate_to_reviewer",
    "write_off",
]

def recommend_action(claim: dict, model, encoder) -> list[dict]:
    results = []
    for action in POSSIBLE_ACTIONS:
        row = {**claim, "action": action}
        X = encoder.transform([list(row.values())])
        p_recovery = model.predict_proba(X)[0][1]
        expected_value = p_recovery * claim["submitted_amount"]
        results.append({
            "action": action,
            "p_recovery": round(p_recovery, 3),
            "expected_value": round(expected_value, 2),
        })
    return sorted(results, key=lambda x: x["expected_value"], reverse=True)
```

#### Three Concrete Examples

**Example 1 — CO-4, missing modifier (clear resubmit)**
```
Claim: CPT 99215, modifier 25 missing, Aetna, $380, denied CO-4

P(recovery | resubmit_with_correction)    = 0.91  ← dominant
P(recovery | appeal_with_clinical_docs)   = 0.44

Recommendation: resubmit_with_correction
Rationale: "CO-4 on 99215 without modifier 25 — Aetna approves 91%
            of resubmissions with modifier added. No appeal needed."
```

**Example 2 — OA-23, timely filing expired (unrecoverable)**
```
Claim: CPT 27447, BCBS, $3,200, denied OA-23

P(recovery | appeal_with_clinical_docs)   = 0.04
P(recovery | resubmit_with_correction)    = 0.02

Recommendation: write_off
Rationale: "OA-23 — timely filing window expired. BCBS upholds 96%
            of timely filing denials. Flag for root cause analysis."
```

**Example 3 — CO-50, non-covered service (nuanced)**
```
Claim: CPT 90837 (therapy), Cigna, $200, denied CO-50

P(recovery | escalate_to_reviewer)        = 0.51  ← best
P(recovery | appeal_with_clinical_docs)   = 0.38

Recommendation: escalate_to_reviewer
Rationale: "CO-50 on 90837 — check for gap exception or OON benefit.
            51% of similar claims recovered after coverage verification.
            Verify before filing appeal to preserve appeal attempt."
```

#### The Selection Bias Warning

> Billers only take actions on claims they already believe are recoverable — so training data is biased toward "good" claims. The model risks learning biller judgment, not the true causal effect of each action.

Mitigations:
- Include **propensity features** (claim dollar, biller tenure, queue pressure) to control for selection
- Run **randomized exploration** — route ~5% of claims to random actions for unbiased labels
- Apply **inverse propensity weighting (IPW)** to reweight training examples

---

### Deep Dive: Modeling Partial Recoveries

Claim outcomes are not always binary. A $450 claim can resolve as:

```
Submitted:    $450.00
Full recovery:   $387.00  ← contracted rate paid (clean)
Partial:         $210.00  ← some line items approved, some denied
Nothing:           $0.00  ← full denial upheld
```

Partial recoveries happen due to multi-line claims, bundling adjustments, underpayments, or partial appeal wins.

#### Three Modeling Options

**Option 1: Binary with threshold (v1 — start here)**

```python
# "Recovered" = paid at least 80% of contracted rate
df["recovered"] = (df["paid_amount"] >= 0.8 * df["contracted_amount"]).astype(int)
```
- Pro: simple, interpretable, works with standard classifiers
- Con: treats $380 and $1 the same — both "not recovered"

---

**Option 2: Recovery rate as regression target (v2)**

```python
df["recovery_rate"] = df["paid_amount"] / df["submitted_amount"]
# 0.00 = nothing paid, 0.86 = $387 of $450, 1.00 = paid in full

model = xgb.XGBRegressor(...)
model.fit(X_train, y_train["recovery_rate"])

# At inference:
predicted_rate = model.predict(X)[0]                         # e.g. 0.74
expected_value = predicted_rate * claim["submitted_amount"]  # 0.74 × $450 = $333
```
- Pro: captures true dollar outcome, more accurate expected value
- Con: noisier target, harder to calibrate

---

**Option 3: Two-stage model (most expressive)**

Stage 1 — Will anything be recovered? (binary classifier)
Stage 2 — How much, given payment? (regressor on paid claims only)

```python
p_any_recovery = stage1_model.predict_proba(X)[1]   # 0.85
expected_rate  = stage2_model.predict(X)[0]          # 0.91

expected_value = p_any_recovery * expected_rate * submitted_amount
               # 0.85 × 0.91 × $450 = $347.93
```
- Pro: handles zero-inflation cleanly (many claims pay exactly $0)
- Con: two models to maintain, errors compound

---

#### Which to Use When

| Scenario | Model |
|---|---|
| v1, single-line professional claims | Binary with threshold |
| v2, want accurate dollar estimates | Recovery rate regression |
| Multi-line or institutional claims | Two-stage model |
| Underpayment detection | Regression against contracted rate |

#### Interview Answer

> "Technically recovered is continuous — partial payments are common, especially on multi-line claims. For v1 I'd treat it as binary with an 80% of contracted rate threshold, which keeps it interpretable. For v2 I'd move to a regression target — recovery rate as a fraction — so the expected value calculation is more accurate. If zero-inflation is a problem (lots of full denials), a two-stage model handles that cleanly: one model for whether anything gets paid, a second for how much."

---

## 12. Model Calibration — Senior ML Engineer Walkthrough

### What Calibration Is

A model is **well-calibrated** if, among all predictions where it outputs probability `p`, approximately `p` fraction of them are actually positive.

When your model says 80% confidence, it should be right about 80% of the time. Not 55%, not 95% — 80%.

```
Perfect calibration:
  Of all claims where model says P=0.80 → 80% actually recovered
  Of all claims where model says P=0.40 → 40% actually recovered

Miscalibrated (overconfident):
  Of all claims where model says P=0.80 → only 55% actually recovered

Miscalibrated (underconfident):
  Of all claims where model says P=0.80 → actually 92% recovered
```

This matters in RCM because billers are making **explicit ROI decisions** from your probabilities. If P=0.81 actually means 55% recovery rate, you're giving them bad financial math.

---

### How to Measure Calibration

**1. Reliability Diagram — always start here**

Bin predictions into buckets (0–0.1, 0.1–0.2, ..., 0.9–1.0). For each bucket, plot mean predicted probability vs. actual positive rate. Perfect calibration = diagonal line.

```python
from sklearn.calibration import calibration_curve
import matplotlib.pyplot as plt

fraction_of_positives, mean_predicted_value = calibration_curve(
    y_true, y_prob, n_bins=10, strategy="uniform"
)

plt.plot(mean_predicted_value, fraction_of_positives, "s-", label="Model")
plt.plot([0, 1], [0, 1], "k--", label="Perfect calibration")
```

- Curve **above** diagonal → underconfident (model says 0.4, reality is 0.6)
- Curve **below** diagonal → overconfident (model says 0.8, reality is 0.55)

**2. Expected Calibration Error (ECE)**

Weighted average of the gap between predicted probability and actual frequency:

```python
import numpy as np

def expected_calibration_error(y_true, y_prob, n_bins=10):
    bins = np.linspace(0, 1, n_bins + 1)
    ece = 0.0
    for i in range(n_bins):
        mask = (y_prob >= bins[i]) & (y_prob < bins[i+1])
        if mask.sum() == 0:
            continue
        bin_accuracy   = y_true[mask].mean()
        bin_confidence = y_prob[mask].mean()
        bin_weight     = mask.sum() / len(y_true)
        ece += bin_weight * abs(bin_accuracy - bin_confidence)
    return ece

# ECE < 0.05 is generally good
```

**3. Brier Score — calibration + discrimination combined**

```python
from sklearn.metrics import brier_score_loss

brier = brier_score_loss(y_true, y_prob)
# 0 = perfect, ~0.25 = random on balanced data
# Decomposes as: Brier = Calibration + Resolution
```

**4. Maximum Calibration Error (MCE)**

Worst-case bin gap — useful when high-confidence wrong predictions are costly:

```python
def mce(y_true, y_prob, n_bins=10):
    bins = np.linspace(0, 1, n_bins + 1)
    max_err = 0.0
    for i in range(n_bins):
        mask = (y_prob >= bins[i]) & (y_prob < bins[i+1])
        if mask.sum() == 0:
            continue
        gap = abs(y_true[mask].mean() - y_prob[mask].mean())
        max_err = max(max_err, gap)
    return max_err
```

---

### Why Models Are Miscalibrated

| Model | Typical Behavior | Why |
|---|---|---|
| **Logistic Regression** | Generally well-calibrated | Trained with log-loss directly |
| **XGBoost / GBM** | Overconfident — pushes toward 0 and 1 | Boosting focuses on hard examples |
| **Random Forest** | Underconfident — clusters around 0.2–0.8 | Averaging smooths extreme predictions |
| **Neural Networks** | Overconfident, especially large models | Modern deep nets famously overconfident |
| **SVMs** | Poorly calibrated by default | Optimizes boundary, not probabilities |
| **Naive Bayes** | Overconfident | Independence assumption violated |

**For Adonis specifically**: XGBoost will be overconfident. When it says P=0.85, the real rate might be 0.62. Calibration is required before billers can trust the numbers.

---

### Calibration Methods

**Method 1: Platt Scaling**

Fits a logistic regression on top of the model's raw output on a held-out calibration set:

```python
from sklearn.linear_model import LogisticRegression

raw_scores = base_model.predict_proba(X_cal)[:, 1].reshape(-1, 1)
platt = LogisticRegression()
platt.fit(raw_scores, y_cal)

def calibrated_predict(X):
    raw = base_model.predict_proba(X)[:, 1].reshape(-1, 1)
    return platt.predict_proba(raw)[:, 1]
```

- Pro: simple, fast, works well for monotonic miscalibration
- Con: only fits a sigmoid — can't fix non-monotonic miscalibration
- **Best for**: small calibration sets (<1000), neural nets, SVMs

---

**Method 2: Isotonic Regression**

Non-parametric — fits a monotonically non-decreasing step function. More flexible than Platt:

```python
from sklearn.isotonic import IsotonicRegression

raw_scores = base_model.predict_proba(X_cal)[:, 1]
iso = IsotonicRegression(out_of_bounds="clip")
iso.fit(raw_scores, y_cal)

calibrated_probs = iso.predict(base_model.predict_proba(X_test)[:, 1])
```

- Pro: flexible, handles arbitrary miscalibration shapes
- Con: can overfit on small sets
- **Best for**: large calibration sets (>1000), tree-based models, GBMs

---

**Method 3: CalibratedClassifierCV (recommended in practice)**

```python
from sklearn.calibration import CalibratedClassifierCV

# After training (cv="prefit") — most common in production
calibrated = CalibratedClassifierCV(base_model, method="isotonic", cv="prefit")
calibrated.fit(X_cal, y_cal)  # X_cal must be held-out, not training data

probs = calibrated.predict_proba(X_test)[:, 1]
```

**Critical**: never calibrate on your training set. Required split:

```
All data → train (60%) | calibration (20%) | test (20%)

1. Train base model on train set
2. Get predictions on calibration set
3. Fit calibrator on (calibration predictions, calibration labels)
4. Evaluate final calibrated model on test set
```

---

**Method 4: Temperature Scaling (for neural networks)**

Divides logits by a learned scalar `T` before softmax:

```python
import torch, torch.nn as nn

class TemperatureScaling(nn.Module):
    def __init__(self):
        super().__init__()
        self.temperature = nn.Parameter(torch.ones(1))

    def forward(self, logits):
        return logits / self.temperature

temp_model = TemperatureScaling()
optimizer = torch.optim.LBFGS([temp_model.temperature], lr=0.01, max_iter=50)
nll = nn.CrossEntropyLoss()

def eval_step():
    optimizer.zero_grad()
    loss = nll(temp_model(val_logits), val_labels)
    loss.backward()
    return loss

optimizer.step(eval_step)
# T > 1 → model was overconfident, softens probabilities
# T < 1 → model was underconfident, sharpens probabilities
```

- **Best for**: neural networks, LLM confidence scores

---

### Calibration vs. Discrimination — Critical Distinction

These are **independent properties**:

```python
auc  = roc_auc_score(y_true, y_prob)                    # measures ranking ability
ece  = expected_calibration_error(y_true, y_prob)        # measures probability accuracy
```

- High AUC + poor calibration: ranking is right, numbers are wrong
- Calibration is a monotonic transformation — it preserves rank order, so **AUC is unchanged after calibration**
- You can always calibrate without hurting discrimination

---

### You Can Only Measure Calibration Once You Have Actual Y Values

Calibration measures the gap between predicted probability and actual outcome rate. You cannot compute that gap without knowing what actually happened. This is a fundamental constraint with real operational implications.

#### The Outcome Lag Problem in RCM

```
Day 0:   Claim denied → model predicts P(recovery) = 0.81
Day 3:   Biller submits appeal
Day 21:  Payer adjudicates → outcome arrives (835)
Day 35:  Payment posted, matched back to original claim

↑ You can't measure calibration until Day 35 minimum
```

For complex appeals, resolution can take 60–120 days. That means:
- Flying blind on calibration for 1–4 months after deployment
- By the time you detect miscalibration, thousands of bad recommendations have already been made
- Seasonal shifts may be over before you can even measure them

#### What You CAN Do Without Outcome Labels

**1. Feature distribution shift monitoring**

```python
from scipy.stats import ks_2samp

for feature in feature_cols:
    stat, p_value = ks_2samp(train_data[feature], live_data[feature])
    if p_value < 0.05:
        alert(f"Distribution shift detected on {feature}")
```

**2. Prediction score distribution shift**

```python
ks_stat, p_val = ks_2samp(last_month_scores, this_week_scores)
# If prediction distribution shifts significantly, something has changed
```

**3. Population Stability Index (PSI)**

```python
def psi(expected, actual, n_bins=10):
    bins = np.linspace(0, 1, n_bins + 1)
    psi_value = 0
    for i in range(n_bins):
        e = ((expected >= bins[i]) & (expected < bins[i+1])).mean() + 1e-8
        a = ((actual  >= bins[i]) & (actual  < bins[i+1])).mean() + 1e-8
        psi_value += (e - a) * np.log(e / a)
    return psi_value

# PSI < 0.1  → no significant shift
# PSI 0.1–0.2 → moderate shift, investigate
# PSI > 0.2  → significant shift, recalibrate
```

**4. Frozen holdout from training time**

Keep a static labeled set locked away at training time. Run the model on it periodically — labels are already known:

```python
holdout_X, holdout_y = load_frozen_holdout()

# Run monthly — no waiting for new outcomes
current_ece = expected_calibration_error(
    holdout_y,
    model.predict_proba(holdout_X)[:, 1]
)
```

Limitation: reflects training distribution only — won't catch new payer behavior drift, but will catch model degradation on known patterns.

#### The Full Calibration Measurement Timeline

```
[Training time]
  → Split: train / calibration / frozen holdout / test
  → Train model, fit calibrator, measure initial ECE
  → Lock frozen holdout away

[Deploy — Weeks 1–4: no outcome labels yet]
  → Monitor: feature drift (KS test, PSI)
  → Monitor: prediction score distribution
  → Monitor: proxy metrics (appeal submission rate, biller override rate)

[Weeks 4–8: early outcomes arriving]
  → Partial calibration on resolved claims
  → Enough signal to detect major miscalibration

[Month 2+: full measurement possible]
  → Compute ECE on rolling 30/60/90-day resolved claims
  → Compare reliability diagram vs. training baseline
  → Trigger recalibration if ECE exceeds threshold
```

---

### Calibration Under Distribution Shift

Models recalibrated on historical data go stale when distribution shifts — which in RCM happens constantly (payer rule changes, seasonal deductible resets, CMS coding updates).

**Recalibration strategy**:
- Full model retrain: heavy, monthly cadence
- Recalibration only (update isotonic/Platt layer): lightweight, weekly cadence
- Key MLOps advantage: you can fix calibration drift without retraining the full model

---

### Calibration with Class Imbalance

Class imbalance causes models to underestimate positive class probability. Two additional concerns:

**Prior probability shift**: if you train on 20% positive rate but deploy on 5%, calibrated probabilities will be off even if the model was well-calibrated at training time:

```python
def adjust_for_prior(p_train_prior, p_deploy_prior, p_model):
    odds_model = p_model / (1 - p_model)
    odds_ratio = (p_deploy_prior / (1 - p_deploy_prior)) / \
                 (p_train_prior  / (1 - p_train_prior))
    adjusted_odds = odds_model * odds_ratio
    return adjusted_odds / (1 + adjusted_odds)
```

---

### Production Checklist

```
Training time:
  [ ] Split: train / calibration / test — never calibrate on train set
  [ ] Measure raw ECE before calibrating (know your baseline)
  [ ] Choose method: isotonic for GBMs, Platt/temperature for NNs
  [ ] Fit calibrator on calibration set only
  [ ] Verify AUC unchanged after calibration
  [ ] Log calibration method + set size as model metadata

Serving:
  [ ] Pipeline = base_model → calibrator (both versioned together)
  [ ] Never ship base model without calibrator if downstream uses probabilities

Monitoring:
  [ ] Track ECE on production predictions as outcomes arrive
  [ ] Track PSI and feature drift as leading indicators (no labels needed)
  [ ] Alert if ECE drifts above threshold
  [ ] Lightweight recalibration pipeline — update calibrator without full retrain
```

---

### Three-Sentence Interview Answer

> "Calibration is whether the model's probability outputs are accurate — not just whether it ranks correctly. A model can have great AUC and terrible calibration; they're orthogonal. For RCM specifically, calibration is non-negotiable because billers are making ROI decisions from the probabilities — if P=0.81 actually means 55% recovery rate, the math breaks. I'd calibrate XGBoost with isotonic regression on a held-out calibration set, monitor ECE weekly once outcomes arrive, use PSI and feature drift as leading indicators before labels are available, and maintain a lightweight recalibration pipeline so I can fix drift without retraining the full model."

---

## 14. Ranking Without Recovery Labels (Label Scarcity)

### The Problem Reframed

```
What you have:
  ✅ Thousands of denied claims (CARC, CPT, ICD, payer, amount, dates)
  ✅ Some rule-based intuition about what's recoverable
  ❌ No labeled outcomes — don't know which claims recovered after resubmission

What you want:
  A ranked list of denied claims ordered by "worth working first"
```

This is a **prioritization problem**, not a prediction problem. You don't need to know the exact probability of recovery — you just need to know which claims to work first given finite biller capacity.

---

### Step 1: Rules-Based Scoring Function (No ML Needed)

Build a deterministic score from domain knowledge first. You already have the signals — you just need to combine them:

```python
def recoverability_score(claim: dict) -> float:
    score = 0.0

    # --- CARC recoverability (domain knowledge) ---
    carc_weights = {
        "CO-4":   0.90,   # wrong modifier — almost always fixable
        "CO-16":  0.70,   # missing info — fixable with documentation
        "CO-11":  0.65,   # dx inconsistent — CDI review often resolves
        "CO-50":  0.35,   # non-covered — depends on plan, worth checking
        "CO-97":  0.40,   # bundling — NCCI edit review sometimes overturns
        "OA-23":  0.00,   # timely filing expired — unrecoverable
        "PR-1":   0.05,   # deductible — patient liability, not payer issue
    }
    score += carc_weights.get(claim["carc_code"], 0.30) * 40  # 40% of total

    # --- Dollar amount ---
    dollar_score = min(claim["submitted_amount"] / 1000, 1.0)
    score += dollar_score * 30  # 30% of total

    # --- Urgency: days to timely filing deadline ---
    days_left = claim["days_to_filing_deadline"]
    if days_left <= 0:
        return 0.0  # expired — don't work it
    urgency = max(0, 1 - (days_left / 365))
    score += urgency * 20  # 20% of total

    # --- Technical vs. clinical denial ---
    is_technical = claim["denial_type"] in ["modifier", "coding", "admin"]
    score += (10 if is_technical else 0)  # 10% of total

    return score

denied_claims_df["score"] = denied_claims_df.apply(recoverability_score, axis=1)
work_queue = denied_claims_df.sort_values("score", ascending=False)
```

This gets you 80% of the value before writing a single line of ML code. It's explainable, auditable, and immediately deployable.

---

### Step 2: Construct Proxy Labels

When you don't have true outcome labels, use **proxy signals** you CAN observe that correlate with recoverability:

| Proxy Signal | What It Tells You | How to Compute |
|---|---|---|
| **Biller chose to work it** | Experienced billers implicitly know what's recoverable | `action != write_off` in historical data |
| **Same CPT × payer × CARC was worked before** | Historical precedent exists | Lookup in Snowflake |
| **Payer has paid this CPT before** | Payer considers service covered | `payer_cpt_historical_payment_rate > 0` |
| **Denial is technical, not clinical** | Technical = far more fixable | CARC category mapping |
| **No prior resubmission attempts** | First denial — highest chance of quick fix | `prior_submission_count == 0` |

```python
def construct_proxy_label(claim, historical_data):
    signals = []

    # Was a similar CARC × CPT × payer ever successfully worked?
    similar = historical_data[
        (historical_data["carc_code"] == claim["carc_code"]) &
        (historical_data["cpt_code"]  == claim["cpt_code"]) &
        (historical_data["payer_id"]  == claim["payer_id"]) &
        (historical_data["action"]    != "write_off")
    ]
    signals.append(len(similar) > 0)

    # Has this payer ever paid this CPT?
    payer_pays_cpt = historical_data[
        (historical_data["payer_id"]    == claim["payer_id"]) &
        (historical_data["cpt_code"]    == claim["cpt_code"]) &
        (historical_data["paid_amount"] >  0)
    ]
    signals.append(len(payer_pays_cpt) > 0)

    # Is the denial in the "fixable" category?
    fixable_carcs = {"CO-4", "CO-16", "CO-11", "CO-96", "CO-97"}
    signals.append(claim["carc_code"] in fixable_carcs)

    # Proxy label: positive if majority of signals agree
    return int(sum(signals) >= 2)
```

These labels are **noisy** — not ground truth. But good enough to train an initial ranking model, especially combined with the rules-based score.

---

### Step 3: Learning to Rank (LTR) — No Outcome Labels Required

If you have any signal about **relative preference** between claims (even just "biller chose to work claim A over claim B"), you can train a Learning to Rank model without absolute outcome labels.

**Pairwise LTR:**

```python
import numpy as np
from sklearn.ensemble import GradientBoostingClassifier

def build_pairwise_training_data(claims_df, preference_col="biller_worked"):
    """
    For each pair (A worked, B not worked), create:
      features(A) - features(B) → label 1  (A preferred)
      features(B) - features(A) → label 0  (B not preferred)
    """
    worked     = claims_df[claims_df[preference_col] == 1]
    not_worked = claims_df[claims_df[preference_col] == 0]

    X_pairs, y_pairs = [], []
    for _, pos in worked.iterrows():
        for _, neg in not_worked.sample(min(5, len(not_worked))).iterrows():
            diff = pos[feature_cols].values - neg[feature_cols].values
            X_pairs.append(diff);  y_pairs.append(1)
            X_pairs.append(-diff); y_pairs.append(0)

    return np.array(X_pairs), np.array(y_pairs)

X_pairs, y_pairs = build_pairwise_training_data(historical_claims)
ranker = GradientBoostingClassifier()
ranker.fit(X_pairs, y_pairs)
```

**Or use XGBoost's native LTR (cleaner in production):**

```python
import xgboost as xgb

dtrain = xgb.DMatrix(X_train, label=y_proxy)
dtrain.set_group(group_sizes)  # claims per billing batch

params = {
    "objective":   "rank:pairwise",
    "eval_metric": "ndcg",
    "max_depth":   6,
    "eta":         0.1,
}
model = xgb.train(params, dtrain, num_boost_round=200)
```

The key insight: **ranking only requires relative preference, not absolute outcomes.** Biller behavior gives you relative preference for free.

---

### Step 4: Bootstrap Real Labels via Exploration

The rules scorer + LTR model generates a working queue. Now use that queue to collect your first real outcome labels:

```python
# Route top-ranked claims to resubmission
# Track outcomes via 835 response and payment posting
# Build labeled dataset: (claim_features, action, recovered: 0/1)

# After 4–6 weeks:
#   Real outcome labels exist for worked claims
#   Retrain as supervised model (Layer 2 from Section 11)
```

This is the **bootstrapping path** — you start with what you have, generate labels through operation, and graduate to a fully supervised system over time.

---

### The Full Progression

```
Week 1–4:    Rules-based scoring (CARC weights + dollar + urgency)
             → Immediately deployable, explainable, no ML needed

Month 1–3:   LTR on proxy labels (biller preference + payer history)
             → Better personalization, handles edge cases rules miss

Month 3–6:   Hybrid: rules gate + LTR + early real outcome labels
             → First supervised signal arriving

Month 6+:    Full supervised model: P(recovery | action)
             → Calibrated probabilities, expected value ranking
             → Full Section 11 architecture becomes possible
```

---

### Interview Answer

> "If we don't have recovery outcome labels yet, I wouldn't block on that — I'd start with a rules-based scoring function using CARC recoverability weights, dollar amount, urgency, and denial type. That's immediately deployable and gives billers a prioritized queue on day one. In parallel, I'd construct proxy labels from observable signals — was a similar CARC × CPT × payer combination ever worked before, has this payer ever paid this CPT — and use those to train a pairwise learning-to-rank model that improves on the rules without needing true outcomes. The key insight is that ranking only requires relative preference, not absolute labels, and we get relative preference for free from biller behavior. Once the system is live and claims are being worked, real outcome labels start accumulating and we graduate to the supervised model. It's a bootstrapping problem, not a blocking problem."

---

## 13. Data Bias, Missing Actions, and Counterfactual Reasoning

### The Core Problem

When building the recommender, your training data is not a random sample of (claim, action, outcome). It's a heavily selected sample where every action was chosen by a human biller following institutional policies and personal judgment. This creates structural gaps that naive ML cannot handle.

Four distinct problems:

```
1. Dollar threshold → structural data void for low-dollar claims
2. Action sparsity → unreliable estimates for rarely-tried actions
3. Selection bias  → model learns biller judgment, not causal action effects
4. Skewed distribution → some CARC codes have near-zero variation in action taken
```

---

### What "Counterfactual" Means

"Counterfactual" literally means **contrary to what actually happened.** It's the answer to: *what would have happened if we had done something different?*

In RCM terms:
> "What would have happened to this claim if we had appealed it — given that we actually wrote it off?"

That's a counterfactual. The appeal never happened, so you never observed the outcome. It's a **missing potential outcome.**

#### The Fundamental Problem

For every claim, multiple actions are possible. But only one is ever taken — and you only observe the outcome of that one:

```
Claim CLM-005 (CO-16, Cigna, $35):

Action actually taken:   write_off  → outcome: $0    ✅ observed
Action never taken:      appeal     → outcome: ???   ❌ counterfactual
Action never taken:      resubmit   → outcome: ???   ❌ counterfactual
```

You will never know what would have happened if CLM-005 had been appealed. That outcome is permanently unobservable. This is the **Fundamental Problem of Causal Inference.**

#### A Concrete Example

Three identical claims — same CARC, CPT, payer, dollar amount. Three different billers, three different actions:

```
Claim A → biller appeals with docs → payer pays $420   ✅ observed
Claim B → biller resubmits         → payer pays $420   ✅ observed
Claim C → biller writes off        → $0 recovered      ✅ observed
                                      (but was this the right call?)
```

For Claim C, the counterfactual questions are:
- What would have happened if biller C had appealed instead?
- What would have happened if biller C had resubmitted?

You don't know. Claim C was written off. Those outcomes are counterfactual — permanently unobserved.

#### Why This Breaks Naive ML

Training on observed (action, outcome) pairs doesn't give you the true causal effect of each action — it gives you a confounded correlation:

```
Model sees: "Appeals tend to succeed"
Reality:     Only because billers appeal claims they already think are winnable.
             The appeal didn't cause recovery — claim quality caused both the
             decision to appeal AND the recovery.

Model sees: "Write-offs never recover"
Reality:     True by definition. You can't recover a claim you gave up on.
             This tells you nothing about whether those claims COULD have recovered.
```

This is **confounding** — the biller's action choice is correlated with the claim's underlying recoverability, which you can't directly observe.

#### What Counterfactual Data Actually Is

Counterfactual data means: **outcome labels for actions that weren't taken under normal policy.**

The only ways to get it:

**1. Randomized exploration (gold standard)**

Force some claims into non-default actions regardless of biller judgment:

```python
def assign_action_with_exploration(claim, model, exploration_rate=0.05):
    if random.random() < exploration_rate:
        # Override normal policy — collect counterfactual outcome
        action = random.choice(["resubmit_with_correction", "appeal_with_clinical_docs"])
        return {"action": action, "exploratory": True}
    return model.recommend(claim)
```

```
Normal policy for $35 CO-16 claim: always write_off
Exploration (5%):                   randomly assign appeal or resubmit

Now you observe:
  Claim D (written off, normal):   $0       → expected
  Claim E (appealed, explored):    $35 paid → counterfactual revealed ✅
  Claim F (resubmitted, explored): $35 paid → counterfactual revealed ✅
```

**2. Natural experiments**

Accidental deviations from normal policy — a new biller who didn't know the threshold policy, a system glitch that routed a write-off candidate to appeal. These accidentally reveal counterfactual outcomes without a designed experiment.

**3. Instrumental variables (advanced)**

Find an external factor that changed which action was taken, but had no direct effect on recoverability. Example: a payer portal was down for a week, forcing billers to use a different submission channel. Claims during that week were treated differently for reasons unrelated to their recoverability — that's a natural instrument.

#### One-Line Version

> "Counterfactual data is the outcome of an action that was never actually taken — it's unobservable by definition, which is why we need randomized exploration to collect it and why naive ML on historical data is always biased toward what billers already chose to do."

---

### The Four Data Problems and How to Handle Them

#### Problem 1: Dollar Threshold → Structural Data Void

**Short-term: be explicit about the void**

```python
def recommend_action(claim, model, min_dollar_threshold=50):
    if claim["submitted_amount"] < min_dollar_threshold:
        return {
            "action": "write_off",
            "p_recovery": None,
            "rationale": "Below working threshold — no historical outcome data for other actions.",
            "flag": "DATA_VOID"
        }
```

Never emit a fake probability. Return a flag, not a number.

**Medium-term: pilot to collect ground truth**

Route a random sample of below-threshold claims to non-write-off actions (exploration policy above). Even 100 explored claims gives real signal on whether low-dollar claims are worth working.

**Long-term: mine natural experiments**

Find claims that were worked despite being below threshold — during slow periods, by new billers, or in one-off audits. These are unintentional natural experiments.

---

#### Problem 2: Action Sparsity → Unreliable Estimates

Attach **confidence intervals** to every P(recovery | action) estimate, and suppress recommendations when sample size is too small:

```python
from scipy.stats import beta as beta_dist

def recovery_probability_with_confidence(successes, trials, confidence=0.90):
    if trials == 0:
        return None, None, None  # no data at all

    alpha = successes + 1           # successes + prior
    b     = trials - successes + 1  # failures + prior

    lower = beta_dist.ppf((1 - confidence) / 2, alpha, b)
    upper = beta_dist.ppf((1 + confidence) / 2, alpha, b)
    mean  = alpha / (alpha + b)
    return mean, lower, upper

# 3 successes out of 4 trials:
p, lo, hi = recovery_probability_with_confidence(3, 4)
# mean=0.71, CI=[0.29, 0.96] → wide interval, suppress recommendation

# 38 successes out of 47 trials:
p, lo, hi = recovery_probability_with_confidence(38, 47)
# mean=0.81, CI=[0.68, 0.91] → tight interval, safe to recommend


MIN_SAMPLE_SIZE = 20
MAX_CI_WIDTH    = 0.30

def should_recommend(p, lower, upper, n_samples):
    if n_samples < MIN_SAMPLE_SIZE:
        return False, "insufficient_data"
    if (upper - lower) > MAX_CI_WIDTH:
        return False, "high_uncertainty"
    return True, "ok"
```

---

#### Problem 3: Selection Bias → Propensity Modeling

Build a **propensity model** that estimates why a biller chose a particular action. Use it to reweight training examples via **Inverse Propensity Weighting (IPW)**:

```python
from sklearn.linear_model import LogisticRegression
import numpy as np

# Step 1: Train propensity model
# Predicts P(action=appeal | claim_features, biller_context)
propensity_features = [
    "submitted_amount",
    "claim_age_days",
    "biller_tenure_months",
    "queue_size_at_time",         # were they rushed?
    "day_of_week",                # Friday afternoon effect
    "prior_appeals_this_week",    # biller fatigue
]

propensity_model = LogisticRegression()
propensity_model.fit(X_propensity[train_idx], action_taken[train_idx])

# Step 2: Compute inverse propensity weights
# Claims unlikely to be appealed but were → upweighted (most informative)
# Claims likely to be appealed and were   → downweighted (expected, less informative)
p_action = propensity_model.predict_proba(X_propensity)
ipw = 1.0 / p_action[np.arange(len(action_taken)), action_taken]
ipw = np.clip(ipw, 1.0, 10.0)  # clip extreme weights

# Step 3: Train outcome model with IPW
outcome_model.fit(X_train, y_train, sample_weight=ipw[train_idx])
```

---

#### Problem 4: Skewed Action Distribution → Stratified Modeling

For CARC × action combinations with extreme skew, don't use ML — route through rules:

```python
HIGH_DATA_THRESHOLD = 200
LOW_DATA_THRESHOLD  = 20

def select_model(claim_type, action, historical_counts):
    n = historical_counts.get((claim_type, action), 0)

    if n >= HIGH_DATA_THRESHOLD:
        return "ml_model"          # reliable ML prediction
    elif n >= LOW_DATA_THRESHOLD:
        return "ml_with_wide_ci"   # ML + flag uncertainty
    else:
        return "rules_engine"      # fall back, don't use ML
```

For OA-23 (99% write_off in training data), the model has no signal for any other action — route it directly to the rules engine.

---

### The Unified Recommendation Gate

Every recommendation passes through a data quality check before the model output is trusted:

```
[Claim arrives]
       │
       ▼
[Data quality gate]
  ├── Below dollar threshold?     → DATA_VOID: return write_off + flag
  ├── CARC in rules-only set?     → RULES: bypass ML entirely
  └── Proceed to ML
       │
       ▼
[ML model: P(recovery | action) for each action]
       │
       ▼
[Confidence gate per action]
  ├── n_samples < 20?             → suppress action
  ├── CI width > 0.30?            → suppress action
  └── Pass → include in ranking
       │
       ▼
[IPW-adjusted ranking by expected value]
       │
       ▼
[Exploration override: 5% routed to random action for data collection]
       │
       ▼
[Final recommendation with uncertainty flag]
```

---

### Interview Answer

> "This is a causal inference problem, not just a supervised learning problem. The observed data is not a random sample of (claim, action, outcome) — it's heavily selected by biller judgment and policies like dollar thresholds. The model risks learning biller behavior instead of the true causal effects of actions.

> I'd handle it in three layers. First, explicit data voids: for claims below threshold or action × claim combinations with fewer than 20 samples, I suppress the ML recommendation entirely and fall back to the rules engine — I never emit a fake probability. Second, propensity modeling: I model why billers chose each action and use inverse propensity weighting to correct for selection bias in the outcome model. Third, an exploration policy: route ~5% of claims to randomized actions to collect counterfactual ground truth over time — that's the only way to escape the data void for low-dollar or rarely-tried actions. The key principle: uncertainty should be visible and actionable, not hidden behind a confident-looking number."

---

## 15. Shadow Mode and Canary Releases

Both are **safe deployment strategies** for getting a new model into production without risking a bad experience for users. They solve the same core problem: "how do I test a new model on real traffic without billers being harmed if it's wrong?" — but at different stages of deployment confidence.

---

### Shadow Mode

The new model runs **in parallel with the current system** but its outputs are never shown to users or acted upon. Completely invisible to billers.

```
[Claim arrives]
       │
       ├──────────────────────────────┐
       ▼                              ▼
[Production model]            [Shadow model]
  output: appeal               output: resubmit
  → shown to biller            → logged only, never shown
  → action taken               → no action taken
       │
       ▼
[Outcome recorded]
  → label attached to BOTH model predictions
  → production: recommended appeal → paid ✅
  → shadow:     recommended resubmit → would have paid? unknown
```

**What you learn from shadow mode:**
- How often do the two models agree vs. disagree?
- On disagreements, which model's recommendation was closer to the actual outcome?
- Are the shadow model's predictions calibrated on real production data?
- What's the shadow model's latency under real production load?
- Does the shadow model fail on any edge cases it never saw in training?

**Key property**: zero risk. Billers never know it's running. If the shadow model crashes or produces garbage, nothing breaks.

**When to use it:**
- Deploying a fundamentally new model (v1 → v2)
- Changing the feature pipeline significantly
- When offline evaluation isn't sufficient and you need real-world validation before committing

---

### Canary Release

The new model is **live and serving real recommendations** — but only to a small percentage of traffic. Most users still get the old model.

```
[Claims arrive]
       │
       ▼
[Traffic splitter]
  ├── 95% → Production model (old)  → recommendations shown to billers
  └──  5% → Canary model (new)      → recommendations shown to billers

[Monitor both groups]
  Production: appeal_success_rate = 0.74, net_recovery = $412K/week
  Canary:     appeal_success_rate = 0.79, net_recovery = $438K/week
  → Canary looks better → gradually increase traffic
```

**The gradual ramp:**
```
Week 1:   5% canary  → monitor, check for errors and metrics
Week 2:  10% canary  → metrics still good, no incidents
Week 3:  25% canary  → confident, accelerate
Week 4:  50% canary  → near parity
Week 5: 100% canary  → full cutover, old model retired
```

**What to monitor during canary:**
- Business metrics: net recovery rate, appeal success rate, biller override rate
- Model metrics: prediction distribution, calibration, latency
- Error rates: null predictions, timeouts, feature pipeline failures
- Biller behavior: are they following canary recommendations at the same rate?

**Rollback trigger**: if any metric degrades beyond threshold at any ramp stage, instantly route 100% back to the production model. Because only 5–10% of traffic was affected, blast radius is small.

---

### Shadow Mode vs. Canary — Side by Side

| | Shadow Mode | Canary Release |
|---|---|---|
| **New model output shown to users?** | No — logged only | Yes — real recommendations |
| **Risk to billers** | Zero | Small (only % in canary) |
| **Can measure business impact?** | Indirectly | Yes — directly |
| **Can catch production bugs?** | Yes | Yes |
| **Typical timing** | Before canary | After shadow validation |
| **Duration** | 1–4 weeks | 2–6 weeks |

---

### The Full Deployment Sequence

```
[Offline evaluation]
  → Train, evaluate on held-out test set (AUC, ECE, business simulations)
  → Gate: does it beat baseline?

       ↓ passes

[Shadow mode]
  → Run alongside production, log predictions silently
  → Compare agreement rate, calibration, latency on real traffic
  → Gate: agrees on >X% of claims, better on disagreements, no crashes

       ↓ passes

[Canary — 5%]
  → Live recommendations to small % of billers
  → Monitor business metrics + error rates
  → Ramp gradually if metrics hold

       ↓ passes

[Full rollout — 100%]
  → Old model retired, new model becomes baseline
  → Continue monitoring for drift
```

---

### Why This Matters for Adonis Specifically

In RCM, a bad model recommendation has direct financial consequences — a biller follows bad advice, works an unrecoverable claim, wastes 45 minutes, and the claim still gets denied. Or worse, skips an appeal that would have recovered $3,000.

Shadow mode and canary releases are how you validate that offline evaluation improvements actually translate to real-world financial outcomes — without betting the entire biller workflow on an untested model.

The feedback loop lag (outcomes take weeks) also means shadow mode needs to run longer than in a typical web ML system. You need enough resolved claims to measure whether the shadow model's recommendations actually led to better outcomes.

---

### One-Line Version for the Interview

> "Shadow mode runs the new model silently alongside production — outputs logged but never acted on — to validate behavior on real traffic before any user exposure. Canary releases go live to a small traffic slice, measure real business impact, and let you ramp gradually with a fast rollback path. Together they close the gap between offline evaluation and production confidence."

---

## Appendix: Deep Dives

### A. What Is a Clearinghouse?

A clearinghouse is a **middleware company** that sits between the provider's billing system and the payer. Think of it as a translator + router + validator.

The core problem it solves: a hospital might bill 50 different payers (Cigna, Aetna, BCBS, Medicare, Medicaid, hundreds of regional plans). Each payer has slightly different formatting requirements, different submission portals, different EDI specs. Without a clearinghouse, the billing team would need to manage 50 direct connections. Instead, they send everything to one place and the clearinghouse handles the routing.

#### What a Clearinghouse Actually Does

**1. Format validation**
Before a claim ever reaches a payer, the clearinghouse checks that the 837 file is well-formed — correct EDI segments, required fields present, valid NPI, valid payer ID, dates in the right format. If it fails here, it bounces back immediately as a **rejection** (not a denial — the payer never saw it).

**2. Translation + routing**
Takes the 837 from the provider's billing system format and routes it to the correct payer using the payer ID. Some payers want 837P (professional), others want 837I (institutional). The clearinghouse handles that mapping.

**3. Payer-specific rule scrubbing**
Good clearinghouses also apply payer-specific pre-edits — they know "Cigna requires field X in loop 2300" or "Medicare rejects claims missing the rendering provider NPI." Catching these before submission saves a round-trip.

**4. Acknowledgment handling**
After submission, the clearinghouse returns:
- **999** — functional acknowledgment ("we received your file, it's structurally valid")
- **277CA** — claim acknowledgment ("here's the status of each individual claim in the batch")

If a claim gets a `277CA` with a rejection reason, it never reached the payer. It needs to be fixed and resubmitted.

**5. 835 delivery on the return trip**
When the payer adjudicates and sends back an 835 (remittance advice), it often flows back through the clearinghouse too, which then delivers it to the provider's billing system.

#### The Major Clearinghouses

| Clearinghouse | Notes |
|---|---|
| **Availity** | Largest in the US, now majority-owned by payers |
| **Change Healthcare** | Massive — processes ~15B transactions/year (acquired by Optum) |
| **Waystar** | Strong in hospital/health system market |
| **Trizetto** | Common in large enterprise |
| **Office Ally** | Popular with smaller practices (free tier) |

> **Industry context worth knowing**: The Change Healthcare ransomware attack in early 2024 took down a large portion of US claims processing nationally for weeks — it exposed just how centralized and fragile this infrastructure is.

#### Why It Matters for Adonis

Clearinghouse rejections are a **different failure mode** than payer denials:

- A **clearinghouse rejection** means the claim has a data or format problem — missing field, wrong code, bad NPI. Usually fast to fix, resubmit same day.
- A **payer denial** means the claim was adjudicated and rejected on policy grounds — no auth, not medically necessary, wrong diagnosis. Requires appeal or correction.

Adonis needs to distinguish these because the intervention is completely different:
- `277CA` rejection → fix the data, resubmit
- `835` denial with `CO-16` → gather clinical documentation, draft appeal

The latency risk is also real: clearinghouse rejections often sit unnoticed in a queue for days while billing teams focus on payer-side denials — all while the timely filing clock is ticking.

#### One-Line Version for the Interview

> "A clearinghouse is a middleware layer that validates, translates, and routes claims from providers to payers — rejections there are format problems, not coverage problems, so they need completely different fixes than payer denials."

---

### B. What Are EDI Specs?

**EDI** stands for **Electronic Data Interchange** — a standardized format for exchanging business documents between computer systems without human intervention.

In healthcare, EDI specs are the technical rules that define exactly how a claims document must be structured so that any billing system can send it and any payer system can read it. The governing standard in US healthcare is **X12**, maintained by ANSI.

#### Why It Exists

Before EDI, claims were paper forms mailed to payers. EDI replaced that with structured electronic files — but to work across hundreds of different billing systems and payers, everyone had to agree on a common format. That's what the X12 specs define.

#### What an EDI File Actually Looks Like

It's a flat text file with tightly packed segments separated by `*` and `~`. Not human-readable. Example snippet from an 837:

```
ISA*00*          *00*          *ZZ*PROVIDER123    *ZZ*CIGNA          *230101*1200*^*00501*000000001*0*P*:~
GS*HC*PROVIDER123*CIGNA*20230101*1200*1*X*005010X222A1~
ST*837*0001*005010X222A1~
NM1*IL*1*SMITH*JOHN****MI*123456789~
CLM*CLM001*500***11:B:1*Y*A*Y*I~
HI*ABK:Z23~
SV1*HC:99215*500*UN*1***1~
SE*12*0001~
```

Each segment has a purpose:
- `ISA/GS` — envelope (sender, receiver, date)
- `NM1*IL` — patient (insured) demographic
- `CLM` — claim-level info (claim ID, total charge, place of service)
- `HI` — diagnosis codes (ICD-10)
- `SV1` — service line (CPT code, charge amount, units)

#### The Key X12 Transaction Sets in RCM

| Transaction | EDI Code | What It Is |
|---|---|---|
| Claim submission | **837P / 837I** | Professional / Institutional claim |
| Eligibility inquiry | **270** | "Is this patient covered?" |
| Eligibility response | **271** | "Yes, here's their coverage" |
| Claim status request | **276** | "What's the status of this claim?" |
| Claim status response | **277 / 277CA** | Status update / clearinghouse ack |
| Prior auth request | **278** | Request for pre-authorization |
| Remittance advice | **835** | Payment explanation (CARC/RARC codes) |

#### Why Payer EDI Specs Vary

The X12 standard defines the structure, but it allows **implementation guides** — payer-specific customizations. So:
- The base `837P` spec says field X is optional
- Cigna's implementation guide says field X is **required** for their claims
- Medicare's implementation guide has different loop requirements than Aetna's

This is exactly why clearinghouses exist — they know each payer's implementation guide and can catch payer-specific violations before submission.

#### Why It Matters for Adonis

When Adonis ingests 837 and 835 files, they're parsing these EDI specs to extract structured data — CPT codes, ICD codes, CARC/RARC codes, amounts, payer IDs, provider NPIs. The quality of that parsing pipeline directly affects every downstream ML feature. Bad EDI parsing = corrupted training data.

#### One-Line Version for the Interview

> "EDI specs are the agreed-upon formatting rules for healthcare documents — like a strict JSON schema, but from the 1980s, still running most of US healthcare."

---

### C. What Does It Mean to Write Off a Claim?

A **write-off** is when a provider permanently removes an outstanding balance from their accounts receivable — essentially deciding "we're not going to collect this money, and we're accepting the loss."

It's an accounting action, not a clinical one. The claim doesn't disappear, but the provider stops pursuing payment and adjusts their books to reflect that the revenue won't come in.

#### The Two Types of Write-Offs

**1. Contractual write-offs (expected, normal)**
When a provider is in-network with a payer, they've agreed to accept a negotiated rate. If they charge $500 but the contracted rate is $420, the $80 difference is written off automatically — it was never collectible. This happens on every single in-network claim and is not a problem.

```
Billed amount:      $500
Contracted rate:    $420  ← what payer agreed to pay
Contractual W/O:    -$80  ← written off, expected
Patient liability:  $50   ← deductible/copay
Payer pays:         $370
```

**2. Bad debt / discretionary write-offs (the problem)**
These are write-offs on money that *could* have been collected but wasn't pursued. This is where revenue leakage lives. Common scenarios:

- **Denial not worked**: billing team receives a CO-16 denial, determines it's not worth the time to appeal, writes it off
- **Timely filing expired**: deadline passed before anyone caught the rejection — now unrecoverable, must write off
- **Low-dollar threshold**: billing team has a policy like "don't work denials under $50" — those get auto written off
- **Exhausted appeals**: appealed twice, payer upheld the denial, give up and write off
- **Patient balance uncollectible**: patient didn't pay, sent to collections, eventually written off as bad debt

#### Why Write-Offs Are a Core Adonis Problem

The insidious thing is that **write-offs look like a clean close on a claim** in the billing system. The account is resolved. But from a revenue perspective, it's lost money — and often it was avoidable.

The pattern Adonis cares about:
- Billing team gets 200 CO-16 denials from Cigna this week
- 140 of them are under $75
- Team writes off the 140, works the 60 high-dollar ones
- Those 140 × $60 average = **$8,400 written off that week alone**
- Multiplied across a year = significant revenue that a better triage system could have recovered

This is exactly why **recoverability scoring** matters — if Adonis can show that 80 of those 140 "low dollar" denials are actually easy fixes (wrong modifier, one-click resubmit), the ROI calculation changes and the biller works them instead of writing them off.

#### Write-Off vs. Adjustment vs. Denial

| Term | Meaning |
|---|---|
| **Denial** | Payer refused to pay — still actionable |
| **Adjustment** | Payer paid less than billed — may be contractual or an underpayment |
| **Write-off** | Provider stops pursuing — balance removed from AR |
| **Bad debt** | Write-off after failed collections attempt |

#### One-Line Version for the Interview

> "A write-off is the provider giving up on collecting a balance — contractual write-offs are expected and normal, but discretionary write-offs on workable denials are pure revenue leakage that Adonis exists to prevent."

---

### D. What Is RPA?

**RPA** stands for **Robotic Process Automation** — software that mimics human interactions with a computer interface to automate repetitive, rule-based tasks.

Instead of a human clicking through a payer portal, filling out a form, and hitting submit, an RPA bot does it automatically by driving the same UI a human would use — no special API required.

#### How It Works

RPA tools (UiPath, Automation Anywhere, Microsoft Power Automate) record or script interactions with applications:
- Open browser → navigate to payer portal
- Log in with credentials
- Search for claim by ID
- Read the status or denial reason
- Fill in appeal form fields
- Upload attachment
- Click submit
- Confirm submission and log the result

From the payer portal's perspective, it looks like a human doing it. The bot is just faster, tireless, and consistent.

#### Why RPA Exists in RCM Specifically

The core problem: **payers don't have good APIs**. Or if they do, they're limited, inconsistently implemented, or require expensive contracts to access. The reality is:

- ~60% of payer interactions still happen through web portals
- Many payers require appeals to be submitted through their proprietary portal (not via clearinghouse)
- Prior auth status checks often require logging into a portal and reading a screen
- Some payers still require **fax** (yes, in 2025) — RPA can automate fax dispatch too

RPA is the workaround for a fragmented payer ecosystem that hasn't standardized on APIs.

#### RPA in the Adonis Context

In the Payer Interaction Agent architecture, RPA is the tool the agent uses to actually execute portal interactions:

```
[Agent decides: "submit appeal to Cigna portal"]
        │
        ▼
[RPA Tool: Playwright/Selenium script]
  - Navigate to cigna.com/provider-portal
  - Log in
  - Search claim ID
  - Upload appeal letter PDF
  - Submit
  - Capture confirmation number
        │
        ▼
[Agent logs outcome, updates claim state]
```

Modern approaches use **Playwright** (Microsoft) or **Selenium** for the browser automation layer, sometimes with an LLM on top to handle dynamic UI changes — if the portal layout changes, a pure script breaks, but an LLM-guided agent can adapt by reading the page and figuring out where to click.

#### RPA Limitations Worth Knowing

| Limitation | Why It Matters |
|---|---|
| **Brittle to UI changes** | Portal redesign breaks the script overnight |
| **Portal ToS risk** | Some payers prohibit automated access — legal review needed |
| **Error handling is hard** | CAPTCHAs, MFA, session timeouts all require special handling |
| **Not real-time** | Portal scraping is slow compared to API calls |
| **Audit trail complexity** | Harder to log exactly what the bot did vs. an API call |

**Architecture principle**: API-first, RPA-as-fallback — use payer APIs (Availity, Change Healthcare) where available, drop down to RPA only when no API exists.

#### One-Line Version for the Interview

> "RPA is software that automates clicks and form-fills through a UI — in RCM it's the workaround for payers that don't have APIs, letting agents submit appeals and check claim status through portals the same way a human biller would."

---

### E. What Is BAA-Covered Infrastructure?

**BAA** stands for **Business Associate Agreement** — a legally required contract under HIPAA between a covered entity (the healthcare provider) and any vendor that handles Protected Health Information (PHI) on their behalf.

#### The HIPAA Framework First

HIPAA defines two categories:

- **Covered entities**: hospitals, clinics, health plans, clearinghouses — organizations directly providing or paying for healthcare
- **Business associates**: any vendor or partner that *touches PHI* while providing a service to a covered entity — cloud providers, analytics platforms, AI vendors, billing companies

If you're a business associate, you **must sign a BAA** with the covered entity before you can handle their PHI. The BAA legally binds you to HIPAA's security and privacy requirements.

#### What BAA-Covered Infrastructure Means

When someone says "BAA-covered infrastructure," they mean: **the cloud service or platform has signed a BAA and is contractually and technically capable of handling PHI legally.**

Not every tier of a cloud provider's offerings qualifies. The vendor has to:
- Sign the BAA with you
- Scope exactly which services are covered
- Meet HIPAA's technical safeguards (encryption, audit logs, access controls)

| Platform | BAA Available? | Notes |
|---|---|---|
| **AWS** | Yes | Most services covered — S3, RDS, SageMaker, etc. |
| **Azure** | Yes | Azure OpenAI Service is BAA-covered |
| **Google Cloud** | Yes | BigQuery, Vertex AI covered |
| **Snowflake** | Yes | Common in healthcare data platforms |
| **OpenAI (direct API)** | No | Raw OpenAI API does **not** sign BAAs — PHI cannot go through it |
| **Azure OpenAI** | Yes | Same models as OpenAI, BAA-covered — the compliant path to GPT-4 |
| **AWS Bedrock** | Yes | BAA-covered access to Claude, Llama, Titan, etc. |

#### Why This Matters for Adonis

Every time Adonis sends a claim record, diagnosis code, or clinical note to any external system, that data likely contains PHI. That means:

- **Can't use raw OpenAI API** with real claim data — no BAA, HIPAA violation
- **Must use Azure OpenAI or AWS Bedrock** for LLM calls involving PHI
- **Snowflake** is safe for storing 835/837 data — they sign BAAs
- **Vector store** holding clinical notes must be on BAA-covered infra (e.g., pgvector on RDS, or Pinecone with a BAA)
- **Every service in the pipeline** that touches PHI needs to be on the covered list

#### Compliant vs. Non-Compliant Architecture

```
NON-COMPLIANT ❌
[Claim data with PHI] → [Raw OpenAI API] → [Appeal draft]
                              ↑ No BAA — HIPAA violation

COMPLIANT ✅
[Claim data with PHI] → [Azure OpenAI (BAA-covered)] → [Appeal draft]
                              ↑ BAA signed — legal
```

#### What PHI Actually Includes

- Patient name, DOB, address
- Member ID / subscriber ID
- Diagnosis codes when linked to a patient
- Claim IDs traceable back to a patient
- Clinical notes, encounter records
- Any data that could identify a patient + relates to their health

Aggregated, de-identified data (e.g., "denial rate for CPT 99215 across all Cigna patients") is generally not PHI and doesn't require BAA-covered infra.

#### Interview Talking Point

If asked about deploying LLMs in healthcare, the right answer isn't just "we'd use GPT-4." It's:

> "We'd route LLM calls through Azure OpenAI or AWS Bedrock — both sign BAAs, so PHI never leaves a compliant environment. For de-identified or aggregated data we have more flexibility, but for anything claim- or patient-level, BAA coverage is non-negotiable."

#### One-Line Version for the Interview

> "BAA-covered infrastructure means the vendor has signed a legal contract to handle patient data under HIPAA rules — without it, sending PHI to any external service including an LLM API is a federal violation."
