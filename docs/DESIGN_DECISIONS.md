# Design decisions & Q&A

The trade-offs behind DELOS, and answers to the questions reviewers ask most often.

## In one paragraph

DELOS is an intrusion detection and prevention system for a university ERP. A gateway in front of
the ERP runs cheap deterministic checks first (block list, honeypots, rate limits, WAF), then asks
an IDS engine for a decision. The IDS combines signature, behaviour, role-rule and ML evidence into
one explainable risk score, raises alerts, groups them into incidents with correlation rules, and
auto-blocks sources only when strong evidence is present. A SOC dashboard shows everything live and
lets an operator investigate, label, unblock and tune.

## Key decisions

| Decision | Alternative considered | Why |
|---|---|---|
| Separate gateway in front of the ERP | Checks embedded in the ERP code | Enforcement can't be skipped by one forgotten route; the ERP stays focused on business logic; the same gateway pattern works for other ERPs |
| Gateway enforces, IDS decides | One combined service | The gateway stays fast and simple; detection can evolve (and fail) without taking the ERP down with it |
| Noisy-OR evidence scoring | Weighted average; stacked meta-model | Strong signals aren't diluted, weak ones accumulate slowly, every alert is explainable, nothing extra to train |
| Evidence-gated auto-block | Block on risk threshold alone | Prevents a single model or anomaly score from locking out legitimate users |
| Rate limits per user, not per IP | Per-IP limits | Per-IP throttled 17% of normal requests in testing when a classroom shared one NAT address |
| Merge behaviour alerts per source and type | One alert per event | A busy legitimate user once cascaded into a critical incident and blocked localhost mid-demo |
| One shared feature module for training and serving | Separate feature code | Training/serving skew silently breaks detection; shared code makes it impossible |
| Manual retraining with labelled feedback | Automatic drift-triggered retraining | Unlabelled production traffic can contain attacks; auto-retraining can learn them as normal |
| Fail open in dev, fail closed in prod | Always one or the other | Demos shouldn't die when the IDS restarts; production shouldn't pass uninspected traffic |
| Stop on unreachable database | Silent SQLite fallback | A silent fallback produced an empty dashboard with no explanation |
| Single process for gateway and IDS | Redis-backed shared state | Right-sized for one university ERP; documented as a limit rather than half-built |
| Schema created on startup | Migration tool from day one | Fine while the schema is young; Alembic is next before altering tables with real data |

## The rewrite (v1 → v2)

The first version was feature-rich on paper. Testing it end to end showed that several features
were not connected to the request path, and some couldn't work as described. I rebuilt each service
around what actually runs:

| v1 | v2 |
|---|---|
| L1 binary → L2 multiclass → L3 anomaly → meta-model pipeline | One calibrated HTTP model + anomaly + flow models, combined by evidence scoring |
| Reinforcement-learning agents, federated learning | Removed: nothing was trained or used at runtime |
| Drift detection with automatic retraining | Analyst labelling in the dashboard + manual retraining |
| Geo map, threat forecast | Removed: no GeoIP data or forecasting model behind them |
| Dashboard read the database directly with a browser-side key | Dashboard authenticates to the gateway; database is backend-only |
| Features computed separately for training and serving | One shared feature module; models refuse to load on mismatch |
| Passwords reset on every startup | Demo users created only on an empty database, never in production |
| Accuracy figures in the README | Removed: they couldn't be reproduced from the code |
| No automated tests | 94 tests across WAF, features, detection, correlation, gateway and ERP permissions |

The lesson I took from this: a smaller system that does what it claims is worth more than a large
one that doesn't, and saying which features were cut is part of doing the engineering honestly.

## Common questions

**Why middleware instead of checks inside the ERP?**
A gateway gives one enforcement point that can't be skipped by a route someone forgot to protect,
keeps security logic out of business logic, and can protect a different ERP without rewriting it.
The cost is that the gateway becomes critical infrastructure, so it's kept small, ordered by cost,
and has explicit failure behaviour.

**What's the difference between an alert and an incident?**
An alert is one risky request (or merged repeats of it) from one source. An incident is a case
opened by a correlation rule when related alerts form a pattern: a brute-force run, repeated
injection, a recon sweep, a privilege-escalation attempt, or several kill-chain phases in an hour.
Incidents have a timeline, an owner and a status.

**How are false positives handled?**
In four places. Design: evidence-gated blocking and per-user rate limits. Detection: behaviour
alerts are merged, and password fields are masked before feature extraction. Measurement: a traffic
generator reports how many normal requests were wrongly blocked. Feedback: analysts label alerts,
and labelled alerts feed retraining.

**Where does ML help, and where are deterministic controls better?**
Signatures, honeypots, rate limits and role rules are precise and explainable, so they make the
high-confidence decisions. ML covers the gaps: variants no signature matches, and traffic that is
unusual for this specific ERP. ML adds evidence to the score; on its own, it only blocks at very
high confidence.

**How good are the models?**
They're trained on public datasets (CSIC-2010, UNSW-NB15, CIC-IDS) plus synthetic ERP traffic.
I report metrics per data source and leave-one-dataset-out, because a high score on a benchmark
says little about real ERP traffic. I don't quote a single headline accuracy number.

**What happens if the IDS goes down?**
The gateway's circuit breaker trips. In demo and development it fails open so the ERP stays usable;
in production it fails closed with a 503. Blocks the gateway already knows about keep working
either way.

**What would change for production?**
Shared state in Redis to scale out, a managed secret store, CI with dependency and secret scanning,
a model registry, schema migrations, MFA and per-operator roles on the dashboard, centralised
logging, and threshold tuning against real traffic. See the
[hardening roadmap](SECURITY_DESIGN.md#hardening-roadmap).

**Why is the source private?**
It contains the detection logic, signatures, schema and deployment layout. Publishing it would
give an attacker more than a reviewer needs. I'm happy to walk through it live or grant temporary
access.
