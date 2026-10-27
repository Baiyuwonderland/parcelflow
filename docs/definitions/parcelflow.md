# ParcelFlow — Product Definition

## 1. Problem and Intended Users

**Problem:** Learning, demonstrating, and teaching event-driven microservice architecture with concrete observable outcomes.

**Intended Users:**

| User | Needs |
|------|-------|
| Developer / Student | A complete, runnable example of event-driven microservices using Kafka, MongoDB, and Express |
| Engineering Manager | A reference architecture for planning real event-driven systems |
| Quality Engineer | Playwright component and E2E tests that exercise the full asynchronous flow |

No real customers or production traffic — this is a learning and demonstration project.

---

## 2. Business Objectives

1. Build a fully local, runnable microservice application that demonstrates event-driven architecture end-to-end.
2. Show how services communicate through Kafka without direct database sharing.
3. Provide a working UI (Vue.js) that lets users observe the full parcel lifecycle.
4. Produce structured JSON logs, viewable in a local Splunk instance.
5. Deliver Playwright tests proving the system works as designed.

---

## 3. In-Scope

- Three Node.js/TypeScript/Express microservices (Parcel, Delivery, Notification).
- Apache Kafka (KRaft mode, single-broker) for event transport.
- MongoDB (single container, three collections).
- Vue.js 3 single-page frontend.
- Swagger/OpenAPI documentation for each service.
- Structured JSON logging with Splunk optional forwarder.
- Playwright component + full-system E2E tests.
- Docker Compose single-command startup.
- Apple Silicon (arm64) support.

---

## 4. Out-of-Scope (Non-Goals)

- Kubernetes, Docker swarm, or any orchestration beyond Docker Compose.
- Authentication, authorization, or API keys.
- Redis, API gateways, service discovery, CQRS, schema registries, distributed tracing.
- Real email/SMS providers, payment gateways, external delivery APIs, Google Maps.
- Scalability beyond a single broker and single service replica.
- CI/CD pipelines (tests run locally).
- Production-grade security, secrets management, or audit logging.

---

## 5. Core End-to-End User Journey

```
User                    Parcel Service               Kafka                          Delivery Service              Notification Service
  |                           |   parcel.created       |                                    |                               |
  |── POST /parcels ───────►  |  ───────────────────►  |  ─────────────────────►                  |                               |
  |                           |                        |       delivery.assigned  ───────────►   |                               |
  |                           |                        |                                   |   ──────────────────────►   |
  |                           |                        |                                   |   record notification            |
  |                           |                        |       delivery.assigned  ◄──────────   |                               |
  |                           |   ───────────────────  |  ◄────────────────────         |                               |
  |                           |   update status        |   update delivery                         |
  |                           |   ASSIGNED             |   ASSIGNED                              |
  |                           |                        |                                    |                               |
  |── Get parcel ────────────►  |  ◄──────────────────  |                                    |                               |
  |── View notifications ──────────────────────────────────────────────────────────────────►  |   ◄─────────────────────   |
  |                           |                        |                                    |                               |
```

**Step-by-step:**

1. User submits a parcel creation form (recipient, address, description).
2. Parcel Service persists parcel as `PENDING` and publishes `parcel.created`.
3. Delivery Service consumes `parcel.created`, assigns a hard-coded driver, persists a delivery record, and publishes `delivery.assigned`.
4. Notification Service consumes both `parcel.created` and `delivery.assigned`, creating two notification records.
5. Parcel Service consumes `delivery.assigned` and updates parcel status to `ASSIGNED`.
6. User opens Vue frontend, sees parcel in ASSIGNED state, checks delivery details, and views notification history.

---

## 6. Service Responsibilities

### Service A — Parcel Service

| Responsibility |
|----------------|
| Create a parcel (`POST /parcels`) |
| List parcels (`GET /parcels`) |
| Get parcel by ID (`GET /parcels/:id`) |
| Own `parcels` MongoDB collection |
| Produce `parcel.created` event |
| Consume `delivery.assigned` event and update parcel status/driver |

### Service B — Delivery Service

