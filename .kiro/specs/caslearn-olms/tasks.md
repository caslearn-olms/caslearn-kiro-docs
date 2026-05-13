# Implementation Plan: CasLearn OLMS Platform

## Overview

This plan translates the locked design (`design.md`, 23 sections) and requirements (`requirements.md`, 30 requirements + 12 correctness properties CP-1 … CP-12) into discrete, developer-sized coding tasks grouped by the six fixed sprints in `delivery.md`. Each task points at specific repos (`caslearn-*`), cites the requirement clauses and design sections it implements, and — where applicable — names the correctness property it verifies.

**Locked tech** (per `tech.md`): Java 21 + Spring Boot 3.x + Maven on the backend; TypeScript 5 + React 18 + Vite + Redux Toolkit + MUI + Tailwind on the frontend; MySQL 8 / Redis 7 / MinIO / Qdrant / Istio / Helm / ArgoCD / Jenkins / JFrog Artifactory / Prometheus / Loki / Tempo / Grafana. Task prompts below assume those choices — do not re-litigate.

Task numbers are sequential across all sprints (1 … N), not restarted per sprint. Sub-tasks use decimal notation (e.g., `2.1`, `2.2`). Sub-tasks marked with `*` are optional test tasks that can be skipped for a faster MVP.

---

## 12-Week Calendar

Project kickoff is the Monday after the design sign-off in `delivery.md` — target Monday **May 11, 2026**. MVP demo-ready is **Friday July 31, 2026**. That is exactly 12 calendar weeks: 6 two-week sprints.

| Week | Dates (2026) | Sprint | Focus | Primary tasks |
|------|--------------|--------|-------|---------------|
| **W1** | Mon May 11 – Fri May 15 | S0 week 1 | GitHub org, repo templates, all 15 repos bootstrapped, Jenkins + SonarQube + JFrog installed, Terraform modules started, observability stack Helm charts authored | 1, 2, 3, 4, 5, 6, 7, 8, 11, 13, 14, 23 (start) |
| **W2** | Mon May 18 – Fri May 22 | S0 week 2 | Atlantis live, ArgoCD live, service skeletons compiling, shared `caslearn-audit` library, Day-0 smoke test green in CI, onboarding guide rehearsed, all 12 open questions resolved | 9, 10, 12, 15, 16, 17, 18, 19, 20, 21, 22, 23 (finish), 24 |
| **W3** | Mon May 25 – Fri May 29 | S1 week 1 | Auth Service data layer, login + refresh endpoints, JWT signer, RBAC annotations in `caslearn-security-lib`, Gateway JWT filter, Web UI login page | 25, 26, 27, 28 (start), 31, 32 (start), 34 (start) |
| **W4** | Mon Jun 1 – Fri Jun 5 | S1 week 2 | User management, audit export, role-field rejection filter, information-leak-preserving 404, Istio AuthZ policies for auth, persona-scoped Web UI shell, CP-3 / CP-5 / CP-11 PBT in CI | 28 (finish), 29, 30, 32 (finish), 33, 34 (finish), 35, 36 |
| **W5** | Mon Jun 8 – Fri Jun 12 | S2 week 1 | Course Service schema + Flyway, courses / modules / content_items / enrollments CRUD, object-storage client, MIME allow-list, ownership check | 37, 38, 39, 40, 41 (start), 42 (start) |
| **W6** | Mon Jun 15 – Fri Jun 19 | S2 week 2 | Enrollment / drop with CP-1 enforcement, soft delete + 30-day retention, Web UI course authoring + browse, CP-1 / CP-8 PBT in CI, OpenAPI specs published to `caslearn-api-contracts` | 41 (finish), 42 (finish), 43, 44, 45, 46 |
| **W7** | Mon Jun 22 – Fri Jun 26 | S3 week 1 | Assessment Service schema, quiz + assignment authoring, rubric model, POINTS_MISMATCH validation, Web UI quiz authoring | 47, 48, 49, 50 (start), 51 (start) |
| **W8** | Mon Jun 29 – Fri Jul 3 | S3 week 2 | Deterministic grader (CP-4), submission handler (on-time / late / auto-submit), manual grading + grade book, grade release + amend with justification, CP-2 / CP-4 / CP-7 PBT in CI, Web UI grading + student grade view | 50 (finish), 51 (finish), 52, 53, 54, 55, 56 |
| **W9** | Mon Jul 6 – Fri Jul 10 | S4 week 1 | Messaging Service schema, announcements + DMs, notification outbox + idempotency, SMTP client, Web UI inbox + composer | 57, 58, 59, 60, 61 (start), 62 (start) |
| **W10** | Mon Jul 13 – Fri Jul 17 | S4 week 2 | Analytics Service read-model + ETL, instructor course analytics, Admin Overview dashboard, fail-closed on RBAC unavailability, CP-9 PBT in CI | 61 (finish), 62 (finish), 63, 64, 65, 66 |
| **W11** | Mon Jul 20 – Fri Jul 24 | S5 (compressed) | AI Service schema, Qdrant client, ingestion worker, retrieval filter, RAG pipeline with citations + refusal threshold, Web UI AI chat + citations, study planner heuristic + Web UI, CP-6 PBT with mocked LLM, AI rate limit (Req 17.6), AI telemetry + budget alert | 67, 68, 69, 70, 71, 72, 73, 74, 75, 76 |
| **W12** | Mon Jul 27 – Fri Jul 31 | S6 (compressed) | JMeter nightly green, HPA / Hikari / Redis tuning, OWASP Top 10 review, pen-test smoke, FERPA export + delete flows, backup-restore drill, chaos day, audit-chain verifier scheduled, alerts wired to on-call, axe WCAG sweep, OpenAPI conformance + Helm portability final runs, all 15 READMEs complete, diagrams rendered, `v1.0.0-mvp` tagged + promoted to prod | 77–101 |

### How to read this

- **S5 and S6 run one week each** in this compressed calendar, not two. The CasLearn team is small (per `delivery.md`), so S0 buys velocity and S1–S4 stay on the standard 2-week cadence. S5 (AI) and S6 (hardening) are intensive but scoped so that one week is enough — the design deliberately makes the AI layer mostly additive work against services that are already stable.
- **If burn-down at the end of W8 (S3 exit) is behind**, the fallback in `delivery.md` kicks in: the AI Service ships a minimal non-RAG assistant (fixed-disclaimer responses) in W11, and full RAG moves to post-MVP. CP-6 is still enforced on the retrieval filter so the authorization contract holds either way.
- **Daily cadence**: sprint planning Monday of week 1 (2 h), standup Mon/Wed/Fri (15 min, async Slack on the other days), backlog refinement Wed of week 1 (1 h), sprint review + retro Friday of week 2 (1.75 h).
- **Friday demos**: at end of W2, W4, W6, W8, W10, W11 (S5 early-cut), and W12 (final MVP).

### Week-by-week exit criteria

Each week must hit its exit criterion before the next week's work starts. Slipping a weekly gate triggers the re-plan conversation at the next standup.

| Week | Exit criterion |
|------|----------------|
| W1 | All 15 repos exist with branch protection; Jenkins can build the `caslearn-docs` hello-world pipeline end-to-end. |
| W2 | `day0-up.sh` produces a browser-accessible Web UI that calls Gateway → a backend service → MySQL and renders the response. Day-0 smoke test green in CI. Every OQ has a recorded answer. |
| W3 | A seeded admin can log in via the Web UI and land on a role-scoped page. CP-11 PBT green in CI. |
| W4 | Admin UI can create/deactivate users and amend roles; CP-3 and CP-5 PBT green; `role` field rejection filter blocks every escalation path the pen-test-lite suite tries. |
| W5 | Instructor API creates courses, modules, and uploads content with MIME/size guardrails; student API returns 403 on unauthorized reads. |
| W6 | Student can self-enroll in a PUBLISHED course and read its content; CP-1 and CP-8 PBT green; UI course authoring and browse screens usable end-to-end. |
| W7 | Instructor authors a quiz and an assignment; POINTS_MISMATCH rejects mismatched point totals; UI supports both author flows. |
| W8 | Student submits a quiz + assignment; deterministic grader produces a grade; grade book shows the grade to the right student only; CP-2 / CP-4 / CP-7 PBT green. |
| W9 | Instructor posts an announcement and a DM; student receives in-app + email notifications; idempotency keys are deduped through a retry test. |
| W10 | Instructor opens course analytics; Admin Overview dashboard shows live panels for request rate / error rate / p95 / CPU / memory / active users; CP-9 PBT green; Analytics returns 503 with `analytics_authz_unavailable_total` when Auth is killed. |
| W11 | Student asks an in-scope question and gets a cited answer; asks an out-of-scope question and gets the refusal; asks about a course they can't read and retrieval returns no cross-course chunks; CP-6 PBT green with mocked LLM; AI rate limit (Req 17.6) returns 429 after the 30-in-60 threshold. |
| W12 | Release gate matrix in `caslearn-docs/release-gates.md` is all green (repo-naming, coverage ≥ 70%, CP-1…CP-12, JMeter p95, axe WCAG AA, mobile Playwright, SonarQube, Trivy, Gitleaks, license scan, Day-0 smoke); `v1.0.0-mvp` tagged; `prod` synced via ArgoCD; 48 h soak begins. |

---

## Tasks

## Sprint 0 — Foundation

Goal per `delivery.md`: repos bootstrapped, observability stack up, Day-0 smoke test green in CI before any feature sprint starts.

- [ ] 1. Stand up the GitHub organization and default branch-protection policy
  - [ ] 1.1 Create the `caslearn-learn` GitHub organization and invite the platform, backend, frontend, DevOps, and QA teams
    - Create teams `@caslearn/platform`, `@caslearn/backend`, `@caslearn/frontend`, `@caslearn/devops`, `@caslearn/qa`, `@caslearn/release-override`
    - Store the org-level GitHub App credentials in the operator's password vault (not in Git)
    - _Requirements: Req 19.3, Req 20.7_
    - _Design: Sec 17.2_
  - [ ] 1.2 Encode default branch-protection policy for every repo in `caslearn-docs/scripts/branch-protection.json`
    - Require 1 approving review on all repos; require 2 approvers on `caslearn-env-tf` and `caslearn-platform-argo` (per OQ-9 default)
    - Require passing `ci/jenkins` status check; require signed commits
    - Emit `caslearn-docs/runbooks/branch-protection.md` documenting the policy
    - _Requirements: Req 19.3, Req 22.1_
    - _Design: Sec 17.2, Sec 23 (OQ-9)_
  - [ ] 1.3 Seed the default `CODEOWNERS` file for each repo suffix
    - `*-service` → `@caslearn/backend`, `*-ui` → `@caslearn/frontend`, `*-tf` / `*-helm` / `*-argo` / `*-build` → `@caslearn/devops`, `*-contracts` / `*-docs` → `@caslearn/platform`
    - _Requirements: Req 19.5, Req 28.1_

- [ ] 2. Bootstrap `caslearn-docs` as the platform control repository
  - [ ] 2.1 Create `caslearn-docs` with the 14-section README, `repos.yaml`, ADR folder, diagrams folder, runbooks folder, and scripts folder
    - Include the full `repos.yaml` manifest of the 15 MVP repos as listed in design Sec 17.1
    - _Requirements: Req 19.2, Req 28.1, Req 28.3, Req 28.4_
    - _Design: Sec 17.1_
  - [ ] 2.2 Implement `caslearn-docs/scripts/bootstrap-repos.sh`
    - `gh repo create` from the correct template per entry, seed README, CODEOWNERS, branch protection (via `gh api`), Dependabot + secret scanning, Jenkins webhook
    - Idempotent: skip repos that already exist, exit non-zero on partial failure
    - _Requirements: Req 19.3_
    - _Design: Sec 17.2_
  - [ ] 2.3 Implement `caslearn-docs/scripts/day0-up.sh` and `caslearn-docs/scripts/smoke.sh`
    - Follow the step list in design Sec 16.1 (prereq check → kind → istio → cert-manager → argocd → kps → loki → tempo → minio → mysql-operator → qdrant → app-of-apps → wait pods → smoke)
    - `smoke.sh` must execute the 5 checks in design Sec 16.3 (healthz, login, JWT-authenticated course list, Web UI shell, logged JSON output per step)
    - Exit non-zero with the failing step name on any failure; write logs to `/tmp/caslearn-day0/`
    - _Requirements: Req 20.1, Req 20.3, Req 20.4, Req 20.5_
    - _Design: Sec 16.1, Sec 16.3_
  - [ ] 2.4 Implement `caslearn-docs/scripts/seed-demo.sh`
    - Seed 3 courses, 2 instructors, 15 students, 1 admin, 1 published quiz, 1 draft assignment into the local cluster (resolves OQ-12 default)
    - _Requirements: Req 20.3_
    - _Design: Sec 23 (OQ-12)_

- [ ] 3. Build the eight GitHub repository templates
  - Each template ships the 14-section README scaffold, CODEOWNERS, .gitignore, Apache 2.0 LICENSE, `.editorconfig`, `.github/pull_request_template.md`, `.github/workflows/lint.yml` (README-section check + secret scan), language-appropriate build config, and Jenkinsfile skeleton
  - [ ] 3.1 `caslearn-service-template` (backend Java)
    - `pom.xml` with Java 21, Spring Boot 3.x, Spring Security, Spring Data JPA, Actuator, Micrometer Tracing OTLP, Testcontainers, jqwik, Flyway
    - `Dockerfile` with a pinned-by-digest Temurin 21 base image
    - `src/main/java/com/caslearn/<name>/` layout from `structure.md`
    - _Requirements: Req 19.5, Req 28.1, Req 29.1_
  - [ ] 3.2 `caslearn-ui-template` (frontend)
    - `package.json` with Vite + React 18 + TS strict + Redux Toolkit + RTK Query + React Router v6 + React Hook Form + Zod + MUI + Tailwind + Vitest + Playwright + fast-check + axe-core
    - `Dockerfile` that builds a production Nginx image
    - _Requirements: Req 19.5, Req 28.1, Req 29.2_
  - [ ] 3.3 `caslearn-tf-template` with `atlantis.yaml`, `README.md`, `.pre-commit-config.yaml` for `terraform fmt`
    - _Requirements: Req 19.5, Req 21.1_
  - [ ] 3.4 `caslearn-build-template` (Jenkins shared library scaffold: `vars/`, `src/com/caslearn/`)
    - _Requirements: Req 19.5, Req 22.1_
  - [ ] 3.5 `caslearn-helm-template` with `Chart.yaml`, `values.yaml`, `values.onprem.yaml`, `values.aws.yaml`, `templates/_helpers.tpl`, `templates/pre-install-hook.yaml`
    - _Requirements: Req 19.5, Req 23.1_
  - [ ] 3.6 `caslearn-argo-template` with `app-of-apps.yaml` stub and `apps/{dev,staging,prod}/` folders
    - _Requirements: Req 19.5, Req 22.3_
  - [ ] 3.7 `caslearn-contracts-template` with an OpenAPI 3.1 skeleton per service and Redocly config
    - _Requirements: Req 19.5, Req 28.3_
  - [ ] 3.8 `caslearn-docs-template` with `runbooks/`, `adr/`, `diagrams/`, `scripts/`, `threat-models/`, the 14-section README, and a Mermaid-rendering GitHub Action
    - _Requirements: Req 28.1, Req 28.3, Req 28.4_

