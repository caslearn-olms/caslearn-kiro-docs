# Requirements Document

## Introduction

CasLearn OLMS is a modern, secure, AI-assisted Online Learning Management System (LMS) intended to fully replace Blackboard for educational institutions. The platform delivers a single, centralized, cloud-native experience covering course delivery, assessments, messaging, grading, analytics, and AI-powered learning assistance. The MVP targets three personas — Administrator, Instructor, and Student — and is deployable on-premises (primary) or on AWS EKS (option) using the same Helm charts and container images.

The system is built as a microservices architecture with strict repository naming conventions, an Istio service mesh for mTLS and authorization, a full open-source observability stack (Prometheus, Grafana, Loki, Tempo), and GitOps-driven delivery via Jenkins and ArgoCD. A Retrieval-Augmented Generation (RAG) AI layer grounds learning assistance in authorized course materials, and an adaptive study planner produces personalized schedules from student performance data.

This document specifies functional requirements per persona and capability, plus first-class requirements for repositories, infrastructure, DevOps, observability, documentation, and a Developer Day-0 end-to-end smoke test. A dedicated Correctness Properties section formalizes invariants suitable for property-based testing.

**Target MVP Release:** July 31, 2026.

## Glossary

- **Caslearn_OLMS**: The full CasLearn OLMS LMS product, composed of all UI, backend services, data stores, AI layer, and infrastructure described in this document.
- **Web_UI**: The `caslearn-web-ui` React/TypeScript single-page application served to end users.
- **Gateway_Service**: The `caslearn-gateway-service` API gateway / Backend-for-Frontend that fronts all backend services.
- **Auth_Service**: The `caslearn-auth-service` responsible for authentication, user management, and token issuance.
- **Course_Service**: The `caslearn-course-service` responsible for course CRUD, enrollment, and content management.
- **Assessment_Service**: The `caslearn-assessment-service` responsible for quizzes, assignments, submissions, auto-grading, and manual grading workflows.
- **Messaging_Service**: The `caslearn-messaging-service` responsible for in-app messages, announcements, and notification delivery.
- **Analytics_Service**: The `caslearn-analytics-service` responsible for reports and dashboards.
- **AI_Service**: The `caslearn-ai-service` responsible for the AI Learning Assistant (RAG) and Adaptive Study Planner.
- **Notification_Subsystem**: The component within the Messaging_Service that delivers in-app and email notifications.
- **Audit_Log**: The append-only, tamper-evident record of security-relevant events produced by every backend service.
- **RBAC_Engine**: The combination of Istio AuthorizationPolicy and Spring Security checks that enforce role-based access control.
- **Administrator**: A user with the `ADMIN` role who manages users, courses, roles, system policies, and observability.
- **Instructor**: A user with the `INSTRUCTOR` role who authors courses, content, and assessments, and grades student work.
- **Student**: A user with the `STUDENT` role who enrolls in courses, consumes content, submits work, and uses the AI assistant.
- **Role**: One of `ADMIN`, `INSTRUCTOR`, or `STUDENT`.
- **JWT**: A JSON Web Token issued by Auth_Service used to authenticate API requests.
- **OAuth2**: The OAuth 2.0 authorization framework used for token issuance and validation.
- **Course**: A unit of instruction owned by one or more Instructors and containing content items, assessments, and enrolled Students.
- **Enrollment**: A record linking a Student to a Course with a status of `ACTIVE`, `DROPPED`, or `COMPLETED`.
- **Content_Item**: An uploaded file or structured resource (document, video link, reading) attached to a Course.
- **Assignment**: A Course deliverable that accepts file submissions and is graded manually or with a rubric.
- **Quiz**: A timed or untimed assessment composed of objective and/or subjective questions.
- **Submission**: A Student's response to an Assignment or Quiz, including uploaded files and/or answers.
- **Grade_Book**: The per-Course record of all Student grades for all graded items in that Course.
- **Rubric**: A structured scoring guide used to grade a Submission.
- **RAG**: Retrieval-Augmented Generation; an LLM answering pattern that grounds responses in retrieved documents.
- **Vector_Store**: The FAISS (or Qdrant) index of embeddings over authorized course materials used by AI_Service for retrieval.
- **Adaptive_Study_Planner**: The AI_Service feature that produces a personalized study schedule for a Student based on performance data.
- **Service_Mesh**: Istio, providing mTLS, traffic management, and AuthorizationPolicy enforcement between services.
- **Observability_Stack**: The combination of Prometheus, Grafana, Loki, Tempo (or Jaeger), and Alertmanager.
- **Platform_Repository**: Any GitHub repository governed by the CasLearn naming conventions (`*-ui`, `*-service`, `*-tf`, `*-build`, `*-helm`, `*-argo`, `*-contracts`).
- **Day_0_Smoke_Test**: The end-to-end verification described in Requirement 20 that confirms the UI, Gateway_Service, a backend service, and the data path work together on a fresh developer machine.
- **FERPA**: The Family Educational Rights and Privacy Act; the U.S. student-data-privacy law the platform aligns with.
- **p95**: The 95th percentile of a latency distribution measured over a rolling 5-minute window at steady-state load.

## Requirements

---

## A. Platform-Wide Requirements

### Requirement 1: Authentication and Token Issuance

**User Story:** As any user of CasLearn OLMS, I want to sign in with my credentials and receive a secure session token, so that I can access the features permitted by my role.

#### Acceptance Criteria

