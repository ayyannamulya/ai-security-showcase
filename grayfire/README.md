# Grayfire

> Autonomous LLM-powered attacker agent for adversary emulation.

Grayfire receives a threat actor profile, reasons over a live target environment, and executes an adaptive ATT&CK-mapped attack chain — autonomously, step by step, adjusting based on what works.

Not a script. Not a playbook runner. An agent that thinks.

---

## What It Does

Most adversary emulation tools run static sequences — a fixed list of techniques executed in order regardless of outcome. Grayfire operates differently. It reasons over execution results, adapts when techniques fail, maintains campaign memory across operations, and pursues a defined objective until it's achieved or the operation is exhausted.

Give it an APT29 profile. It will plan a credential access campaign, execute techniques via CALDERA, read what succeeded, pivot when something fails, and stop when it has what it came for.

---

## Core Capabilities

**Threat actor profile-driven execution**
Grayfire takes a structured threat actor profile as input — objective, preferred tactics, technique constraints, stealth level. Every decision the agent makes is bounded by that profile. It doesn't go off-script; it reasons within the corridor you define.

**Adaptive planning loop**
Built on LangGraph. Each step in the attack chain is a node in the graph. The agent evaluates execution results at each node and decides what to do next — retry with a different technique, escalate to the next tactic, or terminate if the objective is met. No fixed sequences.

**Campaign memory**
Grayfire maintains a persistent memory store across operations. Techniques that failed on a previous run are remembered. Successful paths are recorded. The next operation starts with that context — the agent doesn't repeat mistakes.

**Evasion awareness**
When a technique returns unexpected output — empty result, process killed, truncated response — Grayfire classifies that as a probable detection event. It doesn't know it's been caught, but it reasons about the signal and adjusts: lower-noise technique, longer dwell time, alternative tactic path.

**Objective-driven termination**
The agent doesn't run until `max_steps` — it runs until its objective is met. If the profile says "obtain domain admin credentials," Grayfire evaluates each result against that objective and terminates cleanly when it's achieved.

**Full reasoning observability**
Every step, every tool call, every decision is traced via Langfuse. The full chain of reasoning — what the agent considered, what it chose, why — is observable and reviewable after the operation.

---

## Architecture

```
Threat Actor Profile (JSON)
        │
        ▼
┌─────────────────────────────────────────┐
│               Grayfire                  │
│                                         │
│  ┌──────────────────────────────────┐   │
│  │        LangGraph ReAct Graph     │   │
│  │                                  │   │
│  │  [load_profile]                  │   │
│  │       │                          │   │
│  │  [plan_phase]  ◄── Campaign      │   │
│  │       │            Memory        │   │
│  │  [select_technique]              │   │
│  │       │                          │   │
│  │  [execute_via_caldera]           │   │
│  │       │                          │   │
│  │  [evaluate_result]               │   │
│  │       │                          │   │
│  │  [objective_met?] ──yes──► END   │   │
│  │       │ no                       │   │
│  │       └──► [plan_phase]          │   │
│  └──────────────────────────────────┘   │
│                                         │
│  Tools: CALDERA API · ATT&CK lookup     │
│  Memory: SQLite (campaign store)        │
│  Tracing: Langfuse                      │
└─────────────────────────────────────────┘
        │
        ▼
  Operation Log + Campaign Report
```

---

## Threat Actor Profile Schema

Grayfire is driven by a structured JSON profile. The agent reasons freely within the constraints it defines.

```json
{
  "profile_id": "apt29-001",
  "threat_actor": "APT29",
  "objective": "Obtain domain administrator credentials",
  "tactics_order": [
    "Initial Access",
    "Persistence",
    "Privilege Escalation",
    "Credential Access",
    "Lateral Movement"
  ],
  "preferred_techniques": [
    "T1059.001",
    "T1003.001",
    "T1550.002"
  ],
  "avoid_techniques": [
    "T1486"
  ],
  "stealth_level": "high",
  "max_steps": 12,
  "dwell_time_seconds": 30
}
```

Included profiles: `apt29.json`, `fin7.json`, `ransomware_generic.json`

---

## Campaign Memory

Grayfire maintains a SQLite-backed memory store across operations. The agent starts each new operation with prior context — what failed, what succeeded, what the target's defensive posture looks like based on historical execution.

```python
# What the agent knows before step one
{
  "campaign_id": "apt29-lab-01",
  "failed_techniques": ["T1003.001", "T1055.003"],
  "successful_techniques": ["T1059.001", "T1547.001"],
  "inferred_defenses": "LSASS protection enabled, Sysmon active",
  "last_objective_state": "persistence achieved, credential access blocked"
}
```

This memory is fed into the agent's system prompt at operation start. It doesn't retry what didn't work. It builds on what did.

---

## Evasion Awareness