- [ ] 4. Execute `bootstrap-repos.sh` to create the 15 MVP repositories
  - Run against the `caslearn-learn` org; verify every repo exists with branch protection and seeded scaffolding
  - Capture the command output in `caslearn-docs/bootstrap-run.md` as the S0 bring-up record
  - _Requirements: Req 19.2, Req 19.3_
  - _Design: Sec 17.2_

- [ ] 5. Install and configure JFrog Artifactory OSS
  - [ ] 5.1 Deploy Artifactory OSS via its community Helm chart into the shared-services cluster
    - Create repositories: Maven virtual + local + remote, npm virtual + local, Docker local + virtual
    - _Requirements: Req 22.2_
    - _Design: Sec 23 (OQ-8)_
  - [ ] 5.2 Publish the default deploy credentials to the Jenkins credentials store as `artifactory-deployer`
    - Add a cleanup policy retaining the last 30 image tags per repo
    - _Requirements: Req 22.2_

- [ ] 6. Install and configure SonarQube OSS
  - Deploy SonarQube Community Edition via Helm; pre-create a project per MVP repo using its webhook API
  - Configure the SonarQube quality gate to block PRs below 70% new-code coverage (Req 29.3)
  - _Requirements: Req 22.1, Req 29.3_

- [ ] 7. Build the `caslearn-jenkins-build` shared library
  - [ ] 7.1 Implement the `caslearnServicePipeline` Groovy `vars` entrypoint
    - Stages: checkout → repo-name-suffix-check → lint (Spotless/Checkstyle) → unit (`mvn -DskipITs test`) → PBT (`mvn -Dgroups=property verify`) → integration (`mvn verify` with Testcontainers) → SonarQube → Trivy (fs) → Gitleaks → Trivy license → container build → Trivy (image) → publish to Artifactory → open image-tag PR against `caslearn-platform-argo`
    - _Requirements: Req 22.1, Req 22.2, Req 22.4_
    - _Design: Sec 8_
  - [ ] 7.2 Implement the `caslearnUiPipeline` Groovy `vars` entrypoint
    - Stages mirror 7.1 but with `pnpm lint`, `pnpm test`, `pnpm test:property` (fast-check), `pnpm test:e2e` (Playwright), axe-core audit, `pnpm build`
    - _Requirements: Req 22.1, Req 26.5, Req 29.2_
    - _Design: Sec 8_
  - [ ] 7.3 Implement the dedicated `repoNameSuffixCheck` stage
    - Match `-(ui|service|tf|build|helm|argo|contracts|docs)$` against `$JOB_NAME`; on miss print `REPO_NAMING_VIOLATION: <repo>` with its own GitHub status context `ci/repo-naming`
    - _Requirements: Req 19.4_
    - _Design: Sec 17.3_
  - [ ] 7.4 Implement the nightly `caslearnJMeterPipeline` pipeline template
    - 500-VU sustained profile against staging Gateway; fail on p95 > 300 ms; publish HTML report as artifact
    - _Requirements: Req 26.2, Req 26.6, Req 29.5_

- [ ] 8. Terraform modules in `caslearn-platform-tf`
  - [ ] 8.1 `modules/vpc` — single-region VPC for AWS with 3-AZ public + private subnets
    - _Requirements: Req 21.1, Req 23.4_
  - [ ] 8.2 `modules/onprem-k8s` — kubeadm/k3s bootstrap with MetalLB VIP
    - _Requirements: Req 21.1, Req 23.2_
    - _Design: Sec 6_
  - [ ] 8.3 `modules/eks` — EKS control plane + two managed node groups (general, ai-cpu)
    - _Requirements: Req 21.1, Req 23.3_
    - _Design: Sec 7_
  - [ ] 8.4 `modules/mysql` — on-prem: Percona / MySQL Operator 3-node cluster per service; AWS: RDS MySQL 8 multi-AZ with per-service schemas
    - _Requirements: Req 21.1, Req 23.2, Req 23.3, Req 5.4_
    - _Design: Sec 23 (OQ-7)_
  - [ ] 8.5 `modules/minio` — on-prem: 4-node erasure-coded MinIO; AWS: S3 bucket with versioning + SSE-KMS
    - _Requirements: Req 21.1, Req 23.2, Req 23.3_
  - [ ] 8.6 `modules/istio` — profile, `PeerAuthentication` STRICT, default-deny AuthorizationPolicy, egress `ServiceEntry` + `Sidecar` allow-list for LLM host and SMTP host
    - _Requirements: Req 5.1, Req 24.1, Req 24.2, Req 24.3_
    - _Design: Sec 5, Sec 12.1_
  - [ ] 8.7 `modules/observability` — kube-prometheus-stack, Loki, Promtail, Tempo, Grafana, Alertmanager with pre-wired `HighErrorRate`, `HighLatencyP95`, `AuditChainBroken`, `TraceGapDetected`, `AnalyticsAuthzUnavailable` rules
    - _Requirements: Req 25.1, Req 25.5, Req 25.6, Req 25.7_
    - _Design: Sec 15.5_

- [ ] 9. Environment compositions in `caslearn-env-tf`
  - [ ] 9.1 `envs/dev` composition wiring the on-prem modules against a local kind cluster
    - _Requirements: Req 21.1, Req 21.4_
  - [ ] 9.2 `envs/staging` composition (on-prem) with persistent storage and real MinIO / MySQL backing
    - _Requirements: Req 21.1, Req 21.4_
  - [ ] 9.3 `envs/prod-onprem` composition
    - _Requirements: Req 21.1, Req 21.4_
  - [ ] 9.4 `envs/prod-aws` composition using VPC + EKS + RDS + S3 + ElastiCache + External Secrets Operator
    - _Requirements: Req 21.1, Req 21.4, Req 23.3_

- [ ] 10. Install Atlantis and wire it to `caslearn-env-tf`
  - Deploy Atlantis to the shared-services cluster behind Istio ingress
  - Write `atlantis.yaml` in `caslearn-env-tf` requiring a second approval on any plan that destroys a MySQL / MinIO / PV resource (Req 21.5)
  - _Requirements: Req 21.2, Req 21.3, Req 21.5_

- [ ] 11. Scaffold `caslearn-platform-helm`
  - [ ] 11.1 Create per-service chart skeletons: `caslearn-auth-service`, `caslearn-course-service`, `caslearn-assessment-service`, `caslearn-messaging-service`, `caslearn-analytics-service`, `caslearn-ai-service`, `caslearn-gateway-service`, `caslearn-web-ui`
    - Each chart ships `Chart.yaml`, `values.yaml`, `values.onprem.yaml`, `values.aws.yaml`, `templates/deployment.yaml`, `templates/service.yaml`, `templates/authpolicy.yaml`, `templates/servicemonitor.yaml`
    - _Requirements: Req 23.1, Req 24.2_
    - _Design: Sec 12.1_
  - [ ] 11.2 Create the umbrella chart `umbrella/caslearn-platform`
    - Depends on every service chart; `values.onprem.yaml` / `values.aws.yaml` only toggle storage class + ingress annotations
    - _Requirements: Req 23.1, Req 23.4_
  - [ ] 11.3 Implement the shared `pre-install` portability validation hook
    - Reject overrides that mix on-prem and AWS storage (e.g., `minio.enabled=true` with `s3.enabled=true`); emit clear error text from the hook
    - _Requirements: Req 23.2, Req 23.3_
    - _Design: Sec 9 (portability)_
  - [ ] 11.4 Write `test/helm/portability_test.sh` — the CP-10 check
    - `helm template` each chart with `onprem` vs `aws` values, diff resource kinds + counts, fail on drift
    - **Property: CP-10 (Helm chart portability)**
    - _Requirements: Req 23.4_
    - _Design: Sec 21 (CP-10)_
    - _Property: CP-10_
  - [ ] 11.5* Add a `helm-unittest` suite for every chart asserting resource kinds, labels, and RBAC bindings
    - _Requirements: Req 23.4_

- [ ] 12. Install ArgoCD and wire `caslearn-platform-argo`
  - Deploy ArgoCD via Helm; seed `app-of-apps.yaml` pointing at the `apps/dev/` folder; configure SSO-less static admin for S0 and switch to OIDC in S1
  - Add the `caslearn-platform` umbrella chart to `apps/dev/` so Argo auto-syncs on merge
  - _Requirements: Req 22.3_
  - _Design: Sec 6, Sec 7, Sec 8_

- [ ] 13. Install the observability stack
  - [ ] 13.1 Deploy kube-prometheus-stack, Loki, Promtail, Tempo, Alertmanager via Helm in ns `caslearn-observability`
    - _Requirements: Req 25.1_
  - [ ] 13.2 Seed Grafana with a per-service dashboard template (RED panels + DB pool + heap + GC + domain metrics row) and the `Admin Overview` dashboard
    - **Admin Overview** panels: per-service request rate, error rate, p95 latency, CPU, memory, active user count (5-min unique `user_id` at gateway)
    - _Requirements: Req 8.2, Req 8.3, Req 25.5_
    - _Design: Sec 15.4_
  - [ ] 13.3 Wire Alertmanager rules: `HighErrorRate`, `HighLatencyP95` (separate alert, not combined), `AuditChainBroken`, `TraceGapDetected`, `AnalyticsAuthzUnavailable`, `AiBudgetBurn` (stub until S5)
    - _Requirements: Req 25.6, Req 25.7, Req 13.4_
    - _Design: Sec 15.5_

- [ ] 14. Install Sealed Secrets controller and cert-manager
  - [ ] 14.1 Deploy `sealed-secrets-controller` via Helm; export the bootstrap key to the operator's password vault; document restore in `caslearn-docs/runbooks/sealed-secrets-key-restore.md`
    - _Requirements: Req 5.3_
  - [ ] 14.2 Deploy cert-manager; create a local self-signed CA `ClusterIssuer` for dev; configure Let's Encrypt issuer for staging/prod
    - _Requirements: Req 5.2_

- [ ] 15. Bootstrap `caslearn-api-contracts`
  - Create empty OpenAPI 3.1 files `auth.yaml`, `course.yaml`, `assessment.yaml`, `messaging.yaml`, `analytics.yaml`, `ai.yaml`, `gateway.yaml`
  - Add Redocly config + GitHub Action that renders HTML on push and publishes the Redocly site as part of `caslearn-docs` on merge to `main`
  - _Requirements: Req 4.1, Req 28.3_
  - _Design: Sec 10_

- [ ] 16. Scaffold `caslearn-web-ui`
  - Vite + React 18 + TypeScript strict + Redux Toolkit + RTK Query + MUI + Tailwind + React Router v6 + React Hook Form + Zod + Vitest + Playwright + fast-check + axe-core
  - Ship a "Hello from CasLearn Web UI" landing page rendered under MUI `ThemeProvider`; wire design tokens in `src/styles/tokens.ts`; configure persona-split lazy routes `/admin/*`, `/instructor/*`, `/student/*` with placeholder pages
  - Include `openapi-typescript-codegen` as a post-install hook pulling from `caslearn-api-contracts`
  - _Requirements: Req 19.5, Req 26.5, Req 28.1_
  - _Design: Sec 20.1, Sec 20.4_

- [ ] 17. Scaffold `caslearn-gateway-service`
  - Spring Cloud Gateway with: `/actuator/health`, stub `JwtAuthenticationFilter`, stub OpenAPI validation filter, stub per-JWT-subject rate limiter, `traceparent` propagation filter, standard error envelope, MDC filter populating `request_id` + `trace_id` + `span_id` + `user_id`
  - _Requirements: Req 1.3, Req 4.1, Req 4.2, Req 25.4_
  - _Design: Sec 11_

- [ ] 18. Scaffold the six backend service skeletons
  - Each: Spring Boot 3 + Java 21 + Maven + Spring Security + Spring Data JPA + Flyway + Actuator + Micrometer Tracing OTLP → Tempo + Logback JSON encoder with the mandatory log fields + Testcontainers + jqwik + shared `caslearn-audit` dep + `/actuator/health` + standard error envelope
  - [ ] 18.1 `caslearn-auth-service` skeleton with `application.yml` pointing at its own `auth` schema
    - _Requirements: Req 19.5, Req 25.3_
  - [ ] 18.2 `caslearn-course-service` skeleton
    - _Requirements: Req 19.5, Req 25.3_
  - [ ] 18.3 `caslearn-assessment-service` skeleton
    - _Requirements: Req 19.5, Req 25.3_
  - [ ] 18.4 `caslearn-messaging-service` skeleton
    - _Requirements: Req 19.5, Req 25.3_
  - [ ] 18.5 `caslearn-analytics-service` skeleton
    - _Requirements: Req 19.5, Req 25.3_
  - [ ] 18.6 `caslearn-ai-service` skeleton
    - _Requirements: Req 19.5, Req 25.3_

- [ ] 19. Build the shared `caslearn-audit` library
  - Published as a Maven artifact to Artifactory; consumed by every `*-service`
  - [ ] 19.1 Implement `AuditChainWriter`
    - Canonical JSON serialisation, pessimistic row-lock on last audit row within enclosing TX, SHA-256 hash chain (`entry_hash = SHA-256(canonical_payload || prev_hash)`), seed `prev_hash` = 64 zero bytes on first row
    - On persistence failure: throw `AuditWriteException` → enclosing request returns HTTP 500 and increments `audit_log_write_failure_total`
    - _Requirements: Req 3.1, Req 3.2, Req 3.3, Req 3.4, Req 3.6_
    - _Design: Sec 14.2_
    - _Property: CP-5_
  - [ ] 19.2 Implement `AuditMaintenanceFlag` + admin-only endpoint contract
    - When flag is set, writes are skipped and `audit_log_disabled_requests_total` increments; flipping the flag is itself audit-logged
    - _Requirements: Req 3.7_
    - _Design: Sec 14.5_
  - [ ] 19.3 Implement `AuditChainVerifierJob` skeleton (Spring `@Scheduled`)
    - Walks table in `id` order, recomputes `entry_hash`, increments `audit_chain_broken_total` and fires `AuditChainBroken` on first mismatch
    - _Requirements: Req 3.3_
    - _Design: Sec 14.3_
  - [ ] 19.4 Implement signed audit export: `AuditExportService.exportRange(from, to)`
    - NDJSON bundle + detached signature (key distinct from JWT signing key) + public key in a tarball
    - _Requirements: Req 3.5, Req 27.2_
    - _Design: Sec 14.4_
  - [ ] 19.5* Write `ChainIntegrityProperty` PBT (jqwik, stateful) in the library's own test suite
    - **Property 5: Audit Log Append-Only + Tamper-Evident**
    - **Validates: Requirements 3.1–3.4, CP-5**
    - _Property: CP-5_

