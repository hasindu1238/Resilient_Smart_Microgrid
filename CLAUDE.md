# CLAUDE.md — System Scope & Multi-Agent Orchestration

**Project:** Resilient Smart Microgrid — *The Village Micro-grid Brain*
**Subtitle:** A Secure, Self-Healing & Explainable AI Energy OS for Rural Health Centers
**Primary track:** **SDG 3 — Good Health & Well-being** (uninterrupted power for the medical cold chain)
**Secondary:** SDG 11 (resilient communities) · SDG 13 (climate action — maximised solar self-use, less diesel)
**Event:** IEEE CS R10 Summer School 2026 — Mini Ideathon (theme: AI · IoT · Cybersecurity)
**Status:** Concept blueprint + working simulated prototype for a 3-day ideathon. Not a deployment.
**Repo:** this is the *system scope + agent roles* deliverable. See also [`spec.md`](./spec.md) (inputs / outputs / constraints) and [`README.md`](./README.md) (how to run / understand).

---

## 1. System Scope & Positioning

An **edge-first, cyber-secure controller for a single rural health-center microgrid** that keeps the clinic's critical medical loads energised through grid failures and does so *autonomously, locally, and explainably*. The control loop **closes on an on-site gateway and keeps working when cloud connectivity is completely severed** — because in rural Sri Lanka a grid outage usually takes the internet with it.

The system does two things that are really **two phases of one event**:

1. **Self-healing (islanding).** When the grid destabilises or drops (the 9 Feb 2025 low-inertia scenario — see [`spec.md` §1](./spec.md)), the controller *automatically isolates the clinic from the unstable grid* and runs it on local solar + battery. When the grid returns and stays stable for a sustained window, it *automatically reconnects*. The clinic heals itself off the bad grid and back on when it is safe — smarter than a dumb automatic transfer switch.

2. **Energy prioritisation / intelligent load-sharing (once islanded).** Battery energy is now finite. The controller forecasts battery depletion against solar input and **shares scarce energy across loads by a strict 4-tier priority**, shedding or deferring non-essential loads *to guarantee the vaccine fridge and emergency lighting never go dark*.

> **One-sentence pitch:** *During the February 2025 island-wide blackout, a rural clinic running this system would not have gone dark for six hours — it would have islanded automatically and kept the vaccine fridge running the whole time.*

### 1.1 What we are **NOT** (say this before a judge does)

- We did **not** invent self-healing grids. FLISR (Schneider, S&C, Survalent, G&W, SEL) is a decade-old commercial category. Detect → isolate → restore is solved at utility scale.
- We did **not** invent agentic grid control. Grid-Agent (arXiv:2508.05702), X-GridAgent, GridMind and others established the Monitor → Planner → Validator pattern in 2025.
- We are **not** a generator, a solar installer, or a UPS. We do not make power; we protect continuity of *critical* power.

### 1.2 What we **ARE** (the five defensible claims)

1. **Edge-resident & connectivity-independent.** The loop closes locally; no cloud, no internet dependency. A recovery system that needs a cloud LLM is useless in the exact moment — the blackout — it is needed.
2. **Deterministic command path — no hallucinated switching.** No generative model actuates anything. Every command traces to a rule or a bounded, auditable score.
3. **Explainable to a non-expert operator (XAI).** A clinic is staffed by a nurse or technician, not a power engineer. Every automated action produces a plain-language reason — *and this never compromises claim 2* (see §2.1).
4. **Real-time.** Sub-second local response on the safety path, versus the 6–17 s reasoning loops reported for cloud-LLM approaches.
5. **Security *of* the control system, not just *by* it.** Least-privilege agents, RBAC, signed telemetry, a whitelisted command set, sandboxing, and a human-override timer. The research systems remediate attacks *on the grid*; we also defend the *controller*.

Anchor problem: the 9 Feb 2025 Sri Lanka island-wide blackout (low-inertia, high-solar cascading failure). See [`spec.md` §1](./spec.md).

---

## 2. Multi-Agent OS: Roles, Permissions & the One Decision That Wins the Room

Map to the ideathon flow: **Inputs → Monitor → Planner → Decision/Validator → Action**, with a continuous *monitor → reason → validate → act → learn* loop.

