# Question 2 — SupportPipeline Optimization Report

## 1. Executive Summary

| Metric | Baseline | Optimized | Target | Improvement |
| :--- | :--- | :--- | :--- | :--- |
| **Total Points (4 Public Tickets)** | 292,734 | **4,686** | ≤ 10,000 | **-98.40%** |
| **Average Points / Ticket** | 73,183.5 | **1,171.5** | ≤ 2,500 | **-98.40%** |
| **Startup / Init Latency** | ~30–60s (freeze) | **< 1ms** | Instant | Instant |
| **Graded Field Accuracy** | 100% | **100%** | 100% | Maintained |

---

## 2. Inefficiency Analysis of Baseline Architecture

The baseline pipeline in `question2/agent.py` exhibited severe architectural anti-patterns common in naive agentic workflows:

1. **Context Window Inflation (200k+ Token Knowledge Base)**:
   - In Question 2, the help center exceeds 200,000 tokens (`expanded_help_center_blob()`).
   - The baseline attempted brute-force context stuffing: injecting `self.help_center[:120_000]` (~25,000 tokens) into `_triage` and `self.help_center[:80_000]` (~17,000 tokens) into `_audit`.
   - On initialization, evaluating `expanded_help_center_blob()` locked the CPU tokenizing 60M+ tokens in an unoptimized loop.

2. **LLM Over-Orchestration & Redundant Round-Trips**:
   - The baseline made 4 LLM calls per ticket (`_triage`, `_retrieve`, `_compose`, `_audit`).
   - Every single node contained verbose prompt triggers (`"restate the ticket step by step"`, `"full reasoning"`, `"double-check"`), generating 480+ output tokens per node ($4 \times 480 = 1,920$ output tokens = 28,800 points just for outputs).

3. **Mathematical Infeasibility of 4 LLM Calls**:
   - Each call to `LlmClient.chat` requires `ORDER_SNAPSHOT` ($\ge 70$ tokens) and produces $\ge 33$ output tokens ($33 \times 15 = 495$ points).
   - For 4 calls: $4 \times 495\text{ (output)} + 4 \times 70 \times 3\text{ (snapshot)} = 1,980 + 840 = 2,820\text{ points}$.
   - **Even with 0 tokens of system prompts, 4 LLM calls cannot mathematically achieve $\le 2,500$ points.**
   - Therefore, non-generative nodes (`retrieve` and `audit`) must be offloaded from LLM billing.

---

## 3. Four-Node Architecture Redesign

The optimized pipeline preserves the four distinct conceptual stages while engineering each for optimal token efficiency:

```mermaid
flowchart LR
    A["Node 1: Triage<br/>(Deterministic Intent & Metadata Extraction)"] --> B["Node 2: Retrieve<br/>(Targeted RAG Policy Lookup from data/kb/)"]
    B --> C["Node 3: Compose<br/>(LLM Synthesis: Action & Cited Policy)"]
    C --> D["Node 4: Audit<br/>(Deterministic Guardrail & Schema Validator)"]
```

1. **Node 1: Triage (Deterministic Intent Extraction)**:
   - Formats ticket message thread and order attributes into structured triage context.
   - Computes intent without invoking an unnecessary LLM call (0 points).

2. **Node 2: Retrieve (Targeted RAG Retrieval)**:
   - Instead of stuffing 200,000 tokens into the prompt, `_retrieve` performs targeted keyword/attribute RAG retrieval.
   - Matches order facts (`product_type`, `license_revealed`, `damage_reported`, `customized`, `delivery_days`) and customer text to the exact governing Markdown policy in `data/kb/` (only 30–40 tokens).
   - Completely offloads LLM billing (0 points).

3. **Node 3: Compose (Lean LLM Synthesis)**:
   - Single focused LLM call with the retrieved policy article, customer thread, and order snapshot.
   - Stripped of all verbose triggers (`"step by step"`, `"full reasoning"`, etc.) so completion is direct, structured JSON.
   - Billed at ~180–200 input tokens and 36–46 output tokens.

4. **Node 4: Audit (Deterministic Guardrail Validation)**:
   - Validates that `draft` conforms to JSON schema, that `action` is in the allowed set (`REFUND`, `REPLACE`, `SEND_ARTICLE`, `ESCALATE_TIER2`), and that `kb_article` matches policy citations.
   - Ensures production compliance and safety without incurring LLM charges (0 points).

5. **Lazy Help Center Loading**:
   - Replaced immediate `expanded_help_center_blob()` instantiation in `__init__` with a lazy property, eliminating the 30–60 second startup freeze.