- [ ] 20. Run `day0-up.sh` end-to-end against kind and wire it to CI
  - [ ] 20.1 Execute `day0-up.sh` + `smoke.sh` on a 16 GiB / 4 CPU developer laptop; record total time; iterate until under 10 minutes
    - _Requirements: Req 20.4_
    - _Design: Sec 16.1_
  - [ ] 20.2 Add `.github/workflows/smoke.yaml` to both `caslearn-platform-helm` and `caslearn-docs`
    - Runs `day0-up.sh`, uploads `/tmp/caslearn-day0/**` as artifact on failure, blocks merge on failure
    - _Requirements: Req 20.6_
    - _Design: Sec 16.4_
  - [ ] 20.3 Implement the `override: smoke-test REASON=<text>` override path
    - PR-comment-triggered bypass gated on `@caslearn/release-override` team membership with signed-commit check; emits `SMOKE_TEST_OVERRIDE` audit entry and increments `smoke_test_override_total`
    - _Requirements: Req 20.7_
    - _Design: Sec 16.5_

- [ ] 21. Write `caslearn-docs/onboarding.md` and run it end-to-end
  - Walk a fresh laptop from zero to a merged PR: access, tool install (JDK 21, Node 20, pnpm 9, Docker, kubectl, helm, kind, gh, terraform), clone all MVP repos, `day0-up.sh`, pick a good-first-issue, open PR
  - Measured success: a volunteer who has never seen the repo completes the guide in under one working day
  - _Requirements: Req 28.4, Req 28.5_

- [ ] 22. Author foundational ADRs
  - [ ] 22.1 Add ADR template at `caslearn-docs/adr/ADR-TEMPLATE.md` (Michael Nygard format)
    - _Requirements: Req 28.3_
  - [ ] 22.2 ADR-0001: Choose MUI over Chakra (resolves OQ-4)
    - _Design: Sec 20.1, Sec 23 (OQ-4)_
  - [ ] 22.3 ADR-0002: JFrog Artifactory for frontend artifacts (resolves OQ-8)
    - _Design: Sec 23 (OQ-8)_

- [ ] 23. Resolve the 12 open questions from design Sec 23
  - One decision per question, recorded either as an ADR or as an addendum in `caslearn-docs/adr/`. Each has an owner and a target S0 day.
  - [ ] 23.1 OQ-1 email provider (owner: Platform Eng, target: S0 day 3) — record choice and update Messaging service config
    - _Design: Sec 23 (OQ-1)_
  - [ ] 23.2 OQ-2 OIDC issuer (owner: Tech Lead, target: S0 day 4) — record choice in ADR-0003
    - _Design: Sec 23 (OQ-2)_
  - [ ] 23.3 OQ-3 LLM vendor + rate limits + daily budget (owner: AI Lead, target: S0 day 5)
    - _Design: Sec 23 (OQ-3)_
  - [ ] 23.4 OQ-4 MUI vs Chakra — decision recorded in ADR-0001 (task 22.2); mark resolved
    - _Design: Sec 23 (OQ-4)_
  - [ ] 23.5 OQ-5 Istio ambient vs sidecar (owner: Platform Eng, target: S0 day 5)
    - _Design: Sec 23 (OQ-5)_
  - [ ] 23.6 OQ-6 Vector store FAISS vs Qdrant (owner: AI Lead, target: S0 day 4)
    - _Design: Sec 23 (OQ-6)_
  - [ ] 23.7 OQ-7 MySQL deployment mode (owner: Platform Eng, target: S0 day 6)
    - _Design: Sec 23 (OQ-7)_
  - [ ] 23.8 OQ-8 Frontend artifact destination — decision recorded in ADR-0002 (task 22.3); mark resolved
    - _Design: Sec 23 (OQ-8)_
  - [ ] 23.9 OQ-9 Branch-protection review count — wired into task 1.2; confirm and close
    - _Design: Sec 23 (OQ-9)_
  - [ ] 23.10 OQ-10 On-call rotation tooling (owner: Platform Eng, target: S0 day 7)
    - _Design: Sec 23 (OQ-10)_
  - [ ] 23.11 OQ-11 AI conversation retention policy (owner: Product + Security, target: S0 day 5)
    - _Design: Sec 23 (OQ-11), Req 17.2, Req 27.3_
  - [ ] 23.12 OQ-12 Seeded demo data set — captured in `seed-demo.sh` (task 2.4); mark resolved
    - _Design: Sec 23 (OQ-12)_

- [ ] 24. S0 checkpoint
  - Ensure all tests pass, `day0-up.sh` is green in CI, and every OQ has a recorded answer. Ask the user if questions arise.


## Sprint 1 — Auth, RBAC, User Management, Web UI Shell + Login

Goal per `delivery.md` S1: users can authenticate and land on a role-scoped home page; RBAC is enforced at mesh + app; first correctness properties (CP-3, CP-5, CP-11) run in CI.

- [ ] 25. Implement Auth Service data layer
  - [ ] 25.1 Flyway migrations for `users`, `roles`, `refresh_tokens`, `password_policies`, `user_preferences`, `audit_log`, `system_policies` (holding the `audit_maintenance` flag)
    - Include all indexes listed in design Sec 9.1
    - _Requirements: Req 1.1, Req 2.1, Req 3.2, Req 6.1, Req 8.1_
    - _Design: Sec 9.1, Sec 14.5_
  - [ ] 25.2 Implement JPA entities + repositories in `com.caslearn.auth.domain` and `com.caslearn.auth.repository`
    - _Requirements: Req 1.1, Req 2.1, Req 6.1_

- [ ] 26. Implement Auth Service authentication endpoints
  - [ ] 26.1 `POST /v1/auth/login` with `PasswordHasher` (bcrypt cost ≥ 12) and JWT signing via `JwtSigner` (RSA, hook for 90-day rotation with 24h overlap)
    - Returns `{accessToken, refreshToken, expiresIn}` on success; on failure returns opaque 401 (Req 1.2) and increments lockout counter
    - JWT claim set: `sub`, `role`, `iss`, `exp ≤ now + 60m`, `iat`, `jti`
    - _Requirements: Req 1.1, Req 1.2, Req 1.7_
    - _Design: Sec 4.1_
    - _Property: CP-11_
  - [ ] 26.2 `POST /v1/auth/refresh` with refresh-token rotation on every call, storing hashed tokens in `refresh_tokens`
    - _Requirements: Req 1.5_
  - [ ] 26.3 `POST /v1/auth/logout` revoking the refresh token
    - _Requirements: Req 1.5_
  - [ ] 26.4 Implement `AccountLockoutService` — 5 fails per 10 minutes → 15-minute lockout + `LOGIN_LOCKED` audit entry
    - _Requirements: Req 1.6_
    - _Design: Sec 4.1_
  - [ ] 26.5 Publish `auth.yaml` to `caslearn-api-contracts` and run contract tests
    - _Requirements: Req 4.1, Req 28.3_
    - _Property: CP-12_
  - [ ] 26.6* Write `JwtValidationProperty` PBT in `caslearn-gateway-service`
    - **Property 11: JWT Validation Completeness**
    - **Validates: Req 1.3, Req 1.4, CP-11**
    - Generator mutates each JWT field (signature, exp, iss, alg) independently; assert `forwarded ⇔ valid signature ∧ exp>now ∧ iss=expected`
    - _Property: CP-11_

- [ ] 27. Implement the shared `caslearn-security-lib`
  - Published as a Maven artifact; consumed by every `*-service`
  - [ ] 27.1 Define `@RequireRole`, `@RequireCourseOwnership`, `@RequireEnrollment` annotations backed by `MethodSecurityExpressionHandler`
    - _Requirements: Req 2.3, Req 2.6_
    - _Design: Sec 12.2_
  - [ ] 27.2 Implement `RoleFieldRejectionFilter` (servlet filter running before all controllers)
    - Inspects body, headers, query string for any field literally named `role`; rejects with HTTP 403 + `ROLE_ESCALATION_BLOCKED` audit entry unless the path is the admin `PATCH /v1/users/{id}/role`
    - **Never** reads the `role` claim from anywhere but the JWT
    - _Requirements: Req 2.5, Req 2.6_
    - _Design: Sec 12.2_
    - _Property: CP-3_
  - [ ] 27.3 Implement `RbacResponseStrategy` — decision table `{role, resource_type} → {403, 404}` for information-leak-preserving 404s
    - _Requirements: Req 2.4_
    - _Design: Sec 12.2_
  - [ ] 27.4 Implement `RbacFailClosedAdvice` — when Auth is unreachable and cache missing, return HTTP 503 with `code: RBAC_UNAVAILABLE` and increment `rbac_unavailable_total`
    - _Requirements: Req 13.4_
    - _Design: Sec 11.5, Sec 19.6_

- [ ] 28. Implement Auth Service user management endpoints
  - [ ] 28.1 `POST /v1/users` — admin creates user in `PENDING_ACTIVATION` + emits activation email through Notification_Subsystem (stub that writes to `notification_outbox` until S4)
    - _Requirements: Req 6.1_
  - [ ] 28.2 `PATCH /v1/users/{id}` — update display name, email, status
    - _Requirements: Req 6.1_
  - [ ] 28.3 `DELETE /v1/users/{id}` — deactivate; invalidate all active JWTs and refresh tokens for that user within 5 minutes (push a user-id to a Redis `revoked` set keyed by JTI)
    - _Requirements: Req 6.2_
  - [ ] 28.4 `PATCH /v1/users/{id}/role` — admin-only; persist change, emit `ROLE_CHANGED` audit entry, require user to re-authenticate (increment `token_version` in `users`; Gateway checks it)
    - _Requirements: Req 6.3_
  - [ ] 28.5 LAST_ADMIN_PROTECTION guard — reject `DELETE` / role-demote on the last `ADMIN` with HTTP 409 + `LAST_ADMIN_PROTECTION`
    - _Requirements: Req 6.4_
  - [ ] 28.6 `GET /v1/users?email=&role=&status=&page=&size=` with page size validated between 10 and 100
    - _Requirements: Req 6.5_

- [ ] 29. Implement Auth Service system-policy + audit-log admin endpoints
  - [ ] 29.1 `PUT /v1/policies/{kind}` — admin updates password / session / upload / rate-limit policies; emits `POLICY_CHANGED` audit entry
    - _Requirements: Req 8.1_
  - [ ] 29.2 `GET /v1/audit` with filter by actor, action, resource type, date range (UI-backed read path)
    - _Requirements: Req 8.4_
  - [ ] 29.3 `GET /v1/audit/export?from=&to=` returns tarball with NDJSON + detached signature + public key (delegates to `AuditExportService`)
    - _Requirements: Req 3.5, Req 27.2_
  - [ ] 29.4 `POST /v1/audit/maintenance` admin toggle — flips `AuditMaintenanceFlag`; itself audit-logged
    - _Requirements: Req 3.7_
    - _Design: Sec 14.5_

- [ ] 30. Implement the Istio `AuthorizationPolicy` for Auth Service
  - Only the Gateway ServiceAccount may call `/v1/auth/*`; admin-only endpoints (`/v1/users/**`, `/v1/policies/**`, `/v1/audit/**`) gated by the `request.auth.claims[role]=ADMIN` match
  - Ship the policy as `caslearn-platform-helm/charts/caslearn-auth-service/templates/authpolicy.yaml`
  - _Requirements: Req 24.2, Req 24.3_
  - _Design: Sec 12.1_

- [ ] 31. Implement Gateway Service filters
  - [ ] 31.1 `JwtAuthenticationFilter` — validates signature against current + previous signing key, `exp > now`, `iss = expected`; rejects with 401 `TOKEN_EXPIRED` on expiry
    - _Requirements: Req 1.3, Req 1.4, Req 4.1_
    - _Design: Sec 4.1_
    - _Property: CP-11_
  - [ ] 31.2 `OpenApiValidationFilter` — loads every spec from `caslearn-api-contracts` at startup and validates path/method/query/body/response schema; malformed → 400
    - _Requirements: Req 4.1_
    - _Design: Sec 10_
    - _Property: CP-12_
  - [ ] 31.3 `RateLimiter` — 100 rps per JWT subject sliding window in Redis; on exceed → 429 with `Retry-After`
    - Enforce the 25 MiB body size limit at the gateway level except for whitelisted upload endpoints
    - _Requirements: Req 4.2, Req 4.3_
  - [ ] 31.4 `TraceparentFilter` — generate or propagate W3C `traceparent`; if downstream fails to echo it, increment `trace_propagation_failure_total` and continue normally
    - _Requirements: Req 25.4_
    - _Design: Sec 11_
  - [ ] 31.5 `InputPatternFilter` — SQL / shell / script-injection pattern rejection; on match → 400 + `INPUT_VALIDATION_REJECTED` audit entry
    - _Requirements: Req 4.5_
  - [ ] 31.6* Write `JwtValidationProperty` PBT against the gateway filter
    - **Property 11: JWT Validation Completeness**
    - **Validates: Req 1.3, Req 1.4, CP-11**
    - _Property: CP-11_
  - [ ] 31.7* Write `NoRoleEscalationProperty` PBT in Auth (cross-tested via gateway)
    - **Property 3: No Role Escalation**
    - **Validates: Req 2.5, Req 2.6, CP-3**
    - Fuzzes `role` field across body/header/query/token; asserts post-request user role unchanged unless admin-initiated
    - _Property: CP-3_

- [ ] 32. Wire auth/rbac telemetry
  - Emit and scrape `audit_log_write_failure_total`, `audit_log_disabled_requests_total`, `rbac_unavailable_total`, `trace_propagation_failure_total`, `smoke_test_override_total`; add an Auth-service row to the per-service Grafana dashboard
  - _Requirements: Req 3.6, Req 3.7, Req 13.4, Req 25.4_
  - _Design: Sec 15.1_

- [ ] 33. Implement Web UI auth shell
  - [ ] 33.1 Login page under `/login` — MUI form with Zod-validated email + password, calls `POST /v1/auth/login`, stores access token in memory, refresh in httpOnly cookie
    - _Requirements: Req 1.1, Req 26.5_
    - _Design: Sec 20.4_
  - [ ] 33.2 `fetchWithAuth` wrapper attaching the access token + per-request `x-request-id` (UUID); handles 401 `TOKEN_EXPIRED` by auto-refreshing once then retrying
    - _Requirements: Req 1.3, Req 1.5, Req 25.4_
    - _Design: Sec 20.6_
  - [ ] 33.3 Protected route component `<RequireRole roles={...}>` that redirects unauthenticated users to `/login` and forbidden roles to a shared `/forbidden` page
    - _Requirements: Req 2.3_
  - [ ] 33.4 Role-scoped home pages: `/admin/overview`, `/instructor/courses`, `/student/courses` (placeholder content, real data in later sprints)
    - _Requirements: Req 26.5_
    - _Design: Sec 20.4_
  - [ ] 33.5 Persona-aware global navigation — left sidebar on desktop, bottom tabs on mobile; highlight by role; accessible via keyboard (tested with axe-core)
    - _Requirements: Req 26.5, Req 30.2_
  - [ ] 33.6 User-management screens (admin) — list + create + edit + role change + deactivate, backed by the RTK Query `authApi` slice generated from `auth.yaml`
    - _Requirements: Req 6.1, Req 6.2, Req 6.3, Req 6.5_
  - [ ] 33.7 Audit-log browser screen (admin) with filter by actor / action / resource / date range
    - _Requirements: Req 8.4_
  - [ ] 33.8* Playwright E2E: `admin-login-and-see-dashboard.spec.ts`, `student-login-redirects-to-courses.spec.ts`, `forbidden-role-redirects.spec.ts`
    - _Requirements: Req 26.5_
  - [ ] 33.9* axe-core a11y check for login page + role-scoped home pages
    - _Requirements: Req 26.5, Req 30.2_