```
        ┌──────────────────── PHYSICAL ECOSYSTEM (simulated) ────────────────────┐
        │   Grid feeder · Solar PV · Battery bank · 4 prioritised load channels   │
        │            behind ONE smart controller / transfer point                 │
        └───────────────┬──────────────────────────────────────▲──────────────────┘
                        │ signed JSON telemetry (seq + HMAC)     │ tokened, whitelisted commands
                        ▼                                        │
                 ┌───────────┐    ┌───────────┐    ┌───────────┐ │
                 │  MONITOR  │──► │  PLANNER  │──► │ VALIDATOR │─┼─►  CONTROL
                 │  (sense)  │st  │ (forecast │pln │ (authorise│t│    (actuate)
                 └───────────┘    │  + select)│    │  + guard) │ │
                        │         └───────────┘    └─────▲─────┘ │
                        │                                │ RBAC  │
                        └────────────► SECURITY ◄────────┘       │
                        │       (integrity · sandbox · audit)    │
                        ▼                                        │
                 ┌───────────┐                                   │
                 │    XAI     │  reads the *logged decision*, writes a plain-language
                 │ (explain)  │  reason for the operator. CANNOT issue commands.
                 └───────────┘
```

| Agent | Read scope | Write scope | Notes |
| :-- | :-- | :-- | :-- |
| **Monitor** | live sensor metrics | `telemetry/monitor` queue | thresholds + bounded outlier score; detects grid instability/loss; **no** actuation |
| **Planner** | monitor queue, grid/battery state, solar forecast | `plans/proposed` | selects from the finite whitelist by deterministic logic; **not** free-text; cannot execute |
| **Validator** | proposed plans, safety bounds, RBAC/ACLs | `plans/validated` (signed token) | approves/rejects; enforces limits, `NEVER_SHED`, human-override criteria |
| **Control** | validated tokens only | grid transfer / battery / load-channel actuators | executes **only** tokened, whitelisted commands |
| **Security** | logs, telemetry integrity | node sandbox rules, security flags | replay/tamper detection, quarantine, immutable audit; can block |
| **XAI (Explainer)** | validated plans + audit log | operator-facing text only | local small-LLM **or** deterministic template; **read-only, out of the command path** |

### 2.1 The decision that resolves the biggest contradiction in our own drafts

Two of our earlier drafts pulled in opposite directions: one said *"no generative model in the command path"*; the other introduced *LangChain/AutoGen + a local LLM + gpt-4o-mini*. **A security judge will find that in ten seconds.** Here is the resolution we commit to, and it is a *strength*, not a compromise:

> **Generative AI explains. Deterministic logic decides.**

- The command path (**Monitor → Planner → Validator → Control**) is 100% deterministic: threshold rules + the 4-tier priority hierarchy select from a fixed whitelist ([`spec.md` §6](./spec.md)). No generative model output ever reaches an actuator, so a hallucinated or injected switch command is *structurally impossible to execute*.
- The **XAI/Explainer agent runs a LOCAL small language model** (e.g. a quantised model via a local runtime) **only to translate an already-made, already-logged decision into a human-readable reason** for the clinic operator. It reads decisions; it cannot make or issue them. If the local model is unavailable (low power, cold start), XAI **falls back to deterministic natural-language templates**, so the critical loop never depends on it.
- **No cloud LLM. Remove `gpt-4o-mini` / any OpenAI-API dependency** from the runtime — a cloud call in the loop breaks the connectivity-independence claim, which is the whole point. LangChain/AutoGen, if used at all, orchestrates the *local, deterministic* agents — not a remote model in the command path.

This gives us the XAI transparency judges love **and** the anti-hallucination guarantee, with no conflict.

Deterministic guarantee: Planner → Validator → Control exchange only enumerated whitelist commands ([`spec.md` §6.1](./spec.md)). No generative-model output reaches an actuator.

---

## 3. Development Environment & Commands

```bash
git clone https://github.com/hasindu1238/Resilient_Smart_Microgrid.git
cd Resilient_Smart_Microgrid
python3 -m venv venv && source venv/bin/activate
pip install fastapi uvicorn pydantic paho-mqtt pytest pandas numpy scikit-learn streamlit
```

```bash
# Gateway + agent host (local edge, no cloud)
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000

# Virtual telemetry stream (emulates the clinic's sensors; signed payloads)
python simulation/iot_stream.py --interval 1.0 --delta 0.02 --sign

# Full narrated demo — runs all four scenarios in order
python simulation/inject.py full

# Or run one scenario at a time:
python simulation/inject.py grid_fail     # Scenario 1 — self-heal: grid destabilises → island
python simulation/inject.py low_battery    # Scenario 2 — prioritise: shed P3/P4, protect the fridge
python simulation/inject.py attack         # Scenario 3 — spoofed telemetry → quarantine, keep running
python simulation/inject.py recover        # Scenario 4 — grid stable → auto-reconnect
```

