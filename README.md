# magicpin AI Challenge — Merchant AI Assistant ("Vera")

## Executive Summary & Challenge Goal

**magicpin** is India's largest local-commerce platform, connecting ~100,000 merchant partners (restaurants, salons, gyms, dentists, pharmacies, etc.) across 50+ cities with millions of customers. 

**Vera** is magicpin's merchant-AI assistant operating on WhatsApp. Vera talks to 6,000–10,000 merchants daily, helping them optimize their Google Business Profile (GBP), run marketing campaigns, manage offers, and respond to customer queries.

### The Objective
Build an AI assistant that engages merchants on WhatsApp using the **4-Context Framework** — achieving higher response rates, eliminating auto-reply pollution, handling intent handoffs flawlessly, and delivering category-correct, specific, compelling communications.

---

## Workspace Deep-Dive: File & Component Analysis

This repository contains all context, specifications, datasets, and testing tools for the challenge. Below is a comprehensive analysis of every key file:

```
magicpin-ai-challenge/
├── challenge-brief.md          # Primary specification: product background, 4-context framework, rubric
├── challenge-testing-brief.md  # Harness specification: 5 HTTP REST endpoints, test phases, payload schemas
├── engagement-design.md        # Architectural proposal: 4-context composition design & trigger sources
├── engagement-research.md      # Legacy analysis: production Vera state, MCP servers, gaps, adapter strategy
├── judge_simulator.py          # Local judge & test harness powering automated evaluation & scoring
├── dataset/                    # Synthetic base dataset (categories, merchants, customers, triggers)
│   ├── generate_dataset.py     # Deterministic dataset generation script
│   ├── categories/*.json       # 5 Category contexts (dentists, salons, restaurants, gyms, pharmacies)
│   ├── merchants/*.json        # 50 Merchant contexts (10 per category)
│   ├── customers/*.json        # 200 Customer profiles attached to merchants
│   └── triggers/*.json         # 100 Internal & External event triggers
└── examples/                   # Reference implementations & scenario studies
    ├── api-call-examples.md    # Detailed walkthrough of all 5 API endpoints & payload exchanges
    └── case-studies.md         # 4 deep-dive case studies with scoring breakdowns
```

### 1. `challenge-brief.md` (Product & Core Specifications)
* **Purpose**: The foundational document explaining what Vera is, what problem you are solving, and how your bot will be judged.
* **Key Takeaways**:
  * Vera handles both **Merchant-Facing** (`send_as: "vera"`) and **Customer-Facing** (`send_as: "merchant_on_behalf"`) messages.
  * Today's Vera suffers from 4 major pain points:
    1. **Auto-reply pollution** (40-70% of responses are canned business auto-replies).
    2. **Intent-handoff failures** (asking qualifying questions when merchant says "I want to join").
    3. **Generic discount copy** ("10% off" fails; specific "Haircut @ ₹99" wins).
    4. **Low engagement frequency** (relying only on rare functional reminders instead of curiosity/knowledge drivers).

### 2. `challenge-testing-brief.md` (Testing & API Harness Contract)
* **Purpose**: Defines the technical REST API interface between magicpin's automated testing harness and your bot.
* **Key Takeaways**:
  * If building a live server, your bot must expose 5 HTTPS/HTTP JSON endpoints:
    1. `POST /v1/context` — Ingests category, merchant, customer, or trigger context updates.
    2. `POST /v1/tick` — Periodic trigger check; bot decides whether to initiate proactive messages.
    3. `POST /v1/reply` — Synchronous turn handler for incoming merchant/customer replies.
    4. `GET /v1/healthz` — Liveness & loaded-context probe.
    5. `GET /v1/metadata` — Team metadata, model info, and versioning.
  * Test execution flows across 5 phases: Warmup -> 60-min Simulated Window -> Adaptive Context Injection -> Replay Test (Top 10) -> Scoring.

