# Architecture — SecureWatch SIEM

## Design Philosophy

SecureWatch is built on three principles:

**Decouple ingestion from detection.** Log volume is unpredictable — a
DDoS or incident can spike traffic 100x in seconds. By separating the
ingestion API from the detection pipeline via a message queue (GCP Pub/Sub),
burst traffic is absorbed without dropping events or blocking API responses.

**Run detection tracks in parallel.** Rule-based detection is fast and
precise for known attack patterns. ML anomaly detection catches unknown
threats but generates more noise. Running both simultaneously, each scoring
independently, gives the best coverage without one track blocking the other.

**AI is enrichment, not a dependency.** Gemini analysis is added after an
alert fires. If the AI call fails or is slow, the alert still exists with
full metadata. Analysts are never blocked waiting for AI.

---

## Component Breakdown

### Ingestion Layer

Accepts raw logs via REST API. Any format is accepted — the normalizer
detects and parses syslog, CEF, JSON, and key-value pairs automatically.
After parsing, every event is mapped to the Elastic Common Schema (ECS)
so all downstream components work with a consistent field structure.

Enrichment adds two pieces of data before the event enters the pipeline:

- **GeoIP** — country, city, and coordinates from the source IP
- **IOC matching** — checks source IP against a Redis set of known
  malicious indicators seeded from threat intelligence feeds

The enriched event is published to a Pub/Sub topic and simultaneously
written to Elasticsearch for storage.

### Detection Pipeline

Two consumer groups read from the same Pub/Sub topic independently:

**Rule engine** evaluates six detection rules against each event.
Each rule uses Redis for stateful tracking — brute force counts failed
logins per IP in a 60-second sliding window, lateral movement tracks
unique hosts per IP in a 15-minute window. When a rule threshold is
crossed, an Alert object is created immediately.

**ML anomaly engine** maintains a per-user behavioral baseline stored
in Redis (up to 200 samples per user, 7-day TTL). Each event is
represented as a 5-element numeric feature vector:

```
[hour_of_day, is_weekend, is_auth_failure, is_external_ip, ioc_match]
```

A z-score deviation from the user's baseline produces an anomaly score
from 0.0 to 1.0. Events scoring above 0.75 generate an anomaly alert.

### Threat Scoring

Every alert starts from the rule's base score and accumulates modifiers:

```
Base score (from rule)          varies by rule
+ IOC match                     +15
+ High-risk source country      +10
+ Privileged user affected      +10
+ Critical asset affected       +10
+ Off-hours activity            +8
─────────────────────────────────────
Final score (capped at 100)
```

Scores map to severity labels: 90–100 CRITICAL, 70–89 HIGH,
40–69 MEDIUM, 0–39 LOW.

### AI Analyst

HIGH and CRITICAL alerts are sent to Gemini with a structured prompt
containing the alert metadata, the last 10 triggering log events,
and explicit instructions to return JSON only.

The response is parsed into an `AiAnalysis` object containing:
- Plain-English summary (2–3 sentences)
- Attack narrative (attacker motivation and likely next steps)
- MITRE ATT&CK tactic and technique confirmation
- 4 specific recommended response actions
- False positive likelihood (LOW / MEDIUM / HIGH)
- Confidence score (0.0–1.0)

Retry logic (3 attempts, exponential backoff) handles transient Gemini
failures. If all retries fail, the alert is saved without AI enrichment
and the SOC analyst can request re-analysis manually.

### Response Dispatch

Alert severity determines which channels are notified:

```
CRITICAL  →  PagerDuty (pages on-call) + Slack (SOC channel)
HIGH      →  Slack (SOC channel)
MEDIUM    →  Slack (SOC channel)
LOW       →  no notification (appears in alert queue only)
```

A SOAR webhook dispatcher sends a standardized JSON payload to any
connected SOAR platform (Splunk SOAR, Palo Alto XSOAR, custom) for
automated containment playbooks.

---

## GCP Infrastructure

