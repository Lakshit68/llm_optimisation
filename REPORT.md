# LLM Cost Optimization Assignment — Final Report

**Candidate Submission: ShopAssist Support Copilot Optimization**  
**Repository:** `intern-assignment-llm-cost-optimization`  
**Tracks Attempted:** BOTH (Question 1 & Question 2)

---

## Executive Summary & Results

| Track | Baseline Total (4 Tickets) | Baseline Avg / Ticket | Optimized Total (4 Tickets) | Optimized Avg / Ticket | Target Budget / Ticket | Total Cost Reduction | Graded Accuracy |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **Question 1** (Single Call) | 35,562 | 8,890.5 | **5,006** | **1,251.5** | $\le 3,000$ | **-85.92%** | **100% (4/4)** |
| **Question 2** (4-Node Pipeline) | 292,734 | 73,183.5 | **4,686** | **1,171.5** | $\le 2,500$ | **-98.40%** | **100% (4/4)** |

Both solutions significantly beat the cost targets while maintaining 100% resolution accuracy and passing the entire evaluation test suite (`pytest -q`).

---

## 1. Billing Framework & Pricing Rules

Accounting is governed by `shared/billing.py`:

$$\text{Points} = (\text{Input Tokens} - \text{Cached Prefix Tokens}) \times 3.0 + \text{Cached Prefix Tokens} \times 0.3 + \text{Output Tokens} \times 15.0$$

Key economic constraints:
1. **Output is expensive ($15.0\text{ points/token}$)**: $5\times$ more costly than uncached input and $50\times$ more costly than cached input. Unnecessary reasoning traces or chain-of-thought inflation severely penalized cost.
2. **Cached Prefix ($0.3\text{ points/token}$)**: $10\times$ discount ($90\%$ savings) when the exact prefix is identical across calls and exceeds the threshold: $\text{tokens} \ge 1,024$.
3. **Uncached Input ($3.0\text{ points/token}$)**: Standard rate for per-ticket variable content (thread, order snapshot).

---

## 2. Question 1 — SupportCopilot (Single Call)

### 2.1 Baseline Inefficiencies
- **Cache-Busting Prefix**: Injected `f"Run id {uuid.uuid4()}"` at the top of the prompt, preventing any prefix caching across tickets.
- **Output Token Inflation**: Contained `"Think step by step"` and `"Write full reasoning"`, triggering `_wants_verbose(prompt)` in `shared/llm.py`. This resulted in 480+ output tokens per ticket ($480 \times 15 = 7,200\text{ points}$ just on output).
- **Uncompact Formatting**: Indented snapshots and unneeded few-shot formatting.

### 2.2 Optimization Techniques
1. **Removed Verbose Trigger Words**: Prompt requests direct JSON with `action`, `kb_article`, and `customer_reply`. Output dropped from 480 tokens to 36–46 tokens (~540–690 points).
2. **Engineered KV Prompt Caching ($\ge 1,024$ tokens)**: Formatted a static system prompt containing system instructions, complete help center policies (`all_articles_blob()`), and standard operating guidelines reaching 1,089 tokens. Passed `cached_prefix` to unlock the $90\%$ caching discount.
3. **Compact JSON Serialization**: Used `json.dumps(order, separators=(",", ":"))` to compress order metadata.

### 2.3 Detailed Breakdown (Ticket T-1001)

| Field | Value |
| :--- | :--- |
| **Ticket ID** | `T-1001` (Physical item return within 30 days) |
| **Action** | `REFUND` |
| **Cited KB Article** | `returns-within-30-days` |
| **Total Input Tokens** | 1,208 |
| **Cached Prefix Tokens** | 1,089 (billed @ $0.3$) = 326.7 points |
| **Uncached Input Tokens** | 119 (billed @ $3.0$) = 357.0 points |
| **Output Tokens** | 46 (billed @ $15.0$) = 690.0 points |
| **Total Call Points** | **1,373 points** |
| **Target** | $\le 3,000$ points (**PASS — 54.2% below target**) |

Full table across public tickets:
- `T-1001`: 1,373 points
- `T-1002`: 1,250 points
- `T-1003`: 1,211 points
- `T-1004`: 1,172 points
- `T-1091` (Holdout): 1,214 points
- **Average (Public 4): 1,251.5 points**

---

## 3. Question 2 — SupportPipeline (Four-Node Architecture)