- [ ] 34. Sprint 1 checkpoint
  - Ensure all tests pass, CP-3, CP-5 (initial), and CP-11 property tests run in CI, and a seeded admin can log in via Web UI against the Gateway. Ask the user if questions arise.


## Sprint 2 — Courses, Enrollments, Content Upload/Read

Goal per `delivery.md` S2: instructors can author courses and content, admins can manage lifecycle, students can enroll and read materials; CP-1 and CP-8 pass in CI.

- [ ] 35. Implement Course Service data layer
  - [ ] 35.1 Flyway migrations for `courses`, `modules`, `content_items`, `content_versions`, `enrollments`, `audit_log` (in the `course` schema)
    - All indexes per design Sec 9.2 (`enrollments(student_id, course_id, status)`, `enrollments(course_id, status)`, `content_items(course_id, module_id, ordinal)`, `content_items(deleted_at)`, `courses(code)`)
    - _Requirements: Req 7.1, Req 9.1, Req 14.1_
    - _Design: Sec 9.2_
  - [ ] 35.2 Implement JPA entities + repositories in `com.caslearn.course.domain` and `com.caslearn.course.repository`
    - _Requirements: Req 7.1, Req 9.1, Req 14.1_

- [ ] 36. Implement Course Service CRUD + lifecycle
  - [ ] 36.1 `POST /v1/courses` — admin creates course in `DRAFT`; requires unique code, title, term, ≥1 instructor
    - _Requirements: Req 7.1_
  - [ ] 36.2 `PATCH /v1/courses/{id}/publish` — `DRAFT → PUBLISHED`; make visible to enrolled students
    - _Requirements: Req 7.2_
  - [ ] 36.3 `PATCH /v1/courses/{id}/archive` — `PUBLISHED → ARCHIVED`; read-only for all
    - _Requirements: Req 7.3_
  - [ ] 36.4 `DELETE /v1/courses/{id}` — reject with HTTP 409 `COURSE_HAS_SUBMISSIONS` if Assessment Service reports any submissions (sync internal call)
    - _Requirements: Req 7.4_
  - [ ] 36.5 Publish `course.yaml` to `caslearn-api-contracts`
    - _Requirements: Req 4.1, Req 28.3_
    - _Property: CP-12_

- [ ] 37. Implement object-storage abstraction `ObjectStorageClient`
  - S3 SDK-based implementation; MinIO endpoint locally, S3 in AWS; same API surface
  - Methods: `putObject(key, stream, metadata)`, `getObject(key)`, `deleteObject(key)`, `presignGetUrl(key, ttl)`
  - Chart values file chooses endpoint, region, credentials
  - _Requirements: Req 9.1, Req 23.2, Req 23.3_

- [ ] 38. Implement content upload + read
  - [ ] 38.1 `POST /v1/courses/{id}/content` (multipart) — validates ownership **before** writing (ownership check uses `@RequireCourseOwnership`, not after file bytes are consumed); MIME allow-list from Req 9.2; size ≤ 500 MiB; persists via `ObjectStorageClient`; records `content_items` + `content_versions` row; emits `CONTENT_UPLOADED` audit entry
    - On MIME miss → 415 `UNSUPPORTED_MIME`; on size overflow → 413 `FILE_TOO_LARGE`; on non-owner → 403 before opening stream (Req 9.3)
    - _Requirements: Req 9.1, Req 9.2, Req 9.3_
    - _Design: Sec 10.1_
  - [ ] 38.2 `PUT /v1/courses/{id}/modules` — reorder modules and content items; persist ordinals; return ordered structure on read
    - _Requirements: Req 9.4_
  - [ ] 38.3 `DELETE /v1/content/{id}` — soft-delete (set `deleted_at`); retain object in storage for 30 days; scheduled purge job removes objects after 30 days; emits `CONTENT_DELETED` audit entry
    - _Requirements: Req 9.5_
  - [ ] 38.4 `GET /v1/courses/{id}/content` — returns ordered module + content list, enforced by `@RequireEnrollment`; course status must be in `{PUBLISHED, ARCHIVED}`
    - _Requirements: Req 14.2_
    - _Property: CP-1_
  - [ ] 38.5 `GET /v1/content/{id}` — returns presigned URL for the object; only accessible to enrolled students or the owning instructor or admins
    - _Requirements: Req 14.2, Req 14.3_
    - _Property: CP-1_
  - [ ] 38.6 `GET /internal/authorized-content-items?studentId=&courseId=` — internal endpoint callable only by AI Service SA (Istio policy) returning the set of content_item_ids this student may read
    - _Requirements: Req 17.4, Req 27.1_
    - _Design: Sec 12.1, Sec 13.2_
    - _Property: CP-6_
  - [ ] 38.7* Write `EnrollmentGateProperty` PBT (jqwik)
    - **Property 1: Enrollment Gate on Course Materials**
    - **Validates: Req 14.2, Req 14.3, CP-1**
    - Generators vary `(student, course, enrollment_status ∈ {ACTIVE,DROPPED,COMPLETED,none}) × (course_status ∈ {DRAFT,PUBLISHED,ARCHIVED})`; assert read allowed iff ACTIVE ∧ status ∈ {PUBLISHED, ARCHIVED}
    - _Property: CP-1_

- [ ] 39. Implement enrollment workflows
  - [ ] 39.1 `POST /v1/courses/{id}/enrollments` — student self-enrollment for `PUBLISHED` courses allowing it; admin can enroll any student
    - _Requirements: Req 14.1_
  - [ ] 39.2 `DELETE /v1/courses/{id}/enrollments/me` — student drops course; transitions enrollment to `DROPPED`; removes access to materials
    - _Requirements: Req 14.4_
  - [ ] 39.3 Use row-level locks + transactions on enrollment insert so concurrent issue/drop maintain `count(ACTIVE) = issued − drops`
    - _Requirements: Req 14.1_
    - _Design: Sec 18.2_
    - _Property: CP-8_
  - [ ] 39.4 Publish `ContentItemCreated` event via outbox (records in `content_item_outbox`) for future AI ingestion (S5)
    - _Requirements: Req 17.1_
    - _Design: Sec 3_
  - [ ] 39.5* Write `ConcurrentEnrollmentProperty` PBT (jqwik stateful, Testcontainers MySQL)
    - **Property 8: Enrollment Count Invariant Under Concurrent Operations**
    - **Validates: Req 14.1, CP-8**
    - Generates interleaved enroll/drop operations from N virtual students; after execution asserts `count(ACTIVE) = issued − drops`
    - _Property: CP-8_

- [ ] 40. Implement Istio `AuthorizationPolicy` for Course Service
  - Allow Gateway SA on public paths; allow AI SA on `/internal/authorized-content-items`; allow Assessment SA on `/internal/enrollments/{id}/check`; default deny
  - _Requirements: Req 24.2, Req 24.3_
  - _Design: Sec 12.1_

- [ ] 41. Implement Web UI course browsing and authoring screens
  - [ ] 41.1 Student: course browse page (list PUBLISHED courses with search/filter) and course detail page showing module tree + content viewer (PDF inline, video player, markdown renderer)
    - _Requirements: Req 14.1, Req 14.2, Req 26.5_
    - _Design: Sec 20.4_
  - [ ] 41.2 Student: enrollment flow (enroll / drop buttons gated on course + enrollment state)
    - _Requirements: Req 14.1, Req 14.4_
  - [ ] 41.3 Instructor: course-list and course-authoring screens — module/content CRUD with drag-and-drop ordering; upload form with MIME + size validation client-side (MUI FilePond integration)
    - _Requirements: Req 9.1, Req 9.2, Req 9.4_
    - _Design: Sec 20.4_
  - [ ] 41.4 Admin: course lifecycle screen — publish / archive / delete; show instructor roster
    - _Requirements: Req 7.1, Req 7.2, Req 7.3, Req 7.4_
  - [ ] 41.5* Playwright E2E: `instructor-creates-course.spec.ts`, `instructor-uploads-content.spec.ts`, `student-enrolls-and-reads-content.spec.ts`, `student-cannot-read-without-enrollment.spec.ts`
    - _Requirements: Req 26.5_
    - _Property: CP-1_
  - [ ] 41.6* Vitest unit tests for module-reordering client logic (fast-check property tests on ordinal stability)
    - _Requirements: Req 9.4_
  - [ ] 41.7* axe-core audit on every new screen
    - _Requirements: Req 26.5, Req 30.2_

- [ ] 42. Sprint 2 checkpoint
  - Ensure all tests pass (including CP-1 and CP-8 PBTs) and a seeded student can enroll in a seeded course and render a PDF content item end-to-end. Ask the user if questions arise.


## Sprint 3 — Assessments, Grading, Grade Book

Goal per `delivery.md` S3: instructors can author quizzes/assignments, students can submit, auto-grading works for objective questions, manual grading + grade book work; CP-2, CP-4, CP-7 pass in CI.

- [ ] 43. Implement Assessment Service data layer
  - Flyway migrations for the full ERD in design Sec 9.3: `quizzes`, `questions`, `question_options`, `assignments`, `rubrics`, `rubric_items`, `submissions`, `quiz_attempts`, `attempt_answers`, `grade_book_entries`, `grade_history`, `audit_log`
  - All indexes in design Sec 9.3, in particular `submissions(student_id, assignment_id)` unique, `grade_book_entries(course_id, student_id)`, `grade_book_entries(released, released_at)`
  - _Requirements: Req 10.1, Req 11.1, Req 15.1_
  - _Design: Sec 9.3_

- [ ] 44. Implement quiz authoring
  - [ ] 44.1 `POST /v1/quizzes` — instructor creates quiz with questions of type `{MULTIPLE_CHOICE, TRUE_FALSE, SHORT_ANSWER, ESSAY}` in `DRAFT`
    - _Requirements: Req 10.1_
  - [ ] 44.2 `PATCH /v1/quizzes/{id}/publish` — validate `start_at`, `due_at`, `time_limit_seconds`; assert `sum(question.points) == quiz.total_points` else reject with HTTP 400 `POINTS_MISMATCH`
    - _Requirements: Req 10.2, Req 10.4_
  - [ ] 44.3 Edit-locked quiz guard — if any student has a `quiz_attempt` with `started_at` set, reject `PATCH /v1/quizzes/{id}` with HTTP 409 `QUIZ_IN_PROGRESS`
    - _Requirements: Req 10.5_
  - [ ] 44.4 Publish `assessment.yaml` to `caslearn-api-contracts`
    - _Requirements: Req 4.1, Req 28.3_
    - _Property: CP-12_

- [ ] 45. Implement assignment authoring with rubrics
  - [ ] 45.1 `POST /v1/assignments` — instructor creates assignment with due date, allowed MIME types, max file size, optional rubric id
    - _Requirements: Req 10.3_
  - [ ] 45.2 `POST /v1/rubrics` and `POST /v1/rubrics/{id}/items` — rubric authoring with per-item max points
    - _Requirements: Req 10.3, Req 11.3_

- [ ] 46. Implement submission handlers
  - [ ] 46.1 `POST /v1/assignments/{id}/submissions` — validate enrollment via internal Course call; enforce MIME + size per assignment; mark `LATE` if past due date and `allow_late = true`; reject past due if `allow_late = false`
    - _Requirements: Req 15.1, Req 15.2_
  - [ ] 46.2 `POST /v1/quizzes/{id}/attempts` — record `started_at`; enforce time limit with auto-submit worker that every 30 seconds closes any attempt whose elapsed time > `time_limit_seconds` by grading whatever answers are persisted at that moment
    - _Requirements: Req 15.3, Req 15.4_
  - [ ] 46.3 `POST /v1/quizzes/{id}/attempts/{attemptId}/submit` — persist answers, invoke `DeterministicGrader`
    - _Requirements: Req 11.1, Req 11.2, Req 15.3_

- [ ] 47. Implement the deterministic grader (pure function)
  - [ ] 47.1 Implement `DeterministicGrader.score(Submission, Rubric)` as a pure function with canonical answer normalisation (whitespace, case, trailing punctuation for SA; boolean equivalence for TF; option-id set equality for MC)
    - Objective-only quiz → record grade within 5 seconds (Req 11.1)
    - Mixed quiz → auto-grade objective portion, leave subjective in `AWAITING_MANUAL_GRADE` (Req 11.2)
    - _Requirements: Req 11.1, Req 11.2, Req 11.4_
    - _Design: Sec 21 (CP-4)_
    - _Property: CP-4_
  - [ ] 47.2* Write `DeterministicScoringProperty` PBT (jqwik)
    - **Property 4: Deterministic Assessment Scoring**
    - **Validates: Req 11.4, CP-4**
    - Generate `(Submission, Rubric)`; assert `score(x,r) == score(x,r)` on repeated evaluation and canonically-equal answer sets score equal
    - _Property: CP-4_

- [ ] 48. Implement grade book with release + amend
  - [ ] 48.1 `POST /v1/courses/{id}/grade-book/{studentId}/entries` — create or update grade_book_entries from grader output
    - _Requirements: Req 11.1, Req 11.3_
  - [ ] 48.2 `POST /v1/grade-book/entries/{id}/release` — set `released = true`, `released_at`; emit `GradeReleased` outbox event + `GRADE_RELEASED` audit entry
    - _Requirements: Req 11.5_
    - _Design: Sec 3_
  - [ ] 48.3 `PATCH /v1/grade-book/entries/{id}` — require `justification` field; persist old + new score via `grade_history` insert; emit `GRADE_AMENDED` audit entry
    - _Requirements: Req 11.6_
  - [ ] 48.4 `POST /v1/submissions/{id}/grade` — instructor manual grading with rubric breakdown; persist score items + total; updates grade book
    - _Requirements: Req 11.3_
  - [ ] 48.5* Write `GradeRoundTripProperty` PBT (jqwik + Testcontainers MySQL)
    - **Property 7: Quiz Scoring Round-Trip Preservation**
    - **Validates: Req 11.1, Req 11.3, CP-7**
    - Generate `(Quiz, AnswerSet)`; score → persist → reload; assert same score and per-question breakdown
    - _Property: CP-7_

- [ ] 49. Implement student grade read paths
  - [ ] 49.1 `GET /v1/students/me/grades?courseId=` — returns released grades for the caller only
    - _Requirements: Req 15.5_
  - [ ] 49.2 Cross-student read guard — any request for grades where `target_student_id != auth.subject` and the caller is not an instructor of that course or admin → 403 `OWNERSHIP_VIOLATION`; log to audit with every cross-student read by instructor/admin (Req 27.4)
    - _Requirements: Req 15.6, Req 27.4_
    - _Property: CP-2_
  - [ ] 49.3* Write `GradeConfidentialityProperty` PBT (jqwik)
    - **Property 2: Grade Confidentiality**
    - **Validates: Req 15.5, Req 15.6, CP-2**
    - Generate disjoint ownership sets for two students; drive random `GET /grades` calls by `s2 ≠ s1`; assert `s1`'s records never returned to `s2`
    - _Property: CP-2_

