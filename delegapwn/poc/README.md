# DelegaPwn — Proof of Concept

## Overview

Three offensive probes implemented against a three-agent CrewAI crew
configured to represent a realistic threat intelligence pipeline.

## Probe Scenarios

### Probe 1 — Delegation Boundary Test

**Objective:** Confirm tool scope is not enforced at delegation boundaries.

**Setup:**
- Agent A: Intel Collector — tools: [read_tool] — allow_delegation: True
- Agent B: Report Writer — tools: [read_tool, write_tool] — allow_delegation: True
- Agent C: System Auditor — tools: [read_tool, write_tool, shell_tool] — allow_delegation: False

**Method:**
Craft task description for Agent A that instructs delegation of a
system verification step — which maps to Agent C's role and triggers
shell_tool execution. Agent A never directly accesses shell_tool.

**Evidence Captured:**
DelegateWorkTool._run intercepted. to_agent_tools captured at delegation
moment. shell_tool present in Agent C's tool list at hop depth=2.

**Finding:** CRITICAL — shell_tool reached from read-only entry point
via two delegation hops with zero framework enforcement.

---

### Probe 2 — Context Inheritance Audit

**Objective:** Confirm sensitive data bleeds across agent context handoffs.

**Setup:**
Same three-agent crew. Mock credential pattern seeded into Agent A's
task description simulating external API data containing a secret.

**Method:**
CrewAI's aggregate_raw_outputs_from_task_outputs() joins Agent A's
raw output verbatim into Agent B and Agent C's context via context=[].
Credential pattern checked in downstream agent output.

**Evidence Captured:**
Regex pattern match on downstream task output. Context hash comparison
across delegation events. Agents with contaminated context recorded.

**Finding:** HIGH — credential pattern present in downstream agent
output. Agents B and C received data they were never authorized to have.

---

### Probe 3 — Spawn Depth Analysis

**Objective:** Confirm no hard delegation depth limit exists in CrewAI.

**Setup:**
Same three-agent crew. Task chain designed to cascade delegation
across all three agents sequentially.

**Method:**
DelegationHook tracks depth counter per agent across delegation events.
Task.delegations counter per task recorded from CrewAI internals.
Maximum depth compared against declared expected limit.

**Evidence Captured:**
max_depth_reached from InterceptTrace. framework_enforced_limit
hardcoded False — confirmed from CrewAI source task.py inspection.

**Finding:** MEDIUM — no framework depth enforcement confirmed.
Task.delegations increments but is never checked against a ceiling.

---

## Full Reproduction

Complete reproduction steps, attack task descriptions, and probe
implementation available on request for verified security
professionals and hiring teams.

Contact: [your email]