1. WHEN a user submits valid credentials to the login endpoint, THE Auth_Service SHALL issue a signed JWT containing the user identifier, role, and an expiration claim not exceeding 60 minutes.
2. WHEN a user submits invalid credentials, THE Auth_Service SHALL return an HTTP 401 response and SHALL NOT reveal whether the username or the password was incorrect.
3. WHEN a JWT is presented to any backend service, THE Gateway_Service SHALL validate the JWT signature and expiration before forwarding the request.
4. IF a JWT is expired, THEN THE Gateway_Service SHALL return an HTTP 401 response with an error code of `TOKEN_EXPIRED`.
5. WHEN a user requests a token refresh with a valid refresh token, THE Auth_Service SHALL issue a new JWT and rotate the refresh token.
6. IF five consecutive failed login attempts occur for the same username within 10 minutes, THEN THE Auth_Service SHALL lock the account for 15 minutes and SHALL emit an Audit_Log entry.
7. THE Auth_Service SHALL store passwords using a salted bcrypt hash with a work factor of at least 12.

### Requirement 2: Role-Based Access Control

**User Story:** As an Administrator, I want every API request authorized by role at both the service mesh and application layers, so that users can only perform actions allowed for their role.

#### Acceptance Criteria

1. THE RBAC_Engine SHALL recognize exactly three roles: `ADMIN`, `INSTRUCTOR`, and `STUDENT`.
2. WHEN a request arrives at any backend service, THE Service_Mesh SHALL enforce an Istio AuthorizationPolicy that permits the request only if the JWT role claim is authorized for the target service and path.
3. WHEN a request passes Service_Mesh authorization, THE target backend service SHALL additionally enforce role and ownership checks in Spring Security before executing business logic.
4. IF a request's role is not authorized for the target operation, THEN THE target backend service SHALL return an HTTP 403 response and SHALL emit an Audit_Log entry of type `AUTHZ_DENIED`, EXCEPT that THE target backend service SHALL return an HTTP 404 response when disclosing the existence of the target resource would itself leak information to the caller.
5. IF any API request other than an Administrator-initiated role change contains a `role` field in its body, headers, or query string, THEN THE Auth_Service SHALL reject the request with an HTTP 403 response regardless of whether the submitted value matches the user's current role.
6. THE RBAC_Engine SHALL evaluate role and ownership checks without relying on client-supplied role fields in the request body.

### Requirement 3: Audit Logging

**User Story:** As an Administrator, I want a tamper-evident audit log of security-relevant events, so that I can investigate incidents and demonstrate compliance.

#### Acceptance Criteria

1. WHEN any authentication, authorization, grade change, enrollment change, content deletion, role change, or administrative action occurs, THE originating backend service SHALL write an Audit_Log entry containing actor identifier, role, action, target resource, timestamp, and request identifier.
2. THE Audit_Log SHALL be append-only, and no API path SHALL permit mutation or deletion of existing Audit_Log entries.
3. THE Audit_Log SHALL chain each entry to its predecessor using a cryptographic hash so that tampering is detectable.
4. IF the cryptographic chaining step fails while the base Audit_Log write succeeds, THEN THE originating backend service SHALL reject the triggering request with an HTTP 500 response.
5. WHERE an Administrator requests an Audit_Log export for a date range, THE Auth_Service SHALL return the matching entries in a signed, downloadable file.
6. IF an Audit_Log write fails because of an actual persistence error, THEN THE originating backend service SHALL reject the triggering request with an HTTP 500 response and SHALL emit a metric of name `audit_log_write_failure_total`.
7. WHERE Audit_Log writes have been intentionally disabled or placed in maintenance mode by an Administrator, THE originating backend service SHALL permit the triggering request to proceed and SHALL emit a metric of name `audit_log_disabled_requests_total`.

### Requirement 4: Input Validation and Rate Limiting

**User Story:** As an Administrator, I want every public API validated and rate-limited, so that the platform resists abuse and injection attacks.

#### Acceptance Criteria

1. WHEN any request is received at the Gateway_Service, THE Gateway_Service SHALL validate the request against the OpenAPI contract from `caslearn-api-contracts` and reject malformed requests with HTTP 400.
2. WHEN a client exceeds 100 requests per minute per JWT subject on any endpoint, THE Gateway_Service SHALL return an HTTP 429 response with a `Retry-After` header.
3. THE Gateway_Service SHALL reject any request with a body larger than 25 MiB except for explicitly whitelisted upload endpoints.
4. WHEN a file upload endpoint receives a file, THE receiving backend service SHALL reject files whose MIME type is not on the endpoint's allow-list and SHALL reject files exceeding the endpoint's configured size limit.
5. IF a request contains content matching SQL, shell, or script injection patterns in validated fields, THEN THE receiving backend service SHALL reject the request with HTTP 400 and SHALL emit an Audit_Log entry of type `INPUT_VALIDATION_REJECTED`.

### Requirement 5: Encryption and Secrets

**User Story:** As an Administrator, I want all traffic and sensitive data encrypted and all secrets managed centrally, so that the platform meets security expectations.

#### Acceptance Criteria

1. THE Service_Mesh SHALL enforce mTLS in `STRICT` mode between every backend service.
2. THE Gateway_Service SHALL accept only TLS 1.2 or TLS 1.3 connections from clients.
3. THE Caslearn_OLMS SHALL store all user credentials, JWT signing keys, database passwords, and LLM API keys in Kubernetes Secrets managed by Sealed Secrets or HashiCorp Vault.
4. THE Caslearn_OLMS SHALL encrypt MySQL data at rest using the underlying storage provider's encryption.
5. IF a secret is detected in source code by SAST or dependency scanning, THEN THE CI pipeline SHALL fail the build.