| Responsibility |
|----------------|
| Consume `parcel.created` event |
| Assign one of three hard-coded demo drivers (deterministic) |
| Own `deliveries` MongoDB collection |
| Produce `delivery.assigned` event |
| List deliveries (`GET /deliveries`) |
| Get delivery by ID (`GET /deliveries/:id`) |

### Service C — Notification Service

| Responsibility |
|----------------|
| Consume `parcel.created` event |
| Consume `delivery.assigned` event |
| Own `notifications` MongoDB collection |
| Store simulated notification records |
| List notifications (`GET /notifications`) |
| Query notifications by parcelId (`GET /notifications?parcelId=...`) |

**Golden rule:** Services never access another service's MongoDB. Communication is Kafka-only.

---

## 7. Required Event-Driven Behaviour

### Kafka Topics

| Topic | Producer(s) | Consumer(s) |
|-------|-------------|-------------|
| `parcel.created` | Parcel Service | Delivery Service, Notification Service |
| `delivery.assigned` | Delivery Service | Parcel Service, Notification Service |

### Event Envelope (shared)

```json
{
  "eventId":       "uuid-v4",
  "eventType":     "parcel.created | delivery.assigned",
  "eventVersion":  1,
  "correlationId": "uuid-v4",
  "occurredAt":    "ISO-8601",
  "payload":       { /* service-specific fields */ }
}
```

### Duplication Tolerance

- Each service uses unique indexes (event IDs) to guard against duplicate delivery records, duplicate notifications, and duplicate business effects.
- No claims of exactly-once across MongoDB + Kafka. Document that at-least-once applies.
- Use bounded, explicit error handling with meaningful log statements. No complex retry framework.

---

## 8. Technology Constraints

| Category | Choice |
|----------|--------|
| Language | TypeScript (.ts) |
| Runtime | Node.js LTS |
| Backend framework | Express.js |
| Frontend | Vue.js 3 + Vite + Composition API + native fetch |
| Database | MongoDB (single container) |
| Message queue | Apache Kafka (KRaft, single broker) |
| Client library | KafkaJS |
| API docs | OpenAPI / Swagger UI |
| Logging | Structured JSON |
| Observability | Splunk (optional, via profile) |
| Testing | Playwright (component + E2E) |
| Packaging | Docker Compose (arm64 compatible) |
| Monorepo | Yes (single repo, services in folders) |

### Explicitly Excluded

Kubernetes, NestJS, Redis, API gateways, service discovery, CQRS frameworks, schema registries, distributed tracing, real email/SMS, cloud infrastructure, complex dependency injection, authentication.

---

## 9. Testing Expectations

### Component Tests (Per Service)

| Test Area | What It Verifies |
|-----------|------------------|
| Parcel Service | Creation succeeds; validation rejects invalid input; get and list work; `parcel.created` emitted; `delivery.assigned` consumption updates status. |
| Delivery Service | Processes `parcel.created`; persists assignment; emits `delivery.assigned`; handles duplicate `parcel.created` gracefully. |
| Notification Service | Processes both events; persists notifications; avoids duplicate notifications. |

- Use real local Kafka + MongoDB for component tests where practical.
- No arbitrary `sleep()` calls; use bounded polling.
- Isolated test data, unique IDs.

### Full-System E2E Tests

| Test | What It Verifies |
|------|------------------|
| Browser-based E2E | Open Vue app → create parcel → wait for driver assignment → verify parcel status is `ASSIGNED` → verify delivery exists → verify notification history. |
| API-based E2E | Use Playwright `APIRequestContext` to drive the same flow via REST endpoints. |

- Use real MongoDB and real Kafka. No stubbing in full-system E2E tests.
- At least one test runs in a real browser.

---

## 10. Operational and Local Resource Constraints

| Constraint | Detail |
|------------|--------|
| Hardware | Apple Silicon Mac, 48 GB RAM |
| Startup | `docker compose up --build -d` |
| Kafka | KRaft mode (no ZooKeeper), single-broker |
| MongoDB | One container, three collections: `parcels`, `deliveries`, `notifications` |
| Splunk | **Opt-in** via Docker Compose profile (heavy) |
| Images | `linux/arm64` compatible |
| Secrets | No credentials in committed files; env vars only |

