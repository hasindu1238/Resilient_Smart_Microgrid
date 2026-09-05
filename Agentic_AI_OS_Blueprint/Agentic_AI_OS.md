---
output:
  word_document: default
  html_document: default
  pdf_document: default
---
# Step 3 — Agentic AI OS Blueprint

**Project:** Resilient Smart Microgrid — *The Village Micro-grid Brain*
**Deliverable:** Step 3 of the Mini Ideathon — multi-agent workflow, `Inputs → Monitor → Planner → Decision/Validator → Action`, with a continuous feedback loop.
**Companion:** `Step3_Agentic_AI_OS_Blueprint.html` — the visual blueprint. Open it in a browser; every diagram here is drawn there.

> **The one line to remember:** *Generative AI explains. Deterministic logic decides.*

---

## 1. The required flow, instantiated

| # | Brief's stage | Our agent | What it actually does |
|---|---|---|---|
| 1 | **Inputs** | *(IoT layer)* | Signed JSON telemetry from solar PV, battery/BMS, grid sensor, 4 load channels, cold-chain probe |
| 2 | **Monitor** | **Monitor** | Thresholds + RoCoF check + bounded outlier score → emits a state vector |
| 3 | **Planner** | **Planner** | Forecasts battery depletion, generates 3–4 ranked candidate actions, selects one |
| 4 | **Decision / Validator** | **Validator** | RBAC + whitelist + safety bounds + `NEVER_SHED` → issues a signed token, or logs a rejection |
| 5 | **Action** | **Control** | Executes the tokened, whitelisted command against the actuators |
| — | *(cross-cutting)* | **Security** | Integrity, replay/tamper, sandbox, quarantine — can block any agent |
| — | *(cross-cutting)* | **XAI** | Reads the audit log, writes plain language for the operator — **read-only** |

Four agents form the command path. Two more wrap it: one defends it, one explains it.

---

## 2. Why there are six agents, not four

The brief asks for four stages. We add two because the two heaviest-weighted rubric criteria demand them:

- **Security (20%)** needs an agent whose *only* job is defending the control system — not the grid. It inspects raw ingress, detects replay/tamper, quarantines compromised nodes, and can block any other agent. No other agent has that authority.
- **Pitch & feasibility (20%)** needs the system to be trusted by the person who operates it. The operator is a **nurse or clinic technician, not a power engineer.** An automation that silently cuts power without saying why does not get adopted in the field. XAI is the adoption story.

Critically, **neither addition weakens the safety guarantee**, because of the boundary in §3.

---

## 3. The boundary — the single most important design decision

Our earlier drafts contradicted each other: one promised *"no generative model in the command path"*, the other introduced *LangChain/AutoGen + a local LLM + `gpt-4o-mini`*. A security-minded judge finds that in ten seconds. Resolved as follows:

**Inside the boundary — deterministic, no generative model:**
`Monitor → Planner → Validator → Control`. Every command traces to a threshold, the priority hierarchy, or a bounded score. The Planner **selects** from a finite enumerated whitelist; it never composes free text. Because **no free-text-to-command step exists anywhere**, a hallucinated or injected command has no path to an actuator. This is a *structural* guarantee, not a filter that might miss something.

**Outside the boundary — the one generative component:**
The **XAI Explainer** reads an already-made, already-logged decision from the audit log and writes a human sentence. It has no write access to plans, tokens, or actuators. It runs a **local** small model; if that model is unavailable (cold start, low power), it **falls back to deterministic templates**, so the critical loop never depends on it.

**Removed from the runtime:** any cloud LLM / OpenAI API dependency (`gpt-4o-mini`). A cloud call in the loop breaks connectivity-independence, which is the entire premise — during the Feb 2025 blackout the internet went with the power.

> **Answer to "so is there AI in this or not?"** — Yes: bounded anomaly detection, depletion forecasting, and multi-agent orchestration. What there isn't, deliberately, is a generative model with its hands on a switch.

---

## 4. Inter-agent message contracts

These are the actual objects passed between agents. Freeze them on Day 1 — everything downstream depends on them.

### 4.1 `StateVector` — Monitor → Planner
```json
{
  "ts": "2026-09-04T13:22:07Z",
  "grid":      { "status": "lost", "voltage_v": 0.0, "freq_hz": 0.0, "rocof_hz_s": -1.82 },
  "battery":   { "soc_pct": 54.2, "temp_c": 31.0, "discharge_w": 880 },
  "solar":     { "gen_w": 1420, "trend": "falling" },
  "loads":     { "CH1": { "priority": 1, "w": 180, "on": true },
                 "CH2": { "priority": 2, "w": 240, "on": true },
                 "CH3": { "priority": 3, "w": 420, "on": true },
                 "CH4": { "priority": 4, "w": 160, "on": true } },
  "coldchain": { "fridge_temp_c": 4.1, "status": "ok" },
  "anomaly_score": 0.12,
  "flags": ["GRID_LOST"]
}
```