---

## B. Administrator Capabilities

### Requirement 6: User Management

**User Story:** As an Administrator, I want to create, update, deactivate, and assign roles to users, so that I can control who can access the platform and what they can do.

#### Acceptance Criteria

1. WHEN an Administrator submits a user creation request with a unique email, role, and display name, THE Auth_Service SHALL create the user account in a `PENDING_ACTIVATION` state and SHALL emit an activation email through the Notification_Subsystem.
2. WHEN an Administrator deactivates a user, THE Auth_Service SHALL invalidate all active JWTs and refresh tokens for that user within 5 minutes.
3. WHEN an Administrator changes a user's role, THE Auth_Service SHALL persist the change, emit an Audit_Log entry of type `ROLE_CHANGED`, and require the user to re-authenticate before the new role takes effect.
4. IF an Administrator attempts to delete the last remaining account with the `ADMIN` role, THEN THE Auth_Service SHALL reject the request with HTTP 409 and the error code `LAST_ADMIN_PROTECTION`.
5. WHEN an Administrator searches users by email, role, or status, THE Auth_Service SHALL return paginated results with a page size configurable between 10 and 100.

### Requirement 7: Course Administration

**User Story:** As an Administrator, I want to create courses, assign instructors, and manage course lifecycle, so that the catalog stays accurate.

#### Acceptance Criteria

1. WHEN an Administrator creates a course with a unique course code, title, term, and at least one Instructor, THE Course_Service SHALL persist the course in `DRAFT` status.
2. WHEN an Administrator publishes a course, THE Course_Service SHALL transition the course to `PUBLISHED` status and SHALL make the course visible to enrolled Students.
3. WHEN an Administrator archives a course, THE Course_Service SHALL transition the course to `ARCHIVED` status and SHALL make the course read-only for all roles.
4. IF an Administrator attempts to delete a course with existing Submissions, THEN THE Course_Service SHALL reject the request with HTTP 409 and the error code `COURSE_HAS_SUBMISSIONS`.

### Requirement 8: System Policies and Observability Access

**User Story:** As an Administrator, I want to configure system-wide policies and view system health, so that I can keep the platform healthy.

#### Acceptance Criteria

1. WHEN an Administrator updates a system policy (password policy, session duration, upload size limits, rate-limit thresholds), THE Auth_Service SHALL persist the change and SHALL emit an Audit_Log entry of type `POLICY_CHANGED`.
2. THE Observability_Stack SHALL expose a Grafana dashboard named `Admin Overview` that displays per-service request rate, error rate, p95 latency, CPU, memory, and active user count.
3. WHEN an Administrator opens the `Admin Overview` dashboard, THE Grafana SHALL render all panels with data from the last 15 minutes within 5 seconds.
4. WHERE an Administrator views the Audit_Log UI, THE Auth_Service SHALL provide filtering by actor, action, resource type, and date range.

---

## C. Instructor Capabilities

### Requirement 9: Course Authoring and Content Management

**User Story:** As an Instructor, I want to author course content, so that my students have the materials they need to learn.

#### Acceptance Criteria

1. WHEN an Instructor uploads a Content_Item to a course they own, THE Course_Service SHALL store the file in object storage, record metadata in MySQL, and return the Content_Item identifier.
2. THE Course_Service SHALL accept Content_Item uploads with MIME types in `{application/pdf, application/vnd.openxmlformats-officedocument.wordprocessingml.document, application/vnd.openxmlformats-officedocument.presentationml.presentation, image/png, image/jpeg, video/mp4, text/plain, text/markdown}` and size between 0 bytes and 500 MiB inclusive.
3. IF an Instructor submits a Content_Item upload for a course they do not own, THEN THE Course_Service SHALL reject the request with HTTP 403 before opening, storing, or recording metadata for the uploaded file.
4. WHEN an Instructor organizes Content_Items into modules, THE Course_Service SHALL persist the module order and SHALL return the ordered structure on subsequent reads.
5. WHEN an Instructor deletes a Content_Item, THE Course_Service SHALL soft-delete the item, retain the file in object storage for 30 days, and emit an Audit_Log entry of type `CONTENT_DELETED`.

### Requirement 10: Assessment Authoring

**User Story:** As an Instructor, I want to author quizzes and assignments, so that I can measure student learning.

#### Acceptance Criteria

1. WHEN an Instructor creates a Quiz with one or more questions of types in `{MULTIPLE_CHOICE, TRUE_FALSE, SHORT_ANSWER, ESSAY}`, THE Assessment_Service SHALL persist the Quiz in `DRAFT` status.
2. WHEN an Instructor publishes a Quiz with a start time, due time, and time limit, THE Assessment_Service SHALL transition the Quiz to `PUBLISHED` and SHALL make it available to enrolled Students at the start time.
3. WHEN an Instructor creates an Assignment with a due date, allowed file types, maximum file size, and optional Rubric, THE Assessment_Service SHALL persist the Assignment.
4. IF an Instructor publishes a Quiz whose total possible points do not equal the sum of per-question points, THEN THE Assessment_Service SHALL reject the request with HTTP 400 and the error code `POINTS_MISMATCH`.
5. WHEN an Instructor edits a published Quiz after any Student has started an attempt, THE Assessment_Service SHALL reject the edit with HTTP 409 and the error code `QUIZ_IN_PROGRESS`.