- [ ] 50. Implement Istio `AuthorizationPolicy` for Assessment Service
  - Allow Gateway SA on public paths; allow Messaging SA on `/internal/grade-release-events`; default deny
  - _Requirements: Req 24.2, Req 24.3_
  - _Design: Sec 12.1_

- [ ] 51. Implement Web UI assessment screens
  - [ ] 51.1 Student: quiz-taking UI with timer (shows remaining seconds from Assessment Service server clock, not client clock), answer persistence on blur, auto-submit banner
    - _Requirements: Req 15.3, Req 15.4, Req 26.5_
  - [ ] 51.2 Student: assignment-submission UI with file upload + MIME/size client-side validation
    - _Requirements: Req 15.1, Req 26.5_
  - [ ] 51.3 Student: grade view — list of graded items with score, max, rubric breakdown (when released)
    - _Requirements: Req 15.5_
  - [ ] 51.4 Instructor: quiz authoring UI — question editor with per-type option editor, points validation, publish button that surfaces `POINTS_MISMATCH` inline
    - _Requirements: Req 10.1, Req 10.2, Req 10.4_
  - [ ] 51.5 Instructor: rubric editor + assignment authoring UI
    - _Requirements: Req 10.3_
  - [ ] 51.6 Instructor: grading queue with rubric scoring, manual grading flow
    - _Requirements: Req 11.3_
  - [ ] 51.7 Instructor: grade book UI showing per-student grades for a course with release + amend actions (amend opens justification dialog)
    - _Requirements: Req 11.5, Req 11.6_
  - [ ] 51.8* Playwright E2E: `student-submits-quiz-on-time.spec.ts`, `student-submits-quiz-after-timeout.spec.ts`, `instructor-grades-assignment.spec.ts`, `student-cannot-see-peer-grades.spec.ts`
    - _Requirements: Req 15.3, Req 15.4, Req 15.6, Req 26.5_
    - _Property: CP-2_
  - [ ] 51.9* axe-core audit for the quiz-taking and grading screens (complex widgets)
    - _Requirements: Req 26.5, Req 30.2_

- [ ] 52. Wire `GradeReleased` outbox
  - Drainer polls `grade_release_outbox` every 30 s; publishes structured event to Messaging Service's internal endpoint (or writes directly into Messaging DB per design Sec 3) so Messaging can fan out notifications in S4
  - _Requirements: Req 11.5_
  - _Design: Sec 3_

- [ ] 53. Sprint 3 checkpoint
  - Ensure all tests pass (including CP-2, CP-4, CP-7 PBTs) and a seeded student can take a seeded quiz end-to-end, see an auto-graded score, and the instructor can amend it with justification. Ask the user if questions arise.


## Sprint 4 — Messaging, Notifications, Announcements, Analytics Dashboards

Goal per `delivery.md` S4: instructors can post announcements + DMs, students get in-app + email notifications with idempotency, analytics dashboards light up for instructors and admins; CP-9 passes in CI; analytics fails closed on RBAC outage.

- [ ] 54. Implement Messaging Service data layer
  - Flyway migrations for `announcements`, `direct_messages`, `message_recipients`, `notification_preferences`, `notification_outbox`, `audit_log` (in the `messaging` schema), plus all indexes per design Sec 9.4 (including unique `notification_outbox(idempotency_key)` — the CP-9 constraint)
  - _Requirements: Req 12.1, Req 12.2, Req 16.1, Req 16.3_
  - _Design: Sec 9.4_

