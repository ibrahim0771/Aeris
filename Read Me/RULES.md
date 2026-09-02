# RULES.md

## Smart Anomaly Detection Platform for Automated Weather Stations (AWS)
### Engineering Rulebook — Smart India Hackathon (SIH)

**Document status:** v1.0 — Third planning artifact. This is a rulebook, not an implementation document. It governs how every developer and every AI coding assistant writes code for this project.

---

## Table of Contents

1. Purpose
2. Source of Truth
3. Engineering Principles
4. What to Use
5. What to Avoid
6. Library & Dependency Rules
7. Coding Rules
8. Architecture Rules
9. AI/ML Boundaries
10. AI/ML Confidence & Safety Rules
11. Rule-Based + ML Hybrid Policy
12. Weather Data Integrity Rules
13. Physical & Meteorological Boundaries
14. Data Handling Rules
15. Error Handling Rules
16. API Error Format
17. Security Rules
18. Logging Rules
19. Database Rules
20. API Rules
21. Frontend Rules
22. Data Visualization Rules
23. AI-Generated Code Rules
24. Mock/Synthetic Data Rules
25. Testing Rules
26. Performance Boundaries
27. Reliability & Fail-Safe Rules
28. Environment Rules
29. Git & Collaboration Rules
30. Rule Priority
31. Definition of Done
32. Golden Rules
33. Final Engineering Checklist

---

## 1. Purpose

This document exists to prevent the specific failure modes that commonly derail hackathon and early-stage AI/ML projects: inconsistent coding decisions, unnecessary dependencies, overengineering, unsafe or overstated AI/ML claims, poor error handling, data corruption, hard-coded configuration, security mistakes, unmaintainable code, and AI-generated code that quietly drifts away from the agreed architecture.

Every developer — human or AI — working on this codebase is expected to follow these rules. Rules here are meant to be **practical and enforceable**, not aspirational. Where a rule cannot be followed for a good reason, the conflict must be surfaced explicitly (Section 30), not silently bypassed.

## 2. Source of Truth

This project has three governing documents, in this order of derivation:

1. **`PROJECT_REQUIREMENTS.md`** — defines **WHAT** the system must do.
2. **`APP_ARCHITECTURE.md`** — defines **HOW** the system is structured to do it.
3. **`RULES.md`** (this document) — defines **HOW WORK IS DONE** while building it: coding standards, dependency policy, safety boundaries, and process.

`RULES.md` must remain consistent with the two documents above it. If an implementation need conflicts with something stated in `PROJECT_REQUIREMENTS.md` or `APP_ARCHITECTURE.md`, that is an architecture-level conflict to resolve explicitly (Section 23, Section 30) — it is never resolved by quietly working around this rulebook.

## 3. Engineering Principles

- **Correctness and honesty over impressiveness.** A system that clearly says "I'm not sure" is more valuable than one that fakes confidence.
- **Explainability is not optional.** Every anomaly the system surfaces must be traceable to a reason a non-ML-expert can understand.
- **Simplicity is a feature.** The simplest solution that satisfies the requirement is the correct one until proven otherwise.
- **Raw data is sacred.** Nothing in this system ever overwrites or fabricates a raw observation.
- **Fail safely, never silently.** Every failure mode has a defined, visible behavior — never a swallowed exception and never silent data loss.
- **Configuration over hard-coding.** Anything that could plausibly need to change per station, per parameter, or per environment is a configuration value, not a constant buried in code.
- **Consistency across a 6-person team matters more than individual cleverness.** Follow the agreed patterns even when you personally would do it differently.

## 4. What to Use

Recommendations here extend, and must stay consistent with, `APP_ARCHITECTURE.md` Section 17 (Technology Stack).

### Backend

**Preferred:** Python, FastAPI, Pydantic, SQLAlchemy, PostgreSQL, Alembic (where schema evolution warrants it).

Rules:

- **API design:** REST, resource-based, versioned under `/api/v1/`. Endpoint names are nouns (`/stations`, `/anomalies`), not verbs.
- **Request/response validation:** every endpoint's input and output is defined by a Pydantic schema. No endpoint accepts or returns raw, unvalidated dicts.
- **Service layer usage:** all business logic (ingestion orchestration, rule execution, fusion, alert evaluation) lives in `services/`. Routes call services; services never format HTTP responses.
- **Repository/database access:** only `repositories/` modules import and use the SQLAlchemy session directly. Services call repositories, never the ORM session directly.
- **Dependency injection:** use FastAPI's `Depends()` for database sessions, current-user resolution, and shared services — do not instantiate these manually inside route handlers.
- **Configuration management:** all configuration is loaded once via `app/core/config.py` (Pydantic `BaseSettings`) and injected where needed — never read `os.environ` ad hoc throughout the codebase.
- **Type hints:** required on all function signatures (parameters and return types) in backend and ML code. Untyped code is not acceptable for merge.
- **Async usage:** use `async def` for I/O-bound route handlers and database calls where the underlying driver supports it; do not force synchronous, CPU-bound ML inference into `async def` without offloading it appropriately (e.g., a thread pool), since blocking the event loop degrades the whole API.

### Frontend

**Preferred:** React, TypeScript, Vite, Tailwind CSS, a consistent internal component system, a single chosen chart library (Recharts or ECharts — pick one and use it everywhere), a single chosen map library (Leaflet unless a strong reason changes this).

Rules:

- **Component structure:** presentational components in `components/`, route-level composition in `pages/`, domain logic in `features/`, per `APP_ARCHITECTURE.md` Section 8.2. Do not mix data-fetching logic into purely presentational components.
- **Reusability:** before creating a new component, check whether an existing one in `components/` can be configured/extended to cover the need. Duplicated table/card/chart components are a code-review blocker.
- **State management:** server state (stations, anomalies, alerts) goes through React Query; client/UI state uses React state/context. Do not introduce Redux or another global store for MVP scope.
- **API calls:** all API calls go through the `services/` API client modules — components never call `fetch`/`axios` directly.
- **Loading states:** every data-fetching component must render a defined loading state, not a blank screen.
- **Empty states:** every list/table must have a defined empty state with a clear message (e.g., "No anomalies found for this filter"), not a silently empty container.
- **Error states:** every data-fetching component must render a defined error state distinct from the loading and empty states.
- **Responsive UI:** dashboard and core pages must remain usable at common laptop/tablet widths used by control-room and demo setups; do not design only for one fixed resolution.
- **Accessibility:** interactive elements must be keyboard-navigable and use semantic HTML/ARIA labels where charts or custom components are not natively accessible; color must never be the only way severity is communicated (pair color with text/icon).

### AI/ML

**Preferred:** NumPy, Pandas, SciPy, Scikit-learn, Joblib. **Preferred detection methods, chosen by situation, not by default:**

| Situation | Preferred approach |
|---|---|
| Obvious, well-understood physical violations (range, stuck value, missing data) | Rule-based detection — cheap, deterministic, instantly explainable |
| General-purpose statistical outliers on a single parameter | Statistical detection (rolling z-score, control-chart-style methods) |
| Multivariate, less obvious anomalies across engineered features | Isolation Forest (or comparable unsupervised method) |
| Complex, sequence-dependent, or large-scale pattern anomalies where simpler methods provably fall short | Only then consider more complex methods (e.g., autoencoders), and only with clear justification |

**Rule:** deep learning is not introduced merely because the project is described as "AI/ML." Every model choice must be justified by what it detects that a simpler method cannot, per `APP_ARCHITECTURE.md` Section 10's explainability-first stance.

## 5. What to Avoid

| Item | Why avoided | When it might become appropriate |
|---|---|---|
| Kubernetes | Operational overhead disproportionate to a single-team hackathon deployment; risks demo reliability | Only if the system is later split into independently scaled services with a real operational team |
| Kafka / other message brokers | No ingestion volume or consumer-decoupling need at MVP scale; adds a new failure mode | If ingestion throughput or the number of independent consumers grows well beyond hackathon scale |
| Complex microservices | Increases networking complexity, debugging difficulty, and deployment risk during a time-constrained build | Only if a specific module (e.g., ML inference) has a measured, distinct scaling need |
| Multiple databases without a demonstrated need | Operational complexity with no MVP benefit; PostgreSQL covers all current entities | If a specific workload (e.g., full-text search, specialized time-series load) is measured and PostgreSQL genuinely cannot serve it |
| Unnecessary Redis usage | No caching bottleneck identified; adds a service to run/monitor/fail | Only after profiling identifies a specific, expensive, repeated query worth caching |
| Heavy deep-learning models when simpler models work | Harder to explain, harder to train reliably on limited data, harder to debug under hackathon time pressure | Once historical data volume and labeled feedback justify it, and simpler models have demonstrably plateaued |
| Large cloud infrastructure | Not needed for MVP scale; increases cost, setup time, and demo dependency on internet/cloud availability | As part of a genuine production rollout beyond the hackathon |
| Complex MLOps platforms (MLflow servers, Kubeflow, etc.) | Overhead disproportionate to a single-model pipeline | If model iteration frequency and team size grow enough to need formal experiment tracking infrastructure |
| Premature optimization | Wastes hackathon time on performance the MVP doesn't need yet; obscures code | Once correctness and reliability are solid and a real bottleneck is measured |
| Excessive abstraction | Slows a 6-person team down and obscures what the code actually does | Only introduce an abstraction once a second concrete use case actually needs it |
| Hard-coded secrets | Severe security risk, breaks environment portability | Never appropriate — no exception |
| Hard-coded station data | Breaks the requirement that new stations can be added without code changes | Never appropriate — stations are always data, not code |
| Hard-coded anomaly results | Undermines the entire premise of the platform and misleads demo judges | Never appropriate |
| Copy-pasted business logic | Creates drift between duplicated copies and hidden bugs | Never — extract a shared function/service instead |
| API routes containing large business logic | Violates the thin-route rule in `APP_ARCHITECTURE.md` Section 9.1; makes logic untestable in isolation | Never — logic belongs in `services/` |
| Database queries scattered throughout the application | Makes schema changes error-prone and violates the repository boundary | Never — queries belong in `repositories/` |
| ML logic inside frontend code | Frontend has no business inspecting model internals; violates the frontend/backend boundary in `APP_ARCHITECTURE.md` Section 14 | Never |
| Notebook code directly used as production code | Notebooks are exploratory and untested; production code must live in `ml/` proper modules per `APP_ARCHITECTURE.md` Section 15.4 | Never — extract proven logic into a module first |
| Unvalidated user input | Direct path to data corruption and security vulnerabilities | Never |
| Silent exception handling / `except Exception: pass` | Hides real failures, causes silent data loss, makes debugging nearly impossible | Never appropriate in this codebase |
| Fake/generated metrics presented as real-world results | Misleads both the development team and SIH judges about actual system performance | Never |
| Fake ML confidence presented as scientifically validated probability | Overstates what an unsupervised anomaly score actually represents | Never — confidence must be presented as a relative, model-specific signal |
| Claims that the model is 100% accurate | False, and directly contradicts `PROJECT_REQUIREMENTS.md` Requirements Quality Rule #4 | Never |

## 6. Library & Dependency Rules

### Backend

| Library | Purpose | Allowed usage | Alternatives | Restrictions |
|---|---|---|---|---|
| FastAPI | Web framework / API layer | All backend HTTP endpoints | Flask, Django REST Framework | Do not mix in a second web framework |
| Pydantic | Request/response validation | All request/response schemas | Marshmallow | Schemas must live in `schemas/`, not inline in routes |
| SQLAlchemy | ORM / database access | All database reads/writes, via `repositories/` only | Tortoise ORM, raw SQL | No raw SQL outside `repositories/` without a documented reason |
| Alembic | Migrations | Every schema change | Manual SQL scripts | No manual, un-migrated schema changes to the shared database |
| Uvicorn | ASGI server | Running the FastAPI app | Hypercorn, Gunicorn+Uvicorn workers | Fine to add a process manager (e.g., Gunicorn) later for production, not required for MVP |