### Requirement 11: Grading Workflows

**User Story:** As an Instructor, I want auto-grading for objective questions and efficient manual grading for subjective work, so that I can return grades quickly and accurately.

#### Acceptance Criteria

1. WHEN a Student submits a Quiz attempt containing only objective questions, THE Assessment_Service SHALL compute the score deterministically against the answer key and SHALL record the grade within 5 seconds.
2. WHEN a Student submits a Quiz attempt containing one or more subjective questions, THE Assessment_Service SHALL auto-grade the objective portion and SHALL leave the subjective portion in `AWAITING_MANUAL_GRADE` status.
3. WHEN an Instructor submits manual grades for a Submission using a Rubric, THE Assessment_Service SHALL persist the Rubric score breakdown, compute the total, and update the Grade_Book.
4. THE Assessment_Service SHALL compute a Quiz score deterministically such that the same set of answers evaluated against the same Rubric always yields the same total.
5. WHEN a grade is released to a Student, THE Assessment_Service SHALL emit a notification through the Notification_Subsystem and SHALL emit an Audit_Log entry of type `GRADE_RELEASED`.
6. IF an Instructor attempts to modify a released grade, THEN THE Assessment_Service SHALL require a justification field, SHALL persist both the old and new values, and SHALL emit an Audit_Log entry of type `GRADE_AMENDED`.

### Requirement 12: Instructor Communication

**User Story:** As an Instructor, I want to message students and post announcements, so that I can communicate course updates.

#### Acceptance Criteria

1. WHEN an Instructor posts an announcement to a course they own, THE Messaging_Service SHALL persist the announcement and SHALL notify all enrolled Students through the Notification_Subsystem.
2. WHEN an Instructor sends a direct message to one or more enrolled Students, THE Messaging_Service SHALL deliver the message in-app within 5 seconds and SHALL emit a notification.
3. IF an Instructor attempts to message a user who is not enrolled in one of their courses, THEN THE Messaging_Service SHALL reject the request with HTTP 403.

### Requirement 13: Instructor Analytics

**User Story:** As an Instructor, I want dashboards that show student progress and course engagement, so that I can identify students who need support.

#### Acceptance Criteria

1. WHEN an Instructor opens a course analytics dashboard, THE Analytics_Service SHALL render per-assessment average, median, and distribution within 3 seconds.
2. THE Analytics_Service SHALL display per-Student progress, showing Content_Item completion, Submission status, and grade summary, only after the Gateway_Service and RBAC_Engine have authorized the caller as an Instructor of that Course.
3. IF an Instructor requests analytics for a course they do not own, THEN THE Analytics_Service SHALL reject the request with HTTP 403.
4. IF the Analytics_Service's authorization check cannot complete (for example, because the RBAC_Engine is unavailable), THEN THE Analytics_Service SHALL fail closed by rejecting the request with HTTP 503 and SHALL emit a metric of name `analytics_authz_unavailable_total`.

---

## D. Student Capabilities

### Requirement 14: Enrollment and Content Consumption

**User Story:** As a Student, I want to enroll in available courses and access their materials, so that I can learn.

#### Acceptance Criteria

1. WHEN a Student requests enrollment in a `PUBLISHED` course that accepts self-enrollment, THE Course_Service SHALL create an Enrollment with status `ACTIVE`.
2. WHEN a Student opens a Course, THE Course_Service SHALL return the ordered list of Content_Items and assessments for that Course only if the Student has an `ACTIVE` Enrollment.
3. IF a Student requests a Content_Item for a course in which they have no `ACTIVE` Enrollment, THEN THE Course_Service SHALL reject the request with HTTP 403.
4. WHEN a Student drops a Course before the drop deadline, THE Course_Service SHALL transition the Enrollment to `DROPPED` and SHALL remove access to course materials.

### Requirement 15: Assignment and Quiz Submission

**User Story:** As a Student, I want to submit assignments and take quizzes, so that I can demonstrate my learning.

#### Acceptance Criteria

1. WHEN a Student submits an Assignment before the due date with files matching the Assignment's allowed types and size, THE Assessment_Service SHALL persist the Submission and SHALL return a confirmation receipt.
2. IF a Student attempts to submit an Assignment after the due date, THEN THE Assessment_Service SHALL accept the Submission only when the Assignment permits late submissions and SHALL mark the Submission as `LATE`.
3. WHEN a Student starts a Quiz with a time limit, THE Assessment_Service SHALL record the start timestamp and SHALL auto-submit the attempt when the time limit elapses.
4. IF a Student's Quiz attempt is auto-submitted because of the time limit, THEN THE Assessment_Service SHALL grade the attempt using the answers recorded at the moment of auto-submission.
5. WHEN a Student views their own grades, THE Assessment_Service SHALL return the grades for that Student only.
6. IF a Student requests any grade record belonging to a different Student, THEN THE Assessment_Service SHALL reject the request with HTTP 403.

### Requirement 16: Student Messaging and Notifications

**User Story:** As a Student, I want to receive announcements and messages, so that I stay informed about my courses.

#### Acceptance Criteria