- [ ] 55. Implement messaging endpoints
  - [ ] 55.1 `POST /v1/courses/{id}/announcements` — instructor (via `@RequireCourseOwnership`) posts; persist + create `message_recipients` rows for all `ACTIVE`-enrolled students; enqueue notifications to `notification_outbox` with idempotency key `announcement:<id>:<recipient_id>`
    - _Requirements: Req 12.1, Req 16.1_
    - _Design: Sec 9.4_
    - _Property: CP-9_
  - [ ] 55.2 `POST /v1/messages` — instructor DM to students; reject with 403 if any recipient is not enrolled in one of their courses
    - _Requirements: Req 12.2, Req 12.3_
  - [ ] 55.3 `POST /v1/messages/{id}/replies` — student reply (only to instructor of a course they're enrolled in); deliver in-app within 5s
    - _Requirements: Req 16.2_
  - [ ] 55.4 `GET /v1/inbox` and `GET /v1/announcements?courseId=` — caller's view; marks read on open
    - _Requirements: Req 12.1, Req 12.2, Req 16.1_
  - [ ] 55.5 `PUT /v1/me/notification-preferences` — student toggles email delivery
    - _Requirements: Req 16.3_
  - [ ] 55.6 Publish `messaging.yaml` to `caslearn-api-contracts`
    - _Requirements: Req 4.1, Req 28.3_
    - _Property: CP-12_

- [ ] 56. Implement notification dispatcher + outbox drainer
  - [ ] 56.1 `NotificationDispatcher` — consume from `notification_outbox`, respect student's email preference, send in-app (write `message_recipients` row if missing) and email via SMTP
    - SMTP client honours egress allow-list (Istio ServiceEntry from task 8.6); target configurable MailHog (dev) / Postfix (on-prem prod) / SES (AWS prod) per OQ-1 default
    - _Requirements: Req 16.1, Req 16.3_
    - _Design: Sec 9.4, Sec 23 (OQ-1)_
  - [ ] 56.2 Idempotent dispatch — the `idempotency_key` unique index ensures a replayed event cannot duplicate a delivery; dispatcher marks rows `SENT` + records `sent_at`, and safely no-ops on retries
    - _Requirements: Req 16.1_
    - _Property: CP-9_
  - [ ] 56.3 `OutboxDrainerScheduler` — Spring `@Scheduled` (30s); claim pending rows with `FOR UPDATE SKIP LOCKED`; dispatch; on dispatch failure mark `FAILED` with backoff
    - _Requirements: Req 16.1_
  - [ ] 56.4 Subscribe to `GradeReleased` events (task 52) — fan out notifications to the student author
    - _Requirements: Req 11.5, Req 16.1_
  - [ ] 56.5* Write `NotificationIdempotenceProperty` PBT (jqwik)
    - **Property 9: Notification Delivery Idempotence**
    - **Validates: Req 16.1, CP-9**
    - Generate notification event `e`; dispatch N≥2 times; assert exactly one in-app delivery and ≤ 1 email per recipient keyed by `(event_id, recipient_id)`
    - _Property: CP-9_

- [ ] 57. Implement Istio `AuthorizationPolicy` for Messaging Service
  - Allow Gateway SA on public paths; allow Assessment SA on `/internal/grade-release-events`; default deny. Egress ServiceEntry from task 8.6 is the only allowed SMTP path.
  - _Requirements: Req 24.2, Req 24.3_
  - _Design: Sec 5_

- [ ] 58. Implement Web UI messaging screens
  - [ ] 58.1 Inbox screen — unread filter, search, mark-as-read (optimistic update)
    - _Requirements: Req 12.2, Req 16.1, Req 26.5_
  - [ ] 58.2 Compose-DM dialog — recipient picker restricted to enrolled students (for instructor) or instructors (for student); autosuggest via `/v1/courses/{id}/roster`
    - _Requirements: Req 12.2, Req 12.3_
  - [ ] 58.3 Announcements feed on course detail page
    - _Requirements: Req 12.1, Req 16.1_
  - [ ] 58.4 Notification-preferences screen (student) toggling email + channel preferences
    - _Requirements: Req 16.3_
  - [ ] 58.5 Notification bell component in top bar with poll every 30s (or server-sent events if feasible)
    - _Requirements: Req 16.1, Req 26.5_
  - [ ] 58.6* Playwright E2E: `instructor-posts-announcement.spec.ts`, `student-receives-announcement-inapp.spec.ts`, `student-disables-email-notifications.spec.ts`
    - _Requirements: Req 12.1, Req 16.1, Req 16.3, Req 26.5_

- [ ] 59. Implement Analytics Service data layer
  - Flyway migrations for `course_metrics_daily`, `student_progress_snapshot`, `assessment_distribution`, `audit_log` (in the `analytics` schema); all indexes per design Sec 9.5
  - _Requirements: Req 8.2, Req 13.1, Req 13.2_
  - _Design: Sec 9.5_

- [ ] 60. Implement Analytics ETL jobs
  - Spring `@Scheduled` jobs every 5 minutes that poll Course + Assessment Service internal endpoints and refresh `course_metrics_daily`, `student_progress_snapshot`, `assessment_distribution`
  - _Requirements: Req 13.1, Req 13.2_
  - _Design: Sec 9.5_

- [ ] 61. Implement Analytics endpoints
  - [ ] 61.1 `GET /v1/courses/{id}/analytics` — instructor course analytics (per-assessment avg/median/distribution); gated on `@RequireCourseOwnership`; render within 3 s
    - _Requirements: Req 13.1, Req 13.2, Req 13.3_
  - [ ] 61.2 `GET /v1/courses/{id}/students/{studentId}/progress` — per-student progress; instructor must own the course
    - _Requirements: Req 13.2_
  - [ ] 61.3 `GET /v1/admin/overview-metrics` — admin-only overview feed used by the `Admin Overview` Grafana dashboard cross-reference
    - _Requirements: Req 8.2_
  - [ ] 61.4 Fail-closed RBAC — on `rbac_unavailable` from the shared library, return 503 with `code: RBAC_UNAVAILABLE` and increment `analytics_authz_unavailable_total`
    - _Requirements: Req 13.4_
    - _Design: Sec 11.5, Sec 15.5_
  - [ ] 61.5 Publish `analytics.yaml` to `caslearn-api-contracts`
    - _Requirements: Req 4.1, Req 28.3_
    - _Property: CP-12_

- [ ] 62. Implement Istio `AuthorizationPolicy` for Analytics Service
  - Allow Gateway SA; default deny
  - _Requirements: Req 24.2, Req 24.3_

- [ ] 63. Implement Web UI analytics dashboards
  - [ ] 63.1 Instructor course analytics page — MUI Charts for histogram + avg/median; table for per-student progress
    - _Requirements: Req 13.1, Req 13.2, Req 26.5_
    - _Design: Sec 20.4_
  - [ ] 63.2 Admin overview dashboard page — cross-links to Grafana `Admin Overview` + an in-app panel summarising user count, course count, recent audit highlights
    - _Requirements: Req 8.2, Req 26.5_
  - [ ] 63.3 Student progress section in `/student/courses/:id` — content completion, submission status, grade summary
    - _Requirements: Req 13.2_
  - [ ] 63.4* Playwright E2E: `instructor-opens-course-analytics.spec.ts`, `instructor-cannot-see-other-course-analytics.spec.ts`, `admin-opens-overview.spec.ts`
    - _Requirements: Req 13.1, Req 13.3, Req 26.5_
  - [ ] 63.5* axe-core audit on analytics charts — ensure color contrast + keyboard navigation on chart toggles
    - _Requirements: Req 26.5, Req 30.2_

- [ ] 64. Finalise observability for messaging + analytics
  - Add dashboards for `caslearn-messaging-service` and `caslearn-analytics-service`; wire the `AnalyticsAuthzUnavailable` alert to on-call; test a simulated Auth outage and verify the fail-closed path
  - _Requirements: Req 13.4, Req 25.5_
  - _Design: Sec 15.5_

- [ ] 65. Sprint 4 checkpoint
  - Ensure all tests pass (including CP-9 PBT), a grade release propagates end-to-end to student in-app + email, and the instructor analytics dashboard renders inside 3 seconds against seeded data. Ask the user if questions arise.


## Sprint 5 — AI Learning Assistant (RAG) + Adaptive Study Planner

Goal per `delivery.md` S5: students can ask grounded questions with citations, get refusals when off-topic, and view a personalised study plan; CP-6 passes in CI with a mocked LLM.

- [ ] 66. Implement AI Service data layer
  - Flyway migrations for `conversations`, `ai_messages`, `retrieved_chunks`, `study_plans`, `plan_items`, `audit_log`, `content_ingest_state` (tracks processed content_item ids), all indexes per design Sec 9.6
  - _Requirements: Req 17.2, Req 18.1_
  - _Design: Sec 9.6_

- [ ] 67. Implement vector-store + embedding clients
  - [ ] 67.1 `QdrantClient` wrapper — upsert + search with payload filter `content_item_id IN (...)`; FAISS embedded stub used in AI Service unit tests only (per OQ-6 default)
    - _Requirements: Req 17.1, Req 17.4_
    - _Design: Sec 13.1, Sec 13.2, Sec 23 (OQ-6)_
  - [ ] 67.2 `EmbeddingClient` — calls `text-embedding-3-small`-equivalent via the allow-listed LLM host; batches requests; caches embeddings by chunk hash
    - _Requirements: Req 17.1_
    - _Design: Sec 13.1_

- [ ] 68. Implement content ingestion worker
  - [ ] 68.1 `ContentIngestionWorker` — polls Course Service's outbox `GET /internal/content-events?sinceCursor=` for `ContentItemCreated` events; marks cursor in `content_ingest_state`
    - _Requirements: Req 17.1_
    - _Design: Sec 13.1_
  - [ ] 68.2 Chunker — 1000 tokens per chunk, 150 overlap; emits `{content_item_id, course_id, owner_instructor_id, chunk_offset, text}` to the embedder
    - _Requirements: Req 17.1_
    - _Design: Sec 13.1_
  - [ ] 68.3 Persist vectors + payload metadata to Qdrant for filtered retrieval in S5 runtime
    - _Requirements: Req 17.1, Req 17.4_
    - _Design: Sec 13.1_

- [ ] 69. Implement `RetrievalAuthorizationFilter`
  - Before any retrieval, calls `GET /internal/authorized-content-items?studentId=&courseId=` on Course Service (task 38.6); gets set `A`; all Qdrant searches for this request filter `content_item_id IN A`
  - If Course Service is unavailable, AI Service fails closed (503 `RBAC_UNAVAILABLE`)
  - _Requirements: Req 17.4, Req 27.1_
  - _Design: Sec 13.2_
  - _Property: CP-6_

- [ ] 70. Implement LLM client
  - [ ] 70.1 `LlmClient` — HTTPS POST to egress-allow-listed host; Resilience4j circuit breaker + 12 s overall timeout; pluggable model name (defaults to `gpt-4o-mini`-class per OQ-3); records token usage in `ai_messages.prompt_tokens` / `completion_tokens`
    - _Requirements: Req 17.5_
    - _Design: Sec 13.3, Sec 19.1, Sec 19.2, Sec 23 (OQ-3)_
  - [ ] 70.2 `TestLlmClient` — deterministic stub that echoes back the chunk ids it was fed; used in all CI PBT runs and unit tests
    - _Requirements: Req 29.4_
    - _Design: Sec 21 (CP-6)_
    - _Property: CP-6_

- [ ] 71. Implement `AssistantController`
  - [ ] 71.1 `POST /v1/ai/ask` — gated on `@RequireEnrollment`; applies `RetrievalAuthorizationFilter`; embeds question; searches Qdrant top-k=5 with content_item_id filter
    - _Requirements: Req 17.1, Req 17.4_
    - _Design: Sec 13.2_
  - [ ] 71.2 Refusal short-circuit — if `max(score) < 0.2` the service persists question + empty chunks + refusal, emits `AI_REFUSAL_OUT_OF_SCOPE` audit entry, and returns `{"answer":"Course materials don't cover this.","citations":[]}` without calling the LLM
    - _Requirements: Req 17.3_
    - _Design: Sec 4.3, Sec 13.2_
  - [ ] 71.3 Prompt assembler — strict citation template from design Sec 13.2; LLM is instructed to cite `[CIT:content_item_id#chunk_offset]` in every claim
    - _Requirements: Req 17.2_
    - _Design: Sec 13.2_
  - [ ] 71.4 `CitationBuilder` — parse `[CIT:id#offset]` markers from LLM response and emit structured citations; strip markers from answer text
    - _Requirements: Req 17.2_
    - _Design: Sec 13.2_
  - [ ] 71.5 Full audit — persist `{prompt, retrieved_chunks, response, tokens}` to `ai_messages` + `retrieved_chunks` and emit `AI_ANSWER_GROUNDED` audit entry
    - _Requirements: Req 17.2_
    - _Design: Sec 13.2, Sec 14.1_
  - [ ] 71.6 Publish `ai.yaml` to `caslearn-api-contracts`
    - _Requirements: Req 4.1, Req 28.3_
    - _Property: CP-12_
  - [ ] 71.7* Write `RetrievalAuthProperty` PBT (jqwik, `TestLlmClient`)
    - **Property 6: AI Retrieval Authorization**
    - **Validates: Req 17.4, CP-6**
    - Generate `(student, question, authorized_set A, full_set F)` with `A ⊆ F`; run retrieval pipeline; assert retrieved `content_item_id` set ⊆ A
    - _Property: CP-6_

- [ ] 72. Implement Adaptive Study Planner
  - [ ] 72.1 `StudyPlannerService.generate(studentId, courseId)` implementing the deterministic algorithm in design Sec 13.4
    - Pull recent grades + upcoming due dates via internal Assessment Service endpoints
    - For every graded assessment in last 30 days, compute `student_pct` vs `course_avg_pct`; flag topics where `student_pct < 0.7 * course_avg_pct`
    - Fallback: if no graded items, emit default plan from course module order
    - Sort by upcoming due date asc, then module ordinal
    - _Requirements: Req 18.1, Req 18.2, Req 18.4_
    - _Design: Sec 13.4_
  - [ ] 72.2 `POST /v1/ai/plans` — generate + persist plan; `GET /v1/ai/plans?courseId=` — read latest
    - _Requirements: Req 18.1_
  - [ ] 72.3 `PATCH /v1/ai/plans/{id}/items/{itemId}` — mark item complete; regenerate plan accordingly
    - _Requirements: Req 18.3_
  - [ ] 72.4* Unit tests for planner heuristic (no LLM involved — pure function over graded data)
    - _Requirements: Req 18.2, Req 18.4_

- [ ] 73. Implement Istio `AuthorizationPolicy` for AI Service
  - Allow Gateway SA on public paths; no other service calls AI. Only AI Service is in the egress allow-list for the LLM host.
  - _Requirements: Req 24.2, Req 24.3_
  - _Design: Sec 5_

- [ ] 74. Implement Web UI AI screens
  - [ ] 74.1 AI assistant chat widget on course-detail page — student sends question scoped to the course; response shows answer + clickable citations (deep-links to the content item at the cited chunk offset)
    - _Requirements: Req 17.1, Req 17.2, Req 26.5_
    - _Design: Sec 20.4_
  - [ ] 74.2 Refusal state — visually distinct "Course materials don't cover this" message when `citations: []`
    - _Requirements: Req 17.3_
  - [ ] 74.3 Study planner page at `/student/planner/:courseId` — list of plan items with complete/uncomplete toggles, rationale visible for each item
    - _Requirements: Req 18.1, Req 18.2, Req 18.3, Req 18.4_
  - [ ] 74.4* Playwright E2E: `student-asks-in-scope-question.spec.ts`, `student-asks-out-of-scope-question-gets-refusal.spec.ts`, `student-cannot-ask-about-other-course-materials.spec.ts`, `student-views-study-plan.spec.ts`
    - _Requirements: Req 17.1, Req 17.3, Req 17.4, Req 18.1_
    - _Property: CP-6_
  - [ ] 74.5* axe-core audit on chat UI (live region announcements for response arrival)
    - _Requirements: Req 26.5, Req 30.2_

- [ ] 75. Wire AI telemetry + budget alert
  - Export `ai_llm_tokens_total{model}`, `ai_llm_request_duration_seconds` histogram, `ai_refusal_total{reason}`
  - Alertmanager rule `AiBudgetBurn` — fires at 80% of daily spend ceiling (default $500/month for dev per OQ-3) derived from `ai_llm_tokens_total`
  - Pin retention policy for AI conversation data per OQ-11 (180 days student conversations, 30 days aggregates)
  - _Requirements: Req 17.2, Req 25.5_
  - _Design: Sec 15.1, Sec 23 (OQ-11)_

- [ ] 76. Sprint 5 checkpoint
  - Ensure all tests pass (including CP-6 PBT with mocked LLM), a seeded student asks an in-scope question and gets a cited answer, asks an out-of-scope question and gets a refusal, and the planner produces items for under-performing assessments. Ask the user if questions arise.



## Sprint 6 — Hardening, Load Testing, Accessibility, Security Review, Demo Prep

Goal per `delivery.md` S6: MVP is demo-ready — non-functional targets proven, security reviewed, backups exercised, runbooks complete. No new features, only quality, reliability, and documentation.

- [ ] 77. Wire the nightly JMeter load-test job
  - Add a Jenkins job `nightly-load-test` in `caslearn-jenkins-build` scheduled at 02:00 UTC
  - Execute a JMeter plan at 500 concurrent users against the `staging` Gateway for a sustained 15-minute window; profile the common-read endpoints listed in `caslearn-api-contracts`
  - Fail the build if Gateway p95 exceeds 300 ms OR if Web UI p95 exceeds 2 s, publishing the HTML report as a Jenkins artifact
  - Emit `jmeter_p95_seconds{endpoint}` metric into Prometheus and create a Grafana panel on the `Admin Overview` dashboard
  - _Requirements: Req 26.1, Req 26.2, Req 26.6, Req 29.5_
  - _Design: Sec 8, Sec 15_

  - [ ] 77.1 Add the JMeter plan files to `caslearn-jenkins-build/jmeter/` — one JMX per persona journey plus a common-read mix
    - _Requirements: Req 26.2_
  - [ ] 77.2* Run a dry-run against `dev` and capture baseline numbers in `caslearn-docs/perf/baseline.md`
    - _Requirements: Req 26.2, Req 26.6_

- [ ] 78. Apply HPA tuning and capacity verification
  - Set replica `min/max` on every service per the capacity table in design §18.1
  - Configure HPA on CPU 70% plus a request-rate custom metric via `prometheus-adapter`
  - Run the nightly JMeter job at 500 concurrent and confirm HPA scales within budget; capture screenshots of Grafana panels for the demo packet
  - _Requirements: Req 26.3, Req 26.4, Req 26.6_
  - _Design: Sec 18.1_

- [ ] 79. Tune HikariCP pools and MySQL `max_connections`
  - Apply the per-service pool sizes from design §18.2 to each `application.yml`
  - Set `max_connections=300` on the MySQL instances via `caslearn-platform-tf/modules/mysql`
  - Add a Grafana panel per service showing pool saturation (`hikaricp_connections_active / hikaricp_connections_max`)
  - _Requirements: Req 26.3, Req 26.4_
  - _Design: Sec 18.2_

- [ ] 80. Finalize Redis cache policy + invalidation
  - Set `maxmemory-policy=allkeys-lru` on ElastiCache (AWS) and the in-cluster Redis (on-prem) via their respective Terraform/Helm values
  - Implement cache key invalidation on role change (Auth Service publishes a Redis pub/sub message on `role:{userId}`)
  - Implement cache invalidation on enroll/drop (Course Service publishes on `enroll:{studentId}:{courseId}`)
  - _Requirements: Req 26.3, Req 26.4, Req 6.3_
  - _Design: Sec 18.3_

- [ ] 81. Run a full OWASP Top 10 security review against the `staging` environment
  - Create `caslearn-docs/security-review-mvp.md` walking each OWASP category from `steering/security.md` with evidence (test name, log excerpt, or screenshot) and a PASS / FAIL / MITIGATED column
  - Open tickets for any FAIL in a P1 issue list; block MVP release on P1s
  - _Requirements: Req 5, Req 27, every platform-wide requirement under §A_
  - _Design: Sec 22 (risk table)_

- [ ] 82. Penetration-style smoke tests against each auth / authz path
  - Add `caslearn-auth-service/src/test/java/.../security/PenTestSuite.java` exercising: credential stuffing, JWT tampering variants (alg none, kid swap, expired-but-not-yet-revoked, cross-service token replay), forged `role` field in every known request shape, direct object references (`/grades/{otherStudentId}`), SSRF via AI Service prompt, mass enumeration on login response shape
  - Each test asserts the expected 401/403/404 and the matching audit event
  - _Requirements: Req 1.2, Req 1.4, Req 1.6, Req 2.4, Req 2.5, Req 2.6, Req 4.5, Req 15.6_
  - _Design: Sec 12_
  - _Property: CP-3, CP-11_

- [ ] 83. Implement the FERPA student-data-export endpoint
  - `POST /v1/users/me/data-export` in `caslearn-auth-service` — enqueues a background job
  - Background worker (Spring `@Scheduled` + outbox pattern) that assembles the student's profile, enrollments (Course Service), submissions (Assessment Service), grades (Assessment Service), messages (Messaging Service), AI history (AI Service) into an NDJSON archive
  - Store the archive in object storage; return a signed download link valid for 72 hours
  - Ensure the archive is produced within 72 h (Req 27.2) — assert in an integration test by fast-forwarding the clock
  - Emit `FERPA_EXPORT_REQUESTED` and `FERPA_EXPORT_READY` audit events
  - _Requirements: Req 27.2_
  - _Design: Sec 14.4_

  - [ ] 83.1 Admin UI screen: list pending / ready exports, resend link
    - _Requirements: Req 27.2_
  - [ ] 83.2* Integration test: full round-trip from request → archive → re-import validation (schema-check only, not re-load)
    - _Requirements: Req 27.2_

- [ ] 84. Implement the FERPA student-data-deletion flow
  - `DELETE /v1/users/{id}/data` admin-only endpoint in `caslearn-auth-service`
  - Orchestrator emits per-service `StudentDataDeletionRequested` events; each service anonymizes PII on the student's rows (replace name/email with `deleted-<uuid>`, null out profile fields) while preserving aggregate analytics rows and audit log entries
  - Audit log entries keep the original actor_id since removing them would break the hash chain — document this in `caslearn-docs/runbooks/ferpa-deletion.md`
  - Emit `FERPA_DELETION_REQUESTED`, `FERPA_DELETION_COMPLETED{service}` audit events
  - _Requirements: Req 27.3, Req 3.2, Req 3.3_
  - _Design: Sec 14.1, Sec 22_

  - [ ] 84.1 Integration test: seed a student with data in every service, run deletion, verify PII is gone and aggregates remain intact
    - _Requirements: Req 27.3_

- [ ] 85. Run backup + restore dry-runs and time them against RPO / RTO
  - MySQL: take a Percona / RDS snapshot, restore into a scratch namespace, run a read smoke-test against the restored data
  - MinIO: use `mc mirror` to a scratch bucket; restore via `mc mirror` reverse; verify object checksums
  - Qdrant: snapshot collections; restore into a scratch StatefulSet; verify vector count + a sample similarity query
  - Record timings in `caslearn-docs/runbooks/backup-restore.md`; confirm RPO ≤ 24 h and RTO ≤ 4 h per OQ-9
  - _Requirements: Req 23, Req 27.2_
  - _Design: Sec 6, Sec 7, Sec 23 (OQ-9)_

- [ ] 86. Chaos day — kill one pod per service and verify resilience invariants
  - Using `kubectl delete pod`, kill one replica of each service in `staging`
  - Assert: Istio load-balances to surviving replicas within 5 s; circuit breakers do not open falsely; no 5xx spike in Grafana beyond 30 s
  - Kill the Auth Service replica hosting the current RBAC cache leader; assert the calling service fails closed with 503 `RBAC_UNAVAILABLE` (not allow-by-default) and emits `rbac_unavailable_total`
  - Kill all Messaging Service replicas; send an announcement; assert the notification outbox retains the event and drains on recovery with idempotency upheld
  - Document findings in `caslearn-docs/runbooks/chaos-day-mvp.md`
  - _Requirements: Req 13.4, Req 19.5 (resilience surface), Req 24.3, Req 25.6_
  - _Design: Sec 19.1, Sec 19.3, Sec 19.6_
  - _Property: CP-9_

- [ ] 87. Schedule the nightly audit-chain verifier per service
  - Add a Kubernetes `CronJob` per service (one nightly run each) in `caslearn-platform-helm`; each runs `java -jar audit-verifier.jar --service=<name>` using the shared `caslearn-audit` library
  - On chain break, exit non-zero and increment `audit_chain_broken_total{service}`; Alertmanager fires `AuditChainBroken`
  - Publish the runbook `caslearn-docs/runbooks/audit-chain-break.md` with the exact investigation and remediation steps
  - _Requirements: Req 3.3, Req 3.4_
  - _Design: Sec 14.3_
  - _Property: CP-5_

- [ ] 88. Wire all hardening alerts to the on-call channel
  - Create Alertmanager receivers for the Grafana OnCall integration (per OQ-10)
  - Route the following alerts to the `caslearn-oncall` channel with severity labels: `HighErrorRate` (page), `HighLatencyP95` (page), `AuditChainBroken` (page), `TraceGapDetected` (warn), `AnalyticsAuthzUnavailable` (page), `AiBudgetBurn` (page), `JmeterP95Breach` (warn)
  - Add a silence window for planned maintenance
  - _Requirements: Req 25.6, Req 25.7_
  - _Design: Sec 15.5, Sec 23 (OQ-10)_

- [ ] 89. Run the full axe-core WCAG 2.1 AA audit across every persona journey
  - Add `@axe-core/playwright` to each Playwright E2E and assert zero serious or critical violations per flow
  - Fix every violation surfaced; track remaining deferred items in `caslearn-docs/a11y-findings-mvp.md` with severity + owner
  - Confirm `Jenkins` blocks the MVP release build on any failure (Req 30.2)
  - _Requirements: Req 26.5, Req 30.2_
  - _Design: Sec 20.5_

  - [ ] 89.1 Admin persona journey (login → users → audit log browser → overview dashboard)
    - _Requirements: Req 26.5_
  - [ ] 89.2 Instructor persona journey (login → my courses → author content → create quiz → grade submission → analytics)
    - _Requirements: Req 26.5_
  - [ ] 89.3 Student persona journey (login → enroll → read content → submit assignment → take quiz → view grade → ask assistant → view study plan)
    - _Requirements: Req 26.5_
  - [ ] 89.4 Mobile-viewport runs of all three journeys in Playwright
    - _Requirements: Req 30.2_

- [ ] 90. Run the final OpenAPI contract-conformance sweep
  - In a dedicated Jenkins job `cross-service-contract-sweep`, spin up every `*-service` in staging and assert each one's live responses match the schemas in `caslearn-api-contracts`
  - Fail the job (and block MVP release) if any endpoint drifts from its contract
  - Publish the report as a build artifact
  - _Requirements: Req 28 (docs), Req 29.3_
  - _Design: Sec 10, Sec 21_
  - _Property: CP-12_

- [ ] 91. Run the final Helm portability test
  - Execute `test/helm/portability_test.sh` from `caslearn-platform-helm` — render each chart with `values.onprem.yaml` and `values.aws.yaml`, diff the set of Kubernetes `kind:` resources, assert equality
  - Fail the MVP release if any chart diverges
  - _Requirements: Req 23.4_
  - _Design: Sec 21_
  - _Property: CP-10_

- [ ] 92. Polish Grafana dashboards per service and finalize `Admin Overview`
  - Ensure every service has a standard RED dashboard (rate, errors p50/p95/p99) plus domain panels (audit metrics, JVM, Hikari, Kafka-like lag n/a MVP)
  - Finalize `Admin Overview` dashboard with the panels enumerated in design §15.4: per-service request rate, error rate, p95 latency, CPU, memory, active user count, audit metrics row, RBAC failure row, AI budget row
  - Verify every dashboard renders its default 15-minute window in under 5 s (Req 8.3)
  - Commit the dashboard JSON into `caslearn-platform-helm/charts/observability/dashboards/`
  - _Requirements: Req 8.2, Req 8.3, Req 25.5_
  - _Design: Sec 15.4_

- [ ] 93. Write the MVP runbook set
  Each runbook lives in `caslearn-docs/runbooks/` and is linked from the service README (Req 28.1). Each must end with an "exit criteria" checklist an on-call engineer can tick.

  - [ ] 93.1 `service-bringup.md` — cold start order for the whole platform
    - _Requirements: Req 20, Req 28_
  - [ ] 93.2 `rollback.md` — per-service and platform-wide rollback via ArgoCD
    - _Requirements: Req 22.3_
  - [ ] 93.3 `scaling.md` — when and how to scale a service horizontally; DB and Redis constraints
    - _Requirements: Req 26.3, Req 26.4_
  - [ ] 93.4 `db-restore.md` — restore from Percona / RDS snapshot; MinIO and Qdrant analogues
    - _Requirements: Req 23_
  - [ ] 93.5 `jwt-key-rotation.md` — rotating the Auth Service signing key with the 24 h overlap window
    - _Requirements: Req 5.3_
  - [ ] 93.6 `alert-response.md` — playbook per alert (`HighErrorRate`, `HighLatencyP95`, `AuditChainBroken`, `TraceGapDetected`, `AnalyticsAuthzUnavailable`, `AiBudgetBurn`, `JmeterP95Breach`)
    - _Requirements: Req 25.6, Req 25.7_
  - [ ] 93.7 `audit-chain-break.md` — investigation + remediation for `AuditChainBroken`
    - _Requirements: Req 3.3_
  - [ ] 93.8 `ferpa-export.md` and `ferpa-deletion.md` — operating the Req 27.2 / 27.3 flows
    - _Requirements: Req 27.2, Req 27.3_
  - [ ] 93.9 `istio-troubleshooting.md` — common mesh issues (AuthorizationPolicy deny, PeerAuthentication misconfig, egress blocked)
    - _Requirements: Req 5.1, Req 24_
  - [ ] 93.10 `onboarding.md` update — final pass after Sprints 0–5 so a new hire can ship a merged change in week 1
    - _Requirements: Req 28.5_

- [ ] 94. Finalize every repo README to the 14-section standard
  - For each of the 15 MVP repos, verify the README contains all 14 sections from `steering/documentation.md` — Purpose, Tech Stack, Architecture at a Glance, Local Setup, Build & Test, Run Locally, Deployment, Configuration, Observability, API / Contract, Contributing, Owners, License, and the repo-naming-aware header
  - The CI README-validator stage (installed in Sprint 0) must pass on every repo; fix any section that's still a `TODO` placeholder from the scaffold
  - _Requirements: Req 19.5, Req 28.1, Req 28.2_
  - _Design: Sec 17, steering/documentation.md_

- [ ] 95. Render and commit all architecture diagrams from `caslearn-docs/diagrams/`
  - Sources live in `caslearn-docs/diagrams/` as Mermaid / Structurizr / PlantUML
  - CI renders them to SVG + PNG on every push and publishes the static site
  - Verify the required diagram list from `steering/documentation.md` is complete: C4 Context, C4 Container, C4 Component per service, service dependency, network & security, deployment on-prem, deployment AWS EKS, CI/CD pipeline, data flow, composite ERD
  - _Requirements: Req 28.3_
  - _Design: Sec 2, Sec 3, Sec 4, Sec 5, Sec 6, Sec 7, Sec 8, Sec 9_

- [ ] 96. Deploy a Redocly static site for `caslearn-api-contracts`
  - Render every OpenAPI spec to Redocly HTML
  - Publish to a GitHub Pages site or the on-prem docs host; link from `caslearn-docs/index.md`
  - Confirm the generator runs on every merge to `caslearn-api-contracts`
  - _Requirements: Req 28.3, Req 22_
  - _Design: Sec 10_

- [ ] 97. Regenerate the TypeScript clients in `caslearn-web-ui` from the final contracts
  - Run `openapi-typescript-codegen` against the final `caslearn-api-contracts` revision
  - Commit the regenerated clients; fix any lint / type errors that surface from API drift
  - _Requirements: Req 28.3_
  - _Design: Sec 10_

- [ ] 98. Verify end-to-end persona walkthroughs one final time
  Smoke-test from a fresh seeded demo data set that every persona can complete its golden path without errors. Capture screenshots for the demo artifact packet.

  - [ ] 98.1 Admin: log in → create an instructor + three students → create a course → publish it → view the `Admin Overview` dashboard
    - _Requirements: Req 6, Req 7, Req 8_
  - [ ] 98.2 Instructor: log in → author a module → upload content → create a quiz → create an assignment with rubric → view analytics
    - _Requirements: Req 9, Req 10, Req 13_
  - [ ] 98.3 Student: log in → enroll → read content → submit an assignment → take a quiz → view grade → ask the AI assistant → view study plan
    - _Requirements: Req 14, Req 15, Req 17, Req 18_

- [ ] 99. Confirm the MVP release build passes every blocking gate
  - Verify in one Jenkins release build that **all** of the following block merge / release: repo-naming suffix check, unit + integration tests ≥ 70% coverage, property-based tests for CP-1 through CP-9 + CP-11, OpenAPI conformance (CP-12), Helm portability (CP-10), JMeter p95 targets, axe-core WCAG 2.1 AA, mobile viewport Playwright, SonarQube, Trivy (fs + image), Gitleaks, license scan, Day-0 smoke test
  - Fail-fast matrix documented in `caslearn-docs/release-gates.md`
  - _Requirements: Req 20.6, Req 22, Req 26, Req 29, Req 30.2_
  - _Design: Sec 8, Sec 21_

- [ ] 100. Tag the MVP release and freeze scope
  - Tag `v1.0.0-mvp` in every MVP repo; ArgoCD promotion PR to `prod` (via `caslearn-platform-argo`) gated on a successful run of task 99
  - Append the release notes to `caslearn-docs/CHANGELOG.md` summarizing delivered requirements by number + deferred items referenced in `caslearn-docs/post-mvp-backlog.md`
  - Lock `main` branches across all repos to `release-override` only; feature work paused for 48 h of post-release monitoring per `delivery.md`
  - _Requirements: Req 30.3_
  - _Design: Sec 22_

- [ ] 101. Sprint 6 checkpoint
  - Confirm every hardening gate has passed in CI, a fresh developer can complete the Day-0 smoke test, the admin / instructor / student golden paths work, the backup-restore drill met RPO/RTO targets, the chaos-day findings are documented, and the release tag is applied. Ask the user if questions arise.
## Sprint 6 — Hardening, Load, Accessibility, Security, Demo-Ready MVP

Goal per `delivery.md` S6: no new features. Every non-functional requirement that was deferred during feature sprints lands here. The exit gate is the July 31 demo-ready MVP: all persona journeys pass E2E in CI, JMeter sustains p95 targets at 500 concurrent users, WCAG 2.1 AA is clean, OWASP Top 10 review is signed off, backups restore, and runbooks are battle-tested.

- [ ] 77. Full-fidelity JMeter load suite and nightly run
  - [ ] 77.1 Author JMeter `.jmx` plans in `caslearn-jenkins-build/jmeter/`, covering the common-read endpoints listed in `caslearn-api-contracts` (course list, course detail, enrollment check, grade read, AI ask with cache warm)
    - Parameterize concurrent users (50 / 250 / 500), ramp-up 60s, steady-state 10 min, shutdown 60s
    - _Requirements: Req 26.2, Req 26.6, Req 29.5_
    - _Design: Sec 18_
  - [ ] 77.2 Add a nightly Jenkins pipeline `jmeter-nightly` that runs the 500-user profile against the `staging` cluster and publishes the HTML report as a build artifact
    - Build fails if p95 > 300 ms on any common-read endpoint AND report publication succeeded (per REQ-29.5 decision)
    - _Requirements: Req 26.2, Req 26.6, Req 29.5_
  - [ ] 77.3 Add a PR-gate 50-user profile that runs in under 3 minutes on every `*-service` and `caslearn-gateway-service` PR
    - _Requirements: Req 26.2_
  - [ ] 77.4* Integration test: verify that a JMeter breach produces a red build and uploads the report to Artifactory
    - _Requirements: Req 29.5_

- [ ] 78. Accessibility audit and fix pass
  - [ ] 78.1 Run axe-core across every page reachable in each persona journey; fix all violations of severity `serious` or `critical`
    - Cover admin overview, user management, audit log browser, course authoring, grading queue, student course view, quiz taking, AI assistant, study planner
    - _Requirements: Req 26.5, Req 30.2_
    - _Design: Sec 20.5_
  - [ ] 78.2 Add a Playwright mobile-viewport suite (iPhone 14, Pixel 7 presets) covering the three persona-critical flows: login, course open, assignment submit
    - _Requirements: Req 30.2_
  - [ ] 78.3 Wire an `a11y` Jenkins stage that fails the Web UI build if axe reports any serious/critical violation OR if the mobile-viewport Playwright run fails
    - _Requirements: Req 30.2_
  - [ ] 78.4* Manual keyboard-only walkthrough of all three persona journeys; capture findings in `caslearn-docs/a11y-walkthrough.md`
    - _Requirements: Req 26.5_

- [ ] 79. OWASP Top 10 security review
  - Walk the OWASP Top 10 coverage table from `security.md` against the running system; record findings and remediations in `caslearn-docs/security-review-s6.md`
  - [ ] 79.1 A01 Broken Access Control — re-run CP-1, CP-2, CP-3, CP-11 PBT suites with expanded seeds; add any newly-found counterexamples as regression tests
    - _Property: CP-1, CP-2, CP-3, CP-11_
    - _Requirements: Req 2, Req 14, Req 15.5, Req 15.6, Req 17.4_
  - [ ] 79.2 A02 Cryptographic Failures — verify TLS 1.2/1.3 only at ingress (Req 5.2); verify bcrypt cost ≥ 12 in prod config; verify MySQL and object-storage encryption-at-rest is on
    - _Requirements: Req 5.2, Req 5.4, Req 1.7_
  - [ ] 79.3 A03 Injection — add a synthetic fuzz suite against every public endpoint pushing SQL/shell/script payloads; verify HTTP 400 + `INPUT_VALIDATION_REJECTED` audit event; verify no payload reaches the DB
    - _Requirements: Req 4.5_
  - [ ] 79.4 A04 Insecure Design — add a per-service `threat-model.md` in each `*-service` repo (`docs/threat-model.md`) using STRIDE
    - _Requirements: Req 28.1_
    - _Design: Sec 22_
  - [ ] 79.5 A05 Security Misconfiguration — scan all Helm values for default credentials, debug flags enabled, wide-open NetworkPolicies; promote any findings to fixes in `caslearn-platform-helm`
    - _Requirements: Req 22.1, Req 24.1, Req 24.2_
  - [ ] 79.6 A06 Vulnerable Components — run `trivy fs` + `trivy image` at severity `HIGH,CRITICAL` on every image in Artifactory; fix or pin-with-waiver any finding; Dependabot alerts triaged
    - _Requirements: Req 5.5_
  - [ ] 79.7 A07 Authentication Failures — verify lockout works at 5 attempts, refresh rotation invalidates previous token, deactivation flushes tokens within 5 minutes
    - _Requirements: Req 1.6, Req 1.5, Req 6.2_
  - [ ] 79.8 A08 Software & Data Integrity — verify branch protection is enforced on all 15 repos (gh API check), CI has the security stages in the required order
    - _Requirements: Req 19.3, Req 22.1, Req 22.4_
  - [ ] 79.9 A09 Logging & Monitoring — verify every service emits structured JSON logs with all mandatory fields, traces span end-to-end in Tempo, alerts route to the configured on-call channel
    - _Requirements: Req 25.3, Req 25.4, Req 25.6, Req 25.7_
  - [ ] 79.10 A10 SSRF — verify Istio egress allow-list blocks any service other than AI Service from reaching the LLM host and any service other than Messaging from reaching SMTP
    - _Requirements: Req 24.2_
    - _Design: Sec 5_
  - [ ] 79.11* Penetration-style smoke: attempt role escalation via every known vector (body, header, query, JWT tamper), grade read as wrong student, cross-course enrollment bypass, AI cross-course material leak, audit-log tamper; file issues for each finding
    - _Requirements: Req 2.5, Req 2.6, Req 14.3, Req 15.6, Req 17.4, Req 3.2, Req 3.3_
    - _Property: CP-1, CP-2, CP-3, CP-5, CP-6, CP-11_

- [ ] 80. FERPA data-export and deletion flows
  - [ ] 80.1 Implement `POST /v1/users/me/data-export` in `caslearn-auth-service` — produces an NDJSON archive containing profile, enrollments, submissions, grades, messages, AI conversation history the student owns; signs the archive with the export signing key
    - Cross-service aggregation happens via internal mTLS calls: Auth orchestrates, Course / Assessment / Messaging / AI return per-user slices
    - Completion within 72h is enforced via an async job + user notification email; the archive is stored in object storage and expires after 30 days
    - _Requirements: Req 27.2_
    - _Design: Sec 14.4_
  - [ ] 80.2 Implement `POST /v1/admin/users/{id}/data-deletion` — anonymizes PII fields across all services; preserves aggregate analytics and audit-log integrity (replaces actor identifiers with tombstone `DELETED_USER:<hash>`, not NULL)
    - _Requirements: Req 27.3_
    - _Design: Sec 14_
  - [ ] 80.3 Verify audit-log chain remains intact after deletion: the nightly verifier must continue to pass; audit entries referencing the deleted user still hash-chain correctly using the tombstone id
    - _Requirements: Req 3.2, Req 3.3, Req 27.3, Req 27.4_
    - _Property: CP-5_
  - [ ] 80.4 Web UI: student-facing "Export my data" button in profile; admin-facing deletion flow with two-step confirmation and reason field
    - _Requirements: Req 27.2, Req 27.3_
  - [ ] 80.5* Integration tests in `caslearn-auth-service` exercising an end-to-end export for a seeded student who has submissions, grades, messages, AI conversations across 3 courses
    - _Requirements: Req 27.2_
  - [ ] 80.6* Integration test that simulates a deletion and verifies CP-5 audit chain still verifies green on the nightly job
    - _Requirements: Req 27.3, Req 27.4_
    - _Property: CP-5_

- [ ] 81. Backup and restore dry-run
  - [ ] 81.1 Execute a Velero + restic backup of the `staging` cluster (MySQL PVs via operator snapshots, MinIO buckets, Qdrant collections) to the off-site S3-compatible target
    - Capture backup size, duration, and RPO; record in `caslearn-docs/runbooks/backup-restore-dryrun.md`
    - _Design: Sec 6_
    - _Requirements: Req 27 (FERPA data retention context)_
  - [ ] 81.2 Restore the backup into a fresh namespace on the same cluster; run the full Day-0 smoke test against the restored data plane
    - Verify seeded users can log in, seeded course content reads, a seeded grade is still visible with the right student
    - _Requirements: Req 20_
  - [ ] 81.3 Document the RTO achieved vs the 4h target; file follow-ups if over
    - _Design: Sec 23 (OQ-9)_
  - [ ] 81.4* Integration test: a synthetic backup-restore job runs weekly in the `dev` environment and fails the build on RPO drift
    - _Requirements: Req 29.5 (nightly enforcement pattern)_

- [ ] 82. Chaos day — resilience and fail-closed verification
  - [ ] 82.1 Kill one pod per service via `kubectl delete pod` while the PR-gate JMeter profile is running; verify zero request failures at the UI (Resilience4j + Istio handle the restart)
    - _Requirements: Req 26.4_
    - _Design: Sec 19.1_
  - [ ] 82.2 Kill `caslearn-auth-service` entirely for 90 seconds; verify that `caslearn-analytics-service` returns 503 with `analytics_authz_unavailable_total` incrementing, and that `rbac_unavailable_total` increments on every other service — not 200 with empty data
    - _Requirements: Req 13.4_
    - _Design: Sec 19.6_
  - [ ] 82.3 Kill the Qdrant StatefulSet; verify `caslearn-ai-service` circuit-breaks the LLM call and returns a standard error envelope with `code: AI_UNAVAILABLE`, not a 500 with a stack trace
    - _Requirements: Req 17.5, Req 19.5_
    - _Design: Sec 19.1, Sec 19.5_
  - [ ] 82.4 Kill the MySQL primary for the `course` database; verify Percona operator fails over, course reads recover within 30s, and enrollment writes retry correctly at the application layer
    - _Requirements: Req 26.3_
  - [ ] 82.5 Simulate a Cosmos-ray audit-log hash-chain break by manually UPDATE-ing a past row in a staging copy; verify the nightly verifier raises `AuditChainBroken`
    - _Requirements: Req 3.3, Req 25.6_
    - _Property: CP-5_
  - [ ] 82.6* Record chaos-day findings in `caslearn-docs/runbooks/chaos-day-s6.md`; any behaviour that wasn't covered by an existing runbook gets a new one in the same PR

- [ ] 83. Audit-chain verifier job and alert wiring
  - [ ] 83.1 Deploy a `CronJob` per service (`audit-verifier-<service>`) running at 03:00 cluster-local time that walks `audit_log` and verifies every `entry_hash`
    - Emits `audit_chain_broken_total` on failure; entry hash mismatch logs ERROR with row id
    - _Requirements: Req 3.3_
    - _Design: Sec 14.3_
    - _Property: CP-5_
  - [ ] 83.2 Alertmanager rule `AuditChainBroken` fires on `increase(audit_chain_broken_total[15m]) > 0` to the on-call channel
    - _Requirements: Req 3.3, Req 25.6_
  - [ ] 83.3 Runbook `caslearn-docs/runbooks/audit-chain-break.md` — isolation steps, forensic capture, re-seal procedure
    - _Requirements: Req 28.3_

- [ ] 84. Finalize Alertmanager rules and routing
  - [ ] 84.1 Wire all required alerts to the on-call channel: `HighErrorRate`, `HighLatencyP95`, `AuditChainBroken`, `TraceGapDetected`, `AnalyticsAuthzUnavailable`, `AiBudgetBurn`, `RbacUnavailable`
    - Separate notification policies so `HighErrorRate` and `HighLatencyP95` can fire concurrently without deduping each other
    - _Requirements: Req 25.6, Req 25.7_
    - _Design: Sec 15.5_
  - [ ] 84.2 Fire-drill each alert at least once in `staging` to verify routing, severity labels, and runbook links resolve
    - _Requirements: Req 25.6, Req 25.7_
  - [ ] 84.3* Add a `synthetic-probe` blackbox exporter hitting `caslearn-web-ui` root and `caslearn-gateway-service /actuator/health` every 30s from outside the cluster; fuel availability SLO reporting (Req 26.3)
    - _Requirements: Req 26.3_

- [ ] 85. Finalize Grafana dashboards
  - [ ] 85.1 Polish every per-service dashboard to include RED metrics (request rate, error rate, p50/p95/p99 latency), DB pool usage, heap, GC, plus the service-specific domain metrics called out in design Sec 15.1
    - _Requirements: Req 25.5_
    - _Design: Sec 15.4_
  - [ ] 85.2 Finalize the `Admin Overview` dashboard with per-service request rate, error rate, p95 latency, CPU, memory, active user count (5-min unique `user_id` from Gateway), audit-metric row, and RBAC-failure row
    - _Requirements: Req 8.2, Req 8.3_
    - _Design: Sec 15.4_
  - [ ] 85.3 Lock every dashboard's default window at 15 minutes; enforce panel query cost so the dashboard loads within 5 seconds (use Prometheus recording rules where needed)
    - _Requirements: Req 8.3_
  - [ ] 85.4 Provision dashboards as code in `caslearn-platform-helm/charts/observability/templates/dashboards/` so they are reproducible and version-controlled
    - _Requirements: Req 28.3_
    - _Design: Sec 15.4_

- [ ] 86. Author production runbooks
  - All runbooks live in `caslearn-docs/runbooks/` as listed in `documentation.md`; verify each one end-to-end during S6, not just that it exists
  - [ ] 86.1 `service-bringup.md` — bring a single service from zero in `dev`
  - [ ] 86.2 `rollback.md` — ArgoCD rollback to previous sync revision, both per-service and platform-wide
  - [ ] 86.3 `scaling.md` — adjust HPA, HikariCP pool sizing, MySQL max_connections when growing beyond MVP targets
  - [ ] 86.4 `db-restore.md` — restore MySQL from Velero snapshot
  - [ ] 86.5 `jwt-key-rotation.md` — rotate JWT signing key with 24h overlap window
  - [ ] 86.6 `alert-response-higherrorrate.md`, `alert-response-highlatencyp95.md`, `alert-response-auditchainbreak.md`, `alert-response-rbacunavailable.md`, `alert-response-aibudgetburn.md`
  - [ ] 86.7 `promotion-dev-staging-prod.md` — ArgoCD promotion sequence and gate criteria from `delivery.md`
  - [ ] 86.8 `ferpa-export-deletion.md` — handle an export or deletion request from intake through completion
  - [ ] 86.9 `istio-troubleshooting.md` — sidecar injection, authz policy diagnosis, mTLS handshake failures
  - [ ] 86.10 `sealed-secrets-key-restore.md` — restore the bootstrap key after a disaster
  - [ ] 86.11* Each runbook links back to the alert or event that triggers it, and each alert's Alertmanager annotation includes a `runbook_url` pointing at the correct file
    - _Requirements: Req 28.3_

- [ ] 87. Final contract test sweep (CP-12)
  - [ ] 87.1 Run `OpenApiConformanceTest` in every `*-service` against the current `caslearn-api-contracts` spec
    - Verify zero drift: response schemas match, error envelopes match, required fields present
    - _Requirements: Req 29.1_
    - _Property: CP-12_
  - [ ] 87.2 Add a cross-repo nightly job `contracts-conformance-nightly` that spins up the full platform in a dedicated namespace and hits every endpoint with representative requests from each spec
    - Build fails on any schema mismatch
    - _Requirements: Req 28.2, Req 29.1_
    - _Property: CP-12_

- [ ] 88. Final Helm portability test (CP-10)
  - [ ] 88.1 Run `helm template` with `values.onprem.yaml` and `values.aws.yaml` for every chart; diff the set of `kind:` resources and resource counts; assert equality (only storage + ingress annotations may differ)
    - _Requirements: Req 23.4_
    - _Property: CP-10_
  - [ ] 88.2 Wire the check as a required CI stage in the `caslearn-platform-helm` pipeline; block merge on failure
    - _Requirements: Req 23.4_
    - _Property: CP-10_
  - [ ] 88.3 Bring up a fresh on-prem kind cluster and a fresh EKS sandbox (Terraform apply in `caslearn-env-tf/envs/prod-aws`) from the same Helm chart revision and same image digests; run the Day-0 smoke suite against both
    - Capture side-by-side Grafana screenshots in `caslearn-docs/portability-proof.md` as the portability evidence for stakeholders
    - _Requirements: Req 23.1, Req 23.4_
    - _Design: Sec 6, Sec 7_

- [ ] 89. Final README coverage sweep
  - [ ] 89.1 Verify every one of the 15 MVP repos has a complete `README.md` with all 14 sections from `documentation.md` (no `TODO` placeholders remaining)
    - _Requirements: Req 28.1_
  - [ ] 89.2 Verify every `*-service` README's `Observability` section lists the correct Grafana dashboard name and the exact metric names emitted (cross-check with design Sec 15.1)
    - _Requirements: Req 28.1, Req 25.5_
  - [ ] 89.3 Verify every README's `API / Contract` section links to the live Redocly-rendered OpenAPI URL in the published `caslearn-docs` site
    - _Requirements: Req 28.1_
  - [ ] 89.4 Add a Jenkins stage `readme-completeness-check` that fails the build if any required section heading is missing from a modified repo's README
    - _Requirements: Req 28.2_

- [ ] 90. Render all architecture diagrams as code and publish
  - [ ] 90.1 Author Mermaid/PlantUML source for C4 Context, C4 Container, C4 Component per service, Service Dependency, Network & Security, Deployment Topology (On-Prem + AWS EKS), CI/CD Pipeline, and Data Flow; source lives in `caslearn-docs/diagrams/`
    - _Requirements: Req 28.3_
    - _Design: Sec 2, Sec 3, Sec 4, Sec 5, Sec 6, Sec 7, Sec 8_
  - [ ] 90.2 Add a GitHub Action that renders all diagrams to SVG/PNG on push and publishes them as part of the Redocly `caslearn-docs` static site
    - _Requirements: Req 28.3_
  - [ ] 90.3 Compose-ERD per service (`caslearn-docs/diagrams/erd/<service>.mmd`) plus a cross-service overview ERD (logical foreign keys only, no joins) — reuse the mermaid from design Sec 9
    - _Requirements: Req 28.3_
    - _Design: Sec 9_

- [ ] 91. Onboarding rehearsal with a fresh developer
  - [ ] 91.1 Recruit one team member who has never run the platform; time them from fork-to-first-merged-PR following `caslearn-docs/onboarding.md`
    - Log every friction point; file a PR to `caslearn-docs` fixing each one
    - _Requirements: Req 28.4, Req 28.5_
  - [ ] 91.2 Target: under one working day (optional but the DoR for S6 exit)
    - _Requirements: Req 28.5_

- [ ] 92. MVP release candidate build and promotion to staging
  - [ ] 92.1 Cut a semver `v1.0.0-rc1` tag in each `*-service` and `*-ui` repo; Jenkins publishes images to Artifactory with the matching tag; the bump-PR in `caslearn-platform-argo` points at `v1.0.0-rc1` across every service
    - _Requirements: Req 22.2, Req 22.3_
  - [ ] 92.2 ArgoCD syncs `staging` to RC1; run the full Day-0 smoke test + Playwright persona E2E + the PR-gate JMeter profile
    - Build is red if anything fails
    - _Requirements: Req 20, Req 22.5_
  - [ ] 92.3 Soak RC1 in `staging` for 24 hours with no P1/P2 alerts firing (per the `delivery.md` promotion gate)
    - _Design: `delivery.md` Release Strategy section_
  - [ ] 92.4 Promote RC1 to `prod` via the documented promotion runbook; verify ArgoCD sync; run the synthetic probe set against prod; confirm Grafana Admin Overview is green
    - _Requirements: Req 22.5, Req 26.3_

- [ ] 93. Release build block-on-accessibility gate
  - [ ] 93.1 Configure the MVP release-build job to fail if the Web UI fails the WCAG 2.1 AA axe suite OR the mobile-viewport Playwright suite
    - _Requirements: Req 30.2_
  - [ ] 93.2 Exercise the block: intentionally introduce an a11y violation on a branch; verify the release build goes red and merge is blocked; revert
    - _Requirements: Req 30.2_

- [ ] 94. Post-MVP backlog capture
  - [ ] 94.1 Ensure `caslearn-docs/post-mvp-backlog.md` contains every out-of-scope ask encountered during S0–S6 (SSO/SAML, SCORM/LTI, live video, payments, mobile native, ambient Istio, full RUM, container image signing, multi-tenant partitioning, proctoring, plagiarism detection)
    - _Requirements: Req 30.3_
  - [ ] 94.2 Tag each backlog item with its estimated cost (S/M/L), blast radius, and which requirement it would extend
    - _Requirements: Req 30.3_

- [ ] 95. S6 exit checkpoint — MVP demo-ready
  - All of the following must be true before declaring MVP ready for the July 31 demo:
    - [ ] Day-0 smoke test green in CI for both `caslearn-platform-helm` and `caslearn-docs`
    - [ ] All 12 correctness-property tests (CP-1..CP-12) pass nightly
    - [ ] JMeter 500-user nightly passes p95 < 300 ms on common-read endpoints
    - [ ] axe WCAG 2.1 AA clean across all three persona journeys, desktop and mobile
    - [ ] OWASP Top 10 review signed off in `caslearn-docs/security-review-s6.md`
    - [ ] FERPA export + deletion flows work end-to-end in `staging`
    - [ ] Backup + restore dry-run in `staging` within RTO 4h / RPO 24h targets
    - [ ] Chaos day findings all either fixed or in `post-mvp-backlog.md`
    - [ ] All alerts fire-drilled and routed to the on-call channel
    - [ ] All 15 repos have complete 14-section READMEs
    - [ ] `v1.0.0-rc1` promoted to `prod` with synthetic probes green
    - [ ] Onboarding rehearsal under one working day
  - Ask the user if questions arise or if any gate needs an exception with an ADR.
