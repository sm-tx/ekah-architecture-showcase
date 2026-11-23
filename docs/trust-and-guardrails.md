# Trust, Authority & Human-in-the-Loop Guardrails

> *Note: LLMs were used to help draft and structure this documentation from the underlying system design.*

> **Summary:** Ekah treats LLM outputs as **untrusted proposals**. Every tool call must pass through `ToolCallGate` (`authority/tool_gate.py`), which evaluates a deterministic **`TrustTier × ToolSensitivity`** policy matrix outside the prompt. High-impact or external actions trigger **`NEEDS_CONFIRMATION`**, pausing execution behind a **6-character approval token (5-minute TTL)** until the user explicitly approves.

---

## 1. Deterministic Policy Matrix (`ToolCallGate`)

Rather than relying on system-prompt instructions to prevent unauthorized actions, Ekah enforces permissions in code before any tool executes.

### Trust Tiers (Who Is Asking?)
1. **`OWNER`** — Full administrative and personal access.
2. **`RESIDENT`** — Standard household member; full read/write-safe access, confirmation required for external/destructive actions.
3. **`TRUSTED_GUEST`** — Scoped local control (e.g., lights/climate with confirmation); zero personal data access.
4. **`GUEST`** — Read-only Q&A only; no control or personal data.
5. **`UNTRUSTED`** — Pre-pairing/unknown sender; all tools denied.

### Tool Sensitivity Tiers (What Is the Blast Radius?)
* `READ_ONLY` — Query operations (status, search, calendar read)
* `WRITE_SAFE` — Reversible internal writes (reminders, memory notes)
* `WRITE_DESTRUCTIVE` — Irreversible state changes
* `EXTERNAL` — Outbound side effects (sending email, creating external calendar invites)
* `HOME_CONTROL` — Physical environment actuation (lights, locks, climate)
* `PERSONAL_DATA` — Private user records (email, calendar, contacts)
* `OWNER_ONLY` — System configuration, skill installation, entity tracking

### Policy Matrix (`TrustTier × ToolSensitivity`)

| Trust Tier | `READ_ONLY` | `WRITE_SAFE` | `HOME_CONTROL` | `WRITE_DESTRUCTIVE` | `EXTERNAL` | `PERSONAL_DATA` | `OWNER_ONLY` |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **`OWNER`** | `ALLOWED` | `ALLOWED` | `ALLOWED` | `ALLOWED` | `ALLOWED` | `ALLOWED` | `ALLOWED` |
| **`RESIDENT`** | `ALLOWED` | `ALLOWED` | `ALLOWED` | `CONFIRM` | `CONFIRM` | `CONFIRM` | `DENIED` |
| **`TRUSTED_GUEST`** | `ALLOWED` | `CONFIRM` | `CONFIRM` | `DENIED` | `DENIED` | `DENIED` | `DENIED` |
| **`GUEST`** | `ALLOWED` | `DENIED` | `DENIED` | `DENIED` | `DENIED` | `DENIED` | `DENIED` |
| **`UNTRUSTED`** | `DENIED` | `DENIED` | `DENIED` | `DENIED` | `DENIED` | `DENIED` | `DENIED` |

*(Note: `CONFIRM` = `NEEDS_CONFIRMATION`)*

---

## 2. Human-in-the-Loop Approval Flow (`PendingApprovals`)

When `ToolCallGate` returns `NEEDS_CONFIRMATION`:
1. **Execution Pauses:** `ActionDispatcher` (`frontal/actions.py`) halts the tool call and stores the pending action in `PendingApprovals` (`authority/pending_approvals.py`) with a **6-character confirmation token** and a **5-minute TTL**.
2. **User Prompted:** `AuditoryRouter` sends a concise confirmation alert to the user's active messaging surface (e.g., Telegram approval callback) detailing the exact action and parameters.
3. **Explicit Resolution:**
   * **Approved:** Execution resumes for that specific tool call.
   * **Denied or Expired (5-min TTL):** The action is safely dropped and logged—never retried silently.

---

## 3. Identity, Liveness & Degradation Safeguards

* **Multi-Modal Identity & Guest Onboarding (`identity/`):** Maps messaging IDs and voice biometrics (`VoiceIdentityManager` with 3-tier matching) to a `TrustTier` via `TrustResolver`, with explicit `check_guest_privacy()` guards so guest sessions cannot access resident context.
* **Liveness Detection (`LivenessDetector`):** Uses spectral flatness and media playback state to reject spoofed/TV audio from triggering voice commands.
* **Graceful Degradation (`DegradationMonitor`):** Monitors downstream service health and degrades cleanly to offline/read-only modes without crashing the core conversational loop.
