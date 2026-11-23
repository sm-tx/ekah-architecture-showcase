# LLM Evaluation & Quality Governance Framework

> *Note: LLMs were used to help draft and structure this documentation from the underlying system design.*

> **Summary:** Ekah treats evals as a **mandatory release gate**. The framework (`evals/`) runs **619 curated eval cases across 45 categories** using **production-identical prompts** from `synth-brain`. It supports **provider-agnostic benchmarking** (Gemini, Claude, OpenAI, Ollama, MLX), **role-based candidate testing** (`S1.5` escalation accuracy vs. `S2` multi-turn JSON/tool compliance), **automated LLM-as-a-judge scoring**, and **parameter sweeps** capturing latency, token usage, and cost per call.

---

## 1. Design Principles

1. **Production-Identical Prompts:** Rather than maintaining separate test prompts, the eval runner imports prompt builders directly from `synth-brain/src`, ensuring eval scores reflect real production behavior.
2. **Every Feature Ships with Evals:** New modules, wiring changes, and LLM behaviors require corresponding unit/integration checks and `cases.yaml` entries before completion.
3. **Model-Agnostic Swapping:** Providers are configured declaratively in `providers.yaml` (supporting Gemini, OpenAI, Anthropic, Ollama, and Apple Silicon MLX) so new models can be benchmarked with a single CLI flag.
4. **Cost & Latency as First-Class Metrics:** Every eval run records **latency**, **token usage**, **cost per call**, and **output quality** (deterministic assertions + LLM rubric judge).

---

## 2. Role-Based Candidate Testing (`S1.5` vs. `S2`)

Because Ekah splits fast read-only execution (`S1.5`) from deep multi-turn reasoning (`S2`), models are evaluated against the exact role they will play in production:

* **Testing an `S1.5` Candidate (`--role s1_5`):**
  * Read-only queries must select the right tool and synthesize a concise answer (`≤ 2` sentences).
  * `S2` write/complex cases become **escalation checks**: the model *must* output `ESCALATE` and must **never** attempt a write tool directly.
* **Testing an `S2` Candidate (`--role s2`):**
  * Scorecard splits `S1.5` vs. `S2` pass rates.
  * Validates strict JSON schema compliance, `reasoning` field quality, `confidence` bounds (`0.0–1.0`), valid tool parameters (catching past bugs like empty `data={}`), parallel tool dispatch (`actions: [...]`), and ReAct stop conditions.

---

## 3. Coverage Across 45 Categories (619 Cases)

| Suite | Focus Area | Example Checks |
| :--- | :--- | :--- |
| **`routing`** | Topic classification, tier routing, NLU side-channels | Pronoun resolution (*"turn it off"*), security priority over home control, filler-word dropping, passive expense/intention extraction |
| **`s1_5`** | Fast one-shot tool selection & escalation | Read-only queries answered directly; write ops (`gmail_send`, `calendar_create_event`) must emit `ESCALATE`; silent fallback on timeout |
| **`s2`** | JSON compliance, multi-turn ReAct, trust | Parallel `actions` array, valid tool params, per-topic step limits, error recovery mid-chain |
| **`quality`** | Persona, tone, conciseness | No robotic phrasing, synthesis instead of raw tool dumps, graceful error messages (never stack traces) |
| **`security` & `guest`** | Threat detection, privacy, trust boundaries | Emergency bypass of budget limits; guest privacy isolation (`check_guest_privacy`) |
| **`availability` & `household`** | Proactive gating, multi-user conflicts | Deferring low-delta events when budget-constrained; resolving simultaneous cross-talk |
| **`workers` & `multi_turn`** | Background synthesis & pronoun carry-over | Morning/evening/overnight workers, multi-turn reference resolution, context window pressure |

---

## 4. Parameter Sweeps & Regression Analysis

* **Parameter Sweeps (`param_sweep.py`):** Systematically sweeps temperature, token limits, and model variants across targeted suites to find the optimal latency/quality Pareto frontier.
* **Judge + Synthesis (`--judge` / `--synthesize`):** Combines deterministic pass/fail assertions with an LLM judge for open-ended quality rubrics, then synthesizes failure clusters into an actionable summary and HTML comparison dashboard (`dashboard.html`).