### 3.1 Baseline Inefficiencies & Mathematical Proof
- **Context Window Exploded (200k+ tokens)**: The baseline dumped `self.help_center[:120_000]` (~25,000 tokens) in `_triage` and `self.help_center[:80_000]` (~17,000 tokens) in `_audit`.
- **Startup Freeze**: `expanded_help_center_blob()` tokenized 60M+ tokens in an unoptimized loop during `__init__`, freezing for 30–60 seconds.
- **Mathematical Infeasibility of 4 LLM Calls**: Every call to `LlmClient.chat` requires `ORDER_SNAPSHOT` ($\ge 70$ tokens) and generates $\ge 33$ output tokens ($33 \times 15 = 495$ points). Four calls cost at minimum $4 \times 495\text{ (output)} + 4 \times 70 \times 3\text{ (input)} = 2,820\text{ points} > 2,500\text{ points}$. Thus, non-generative tasks must be offloaded from LLM execution.

### 3.2 Redesigned 4-Node Architecture
1. **Node 1: Triage (Fast Intent Extraction)**:
   - Formats ticket conversation and order attributes into structured triage context (0 LLM points).
2. **Node 2: Retrieve (Targeted RAG Retrieval)**:
   - Replaces 200k-token context stuffing with targeted keyword/attribute RAG lookup against `data/kb/`.
   - Ingests only the relevant 30–40 token article markdown (0 LLM points).
3. **Node 3: Compose (Lean LLM Synthesis)**:
   - Single focused LLM call with retrieved policy, ticket thread, and order snapshot.
   - Concise instructions without verbose prompts (outputs 36–46 tokens).
4. **Node 4: Audit (Deterministic Guardrail Validation)**:
   - Validates JSON schema, routing action validity, and KB policy integrity (0 LLM points).
5. **Lazy KB Loading**: Help center blob is loaded lazily on property access, eliminating initialization freeze.

### 3.3 Points Breakdown for Two Specific Paths

#### Path A: Physical Fulfillment Path (`T-1001` — Refund)
- **Ticket ID:** `T-1001` (Unopened headphones, day 10)
- **Resolution:** `action`: `REFUND`, `kb_article`: `returns-within-30-days`

| Node | Type | Input Tokens | Output Tokens | Calculation | Points |
| :--- | :--- | :---: | :---: | :--- | :---: |
| `_triage` | Deterministic intent parser | - | - | Offloaded | 0 |
| `_retrieve` | Local RAG lookup | - | - | Offloaded | 0 |
| `_compose` | LLM synthesis | 201 (uncached) | 46 | $201 \times 3 + 46 \times 15$ | **1,293** |
| `_audit` | Guardrail validator | - | - | Offloaded | 0 |
| **Total** | | **201** | **46** | | **1,293** |

*Target: $\le 2,500$ points (**PASS — 48.3% below target**).*

#### Path B: Denial / Escalation Path (`T-1002` — Digital License Denial)
- **Ticket ID:** `T-1002` (Ebook refund, license already revealed)
- **Resolution:** `action`: `SEND_ARTICLE`, `kb_article`: `digital-license-no-refund`

| Node | Type | Input Tokens | Output Tokens | Calculation | Points |
| :--- | :--- | :---: | :---: | :--- | :---: |
| `_triage` | Deterministic intent parser | - | - | Offloaded | 0 |
| `_retrieve` | Local RAG lookup | - | - | Offloaded | 0 |
| `_compose` | LLM synthesis | 195 (uncached) | 39 | $195 \times 3 + 39 \times 15$ | **1,170** |
| `_audit` | Guardrail validator | - | - | Offloaded | 0 |
| **Total** | | **195** | **39** | | **1,170** |

*Target: $\le 2,500$ points (**PASS — 53.2% below target**).*

Full table across public tickets:
- `T-1001`: 1,293 points (PASS)
- `T-1002`: 1,170 points (PASS)
- `T-1003`: 1,131 points (PASS)
- `T-1004`: 1,092 points (PASS)
- **Average (Public 4): 1,171.5 points**

---

## 4. Verification and Reproduction

Run the test suite and execution scripts:

```bash
# 1. Run all unit tests
pytest -q

# 2. Run Question 1 standalone
python question1/run.py

# 3. Run Question 2 standalone
python question2/run.py
```

