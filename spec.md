# spec.md — Inputs, Outputs & Constraints

**Project:** Resilient Smart Microgrid — *The Village Micro-grid Brain*
**Primary track:** SDG 3 — Good Health & Well-being · **Secondary:** SDG 11, SDG 13
**Status:** Concept blueprint + simulated prototype for a 3-day ideathon. Not a deployment.
**Repo:** this is the *inputs / outputs / constraints* deliverable. See also [`CLAUDE.md`](./CLAUDE.md) (scope + agent roles) and [`README.md`](./README.md) (how to run).

---

## 1. Problem → Case Study (grounded, not invented)

### 1.1 The problem
A rural health center runs on an unreliable grid, backed by a modest solar array and a battery bank. When the grid fails — which is often — two things go wrong at once:

1. **Dumb backup drains fast.** Without intelligent control, the battery powers *everything equally* — the vaccine fridge, the emergency light, the water pump, the admin PC and the fans — until it is flat. Nothing is prioritised.
2. **Dumb load-shedding kills the wrong things.** Indiscriminate shedding can cut the medicine refrigerator alongside the non-essentials.

The result is a **silent failure of the cold chain**: temperature-sensitive vaccines and medicines spoil during a multi-hour outage, and nobody notices until it is too late. **0% power means 0% cold-chain integrity.** In rural healthcare, that is not an inconvenience — it is a threat to life, and it directly undermines **SDG 3 (Good Health & Well-being).**

### 1.2 The case study — national, recent, and almost too good
On **9 February 2025, Sri Lanka suffered an island-wide blackout** lasting roughly six hours and affecting the country's ~22 million people. The reported physical trigger was a monkey contacting a transformer at the **33 kV Panadura grid substation**.

**The trigger is not the story.** Per the Ceylon Electricity Board's own statement, the disturbance at Panadura caused *"a sudden voltage drop"* and cascading disconnections, and the underlying vulnerability was **low grid inertia from high non-synchronous solar**:

> *"Over 50% of national electricity demand was met by 800 MW of solar photovoltaic (PV) generation … Due to the high penetration of non-synchronous solar PV generation, the grid had a low system inertia, making it vulnerable to faults."* — CEB, on the "Sunny Sunday" cascading failure.

**the CEB's own remediation list**:

> *"Deploying grid-forming inverters with BESS to provide synthetic inertia and frequency stabilization"* and *"Advancing Smart Grid investments to improve real-time monitoring and control of renewable energy integration."*

**Our project builds, at the scale of a single clinic, exactly what the national utility publicly says it needs:** local battery-backed resilience, real-time monitoring, and intelligent control of a solar-heavy supply. A monkey should not be able to black out a hospital's vaccine fridge — the fragility is the point, and one animal made it visible to 22 million people.

> ⚠️ **Verify before pitch day.** The >50% / 800 MW / low-inertia / Panadura figures above are quoted from CEB via EconomyNext and Ada Derana (see [`README.md`](./README.md) sources). Confirm against the primary CEB media release and cite the utility, not an aggregator. Present duration (~6 h) and population (~22 M) as *widely reported*.

---

## 2. Gap Analysis — Where This Sits Relative to Existing Work

Three mature bodies of work already exist. We do **not** claim to have invented any of them. Claiming so is the fastest way to lose the room. Our contribution is the *combination* none of them offers at clinic scale.

| Dimension | FLISR (commercial) | Grid-Agent / cloud-LLM research | Dumb ATS / UPS | **This project** |
|---|---|---|---|---|
| Isolate + restore supply | Yes (reroute) | Partial | All-or-nothing switch | **Yes (island + reconnect)** |
| Intelligent load priority | No | No | No | **Yes (4-tier, protects P1)** |
| Decision method | Fixed rules | Cloud LLM reasoning | Fixed relay | **Deterministic rules + bounded score** |
| Works during a blackout (no cloud) | Yes (but proprietary) | **No — needs cloud** | Yes | **Yes — closes loop locally** |
| Decision latency | cycles–seconds | **6–17 s (reported)** | instant | **Sub-second (target)** |
| Hallucinated-command risk | None | **Present (generative LLM)** | None | **None (deterministic path)** |
| Explains itself to a non-expert | No | Partly | No | **Yes (XAI, read-only)** |
| Security *of* the controller | Minimal | Not addressed | None | **RBAC + whitelist + sandbox + override** |
| Backup duration | n/a | n/a | **Minutes** | **Hours–days (intelligent shedding)** |
| Target scale / cost | Utility / high | Utility / high (cloud) | Building / low | **Clinic / low (commodity edge)** |

