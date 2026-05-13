# Delivery & SDLC

## Timeline

- **Project start (planning complete):** May 2026
- **MVP demo-ready:** July 31, 2026
- **Total runway:** ~12 weeks → six 2-week sprints

## Agile Cadence

- **Sprint length:** 2 weeks
- **Ceremonies:**
  - Sprint planning (Mon week 1, 2h)
  - Daily standup (15 min async Slack by default; sync 2× per week)
  - Backlog refinement (Wed week 1, 1h)
  - Sprint review + demo (Fri week 2, 1h)
  - Retrospective (Fri week 2, 45 min)
- **Definition of Ready (DoR):** user story has acceptance criteria in EARS, UX mockup (if UI), API contract (if backend), estimated, linked to a requirement in `requirements.md`.
- **Definition of Done (DoD):** code merged to `main`, CI green, unit + integration tests added, README/docs updated if applicable, deployed to `dev` via ArgoCD, telemetry visible in Grafana, acceptance criteria demonstrably met.

## Full SDLC Flow

```
[Discovery] → [Requirements] → [Design] → [Implementation] → [Verification] → [Release] → [Operate] → [Improve]
```

1. **Discovery** — personas, pain points, scope lock (captured in `product.md`, `requirements.md`).
2. **Requirements** — EARS-format acceptance criteria, correctness properties (in `.kiro/specs/caslearn-olms/requirements.md`).
3. **Design** — C4 diagrams, API contracts, ERD, deployment topology (in `.kiro/specs/caslearn-olms/design.md` + `caslearn-docs`).
4. **Implementation** — feature branches, PRs, reviews, CI gates.
5. **Verification** — unit, integration, property-based, E2E, load (JMeter), security (SAST/Trivy/Gitleaks), accessibility (axe).
6. **Release** — Jenkins builds artifact → Artifactory → ArgoCD syncs → progressive rollout `dev` → `staging` → `prod`.
7. **Operate** — on-call rotation, runbooks in `caslearn-docs`, alerts via Alertmanager, dashboards in Grafana.
8. **Improve** — retrospectives, post-incident reviews, ADRs for significant decisions.

## Sprint Plan (MVP)

| Sprint | Weeks          | Primary Goal                                                                 |
|--------|----------------|------------------------------------------------------------------------------|
| S0     | Pre-sprint 1   | Foundation: repos bootstrapped, Day-0 smoke test passes, observability stack up |
| S1     | Weeks 1–2      | Auth, RBAC, user management, Web UI shell + login                             |
| S2     | Weeks 3–4      | Courses, enrollments, content upload/read                                     |
| S3     | Weeks 5–6      | Assignments, quizzes, auto-grading, grade book                                |
| S4     | Weeks 7–8      | Messaging, notifications, announcements, analytics dashboards                 |
| S5     | Weeks 9–10     | AI Learning Assistant (RAG) + Adaptive Study Planner                          |
| S6     | Weeks 11–12    | Hardening: security review, load tests, accessibility audit, a11y fixes, demo prep |

S0 is the **Developer Day-0** sprint and is non-negotiable. No feature work starts until the end-to-end smoke test (UI → Gateway → backend → response → UI render) is reproducibly green in CI.

## Release Strategy

- **Trunk-based** development on protected `main`.
- **Environments:** `dev` (auto-deploy on merge) → `staging` (auto-deploy nightly from `main`) → `prod` (manual promotion via ArgoCD after approval).
- **Promotion gate:** previous environment's synthetic probes green for 24h + no P1/P2 alerts firing.
- **Rollback:** ArgoCD "Rollback" to previous sync revision; documented in `caslearn-docs/runbooks/rollback.md`.

## Roles & Responsibilities

- **Product / Project Manager:** backlog owner, prioritization, stakeholder comms.
- **Backend Engineer(s):** Java/Spring services, API contracts, database schemas.
- **Frontend Engineer:** React/TS UI, UX, a11y, E2E tests.
- **DevOps / Platform Engineer:** Terraform, Jenkins, ArgoCD, Helm, Istio, observability.
- **QA / Test Engineer:** test strategy, E2E, JMeter, security scans.
- **Tech Lead / Architect:** ADRs, cross-cutting decisions, review authority.

## Risks Tracked Weekly

Kept in `caslearn-docs/risk-register.md`:
- Scope creep → hard scope lock; new items go to post-MVP backlog
- AI cost/latency → mocked LLM in tests; usage budget alerts
- RBAC complexity → property-based tests from day 1
- Istio learning curve → pair on first rollout; runbook for common failures
- Timeline compression → S0 buys us velocity; re-plan at end of S3 if slipping
