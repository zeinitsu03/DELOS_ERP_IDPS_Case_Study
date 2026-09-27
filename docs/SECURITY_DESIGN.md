# Security design

DELOS applies defence in depth around a protected ERP: deterministic controls at the gateway,
ML-assisted detection in the IDS, explainable risk scoring, and an operator in the loop through
the SOC dashboard.

## Threat model

| Threat | Example | Primary control | Secondary control |
|---|---|---|---|
| Injection | SQLi / XSS / command injection in query or body | WAF signatures (block) | HTTP model, repeated-injection correlation rule |
| Credential attacks | Password guessing on the login endpoint | Outcome-based brute-force detector, auto-block | Brute-force correlation rule, per-user rate limits |
| Reconnaissance | Scanners, `/.env`, `/wp-admin`, endpoint enumeration | Honeypot paths (instant block), scanner signatures | Reconnaissance-sweep correlation rule |
| Privilege escalation | A student calling admin endpoints; tampered JWT | ERP role rules, JWT `alg=none` signature | Privilege-escalation correlation rule |
| Data scraping | Bulk reads above a role's normal rate | Per-role request-rate limits | Anomaly model |
| Session misuse | One session used from many IPs | Session behaviour detector | Attacker profile view |
| Bypassing the IDS | Calling the ERP directly | ERP accepts only gateway-signed requests | Internal-only network placement |
| IP spoofing | Forged `X-Forwarded-For` to dodge blocks or frame someone | Header trusted only from configured proxies | — |
| Abuse of the defence | Flooding to fill the database | Blocked-IP reports rate-limited per IP | Traffic retention purge |

## Controls

| Control | Purpose |
|---|---|
| Security gateway | Single enforcement point; nothing reaches the ERP without passing it |
| Block list | Instant rejection of known-bad sources; survives IDS outages |
| Honeypot paths | High-confidence early signal of scanning; blocks immediately |
| Rate limits | Per user when signed in, per IP otherwise |
| WAF signatures | Deterministic detection of common web attack payloads, double-decoded input |
| IDS detection | ML models + behaviour detectors + ERP role rules, combined into one risk score |
| Evidence-gated auto-block | Blocks only when strong evidence is present, for a limited time |
| Incident correlation | Groups related alerts into investigable cases on a kill chain |
| SOC dashboard | Triage, investigation, labelling, unblocking, rule editing |
| Notifications | Telegram alerts to contacts who verified with a one-time code |

## Detection engine

### Evidence

Every detector returns evidence: a score in 0..1, a weight, an attack type and a short
explanation. The dashboard shows each item and what it contributed, so every alert can be
explained.

| Source | Weight | Notes |
|---|---|---|
| WAF signatures | 1.0 | Signature groups for SQLi, XSS, path traversal, command injection, SSRF, XXE, template injection, header injection, JWT manipulation, scanners |
| Gateway checks | 1.0 | Honeypot hits, rate-limit blocks, failed-login bursts |
| ERP role rules | 1.0 | Role boundary violations and bulk reads, from the ERP domain profile |
| HTTP model | 1.0 | Primary ML model |
| Anomaly model | 0.5 | Knows "unusual", not "malicious" |
| Flow model | 0.35 | Flow features can only be approximated from one HTTP exchange |

### Risk

```
risk = 1 − ∏ (1 − scoreᵢ · weightᵢ)
```

| Why noisy-OR | Instead of |
|---|---|
| One strong signal is enough: a 0.95 WAF match gives risk ≥ 0.95 | A weighted average, which dilutes it with "all clear" scores from other detectors |
| Weak signals accumulate slowly: two 0.5 anomaly scores at weight 0.5 give 0.44, below the alert threshold | Summing, which lets noise trigger alerts |
| Each contribution is visible and explainable | A stacked meta-model, which would need labelled examples of every detector combination and none exist |

A model's own decision threshold (chosen during training) maps to an evidence score of 0.55, so a
model that is only just over its threshold produces a low-severity alert, not a block.

| Risk | Result |
|---|---|
| ≥ 0.5 (default) | Alert. Severity low, medium (≥ 0.6), high (≥ 0.75) or critical (≥ 0.9) |
| ≥ 0.85 (default) | Request blocked |
| ≥ 0.85 **and** strong evidence contributes ≥ 0.8 | Source IP auto-blocked for a limited time (if auto-block is on) |

