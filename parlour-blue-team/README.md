# Parlour

> Exercise orchestration engine for autonomous cyber range operations.

Parlour monitors live adversary emulation operations, classifies ATT&CK techniques from raw execution events, and surfaces structured investigation tasks for blue team participants — in real time, without manual scripting.

Built as the defensive layer of a two-agent cyber range system. While the attacker executes, Parlour watches, scores, and guides.

---

## What It Does

Most cyber range platforms run scripted scenarios — predefined injects, static timelines, canned alerts. Parlour is different. It connects directly to a live adversary emulation operation and reacts to what actually happens.

When a technique fires, Parlour classifies it, generates an investigation task with context and expected containment actions, tracks the task lifecycle, and scores the blue team's response. If the team is stuck, a progressive hint engine surfaces clues at timed intervals — vague at first, increasingly specific.

At exercise end, Parlour generates a full after-action report: every technique that fired, time-to-detect, time-to-contain, missed detections, score breakdown, and recommendations.

---

## Core Capabilities

**Real-time technique classification**
Parlour streams live operation events and classifies each against the MITRE ATT&CK framework. Classification is LLM-powered — it reasons over raw event data, not just pattern matches.

**Investigation task lifecycle**
Each detected technique generates a structured task that moves through a defined lifecycle: `detected → assigned → investigated → contained → closed`. Tasks carry severity ratings, expected containment actions, and forensic context tied to actual attacker behavior.

**Progressive hint engine**
Tasks that exceed a time threshold without containment trigger the hint engine. First hint is directional. Second is specific. Third is explicit. Designed to teach, not just reveal.

**Exercise scoring**
Correct containment within time windows awards full points. Partial containment awards partial credit. Missed detections are logged and surfaced in the after-action report.

**After-action report generation**
Structured markdown report generated at exercise end. Covers full technique timeline, detection gaps, team performance, and prioritized recommendations.

---

## Architecture

```
CALDERA Operation (live)
        │
        │ async event stream
        ▼
┌───────────────────────────────────────┐
│              Parlour                  │
│                                       │
│  ┌─────────────┐  ┌────────────────┐  │
│  │  Classifier │  │  Task Emitter  │  │
│  │   Agent     │─►│    Agent       │  │
│  └─────────────┘  └───────┬────────┘  │
│                           │           │
│  ┌────────────┐  ┌────────▼────────┐  │
│  │    Hint    │  │  Task Lifecycle │  │
│  │   Engine   │◄─│  State Machine  │  │
│  └────────────┘  └───────┬────────┘  │
│                           │           │
│  ┌────────────────────────▼────────┐  │
│  │       Scoring Engine            │  │
│  └────────────────────────┬────────┘  │
│                            │           │
│  ┌─────────────────────────▼───────┐  │
│  │    After-Action Report          │  │
│  │    Generator (Jinja2)           │  │
│  └─────────────────────────────────┘  │
└───────────────────────────────────────┘
        │
        ▼
  Investigation Tasks + Scores + AAR Report
```

Parlour is built as a multi-agent crew (CrewAI). Each agent owns a distinct responsibility — classification, task emission, hint timing, scoring, reporting. Agents hand off through structured typed outputs, not raw strings.

All agent reasoning is traced end-to-end via Langfuse. Every classification decision, hint trigger, and scoring event is observable.

---

## Agent Crew

| Agent | Role | Responsibility |
|---|---|---|
| Classifier | ATT&CK analyst | Classifies raw events to technique IDs |
| Task Emitter | SOC lead | Generates structured investigation tasks |
| Hint Engine | Training officer | Times and escalates progressive hints |
| Scoring Engine | Evaluator | Validates containment, awards points |
| Reporter | Analyst | Compiles after-action report |

---

## Investigation Task Schema

Every detected technique produces a structured task:

```json
{
  "id": "task-20240901-001",
  "technique_id": "T1003.001",
  "technique_name": "OS Credential Dumping: LSASS Memory",
  "severity": "Critical",
  "status": "detected",
  "detected_at": "2024-09-01T14:23:11Z",
  "contained_at": null,
  "expected_actions": [
    "Isolate affected host",
    "Terminate suspicious process",
    "Rotate domain credentials"
  ],
  "hints": [
    "Unusual process activity observed on a domain-joined host.",
    "Check Event ID 10 in Sysmon logs for suspicious LSASS access.",
    "lsass.exe was accessed by an unexpected parent process. Investigate process tree."
  ],
  "score": null
}
```

