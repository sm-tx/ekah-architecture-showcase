# System Architecture

> *Note: LLMs were used to help draft and structure this documentation from the underlying system design.*

> **Summary:** Ekah is organized as a **3-module event-driven pipeline** (`Sensory Router → Synth-Brain → Auditory Router`) that decouples multi-surface ingress/egress from cognitive processing. Inside `Synth-Brain`, events pass through a **latency-tiered routing pipeline** (sub-ms deterministic rules → `<50ms` local NLU & noise gating → `<500ms` fast lane → one-shot `S1.5` → multi-turn ReAct `S2`), backed by **proactive background workers**, **budget-aware overnight deferral**, and **graceful offline degradation**.

---

## 1. Three-Module Pipeline

Ekah avoids tight coupling between messaging/voice channels and LLM reasoning by normalizing all inputs into `SensoryEvent` objects and all outputs into `DeliveryIntent` objects:

```text
┌─────────────────────────── SENSORY ROUTER (Ingress) ──────────────────────────┐
│  Messaging (Telegram, WhatsApp) ──┐                                           │
│  Voice (Mic, Room Speakers, Phone)├──► Normalizes into SensoryEvent           │
│  Tablet Chat & Event WebSockets ──┤    Attaches ProvenanceEnvelope            │
│  Background Pollers (Email, Cal) ─┘    (sender, trust_tier, source, room)     │
└───────────────────────────────────────┬───────────────────────────────────────┘
                                        │ SensoryEvent
                                        ▼
┌───────────────────────────── SYNTH-BRAIN (Core) ──────────────────────────────┐
│  1. Tier 0 / 0.5: RuleEngine (sub-ms) & MacroMatcher / Fast Lane (<500ms)     │
│  2. Local NLU (<50ms): Intent, entities, sentiment, & passive side-channels   │
│  3. Gate: ThalamicGate (noise filter) → GravityScorer (priority / delta)      │
│  4. Orchestrator: Routes to S1.5 (fast, read-only) or S2 (deep, multi-turn)   │
│  5. Authority: ToolCallGate (TrustTier × Sensitivity) + PendingApprovals      │
│  6. Background Workers: Compilers, ProactiveWakeup, OvernightQueue, Memory    │
└───────────────────────────────────────┬───────────────────────────────────────┘
                                        │ DeliveryIntent
                                        ▼
┌─────────────────────────── AUDITORY ROUTER (Egress) ──────────────────────────┐
│  Routes DeliveryIntent to the right surface (Telegram, WhatsApp, Room TTS,    │
│  Phone, Tablet WS/HTTP, Device Action, or Silent Log).                        │
│  Handles TTS streaming, volume ducking, barge-in detection, & approval UI.    │
└───────────────────────────────────────────────────────────────────────────────┘
```

### Why This Separation Matters (Product & UX)
* **Channel-Agnostic Agent:** Adding a new surface (e.g., voice vs. messaging vs. tablet) only requires an ingress/egress adapter—the core reasoning, authority, and memory layers remain identical.
* **Provenance by Default:** Every event carries a `ProvenanceEnvelope` (who asked, from which channel/room, at what trust level) so downstream policy gates never have to guess.

---

## 2. Latency-Tiered Cognitive Routing

The `Orchestrator` (`gate/orchestrator.py`) routes each event to the cheapest, fastest tier capable of handling it accurately:

| Tier | Component | Latency Target | LLM Used? | Responsibility |
| :--- | :--- | :--- | :--- | :--- |
| **Tier 0** | `RuleEngine` | `< 1 ms` | No | Deterministic rules, hard security triggers, state deltas |
| **Tier 0.5** | `MacroMatcher` / `ha_fast_lane` | `< 500 ms` | No | User-taught macro automations and direct device queries/controls |
| **NLU + Gate** | Local NLU + `ThalamicGate` + `GravityScorer` | `< 50 ms` | Distilled local model | Intent classification, entity extraction, noise filtering, priority scoring |
| **System 1.5** | `ConversationalLobe` | `~500–1200 ms` | Fast model (e.g., Flash) | One-shot `generate_with_tools` for **read-only** queries & concise synthesis; emits `ESCALATE` on writes |
| **System 2** | `FrontalLobe` | Multi-second | Reasoning model (e.g., Pro) | Multi-turn ReAct loop (`tool_call`, `observe`, `alert_user`, `defer`) with parallel tool dispatch (`asyncio.gather`) |

### Key Routing Safeguards
* **Read-Only Enforcement in S1.5:** `ConversationalLobe` is structurally prevented from executing write tools (`gmail_send`, `calendar_create_event`, `ha_call_service`, etc.). Any write intent triggers `ESCALATE` to System 2.
* **Per-Topic Step Limits in S2:** The ReAct loop enforces domain-specific step budgets (e.g., max 3 steps for home control vs. up to 6 steps for multi-source research) to prevent runaway token loops.
* **Silent Timeout Recovery:** If `S1.5` hits a provider timeout or error, the orchestrator silently escalates to `S2` rather than surfacing a raw error to the user.

---

## 3. NLU Side-Channel Detection

In parallel with primary intent routing, the local NLU extracts passive signals ("side-channels") from natural conversation without interrupting the user's primary request:
* **Example:** *"I had a salad for lunch, turn off the kitchen lights"* → Primary intent is `HA_CONTROL` (turn off lights), while the side-channel logs `detected_health=True` (meal).
* **Other Side-Channels:** Intentions (*"I should exercise more"*), expenses (*"spent £45"*), contact mentions, open threads, and curiosities.

---

## 4. Proactive Loops, Budget Deferral & Memory

* **Noise Filtering (`ThalamicGate` & `GravityScorer`):** Background events (polls, state changes) are scored for delta/gravity. Low-signal noise is dropped before reaching an LLM.
* **Budget-Aware Deferral (`OvernightQueue`):** When API/token budgets are constrained, low-delta events (e.g., `delta 0.2`) are deferred to an overnight batch window (2–5 AM), while high-delta emergencies (e.g., `delta 0.9` security alerts) bypass the budget gate immediately.
* **Behavioral Proactive Outreach (`ProactiveWakeup` & `CognitiveAvailability`):** Proactive suggestions are gated on user availability and context rather than rigid cron schedules, reducing notification fatigue.
* **Context Compilation (`KnowledgeCompiler`):** A 30-second background loop compiles per-user context snapshots so `S1.5` and `S2` start with compact, relevant state instead of raw database dumps.
* **Multi-User Conflict & Graceful Degradation:** `ConflictDetector` / `CrossTalkDetector` handle simultaneous multi-user inputs, while `DegradationMonitor` tracks service health to keep core flows operational during partial outages.
