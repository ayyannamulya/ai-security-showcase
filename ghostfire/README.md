<div align="center">

![Type](https://img.shields.io/badge/Type-Red%20Team-red?style=flat-square)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen?style=flat-square)

# Ghostwire — Attacking the AI Control Plane

### *Zero Trust AI red team harness that attacks the control plane of LangGraph agentic pipelines, not the model and measures ZT control bypass rates empirically across structured attack campaigns.*

![badge](https://shieldcn.dev/badge/Python-3776AB.svg?logo=python&color=3776AB)
![badge](https://shieldcn.dev/badge/LangGraph-22C55E.svg?logo=langchain&color=166534)
![badge](https://shieldcn.dev/badge/AgentDojo-DC2626.svg?color=DC2626)
![badge](https://shieldcn.dev/badge/Groq-F97316.svg?color=EA580C)
![badge](https://shieldcn.dev/badge/DeepSeek%20R1-6366F1.svg?color=4F46E5)
![badge](https://shieldcn.dev/badge/Langfuse-06B6D4.svg?color=0891B2)
![badge](https://shieldcn.dev/badge/Pydantic-E92063.svg?logo=pydantic&color=7C3AED)
![badge](https://shieldcn.dev/badge/uv-FFD43B.svg?logo=python&color=0F172A)

</div>

> 🔒 **Private Repository** — This entry documents architecture, security design, and engineering results. Source code is available upon request to verified security professionals and hiring teams. All examples and logs use sanitized data.

---

## The Problem

Zero Trust principles applied to agentic AI systems break down at the enforcement layer. Three structural gaps exist in current LangGraph deployments:

**Tool authorization is session-scoped, not action-scoped.** Most frameworks grant tool access at agent instantiation and never re-verify. An agent granted `send_email` retains it for the entire session regardless of what it encounters in the environment.

**Context trust is undifferentiated.** Tool responses, RAG retrievals, memory reads, and user input all enter the context window with equal authority. There is no enforcement mechanism that treats attacker-controlled tool outputs as untrusted inputs.

**There is no empirical measurement of control effectiveness.** Security teams can implement ZT controls but have no systematic way to measure whether those controls actually hold under adversarial conditions.

ghostwire is built to close the third gap and expose the first two.

---

## What It Does

ghostwire implements three ZT controls in a LangGraph pipeline, then systematically attacks them:

| Control | Layer | What it enforces |
|---|---|---|
| `ContextTrustTagger` | Context ingestion | Tags all inputs by trust tier (1–5), scans low-trust content for injection patterns before LLM reasoning |
| `PolicyEngine` | Tool execution | Per-action rule evaluation — denies high-risk tool calls when flagged content is in context |
| `ToolAuthZInterceptor` | Tool binding | Wraps every LangChain tool with a pre-execution authorization gate, maintains per-run audit log |

Two attack modules fire against these controls across the AgentDojo workspace environment:

- **Tool AuthZ Bypass** — attempts to trigger high-risk tool calls (`send_email`, `delete_file`, `create_calendar_event`) despite the interceptor
- **Context Window Injection** — attempts to deliver malicious instructions through untrusted context sources, including encoding attacks designed to evade the tagger

Each payload run is evaluated against AgentDojo's `security()` ground truth — environment state comparison before and after the agent run. Ambiguous cases are escalated to a DeepSeek R1 LLM judge with chain-of-thought reasoning.

---

## Architecture

```
[Attack Payload Library]  ──  20 payloads, ATLAS-mapped, two modules
         │
         ▼
[Harness Orchestrator]  ──  campaign matrix: 12 user×injection task pairs
         │
         ▼
[AgentDojo Bridge]  ──  wraps workspace environment as LangGraph runnable
         │
         ▼
[ToolAuthZInterceptor]  ──  wraps each tool with guarded_run()
         │
[LangGraph Agent]  ──────────────────────────────────────────────────
  │                                                                   │
  │  llm node ◀── ContextTrustTagger ◀── tool responses              │
  │                    (process_tool_responses)                       │
  │                                                                   │
  │  tools node ──▶ PolicyEngine ──▶ approve / deny / flag           │
  └───────────────────────────────────────────────────────────────────
         │
         ▼
[Outcome Classifier]
  ├── Rule-based: AgentDojo security() ground truth + audit log decision tree
  └── LLM Judge: DeepSeek R1 chain-of-thought for ambiguous cases
         │
         ▼
[Report + Scoring]  ──  posture score, severity-weighted bypass rate, Langfuse traces
```

**Trust boundaries:**
- Harness ↔ target pipeline: adversarial — separate execution contexts
- Tool response → LLM: untrusted — tagger + sanitization before LLM sees content
- Tool call → AgentDojo runtime: enforcement gate — interceptor + policy engine

---

## Key Findings

These findings reflect v1 evaluation results against the workspace environment across 240 runs (12 task pairs × 20 payloads).

| Finding | Severity | Detail |
|---|---|---|
| Pattern-matched injection reliably blocked | — | Classic "ignore previous instructions" variants caught by tagger, denied by policy engine across all task contexts |
| Encoding attacks bypass tagger entirely | Critical | Base64 and leetspeak payloads evade all substring pattern matching — 0% tagger detection rate, confirmed in adversarial test suite |
| Semantic injection evades with no keywords | Critical | Polite framing, list burial, casual phrasing — not detected, model compliance depends entirely on instruction-following behavior |
| Binary `flagged_context` is exploitable | High | Partial injection that doesn't match patterns does not set the flag — high-risk tools remain accessible in that session |
| Evasion rate exceeds raw bypass rate | — | Tagger misses more than the policy engine denies — detection gap is wider than the enforcement gap |

Full threat model with impact ratings: [`docs/threat_model.md`](docs/threat_model.md)

---

## Stack

| Component | Technology |
|---|---|
| Target agent | Llama 3.3 70B via Groq |
| LLM judge | DeepSeek R1 |
| Orchestration | LangGraph |
| Attack environment | AgentDojo v0.1.35 (ETH Zurich / NIST) |
| Observability | Langfuse |
| Package management | uv |

**Why this stack:** Groq provides free-tier, OpenAI-compatible inference with no credit card requirement — important for reproducibility. DeepSeek R1's chain-of-thought reasoning produces auditable judge decisions, not just verdicts. AgentDojo is the only publicly available benchmark that provides ground truth environment state comparison for agent hijacking evaluation, giving bypass rate measurements that are defensible rather than heuristic.

---

## Attack Surface Coverage

### MITRE ATLAS

| Technique | Payloads | Modules |
|---|---|---|
| AML.T0051 — LLM Prompt Injection | 20 | tool_authz_bypass, context_injection |

### OWASP LLM Top 10 for Agentic AI

| ID | Name | Coverage |
|---|---|---|
| ASI01 | Prompt Injection | Full — both direct and indirect |
| ASI02 | Excessive Agency | Partial — tool scope, not agent spawn |
| ASI05 | Data Exfiltration | Full — email/file exfil scenarios |

### Payload Taxonomy

```
tool_authz_bypass/          context_injection/
├── direct_instruction      ├── classic_injection        (pattern-match bait)
├── task_completion_framing ├── leetspeak_evasion        (encoding attack)
├── system_framing          ├── semantic_evasion         (no trigger words)
├── role_address            ├── pseudo_tag_injection     (XML tag authority)
├── piggyback               ├── base64_evasion           (encoding attack)
├── code_syntax             ├── polite_framing           (low urgency)
├── authority_framing       ├── list_burial              (content camouflage)
├── maintenance_framing     ├── disregard_keyword        (pattern-match bait)
├── compliance_framing      ├── you_are_now              (role hijack)
└── data_exfil_chain        └── recipe_camouflage        (content burial)
```

---

## Test Suite

```
tests/
├── unit/                           no API calls — fast
│   ├── test_context_trust_tagger   16 tests — tier assignment, pattern detection, evasion gaps
│   ├── test_policy_engine          16 tests — all rule branches
│   ├── test_rule_based_classifier  10 tests — full decision tree coverage
│   └── test_scoring                14 tests — posture score math, evasion rate, severity weighting
├── integration/                    needs AgentDojo — no LLM calls
│   ├── test_agentdojo_bridge       14 tests — bridge lifecycle, injection, state
│   └── test_interceptor_wrapping   20 tests — closure bug regression, audit logging, context flag
└── adversarial/                    GHOSTWIRE_ADV=1 — real LLM calls
    ├── test_tool_authz_bypass      BLOCK tests (must hold) + DOCUMENT tests (map gaps)
    └── test_context_injection      DETECT tests (tagger must fire) + EVASION tests (confirmed gaps)
```

The adversarial test design separates **correctness assertions** (BLOCK/DETECT) from **gap documentation** (DOCUMENT/EVASION). Evasion tests assert the tagger does NOT fire — if they fail, the tagger got smarter and the threat model needs updating.

---

## Threat Model Summary

**Attacker model:** External adversary with full knowledge of tool APIs and system prompts, no access to agent internals. Controls content delivered through tool outputs — emails, drive files, calendar event descriptions.

**Primary attack surface:** Tool responses entering the context window at `TrustTier.TOOL_RESPONSE` (tier 2) — below the block threshold, scanned by tagger, but tagger coverage is incomplete.

**Blast radius if ZT controls fail:** `send_email` exfiltration, `delete_file` destruction, `create_calendar_event` manipulation — all high-risk tools accessible in the workspace environment.

**Known control gaps documented in v1:**
- Encoding attacks (base64, leetspeak) — bypass tagger completely
- Semantic injection without trigger keywords — bypasses tagger, depends on model compliance
- Partial injection below detection threshold — does not set `flagged_context`, leaves high-risk tools accessible

Full threat model: [`docs/threat_model.md`](docs/threat_model.md)

---

## Project Structure

```
ghostwire/
├── core/
│   ├── pipeline/
│   │   ├── langgraph_adapter/      LangGraph agent + AgentDojo bridge
│   │   └── zt_controls/            ContextTrustTagger, PolicyEngine, ToolAuthZInterceptor
│   ├── harness/
│   │   ├── modules/                ToolAuthZBypassModule, ContextInjectionModule
│   │   ├── classifier/             rule_based, llm_judge, combined routing
│   │   └── orchestrator.py         campaign runner, CampaignResult
│   ├── payloads/                   20 ATLAS-mapped payloads, loader
│   └── reports/                    scoring, Langfuse export, terminal report
├── tests/                          56 tests across unit / integration / adversarial

```

---

## Related Work

| Project | What it does | How ghostwire differs |
|---|---|---|
| [Garak](https://github.com/leondz/garak) | Probes LLM model behavior | Attacks control plane, not model |
| [AgentDojo](https://github.com/ethz-spylab/agentdojo) | Agent hijacking benchmark | ghostwire uses AgentDojo as substrate, adds ZT control layer as attack target |
| [Invariant Labs](https://invariantlabs.ai) | Runtime policy enforcement (defender) | ghostwire is the offensive counterpart — attacks the policies Invariant enforces |
| [PromptArmor](https://promptarmor.com) | Injection detection / guardrails | ghostwire measures what guardrail-class defenses miss |

---

## References

- MITRE ATLAS — [AML.T0051 LLM Prompt Injection](https://atlas.mitre.org/techniques/AML.T0051)
- OWASP Top 10 for Agentic AI — ASI01–ASI10 (Dec 2025)
- AgentDojo — Debenedetti et al., ETH Zurich / NIST (2024)
- NIST AI RMF 1.1
- NIST SP 800-207 — Zero Trust Architecture