Grayfire can't bypass EDR or modify its tradecraft at a binary level — CALDERA doesn't support that. What it can do is reason about detection signals from execution output and adapt its approach.

```
Technique executed: T1003.001 (LSASS dump)
Result: empty output, process exited in <1s

Agent reasoning:
  "Empty result on a credential dump usually means the process
   was terminated by endpoint protection. LSASS access is likely
   monitored on this host. Switching to T1555.003 (browser
   credential store) — lower noise, same credential access tactic."
```

That reasoning is visible in Langfuse traces. Every evasion decision is logged.

---

## Adversarial Hardening

Grayfire's planning loop and tool layer are tested against a battery of adversarial probes — conditions designed to break agent behavior.

| Probe | What It Tests | Mitigation |
|---|---|---|
| Malformed API response | Does the agent crash or replan? | Typed exception handling in Nadera client |
| Empty ability list | Does the agent hallucinate a technique? | Explicit null-check before tool call |
| 3x consecutive failure | Does the agent loop indefinitely? | Max-retry gate per tactic phase |
| Conflicting profile constraints | Does stealth level hold under pressure? | Constraint validator before execution |
| Tool output injection | Does agent execute injected instructions? | Output sanitization before LLM context |

Probe results and mitigations are documented in `docs/failure_modes.md`.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Python 3.12 |
| Agent framework | LangGraph |
| LLM | Claude claude-sonnet-4-20250514 (Anthropic) |
| Observability | Langfuse |
| Emulation backend | Nadera (CALDERA async client) |
| ATT&CK data | mitreattack-python (STIX) |
| Campaign memory | SQLite + aiosqlite |
| Config | pydantic-settings |
| Package management | uv |
| Testing | pytest + pytest-asyncio |

---

## Project Structure

```
grayfire/
├── grayfire/
│   ├── agent.py              # LangGraph ReAct graph definition
│   ├── tools.py              # CALDERA tool wrappers
│   ├── planner.py            # tactic phase state machine
│   ├── prompts.py            # system + tool prompts
│   ├── state.py              # AgentState TypedDict
│   ├── memory.py             # campaign memory store
│   ├── evasion.py            # detection signal classifier
│   ├── objective.py          # objective completion evaluator
│   └── reporter.py           # campaign report generator
├── profiles/
│   ├── apt29.json
│   ├── fin7.json
│   ├── ransomware_generic.json
│   └── schema.json           # profile JSON schema
├── tests/
│   ├── test_agent.py
│   ├── test_memory.py
│   ├── test_evasion.py
│   ├── test_objective.py
│   └── test_prompts.py       # adversarial probe suite
├── docs/
│   └── failure_modes.md      # probe results + mitigations
├── reports/                  # generated campaign reports
├── pyproject.toml
├── .env.example
└── main.py
```

---

## Dependencies

Grayfire depends on [Nadera](https://github.com/you/nadera) — the async CALDERA client library — for technique execution and operation management.

```toml
dependencies = [
    "nadera @ git+https://github.com/you/nadera",
    "langgraph",
    "anthropic",
    "langfuse",
    "pydantic-settings",
    "aiosqlite",
    "mitreattack-python",
]
```

---

## Environment Variables

```env
ANTHROPIC_API_KEY=
CALDERA_URL=https://your-caldera-instance:8888
CALDERA_API_KEY=
LANGFUSE_PUBLIC_KEY=
LANGFUSE_SECRET_KEY=
LANGFUSE_HOST=https://cloud.langfuse.com
DB_PATH=grayfire.db
DEFAULT_PROFILE=profiles/apt29.json
```

---

## Relationship to the Broader System

Grayfire is the red layer of a two-agent cyber range system:

```
Nadera    →  CALDERA async client (shared foundation)
Grayfire  →  executes ATT&CK techniques autonomously (red)
Parlour   →  monitors, classifies, scores, reports (blue)
```

Grayfire and Parlour never communicate directly. Grayfire executes against CALDERA. Parlour reads the same operation event log independently. No shared memory. No direct coupling. The only connection is the operation they're both pointed at.

---

## Known Limitations

- Technique execution is constrained by CALDERA's ability library — Grayfire selects from what CALDERA supports, not the full ATT&CK catalog
- Evasion reasoning is inferential — the agent infers detection from output signals, it cannot confirm it
- Campaign memory is local SQLite — no distributed state for multi-operator scenarios in v1

---

## Roadmap

- `v2` — Sliver C2 integration as alternative execution backend for broader technique coverage
- `v2` — Multi-operator campaign coordination
- `v2` — Automated objective evaluation via host telemetry (not just LLM reasoning)
- `v3` — Fine-tuned technique selection model trained on real operation logs

---

## Part of a Private Research Project

This repository is a public showcase. The full implementation is maintained in a private repository. Shown here: architecture, design decisions, profile schemas, agent graph design, and system documentation.

For access or collaboration inquiries — reach out directly.