### Resource Awareness

MongoDB + Kafka + 3 Node services + Vue dev server + optional Splunk will consume 2–4 GB RSS. The project should be sized to stay comfortably under 8 GB total to avoid swapping. Splunk runs only when explicitly requested.

---

## 11. Success Criteria

The application is **Done** when **all** of the following hold:

| # | Criterion | Verification |
|---|-----------|-------------|
| 1 | `docker compose up --build -d` starts all services, Kafka, and MongoDB | `docker ps` shows healthy containers |
| 2 | Parcel Service Swagger UI at `http://localhost:3001/api-docs` | GET returns 200 |
| 3 | Delivery Service Swagger UI at `http://localhost:3002/api-docs` | GET returns 200 |
| 4 | Notification Service Swagger UI at `http://localhost:3003/api-docs` | GET returns 200 |
| 5 | Vue frontend at `http://localhost:5173` renders | Browser loads parcel creation form |
| 6 | Creating a parcel via UI or API publishes `parcel.created` | Kafka topic check or log inspection |
| 7 | Parcel status transitions PENDING → ASSIGNED | UI or API shows `ASSIGNED` |
| 8 | A delivery record exists for the parcel | `GET /deliveries` returns it |
| 9 | Notification records exist for the parcel | `GET /notifications?parcelId=...` returns them |
| 10 | All component Playwright tests pass | `npm test` passes for every service |
| 11 | Full-system E2E Playwright test passes | Same command in full-system mode |
| 12 | Structured JSON logs produced | `docker logs` contain valid JSON with timestamps, levels, service names |
| 13 | Splunk (opt-in) is functional | Logs queryable by `parcelId` or `correlationId` |
| 14 | README documents startup, testing, and architecture | Readable and accurate |

---

## 12. Significant Technical Risks

| Risk | Impact | Mitigation |
|------|--------|------------|
| KafkaJS consumer groups on Apple Silicon arm64 | Build failure | Use `confluentinc/kafka-gpl` images with `linux/arm64` support |
| MongoDB single-container sharing across three logical databases | Data leakage between services | Enforce collection ownership per service in tests; document the risk |
| Event ordering under rapid production/consumption | Race conditions in tests | Use `correlationId` for linking; add bounded-polling assertions |
| Duplicate Kafka deliveries (at-least-once semantics) | Duplicate records | Unique indexes on `eventId` per service |
| Splunk resource consumption (JVM-based) | System OOM on lower-RAM Macs | Make Splunk opt-in via Docker Compose profiles |
| Playwright E2E test flakiness with Kafka latency | False-negative test failures | Replace fixed waits with polling; add `--retries 2` to test runner |
| Vue dev server + 3 Express services + Kafka + MongoDB under memory pressure | Swap / slowdown | Keep services lean; avoid unnecessary deps; use `docker compose` limits |

---

## 13. Recommended Minimal Three-Service Approach

We recommend **exactly three services** plus a frontend, as defined above:

1. **Parcel Service** — entry point; owns parcels; publishes/subscribes; the "glue" service.
2. **Delivery Service** — consumes parcel creation; assigns drive; publishes assignment.
3. **Notification Service** — a pure subscriber; records everything it hears.

This is the smallest set that demonstrates a complete **source → middleware observer → downstream subscriber → downstream consumer** flow while keeping responsibilities separable, testable, and understandable.

Adding a fourth service (e.g., "Address Validation" or "Payment") would break the constraint of implementability in small, independent tickets by a 35B local model. **Do not expand scope without explicit approval.**

---

## 14. Next Steps (After Approval)

1. **Architecture definition** — sequence diagrams, message contracts in full JSON, Docker Compose layout.
2. **Backlog planning** — split into individual GitHub issues / tickets, each with explicit acceptance criteria, expected files, and executable verification steps.
3. **Implementation** — one ticket at a time, verified after each.

**Do not proceed to architecture or backlog without approval of this definition.**

---

*Approved / Not Approved? Reply "Approved" to proceed.*
