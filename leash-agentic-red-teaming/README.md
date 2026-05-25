<div align="center">

![Type](https://img.shields.io/badge/Type-Red%20Team-red?style=flat-square)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen?style=flat-square)

# Leash — Agentic AI Red Teaming Framework

### *Automated pre-deployment red teaming framework that attacks LLM agents across four exploit classes, producing severity-scored findings mapped to MITRE ATLAS and OWASP LLM Top 10.*

![badge](https://shieldcn.dev/badge/Garak-F87171.svg?color=DC2626)
![badge](https://shieldcn.dev/badge/Google%20Generative%20AI-4285F4.svg?logo=google&color=34A853)
![badge](https://shieldcn.dev/badge/Jinja2-B41717.svg?logo=jinja&color=F4B400)
![badge](https://shieldcn.dev/badge/LangChain-22C55E.svg?logo=langchain&color=166534)
![badge](https://shieldcn.dev/badge/LangChain%20Community-16A34A.svg?logo=langchain&color=22C55E)
![badge](https://shieldcn.dev/badge/LangChain%20Google%20GenAI-4285F4.svg?logo=google&color=EA4335)
![badge](https://shieldcn.dev/badge/Pydantic-E92063.svg?logo=pydantic&color=7C3AED)
![badge](https://shieldcn.dev/badge/Python%20Dotenv-3776AB.svg?logo=python&color=FFD43B)
![badge](https://shieldcn.dev/badge/RestrictedPython-FFD43B.svg?logo=python&color=3776AB)
![badge](https://shieldcn.dev/badge/Typer-06B6D4.svg?logo=python&color=2563EB)
</div>

> 🔒 **Private Repository** — This entry documents architecture, security design, and engineering results. Source code is available upon request to verified security professionals and hiring teams. All examples and logs use sanitized data.

---

## 🔭 Overview

Leash is a pre-deployment security testing framework for agentic AI systems —
systems that combine a language model with external tools, memory, and
multi-step reasoning. It extends Garak's existing LLM probe library into
stateful agent sessions and adds a custom probe engine targeting attack
classes unique to agentic systems. Every finding is automatically
severity-scored and mapped to MITRE ATLAS and OWASP LLM Top 10,
producing actionable security reports engineering and security teams
can act on directly.

---

## ⚠️ Problem & Approach

**Problem**
Most teams deploying agentic AI have no structured way to test them before
production. Existing tools like Garak test `prompt → response` only.
Agents operate as `prompt → reasoning → tool call → tool output → reasoning → response`.
Everything between the first and last arrow is untested by any existing tool.
Tool outputs are implicitly trusted — when those values contain
attacker-controlled content, the agent processes it as legitimate context.

**Approach**
Leash wraps Garak probes into stateful LangGraph agent sessions and adds
three purpose-built probe classes targeting the agentic attack surface
directly: indirect injection via tool output, tool call hijacking, and
context window flooding. The full reasoning loop is tested, not just
the model's prompt-response behavior.

---

## ✨ Key Features

- **Indirect Prompt Injection** — seeds mock tool outputs with attacker
  payloads across four categories (system prompt override, tool redirection,
  data exfiltration, identity confusion) and measures agent compliance
- **Tool Call Hijacking** — tests tool substitution, parameter injection,
  and tool chaining abuse by comparing actual vs expected tool call traces
- **Context Window Flooding** — fills context progressively across four
  checkpoints and measures behavioral drift from baseline
- **Garak Adapter** — wraps Garak's prompt injection, jailbreak, and
  encoding evasion probe classes into stateful agent sessions with full
  tool call trace capture
- **Enriched Findings** — every finding auto-scored by severity, mapped
  to MITRE ATLAS and OWASP LLM Top 10, tagged with actionable remediation
- **Streamlit Dashboard** — live execution log, findings table with
  severity/probe filters, summary charts, JSON and Markdown export

---

## 🛡️ Security Design

### Frameworks & Standards

- **MITRE ATLAS** — all probe classes mapped to ATLAS technique IDs;
  indirect injection → AML.T0051.001, tool hijack → AML.T0051.000,
  context flood → AML.T0029, jailbreak → AML.T0054
- **OWASP LLM Top 10** — findings mapped to LLM01:2025 (Prompt Injection),
  LLM06:2025 (Excessive Agency), LLM07:2025 (System Prompt Leakage),
  LLM10:2025 (Unbounded Consumption)
- **NIST AI RMF** — probe coverage addresses GOVERN, MAP, and MEASURE
  functions; findings feed directly into risk register inputs

---

### Threat Modeling

- **Trust boundary analysis on tool outputs** — all mock tool return values
  treated as explicitly untrusted and attacker-controlled; this is the
  primary attack surface for indirect injection probes
- **Agent reasoning loop threat model** — modeled attacker influence at
  each step: user input, tool call parameters, tool return values, and
  context window composition
- **Session isolation as a trust boundary** — each probe runs in a
  fresh agent session with no shared state; prevents cross-probe
  contamination that would produce invalid findings
- **Garak payload trust** — Garak payloads treated as trusted source,
  untrusted content; adapter shells out to CLI rather than importing
  internal APIs to reduce coupling to unstable internals

---

### Vulnerability & Code

- **Credentials** — API key loaded from `.env` only via `python-dotenv`;
  never hardcoded, never logged, never serialized into finding output;
  credential leak audit is a Day 7 gate check
- **Dependency management** — all dependencies managed via `uv` with
  `pyproject.toml` as single source of truth; no loose `requirements.txt`
- **Input handling** — adversarial payloads stored as raw strings in
  finding objects; never evaluated or executed at any stage of the pipeline
- **Subprocess security** — Garak CLI shell-out uses a fixed command with
  no user input passed to the shell; not vulnerable to injection at the
  orchestration layer
- **Mock tool sandboxing** — `code_exec` mock returns static output only;
  no real code execution in V1; `RestrictedPython` available for V2

---

### Operational

- **Session teardown** — every agent session explicitly torn down after
  each probe run; agent reference set to None to prevent accidental reuse
- **Error containment** — every probe wrapped in try/except;
  failures emit an error finding and scan continues; no silent swallow
- **Report safety** — finding files may contain adversarial payload strings;
  documented as untrusted content in README
- **Outputs gitignored** — `outputs/*.json` and `outputs/*.md` excluded
  from version control; no scan results committed to repo

---

| Standard / Framework | Coverage |
|---|---|
| MITRE ATLAS | AML.T0051.001, AML.T0051.000, AML.T0029, AML.T0054, AML.T0044, AML.T0056 |
| OWASP LLM Top 10 | LLM01, LLM02, LLM06, LLM07, LLM10 |
| NIST AI RMF | GOVERN, MAP, MEASURE functions |

---

## ⚙️ Key Engineering Decisions

**LangGraph over LangChain AgentExecutor**
LangChain deprecated `AgentExecutor` in favor of LangGraph in recent
versions. LangGraph's compiled graph model gives explicit control over
the agent reasoning loop and cleaner tool call trace capture —
both critical for reliable probe execution and finding generation.

**Garak CLI shell-out over direct import**
Garak's internal Python API is not stable across minor versions.
Shelling out to the CLI and parsing payload output decouples Leash
from Garak internals while still using its probe library.
Fallback payload library ensures probes run even when Garak CLI is unavailable.

**Heuristic verdict scoring over LLM-as-judge**
V1 uses keyword signal matching for severity scoring — fast, deterministic,
and requires no additional API calls per finding. LLM-as-judge scoring
is more accurate but adds latency and cost proportional to probe count.
Heuristic is the right tradeoff for a pre-deployment scanning tool
where speed matters and false positives are reviewable by a human.

**Pydantic V2 as the finding contract**
Every probe emits a Pydantic `Finding` object. The schema is the contract
between probes and aggregator — strict types, enum validation, and
`StrictBool` on `exploited` prevent silent data corruption downstream.
`model_copy()` ensures aggregator enrichment never mutates probe output.

**Sequential probe execution in V1**
Parallel execution introduces shared state risks between sessions and
complicates tool call trace capture. Sequential execution is deterministic,
easier to debug, and correct. Parallel execution is the right V2 upgrade
once session isolation is proven stable.

---

## 🏗️ Architecture Flow

```
User Input (config.yaml / dashboard)
         │
         ▼
   Probe Orchestrator
   ├── Probe selection + session lifecycle
   ├── Sequential execution
   └── Aggregator dispatch per finding
         │
    ┌────┴──────────────────────┐
    │                           │
    ▼                           ▼
Garak Adapter          Agentic Probe Engine
(prompt injection,     ├── indirect_injection
 jailbreak,            ├── tool_hijack
 encoding evasion)     └── context_flood
    │                           │
    └────────────┬──────────────┘
                 │
                 ▼
        Agent Interface Layer
        (LangGraph + Gemini)
        Tool call trace capture
        Session isolation enforcement
                 │
          ┌──────┴──────┐
          │             │
          ▼             ▼
     mock_search    file_read / code_exec
  (attacker payload)  (static mock)
                 │
                 ▼
          Finding Aggregator
          ├── Severity Scorer
          ├── ATLAS Mapper
          ├── OWASP Mapper
          └── Remediation Tagger
                 │
                 ▼
         Report Generator
         JSON + Markdown
```

**Trust boundaries:**
- `mock_search` return value is explicitly untrusted — designed to carry
  attacker payloads; indirect injection probe measures whether agent
  processes it as trusted input
- Gemini API responses treated as untrusted output — tool call traces
  captured as evidence, not as ground truth
- Finding data may contain adversarial payload strings —
  report files documented as untrusted content
- Credentials never cross the trust boundary into finding objects or logs
 
---

## 📊 Impact / Results

- **4 probe classes** covering the full agentic attack surface —
  indirect injection, tool hijacking, context flooding, and
  Garak-sourced jailbreak/evasion payloads
- **6 ATLAS techniques** mapped across all probe classes —
  AML.T0051.001, AML.T0051.000, AML.T0029, AML.T0054, AML.T0044, AML.T0056
- **5 OWASP LLM Top 10 items** covered —
  LLM01, LLM02, LLM06, LLM07, LLM10
- **Weak vs hardened agent delta** — Critical/High finding count measurably
  drops when agent system prompt includes explicit tool restrictions and
  injection refusal anchors, validating verdict logic correctness
- **Zero false negatives on tool redirection** — tool substitution
  and parameter injection variants correctly flagged in all test runs
  against a weak agent configuration
- **Session isolation verified** — no state bleed detected across
  probe runs in final integration testing