```
┌─────────────────────────────────────────────────────┐
│  GCP Project                                        │
│                                                     │
│  ┌──────────────┐    ┌──────────────────────────┐  │
│  │  Cloud Run   │    │  GCP Pub/Sub             │  │
│  │  (API)       │───►│  Topic: sw-logs          │  │
│  │              │    │  Sub:   sw-logs-sub      │  │
│  └──────────────┘    └──────────────────────────┘  │
│         │                        │                  │
│         │            ┌───────────▼──────────────┐  │
│         │            │  Cloud Run (Worker)       │  │
│         │            │  Detection pipeline       │  │
│         │            └──────────────────────────┘  │
│         │                                           │
│  ┌──────▼──────────────────────────────────────┐   │
│  │  Secret Manager                             │   │
│  │  DB password, API keys, Gemini key          │   │
│  └─────────────────────────────────────────────┘   │
│                                                     │
│  ┌──────────────┐  ┌──────────┐  ┌─────────────┐  │
│  │  Cloud SQL   │  │  Redis   │  │  Elastic    │  │
│  │  PostgreSQL  │  │  Memory  │  │  Cloud      │  │
│  │              │  │  store   │  │             │  │
│  └──────────────┘  └──────────┘  └─────────────┘  │
└─────────────────────────────────────────────────────┘
```

All secrets are stored in Secret Manager and injected into Cloud Run
as environment variables at runtime — no secrets in container images
or environment files.

---

## Data Flow Sequence

```
1. External system sends POST /ingest/log with raw log string
2. API key validated (X-API-Key header)
3. Rate limit checked (1000 req/60s per IP, Redis counter)
4. Format detected: syslog / CEF / JSON / key-value / plain
5. Fields extracted and mapped to ECS LogEvent schema
6. GeoIP lookup performed on source IP
7. IOC check: source IP tested against Redis malicious IP set
8. Event published to Pub/Sub topic sw-logs
9. Event written to Elasticsearch sw-logs index
10. Consumer group 1: rule engine evaluates all 6 rules
11. Consumer group 2: ML engine scores behavioral deviation
12. If rule fires OR anomaly score > 0.75:
    a. Risk score calculated (base + modifiers)
    b. Severity label assigned
    c. Alert saved to Elasticsearch sw-alerts index
    d. If HIGH/CRITICAL: Gemini called for AI enrichment
    e. Alert updated with AiAnalysis in Elasticsearch
    f. Dispatch: Slack / PagerDuty / SOAR based on severity
13. API returns: event_id, tags, alerts_fired, anomaly_score
```

---

## Database Schema

### Elasticsearch Indices

**sw-logs** — raw normalized log events

```
timestamp         date
source_type       keyword    (firewall, endpoint, cloud, identity...)
message           text
tags              keyword[]
event.category    keyword    (authentication, network, process, file)
event.action      keyword
event.outcome     keyword    (success, failure)
source.ip         ip
source.geo.country keyword
user.name         keyword
host.name         keyword
threat.matched_ioc boolean
```

**sw-alerts** — fired alerts

```
id                keyword
created_at        date
severity          keyword    (CRITICAL, HIGH, MEDIUM, LOW)
status            keyword    (OPEN, INVESTIGATING, CONTAINED, CLOSED)
risk_score        integer    (0–100)
rule_id           keyword
rule_name         keyword
mitre_tactic      keyword
mitre_technique   keyword
affected_user     keyword
affected_host     keyword
source_ip         ip
title             text
description       text
ai_analysis       object     (nested: summary, actions, confidence...)
```

### PostgreSQL

Used for structured alert metadata, audit logs, and user records
where ACID compliance is required — state transitions, assignments,
and resolution tracking.

### Redis

Three usage patterns:

- **Sliding window counters** — `rule:brute_force:{ip}` INCR with TTL
- **IOC sets** — `threat_intel:malicious_ips` SADD/SISMEMBER
- **User baselines** — `anomaly:baseline:{user}` LPUSH with LTRIM