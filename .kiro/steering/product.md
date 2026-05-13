# Product Overview

## CasLearn OLMS

**Tagline:** Centralized. Intelligent. Secure Learning.

CasLearn OLMS is a modern, secure, AI-assisted Online Learning Management System (LMS) built as a full replacement for Blackboard. It delivers a single, centralized, cloud-native platform covering course delivery, assessments, messaging, grading, analytics, and AI-powered learning assistance.

## Why CasLearn OLMS

Existing LMS platforms suffer from clunky UIs, fragmented experiences, slow backends, and poor extensibility. Institutions end up stitching together multiple disconnected tools for course delivery, assessments, communication, and grading. CasLearn OLMS solves this with one unified platform that prioritizes a great UI, fast APIs, strong security, and intelligent AI assistance grounded in the instructor's own course materials.

## Target Personas

- **Administrator** — manages users, courses, roles, policies, and platform observability.
- **Instructor** — creates courses, authors content and assessments, grades submissions, communicates with students, and views course analytics.
- **Student** — enrolls in courses, consumes content, submits work, takes quizzes, views grades, and uses the AI Learning Assistant and Adaptive Study Planner.

## MVP Scope

**In scope**
- Authentication (JWT/OAuth2) and strict RBAC
- Course CRUD, enrollment, content upload & management
- Assignments (with file upload) and quizzes (with auto-grading for objective questions)
- Manual grading workflows and a course grade book
- In-app messaging and announcements; in-app + email notifications
- Reporting & analytics dashboards for each persona
- AI Learning Assistant (RAG grounded in authorized course materials)
- Adaptive Study Planner driven by student performance data
- Append-only, tamper-evident audit logging
- On-prem Kubernetes deployment (primary) with AWS EKS as a supported option

**Explicitly out of scope for MVP**
- Live video conferencing
- SCORM / LTI enterprise integrations
- Payment processing
- Native mobile apps (responsive web covers mobile)
- SSO via SAML / LDAP

## Delivery Target

MVP demo-ready by **July 31, 2026**. Agile delivery with 2-week sprints. Scope is protected — new requests go to the post-MVP backlog in `caslearn-docs`.

## Core Principles

1. **Security first.** mTLS between services, RBAC at mesh and app layers, audit log for every sensitive action, FERPA alignment.
2. **Fast, delightful UI.** Modern React/TypeScript with a consistent design system. No Blackboard-style clunk.
3. **Grounded AI.** The AI assistant only answers from materials the student is authorized to see; citations always included.
4. **Portability.** Same Helm charts deploy on-prem and on AWS EKS. No cloud lock-in.
5. **Observability built-in.** Every service emits metrics, structured logs, and traces from day one. OSS stack only — no Datadog.
6. **Open source by default.** Free / OSS tools during development; commercial only when justified.
7. **Docs are part of the product.** Every repo has a README; architecture is diagrammed as code.

## Non-Functional Targets

- Web UI p95 page load < 2s; API p95 < 300ms for common reads
- 99.5% monthly availability for the Web UI and Gateway
- WCAG 2.1 AA conformance on all persona journeys
- 70% minimum line coverage per service
- Sustained p95 targets at 500 concurrent users in JMeter load tests

## Roadmap Beyond MVP

Tracked in `caslearn-docs/post-mvp-backlog.md`. Candidates include SSO, SCORM/LTI, mobile native apps, live video, payments, proctoring, plagiarism detection, advanced analytics.
