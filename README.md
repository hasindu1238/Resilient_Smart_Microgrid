# Resilient Smart Microgrid — *The Village Micro-grid Brain*

### A Secure, Self-Healing & Explainable AI Energy OS for Rural Health Center

> A local controller that keeps a rural clinic's **vaccine fridge alive** through a blackout — islanding onto solar + battery, intelligently shedding non-essential loads, explaining every decision to the nurse in plain language, and defending itself from cyber-tampering. **On-site, in real time, with no cloud and no internet** — because a rural blackout usually takes the network down too.

---

## What this is

When the grid fails at a rural health center, two things go wrong at once: a dumb battery drains powering *everything equally*, and dumb load-shedding can cut the **medicine refrigerator** alongside the fans and the admin PC. Either way the cold chain dies silently and vaccines spoil.

This is a **simulated** controller — the "Brain" for the IoT "Nervous System" — that runs a closed loop:

**Sense → Forecast → Decide (safely) → Act → Explain → Recover**

It watches solar, battery and grid; **self-heals** by islanding the clinic onto local solar + battery the moment the grid destabilises; **prioritises** the finite battery energy across a strict 4-tier hierarchy so the fridge *never* goes dark; *authorises* every action through a separate security component; **explains** each action to the operator; and **reconnects** automatically once the grid is stable again. The loop closes on a local edge gateway even with the internet completely severed.

---

## Why it matters — the real case

On **9 February 2025, Sri Lanka suffered an island-wide blackout** (~6 hours, ~22 million people). The reported trigger was a monkey at the **33 kV Panadura substation** — but per the Ceylon Electricity Board, the real root cause was **low grid inertia from high non-synchronous solar**: *"over 50% of national electricity demand was met by 800 MW of solar PV … the grid had a low system inertia, making it vulnerable to faults."* The CEB's own fix list: **grid-forming inverters with battery storage** and **smart-grid real-time monitoring and control**.

**We build, at the scale of one clinic, what the national utility says it needs.** A rural health center running this system during that blackout would not have gone dark for six hours — it would have **islanded automatically and kept the vaccine fridge running the whole time.**

---

## Who it's for

- **Primary:** rural / divisional health centers and small regional hospitals (Sri Lanka and comparable settings) running on **grid + solar + battery** with a **vaccine/medicine cold chain.**
- **The operator is a nurse or technician, not a power engineer** — which is why every decision is explained in plain language ("*shed the water pump to keep the vaccine fridge alive 6 more hours*"), not a relay code.
- **Scale-out (spillover):** any community-scale critical node — a school server room, a telecom tower, a water-pumping station. We demo the clinic because a spoiled vaccine is the most legible stake there is.

See [`spec.md` §3](./spec.md) for the full user analysis and [`spec.md` §9.2](./spec.md) for community impact.

---

## What's genuinely ours (and what isn't)

We do **not** claim to have invented self-healing grids (**FLISR** — Schneider, S&C, Survalent, G&W, SEL — is a decade-old commercial category) or agentic grid control (**Grid-Agent**, arXiv:2508.05702, and the 2025 LLM research wave established the Monitor→Planner→Validator pattern). Our contribution is the *combination* none of them offers at clinic scale:

| | FLISR | Cloud-LLM research | Dumb ATS / UPS | **This project** |
|---|:--:|:--:|:--:|:--:|
| Works during a blackout (no cloud) | ✅ (proprietary) | ❌ needs cloud | ✅ | ✅ **edge-local** |
| Intelligent load priority (protect the fridge) | ❌ | ❌ | ❌ | ✅ **4-tier, `NEVER_SHED`** |
| Decision latency | cycles–sec | 6–17 s | instant | **sub-second (target)** |
| Hallucinated-command risk | none | present | none | **none (deterministic path)** |
| Explains itself to a non-expert | ❌ | partly | ❌ | ✅ **XAI (read-only)** |
| Security *of* the controller | minimal | not addressed | none | ✅ **RBAC + whitelist + sandbox** |
| Backup duration | n/a | n/a | minutes | **hours–days** |
| Target scale / cost | utility / high | utility / high | building / low | **clinic / low** |

The key distinction is that this system works locally during a blackout while protecting a specific critical asset. Unlike a conventional UPS or ATS, it **forecasts** battery depletion, **sheds loads by priority**, **explains** its decisions, and **defends itself** against cyber-tampering. See [`spec.md` §2](./spec.md) for the full gap analysis.

---

## Architecture at a glance

```
   SIMULATED PHYSICAL GRID  (grid feeder + solar PV + battery + 4 prioritised load channels
                             behind ONE smart controller / transfer point)
                    │  signed JSON telemetry (seq + HMAC)
                    ▼
   ┌──────────┐   ┌──────────┐   ┌───────────┐   ┌──────────┐
   │ MONITOR  │──►│ PLANNER  │──►│ VALIDATOR │──►│ CONTROL  │──► actuators
   │ (sense)  │   │(forecast │   │(authorise)│   │  (act)   │
   └──────────┘   │ +select) │   └─────▲─────┘   └──────────┘
        │         └──────────┘         │ RBAC + signed token
        ├────────────► SECURITY ◄──────┘   (integrity · sandbox · quarantine · audit)
        ▼
   ┌──────────┐   reads the logged decision, writes a plain-language reason.
   │   XAI    │   Local model or template. CANNOT issue commands.
   │(explain) │
   └──────────┘
```

