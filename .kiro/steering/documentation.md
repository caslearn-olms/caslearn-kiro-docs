# Documentation Standards

Docs are a first-class deliverable. A repo without a valid README fails CI.

## Every Repo Ships a README.md

**Required sections (exact headings, in order):**

1. `# <Repository Name>` — one-line purpose.
2. `## Purpose` — why this repo exists; what problem it solves.
3. `## Tech Stack` — languages, frameworks, major libraries, runtimes, with versions.
4. `## Architecture at a Glance` — a short diagram or paragraph on how this repo fits into CasLearn OLMS. Link to `caslearn-docs` for the full picture.
5. `## Local Setup` — prerequisites, clone commands, dependency install, environment variables (with example), how to bring up local dependencies (docker-compose or kind).
6. `## Build & Test` — exact commands for compile, unit tests, integration tests, lint, format.
7. `## Run Locally` — exact commands to start the service/app locally.
8. `## Deployment` — how this repo reaches an environment (Jenkins pipeline → Artifactory → ArgoCD). Link to the Helm chart.
9. `## Configuration` — environment variables, feature flags, secrets referenced.
10. `## Observability` — which Grafana dashboard to open, which metric names are emitted, which log fields matter.
11. `## API / Contract` — link to the OpenAPI spec in `caslearn-api-contracts` (for services) or to component docs (for UI).
12. `## Contributing` — branching, commit format (Conventional Commits), PR requirements.
13. `## Owners` — CODEOWNERS group + on-call channel.
14. `## License` — link to LICENSE file.

## Repo Template

A GitHub repo template pre-populates:
- `README.md` scaffold with all 14 sections as `TODO` placeholders
- `CODEOWNERS`
- `LICENSE` (Apache 2.0 unless otherwise noted)
- `.gitignore`
- `.editorconfig`
- Language-appropriate build config (`pom.xml`, `package.json`, `Chart.yaml`, etc.)
- `Jenkinsfile` with standard pipeline stages
- `Dockerfile` (if applicable)
- `.github/pull_request_template.md`
- `.github/workflows/` for GitHub-native checks (lint, secret scan)

`caslearn-docs/scripts/new-repo.sh` creates a new repo from the template, sets the naming suffix, applies branch protection, and adds the default CODEOWNERS team.

## CI Enforcement

The CI pipeline fails if:
- The repo lacks a `README.md` with all required section headings.
- A PR adds a new public API endpoint, environment variable, or feature flag without a corresponding `README.md` update.
- A PR adds a new top-level module/package without a local `README.md` describing it.

## Architecture Docs (in `caslearn-docs`)

Diagrams live as code (Structurizr, PlantUML, or Mermaid) in `caslearn-docs/diagrams/`. CI renders them to SVG/PNG on push.

Required diagrams for MVP:
- **C4 Context** — CasLearn OLMS + external actors (users, LLM API, email provider).
- **C4 Container** — all services, UI, DBs, queue (if any), vector store.
- **C4 Component** (per service) — internal structure of each `*-service`.
- **Service Dependency** — who calls whom; direction; sync vs async.
- **Network & Security** — VLANs/namespaces, Istio mesh, ingress, egress, WAF, mTLS boundaries.
- **Deployment Topology — On-Prem** — nodes, storage, ingress.
- **Deployment Topology — AWS EKS** — VPC, subnets, ALB, EBS/S3, RDS (if used).
- **CI/CD Pipeline** — GitHub → Jenkins → Trivy/Sonar/Tests → Artifactory → ArgoCD → cluster.
- **Data Flow** — end-to-end request (student opens course → UI → Gateway → services → DB → response).
- **ERD** — per-service ERDs; a composite view in `caslearn-docs`.

## Runbooks

`caslearn-docs/runbooks/` contains operational procedures:
- Service bring-up
- Rollback (per service and platform-wide)
- Scaling a service
- DB restore from backup
- Rotating JWT signing keys
- Responding to `HighErrorRate` / `HighLatencyP95` alerts
- Investigating an audit-log chain break
- Onboarding a new developer (Day-0 smoke test)
- Promoting `dev` → `staging` → `prod`
- Handling a student data export / deletion request (FERPA)

## ADRs

`caslearn-docs/adr/` uses the Michael Nygard template:
```
# ADR-NNNN: <Title>
Date: YYYY-MM-DD
Status: Proposed | Accepted | Superseded by ADR-XXXX

## Context
## Decision
## Consequences
```

Any deviation from `tech.md` requires an ADR. Examples:
- Choosing Chakra over MUI
- Replacing FAISS with Qdrant
- Swapping Tempo for Jaeger

## Onboarding

`caslearn-docs/onboarding.md` is the single entry point. It gets a new developer from a clean machine to a merged PR in under one week by walking through:
1. Access prerequisites (GitHub org, Slack, VPN if on-prem)
2. Local tool install (JDK 21, Node, pnpm, Docker, kubectl, helm, kind, gh, terraform)
3. Clone all MVP repos
4. Run the Day-0 smoke test
5. Pick a "good-first-issue" ticket
6. Submit a PR

## API Documentation

- **Source of truth:** OpenAPI specs in `caslearn-api-contracts`.
- **Generation:** each service publishes its spec to the contracts repo on merge to `main`.
- **Rendering:** Redocly rendered site deployed as part of `caslearn-docs` static site.
- **Client generation:** TypeScript clients for the UI generated from the contracts repo via `openapi-typescript-codegen`.

## Changelogs

- Each service keeps a `CHANGELOG.md` auto-generated from Conventional Commits.
- `caslearn-docs/CHANGELOG.md` summarizes platform-level releases.
