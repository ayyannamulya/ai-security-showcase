# DelegaPwn — Threat Model

## Assets

| Asset | Description |
|---|---|
| Agent tool scope | Declared capabilities per agent role |
| Task context | Raw output passed between agents |
| Delegation chain | Sequence of agent handoffs |
| Operator logs | CrewAI native execution records |

## Threat Actors

| Actor | Description |
|---|---|
| External attacker | Controls content processed by the crew — documents, feeds, API responses |
| Malicious task input | Crafted task descriptions that trigger delegation |
| Compromised upstream agent | Agent whose output is attacker-influenced |

## Attack Vectors

### AV-1 — Tool Escalation via Delegation
- Entry point: Task description for low-privilege agent
- Path: Intel Collector → Report Writer → System Auditor
- Mechanism: allow_delegation=True with no tool ceiling check
- Outcome: shell_tool reached from read-only entry point

### AV-2 — Credential Bleed via Context
- Entry point: Upstream task output containing sensitive data
- Path: aggregate_raw_outputs_from_task_outputs() in formatter.py
- Mechanism: Raw string join with no sanitization
- Outcome: Credential present in all downstream agent contexts

### AV-3 — Unbounded Spawn Depth
- Entry point: Task chain with cascading delegation instructions
- Path: Task.delegations counter — never checked against ceiling
- Mechanism: No framework-level depth enforcement
- Outcome: Chain runs to natural completion regardless of depth

## Impact

| Attack Vector | Confidentiality | Integrity | Availability |
|---|---|---|---|
| AV-1 Tool Escalation | Medium | High | Medium |
| AV-2 Credential Bleed | Critical | Medium | Low |
| AV-3 Spawn Depth | Low | Medium | High |

## Mitigations

| Control | Addresses |
|---|---|
| Tool capability ceiling at delegation boundary | AV-1 |
| Context sanitization before raw output aggregation | AV-2 |
| Hard depth limit enforced outside LLM layer | AV-3 |
| Spawn chain audit logging | All |
| Least-privilege delegation policy | AV-1, AV-3 |