**No agent both decides and acts. No generative model touches the command path — it only *explains* the result.** This is the deliberate design choice that gives us XAI transparency *and* a structural anti-hallucination guarantee. Full detail in [`CLAUDE.md` §2](./CLAUDE.md).

---

## Quick start

**Requirements:** Python 3.10+

```bash
git clone https://github.com/hasindu1238/Resilient_Smart_Microgrid.git
cd Resilient_Smart_Microgrid
python3 -m venv venv && source venv/bin/activate
pip install fastapi uvicorn pydantic paho-mqtt pytest pandas numpy scikit-learn streamlit
```

**Run the system (three terminals):**
```bash
# 1 — local edge gateway + agent host (no cloud)
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000

# 2 — virtual telemetry stream (emulated sensors, signed payloads)
python simulation/iot_stream.py --interval 1.0 --delta 0.02 --sign
```

**Run the demo (terminal 3) — one command runs all four scenarios in order:**
```bash
python simulation/inject.py full
```
Or one at a time: `grid_fail` · `low_battery` · `attack` · `recover`.

**Dashboard:** open `http://127.0.0.1:8000` — shows the topology, the grid⇄island state, live agent activity, the XAI reasons, and the audit log.

**Tests:**
```bash
pytest tests/                        # all
pytest tests/ -k "never_shed" -v     # the fridge can NEVER be shed — our headline security test
```

---

## The 90-second demo 

1. **Baseline** — clinic green, grid connected, all four load channels supplied, battery full.
2. **Grid fails** — inject `grid_fail`; Monitor flags instability/loss; Planner proposes candidates; **`ISLAND`** fires — the clinic runs on solar + battery. *XAI: "Grid unstable — islanded to protect supply."*
3. **Battery drains** — inject `low_battery`; Planner forecasts depletion; sheds **P4 then P3** (`SHED_LOAD` admin PC, `DELAY_LOAD` water pump); **the fridge (P1) is untouched.** *XAI: "Shed admin power to keep the vaccine fridge alive ~6 h longer."*
4. **Attack** — inject `attack` (spoofed "healthy" telemetry from a compromised meter); Security **quarantines** the node; grid keeps running on adjacent-node estimation. **The fridge never notices.**
5. **Recover** — inject `recover`; grid stable for *N* readings; **`RECONNECT_GRID`** fires; loads restored; audit log full.

**Record a backup video of this on Day 2 evening. Never demo live without a fallback.**

---

## Repository contents (the Step-2 deliverables)

| File | Deliverable | What's in it |
|---|---|---|
| [`CLAUDE.md`](./CLAUDE.md) | **System scope + agent roles** | Positioning, the 6-agent OS, permissions matrix, the deterministic-vs-XAI decision, dev commands, team split, 2-day runway |
| [`spec.md`](./spec.md) | **Inputs, outputs + constraints** | Problem→gap→users→case study, telemetry schema, topology, command whitelist, priority hierarchy, zero-trust guardrails, RAID, community value, SDG mapping |
| `README.md` | **How to run / understand** | This file |
| *(deck)* | **Live presentation** | 8-min pitch — built separately |

---

## Scope & honesty notes

- **Simulation is the assignment,** not a shortcut — the grid, PV, battery, sensors and relays are simulated; the agents, security path, audit log, XAI and dashboard are real.
- **Generative AI explains; deterministic logic decides.** No LLM sits in the command path — a deliberate safety choice, not a limitation ([`CLAUDE.md` §2.1](./CLAUDE.md)). Any cloud-LLM / OpenAI dependency is removed from the runtime; the explainer runs locally with a template fallback so it works offline.
- **No `setInsecure()` and no hardcoded credentials** in any committed device code; pin the gateway certificate (`setCACert`) or document TLS-pinning as the known production step in [`docs/RAID.md`](./docs/RAID.md).
- **Verify before pitch day:** confirm the CEB figures against the primary CEB media release, and show the grid⇄island transfer point on your architecture slide so the self-healing story is visible.

---

## Sources (case study)

- CEB statement on the "Sunny Sunday" cascading failure — [EconomyNext](https://economynext.com/sri-lankas-ceb-makes-statement-on-sunny-sunday-cascading-failure-205838/)
- Expert committee confirms reason for the 9 Feb blackout — [Ada Derana](https://www.adaderana.lk/news.php?nid=107687)
- Causes and preventive measures — [LNW Lanka News Web](https://lankanewsweb.net/archives/69799/sri-lankas-island-wide-power-failure-causes-and-preventive-measures/)

---

## Team

See [`CLAUDE.md` §4](./CLAUDE.md) for the 5–6-member division of labour and the 2-day runway.
