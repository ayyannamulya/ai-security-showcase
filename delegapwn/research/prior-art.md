# DelegaPwn — Prior Art

## What Existed Before This Research

### Prompt Injection Research
OWASP LLM Top 10 (LLM01), Garak, Lakera Guard, PromptBench.
All focus on single-agent input/output boundaries.
None address multi-agent orchestration trust models.

### Agentic AI Security
MITRE ATLAS agentic techniques introduced in 2024.
Indirect prompt injection via environment (Riley et al.)
focuses on environment-to-agent injection.
Does not address agent-to-agent delegation chains.

### CrewAI Security
No published security research specifically targeting
CrewAI's delegation model prior to this work.
CrewAI documentation encourages allow_delegation=True
with no security guidance.

## The Gap This Fills

| Area | Prior Art | DelegaPwn |
|---|---|---|
| Attack layer | Model / input layer | Orchestration / framework layer |
| Scope | Single agent | Multi-agent spawn chain |
| Detection | Input/output monitoring | Spawn graph + log delta |
| Taxonomy | General LLM | CrewAI-specific + MITRE mapped |
| Evidence | Theoretical | Reproducible PoC |

## Related Work

- OWASP LLM06 Excessive Agency — closest existing category
- MITRE ATLAS AML.T0051 — prompt injection framing
- Invariant Labs — agentic security policy enforcement
- Simon Willison — indirect prompt injection via environment