### Data/ML

| Library | Purpose | Allowed usage | Alternatives | Restrictions |
|---|---|---|---|---|
| NumPy | Numerical computation | Feature engineering, detector internals | — | — |
| Pandas | Tabular data handling | Preprocessing, feature engineering, training scripts | Polars | Polars is an acceptable substitute if a team member has strong reason/familiarity, but do not mix both in the same module |
| SciPy | Statistical functions | Statistical detector, evaluation | — | — |
| Scikit-learn | ML models (Isolation Forest, etc.) | Detector implementations behind `BaseDetector` | — | Models must be wrapped by the `BaseDetector` interface, never called ad hoc from services |
| Joblib | Model serialization | `save_model()`/`load_model()` in detectors | Pickle directly | Prefer Joblib over raw Pickle for scikit-learn artifacts (better handling of NumPy arrays) |

### Frontend

| Library | Purpose | Allowed usage | Alternatives | Restrictions |
|---|---|---|---|---|
| React | UI framework | All frontend UI | Vue, Svelte | — |
| TypeScript | Type safety | All frontend code | Plain JavaScript | No new `.jsx`/`.js` files — TypeScript only |
| Tailwind CSS | Styling | All component styling | CSS Modules, styled-components | Avoid mixing Tailwind with a second styling system |
| Chart library (Recharts or ECharts) | Time-series/anomaly charts | All chart components | Chart.js, Plotly | Pick one at project start and use it consistently — do not mix chart libraries across pages |
| Map library (Leaflet, unless changed) | Station map | Station map page/component | Mapbox GL, Google Maps | Avoid libraries requiring a paid API key unless the team has explicitly provisioned one |
| API client (Axios, or Fetch) | HTTP calls to backend | `services/` modules only | — | Never call directly from components |

### Testing

| Tool | Purpose |
|---|---|
| Pytest | Backend and ML unit/integration tests |
| HTTPX (via FastAPI TestClient) | API integration tests |
| Vitest | Frontend unit/component tests |
| React Testing Library | Frontend component behavior testing |
| Playwright | Only if time allows and end-to-end browser testing has clear value; not required for MVP |

### Code Quality

| Tool | Purpose |
|---|---|
| Ruff | Python linting (fast, covers most Flake8 rules) |
| Black | Python formatting, if the team wants opinionated auto-formatting (optional but recommended for a 6-person team) |
| ESLint | TypeScript/JavaScript linting |
| Prettier | Frontend formatting |
| MyPy | Static type checking, where practical — not required to pass on 100% of the codebase for MVP, but should run on core modules (services, rule engine, ML detectors) |

**Rule:** do not add a tool merely for the sake of having more tools. Each tool above earns its place because it directly supports a rule elsewhere in this document (type safety, formatting consistency, testability).

## 7. Dependency Management Rules

Before adding any new dependency, the developer must answer:

1. Can the requirement be solved with the existing stack?
2. Is the package actively maintained (recent commits/releases, not abandoned)?
3. Does it significantly simplify the implementation, versus a modest amount of custom code?
4. Does it introduce unnecessary security or deployment risk (native binaries, large transitive dependency trees, unclear license)?
5. Is it actually necessary for MVP, or is it a "nice to have" that can wait?

Additional rules:

- **Pinning versions:** all dependencies are pinned to specific versions (`requirements.txt`/`pyproject.toml` lock, `package-lock.json`/`pnpm-lock.yaml`) so builds are reproducible across the team and in Docker.
- **Removing dependencies:** if a dependency's usage is fully removed from the code, remove it from the manifest in the same change — do not leave unused dependencies behind.
- **Updating packages:** version bumps are a deliberate, reviewed change, not an incidental side effect of an unrelated PR.
- **License compatibility:** avoid copyleft-licensed dependencies (e.g., GPL) for anything that would be redistributed; prefer MIT/BSD/Apache-2.0 licensed packages, consistent with a typical hackathon/open project.
- **Avoiding duplicate libraries:** do not add a second library that solves a problem an existing dependency already solves (e.g., a second HTTP client, a second charting library).
- **Avoiding abandoned/unmaintained packages:** check for recent activity before adopting a new package; prefer well-established, actively maintained libraries already listed in Section 6.

## 8. Coding Rules

- **Clear naming:** names describe what something is or does; avoid abbreviations that aren't immediately obvious (`temp` for temperature is acceptable and common in this domain; `tmp` for a temporary variable is fine; single-letter names are not, except conventional loop indices).
- **Type hints:** required on all Python function signatures; required on all TypeScript function signatures and component props.
- **Small functions:** a function should do one identifiable thing; if it needs a "and then" in its description, consider splitting it.
- **Single responsibility:** classes/modules should have one clear reason to change (e.g., a detector implements detection, not database access).
- **DRY where useful:** extract shared logic once a second real use case appears — don't abstract speculatively (see "avoid premature abstraction" below).
- **Avoiding premature abstraction:** do not build a generic plugin system, factory, or configuration DSL for something that currently has one implementation. `BaseDetector` (Section 5.6/10 of `APP_ARCHITECTURE.md`) is justified because MVP already ships two detectors; do not extend that pattern speculatively elsewhere without similar justification.
- **Constants instead of unexplained magic numbers:** thresholds, window sizes, and limits are named constants or configuration values, never bare numbers inline.
- **Configuration instead of hard-coded values:** per Section 4 and `APP_ARCHITECTURE.md` Section 19 — anomaly thresholds, data source selection, and model paths are configuration, not code.
- **Documentation for complex algorithms:** the rule engine, fusion engine, and each ML detector require a short docstring/comment explaining the approach and its known limitations — not a line-by-line narration, but enough for a new team member to understand intent.
- **Consistent formatting:** enforced via Black/Prettier (Section 6); do not hand-format against the configured style.
- **Meaningful comments:** comments explain **why**, not what — the code already says what it does; a comment repeating that adds no value (e.g., avoid `# increment i` above `i += 1`; prefer `# retry once before falling back to rule-only scoring`).
- **No dead code:** remove commented-out code and unused functions before merging — Git history preserves it if needed later.
- **No unused imports.**
- **No debug prints in production code:** use the structured logger (Section 18), never `print()`/`console.log` left in committed code.

