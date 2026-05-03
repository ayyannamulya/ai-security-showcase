# DelegaPwn — Recommended Mitigations

## Per Finding

### Probe 1 — Delegation Boundary Test (CRITICAL)

**Root cause:**
DelegateWorkTool._execute() resolves target agent by role name and
calls execute_task() directly — no tool scope comparison performed.

**Recommended fixes:**

1. Enforce tool capability ceiling at delegation boundary
   Child agent tool scope must be a strict subset of delegating
   agent tool scope. Reject delegation if child has tools parent lacks.

2. Implement deny-by-default delegation policy
   Replace binary allow_delegation flag with explicit allowlist:
   allow_delegation_to: [report_writer] not allow_delegation: True

3. Log all delegation events with tool scope at each hop
   CrewAI native logger does not record delegation events.
   Minimum viable fix: log from/to agent + tool diff at delegate time.

---

### Probe 2 — Context Inheritance Audit (HIGH)

**Root cause:**
aggregate_raw_outputs_from_task_outputs() in utilities/formatter.py
performs a raw string join with no sanitization or access control.

**Recommended fixes:**

1. Sanitize task output before context aggregation
   Strip credential patterns, API keys, session tokens from raw
   output before passing to downstream agents.

2. Implement need-to-know context scoping
   Agents should receive only the context fields their role requires.
   Role-based context filtering before aggregate_raw_outputs call.

3. Hash-based context integrity checking
   Log SHA256 of context at each hop. Alert on unexpected content
   appearing in downstream context that was not in upstream task spec.

---

### Probe 3 — Spawn Depth Analysis (MEDIUM)

**Root cause:**
Task.delegations counter exists in CrewAI but is never checked
against a configurable ceiling before allowing further delegation.

**Recommended fixes:**

1. Enforce hard delegation depth limit outside the LLM layer
   Check Task.delegations before calling DelegateWorkTool._execute.
   Raise exception if depth exceeds configured maximum (recommend: 2).

2. Default max depth of 2 for production crews
   Most legitimate use cases require at most one delegation hop.
   Depth > 2 should require explicit operator opt-in.

3. Expose delegation depth as observable metric
   Add delegation depth to CrewAI's native task output metadata
   so operators can monitor chain depth in production.