### 4.2 `Plan` — Planner → Validator
The **candidate list is what makes this agentic.** A single hardcoded response is automation, not planning.
```json
{
  "plan_id": "pln-000412",
  "issued_by": "planner",
  "trigger": "GRID_LOST",
  "candidates": [
    { "rank": 1, "cmd": "ISLAND",     "args": {},                  "est_runtime_h": 9.4, "risk": "low" },
    { "rank": 2, "cmd": "SHED_LOAD",  "args": { "channel": "CH4" }, "est_runtime_h": 6.1, "risk": "low" },
    { "rank": 3, "cmd": "DELAY_LOAD", "args": { "channel": "CH3", "duration_min": 90 }, "est_runtime_h": 7.8, "risk": "med" },
    { "rank": 4, "cmd": "NO_ACTION",  "args": {},                  "est_runtime_h": 2.2, "risk": "high" }
  ],
  "selected": { "cmd": "ISLAND", "args": {} },
  "rationale_code": "GRID_UNSTABLE_ISLAND",
  "forecast": { "soc_now": 54.2, "soc_in_1h": 49.0, "depletion_eta_h": 9.4 }
}
```

### 4.3 `SignedToken` — Validator → Control
```json
{
  "token_id": "tok-8f2a91",
  "plan_id":  "pln-000412",
  "cmd": "ISLAND",
  "args": {},
  "checks": { "rbac": "pass", "whitelist": "pass", "bounds": "pass", "never_shed": "n/a" },
  "issued_at": "2026-09-04T13:22:07Z",
  "expires_at": "2026-09-04T13:22:12Z",
  "sig": "hmac-sha256:<truncated>"
}
```
Short expiry (5 s) means a captured token cannot be replayed later.

### 4.4 `Rejection` — the object to show on stage
```json
{
  "plan_id": "pln-000455",
  "decision": "REJECT",
  "reason": "NEVER_SHED_VIOLATION: CH1 is P1-critical (fridge-01)",
  "requested_by": "planner",
  "logged": true
}
```

---

## 5. Agent contracts (least privilege)

| Agent | Consumes | Produces | Write scope | Can actuate? | Failure mode |
|---|---|---|---|---|---|
| **Monitor** | Validated telemetry | `StateVector` + flags | `telemetry/monitor` | **No** | Holds last-known state; stale alarm at 30 s |
| **Planner** | State vector, forecast | `Plan` (3–4 candidates) | `plans/proposed` | **No** | Falls back to static threshold rules |
| **Validator** | Plan, RBAC table, bounds | `SignedToken` or `Rejection` | `plans/validated` | **No** | **Fails closed** — no token, no action |
| **Control** | Signed token **only** | Actuator writes + receipt | Transfer point, battery, channels | **Yes — tokened only** | Safe state: hold config, alarm |
| **Security** | Raw ingress, logs, agent traffic | Quarantine rules, flags | Sandbox rules, block list | **No — but can block** | Ingress defaults to strict-reject |
| **XAI** | Audit log (read-only) | Operator-facing text | Dashboard text only | **Never** | Deterministic templates take over |

**Invariant:** no agent both decides and acts. The Planner decides but cannot execute; the Control executes but cannot decide.

---

## 6. Decision logic

### 6.1 State machine
```
   ┌────────────────────┐   ISLAND()  — grid lost | |RoCoF|>1.0 | V<210    ┌──────────────┐
   │  GRID-CONNECTED    │ ─────────────────────────────────────────────►  │   ISLANDED   │
   │  all channels on   │                                                  │  PV+battery  │
   │  battery charging  │ ◄─────────────────────────────────────────────  │ finite energy│
   └────────────────────┘   RECONNECT_GRID() — stable N consecutive reads └──────────────┘
```

### 6.2 The shed ladder (while islanded)
```
   P1 · CRITICAL      Vaccine fridge · emergency lighting     ►  PROTECTED — ALWAYS ON
  ═══════════════ NEVER_SHED — Validator hard stop ═══════════════
   P2 · HIGH          Comms · essential medical devices       ►  held unless severe deficit
   P3 · FLEXIBLE      Water pump · general fans               ►  DELAY_LOAD — deferred     ▲
   P4 · NON-ESSENTIAL Admin computers · non-essential light   ►  SHED_LOAD — dropped first  │ shed order
```
Loads are restored in reverse order as solar recovers.

**Where the rule lives matters.** `NEVER_SHED` is enforced in the **Validator**, not the Planner. A compromised Planner can *request* shedding the fridge; it will be rejected and logged. Safety is enforced at the gate, never trusted at the source.

### 6.3 Command whitelist (immutable)
`ISLAND()` · `RECONNECT_GRID()` · `SHED_LOAD(channel)` · `RESTORE_LOAD(channel)` · `DELAY_LOAD(channel, min)` · `DISCHARGE_BATTERY(rate, min)` · `ISOLATE_NODE(device_id)`

Anything outside this set — modify audit log, drop tables, run shell — is **not expressible** through the agent interface.

---

## 7. Worked trace — the four demo scenarios