```bash
pytest tests/                                  # all
pytest tests/test_core.py -v                   # end-to-end path
pytest tests/ -k "never_shed" -v               # the fridge can NEVER be shed (our headline security test)
```

> **Firmware / integrity note:** any committed device reference code must pin the gateway certificate (`setCACert`), **never** `client.setInsecure()`, and must carry **no hardcoded credentials**. If cert-pinning is not finished in time, document TLS-pinning as the known production step in [`docs/RAID.md`](./docs/RAID.md) and keep the insecure call out of committed code. Do not leave `WiFiClientSecure` set to accept any certificate.

---

## 4. Team Division of Labour (5–6 members)

Six roles; if the team is five, **M3 absorbs M6** (the anomaly model and the XAI/agent path are one code area).

- **M1 — Architect & Backend.** Repo, FastAPI gateway, telemetry endpoint, SQLite/Postgres immutable audit DB, end-to-end integration.
- **M2 — IoT Simulation.** Canonical-schema telemetry generator, send-on-delta, `seq`/`sig` signing, the fault- and attack-injection scripts (`inject.py`).
- **M3 — Data Science & Agents.** Threshold + bounded outlier detection, Monitor + Planner logic, battery-depletion forecasting. **Thresholds first as an always-works fallback; model on top only if the threshold path runs.**
- **M4 — Cybersecurity.** Validator (RBAC + signed token), command whitelist, `NEVER_SHED` rule, replay/tamper checks, sandboxing/quarantine, 10-minute human rule, **RAID register (a Day-1 artefact)**.
- **M5 — Dashboard & Pitch.** Streamlit/React dashboard (topology + live agent activity + audit log + XAI reasons), the 8-minute deck, demo script, Q&A drill, **backup video**.
- **M6 — XAI & Explainability** *(or folded into M3)*. Local-LLM/template explainer; wording of operator-facing rationale; the "why did it shed the pump?" panel.

### 4.1 What each member must **NOT** do
- **M2:** no 10 kHz sampling — frequency/RoCoF at 10–50 Hz, SoC/thermal at 1 Hz ([`spec.md` §4.3](./spec.md)).
- **M3 / M6:** no generative/LLM model **in the command path** — that forfeits our headline differentiator. LLM lives only in read-only XAI.
- **M4:** no `setInsecure()` in committed code; no hardcoded secrets; RAID is Day-1, not Day-3.
- **M5:** every dashboard element must appear in the 90-second demo; the topology view must show the grid/island transfer point and the protected P1 loads, or the story is invisible.

---

## 5. Two-Day Runway (of a 3-day build)

**Day 1 — decide & stub (no production code before lunch).**
Morning, all together: freeze the topology on a whiteboard (grid feeder + PV + battery + 4 load channels behind one transfer point); freeze the telemetry schema; freeze the **one** self-heal scenario (Panadura-style grid loss) and the **one** attack scenario (spoofed "healthy" meter); lock the deterministic-command / XAI-explains decision (§2.1); resolve 10 kHz → 10–50 Hz; strip `setInsecure`/OpenAI from the runtime.
Afternoon, split: M1 gateway skeleton + audit table; M2 telemetry generator (normal only); M3 state interface + threshold rules + priority hierarchy; M4 whitelist + RBAC matrix as code + `NEVER_SHED` + RAID; M5 dashboard shell with topology + deck outline; M6 XAI template stubs.
*Checkpoint:* telemetry flows generator → gateway → DB → dashboard with fake numbers; nothing intelligent yet — **correct.**

**Day 2 — make it think, make it safe.**
Morning: M2 fault + attack injection; M3 Monitor detects instability, Planner generates 3–4 candidate responses with estimated consequences; M4 Validator (identity + whitelist + bounds + `NEVER_SHED` → approve/reject, all logged); M1 wires agents into the path; M5 shows live agent activity; M6 turns each logged decision into a plain-language reason.
Midday sync (15 min): one grid-loss travels the full path and produces an `ISLAND` + prioritised shed. Expect breakage — better today than tomorrow.
Afternoon: M3 adds the outlier model over thresholds (only if thresholds work); M4 builds the sandbox/quarantine attack demo (**our best 30 seconds on stage**); M5 records a backup video of whatever works.
*Checkpoint:* full four-scenario demo runs end-to-end, ugly but complete. **Day 3 is tuning, security polish and rehearsal — no new features.**

> One thing to settle tonight, not tomorrow: who is M3/M6's backup if the optional solar-forecast hook or the local LLM isn't done. Both are marked nice-to-have for a reason — the four-scenario core does **not** need them (templates cover XAI; thresholds cover forecasting). Walk in with the core rock-solid rather than risk it for a bonus feature nobody will ask about.