**The gap we occupy (the four claims we defend in Q&A):**
1. **Edge-resident & connectivity-independent** — the loop closes locally; the cloud-LLM systems cannot run during the blackout they aim to fix.
2. **Deterministic command path** — no generative model actuates anything; an LLM planner *can* hallucinate a switch command, ours structurally cannot.
3. **Explainable + intelligent, not dumb** — unlike an ATS/UPS, we forecast depletion and shed by priority to stretch hours of runtime for the loads that matter, and we tell the operator *why* in plain language.
4. **Secure by and of the system** — we defend the controller itself against spoofed telemetry, command injection and compromised nodes.

> *"Isn't this just an automatic transfer switch or a UPS?"* — A UPS gives you minutes and switches everything; an ATS flips the whole building to backup blindly. Ours **forecasts** when the battery will die, **sheds intelligently** to keep the fridge alive for *hours to days*, **explains** each action, and **defends** itself from cyber-tampering. None of those exist in an ATS or UPS.

---

## 3. Users & Target Audience

**Primary users — rural & regional health facilities in Sri Lanka (and comparable LMIC settings):**
- **The facility:** a rural/divisional health center, MOH clinic, or small regional hospital running on **grid + solar PV + battery**, holding a **vaccine/medicine cold chain** (EPI vaccines, insulin, some antibiotics) plus emergency lighting, essential medical devices, communication equipment, a water pump and administrative loads.
- **The operator (the human in the loop):** a **nurse, midwife (PHM), or clinic technician — not a power engineer.** This is *why* explainability is a hard requirement: the dashboard must say *"I turned off the water pump to keep the vaccine fridge running for 6 more hours"*, not print a relay code.
- **The overseer:** the **regional/district health authority** and the facility administrator, who need an immutable audit trail and remote-free assurance that P1 loads held.

**Secondary / scale-out audience (state as spillover, one sentence, then move on):** any community-scale critical node with a mix of critical and flexible loads — a **rural school server room, a telecom tower, a community water-pumping station, an off-grid research post.** We *demo* the clinic because the stakes (a spoiled vaccine) are immediate and legible to any judge in 90 seconds; the same controller generalises to these nodes.

**Why the clinic is the right beachhead:** the value is *continuity of a specific critical asset*, and a clinic has the clearest, most defensible critical asset there is — a fridge full of vaccines whose loss is measured in lives, not inconvenience.

---

## 4. Inputs — IoT Telemetry Pipeline & JSON Schema

Edge devices are modelled as solar-PV inverters, a battery-management system (BMS), per-channel smart load switches, a grid-status sensor and a cold-chain temperature probe. All simulated nodes stream one canonical schema.

### 4.1 Canonical telemetry JSON schema
```json
{
  "device_id": "battery-01",
  "ts": "2026-09-03T09:15:22Z",
  "metric": "battery_soc",
  "value": 62.4,
  "unit": "%",
  "seq": 10432,
  "sig": "hmac-sha256:<truncated>"
}
```
`device_id`, `ts`, `metric`, `value` satisfy the ideathon's required schema; `unit` is added for safety; **`seq`** (monotonic counter, replay detection) and **`sig`** (HMAC-SHA256 over the payload with a per-device key, tamper detection) are the integrity fields — see §7. *(This replaces any earlier `metadata.secure_client: "WiFiClientSecure"` field, which advertised an insecure TLS mode and is removed.)*

### 4.2 Supported metrics & operational bounds
| Metric | Unit | Nominal | Lower | Upper | Action threshold |
| :-- | :-- | :-- | :-- | :-- | :-- |
| `grid_status` | enum | `connected` | — | — | `lost`/`unstable` → evaluate **ISLAND** |
| `grid_voltage_v` | V | 230.0 | 216.2 (−6%) | 243.8 (+6%) | < 210 (under-V) / > 248 (over-V) |
| `grid_frequency_hz` | Hz | 50.00 | 49.50 | 50.50 | < 49.30 / > 50.70 |
| `rocof_hz_s` | Hz/s | 0.00 | −0.5 | 0.50 | \|value\| > 1.0 (low-inertia signature) |
| `solar_generation_w` | W | — | 0 | 5000 | N/A (climatic variation; feeds forecast) |
| `battery_soc` | % | 100.0 | 20.0 | 100.0 | < 40 (forecast shed) / < 20 (hard shed) |
| `battery_temp_c` | °C | 25.0 | 10.0 | 45.0 | > 55 (thermal-runaway hazard) |
| `fridge_temp_c` | °C | 4.0 | 2.0 | 8.0 | > 8 (cold-chain breach — P1 alarm) |
| `load_channel_state` | enum | `on` | — | — | reported per channel (on/off + W) |
| `tamper_status` | binary | 0 | 0 | 0 | 1 → immediate isolation |