| Scenario | Monitor sees | Planner proposes | Validator decides | Control does | XAI tells the nurse |
|---|---|---|---|---|---|
| **1 · Grid fails** | `grid_status=lost`, RoCoF −1.8 Hz/s → `GRID_LOST` | 4 candidates; ranks `ISLAND()` #1 (9.4 h) | RBAC ✓ whitelist ✓ bounds ✓ → token | Opens grid transfer; runs on PV + battery | *"Grid became unstable — I disconnected and switched the clinic to solar and battery."* |
| **2 · Battery drains** | SoC 38% falling, solar weak → `DEFICIT_FORECAST` | `SHED_LOAD(CH4)` then `DELAY_LOAD(CH3)` | target ≠ P1 → **approve** | Drops admin power; defers water pump | *"I turned off admin power and paused the pump to keep the vaccine fridge running ~6 h longer."* |
| **3 · Attack** | Meter claims "healthy" but `seq` replays, `sig` fails | — (Security intercepts pre-planning) | Rejects any plan citing the tainted node | `ISOLATE_NODE()` — quarantined, stripped from estimation | *"One meter was sending false readings. I isolated it; the clinic is unaffected."* |
| **4 · Recovery** | Grid within bounds for N consecutive reads | `RECONNECT_GRID()`, restore P3 → P4 | stability window met → token | Closes transfer; restores in reverse order | *"Mains power is stable again — reconnected and everything is back on."* |

**Demo the rejection too.** Inject a plan targeting `CH1`; the audit log prints
`REJECT — NEVER_SHED_VIOLATION: CH1 is P1-critical (fridge-01)`.
That one line proves the security model is real code, not a slide.

---

## 8. The feedback loop — what "learn" means here

The brief asks for `monitor → reason → validate → act → learn`. Most teams skip the last word. Ours:

1. **Baselines recalibrate.** Rolling per-device statistics sharpen outlier detection; adjacent-node cross-checks catch slow sensor drift without an outage to reveal it.
2. **Forecast error corrects.** Predicted vs. actual SoC decay adjusts the discharge model coefficient, so depletion estimates improve after every outage.
3. **Threat memory persists.** Patterns from a quarantined node are retained by the Security agent and matched against future traffic.
4. **Operator overrides are studied.** Every manual override is logged with its reason and reviewed between runs — **human-approved, never auto-applied.**

### 8.1 The immutability guardrail (say this before a judge asks)
Learning updates **estimates only**. It can never modify:
- the **command whitelist**
- the **4-tier priority hierarchy**
- the **`NEVER_SHED` rule**
- the **RBAC permission matrix**

Those are immutable constants, changeable only by a signed configuration change under **admin** RBAC — never by the running system. *An adaptive controller that could re-learn its own safety limits is exactly the failure mode we designed out.*

---

## 9. Security guardrails at the agent layer (rubric: 20%)

| Control | Mechanism |
|---|---|
| **Identity** | Signed JWT handshake per device and per agent |
| **Least privilege** | RBAC roles: `device`, `monitor`, `planner`, `validator`, `control`, `operator`, `admin` |
| **Approved command set** | Immutable enumerated whitelist (§6.3); anything else inexpressible |
| **Anti-hallucination** | Deterministic command path — no free-text-to-command step exists |
| **Critical-asset lock** | `NEVER_SHED` on P1, Validator-enforced regardless of requesting agent |
| **Telemetry integrity** | `seq` monotonic (replay) + `sig` HMAC-SHA256 (tamper) |
| **Compromise containment** | Sandbox + quarantine; node stripped from state estimation, grid continues |
| **Human override** | 10-minute watchdog → alarm → dashboard confirmation; admin manual override always available |
| **Accountability** | Append-only audit log: every state, plan, check, token, action, rejection + XAI rationale |
| **Token replay defence** | 5-second token expiry |

---

## 10. How to present this (8-minute pitch)

1. **Show Section 1 of the HTML first** — the five-box row. It reads as literal compliance with the brief in three seconds.
2. **Then Section 2** — say *"same loop, engineering resolution"* and point at the cyan dashed boundary. That's your differentiator slide.
3. **Then run the demo** (`python simulation/inject.py full`) and narrate against Section 5's table.
4. **Close on the rejection line.** It's the most concrete security evidence you have.

**Anticipated Q&A**
- *"Where's the AI?"* → Bounded anomaly detection, depletion forecasting, multi-agent orchestration, and a local explainer. What's deliberately absent is a generative model touching a switch.
- *"Why not just use an LLM planner?"* → It can hallucinate a switch command, needs the cloud that a blackout takes down, and reports 6–17 s latency. Frequency events develop far faster.
- *"Isn't this just a transfer switch?"* → An ATS flips everything blindly and a UPS gives minutes. We forecast depletion, shed by priority for hours, explain, and defend ourselves.
- *"What if the LLM is down?"* → Templates take over. The command path never depended on it.
- *"Can the system learn its way into an unsafe state?"* → No — §8.1.

---

*Companion visual: `Step3_Agentic_AI_OS_Blueprint.html`. Repo deliverables: `CLAUDE.md`, `spec.md`, `README.md` in `Resilient_Smart_Microgrid`.*
