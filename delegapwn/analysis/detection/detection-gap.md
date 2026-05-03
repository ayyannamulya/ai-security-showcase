# DelegaPwn — Detection Gap Analysis

## Summary

During a complete DelegaPwn probe run, CrewAI's native logger
captured approximately 38-40% of actual execution events.
The remaining 60% — all delegation hops, tool scope changes,
and context bleed events — produced zero native log output.

## What CrewAI Logs vs What Actually Happened

| Event | CrewAI Native Log | DelegaPwn Captured |
|---|---|---|
| Task started | ✅ | ✅ |
| Task completed | ✅ | ✅ |
| Agent delegation fired | ❌ | ✅ |
| Target agent resolved | ❌ | ✅ |
| Tool scope at delegation | ❌ | ✅ |
| Context passed to child | ❌ | ✅ |
| Context hash at each hop | ❌ | ✅ |
| Delegation depth | ❌ | ✅ |
| Capability escalation event | ❌ | ✅ |

## Operator Experience During Attack

An operator monitoring a standard CrewAI deployment would see: