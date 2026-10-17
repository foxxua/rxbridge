# RxBridge: Roadmap

Created 2026-10-12

## Repository structure

The umbrella repository `rxbridge` links the services as git submodules. Each repository is created when its stage begins.

| Repository | Contents | Language | Stage |
| --- | --- | --- | --- |
| `rxbridge` | Umbrella: submodules, `compose.yaml`, docs, plan, ADRs, Keycloak and Grafana configuration | — | 1 |
| `rxbridge-enrollment-service` | Patients, enrollments | Java | 1 |
| `rxbridge-web-ui` | Thymeleaf + htmx + Web Components, BFF | Java | 2 |
| `rxbridge-adverse-event-service` | Adverse event reports | Java | 4 |
| `rxbridge-salesforce-sync-service` | Salesforce integration, Pub/Sub API | Java | 5 |
| `rxbridge-events` | Shared event contracts library, published to GitHub Packages | Java | 6 |
| `rxbridge-import-service` | CSV import, Spring Batch | Java | 7 |
| `rxbridge-config` | Configuration of all services for Config Server | YAML | 8 |
| `rxbridge-config-server` | Spring Cloud Config Server | Java | 8 |
| `rxbridge-discovery-server` | Eureka | Java | 8 |
| `rxbridge-api-gateway` | Spring Cloud Gateway | Java | 8 |
| `rxbridge-notification-service` | Notifications, escalations | Kotlin | 10 |
| `rxbridge-inventory-service` | Warehouses, stock reservations, saga | Kotlin | 11 |

Working with submodules:
- Clone: `git clone --recurse-submodules`
- After committing in a service, make a separate commit in `rxbridge` with the updated submodule reference. This way the umbrella repository pins a compatible set of service versions.
- Push the service before pushing `rxbridge`, so the referenced commit exists on the remote.

## Principles

14 stages from simple to complex: each new technology is introduced only when working without it becomes painful. The goal is to understand not only *how* but also *why*.

- **Infrastructure is deferred.** Stages 1–7 have no Eureka, Gateway or Config Server: services find each other via addresses in `application.yml`. Spring Cloud arrives in stage 8, when there are 4–5 services and manual configuration becomes a burden.
- **Tests from the start.** Every stage includes tests for what it delivers, not "later".
- **Every stage ends with a working version** and a version tag in the changed repositories (`v0.1.0`, `v0.2.0`, …) and in `rxbridge`.
- **One repository per service**, each with its own Gradle build, combined in `rxbridge` via submodules. A new service means a new repository.
- **ADRs** (architecture decision records): a short note whenever choosing between approaches, 5–10 lines.
- Secrets live only in `.env`, which is git-ignored.

Each stage heading states its difficulty: easy, medium or hard.

## Stage 1: First service, enrollment-service (easy)

A single service with a REST API for patients and medication enrollments. The only infrastructure is PostgreSQL in `compose.yaml`, started by Spring Boot itself (`spring-boot-docker-compose`).

**Topics:** Spring Boot 4 project structure, Controller → Service → Repository layers, Spring Data JPA, Flyway, Bean Validation, Actuator basics, Java 25.

- [x] `rxbridge` repository (README skeleton, `.gitignore`, `docs/PLAN.md`)
- [x] `rxbridge-enrollment-service` repository (Gradle, Spring Initializr), added as a submodule
- [ ] `Patient`, `Enrollment`, `Medication` entities; Flyway migrations
- [ ] DTOs as `record`s; enrollment status as a `sealed interface` or `enum` with pattern-matching `switch`
- [ ] CRUD endpoints with pagination and sorting
- [ ] Validation: `@Valid`, a custom validator for the medication code format
- [ ] Error handling with `@RestControllerAdvice` and `ProblemDetail`
- [ ] `spring.threads.virtual.enabled=true`
- [ ] Swagger UI via springdoc-openapi
- [ ] Tests: unit (JUnit 5 + Mockito), `@WebMvcTest`, `@DataJpaTest` with Testcontainers
- [ ] Actuator: only `health` and `info` exposed; `info` shows the build version (`springBoot { buildInfo() }`) and git commit (`gradle-git-properties` plugin)

**Done when:** a patient and an enrollment can be created via Swagger UI and all tests pass.

## Stage 2: web-ui with Thymeleaf and htmx (easy)

A separate `rxbridge-web-ui` repository that calls enrollment-service directly at a configured address. No security and no Gateway yet.

**Topics:** Thymeleaf (templates, layouts, fragments), htmx, `RestClient` for calling another service.

- [ ] Pages: patient list, patient details, enrollment form
- [ ] Shared layout (header, menu) with Thymeleaf fragments
- [ ] htmx: search without page reload, pagination, form submission with validation errors
- [ ] `RestClient` with a base URL from `application.yml`; handling of service errors
- [ ] JS and CSS served locally (WebJars or `static/`), no CDN
- [ ] Controller tests with `@WebMvcTest` and a mocked client
- [ ] First ADR: "Why virtual threads and `RestClient` instead of WebFlux"

**Done when:** a patient can be found and enrolled in a medication program through the browser.

## Stage 3: Spring Security basics (medium)

Form login and roles in web-ui. Users are stored in the web-ui database, without Keycloak, to understand the Spring Security mechanics before adding complexity.

**Topics:** `SecurityFilterChain`, authentication vs authorization, `UserDetailsService`, `PasswordEncoder`, roles, CSRF, sessions.

- [ ] Login form; users with BCrypt passwords in the database
- [ ] `PHARMACY_OPERATOR` and `ADMIN` roles; URL restrictions and method-level `@PreAuthorize`
- [ ] Thymeleaf Extras Spring Security: show buttons by role
- [ ] CSRF token in htmx request headers
- [ ] Session cookie: `HttpOnly`, `Secure`, `SameSite`; Content Security Policy header
- [ ] Tests with `@WithMockUser` and `spring-security-test`

**Done when:** an operator can access only their own actions, an admin can access everything, and a request without a CSRF token is rejected.

## Stage 4: Second service and inter-service calls (medium)

A new `adverse-event-service` with its own database. When a report is created, it checks with enrollment-service that the patient exists: the first synchronous call between services and the first "what if the other service is down?" situation.

**Topics:** database per service, OpenFeign, Resilience4j (timeout, retry, circuit breaker), the first web component.

- [ ] `rxbridge-adverse-event-service` repository (submodule) with its own PostgreSQL database
- [ ] enrollment-service client via OpenFeign
- [ ] Resilience4j: timeout, retry, circuit breaker; a clear error when the service is unavailable
- [ ] Severity field and report processing deadline
- [ ] In web-ui: the first web component (Lit or plain JS), a report table with filters
- [ ] Client tests with WireMock: success, timeout, 500

**Done when:** with enrollment-service stopped, adverse-event-service responds with a controlled error instead of failing.

## Stage 5: Salesforce integration (medium)

A new `salesforce-sync-service`. For now data is sent via a plain HTTP call, without queues.

**Topics:** OAuth 2.0 JWT Bearer flow from Java, `RestClient` for an external API, working with API limits.

- [ ] `rxbridge-salesforce-sync-service` repository (submodule)
- [ ] Salesforce Developer Edition org with `Patient_Enrollment__c` and `Adverse_Event__c` objects and `External_Id__c`
- [ ] Connected App with a certificate; token via JWT Bearer flow, cached until expiry
- [ ] Upsert by External Id via Composite API (up to 25 records per request)
- [ ] Salesforce error handling: partial failures in Composite, 401 (refresh token), `REQUEST_LIMIT_EXCEEDED`
- [ ] enrollment-service calls sync-service directly after saving
- [ ] In web-ui: record sync status
- [ ] Tests with WireMock simulating Salesforce responses

**Done when:** an enrollment created in web-ui appears in the Salesforce org.

**Known problem:** if sync-service is unavailable at save time, the data is lost. This leads to stage 6.

## Stage 6: Outbox and Kafka (medium)

The direct call is replaced with reliable delivery: enrollment-service writes an event to an outbox table in the same transaction, and a separate process publishes it to Kafka. Kafka is added to `compose.yaml`, the first real piece of infrastructure.

**Topics:** transactional outbox, Spring Kafka, at-least-once delivery, idempotent consumers, dead letter topic, versioning a shared contracts library.

- [ ] `outbox_event` table, written in the same transaction as the business data
- [ ] Publisher: `@Scheduled` job reads the outbox and sends to Kafka
- [ ] `rxbridge-events` repository: events as `record`s, JSON format; published to GitHub Packages with semver
- [ ] Services depend on a versioned `rxbridge-events`; the GitHub token for reading packages comes from environment variables, not code
- [ ] sync-service consumes from Kafka; a processed-events table so duplicates are not processed twice
- [ ] Retry and dead letter topic for failed events
- [ ] Batching: accumulate events and send them to Composite API in groups
- [ ] Integration tests with Testcontainers Kafka
- [ ] ADR: "Why outbox instead of publishing directly to Kafka"

**Done when:** with sync-service stopped, 10 enrollments are saved; after restarting it, all 10 reach Salesforce with no duplicates.

## Stage 7: import-service with Spring Batch (medium)

Bulk CSV upload from pharmacies and distribution centers. With Kafka in place, the import simply publishes events, and the rest of the system already knows how to handle them.

**Topics:** Spring Batch (job, step, chunk, reader/processor/writer), skip and retry policies, restart after failure, large data volumes in Salesforce.

- [ ] `rxbridge-import-service` repository (submodule); file upload via web-ui
- [ ] Job: read CSV → validate → write → publish events
- [ ] Invalid rows are skipped and reported instead of failing the whole file
- [ ] Restart a job from the point of failure
- [ ] In sync-service: Bulk API 2.0 instead of Composite for large batches (chosen by record count)
- [ ] Synthetic test data generator, a 50,000-row file
- [ ] In web-ui: import progress (htmx polling or a web component)

**Done when:** a 50,000-row file is imported, invalid rows are visible in the error report, and the data is in Salesforce.

## Stage 8: Spring Cloud: Config, Eureka, Gateway (hard)

With 5 services, addresses and settings are scattered across `application.yml` files and ports must be remembered. Now Spring Cloud solves a real problem.

**Topics:** centralized configuration, service discovery, client-side load balancing, API Gateway, environment profiles, reactive basics (`Mono`/`Flux`) in Gateway filters.

- [ ] `rxbridge-config` repository: settings for all services, `local` and `docker` profiles
- [ ] `rxbridge-config-server` reads configuration from `rxbridge-config`
- [ ] `rxbridge-discovery-server` (Eureka); Feign clients call services by name, not address
- [ ] `rxbridge-api-gateway` (Spring Cloud Gateway, reactive WebFlux variant): routes, rate limiting, CORS
- [ ] Custom reactive `GlobalFilter`: correlation ID and request logging
- [ ] web-ui calls services only through the Gateway
- [ ] Run two instances of enrollment-service and verify load balancing
- [ ] Full `compose.yaml`: all services plus infrastructure, health checks and startup order
- [ ] ADR: "Eureka vs Kubernetes service discovery"

**Done when:** `docker compose up` starts the whole system with one command.

## Stage 9: Keycloak, OAuth2 and BFF (hard)

The stage 3 users are replaced with Keycloak. web-ui becomes a Backend for Frontend: tokens live on the server, and the browser holds only a session cookie.

**Topics:** OAuth2 / OpenID Connect, Authorization Code flow, resource server, token relay, roles from JWT.

- [ ] Keycloak in `compose.yaml`; realm, clients and roles imported from JSON at startup
- [ ] web-ui: `oauth2Login`, logout, roles from the token
- [ ] Gateway: token relay passes the token on to services
- [ ] Every service is a resource server that validates the JWT and roles
- [ ] Background calls between services (no user): client credentials flow for service accounts
- [ ] Tests with `jwt()` from `spring-security-test`
- [ ] ADR: "Why BFF instead of tokens in the browser"

**Done when:** there are no tokens in the browser (checked in DevTools), and a service returns 401 without a valid JWT.

## Stage 10: Kotlin notification-service (medium)

The first Kotlin service is deliberately simple: it consumes Kafka events and sends notifications, so the focus is on the language rather than on complex logic.

**Topics:** Kotlin syntax, null safety, data classes, extension functions, Spring Boot with Kotlin, using the Java `rxbridge-events` library from Kotlin.

- [ ] `rxbridge-notification-service` repository in Kotlin, Gradle Kotlin DSL
- [ ] Consumes "serious adverse event" and "enrollment approved" events
- [ ] Email delivery (Mailpit in `compose.yaml` for viewing emails) and webhooks
- [ ] Email templates
- [ ] Escalation: a report not processed by its deadline triggers a repeat notification to an admin
- [ ] Kotlin tests: JUnit 5 + MockK

**Done when:** creating a serious report produces an email in Mailpit, and a missed deadline produces an escalation.

## Stage 11: Saga with Kotlin inventory-service (hard)

Medication stock in distribution center warehouses. An enrollment is approved only after the medication is reserved; if no stock is available, the enrollment is rejected automatically. A distributed transaction without a shared database.

**Topics:** Saga pattern (choreography via events), compensating actions, optimistic locking, Kotlin coroutines (optional).

- [ ] `rxbridge-inventory-service` repository in Kotlin: warehouses, stock, reservations
- [ ] Event chain: `EnrollmentRequested` → `StockReserved` or `StockReservationFailed` → `EnrollmentApproved` or `EnrollmentRejected`
- [ ] Compensation: a cancelled enrollment returns the reserved stock
- [ ] Optimistic locking (`@Version`) so two requests cannot take the same unit
- [ ] Saga state visible in web-ui
- [ ] Test: 20 concurrent enrollments for 10 units, exactly 10 approved
- [ ] ADR: "Saga choreography vs orchestration"

**Done when:** the concurrency test passes consistently and compensation restores stock.

## Stage 12: Salesforce events via Pub/Sub API (hard)

Two-way integration: changes made in Salesforce (for example, a manager closing an adverse event report) flow back into the system.

**Topics:** gRPC in Java, Avro schemas, Change Data Capture, replay IDs, recovery after disconnects.

- [ ] Enable Change Data Capture for `Adverse_Event__c` in the org
- [ ] Pub/Sub API gRPC client: subscription, fetching and caching Avro schemas, decoding events
- [ ] Persist the last replay ID so restarts neither lose nor replay events
- [ ] Echo protection: changes sent by RxBridge itself do not loop back
- [ ] Salesforce events are published to Kafka and update adverse-event-service
- [ ] Reconnect with backoff on disconnect

**Done when:** changing a report status in Salesforce updates it in web-ui, and restarting sync-service loses no events.

## Stage 13: Observability: tracing, metrics, logs (medium)

When a request passes through 5 services and Kafka, tracing is the only way to see where it got lost.

**Topics:** Micrometer Tracing, trace ID propagation over HTTP and Kafka, custom metrics, structured logging.

- [ ] Micrometer Tracing + Zipkin: one request visible from web-ui to Salesforce
- [ ] Trace ID in every service's logs; logs in JSON format
- [ ] Prometheus + Grafana in `compose.yaml`
- [ ] Custom metrics: synced records count, Salesforce errors, remaining daily API limit
- [ ] Grafana dashboard exported to the repository
- [ ] Custom Actuator health indicators for Kafka and Salesforce connectivity

**Done when:** Zipkin shows the full path of an enrollment through all services, and Grafana shows a sync chart.

## Stage 14: CI/CD, contract tests, README (medium)

A project that a reviewer can start in 5 minutes and understand in 10.

**Topics:** GitHub Actions, Spring Cloud Contract, Docker images with Buildpacks or Jib.

- [ ] GitHub Actions in every service: build and test on every push, Testcontainers in CI
- [ ] GitHub Actions in `rxbridge`: start the whole system via `compose.yaml` and run an end-to-end smoke test
- [ ] Docker images in GitHub Container Registry
- [ ] Contract tests between enrollment-service and adverse-event-service
- [ ] Test coverage (JaCoCo) with a badge in the README
- [ ] README: overview, Mermaid architecture diagram, one-command startup, test logins
- [ ] `docs/adr` folder in `rxbridge` with all decisions
- [ ] Demo GIF or 2-minute video
- [ ] Profile for running without a Salesforce org (WireMock), so anyone can run the project

**Done when:** someone can clone the repository and start the system in 5 minutes without help.