## 9. Architecture Rules

Implementation must follow `APP_ARCHITECTURE.md`. The layered boundaries are non-negotiable:

```
Frontend  →  Backend API (routes)  →  Services  →  Repositories  →  Database
                                     ↘  ML Inference Service ↘ ml/ module
Data Ingestion → Preprocessing → Rule Engine ↘
                                Feature Engineering → ML Detectors ↘ Fusion Engine → Explanation Engine → Storage → Alerts → Analytics
```

Rules:

- **API routes remain thin:** parse/validate the request, call a service, format the response. No conditional business logic in a route handler.
- **Business logic belongs in services.** Anything that decides *what should happen* (should this be flagged, what severity, should an alert fire) lives in `services/`, `anomaly/`, or `alerts/` — never in `api/`.
- **Database operations belong in repository/data-access layers.** Only `repositories/` imports and queries via SQLAlchemy session objects.
- **ML logic should not be embedded in API route handlers.** Routes never import scikit-learn or call a detector directly — always through `MLInferenceService`.
- **Frontend should not directly access the database.** There is no scenario in this project where the frontend connects to PostgreSQL directly.
- **Frontend should communicate through backend APIs only,** including for any future third-party integrations.
- **Model artifacts should not be manually edited.** Artifacts are produced only by `training/` scripts and loaded only by `inference/`; do not hand-edit a serialized model file.
- **Notebooks must not become hidden production dependencies.** Nothing in `backend/` or the running application ever imports from `ml/notebooks/`.

## 10. AI/ML Boundaries

This is one of the most important sections in this document.

### AI/ML SHOULD

