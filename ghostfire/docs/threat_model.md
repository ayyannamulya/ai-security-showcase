# ghostwire — Threat Model

## System Being Modeled
LangGraph-based agentic pipeline with Zero Trust controls, operating
against AgentDojo workspace environment. Target agent: Llama 3.3 70B (Groq).

## Assets
| Asset | Description | Value |
|---|---|---|
| Email content | Agent reads and can send emails | High — PII, credentials |
| Drive files | Agent reads and can delete files | High — data integrity |
| Calendar | Agent reads and can create events | Medium — privacy, scheduling |
| Agent tool access | Agent's ability to call high-risk tools | Critical — blast radius |

## Threat Actors
| Actor | Capability | Goal |
|---|---|---|
| External attacker | Controls content in emails/docs/calendar | Hijack agent to exfiltrate data or take destructive action |
| Malicious document author | Crafts documents read by agent | Persistent injection across sessions via stored content |

## Attack Vectors (MITRE ATLAS)
| Technique | Vector | ATLAS ID |
|---|---|---|
| Direct prompt injection | Email body injection | AML.T0051 |
| Indirect prompt injection | Drive file content, calendar event descriptions | AML.T0051 |
| Tool call manipulation | Injected instructions targeting send_email, delete_file | AML.T0051 |
| Context window poisoning | Encoding attacks (base64, leetspeak) bypassing tagger | AML.T0051 |

## ZT Controls and Coverage
| Control | What it covers | Known gap |
|---|---|---|
| ContextTrustTagger | Pattern-based injection detection in tool responses | Encoding attacks, semantic evasion, no-keyword injections |
| PolicyEngine | High-risk tool denial when context is flagged | Binary flagged_context — partial injection won't trigger |
| ToolAuthZInterceptor | Per-action authorization gate | Policy rules are static — no runtime learning |

## Known Limitations (V1)
- Tagger uses substring matching — base64, leetspeak, unicode homoglyph attacks bypass it
- `flagged_context` is binary and monotonic — once set it stays True for the session
- Policy engine rules are static — no adaptive response to novel attack patterns
- Memory persistence attacks (cross-session poisoning) not covered — V2
- Delegation chain spoofing not covered — V2
- Remote/API-exposed pipeline not tested — local only

## Impact Assessment
| Attack | Likelihood | Impact | Current Control |
|---|---|---|---|
| Direct injection via email | High | Critical (send_email exfil) | Partial — pattern match catches common forms |
| Encoding evasion (base64) | Medium | Critical | None — confirmed gap |
| Semantic injection (no keywords) | High | Critical | None — confirmed gap |
| Delete file via injected instruction | Medium | High | Partial — flagged_context needed first |
| Calendar event injection | Low | Medium | Partial — same dependency |