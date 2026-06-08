# SecureWatch — AI-Powered SIEM Backend

> A production-grade Security Information and Event Management (SIEM) system
> built with FastAPI, Google Cloud Platform, and Gemini AI for intelligent
> threat detection and automated SOC response.

![Python](https://img.shields.io/badge/Python-3.11-blue?style=flat-square&logo=python)
![FastAPI](https://img.shields.io/badge/FastAPI-0.110-green?style=flat-square&logo=fastapi)
![GCP](https://img.shields.io/badge/GCP-Cloud%20Run-orange?style=flat-square&logo=googlecloud)
![Elasticsearch](https://img.shields.io/badge/Elasticsearch-8.12-yellow?style=flat-square&logo=elasticsearch)
![License](https://img.shields.io/badge/License-MIT-lightgrey?style=flat-square)

---

## Overview

SecureWatch is a backend SIEM platform designed to ingest security logs
from multiple sources, detect threats in real-time using both rule-based
correlation and machine learning anomaly detection, and enrich every alert
with AI-generated analysis via Google Gemini.

This repository is a **public showcase** of the system architecture,
threat detection methodology, API design, and defensive strategies.
The full production codebase is maintained in a private repository.

---

## Key Features

- **Multi-source log ingestion** — accepts syslog, CEF, JSON, and key-value
  formats from firewalls, endpoints, cloud platforms, and identity providers
- **Dual-track detection** — rule-based correlation engine (MITRE ATT&CK
  mapped) running in parallel with ML behavioral anomaly detection (UEBA)
- **AI threat analysis** — every HIGH/CRITICAL alert is automatically
  enriched with a Gemini-generated summary, attack narrative, and
  recommended response actions
- **Natural language threat hunting** — analysts query the log store in
  plain English; Gemini translates to Elasticsearch DSL
- **Automated response dispatch** — Slack, PagerDuty, and SOAR webhook
  integration with severity-based routing
- **REST API** — full OpenAPI 3.0 specification with interactive Swagger UI

---

## System Architecture

```
External Systems (Firewall / EDR / Cloud / IAM)
         │
         ▼
   POST /ingest/log  ◄── API Key Auth + Rate Limiting
         │
         ▼
   Normalizer ──► ECS Schema (format: syslog / CEF / JSON / kv)
         │
         ▼
   Enricher ──► GeoIP lookup + IOC threat intel match
         │
         ▼
   Pub/Sub (GCP) ──► raw-events topic
         │
    ┌────┴────┐
    ▼         ▼
Rule Engine   ML Anomaly Engine
(MITRE Rules) (UEBA Baseline)
    │         │
    └────┬────┘
         ▼
   Threat Scorer (risk 0–100)
         │
         ▼
   Gemini AI Analyst
   (summary + MITRE map + recommended actions)
         │
         ▼
   Alert → Elasticsearch
         │
    ┌────┴────────────┐
    ▼                 ▼
Slack / PagerDuty   SOAR Webhook
```

---

## Detection Rules (MITRE ATT&CK Mapped)

| Rule ID | Rule Name | MITRE Tactic | MITRE Technique | Severity |
|---|---|---|---|---|
| SW-001 | Brute Force Attack | Credential Access | T1110 | HIGH |
| SW-002 | Impossible Travel | Initial Access | T1078 | CRITICAL |
| SW-003 | Lateral Movement | Lateral Movement | T1021 | HIGH |
| SW-004 | Privilege Escalation | Privilege Escalation | T1068 | HIGH |
| SW-005 | Data Exfiltration | Exfiltration | T1041 | CRITICAL |
| SW-006 | Threat Intel IOC Match | Initial Access | T1190 | HIGH |
| SW-ML-001 | Behavioral Anomaly | — | ML-based | varies |

---

## Tech Stack

| Layer | Technology |
|---|---|
| API framework | FastAPI + Uvicorn |
| Event streaming | GCP Pub/Sub |
| Log storage | Elasticsearch 8.12 |
| Alert metadata | PostgreSQL 15 |
| Sliding windows / IOC | Redis 7 |
| AI analysis | Google Gemini 1.5 Flash |
| ML anomaly detection | scikit-learn (Isolation Forest + Z-score) |
| GeoIP enrichment | MaxMind GeoLite2 |
| Containerization | Docker + Cloud Run |
| CI/CD | GitHub Actions + Artifact Registry |
| Infrastructure | Terraform |

---

## Repository Structure

```
securewatch-showcase/
├── README.md               ← this file
├── docs/
│   ├── architecture.md     ← detailed system design
│   ├── detection.md        ← threat detection methodology
│   ├── api.md              ← API reference overview
│   └── swagger.yaml        ← OpenAPI 3.0 specification
├── poc/
│   ├── README.md           ← PoC overview and setup
│   ├── normalizer.py       ← log format detection + ECS normalization
│   ├── rules.py            ← detection rule examples
│   └── scorer.py           ← risk scoring algorithm
└── defense/
    ├── README.md           ← defensive strategy overview
    ├── mitigations.md      ← specific mitigations per attack vector
    └── hardening.md        ← system and infrastructure hardening
```

---

## API Documentation

The full interactive API documentation is available as an OpenAPI 3.0
specification at [`docs/swagger.yaml`](docs/swagger.yaml).

You can view it in:
- **Swagger UI** — paste the YAML into [editor.swagger.io](https://editor.swagger.io)
- **Redoc** — paste into [redocly.github.io/redoc](https://redocly.github.io/redoc/)
- **Portfolio** — embed via Swagger UI CDN (see `docs/api.md`)

---

## Showcase Goals

This project demonstrates:

1. **Backend API design** — RESTful design, auth middleware, rate limiting,
   pagination, structured error responses
2. **Security engineering** — log normalization, threat detection rules,
   IOC matching, MITRE ATT&CK framework application
3. **ML in production** — behavioral baseline building, z-score anomaly
   scoring, per-user UEBA without labeled training data
4. **AI integration** — structured prompt engineering, JSON response parsing,
   retry logic, graceful degradation when AI is unavailable
5. **GCP architecture** — Cloud Run, Pub/Sub, Secret Manager, Artifact
   Registry, Terraform IaC
6. **Engineering practices** — 90+ unit and integration tests, CI/CD
   pipeline, Docker multi-stage builds, load testing

---

## License

MIT — see [LICENSE](LICENSE) for details.
This showcase repository contains no proprietary credentials,
production infrastructure details, or sensitive configuration.