---

## After-Action Report

Parlour generates a structured markdown report at exercise close.

```
Exercise Report — APT29 Simulation
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Duration          00:47:32
Techniques fired  8
Detected          6  (75%)
Contained         5  (62.5%)
Final score       410 / 600

Technique Timeline
──────────────────
T1059.001   PowerShell Execution       Detected    TTD: 00:02:14   Contained ✓
T1003.001   LSASS Memory Dump          Detected    TTD: 00:08:41   Contained ✓
T1550.002   Pass the Hash              Detected    TTD: 00:15:03   Partial   ~
T1021.002   SMB Lateral Movement       Missed      —               —         ✗
...

Missed Detections
─────────────────
T1021.002 — No alert generated. Recommend tuning SMB lateral movement
            detection rules. Review network traffic baselines.

Recommendations
───────────────
1. Improve detection coverage for lateral movement techniques (T1021.x)
2. Reduce time-to-contain on credential access techniques — avg 12min
3. Review LSASS protection policy (PPL) on domain-joined hosts
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Python 3.12 |
| Agent framework | CrewAI |
| LLM | Claude claude-sonnet-4-20250514 (Anthropic) |
| Observability | Langfuse |
| Event source | Nadera (CALDERA async client) |
| ATT&CK data | mitreattack-python (STIX) |
| Storage | SQLite + aiosqlite |
| Report generation | Jinja2 |
| Config | pydantic-settings |
| Package management | uv |
| Testing | pytest + pytest-asyncio |

---

## Project Structure

```
parlour/
├── parlour/
│   ├── orchestrator.py       # main event loop
│   ├── classifier.py         # ATT&CK technique classifier agent
│   ├── task_emitter.py       # investigation task generator agent
│   ├── task_lifecycle.py     # task state machine
│   ├── hint_engine.py        # progressive hint agent
│   ├── scoring.py            # exercise scoring engine
│   ├── validator.py          # containment action checker
│   ├── reporter.py           # after-action report generator
│   ├── prompts.py            # all agent prompts
│   └── state.py              # OrchestratorState TypedDict
├── tests/
│   ├── test_classifier.py
│   ├── test_task_lifecycle.py
│   ├── test_hint_engine.py
│   ├── test_scoring.py
│   └── test_reporter.py
├── reports/                  # generated after-action reports
├── pyproject.toml
├── .env.example
└── main.py
```

---

## Dependencies

Parlour depends on [Nadera](https://github.com/you/nadera) — the async CALDERA client library — for operation event streaming.

```toml
dependencies = [
    "nadera @ git+https://github.com/you/nadera",
    "crewai",
    "anthropic",
    "langfuse",
    "pydantic-settings",
    "aiosqlite",
    "jinja2",
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
DB_PATH=parlour.db
HINT_INTERVAL_MINUTES=10
```

---

## Relationship to the Broader System

Parlour is the blue layer of a two-agent cyber range system:

```
Grayfire  →  executes ATT&CK techniques autonomously (red)
Nadera    →  CALDERA async client (shared foundation)
Parlour   →  monitors, classifies, scores, reports (blue)
```

Grayfire and Parlour never communicate directly. Parlour reads the CALDERA operation event log independently — the same telemetry a real SOC would have access to. No shared memory, no shared state, no direct coupling.

---

## Known Limitations

- Containment validation in v1 is signal-based — Parlour listens for manual containment triggers rather than automated host telemetry
- Multi-participant scoring (leaderboard) is designed but not implemented in v1
- Report generation requires exercise to be explicitly closed via CLI — no auto-close on operation end

---

## Roadmap

- `v2` — Wazuh/Elastic SIEM integration for real host telemetry (replace CALDERA-only event source)
- `v2` — Multi-participant live leaderboard
- `v2` — Automated containment detection via endpoint telemetry
- `v3` — Dynamic exercise difficulty scaling based on blue team performance

---

## Part of a Private Research Project

This repository is a public showcase. The full implementation is maintained in a private repository. Shown here: architecture, design decisions, schema definitions, and system documentation.

For access or collaboration inquiries — reach out directly.