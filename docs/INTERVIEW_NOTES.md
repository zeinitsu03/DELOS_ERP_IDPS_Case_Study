# Interview Notes

## Short Explanation

DELOS ERP IDPS is a full-stack ERP security prototype. It places a middleware gateway in front of an ERP backend, analyzes traffic using deterministic checks and ML-assisted detection, scores risk, correlates incidents, and presents everything in a SOC-style admin dashboard.

## Why It Is Private

The full source repository is private because it contains the complete backend implementation, security workflows, schema design, deployment structure, and detection logic. Publishing all of that would expose more than is necessary for portfolio review.

The public case study shows architecture, screenshots, design decisions, and demo flow. I can walk through the source live or share access privately when appropriate.

## Strong Talking Points

- It is a full product prototype, not only an ML notebook.
- The middleware design allows normal ERP workflows while adding transparent security enforcement.
- The IDS pipeline combines rule-style checks, ML classification, anomaly scoring, risk scoring, and incident correlation.
- The dashboard makes detection explainable through sessions, model health, pipeline status, incidents, and sequence analysis.
- The ERP profile system makes the design portable beyond one academic ERP scenario.
- Deployment planning includes container and Kubernetes-oriented design.

## What I Built

- ERP portal screens for student/admin workflows.
- Backend APIs for ERP data and security services.
- Middleware gateway for inspection and enforcement.
- IDS service for alerts, risk, incidents, ML, and correlation.
- Admin dashboard pages for monitoring and response.
- Database design for alerts, incidents, SIEM events, audit data, and risk state.
- Architecture diagrams, screenshots, and demo flow.

## Questions I Can Answer

- Why middleware was used instead of embedding checks directly in the ERP API.
- How risk scoring combines endpoint sensitivity and behavior.
- How incidents differ from individual alerts.
- How false positives could be handled.
- What would need to change for production deployment.
- Where ML helps and where deterministic controls are better.