### 3. `engagement-design.md` & `engagement-research.md` (Architecture & Production Gaps)
* **Purpose**: Documents the structural rationale behind moving away from ad-hoc scripts to a unified 4-Context composer.
* **Key Takeaways**:
  * Production Vera relies on `vera-mcp` and `merchant-support-mcp`.
  * Existing Vera already maintains parts of `MerchantContext`, but lacks a structured `CategoryContext`, normalized `TriggerContext`, and unified `EngagementComposer`.

### 4. `judge_simulator.py` (Local Evaluation Engine)
* **Purpose**: A self-contained Python script to test your bot locally against LLM-based evaluation rubrics before submitting.
* **Key Takeaways**:
  * Supports OpenAI, Anthropic, Gemini, DeepSeek, Groq, and Ollama.
  * Evaluates messages on a 50-point scale and runs multi-turn replay simulations.

### 5. `dataset/` (Base Dataset)
* **Purpose**: Provides realistic synthetic data across 5 verticals:
  * **Categories**: `dentists`, `salons`, `restaurants`, `gyms`, `pharmacies`.
  * **Merchants**: 50 detailed profiles (10 per category) with identity, subscription, GBP performance metrics, active offers, and signals.
  * **Customers**: 200 customer relationship profiles (visit history, preferences, consent, lapse state).
  * **Triggers**: 100 external (news, weather, research digests) and internal (perf spikes/dips, recalls, review themes) events.

---

## The 4-Context Framework

Every message composed by Vera follows the pure signature:

$$\text{Message} = \text{compose}(\text{CategoryContext}, \text{MerchantContext}, \text{TriggerContext}, \text{CustomerContext?})$$

```
  ┌──────────────────────────────────────────────────────────┐
  │ CategoryContext  : Vertical knowledge, voice, peer stats │
  │ MerchantContext  : Business identity, perf, offers, logs │ ──► [ Composer Engine ] ──► ComposedMessage
  │ TriggerContext   : Event source, urgency, payload        │                              - body
  │ CustomerContext? : (Optional) Customer info, preferences │                              - cta
  └──────────────────────────────────────────────────────────┘                              - send_as
                                                                                            - suppression_key
                                                                                            - rationale
```

### Context Definitions:
1. **CategoryContext**: Slow-changing vertical knowledge pack (e.g., dentist voice taboos like "no guaranteed cures", standard service pricing like "Dental Cleaning @ ₹299", research digests).
2. **MerchantContext**: Business snapshot (GBP CTR, views, calls, subscription days remaining, active offers, past conversation tags).
3. **TriggerContext**: The specific reason for messaging *right now* (e.g., heatwave warning, JIDA research digest release, 6-month recall due, performance spike).
4. **CustomerContext** *(Optional)*: Populated when Vera speaks *on behalf of the merchant* to a customer (e.g., patient Priya due for a 6-month dental checkup).

---

## Deliverables & Submission Requirements

Candidates must provide the following core files:

1. **`bot.py`**:
   * Contains the primary logic: `compose(category, merchant, trigger, customer=None) -> dict`.
   * Fast response time (< 30s per invocation), deterministic outputs (LLM temperature = 0).
   * Can also serve as the FastAPI web server exposing `/v1/*` endpoints if running in live harness mode.
2. **`submission.jsonl`**:
   * Exactly **30 lines** of outputs corresponding to the canonical test set pairs.
   * Each JSON line includes: `test_id`, `body`, `cta`, `send_as`, `suppression_key`, and `rationale`.
3. **`README.md`**:
   * Max 1-2 pages explaining your architectural approach, tradeoffs, prompt strategy, and context usage.
4. **`conversation_handlers.py`** *(Optional / Bonus)*:
   * Multi-turn interaction handler: `respond(state, merchant_message) -> dict`.

---

## Evaluation Rubric (50 Points Total)

Submissions are scored across 5 dimensions (0–10 points each):