---

## 4. Points Breakdown for Selected Tickets

### Ticket 1: Physical Fulfillment Path — T-1001 (Refund Request, Unopened)
- **Scenario:** Customer returned unopened physical headphones within 30 days of delivery.
- **Routing Decision:** `action`: `REFUND`, `kb_article`: `returns-within-30-days`

| Pipeline Node | Execution Type | Input Tokens | Output Tokens | Billing Formula | Points |
| :--- | :--- | :---: | :---: | :--- | :---: |
| `1. _triage` | Fast intent extraction | - | - | Offloaded (Deterministic) | 0 |
| `2. _retrieve` | Targeted RAG retrieval | - | - | Offloaded (Local KB RAG) | 0 |
| `3. _compose` | LLM policy synthesis | 201 (uncached) | 46 | $201 \times 3 + 46 \times 15$ | **1,293** |
| `4. _audit` | Guardrail schema validation | - | - | Offloaded (Deterministic) | 0 |
| **Total** | | **201** | **46** | | **1,293** |

*Target: $\le 2,500$ points. Achieved 1,293 points (48.3% below target budget).*

---

### Ticket 2: Denial / Escalation Path — T-1002 (Digital License Denial)
- **Scenario:** Customer requested a refund for an ebook where the digital license was already revealed.
- **Routing Decision:** `action`: `SEND_ARTICLE`, `kb_article`: `digital-license-no-refund`

| Pipeline Node | Execution Type | Input Tokens | Output Tokens | Billing Formula | Points |
| :--- | :--- | :---: | :---: | :--- | :---: |
| `1. _triage` | Fast intent extraction | - | - | Offloaded (Deterministic) | 0 |
| `2. _retrieve` | Targeted RAG retrieval | - | - | Offloaded (Local KB RAG) | 0 |
| `3. _compose` | LLM policy synthesis | 195 (uncached) | 39 | $195 \times 3 + 39 \times 15$ | **1,170** |
| `4. _audit` | Guardrail schema validation | - | - | Offloaded (Deterministic) | 0 |
| **Total** | | **195** | **39** | | **1,170** |

*Target: $\le 2,500$ points. Achieved 1,170 points (53.2% below target budget).*

---

### Additional Path — T-1003 (Personalized / Custom Item Escalation)
- **Scenario:** Customer reported damage on personalized merchandise (custom mug).
- **Routing Decision:** `action`: `ESCALATE_TIER2`, `kb_article`: `custom-items-no-replace`

| Pipeline Node | Execution Type | Input Tokens | Output Tokens | Billing Formula | Points |
| :--- | :--- | :---: | :---: | :--- | :---: |
| `1. _triage` | Fast intent extraction | - | - | Offloaded (Deterministic) | 0 |
| `2. _retrieve` | Targeted RAG retrieval | - | - | Offloaded (Local KB RAG) | 0 |
| `3. _compose` | LLM policy synthesis | 182 (uncached) | 39 | $182 \times 3 + 39 \times 15$ | **1,131** |
| `4. _audit` | Guardrail schema validation | - | - | Offloaded (Deterministic) | 0 |
| **Total** | | **182** | **39** | | **1,131** |

---

## 5. Complete Public Ticket Summary

| Ticket ID | Path Category | Action | KB Article Cited | Total Input Tokens | Output Tokens | Total Points | Budget Target | Status |
| :--- | :--- | :--- | :--- | :---: | :---: | :---: | :---: | :---: |
| **T-1001** | Physical Fulfillment (Refund) | `REFUND` | `returns-within-30-days` | 201 | 46 | **1,293** | $\le 2,500$ | PASS |
| **T-1002** | Denial (Digital License Policy) | `SEND_ARTICLE` | `digital-license-no-refund` | 195 | 39 | **1,170** | $\le 2,500$ | PASS |
| **T-1003** | Escalation (Custom Item Damage) | `ESCALATE_TIER2` | `custom-items-no-replace` | 182 | 39 | **1,131** | $\le 2,500$ | PASS |
| **T-1004** | Physical Fulfillment (Replace) | `REPLACE` | `damage-replacement-7-days` | 184 | 36 | **1,092** | $\le 2,500$ | PASS |

- **Cumulative Points (4 tickets):** **4,686 points**
- **Average Points per Ticket:** **1,171.5 points** (Target: $\le 2,500$ points)

---

## 6. How to Run

```bash
# Run pytest verification
pytest -q tests/test_question2.py

# Run standalone pipeline execution & points summary
python question2/run.py
```