> `rocof_hz_s` (rate-of-change-of-frequency) is deliberate: it is the direct measurable signature of the **low-inertia failure mode that caused the Sri Lanka blackout**, and it is how we tie the live demo to the real case study. `fridge_temp_c` is the health of our P1 critical asset — the number the whole system exists to protect.

### 4.3 Sampling rates — resolved (do **not** reintroduce 10 kHz)
Grid **frequency / RoCoF and RMS voltage at 10–50 Hz**; **battery SoC, temperatures and load state at 1 Hz.** We do **not** sample at 10 kHz — that is only needed for harmonic analysis, which is out of scope. Removing it eliminates a bandwidth problem we would otherwise spend a slide solving, and keeps the edge-device energy story defensible.

### 4.4 Edge send-on-delta (bandwidth, not battery-decade claims)
Each node transmits only when `|X_current − X_last_sent| > Δ` (Δ = 2% of nominal range) or every 6 h as a heartbeat. On steady-state data this cuts transmissions from ~1440/day to ~12/day — **a ~99% reduction in radio transmissions**, the largest discretionary energy cost on a wireless node. *"The cheapest packet is the one you never send."*

> We claim the transmission reduction. We do **not** claim a specific multi-year battery life, because sleep/sampling current dominates node energy. The **battery-longevity story lives elsewhere and is stronger:** by *forecasting depletion and shedding intelligently, the system prevents deep discharges*, and preventing deep discharge is a well-established way to extend battery lifespan and defer replacement cost — a real, defensible sustainability benefit.

---

## 5. Topology Assumption (self-healing needs somewhere to heal to)

We model a **single-facility microgrid**: the clinic sits behind **one smart controller / transfer point** with three sources/paths — the **grid feeder**, the **solar PV array**, and the **battery bank** — feeding **four prioritised load channels**. Self-healing here means **islanding** (open the grid transfer, run on PV + battery) and **reconnection** (close it again once the grid is stable), *not* utility-style feeder rerouting. Stating this up front pre-empts the first question an electrical-engineering judge will ask ("heal to *what*?"): on a single radial feed with no local generation, "reconfigure" is impossible — our alternative path is the local PV + battery island.

```
            GRID FEEDER ──┐
                          │        ┌───────────── P1: Vaccine fridge + emergency light  (NEVER shed)
   SOLAR PV ──► [ BATTERY ]──►[ SMART ]──► CH1 ──┤
                 [  BANK   ]   [ CTRL /  ]──► CH2 ── P2: Comms + essential medical devices
                          │    [ TRANSFER]──► CH3 ── P3: Water pump + fans        (deferrable)
                          │    [  POINT  ]──► CH4 ── P4: Admin PC + non-essential  (shed first)
                (island ⇄ reconnect here)
```

---

## 6. Outputs — Command Whitelist & Priority Hierarchy

### 6.1 Approved command whitelist (immutable)
The Control agent ignores any command not in this finite set. Commands to modify audit logs, drop tables, or run a shell are **not expressible** through the agent interface.
1. `ISLAND()` — self-heal: open the grid transfer, run on PV + battery
2. `RECONNECT_GRID()` — self-heal: close the transfer after the grid is stable for *N* consecutive readings
3. `SHED_LOAD(channel_id)` — energy prioritisation: drop a channel (P3/P4 first)
4. `RESTORE_LOAD(channel_id)` — return a channel when energy allows
5. `DELAY_LOAD(channel_id, duration_min)` — defer a flexible load (e.g. the water pump)
6. `DISCHARGE_BATTERY(rate_kw, duration_min)` — bounded battery support
7. `ISOLATE_NODE(device_id)` — security: quarantine a compromised node

### 6.2 The 4-tier priority hierarchy (what to sacrifice to save what matters)
| Priority | Load type | Action during outage / low battery |
| :-- | :-- | :-- |
| **P1 — Critical** | Medicine/vaccine refrigerator, emergency lighting | **Always maintained. Non-negotiable. `NEVER_SHED`.** |
| **P2 — High** | Communication equipment, essential medical devices | Maintained unless a *severe* deficit is forecast |
| **P3 — Flexible** | Water pump, general fans | Temporarily delayed / shifted during peak shortage |
| **P4 — Non-essential** | Administrative computers, non-essential lighting | Shed immediately on grid failure or low storage |

