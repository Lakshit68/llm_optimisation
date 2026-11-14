# Question 1 — SupportCopilot Optimization Report

## 1. Executive Summary

| Metric | Baseline | Optimized | Target | Improvement |
| :--- | :--- | :--- | :--- | :--- |
| **Total Points (4 Public Tickets)** | 35,562 | **5,006** | ≤ 12,000 | **-85.92%** |
| **Average Points / Ticket** | 8,890.5 | **1,251.5** | ≤ 3,000 | **-85.92%** |
| **Graded Field Accuracy** | 100% | **100%** | 100% | Maintained |
| **Holdout (T-1091) Accuracy** | 100% | **100%** | 100% | Maintained |

---

## 2. Baseline Inefficiency Analysis

Profiling the baseline implementation in `question1/agent.py` identified three major cost drivers:

1. **Cache-Busting Dynamic Prefix (`uuid.uuid4()`)**:
   - The baseline system prompt inserted `f"Run id {uuid.uuid4()}"` at line 65.
   - Because the prefix changed on every single invocation, the system prompt never qualified for KV prompt caching (billing at 0.3 points/token), forcing all ~450+ system tokens to be billed as uncached input (3.0 points/token) on every call.

2. **Verbose Reasoning Triggers (Output Bloat)**:
   - The baseline instructions included phrases like `"Think step by step"` and `"Write full reasoning"`.
   - In `shared/llm.py`, these phrases trigger `_wants_verbose(prompt)`, causing the model simulator to generate a verbose internal trace of 480+ tokens.
   - At 15 points per output token, output alone cost $480 \times 15 = 7,200\text{ points}$ per ticket.

3. **Indented JSON Serialization**:
   - Ticket order snapshots were formatted with `indent=2`, adding unnecessary whitespace and newline tokens to every user message.

---

## 3. Optimization Techniques

1. **Elimination of Verbose Prompts**:
   - Replaced verbose phrasing with concise instructions requesting direct JSON output with `action`, `kb_article`, and `customer_reply`.
   - Output tokens dropped from **480+ tokens** to **36–46 tokens**, reducing output cost from 7,200 points to 540–690 points (**>91% output cost savings**).

2. **KV Prompt Caching Activation ($\ge 1024$ tokens)**:
   - Designed a static, stable system prompt containing system instructions, the complete help center policies (`all_articles_blob()`), and standard operating compliance guidelines.
   - Ensured the static prefix exceeds the 1,024-token threshold (`count_tokens(prefix) == 1089 tokens`).
   - Removed all dynamic values (`uuid.uuid4()`).
   - Passed `cached_prefix` to `llm.chat()`. All 1,089 prefix tokens are billed at **0.3 points/token** instead of 3.0 points/token (**90% discount on common prefix**).

3. **Compact Payload Serialization**:
   - Serialized the order snapshot using `json.dumps(order, separators=(",", ":"))`, minimizing uncached input tokens to 100–119 tokens per ticket.

---

## 4. Per-Call Token and Points Breakdown

### Billing Formulation
$$\text{points} = \text{uncached\_input} \times 3 + \text{cached\_input} \times 0.3 + \text{output} \times 15$$

### Breakdown for Ticket T-1001 (Refund, physical item, unopened)

| Call Label | Cached Input Tokens | Uncached Input Tokens | Total Input Tokens | Output Tokens | Calculation | Total Points |
| :--- | :---: | :---: | :---: | :---: | :--- | :---: |
| `handle_ticket` | 1,089 | 119 | 1,208 | 46 | $119 \times 3 + 1089 \times 0.3 + 46 \times 15$ | **1,373** |

*Target: $\le 3,000$ points. Ticket T-1001 achieved 1,373 points (54.2% below target).*

### Complete Run Breakdown (All Public Tickets + Holdout)

| Ticket ID | Scenario | Action | KB Article | Cached In | Uncached In | Output | Points | Target | Status |
| :--- | :--- | :--- | :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| **T-1001** | Physical return within 30 days | `REFUND` | `returns-within-30-days` | 1,089 | 119 | 46 | **1,373** | $\le 3,000$ | PASS |
| **T-1002** | Digital item, revealed license | `SEND_ARTICLE` | `digital-license-no-refund` | 1,089 | 113 | 39 | **1,250** | $\le 3,000$ | PASS |
| **T-1003** | Damaged customized merchandise | `ESCALATE_TIER2` | `custom-items-no-replace` | 1,089 | 100 | 39 | **1,211** | $\le 3,000$ | PASS |
| **T-1004** | Damaged stock item $\le 7$ days | `REPLACE` | `damage-replacement-7-days` | 1,089 | 102 | 36 | **1,172** | $\le 3,000$ | PASS |
| **T-1091** (Holdout) | Subscription cooling off | `SEND_ARTICLE` | `subscription-cooling-off` | 1,089 | 106 | 38 | **1,214** | - | PASS |

---

## 5. How to Run

```bash
# Run pytest verification
pytest -q tests/test_question1.py

# Run standalone resolution & points summary
python question1/run.py
```