| Dimension | What the Judge Evaluates | Key Winning Strategy |
|---|---|---|
| **1. Specificity** | Concrete, verifiable facts (numbers, dates, citations, specific prices). | Use exact numbers ("2,100-patient trial", "Haircut @ ₹99", "6,777 missed searches"). Avoid generic "10% off". |
| **2. Category Fit** | Tone, vocabulary, and compliance rules matching the vertical. | Dentists receive clinical/peer tone; Salons receive aesthetic/trend tone. Adhere to taboos. |
| **3. Merchant Fit** | Tailored to merchant identity, locality, performance deltas, and language preference. | Highlight local area ("Lajpat Nagar"), address language preference (e.g., Hinglish/Hing-En mix). |
| **4. Trigger Relevance** | Clear explanation of *why now* based on the trigger. | Explicitly tie the message hook to the trigger event (e.g., "JIDA Oct issue landed...", "Heatwave alert in Delhi..."). |
| **5. Engagement Compulsion** | High likelihood of response using compulsion levers. | Incorporate curiosity, loss aversion, social proof, and a clear single binary CTA (YES/STOP). |

### Penalties & Hard Failures:
* **Hallucination Penalty**: Citing fake research, fake competitor names, or prices not in the catalog.
* **Auto-Reply Loop Penalty**: Failing to exit when encountering repeated canned business replies.
* **Multi-CTA Penalty**: Including more than one primary call-to-action in a single message.
* **Tone Mismatch**: Using hype/promotional tone for clinical verticals (Dentists/Doctors).

---

## Compulsion Levers & Anti-Patterns

### The 8 Compulsion Levers (Use 1–2 per message)
1. **Verifiable Specificity**: Exact stats, dates, citations.
2. **Loss Aversion**: "You're missing 45 searches in Lajpat Nagar."
3. **Social Proof**: "3 dental clinics in Delhi updated their fluoride recall this week."
4. **Effort Externalization**: "I've drafted the post for you — just reply YES to publish."
5. **Curiosity Hook**: "Want to see how your listing ranks against local peers?"
6. **Reciprocity**: "I noticed your GBP missing business hours, so I prepared an update."
7. **Asking the Merchant**: "What was your most requested treatment this week?"
8. **Single Binary CTA**: Clear YES/STOP or single option response.

### Anti-Patterns to Avoid
* ❌ Generic discount offers ("Flat 20% off everything").
* ❌ Multiple CTAs ("Reply YES for X, NO for Y, MAYBE for Z").
* ❌ Buried call-to-action (CTA must be the final clear sentence).
* ❌ Re-introducing the bot after initial touchpoint.
* ❌ Ignoring language preferences (e.g., sending formal English to a Hinglish preference merchant).

---

## Step-by-Step Build & Implementation Plan

### Step 1: Core Composer Setup (`bot.py`)
1. Parse incoming `category`, `merchant`, `trigger`, and `customer` context dictionaries.
2. Construct a structured system prompt encoding vertical constraints, voice taboos, and compulsion levers.
3. Call an LLM (Claude, GPT, Gemini, etc.) with `temperature=0`.
4. Validate output format (body, CTA, suppression key, rationale).

### Step 2: Multi-Turn & Auto-Reply Handling
1. Implement detection logic for canned WhatsApp auto-replies (e.g., verbatim matching repeated messages).
2. Exit gracefully after detecting auto-replies to save turns.
3. Detect explicit intent (e.g., "I want to join") and immediately trigger the action phase without re-qualifying.

### Step 3: Local Testing & Validation
1. Start your local server or standalone runner.
2. Configure `judge_simulator.py` with your LLM provider and API key.
3. Run `python judge_simulator.py` to evaluate your bot against test scenarios.
4. Review the breakdown across all 5 dimensions and adjust prompt templates accordingly.

---

## Quick Start & Testing

To test your implementation locally using the built-in judge simulator:

```bash
# 1. Install dependencies (FastAPI, uvicorn, requests, etc.)
pip install fastapi uvicorn requests

# 2. Start your bot endpoint (if using live API mode)
python bot.py

# 3. Configure API key and run judge simulator
export LLM_API_KEY="your-api-key"
python judge_simulator.py
```
# vera-engagement-composer