**Strong evidence** means a blocking WAF signature, a honeypot hit, a rate-limit block, five or
more failed logins, repeated role violations, or a model at ≥ 0.9 probability. An anomaly score
alone never blocks anyone.

### Behaviour detectors

Brute force, enumeration, bulk reads above the role's limit, sessions used from many IPs, and role
violations. Per source, these merge into **one alert per attack type over a 10-minute window** with
an occurrence counter. Without merging, a busy legitimate user produced dozens of medium alerts,
which the multi-stage rule then chained into a critical incident, and localhost got blocked during
a demo. That bug is why the merge exists.

### Machine learning

| Model | Training data | Role |
|---|---|---|
| HTTP classifier (primary) | CSIC-2010 + synthetic ERP traffic + analyst-labelled alerts | The gateway sees HTTP requests, so a model trained on HTTP requests matches its runtime input |
| Anomaly detector | Normal ERP traffic only | Flags requests unlike this ERP's traffic without needing attack labels; calibrated so about 0.5% of normal traffic scores ≥ 0.5 |
| Flow classifier (secondary) | UNSW-NB15, CIC-IDS | Kept at low weight because flow features are approximated |

Safeguards:

- **One feature module** is imported by both the training scripts and the live IDS. A mismatch
  between training-time and serving-time features breaks detection silently; sharing the code
  removes that risk.
- **Artifacts are self-describing.** Each model stores its feature list and scikit-learn version.
  The IDS refuses to load a model whose features don't match, and warns on a version mismatch.
- **Password fields are masked** before feature extraction, so a password containing `'` or `--`
  doesn't look like SQL injection.
- **The IDS degrades gracefully.** Without models, it still runs on signatures, rules and
  behaviour checks, and the dashboard says so clearly.
- **Metrics are reported honestly.** Scores are reported per data source and, for the flow model,
  leave-one-dataset-out. High accuracy on a public benchmark is not presented as accuracy on real
  ERP traffic.
- **Human-in-the-loop retraining.** Analysts label alerts as true or false positives in the
  dashboard; labelled alerts are exported and fed back into training. There is no automatic
  retraining on unlabelled traffic, which could teach the model that attacks are normal.

### Correlation and incidents

| Built-in rule | Condition (defaults) | Incident | Auto-block |
|---|---|---|---|
| Brute-force login | 1+ brute-force alert in 10 min | High | No |
| Repeated injection attempts | 3+ injection-type alerts (medium+) in 10 min | High | Yes |
| Reconnaissance sweep | 5+ scanner / honeypot / enumeration alerts in 5 min | Medium | No |
| Privilege escalation attempts | 3+ role-violation / JWT-manipulation / method-override alerts (medium+) in 10 min | High | Yes |
| Multi-stage attack | 3+ kill-chain phases with high-severity alerts in 60 min | Critical | Yes |

Rules are stored in the database and editable from the dashboard. Alerts map onto a six-phase kill
chain: reconnaissance → credential access → exploitation → privilege escalation → exfiltration →
impact. Incidents keep a timeline of system actions and analyst notes, a status and an owner.

## Secrets and configuration

- Secrets are generated randomly by a setup script and kept in an untracked `.env`; an example file
  documents every setting.
- In production mode the services **refuse to start** with default or weak secrets, and demo users
  are never created.
- Demo users are only created on an empty database. Existing passwords are never reset on startup
  (the first version did reset them on every start).
- In Kubernetes, secrets come from a Kubernetes Secret; network policies restrict which pods can
  reach the ERP and the IDS.
- Notification contacts must confirm a one-time code before receiving alerts, so a mistyped chat
  ID can't send security alerts to a stranger.

## Hardening roadmap

What I would do before a real production deployment:

- [ ] Move rate-limit, block and behaviour state to Redis so the gateway and IDS can scale out
- [ ] Put secrets in a managed secret store with rotation
- [ ] Add CI: tests, frontend lint/build, dependency and secret scanning
- [ ] Add a model registry with versioned, signed artifacts
- [ ] Add schema migrations (Alembic)
- [ ] Add MFA and per-operator accounts with roles for the SOC dashboard, instead of one admin login
- [ ] Ship structured logs to a central store with a retention policy
- [ ] Tune thresholds and rate limits against real traffic, measuring the false-positive rate
- [ ] Add email / chat-ops notification channels
