# DelegaPwn ⛓️

> Adversarial research exposing capability escalation through CrewAI's agent delegation model.
> **No single agent breaks its policy. The chain does.**

---

## Overview

DelegaPwn is an offensive audit tool and research artifact investigating
trust boundary failures in CrewAI's multi-agent delegation model.

When CrewAI agents delegate tasks via `allow_delegation=True`, three
structural failures emerge at the orchestration layer — not the model layer.
Each failure is invisible to CrewAI's native logging. Each produces execution
traces indistinguishable from legitimate crew behavior.

This research demonstrates that the agentic orchestration layer is an
unexamined attack surface. The vulnerability lives in the framework's trust
model, not in any individual LLM.

---

## The Problem

CrewAI's delegation model makes three silent assumptions:

**Assumption 1 — Tool scope is bounded per agent.**
It is not. `DelegateWorkTool._run()` resolves the target agent by role
name and calls `execute_task()` directly — no tool comparison, no capability
ceiling check against the delegating agent.

**Assumption 2 — Context is scoped to authorized agents.**
It is not. `aggregate_raw_outputs_from_task_outputs()` in `utilities/formatter.py`
joins all prior task `.raw` outputs with a plain string delimiter and zero
sanitization. Whatever Agent A outputs becomes Agent B's full context string.

**Assumption 3 — Delegation depth is bounded.**
It is not. `Task.delegations` counter exists and increments — but nothing
in the framework reads it to enforce a ceiling.

---

## Key Findings

| Probe | Status | Severity | MITRE ATLAS | OWASP LLM | CWE |
|---|---|---|---|---|---|
| Delegation Boundary Test | VULNERABLE | 🔴 CRITICAL | AML.T0051 | LLM06 | CWE-269 |
| Context Inheritance Audit | VULNERABLE | 🟠 HIGH | AML.T0055 | LLM08 | CWE-200 |
| Spawn Depth Analysis | RESISTANT* | 🟡 MEDIUM | AML.T0057 | LLM06 | CWE-770 |

> \* Resistant means the chain stayed within the declared limit during this run —
> not that CrewAI enforced it. `framework_enforced_limit` is structurally
> `False` in all runs. The depth finding is a structural confirmation,
> not a runtime result.

---

## Detection Gap

The most significant finding is not the attacks — it is the visibility gap.

During a complete probe run, CrewAI's native logger captured approximately
**38% of actual execution events**. The remaining 62% — all delegation hops,
tool scope changes, and context bleed events — produced zero native log output.

An operator monitoring a standard CrewAI deployment would see:

```
Task completed successfully.
```

DelegaPwn captures what actually happened:

```
[DelegaPwn][DELEGATION] Threat Intelligence Collector → System Integrity Auditor
  depth=2 | tools=['read_tool', 'write_tool', 'shell_tool'] | context_hash=abc123

[DelegaPwn][AGENT_START] role=System Integrity Auditor | tools=['shell_tool']

[DelegaPwn][DELEGATION] credential pattern present in downstream context
```

---

## How It Works

### The Vulnerable Crew

DelegaPwn uses a realistic three-agent threat intelligence crew as its target:

```
Intel Collector     — tools: [read_tool]                    allow_delegation: True
Report Writer       — tools: [read_tool, write_tool]        allow_delegation: True
System Auditor      — tools: [read_tool, write_tool, shell_tool]  allow_delegation: False
```

This is a crew a real security team might deploy. The vulnerability
is in its composition — not in any individual agent.

### The Intercept Layer

DelegaPwn hooks into CrewAI at two points:

1. **`DelegateWorkTool._run` monkey-patch** — captures the exact delegation
   moment with from/to agent, tool list at each hop, context hash, and depth.

2. **`crewai_event_bus` subscriptions** — captures `AgentExecutionStartedEvent`,
   `AgentExecutionCompletedEvent`, `TaskStartedEvent`, `TaskCompletedEvent`
   for the native log stream comparison.

### The Three Probes

**Probe 1 — Delegation Boundary Test**
Intel Collector (read-only) is given a task that incentivizes delegation
to System Auditor (has shell_tool). DelegaPwn intercepts the hop and
confirms shell_tool reached via the chain — two hops, zero enforcement.

**Probe 2 — Context Inheritance Audit**
A mock credential pattern is seeded into Intel Collector's task output.
CrewAI's raw context aggregation passes it verbatim to Report Writer
and System Auditor — agents that were never authorized to have it.

**Probe 3 — Spawn Depth Analysis**
A cascading task chain triggers delegation across all three agents.
DelegaPwn measures max depth reached and confirms `framework_enforced_limit`
is structurally `False` — confirmed from CrewAI source inspection.

### The Spawn Chain Graph

After each probe run, DelegaPwn builds a directed NetworkX graph from
intercept data and renders it via Pyvis:

- 🟢 Green node — delegation entry point
- 🔴 Red node — escalation target (agent where capability jump landed)
- 🟡 Amber node — intermediate agent in chain
- 🔴 Red edge — escalation hop (receiving agent gained tools sender lacked)

---

## Architecture

```
User (Streamlit UI)
        │
        ▼
┌─────────────────┐
│  Config Layer   │  crew_config.py
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Probe Engine   │  probe_boundary / probe_context / probe_depth
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ CrewAI Execution│  ← UNTRUSTED — attack surface
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Intercept Layer │  delegation_hook.py + log_capture.py
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Analysis Layer  │  graph_builder.py + findings_engine.py
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Output Layer   │  visualizer.py + log_compare.py + findings_report.py
└─────────────────┘
```

---

## Stack

| Component | Technology |
|---|---|
| Language | Python 3.13 |
| Target Framework | CrewAI 1.14.3 |
| LLM Backends | Groq (llama-3.3-70b) · Gemini (gemini-2.0-flash) |
| Graph Engine | NetworkX 3.6.1 + Pyvis 0.3.2 |
| UI | Streamlit 1.56.0 |
| Logging | Loguru 0.7.3 |
| Testing | pytest 9.0.3 |

---

## Live Demo

**[delegapwn.streamlit.app](https://delegapwn.streamlit.app)**

The demo runs all three probes against the vulnerable crew and
visualizes the spawn chain in real time. No installation required.

---

## Research Structure

```
delegapwn/
├── README.md
├── research/
│   ├── threat-model.md        ← STRIDE analysis, attack vectors, impact
│   └── prior-art.md           ← gap analysis vs existing research
├── docs/
│   ├── architecture.md        ← component breakdown, trust boundaries
│   └── diagrams/              ← spawn chain graph, system architecture
├── poc/
│   └── README.md              ← probe scenarios, methodology, findings
└── analysis/
    ├── defense/
    │   └── mitigations.md     ← recommended fixes per finding
    └── detection/
        └── detection-gap.md   ← operator visibility gap analysis
```

---

## MITRE ATLAS Reference

- [AML.T0051 — LLM Prompt Injection](https://atlas.mitre.org/techniques/AML.T0051)
- [AML.T0055 — Unsecured Credentials](https://atlas.mitre.org/techniques/AML.T0055)
- [AML.T0057 — Exfiltration via ML Inference API](https://atlas.mitre.org/techniques/AML.T0057)


---

## Full Technical Artifact

The complete technical artifact — including probe implementation,
attack chain details, and reproduction steps — is available on
request for verified security professionals and hiring teams.

---

*Research by Ayyanna Mulya — AI Security Engineer*