### 6.3 Deterministic command path (our anti-hallucination guarantee)
No generative model sits between detection and actuation. The **Planner** selects from the whitelist using bounded logic (thresholds + priority + battery-vs-solar forecast); the **Validator** authorises with a signed token after RBAC + safety-bound + `NEVER_SHED` checks; the **Control** agent executes only tokened, whitelisted commands. There is no free-text-to-command step, so a hallucinated or injected command **cannot execute**. The **XAI agent** explains the result afterwards and is *outside* this path ([`CLAUDE.md` §2.1](./CLAUDE.md)).

### 6.4 End-to-End Operational Workflow
The runtime is a local-first, closed-loop sequence connecting the requirements above:

1. **Telemetry generation and JSON packaging.** Simulated edge nodes sample grid, solar, battery, load, and cold-chain signals at the rates in §4.3, apply the send-on-delta rule in §4.4, and package each reading using the canonical schema in §4.1, including monotonic `seq` and HMAC `sig`.
2. **Authenticated telemetry ingestion.** The gateway accepts device telemetry over the local API only after the signed JWT handshake in §7.1. It verifies schema, bounds, `seq`, and `sig`; replayed or tampered payloads follow the integrity and quarantine controls in §§7.4–7.5.
3. **Monitor Agent.** The Monitor consumes accepted readings, applies the action thresholds in §4.2, and detects grid loss or instability, battery stress, cold-chain alarms, and other bounded anomalies. It emits state for planning but cannot actuate.
4. **Planner Agent and energy triage.** The Planner combines the Monitor state with battery-versus-solar forecasting and the 4-tier hierarchy in §6.2 to select a bounded response from §6.1. It proposes islanding, reconnection, shedding, restoration, delay, or bounded battery support as appropriate; it does not execute commands.
5. **Security Validator Agent.** The Validator checks identity, RBAC, safety bounds, the immutable whitelist, watchdog criteria, and `NEVER_SHED`. It rejects unsafe plans and issues a signed token only for an approved plan.
6. **Closed-loop local actuation.** Control executes only the tokened whitelist command. The gateway and local actuators then feed their resulting state back into telemetry, keeping the loop operational without cloud or backhaul connectivity.
7. **XAI explanation generation.** After the decision is logged and acted on, the read-only XAI agent converts the finalized state and action into an operator-facing explanation using a local model or deterministic template; it never generates commands.
8. **Human-in-the-loop monitoring and audit logging.** The local dashboard shows telemetry, agent state, load priorities, actions, and the XAI rationale. Watchdog, alarm, confirmation, manual override, and safe-state behavior follow §7.6, while every telemetry value, proposal, validation result, token, action, and rationale is recorded in the immutable audit log.

---

## 7. Constraints — Cybersecurity Guardrails & Zero-Trust

Security-by-design across two axes: (a) prevent unauthorised actuation, (b) survive compromised / spoofed nodes. This is the 20%-weighted axis the LLM research does not address. Think of it as **concentric defence-in-depth around the physical medical assets.**

### 7.1 Identity & least-privilege (RBAC)
Every device and agent authenticates with a **signed JWT handshake**. **Role-Based Access Control** gives each principal the minimum it needs: `device` (telemetry write only), `monitor`, `planner`, `validator`, `control` (actuate only with a token), `operator` (dashboard + override), `admin`. No agent both decides and acts.

### 7.2 Deterministic command path + immutable whitelist
See §6.1 / §6.3. The finite whitelist is the *only* vocabulary; anything else is inexpressible.

### 7.3 The `NEVER_SHED` hard rule (our single strongest security detail)
A hard-coded rule blocks **any** command that would de-energise a **P1** channel (the vaccine fridge / emergency light), **enforced by the Validator independently of which agent issued it** — even a fully compromised Planner cannot switch off the fridge. This is the detail most likely to impress a security-minded judge: safety is enforced at the gate, not trusted at the source.

### 7.4 Telemetry integrity: replay & tamper
Each payload carries `seq` (monotonic) and `sig` (HMAC-SHA256). The gateway rejects out-of-order `seq` (replay) and bad `sig` (tamper). A node emitting physically impossible values, frozen repeats, or injection strings is flagged by the Security agent.

### 7.5 Device sandboxing & quarantine (our best 30 seconds on stage)
On detecting a compromised node, the gateway (a) blocks the node's route, (b) diverts its queue to a sterile quarantine sandbox, (c) strips it from the state estimator. The microgrid continues on adjacent-node readings and time-series estimation. **"Attack the control system live, and it degrades safely — the fridge never notices."**