1. WHEN an announcement is posted in a Course a Student is enrolled in, THE Notification_Subsystem SHALL deliver an in-app notification within 5 seconds and an email within 5 minutes.
2. WHEN a Student replies to a direct message from an Instructor, THE Messaging_Service SHALL deliver the reply within 5 seconds.
3. WHERE a Student has disabled email notifications in preferences, THE Notification_Subsystem SHALL suppress email delivery while still delivering in-app notifications.

### Requirement 17: AI Learning Assistant

**User Story:** As a Student, I want an AI assistant that answers my questions using my course materials, so that I can get help grounded in what the instructor actually taught.

#### Acceptance Criteria

1. WHEN a Student submits a question to the AI Learning Assistant in the context of a Course, THE AI_Service SHALL retrieve the top K most relevant chunks from the Vector_Store restricted to Content_Items the Student is authorized to access.
2. WHEN the AI_Service generates a response, THE AI_Service SHALL include citations referencing the Content_Items used and SHALL persist the prompt, retrieved chunks, and response for audit.
3. IF the retrieval step returns no chunks above a similarity threshold of 0.2, THEN THE AI_Service SHALL return a response stating that the course materials do not cover the question rather than answering from the LLM's general knowledge.
4. THE AI_Service SHALL exclude every Content_Item that the Student is not authorized to read from the retrieval candidate set, regardless of whether the question implicitly or explicitly references such materials.
5. THE AI_Service SHALL return a response within 10 seconds at p95 for questions under 500 tokens.
6. WHEN a Student exceeds 30 AI Learning Assistant questions within a rolling 60-minute window, THE Gateway_Service SHALL return an HTTP 429 response with a `Retry-After` header and the error code `AI_RATE_LIMITED`, and THE AI_Service SHALL emit a metric of name `ai_rate_limit_total` labelled by Student identifier, in addition to the platform-wide rate limit in Requirement 4.2.
7. WHERE an Administrator configures a different per-Student AI rate limit through the system policies in Requirement 8.1, THE Gateway_Service SHALL apply that configured value instead of the default 30-per-hour value in clause 6.

### Requirement 18: Adaptive Study Planner

**User Story:** As a Student, I want a personalized study schedule based on my performance, so that I focus my time where it will help most.

#### Acceptance Criteria

1. WHEN a Student opens the Adaptive_Study_Planner for a Course, THE AI_Service SHALL generate a schedule using the Student's recent Submissions, grades, and upcoming assessment due dates.
2. THE Adaptive_Study_Planner SHALL prioritize topics where the Student's recent assessment score is below 70% of the Course average.
3. WHEN a Student marks a planner item as complete, THE AI_Service SHALL update the schedule to reflect progress.
4. IF the AI_Service cannot generate a plan because the Student has no graded Submissions yet, THEN THE AI_Service SHALL return a default plan derived from the Course's module order.

---

## E. Repository, Infrastructure, DevOps, Observability, and Documentation

### Requirement 19: Repository Naming and Bootstrapping

**User Story:** As an Administrator, I want every repository to follow strict naming conventions and ship with standard scaffolding, so that the codebase stays consistent as the team grows.

#### Acceptance Criteria

1. THE Caslearn_OLMS SHALL use exactly these repository suffixes: `-ui` for frontend applications, `-service` for backend Java services, `-tf` for Terraform repositories, `-build` for Jenkins shared libraries and pipeline templates, `-helm` for Helm chart repositories, `-argo` for ArgoCD Application manifests, and `-contracts` for shared OpenAPI specifications.
2. THE Caslearn_OLMS SHALL create the following initial repositories for MVP: `caslearn-web-ui`, `caslearn-auth-service`, `caslearn-course-service`, `caslearn-assessment-service`, `caslearn-messaging-service`, `caslearn-analytics-service`, `caslearn-ai-service`, `caslearn-gateway-service`, `caslearn-platform-tf`, `caslearn-env-tf`, `caslearn-jenkins-build`, `caslearn-platform-helm`, `caslearn-platform-argo`, `caslearn-api-contracts`, and `caslearn-docs`.
3. THE Caslearn_OLMS SHALL provide a `gh`-CLI-based bootstrap script in `caslearn-docs` that creates every MVP repository in the configured GitHub organization, pushes initial scaffolding from a template, and configures branch protection on `main` requiring at least one review and passing CI.
4. IF a repository name does not match one of the allowed suffixes, THEN THE CI pipeline SHALL fail the build and SHALL report the repository-naming violation as a distinct failure reason, independently of any other concurrent CI failures for the same build.
5. THE Caslearn_OLMS SHALL provide a GitHub repository template that pre-populates each new repository with a standard `README.md`, `CODEOWNERS`, `.gitignore`, license, pull request template, and language-appropriate build configuration.

### Requirement 20: Developer Day-0 Smoke Test

**User Story:** As a new developer, I want a one-command local bring-up and an end-to-end smoke test, so that I can verify the platform works on my machine before touching any feature code.

#### Acceptance Criteria

