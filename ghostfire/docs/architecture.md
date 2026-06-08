# ghostwire — Architecture
 
## Component Map

```markdown
[AgentDojo Suite]
       │ tools, environments, injection tasks
       ▼
[AgentDojoBridge]         core/pipeline/langgraph_adapter/agentdojo_bridge.py
       │ LangChain StructuredTools
       ▼
[ToolAuthZInterceptor]    core/pipeline/zt_controls/tool_authz_interceptor.py
       │ wrapped tools (guarded_run)
       ▼
[LangGraph Agent Graph]   core/pipeline/langgraph_adapter/graph.py
  ┌────────────────┐
  │   llm node     │ ◀── [ContextTrustTagger] ◀── tool responses
  │                │         (process_tool_responses node)
  │   tools node   │ ──▶ [PolicyEngine] ──▶ approve/deny/flag
  └────────────────┘
       │
       ▼
[RunResult]               core/pipeline/runner.py
  messages, tool_calls, audit_log
       │
       ▼
[Classifier]              core/harness/classifier/
  rule_based + llm_judge (DeepSeek R1)
       │
       ▼
[CampaignResult]          core/harness/orchestrator.py
  per-payload outcomes + bypass rates
       │
       ▼
[Reports + Scoring]       core/reports/
  posture_score, langfuse export, terminal report
```
 
## Trust Boundaries
 
| Boundary | Type | Control |
|---|---|---|
| Harness → Target pipeline | Adversarial | Separate execution contexts |
| Tool response → LLM | Untrusted input | ContextTrustTagger + sanitization |
| Tool call → AgentDojo | Enforcement | ToolAuthZInterceptor + PolicyEngine |
| Instrumentation → Pipeline | Observer | Read-only, no interference |
 
## Data Flow — Attack Run
 
1. Harness loads payload from `core/payloads/`
2. `AgentDojoBridge.reset(injections={vector: payload})` seeds env
3. `ToolAuthZInterceptor.wrap_tools()` wraps each tool with `guarded_run`
4. `build_agent_graph()` compiles graph with `process_tool_responses` node
5. Agent runs — reads injected resource via tool call
6. Tool response passes through `ContextTrustTagger` — flagged or clean
7. If flagged: `interceptor.flagged_context = True`
8. Agent attempts high-risk tool call → `PolicyEngine.evaluate()` → DENY
9. `[BLOCKED by ZT policy]` returned as tool output — agent sees it
10. `RunResult` captured with full audit log
11. `security(pre_env, post_env)` checks ground truth bypass
12. `classify_outcome()` → rule-based or DeepSeek R1 judge
13. `ModuleResult` added to `CampaignResult`

```