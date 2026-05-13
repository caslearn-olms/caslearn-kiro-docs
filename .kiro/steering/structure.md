# Repository & Code Structure

## Repository Naming Convention (STRICT — enforced in CI)

Every repository MUST end with one of these suffixes:

| Suffix         | Purpose                                                     | Example                   |
|----------------|-------------------------------------------------------------|---------------------------|
| `-ui`          | Frontend application (TypeScript/React)                     | `caslearn-web-ui`            |
| `-service`     | Backend Java microservice                                   | `caslearn-auth-service`      |
| `-tf`          | Terraform code (modules or environments)                    | `caslearn-platform-tf`       |
| `-build`       | Jenkins shared libraries / pipeline templates               | `caslearn-jenkins-build`     |
| `-helm`        | Helm charts                                                 | `caslearn-platform-helm`     |
| `-argo`        | ArgoCD Application manifests                                | `caslearn-platform-argo`     |
| `-contracts`   | Shared OpenAPI specs / API contracts                        | `caslearn-api-contracts`     |
| `-docs`        | Documentation                                               | `caslearn-docs`              |

A repository name that doesn't match an allowed suffix must fail CI with a distinct, dedicated error message so the naming violation is never hidden behind another failure.

## MVP Repository List

Services (backend):
- `caslearn-auth-service` — authentication, users, RBAC, audit log API
- `caslearn-course-service` — courses, enrollments, content
- `caslearn-assessment-service` — quizzes, assignments, submissions, grading, grade book
- `caslearn-messaging-service` — in-app messages, announcements, notifications
- `caslearn-analytics-service` — reports, dashboards
- `caslearn-ai-service` — RAG-based AI assistant and adaptive study planner
- `caslearn-gateway-service` — API gateway / BFF (Spring Cloud Gateway)

Frontend:
- `caslearn-web-ui` — main React/TypeScript single-page app

Infrastructure:
- `caslearn-platform-tf` — shared Terraform modules (VPC, k8s, databases, object storage)
- `caslearn-env-tf` — environment compositions (dev, staging, prod, on-prem, aws-eks)

CI/CD & Delivery:
- `caslearn-jenkins-build` — shared Jenkins library + reusable pipeline templates
- `caslearn-platform-helm` — Helm charts for every service
- `caslearn-platform-argo` — ArgoCD app-of-apps manifests per environment

Contracts & Docs:
- `caslearn-api-contracts` — OpenAPI specs shared across services; source of truth for Gateway validation
- `caslearn-docs` — architecture, runbooks, onboarding, ADRs, post-MVP backlog

## Standard Layout for a `*-service` Repo

```
caslearn-<name>-service/
├── README.md
├── CODEOWNERS
├── LICENSE
├── .gitignore
├── .editorconfig
├── Jenkinsfile
├── Dockerfile
├── pom.xml
├── .mvn/
├── mvnw, mvnw.cmd
├── src/
│   ├── main/
│   │   ├── java/com/caslearn/<name>/
│   │   │   ├── <Name>Application.java
│   │   │   ├── api/              # controllers
│   │   │   ├── domain/           # entities, value objects
│   │   │   ├── service/          # business logic
│   │   │   ├── repository/       # JPA repos
│   │   │   ├── config/           # Spring config
│   │   │   └── security/         # Spring Security filters
│   │   └── resources/
│   │       ├── application.yml
│   │       └── db/migration/     # Flyway
│   └── test/
│       ├── java/com/caslearn/<name>/
│       │   ├── unit/
│       │   ├── integration/      # Testcontainers
│       │   └── property/         # jqwik
│       └── resources/
├── openapi/                      # published to caslearn-api-contracts
└── docs/                         # service-specific docs
```

## Standard Layout for a `*-ui` Repo

```
caslearn-<name>-ui/
├── README.md
├── CODEOWNERS
├── LICENSE
├── .gitignore
├── .editorconfig
├── Jenkinsfile
├── Dockerfile
├── package.json
├── pnpm-lock.yaml
├── tsconfig.json
├── vite.config.ts
├── public/
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   ├── app/                      # Redux store
│   ├── features/                 # feature-first Redux slices
│   │   └── <feature>/
│   │       ├── api.ts            # RTK Query
│   │       ├── slice.ts
│   │       ├── components/
│   │       ├── pages/
│   │       └── types.ts
│   ├── components/               # shared UI components
│   ├── hooks/
│   ├── lib/                      # utilities
│   └── styles/
├── tests/
│   ├── unit/                     # Vitest
│   └── e2e/                      # Playwright
└── docs/
```

## Standard Layout for a `*-tf` Repo

```
caslearn-platform-tf/                # or caslearn-env-tf
├── README.md
├── atlantis.yaml
├── modules/                      # in caslearn-platform-tf
│   ├── vpc/
│   ├── eks/
│   ├── onprem-k8s/
│   ├── mysql/
│   ├── minio/
│   └── observability/
└── envs/                         # in caslearn-env-tf
    ├── dev/
    ├── staging/
    ├── prod-onprem/
    └── prod-aws/
```

## Standard Layout for `caslearn-platform-helm`

```
caslearn-platform-helm/
├── README.md
├── charts/
│   ├── caslearn-auth-service/
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   ├── values.onprem.yaml
│   │   ├── values.aws.yaml
│   │   └── templates/
│   ├── caslearn-course-service/
│   └── ...
└── umbrella/
    └── caslearn-platform/           # app-of-apps meta chart
```

## Standard Layout for `caslearn-platform-argo`

```
caslearn-platform-argo/
├── README.md
├── apps/
│   ├── dev/
│   ├── staging/
│   └── prod/
└── app-of-apps.yaml
```

## Package & Class Naming (Java)

- Package root: `com.caslearn.<service-short-name>`
- Controllers end in `Controller`
- Services end in `Service`
- Repositories end in `Repository`
- DTOs end in `Dto`; domain entities have no suffix
- Configuration classes end in `Config`

## File Naming (Frontend)

- Components: `PascalCase.tsx`
- Hooks: `useCamelCase.ts`
- Redux slices: `camelCaseSlice.ts`
- RTK Query API: `camelCaseApi.ts`
- Tests: `<target>.test.ts` / `.test.tsx`
- E2E: `<flow>.spec.ts`

## Git Branching

- `main` — protected, always deployable
- Feature branches: `feat/<ticket>-<short-desc>`
- Bug branches: `fix/<ticket>-<short-desc>`
- Chore branches: `chore/<short-desc>`
- All merges via PR with at least one review + passing CI

## Commit Messages

Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`, `test:`, `refactor:`, `perf:`, `build:`, `ci:`). Include ticket ID in the body.