1. THE Caslearn_OLMS SHALL provide a single command in `caslearn-docs` that starts a local `kind` or `k3s` cluster, deploys MySQL, Redis, object storage, the Vector_Store, and every MVP backend service plus the Web_UI using the MVP Helm charts.
2. WHEN the Day_0_Smoke_Test command completes, THE Web_UI SHALL be reachable at a documented local URL over HTTPS using a local development certificate.
3. WHEN a developer performs the Day_0_Smoke_Test, THE Web_UI SHALL accept login with a seeded Administrator account, SHALL call the Gateway_Service with the issued JWT, SHALL route to at least one backend service, and SHALL render the response in the UI, and THE Web_UI MAY begin accepting this traffic before the bring-up command officially returns success.
4. THE Day_0_Smoke_Test SHALL complete end-to-end within 10 minutes on a developer machine with 16 GiB of RAM and 4 CPU cores, measured from the start of the bring-up command to the UI rendering the backend response.
5. IF any step of the Day_0_Smoke_Test fails, THEN THE bring-up command SHALL exit with a non-zero status and SHALL print the failing step name and its log location.
6. THE Day_0_Smoke_Test SHALL run in CI on every change to `caslearn-platform-helm` and `caslearn-docs` and SHALL block merge on failure.
7. WHERE an authorized developer with the `release-override` GitHub team membership attaches a signed justification comment of the form `override: smoke-test REASON=<text>` to the pull request, THE CI pipeline SHALL permit merge despite a failing Day_0_Smoke_Test and SHALL emit an Audit_Log entry of type `SMOKE_TEST_OVERRIDE`.

### Requirement 21: Infrastructure-as-Code with Terraform and Atlantis

**User Story:** As an Administrator, I want all infrastructure defined as code and changes gated by pull request automation, so that infrastructure changes are reviewable and reproducible.

#### Acceptance Criteria

1. THE Caslearn_OLMS SHALL define all cloud and cluster infrastructure in Terraform modules within `caslearn-platform-tf` and environment compositions within `caslearn-env-tf`.
2. WHEN a pull request is opened against `caslearn-env-tf`, THE Atlantis SHALL run `terraform plan` and SHALL post the plan output as a comment on the pull request.
3. WHEN a reviewer comments `atlantis apply` on an approved pull request, THE Atlantis SHALL run `terraform apply` and SHALL post the result as a comment.
4. THE Caslearn_OLMS SHALL provide two environment compositions in `caslearn-env-tf`: `on-prem` and `aws-eks`, producing functionally equivalent clusters.
5. IF a Terraform plan proposes destroying a stateful resource (MySQL database, object storage bucket, persistent volume), THEN THE Atlantis SHALL require a second approving review before permitting apply.

### Requirement 22: CI/CD with Jenkins and ArgoCD

**User Story:** As a developer, I want automated build, test, and deploy pipelines, so that changes reach environments safely and quickly.

#### Acceptance Criteria

1. WHEN a pull request is opened against any `*-service` or `*-ui` repository, THE Jenkins SHALL run unit tests, SAST via SonarQube, dependency scanning via Trivy, and a container image build, and SHALL report status back to GitHub.
2. WHEN a pull request is merged to `main` in any `*-service` or `*-ui` repository, THE Jenkins SHALL publish a versioned container image to JFrog Artifactory and SHALL update the image tag in `caslearn-platform-argo`.
3. WHEN an image tag is updated in `caslearn-platform-argo`, THE ArgoCD SHALL synchronize the target environment cluster to the new desired state.
4. IF any CI stage fails, THEN THE Jenkins SHALL mark the build as failed and SHALL prevent image publication.
5. THE Caslearn_OLMS SHALL deploy to environments in the order `dev` → `staging` → `prod`, with promotion requiring the previous environment's health checks to pass.

### Requirement 23: Helm Charts and Portability

**User Story:** As an Administrator, I want the same Helm charts to deploy the platform on-prem and on AWS EKS, so that portability is preserved.

#### Acceptance Criteria

1. THE Caslearn_OLMS SHALL package every backend service and the Web_UI as Helm charts in `caslearn-platform-helm`.
2. WHEN a Helm chart is installed with the `onprem` values file, THE chart SHALL use MinIO for object storage and local-path persistent volumes, and THE chart SHALL reject any override that selects S3 or EBS by failing the Helm `pre-install` validation hook.
3. WHEN a Helm chart is installed with the `aws` values file, THE chart SHALL use S3 for object storage and EBS-backed persistent volumes, and THE chart SHALL reject any override that selects MinIO or local-path storage by failing the Helm `pre-install` validation hook.
4. THE Helm charts SHALL produce the same Kubernetes resource kinds and counts for `onprem` and `aws` values, differing only in storage and ingress annotations.

### Requirement 24: Service Mesh and Inter-Service Security

**User Story:** As an Administrator, I want Istio to enforce mTLS and authorization policies between services, so that a compromised service cannot escalate access.

#### Acceptance Criteria

1. THE Service_Mesh SHALL apply a `PeerAuthentication` in `STRICT` mTLS mode to every namespace hosting a CasLearn service.
2. THE Service_Mesh SHALL apply an Istio `AuthorizationPolicy` for each backend service that permits calls only from the specific peer service accounts that need access.
3. IF a service attempts to call another service without a matching AuthorizationPolicy, OR IF THE Service_Mesh cannot evaluate AuthorizationPolicy (for example, because the control plane is unreachable), THEN THE Service_Mesh SHALL deny the request with HTTP 403 (fail closed).
4. THE Service_Mesh SHALL expose request-level metrics for every service-to-service call, including source, destination, response code, and duration.

### Requirement 25: Observability

**User Story:** As an Administrator, I want every service to emit metrics, structured logs, and traces to a central open-source stack, so that I can observe platform behavior without paying for commercial tools.

#### Acceptance Criteria