### 7.6 Human-in-the-loop & emergency failsafe
Any autonomous load-shed or disconnection starts a **10-minute watchdog**; at 10 minutes the system raises an alarm and requires dashboard confirmation. If no override token arrives within 2 minutes, it restores power to low-priority sectors and falls back to a safe state. Administrators always have **full manual override**. The **audit log is immutable** — every decision, token, and action is recorded with its XAI rationale.

---

## 8. RAID Risk Model

- **R — Cyber-actuation hijacking.** Attacker compromises the Planner and issues malicious commands.
  *Mitigation:* Control accepts commands **only** with a valid Validator-signed token; whitelist + deterministic path make arbitrary commands inexpressible; `NEVER_SHED` protects P1 regardless of source.
- **R — Latency-induced instability.** Reasoning too slow to catch fast frequency events.
  *Mitigation:* deterministic threshold/RoCoF path runs locally, sub-second, no cloud round-trip — precisely where the 6–17 s LLM approaches fail.
- **R — Sensor drift / inaccurate SoC.** Meters/probes drift without an outage.
  *Mitigation:* a dedicated outlier-detection check validates telemetry against historical baselines and adjacent nodes; flag slow divergence.
- **R — Backhaul connectivity cuts (frequent in rural areas).**
  *Mitigation:* system engineered for 100% local operation; **no cloud-API dependency in the critical loop** (XAI falls back to templates offline).
- **A — Alternative path exists.** Self-healing assumes local PV + battery behind the transfer point (§5).
  *Mitigation:* stated explicitly; with no local generation we degrade to *isolate-and-alert*, not island.
- **A — Local compute is sufficient.** Edge gateway has enough CPU/RAM for lightweight detection.
  *Mitigation:* quantised scikit-learn / thresholding under ~15 MB RAM; thresholds always available as fallback.
- **I — LLM unavailable at the edge.** The local explainer model may be too heavy or cold-start.
  *Mitigation:* XAI is non-critical and **falls back to deterministic templates**; the command path never depends on it.
- **D — Variable solar generation.** Cloud cover changes available energy hour to hour.
  *Mitigation:* the Planner runs continuous forward-looking energy-deficit forecasting (battery SoC + solar trend vs. load).
- **D — Gateway power.** The controller needs power to run.
  *Mitigation:* gateway on a small dedicated LiFePO4 backup — autonomous through the blackout, which is exactly when it is needed.

---

## 9. Prototype Scope (3 days) & Community Value

### 9.1 Build vs simulate
**Build for real:** telemetry generator (canonical schema), FastAPI gateway, deterministic Monitor + Planner + Validator + Control + Security agents, the XAI explainer (template + optional local model), whitelist + signed-token path, RBAC, immutable audit log, sandboxing/quarantine, the dashboard, and the four scripted scenarios.
**Simulate on purpose:** the physical grid, PV, battery, sensors and relays (Python models). *The guideline asks for a **simulated** IoT pipeline — simulation is the assignment, not a shortcut.*

### 9.2 How this helps the community (why it matters)
- **Protects health outcomes (SDG 3):** guarantees the vaccine cold chain and emergency lighting through outages — the difference between usable and spoiled medicine.
- **Keeps the clinic operational at night and in emergencies:** comms and essential devices stay up for deliveries and referrals.
- **Extends battery life & cuts cost (SDG 12/13):** preventing deep discharge defers costly battery replacement — decisive for a cash-strapped rural facility.
- **Maximises solar self-consumption (SDG 13):** runs the clinic on its own clean generation first, reducing diesel reliance and emissions.
- **Low-cost & scalable (SDG 11):** commodity edge hardware means one design replicates across a national network of rural facilities.
- **Trusted because it explains itself:** XAI turns an opaque automation into a tool a nurse can trust and a health authority can audit — the real barrier to adoption in the field.
- **Secure by design:** as clinics get connected, this defends the controller against the new cyber-attack surface that connectivity introduces.

### 9.3 SDG mapping (and how to answer the "isn't this SDG 7/13?" question)
**Primary — SDG 3 (Good Health & Well-being):** the value is *continuity of critical medical services*; the protected asset is a vaccine fridge. **Secondary — SDG 11** (resilient community infrastructure) and **SDG 13** (climate — clean-energy self-use, less diesel). If a judge says "this looks like SDG 7 (affordable & clean energy)": we don't *generate* energy — we keep the clinic's critical care alive and make best use of the clean energy it already has. Lead with health; mention 11 and 13 as spillover and move on.
