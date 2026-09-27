# Demo walkthrough

The flow I use to present DELOS in a review or interview. It takes about 10 minutes live and works
from the screenshots alone if a live demo isn't possible.

## 1. The problem (1 min)

ERP systems hold identities, grades, attendance, fees and admin actions. They authenticate users
and keep logs, but nothing inspects traffic in real time, links related events, or stops an
attacker mid-attempt.

## 2. The architecture (2 min)

Show the diagram in the [README](../README.md#architecture) and make three points:

- The **gateway is the only public service**. The ERP rejects traffic that didn't pass through it.
- The **IDS is internal** and decides; the gateway enforces.
- The **dashboard talks only to the gateway**, never to the database.

## 3. The protected ERP is real (1 min)

Sign in to the portal as a student. The point is that this is a working ERP with real workflows
(courses, attendance, fees, library), not only a security dashboard.

![ERP student dashboard](../assets/screenshots/erp-student-dashboard.png)

## 4. Normal traffic stays clean (1 min)

Run the traffic generator. It signs in as the demo users and browses like real people, then
reports how many normal requests were wrongly blocked. This is the false-positive check, and it's
the number I care about most: a security layer that blocks students doesn't get deployed.

## 5. Attack it (2 min)

Live, in this order:

1. Request `/.env` → honeypot hit, the source is blocked instantly.
2. Send a SQL injection payload to a search endpoint → WAF signature, request blocked.
3. Sign in with a wrong password several times → the brute-force detector fires on the login
   outcomes and blocks the IP.
4. As a student, call an admin endpoint → role boundary violation.

Then open the SOC dashboard; the live feed has already shown every one of them.

![SOC overview](../assets/screenshots/soc-overview.png)

## 6. Triage and investigate (2 min)

- **Alerts**: repeats are grouped per source and type; risk and block status are visible.
- **Incident**: open the brute-force incident. Walk through the kill-chain progress, the alert
  that opened it, and the timeline (opened by rule → alert → IP blocked). Show owner, status and
  the unblock button.
- **Attacker profile**: everything about one source in one place.

![Alerts](../assets/screenshots/soc-alerts.png)

![Incident detail](../assets/screenshots/soc-incident-detail.png)

![Attacker detail](../assets/screenshots/soc-attacker-detail.png)

## 7. Tuning and ML (1 min)

- **Correlation rules** are data, not code: toggle one, edit its threshold.
- **Detection models** page: model status, thresholds, WAF signature groups, and the export of
  analyst-labelled alerts for retraining. If the models aren't trained, the dashboard says so and
  detection continues on signatures and rules; that's a deliberate design choice, not a failure.

![Correlation rules](../assets/screenshots/soc-rules.png)

![Detection models](../assets/screenshots/soc-models.png)

## 8. Close with trade-offs

- Centralising enforcement in the gateway gives full visibility, but the gateway must be reliable,
  so it fails closed in production and uses a circuit breaker.
- ML supports decisions; deterministic controls make the high-confidence ones. An anomaly score
  alone never blocks.
- Scoring is explainable by construction, so operators can trust it and tune it.
- The version I show is a rewrite. I removed features that looked good but weren't real (see
  [Design decisions](DESIGN_DECISIONS.md#the-rewrite-v1--v2)).