1. THE Caslearn_OLMS SHALL collect metrics with Prometheus, logs with Loki, traces with Tempo or Jaeger, and SHALL render them in Grafana.
2. THE Caslearn_OLMS SHALL NOT use Datadog or any other paid observability SaaS.
3. THE Caslearn_OLMS SHALL emit structured JSON logs from every backend service containing timestamp, service name, trace identifier, span identifier, user identifier when authenticated, request path, and log level.
4. WHEN a request traverses multiple services, THE Gateway_Service and each backend service SHALL propagate the W3C `traceparent` header so the full trace is visible in Tempo or Jaeger, and IF a downstream service fails to propagate the header, THEN THE calling service SHALL complete the request normally and SHALL emit a metric of name `trace_propagation_failure_total`.
5. THE Observability_Stack SHALL provide a Grafana dashboard per service showing request rate, error rate, p95 latency, and resource usage.
6. WHEN error rate for any service exceeds 5% over 5 minutes, THE Alertmanager SHALL fire an alert named `HighErrorRate` for that service to the configured on-call channel.
7. WHEN p95 latency for any service exceeds 1 second over 5 minutes, THE Alertmanager SHALL fire a separate alert named `HighLatencyP95` for that service to the configured on-call channel, independently of any concurrent `HighErrorRate` alert.

### Requirement 26: Non-Functional Performance, Availability, Scalability, and Accessibility

**User Story:** As any user, I want the platform to be fast, reliable, and accessible, so that I can use it productively.

#### Acceptance Criteria

1. THE Web_UI SHALL achieve a p95 page load time of less than 2 seconds on a broadband connection during steady-state load.
2. THE Gateway_Service SHALL achieve a p95 response time of less than 300 milliseconds for common read endpoints listed in `caslearn-api-contracts` under steady-state load.
3. THE Caslearn_OLMS SHALL target 99.5% monthly availability for the Web_UI and Gateway_Service, measured by synthetic probe.
4. THE backend services SHALL be stateless and SHALL horizontally scale by replica count under load, with session state held in Redis and persistent data held in MySQL or object storage.
5. THE Web_UI SHALL conform to WCAG 2.1 Level AA for all screens in the three persona journeys defined in Requirements 6 through 18.
6. WHEN load testing is performed with JMeter on the Gateway_Service's common read endpoints at 500 concurrent users, THE Gateway_Service SHALL sustain the p95 target in clause 2 of this requirement.

### Requirement 27: Compliance with FERPA

**User Story:** As an Administrator, I want FERPA-aligned handling of student data, so that the institution meets its legal obligations.

#### Acceptance Criteria

1. THE Caslearn_OLMS SHALL restrict access to any Student data — including profile fields, Enrollments, Submissions, grades, messages, analytics records, and AI interaction history — to that Student, Instructors of that Student's Courses for data related to those Courses, and Administrators.
2. WHEN a Student requests an export of their own data, THE Auth_Service SHALL produce a downloadable archive of the Student's profile, Enrollments, Submissions, and grades within 72 hours.
3. WHEN an Administrator processes a data deletion request for a former Student, THE Caslearn_OLMS SHALL remove or anonymize the Student's personally identifiable information while preserving aggregate analytics and Audit_Log integrity.
4. THE Audit_Log SHALL record every access to a Student's grades or Submissions that is performed by any user other than the Student themselves.

### Requirement 28: Documentation Standards

**User Story:** As a new developer or operator, I want every repository documented to a consistent standard, so that I can contribute or operate the platform quickly.

#### Acceptance Criteria

1. THE Caslearn_OLMS SHALL require every Platform_Repository to include a `README.md` with sections for Purpose, Tech Stack, Local Setup, Build and Test Commands, Deployment, Contribution Guidelines, and Owners.
2. IF a pull request to any Platform_Repository adds a public API, environment variable, or configuration flag without a corresponding `README.md` update, THEN THE CI pipeline SHALL fail the build.
3. THE `caslearn-docs` repository SHALL contain the following architecture diagrams maintained as code (for example, Structurizr, PlantUML, or Mermaid): C4 context, C4 container, C4 component per service, service dependency diagram, network and security diagram, on-prem deployment topology, AWS EKS deployment topology, CI/CD pipeline diagram, and data flow diagram.
4. THE `caslearn-docs` repository SHALL contain an onboarding guide that takes a new developer from a clean machine to running the Day_0_Smoke_Test.
5. WHEN a new developer follows the onboarding guide on a supported operating system, THE developer SHALL be able to ship a merged change to any `*-service` or `*-ui` repository within their first week, measured as a success criterion in onboarding feedback surveys.

### Requirement 29: Testing Strategy

**User Story:** As a developer, I want a consistent testing strategy across languages and services, so that quality is enforced uniformly.

#### Acceptance Criteria

1. THE backend services SHALL use JUnit 5 for unit tests and Spring Boot Test with Testcontainers for integration tests.
2. THE Web_UI SHALL use Vitest with React Testing Library for unit tests and Playwright for end-to-end tests.
3. THE Caslearn_OLMS SHALL enforce a minimum line coverage of 70% per backend service and per frontend application, measured in CI.
4. THE Caslearn_OLMS SHALL run property-based tests for each correctness property listed in the Correctness Properties section, using an appropriate property-based testing library in each language.
5. WHEN JMeter load tests are executed in CI on a nightly schedule, THE Jenkins SHALL publish the report as a build artifact and SHALL fail the build if the p95 targets in Requirement 26 are violated.

### Requirement 30: Out-of-Scope Boundaries for MVP

