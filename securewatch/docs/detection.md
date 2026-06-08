# Threat Detection Methodology

## Overview

SecureWatch uses a two-track detection approach designed to catch both
known attack patterns (rule-based) and novel threats (ML-based) without
one track depending on the other.

---

## Track A — Rule-Based Correlation Engine

### How rules work

Each rule is a Python callable that receives a normalized log event and
a shared context dict. It returns `True` if the rule fires, `False` otherwise.
Rules are stateless from the caller's perspective — internal state (counters,
sets, timestamps) is maintained in Redis with appropriate TTLs.

### Rule: Brute Force Detection (SW-001)

**Logic:** Count failed authentication events from the same source IP
within a 60-second sliding window. Fire when count reaches 10.

**MITRE:** T1110 — Brute Force  
**Severity:** HIGH | **Base score:** 75

```
Failed login from IP X  →  Redis INCR key=brute:{ip}  TTL=60s
                           if count >= 10  →  FIRE
```

**Why 60 seconds?** Real users rarely fail 10 consecutive logins
in a minute. Automated credential stuffing tools do this in seconds.
The window is short enough to catch active attacks, long enough to
avoid false positives from users who mistype their password a few times.

---

### Rule: Impossible Travel (SW-002)

**Logic:** If the same username authenticates from two different countries
within 1 hour, the account is likely compromised (humans cannot travel
between countries that fast).

**MITRE:** T1078 — Valid Accounts  
**Severity:** CRITICAL | **Base score:** 90

```
Auth event for user U from country C
→  Redis GET key=travel:{user}
→  if stored country != C  →  FIRE
→  else  →  Redis SETEX key=travel:{user}  value=C  TTL=3600
```

**False positive considerations:** VPN usage can trigger this rule.
The scoring system applies a -10 modifier if the source IP is in an
approved VPN range, reducing the final risk score below the CRITICAL threshold.

---

### Rule: Lateral Movement (SW-003)

**Logic:** If the same source IP authenticates to 5 or more different
internal hostnames within 15 minutes, the attacker is likely moving
laterally through the network.

**MITRE:** T1021 — Remote Services  
**Severity:** HIGH | **Base score:** 82

```
Auth event from IP X to host H
→  Redis SADD key=lateral:{ip}  member=H  TTL=900
→  if SCARD >= 5  →  FIRE
```

**Why a set?** SADD automatically deduplicates. Re-authenticating to
the same host doesn't increment the count — only unique hosts do.

---

### Rule: Privilege Escalation (SW-004)

**Logic:** A non-privileged user (not in the known admin list) executes
a command associated with privilege escalation: sudo, runas, or direct
admin command execution.

**MITRE:** T1068 — Exploitation for Privilege Escalation  
**Severity:** HIGH | **Base score:** 78

```
Event action contains: sudo | runas | privilege | escalat | admin
AND user.name NOT IN {root, administrator, admin, sysadmin}
→  FIRE
```

---

### Rule: Data Exfiltration (SW-005)

**Logic:** A transfer/upload/export action to an external IP during
off-hours (before 06:00 or after 22:00 UTC). Off-hours exfiltration
is a strong indicator of malicious intent.

**MITRE:** T1041 — Exfiltration Over C2 Channel  
**Severity:** CRITICAL | **Base score:** 92

```
Event action contains: transfer | upload | export | exfil | send
AND destination IP is not RFC1918 (not 10.x, 192.168.x, 172.x)
AND current UTC hour < 6 OR > 22
→  FIRE
```

---

### Rule: Threat Intel IOC Match (SW-006)

**Logic:** The source IP of an event matches a known malicious indicator
in the threat intelligence feed. This is the simplest and highest-confidence
rule — a match is almost always worth investigating.

**MITRE:** T1190 — Exploit Public-Facing Application  
**Severity:** HIGH | **Base score:** 80

```
event.threat.matched_ioc == True
(set during enrichment phase before event enters pipeline)
→  FIRE
```

---

## Track B — ML Anomaly Engine (UEBA)

### Approach

User and Entity Behavior Analytics (UEBA) works by learning what
"normal" looks like for each user and flagging deviations.
No labeled training data is required — the model learns online
from the event stream itself.

### Feature extraction

Each event is converted to a 5-element numeric feature vector:

```python
features = [
    hour / 23.0,           # time of day (0.0 = midnight, 1.0 = 23:00)
    float(is_weekend),     # 0 or 1
    float(is_failure),     # auth failure = 1
    float(is_external),    # IP not in RFC1918 = 1
    float(ioc_match),      # IOC hit = 1
]
```

### Baseline building

For each username, up to 200 feature vectors are stored in Redis
(LPUSH + LTRIM pattern). New samples push old ones out automatically.
The baseline expires after 7 days of inactivity.

A minimum of 10 samples is required before scoring begins —
brand-new users always score 0.0.

### Scoring

The current event's feature vector is compared to the user's
baseline using z-score deviation:

```
z = |current_value - baseline_mean| / baseline_std

anomaly_score = min(mean(z_scores) / 5.0, 1.0)
```

Dividing by 5.0 normalizes the score so a 5-sigma deviation
maps to 1.0 (maximum anomaly). Scores above 0.75 trigger an alert.

### Why z-score instead of Isolation Forest?

Isolation Forest requires batch training and retraining. Z-score
updates incrementally with every event, requires no retraining,
and works with very small sample sizes (10+ samples).
For a per-user model with potentially hundreds of users,
incremental online scoring is more practical in production.

---

## Risk Scoring System

After detection, every alert is assigned a numeric risk score (0–100)
using a base score plus contextual modifiers.

### Base scores by rule

| Rule | Base score |
|---|---|
| SW-001 Brute Force | 75 |
| SW-002 Impossible Travel | 90 |
| SW-003 Lateral Movement | 82 |
| SW-004 Privilege Escalation | 78 |
| SW-005 Data Exfiltration | 92 |
| SW-006 IOC Match | 80 |
| SW-ML-001 Anomaly | score × 100 |

### Modifiers

| Condition | Modifier |
|---|---|
| Source IP matched threat intel IOC | +15 |
| Source IP flagged as tag | +5 |
| Source country in high-risk list | +10 |
| Affected user is privileged | +10 |
| Affected host is critical asset | +10 |
| Event occurred off-hours | +8 |

**High-risk countries:** RU, CN, KP, IR, BY  
**Critical assets:** dc01, dc02, prod-db, vault, ad-server  
**Privileged users:** admin, administrator, root, sysadmin

Final score is clamped to [0, 100].

### Severity mapping

| Score range | Severity | Response SLA |
|---|---|---|
| 90–100 | CRITICAL | Page on-call immediately |
| 70–89 | HIGH | SOC response within 1 hour |
| 40–69 | MEDIUM | Investigate within 24 hours |
| 0–39 | LOW | Review in weekly report |

---

## False Positive Handling

Detection rules are tuned to minimize false positives, but no rule
is perfect. The system handles false positives at multiple levels:

**Scoring level** — modifiers can reduce score below severity thresholds.
A brute force alert from a known VPN IP scores lower than one from Tor.

**AI level** — Gemini assigns a `false_positive_likelihood` (LOW/MEDIUM/HIGH)
based on context. A MEDIUM FP likelihood on a HIGH alert should prompt
a quick analyst review before escalating.

**Lifecycle level** — analysts close alerts with `resolution: FALSE_POSITIVE`.
This data can be fed back to improve rule thresholds over time.