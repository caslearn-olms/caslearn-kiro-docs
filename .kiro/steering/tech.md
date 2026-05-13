# Technology Stack

This document is the source of truth for every technology choice in CasLearn OLMS. Deviations require an ADR in `caslearn-docs/adr/`.

## Backend

- **Language:** Java 21 (latest LTS). Use modern features: virtual threads, records, sealed types, pattern matching for `switch`, text blocks.
- **Framework:** Spring Boot 3.x (Spring Security, Spring Data JPA, Spring Web, Spring Cloud Gateway for the gateway service).
- **Build tool:** Maven (multi-module where appropriate). One service = one repo = one Maven project.
- **JDK distribution:** Eclipse Temurin 21.
- **Testing:**
  - JUnit 5 for unit tests
  - Spring Boot Test + Testcontainers for integration tests (real MySQL, Redis, MinIO)
  - Mockito for mocking
  - jqwik for property-based tests (see correctness properties in `requirements.md`)
  - JMeter for load/performance testing (nightly in CI)
- **Code quality:** SonarQube (self-hosted, OSS edition), Spotless for formatting, Checkstyle for linting.
- **Minimum line coverage:** 70% enforced in CI per service.

## Frontend

- **Language:** TypeScript (strict mode, `noImplicitAny`, `strictNullChecks`).
- **Framework:** React 18+.
- **State management:** Redux Toolkit with RTK Query for data fetching and caching.
- **Build tool:** Vite.
- **Styling:** Tailwind CSS + a component library (Material UI or Chakra — final pick in design phase). Pick one and stay consistent.
- **Routing:** React Router v6.
- **Forms:** React Hook Form + Zod for validation.
- **Testing:**
  - Vitest + React Testing Library for unit/component tests
  - Playwright for E2E
  - fast-check for property-based tests where applicable
- **Accessibility:** WCAG 2.1 AA target; axe-core in CI.
- **Package manager:** pnpm.

## Data Layer

- **Primary DB:** MySQL 8 (per-service schema; no cross-service joins).
- **Caching / sessions:** Redis 7.
- **Object storage:** MinIO on-prem, S3 on AWS. Accessed via the S3 SDK — behavior is interchangeable.
- **Vector store:** FAISS (embedded) for local dev; Qdrant for production (OSS, runs in-cluster).
- **Search (future):** OpenSearch if needed post-MVP.

## AI / ML

- **LLM:** OpenAI-compatible API endpoint (configurable). Never call the LLM directly from any service other than `caslearn-ai-service`.
- **Pattern:** Retrieval-Augmented Generation (RAG). Always ground responses in retrieved chunks; always include citations.
- **Embeddings:** `text-embedding-3-small` or equivalent OSS model via a local embedding service.
- **Guardrail:** Retrieval candidate set is filtered by the caller's authorized Content_Items before the prompt is assembled.

## Infrastructure & Runtime

- **Orchestration:** Kubernetes. Local dev on `kind` or `k3s`; on-prem production cluster; optional AWS EKS.
- **IaC:** Terraform (modules in `caslearn-platform-tf`, env compositions in `caslearn-env-tf`).
- **IaC automation:** Atlantis for `terraform plan` / `apply` from GitHub PR comments.
- **Packaging:** Helm charts in `caslearn-platform-helm`. Two values files per chart: `values.onprem.yaml`, `values.aws.yaml`.
- **GitOps deployment:** ArgoCD (app-of-apps pattern in `caslearn-platform-argo`).
- **CI:** Jenkins (pipelines defined as Jenkinsfile; shared library in `caslearn-jenkins-build`).
- **Artifact registry:**
  - Java artifacts → JFrog Artifactory (OSS edition for dev)
  - Frontend artifacts → JFrog Artifactory (npm repo) or GitHub Packages — decide in design phase
  - Container images → JFrog Artifactory Docker registry
- **Service mesh:** Istio with `STRICT` mTLS and per-service `AuthorizationPolicy`.
- **Ingress:** Istio gateway + cert-manager for TLS.
- **Secrets:** Kubernetes Secrets + Sealed Secrets (or HashiCorp Vault OSS).
- **Source control:** GitHub (team-shared org). Branch protection on `main` requires review + passing CI.

## Observability (OSS only — no Datadog)

- **Metrics:** Prometheus + kube-state-metrics + node-exporter.
- **Logs:** Loki + Promtail (structured JSON from every service).
- **Traces:** Tempo (or Jaeger). W3C `traceparent` propagation across services.
- **Dashboards:** Grafana (one dashboard per service + an `Admin Overview` dashboard).
- **Alerts:** Alertmanager. Default alerts: `HighErrorRate` (>5% over 5min), `HighLatencyP95` (>1s over 5min).

## Security Tooling

- **SAST:** SonarQube.
- **Dependency scan:** Trivy + Dependabot.
- **Container scan:** Trivy in CI.
- **Secret scan:** Gitleaks in CI; block on findings.
- **Runtime:** Falco (optional post-MVP).

## Standard Commands Per Repo

Every `*-service` (Java) repo supports:
```bash
mvn verify              # compile + unit tests + integration tests
mvn spring-boot:run     # run locally
docker build -t ...     # build image
helm lint ../caslearn-platform-helm/charts/<service>
```

Every `*-ui` repo supports:
```bash
pnpm install
pnpm dev                # local dev server
pnpm test               # unit tests
pnpm test:e2e           # Playwright
pnpm build              # production build
```

Every `*-tf` repo supports:
```bash
terraform fmt -check
terraform validate
terraform plan -var-file=envs/<env>.tfvars
# apply only via Atlantis
```

## Versioning

- Backend services: Semantic Versioning on container images. Tag format `MAJOR.MINOR.PATCH-<git-sha>`.
- Helm charts: SemVer independent of service version.
- API contracts in `caslearn-api-contracts`: SemVer with breaking-change review required.
