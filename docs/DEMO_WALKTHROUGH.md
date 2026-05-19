# Demo Walkthrough

This is the demo flow I use to explain the project in interviews or reviews.

## 1. Explain the Problem

ERP systems hold high-value records such as student data, attendance, fees, grades, identity information, and admin actions. A normal ERP backend can authenticate users, but it often lacks real-time detection, correlation, and response.

## 2. Show the Architecture

Open the architecture diagram and explain the separation between:

- ERP portal
- Middleware gateway
- ERP API
- IDS service
- Admin dashboard
- Persistence layer

![ERP IDS architecture](../assets/diagrams/erp_ids_architecture.png)

## 3. Show Normal ERP Usage

Use the ERP student dashboard to show that the protected application is realistic and not only a security dashboard.

![ERP student dashboard](../assets/screenshots/erp-student-dashboard.png)

## 4. Explain Prevention

Show the blocked login flow and explain that risky sources can be stopped before reaching protected ERP workflows.

![Blocked ERP login](../assets/screenshots/erp-blocked-login.png)

## 5. Show SOC Monitoring

Open the security posture and incident views. Explain how alerts become incidents and how operators can inspect severity, source IPs, event counts, and timelines.

![Security posture overview](../assets/screenshots/admin-security-posture-overview.png)

![Incidents](../assets/screenshots/admin-incidents.png)

## 6. Explain Analysis Views

Use sequence analysis, attacker profiles, and sessions to explain investigation workflows.

![Sequence analysis](../assets/screenshots/admin-sequence-analysis.png)

![Attacker profiles](../assets/screenshots/admin-attacker-profiles.png)

## 7. Show Model and Pipeline Visibility

Use model health and pipeline monitor screens to show that the ML system is observable.

![Model health](../assets/screenshots/admin-model-health.png)

![Pipeline monitor](../assets/screenshots/admin-pipeline-monitor.png)

## 8. Close With Engineering Tradeoffs

Key tradeoffs to discuss:

- Middleware centralization gives strong visibility but must be highly reliable.
- ML should support detection decisions, not replace deterministic security controls.
- Risk scoring needs explainability so operators can trust the system.
- False positives need feedback and review flows.
- Production use would require stronger secret handling, CI, monitoring, and policy enforcement.
