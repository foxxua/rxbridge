# RxBridge

A microservice platform for enrolling patients in specialty medication programs and reporting adverse events, with two-way Salesforce integration.

> **Status: work in progress.** This is a learning and portfolio project, built stage by stage. See [Roadmap](#roadmap) for what is done and what comes next.

## Overview

RxBridge models a specialty pharmacy hub:

- **Enrollment**: register patients and enroll them in medication programs.
- **Adverse events**: report side effects, track severity and processing deadlines.
- **Salesforce sync**: deliver enrollments and adverse events to Salesforce reliably, and receive changes back.
- **Bulk import**: load large CSV files from pharmacies and distribution centers.
- **Notifications and inventory**: escalations, stock reservation, and a saga for enrollment approval.

## Architecture

_Diagram coming soon._

Each service is a separate repository with its own Gradle build, database, tests and CI. This repository ties them together as git submodules and holds shared infrastructure (`compose.yaml`), documentation and architecture decision records.

## Tech stack

Java 21 · Spring Boot 3 · Gradle · PostgreSQL · Flyway · Testcontainers

_More will be added as the project grows (Kafka, Spring Cloud, Keycloak, Kotlin, and others)._

## Repositories

| Repository | Purpose | Status |
| --- | --- | --- |
| `rxbridge` | Umbrella repo: submodules, infrastructure, docs | in progress |
| `rxbridge-enrollment-service` | Patients and enrollments REST API | planned |

## Getting started

_Coming soon._

```bash
git clone --recurse-submodules <repository-url>
```

## Roadmap

- [ ] Stage 1: enrollment-service (REST API, JPA, Flyway, validation)
- [ ] Stage 2: web UI with Thymeleaf and htmx
- [ ] Stage 3: Spring Security basics
- [ ] Stage 4: adverse-event-service, inter-service calls, resilience
- [ ] Stage 5: Salesforce integration (JWT Bearer flow, Composite API)
- [ ] Stage 6: transactional outbox and Kafka
- [ ] Stage 7: bulk CSV import with Spring Batch
- [ ] Stage 8: Spring Cloud (Config Server, Eureka, Gateway)
- [ ] Stage 9: Keycloak, OAuth2 and BFF
- [ ] Stage 10: notification-service in Kotlin
- [ ] Stage 11: inventory-service and saga
- [ ] Stage 12: Salesforce Pub/Sub API (Change Data Capture over gRPC)
- [ ] Stage 13: observability (tracing, metrics, logs)
- [ ] Stage 14: CI/CD, contract tests, documentation

The detailed plan is in [docs/PLAN.md](docs/PLAN.md).