**User Story:** As a stakeholder, I want explicit boundaries on what the MVP excludes, so that scope stays controlled for the July 31, 2026 deadline.

#### Acceptance Criteria

1. THE MVP SHALL NOT include live video conferencing, SCORM integration, LTI integration, payment processing, native mobile applications, single sign-on via SAML or LDAP, multi-factor authentication, proctoring, or plagiarism detection.
2. THE Web_UI SHALL be responsive and usable on modern mobile browsers in place of a native mobile application, and THE Jenkins SHALL block the MVP release build if the Web_UI fails the WCAG 2.1 AA conformance test suite or the Playwright mobile-viewport responsiveness test suite.
3. WHERE a stakeholder requests an out-of-scope capability, THE Caslearn_OLMS roadmap SHALL record the request in `caslearn-docs` under the post-MVP backlog.

---

## Correctness Properties

The properties below are expressed as universally-quantified invariants over the platform's observable state and operations. They inform property-based testing (PBT) in backend services where applicable and integration-style checks elsewhere. Each property lists its form, the component under test, and the PBT suitability per the workflow's PBT decision guide.

### CP-1: Enrollment Gate on Course Materials (PBT)

- **Form**: For every Student `s`, Course `c`, and Content_Item `i` belonging to `c`, a request by `s` to read `i` succeeds if and only if `s` has an Enrollment in `c` with status in `{ACTIVE}` AND `c` is in `{PUBLISHED, ARCHIVED}`.
- **Component**: Course_Service authorization logic.
- **PBT suitability**: Input (Student, Course, Enrollment status, Course status) varies meaningfully; logic under test is our code; high iteration count exposes edge cases.

### CP-2: Grade Confidentiality (PBT)

- **Form**: For every pair of Students `s1 ≠ s2`, every grade record belonging to `s1` is never returned by any API path to `s2`.
- **Component**: Assessment_Service grade read endpoints.
- **PBT suitability**: Negative authorization is a classic property-based target.

### CP-3: No Role Escalation (PBT)

- **Form**: For every API request by a user with role `r`, the user's post-request role is `r` UNLESS the request is an Administrator-initiated role change targeting that user.
- **Component**: Auth_Service and Gateway_Service.
- **PBT suitability**: Role field in body, header, and token can be fuzzed across all endpoints.

### CP-4: Deterministic Assessment Scoring (PBT)

- **Form**: For every Submission `x` and Rubric `r`, `score(x, r) = score(x, r)` on repeated evaluation, and for any two answer sets producing the same canonical form against `r`, their scores are equal.
- **Component**: Assessment_Service grading function.
- **PBT suitability**: Pure function of inputs; ideal for PBT.

### CP-5: Audit Log Append-Only and Tamper-Evident (PBT)

- **Form**: For every sequence of Audit_Log writes `e1, e2, …, en`, no API path produces a state where any `ei` is absent, modified, or reordered, and each `ei.hash = H(ei.payload || e(i-1).hash)`.
- **Component**: Audit_Log subsystem across all services.
- **PBT suitability**: Generate sequences of events and verify chain invariant.

### CP-6: AI Retrieval Authorization (PBT with mocked LLM)

- **Form**: For every Student `s` and question `q`, the set of Content_Items used by AI_Service to ground the response is a subset of the Content_Items `s` is authorized to read.
- **Component**: AI_Service retrieval pipeline.
- **PBT suitability**: Vary user, question, and authorized set; mock the LLM to keep cost bounded.

### CP-7: Quiz Scoring Round-Trip Preservation (PBT)

- **Form**: For every Quiz `Q` and answer set `A`, serializing `(Q, A, score)` to the Grade_Book and reading it back yields the same score and per-question breakdown.
- **Component**: Assessment_Service persistence.
- **PBT suitability**: Serialization round-trip is a canonical PBT target.

### CP-8: Enrollment Count Invariant Under Concurrent Operations (PBT)

- **Form**: For every Course `c`, after any interleaving of enroll and drop operations, `count(enrollments(c, status=ACTIVE)) = enrollments_issued − drops`.
- **Component**: Course_Service.
- **PBT suitability**: Concurrency invariants are well-suited to stateful PBT.

### CP-9: Notification Delivery Idempotence (PBT)

- **Form**: For every notification event `e`, repeated processing of `e` produces exactly one delivered in-app notification and at most one delivered email per recipient.
- **Component**: Notification_Subsystem.
- **PBT suitability**: Idempotence under retries is a classic PBT target.

### CP-10: Helm Chart Portability (Integration, not PBT)

- **Form**: For the same chart version, the set of Kubernetes resource kinds rendered with `onprem` values equals the set rendered with `aws` values.
- **Component**: `caslearn-platform-helm`.
- **PBT suitability**: Not PBT — a single rendering comparison suffices.

### CP-11: JWT Validation Completeness (PBT)

- **Form**: For every token `t` presented to the Gateway_Service, the request is forwarded if and only if `t` has a valid signature against the current or previous signing key AND `t.exp > now` AND `t.iss = expected_issuer`.
- **Component**: Gateway_Service JWT filter.
- **PBT suitability**: Token fields can be fuzzed; high iteration count finds validator bugs.

### CP-12: OpenAPI Contract Conformance (Integration, not PBT)

- **Form**: Every deployed `*-service` responds to every endpoint defined in `caslearn-api-contracts` with responses whose schema matches the contract.
- **Component**: All backend services.
- **PBT suitability**: Not PBT — contract tests with representative examples are more appropriate.