- Detect potentially anomalous weather observations.
- Produce an anomaly score.
- Identify unusual patterns, including temporal and (where implemented) multi-parameter context.
- Consider station-specific behavior where appropriate (a station's own baseline, not only a global one).
- Work together with deterministic quality rules, never as the sole detection mechanism.
- Provide an interpretable reason whenever possible.
- Flag observations for human review.
- Learn from validated feedback in future iterations (explicitly not automatically, per Section 10 below and `APP_ARCHITECTURE.md` Section 5.13).

### AI/ML SHOULD NOT

- Automatically declare every unusual observation as a sensor failure — an anomaly is a flag for review, not a diagnosis.
- Claim certainty when confidence is low.
- Override physical safety/data-quality rule violations without justification recorded in the fusion output.
- Invent weather observations (e.g., imputing missing data must never masquerade as a real measurement — see Section 12).
- Modify raw observations to make them appear normal.
- Delete suspicious data simply because it is anomalous — anomalous data is retained and flagged, never discarded.
- Present predictions as measured observations.
- Present synthetic/demo data as real AWS observations (see Section 24).
- Automatically retrain itself in production without human-controlled review of the new model.
- Make unsupported meteorological conclusions (e.g., asserting a specific physical cause for an anomaly beyond what the data and rules actually support).
- Treat model output as ground truth (see Section 8's critical principle below, restated for structural clarity in Section 10).

## 11. AI/ML Confidence & Safety Rules

**Confidence/severity behavior policy** (thresholds are configurable per `APP_ARCHITECTURE.md` Section 19 — the tiers below are conceptual, not fixed numeric constants):

| Suspicion level | System behavior |
|---|---|
| Very Low Suspicion | Do not create a critical alert; may be logged for analytics only |
| Moderate Suspicion | Mark as a possible anomaly; show on dashboard; request human review when appropriate |
| High Suspicion | Create an anomaly record; show severity; trigger the configured alert |
| Critical / High-Impact Situation | Escalate per configured alert rules; clearly distinguish "model suspicion" from "human-confirmed event" in the UI and data model |

**Rule:** do not invent scientifically universal threshold values baked into code. Thresholds are configurable parameters, to be calibrated against synthetic/validation data during development — treat any initial numeric threshold as a starting estimate, not a claimed scientific constant.

**Model input/runtime safety rules** — define explicit behavior for:

- **Missing features:** if required features cannot be computed (e.g., insufficient history for a rolling window), the detector must report "insufficient data" rather than guessing or silently defaulting to a score.
- **Unexpected feature ranges:** out-of-expected-range feature values are handled explicitly (e.g., clipped with a logged warning, or flagged as unreliable input) — never passed silently into the model.
- **Model loading failure:** caught at startup/load time; system falls back to rule-only detection (see below); never crashes the whole service.
- **Model version mismatch:** detected via stored model metadata (`APP_ARCHITECTURE.md` Section 5.6); mismatches are logged and block silent use of an incompatible artifact.
- **Inference timeout/failure:** caught per-observation; that observation falls back to rule-only scoring; the failure is logged and counted, not swallowed.
- **Invalid model output / NaN / Infinity:** detected and rejected before being stored or surfaced; treated as an inference failure (fallback to rule-only), not as a valid anomaly score of zero.
- **Out-of-distribution data / data drift / concept drift / model degradation:** not solved automatically in MVP, but the architecture must not hide these possibilities — model metadata and basic score-distribution logging (Section 18) exist so degradation is at least observable, and is flagged as a known limitation rather than assumed away.

> **The application must fail safely and continue deterministic data-quality checks where possible.**

ML failure must never cause the entire weather-data ingestion pipeline to silently lose data. A failed ML inference degrades the system to rule-only detection for that observation, with the degradation itself logged.

## 12. Rule-Based + ML Hybrid Policy

The system must maintain **at least two independent detection mechanisms**, per `APP_ARCHITECTURE.md` Section 5.7:

**Layer 1 — Deterministic Quality Rules:** physical range violation, impossible timestamp, missing reading, sensor stuck at one value, unrealistic rate of change, and other checks defined in `PROJECT_REQUIREMENTS.md` Section 7.C.

**Layer 2 — AI/ML Detection:** statistical outlier detection, contextual/multivariate anomaly detection via the feature-engineered pipeline.

**Anomaly Fusion Layer:** combines evidence from both layers without letting either dominate blindly — a rule violation alone and an ML flag alone are each treated with appropriately bounded severity; agreement between the two raises both severity and confidence (per `APP_ARCHITECTURE.md` Section 5.7). The fusion output must always contain: final score, severity, detection method(s), supporting evidence, and a human-readable explanation. No anomaly record may exist without all five of these fields populated.

## 13. Weather Data Integrity Rules

- **Never overwrite raw incoming data.** The originally ingested observation is immutable once stored.
- **Store normalized/processed data separately** from the raw record (or as clearly-labeled derived fields on the same record — never in place of the raw value).
- **Record processing timestamps** distinct from the observation's own timestamp (ingestion time vs. observation time are different fields).
- **Preserve the original station-reported timestamp** even after UTC normalization — store both.
- **Track units** on every stored value; never assume a unit implicitly.
- **Track station ID and parameter/source metadata** on every observation — no orphaned or ambiguous data points.
- **Detect duplicates** deterministically (station + parameter + timestamp).
- **Handle missing values explicitly** — represent a gap as a gap, not as zero or an omitted row that silently disappears from analysis.
- **Do not silently replace missing observations with fabricated values.**
- **If imputation is used** (e.g., for a chart's visual continuity), **mark imputed values clearly** in both the data model and any UI that displays them.
- **Never confuse imputed data with measured data** — these are always distinguishable fields, never merged into one ambiguous "value."

## 14. Physical & Meteorological Boundaries

The system supports **configurable validation ranges** per parameter (temperature, relative humidity, rainfall, wind speed, wind direction, atmospheric pressure, solar radiation, and any future parameter), because:

- **Do not claim one fixed range is universally correct for all AWS stations.** Sensor specifications, station location, season, time of day, and units all affect what is "normal."

Rules must be able to account for, at minimum:

- Sensor specifications (a given sensor's documented operating range).
- Location (climatic norms differ regionally).
- Season and time of day (e.g., solar radiation expectations shift with daylight hours).
- Units (never compare mismatched units without explicit conversion).
- Station-specific characteristics/history (a station's own baseline, per `APP_ARCHITECTURE.md` Section 5.5).

**Hard limits** (physically impossible values, e.g., humidity outside 0–100%) and **soft expected ranges** (statistically unusual but not physically impossible) must be represented as distinct concepts in the rule engine — a violation of a hard limit and a deviation from a soft expected range should never be reported with the same certainty language.

## 15. Data Handling Rules

- **Missing data:** never treated as zero unless the parameter's semantics genuinely define zero as a valid "no reading" state (and even then, this must be an explicit, documented choice, not a default).
- **Duplicate data:** detected and handled deterministically per Section 13 — the same input always produces the same duplicate-handling outcome.
- **Out-of-order data:** the ingestion pipeline supports late-arriving/out-of-order observations where feasible, re-sorting per station/parameter stream before feature computation.
- **Time zones:** internal storage and processing use UTC; original source timezone metadata is preserved alongside it.
- **Units:** normalized internally to canonical units; original unit metadata is retained.
- **Null values:** handled explicitly at the schema level — a null is a deliberate "no value," not an accidental omission that crashes downstream code.
- **Corrupt records:** rejected or quarantined (logged with reason) without crashing the ingestion process for other records (per `APP_ARCHITECTURE.md` Section 5.2's per-record isolation).

## 16. Error Handling Rules

### General Principles

- Never silently swallow errors.
- Never expose stack traces to end users.
- Log technical details internally (Section 18).
- Return safe, meaningful API errors (Section 17).
- Use consistent HTTP status codes.
- Validate input before processing, not after.
- Fail gracefully — a failure in one part of the system does not cascade into unrelated failures.

### Frontend Errors

Define and handle: API unavailable, request timeout, invalid/malformed response, empty data (distinct from an error), authentication failure (redirect to login, do not silently fail).

### Backend Errors

Define and handle: validation failure (4xx with field detail), database failure (5xx, logged, safe message to client), internal service failure (5xx, logged), missing configuration (fail fast at startup, not at first use).

### Ingestion Errors

Define and handle: invalid payload, unknown station, unknown parameter, malformed timestamp, duplicate record, connection failure to the data source — each logged and counted, none halting the rest of the batch/stream.

### ML Errors

Define and handle: model unavailable, model loading failure, invalid feature vector, inference failure, invalid model output — all fall back to rule-only detection per Section 11, never a hard crash.

## 17. API Error Format

All API errors use a consistent structure:

```json
{
  "error": {
    "code": "INVALID_WEATHER_DATA",
    "message": "The submitted observation is invalid.",
    "details": {},
    "request_id": "..."
  }
}
```

`code` is a stable, machine-readable identifier the frontend can branch on; `message` is safe to show a user; `details` carries structured, non-sensitive validation context (e.g., which field failed); `request_id` supports correlating a user-facing error with backend logs.

**Never expose in an API error response:** stack traces, file paths, secrets, internal database details (table/column names, raw SQL errors), or model internals that could create a security or gaming risk (e.g., exact threshold values that would let someone craft data to evade detection).

## 18. Security Rules

- **Secrets and environment variables:** all secrets (JWT signing key, database credentials, any future API keys) live in environment variables, loaded via the centralized config (`APP_ARCHITECTURE.md` Section 19) — never hard-coded, never committed.
- **Authentication:** JWT-based, per `APP_ARCHITECTURE.md` Section 14. Tokens have a defined, reasonable expiry.
- **Authorization:** enforced server-side on every protected endpoint via a role-checking dependency — never trusted from frontend state alone.
- **Password handling:** passwords are hashed (e.g., bcrypt) before storage; plaintext passwords are never logged or stored.
- **JWT handling:** signing secret is environment-specific and never reused across environments; tokens are validated on every request, not cached as "already checked."
- **CORS:** configured explicitly per environment (`CORS_ORIGINS` in config) — never wildcard `*` in a deployed environment.
- **Input validation:** every external input (API request body, uploaded file, query parameter) is validated via Pydantic/schema before use.
- **SQL injection prevention:** achieved by using the SQLAlchemy ORM/parameterized queries exclusively — no string-concatenated SQL.
- **XSS considerations:** the frontend must not render unescaped user-supplied content as raw HTML; React's default escaping is relied upon, and `dangerouslySetInnerHTML` is avoided unless a specific, reviewed need exists.
- **File-upload validation:** uploaded CSV/JSON files (historical data ingestion) are validated for type, size limits, and schema before processing — never executed or parsed in a way that could allow arbitrary code execution.
- **Rate limiting:** not required for MVP demo scale, but the authentication endpoint in particular should have basic protection against brute-force if time allows; noted as a near-term hardening item.
- **Logging sensitive information:** never log passwords, tokens, or secrets (see Section 18 below).

**Never commit:** API keys, passwords, JWT secrets, database credentials, cloud credentials. `.env` is gitignored; `.env.example` documents required variables with placeholders only.

## 19. Logging Rules

**Log:**

- Application errors.
- Ingestion failures (per-record and per-batch summaries).
- Important anomaly events (created, severity change, status change).
- Model failures/fallbacks.
- Authentication events (login success/failure — without credentials).
- Important system events (startup, model load, configuration issues).

**Do not log:**

- Passwords.
- Tokens (JWTs, API keys).
- Secrets.
- Sensitive credentials of any kind.

**Format:** structured (JSON) logs where practical, per `APP_ARCHITECTURE.md` Section 21, written to stdout for the Docker-based MVP environment.

## 20. Database Rules

- **Use migrations** (Alembic) for every schema change — no manual, unmigrated changes to a shared database.
- **Avoid destructive schema changes** without a migration plan that considers existing data (e.g., a column removal is staged, not immediate, if data would be lost).
- **Add indexes based on actual query patterns** observed during development — do not index speculatively before knowing the access patterns.
- **Use foreign keys appropriately** to enforce referential integrity between, e.g., Observations → Stations, Anomalies → Observations.
- **Validate data at application boundaries** (Pydantic schemas), not only relying on database constraints as the first line of defense.
- **Preserve anomaly history** — anomaly records are never deleted, only status-transitioned (per `PROJECT_REQUIREMENTS.md` Section 7.K).
- **Preserve raw weather observations** — never deleted or overwritten (Section 13).
- **Use UTC timestamps consistently** in storage (Section 15).
- **Avoid storing large ML artifacts inside normal relational tables** — model binaries live on disk/artifact storage (`ml/artifacts/`), with only metadata (path, version, training date) in the database.

## 21. API Rules

- **RESTful naming:** plural nouns for collections (`/api/v1/stations`), nested resources where logical (`/api/v1/stations/{id}/anomalies`).
- **Versioning strategy:** all routes under `/api/v1/`; breaking changes require a new version prefix, not silent in-place changes.
- **Validation:** every request validated via Pydantic before reaching service logic.
- **Pagination:** all list endpoints (observations, anomalies, alerts) support pagination — never return an unbounded result set.
- **Filtering:** list endpoints support filtering by the fields a user would realistically filter by (station, severity, date range, status).
- **Sorting:** list endpoints support at least one sensible default sort (e.g., anomalies by timestamp descending) and, where useful, a sort parameter.
- **Consistent status codes:** 200/201 for success, 400 for validation errors, 401/403 for auth failures, 404 for missing resources, 409 for conflicts (e.g., duplicate record), 500 for unexpected server errors.
- **Consistent error responses:** per the format in Section 17, on every error path.
- **Authentication requirements:** all write endpoints require authentication; read endpoints require at minimum Viewer-level authentication unless a specific public endpoint is deliberately designed otherwise.
- **Rate limiting considerations:** noted for future hardening (Section 18); not a hard MVP blocker.
- **API documentation:** FastAPI's automatic OpenAPI/Swagger docs are kept accurate — schemas and route descriptions are not left as generic defaults.

**Avoid:**

- Extremely large API responses — paginate or summarize instead of dumping full historical datasets in one call.
- Inconsistent endpoint naming (mixing verbs and nouns, inconsistent pluralization).
- Business logic inside routes (Section 9).
- Returning database internals directly — responses go through Pydantic response schemas, never raw ORM objects serialized as-is.

## 22. Frontend Rules

- **No direct database access** (Section 9).
- **API abstraction:** all backend communication goes through `services/` API client modules.
- **Loading indicators:** every async data view shows one.
- **Error states:** every async data view has a defined error state.
- **Empty states:** every list/table has a defined empty state.
- **Responsive design:** core pages remain usable across common demo/judging display sizes.
- **Accessible controls:** keyboard navigation and semantic markup for interactive elements; severity communicated via more than color alone.
- **Consistent terminology:** the same words are used for the same concepts across the whole UI (e.g., always "Station," never switching to "Site" elsewhere) and must match the vocabulary used in `PROJECT_REQUIREMENTS.md`.
- **Avoid excessive animations** that could distract from or delay a live demo.
- **Avoid misleading visualizations** (Section 22 below expands on this for charts specifically).

**For anomaly visualizations specifically:**

- Clearly distinguish observed values from predicted/expected values (e.g., different line styles/colors, labeled legend).
- Clearly mark anomalies (a visually distinct marker, not something a viewer could miss or confuse with normal data).
- Show severity (color and/or icon, consistent with the severity scale used everywhere else in the UI).
- **Never imply that an anomaly marker means the event is confirmed by a human.** The UI must visually and textually distinguish "flagged by the system" from "confirmed by an analyst" at all times (Section 10's critical principle, enforced in the UI).

## 23. Data Visualization Rules

Because this is a weather-monitoring application whose entire value proposition is trustworthy data, charts must:

- Show units on every axis/value.
- Show timestamps clearly, with an unambiguous timezone (UTC, labeled, or clearly converted to local time and labeled as such).
- Handle missing values honestly — a gap in the data is shown as a gap, not smoothly interpolated in a way that hides that data was missing (unless explicitly showing a marked imputed-value line, per Section 13).
- Clearly mark anomalies (Section 22).
- Avoid misleading axis scales — do not truncate a y-axis in a way that exaggerates the visual size of a deviation without a clear, honest reason.
- Distinguish raw and processed data where both are shown on the same chart.
- Show expected/baseline values separately where applicable (e.g., a shaded "expected range" band alongside the observed line), rather than only showing the flagged point in isolation.

**Rule:** never manipulate chart scales merely to make anomalies look more dramatic for demo purposes. The data must speak for itself, honestly presented — this is both an ethical requirement and, practically, a credibility requirement in front of SIH judges.

## 24. AI-Generated Code Rules

Because AI coding assistants (including Claude) may be used heavily in development, AI-generated code must:

- Follow `PROJECT_REQUIREMENTS.md`.
- Follow `APP_ARCHITECTURE.md`.
- Follow this `RULES.md`.
- Reuse existing utilities before creating duplicates — check `utils/`, `services/`, and `components/` before writing new helper code.
- Not introduce dependencies without justification (Section 7).
- Not invent APIs — if an endpoint doesn't exist yet, propose it explicitly rather than assuming it and writing code against it.
- Not invent database fields — if a field doesn't exist in the agreed schema, propose the addition explicitly.
- Not invent external services — no assumed third-party integrations beyond what `APP_ARCHITECTURE.md` defines.
- Not fabricate ML evaluation results — any reported metric must come from an actual run against actual (even if synthetic) data, never a plausible-sounding invented number.
- Not silently change architecture — if a task seems to require deviating from `APP_ARCHITECTURE.md`, the AI assistant must say so explicitly and explain the proposed change before implementing it, rather than quietly restructuring the codebase.
- Include error handling consistent with Sections 16–17.
- Include tests for critical logic (rule engine, fusion engine, detectors, ingestion validation), per Section 25.

**AI should explain architectural changes before making them.** This is a hard requirement, not a courtesy: silent architectural drift is one of the primary risks this rulebook exists to prevent.

## 25. Mock/Synthetic Data Rules

Synthetic data may be used for: development, testing, demo, load testing, and demonstrating anomaly detection capability.

Synthetic data must:

- Be clearly identifiable as synthetic (a `source: "synthetic"` field or equivalent marker on every generated record, surfaced in the UI where relevant — e.g., a "Demo Data" indicator).
- **Not be presented as actual government/AWS measurements**, in the UI, in documentation, or in the SIH demo narrative.
- Support realistic weather patterns (diurnal/seasonal shapes where applicable, per `PROJECT_REQUIREMENTS.md` Section 9.3).
- Include controlled, injectable anomalies for testing and demonstration.
- Include both normal and abnormal scenarios — not only anomalies, since the system's behavior on clean data matters too (false-positive rate).
- Preserve realistic timestamps and units, exactly as a real feed would provide them.

**The system must make it easy to replace synthetic data with real AWS data** — enforced architecturally via the `DataSourceAdapter` interface (`APP_ARCHITECTURE.md` Section 5.1); no downstream code may assume the synthetic source specifically.

## 26. Testing Rules

Minimum testing standards:

### Unit Tests
- Rule engine (every individual rule, with known-good and known-bad inputs).
- Feature engineering (correctness of rolling statistics, z-scores, etc.).
- ML detectors (each detector's `predict`/`score_anomaly` behavior on known synthetic cases).
- Anomaly fusion (combination logic produces expected severity/category given known rule + ML inputs).
- Data validation (schema/type rejection behaves as expected).

### Integration Tests
- Ingestion → processing (a raw payload produces the expected cleaned observation).
- Processing → anomaly detection (a cleaned observation with known characteristics produces the expected anomaly outcome).
- Backend → database (repository operations round-trip correctly).
- API → frontend contract (API responses match the shape the frontend expects — via API integration tests using FastAPI's TestClient).

### UI Tests
- Dashboard loading (renders expected summary widgets given mock API data).
- Station selection (navigating to a station detail view works and shows expected data).
- Anomaly review (an analyst can open, classify, and see the status update reflected).
- Error states (each key page renders its defined error state given a failed API call).

**Critical anomaly-detection logic must have automated tests** — this is the single highest-priority testing requirement in the project, per `APP_ARCHITECTURE.md` Section 22.

## 27. Performance Boundaries

- Do not optimize prematurely. MVP focus order: **correctness → reliability → clear architecture → reasonable response time → efficient queries → batch processing where appropriate.**
- "Reasonable response time" means API responses fast enough for a smooth live demo and normal operator use — not a formally benchmarked SLA at MVP stage.
- Efficient database queries means avoiding obviously wasteful patterns (N+1 queries, unindexed lookups on large tables) once query patterns are known — not speculative micro-optimization.
- Batch processing (e.g., for historical CSV ingestion) is used where it is the natural fit, not introduced as a general-purpose performance trick elsewhere.
- **Do not introduce distributed systems merely to optimize hypothetical future workloads.** Every performance-motivated architectural decision must be backed by an actual, current need, not a guess about future scale (consistent with Section 5 and `APP_ARCHITECTURE.md` Section 28).

## 28. Reliability & Fail-Safe Rules

Define behavior explicitly for each failure mode:

### Database unavailable
- Return a meaningful, safe API error (Section 17).
- Do not silently discard incoming data — if persistence fails, the failure is surfaced and logged, not swallowed.

### ML model unavailable
- Preserve the observation (it is always stored regardless of ML outcome).
- Run rule-based checks if possible (Section 11's fail-safe policy).
- Mark the ML result as unavailable/unscored, distinctly from a legitimate low anomaly score.

### External AWS source unavailable (future, once a real feed exists)
- Show source connection status clearly in the system (e.g., station status reflecting "no recent data").
- Preserve historical data regardless of current connectivity.
- Allow falling back to mock/demo mode for continued development/demo purposes.

### Frontend unavailable
- The backend must remain independently testable and operable (via API docs/TestClient) without requiring the frontend to be running — this is also a practical development requirement for a 6-person team working in parallel.

## 29. Environment Rules

Define separate environments: **Development, Testing, Demo, Production/future.**

- Configuration is environment-specific, loaded via environment variables per `APP_ARCHITECTURE.md` Section 19 (`ENVIRONMENT` setting).
- **Never use production credentials locally** — even before a "production" environment formally exists, this rule establishes the discipline for when it does.
- The Demo environment (Docker Compose, per `APP_ARCHITECTURE.md` Section 23.2) should be treated as its own distinct, tested configuration — not just "whatever development happens to look like at the time" — since it is what SIH judges will actually see.

## 30. Git & Collaboration Rules

For a six-person SIH team:

- **Feature branches** for all work — no direct commits to `main` where avoidable.
- **Pull requests** required for merging into `main`/`develop`.
- **Code review** — at least one other team member reviews before merge, focused especially on architecture-boundary and AI/ML-safety rule adherence.
- **Meaningful commit messages** describing what changed and why, not "fix" or "wip" as a final commit message.
- **No direct pushing to the protected main branch** where the team's Git hosting allows branch protection.
- **No committing generated datasets unnecessarily** — large synthetic data dumps stay out of the repo (Section 15.5 of `APP_ARCHITECTURE.md`); only small samples and generator configuration are committed.
- **No committing secrets** (Section 18).
- **Resolve conflicts carefully** — a merge conflict in shared files (e.g., API schemas, database models) is resolved by understanding both changes, not by blindly picking one side.
- **Keep commits focused** — one logical change per commit where practical, to keep history reviewable.

## 31. Rule Priority

When rules conflict, resolve in this order:

**1. Data integrity and security**
**2. Project requirements** (`PROJECT_REQUIREMENTS.md`)
**3. Architecture** (`APP_ARCHITECTURE.md`)
**4. AI/ML safety boundaries** (Sections 10–12 of this document)
**5. Reliability**
**6. Maintainability**
**7. Performance**
**8. Convenience**

**If a shortcut violates a higher-priority rule, the shortcut must not be used** — regardless of time pressure, demo deadlines, or how small the violation seems. A convenient shortcut that compromises data integrity or fabricates a result is never acceptable, even under hackathon time pressure.

## 32. Definition of Done

A feature is not complete until all of the following are true:

- The requirement it addresses (traceable to `PROJECT_REQUIREMENTS.md`) is actually implemented, not partially stubbed.
- The architecture in `APP_ARCHITECTURE.md` is followed (correct layer boundaries, correct module placement).
- Input validation is added at every relevant boundary.
- Error handling is added per Sections 16–17.
- Tests are added for critical behavior (Section 26), especially anomaly-detection logic.
- No secrets are committed.
- No unnecessary dependencies were introduced (Section 7 checklist was actually applied).
- Documentation is updated where needed (README, docstrings, or the relevant `docs/` file).
- UI handles loading/error/empty states (Section 22), for any frontend-facing feature.
- ML behavior is explainable where applicable — no anomaly-producing change ships without a corresponding explanation output.

## 33. Golden Rules

A concise, memorable checklist for every developer and AI assistant on this project:

1. Preserve raw data — never overwrite it.
2. Never fabricate weather data.
3. Never treat ML output as absolute truth.
4. Use rules + ML together, not one alone.
5. Keep AI boundaries explicit (Section 10).
6. Validate every external input.
7. Fail safely — never crash the whole pipeline over one bad record or one failed model.
8. Never swallow exceptions.
9. Keep API routes thin.
10. Keep ML isolated from the UI.
11. Avoid unnecessary dependencies.
12. Do not overengineer the MVP.
13. Never commit secrets.
14. Test anomaly logic — it's the core of the project.
15. Label synthetic data clearly, everywhere it appears.
16. Make thresholds configurable, not hard-coded.
17. Make explanations human-readable, always.
18. Do not overwrite original observations, ever.
19. Prefer simple solutions that are reliable over clever ones that aren't.
20. Follow `PROJECT_REQUIREMENTS.md` and `APP_ARCHITECTURE.md` — this rulebook exists to support them, not replace them.
21. If you're about to violate a rule to save time, stop and flag it instead — don't do it silently.

---

## Final Engineering Checklist

Before considering any milestone (feature, module, or the MVP itself) ready:

- [ ] Does it satisfy a requirement traceable to `PROJECT_REQUIREMENTS.md`?
- [ ] Does it respect the layer boundaries in `APP_ARCHITECTURE.md`?
- [ ] Are all inputs validated and all errors handled per Sections 16–17?
- [ ] Are AI/ML boundaries respected (Sections 10–12) — no overstated confidence, no silent overrides, no fabricated results?
- [ ] Is raw data preserved and clearly separated from processed/imputed data?
- [ ] Are thresholds and configuration externalized, not hard-coded?
- [ ] Are tests present for the critical/anomaly-detection paths?
- [ ] Is synthetic data clearly labeled as such wherever it appears?
- [ ] Are secrets absent from the codebase and Git history?
- [ ] Does the UI (if applicable) handle loading, empty, and error states, and avoid misleading visualizations?
- [ ] If anything here required deviating from `APP_ARCHITECTURE.md` or this rulebook, was that deviation explicitly identified and discussed rather than silently made?

If any answer is "no," the work is not done.
