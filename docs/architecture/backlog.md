# ParcelFlow — Implementation Backlog

Each ticket is designed for a single implementation pass by a 35B coding model.
Order is sequential — later tickets depend on earlier ones.

---

## Phase 1: Foundation

### Ticket 1 — Project Scaffold (S)

What: Set up monorepo structure with root `package.json` (workspaces), shared `tsconfig.json`, Dockerfile.node base image.

Acceptance:
- `npm install` from root installs all workspace deps without errors
- 3 service package.json files under `packages/`
- 1 frontend package.json under `apps/parcel-ui/`
- Build passes for an empty Express service in each

Verification:
```bash
npm install
npm run build parcel-service   # compiles empty main.ts
```

Files: `package.json`, `tsconfig.json`, `Dockerfile.node`, `packages/*/package.json`
Dependencies: None
Effort: S

---

### Ticket 2 — Docker Compose Infrastructure (S)

What: `docker-compose.yml` with MongoDB (single container, 3 collections) and Kafka (KRaft, single broker). `init-mongo.js` for collection setup.

Acceptance:
- `docker compose up --build -d` starts `mongodb` and `kafka` containers
- `docker ps` shows both as healthy
- MongoDB is accessible on `localhost:27017`
- Kafka topics can be listed via `kafka-topics.sh --bootstrap-server ... --list`

Verification:
```bash
docker compose up --build -d
docker ps --filter name=pf-mongodb --filter name=pf-kafka
docker exec pf-mongodb mongosh --eval 'db.runCommand("ping")'
docker exec pf-kafka kafka-topics.sh --bootstrap-server localhost:9092 --list
```

Files: `docker-compose.yml`, `init-mongo.js`, `docker-compose.kafka-only.yml`
Dependencies: Ticket 1
Effort: S

---

## Phase 2: Parcel Service

### Ticket 3 — Parcel Service: Create + List API + Swagger (M)

What: Parcel Service Express app with MongoDB connectivity, `POST /parcels` endpoint, `GET /parcels` list endpoint, `GET /parcels/:id` by ID, `/health`, `/api-docs` (Swagger UI), and `/api` (API spec).

Acceptance:
- `GET /health` returns `200 OK` with service name and MongoDB/Kafka connection status
- `POST /parcels` accepts `{ recipientName, deliveryAddress, description }` and returns the created parcel with `status: "PENDING"`
- `GET /parcels` returns an array of all parcels
- `GET /parcels/:id` returns the parcel or `404`
- Swagger UI renders at `/api-docs` with documented endpoints
- All inputs validated — empty strings rejected with `400`

Verification:
```bash
curl -s http://localhost:3001/health | jq .
curl -X POST http://localhost:3001/parcels -H 'Content-Type: application/json' \
  -d '{"recipientName":"Jane Doe","deliveryAddress":"5 Wall St, Sydney","description":"Small box"}'
curl -s http://localhost:3001/parcels | jq .
```

Expected Files:
```
packages/parcel-service/src/
  routes/parcels.ts
  services/parcelRepository.ts
  services/parcelService.ts
  kafka/producer.ts
  middleware/log.ts
  index.ts
```

Dependencies: Ticket 2
Effort: M

---

### Ticket 4 — Parcel Service: Component Tests (S)

What: Playwright tests for Parcel Service that exercise create, list, get by ID, validation errors, and health checks. Uses a real local MongoDB instance.

Acceptance:
- All tests pass on first run
- Tests verify successful parcel creation
- Tests verify validation errors (empty strings return `400`)
- Tests verify list return an array
- Tests verify `GET /parcels/:id` returns `404` for absent IDs
- No `sleep()` calls; use polling for async readiness

Verification:
```bash
cd packages/parcel-service
npm test    # playwright test --config=playwright.config.ts
```

Expected Files:
```
packages/parcel-service/tests/parcel.test.ts
packages/parcel-service/playwright.config.ts
```

Dependencies: Ticket 3
Effort: S

---

## Phase 3: Delivery Service

### Ticket 5 — Delivery Service: Kafka Consumer + Driver Assignment + Producer (M)

What: Delivery Service Express app, MongoDB connectivity, Kafka consumer for `parcel.created`, deterministic driver assignment (hash-based modulo), `delivery.assigned` event producer, and `GET /deliveries` endpoints.

Acceptance:
- Delivery Service starts and connects to MongoDB + Kafka
- On consuming `parcel.created`, it assigns a driver via `hash(parcelId) % 3` from `[Alex Chen, Sam Okonkwo, Maria Garcia]`
- Delivers the assignment to MongoDB
- Produces `delivery.assigned` event to Kafka
- `GET /deliveries` lists all deliveries
- `GET /deliveries/:id` returns a specific delivery or `404`
- Duplicate `parcel.created` for the same parcelId is idempotent (no-second record)

Verification:
```bash
# Create a parcel (triggers parcel.created)
curl -X POST http://localhost:3001/parcels -H 'Content-Type: application/json' \
  -d '{"recipientName":"Test","deliveryAddress":"123 St","description":"Box"}'

# Wait ~3-5 seconds for delivery service to consume
# Then:
curl -s http://localhost:3002/deliveries | jq .
```

Expected Files:
```
packages/delivery-service/src/
  routes/deliveries.ts
  services/deliveryRepository.ts
  services/deliveryService.ts
  events/ParcelCreatedConsumer.ts
  kafka/consumer.ts
  kafka/producer.ts
  middleware/log.ts
  index.ts
```

Dependencies: Ticket 2 (infra), Ticket 3 (parcel.created contract)
Effort: M

---

### Ticket 6 — Delivery Service: Component Tests (S)

What: Playwright tests for Delivery Service — verify parcel.created consumption, delivery persistence, delivery.assigned production, and duplicate handling.

Acceptance:
- Tests create a test parcel and verify delivery is created
- Tests verify driver name is one of the three expected values
- Tests verify idempotent behavior on duplicate parcel.created
- All tests pass without `sleep()`

Verification:
```bash
cd packages/delivery-service
npm test
```

Expected Files:
```
packages/delivery-service/tests/delivery.test.ts
```

Dependencies: Ticket 5
Effort: S

---

## Phase 4: Notification Service

### Ticket 7 — Notification Service: Kafka Consumers + CRUD API (M)

What: Notification Service consumes `parcel.created` and `delivery.assigned`, stores simulated notifications, and exposes `GET /notifications` and `GET /notifications?parcelId=...`.

Acceptance:
- Notification Service starts, connects to MongoDB + Kafka
- Consuming `parcel.created` creates a notification: "Parcel created successfully for [recipientName]"
- Consuming `delivery.assigned` creates: "Driver [driverName] assigned to parcel [parcelId]"
- Both notifications link to the same `correlationId`
- Duplicate handling via unique `eventId` (no double notifications)
- `GET /notifications` lists all notifications
- `GET /notifications?parcelId=...` filters by parcel ID
- `/health`, `/api-docs` endpoints present

Verification:
```bash
curl -s http://localhost:3003/notifications | jq .
curl -s "http://localhost:3003/notifications?parcelId=abc" | jq .
```

Expected Files:
```
packages/notification-service/src/
  routes/notifications.ts
  services/notificationRepository.ts
  services/notificationService.ts
  kafka/consumer.ts
  middleware/log.ts
  index.ts
```

Dependencies: Ticket 2 (infra), Ticket 3 (parcel.created), Ticket 5 (delivery.assigned)
Effort: M

---

### Ticket 8 — Notification Service: Component Tests (S)

What: Playwright tests for Notification Service — verify both event consumptions, notification persistence, duplicate handling, and filtering.

Acceptance:
- Test creates parcel via Parcel Service, waits for Kafka, verifies 2 notifications created
- Test verifies filtering by parcelId returns only related notifications
- All tests pass without `sleep()`

Verification:
```bash
cd packages/notification-service
npm test
```

Expected Files:
```
packages/notification-service/tests/notification.test.ts
```

Dependencies: Ticket 7
Effort: S

---

## Phase 5: Parcel Service Kafka Consumer (Integration Bridge)

### Ticket 9 — Parcel Service: Consume delivery.assigned (M)

What: Parcel Service consumer that subscribes to `delivery.assigned`, updates the corresponding parcel's status to `ASSIGNED` and sets the `assignedDriver` field. The parcel's MongoDB record should now include driver info.

Acceptance:
- Parcel Service consumes `delivery.assigned` from Kafka
- Parcel with matching `parcelId` is updated: `status: "ASSIGNED"`, `assignedDriver: "<name>"`
- Parcel `GET /parcels/:id` returns the updated record after consumption
- Duplicate `delivery.assigned` is idempotent (parcel stays `ASSIGNED`, not toggled)
- Log entry for the status update

Verification:
```bash
# Create parcel (status = PENDING)
curl -X POST http://localhost:3001/parcels -H 'Content-Type: application/json' \
  -d '{"recipientName":"Test","deliveryAddress":"123 St","description":"Box"}'

# Wait ~5-10 seconds for full flow: parcel.created → delivery → delivery.assigned → parcel update
curl -s http://localhost:3001/parcels | jq '.[] | select(.recipientName=="Test")'
# Expected: status = "ASSIGNED", assignedDriver present
```

Expected Files:
```
packages/parcel-service/src/events/DeliveryAssignedConsumer.ts
packages/parcel-service/src/kafka/consumer.ts
```

Dependencies: Ticket 5 (delivery.assigned producer), Ticket 3 (Parcel Service base)
Effort: M

---

### Ticket 10 — Parcel Service: Integration + Component Tests (M)

What: Updated component tests that verify the full parcel lifecycle within Parcel Service: creation → Kafka event → (mock event injection or direct) delivery.assigned consumption → status update to `ASSIGNED`.

Acceptance:
- Component tests create a parcel
- Tests verify `parcel.created` was produced (by checking logs or a test consumer)
- Tests inject a `delivery.assigned` event and verify parcel status → `ASSIGNED`
- All new and existing tests pass

Verification:
```bash
cd packages/parcel-service
npm test
```

Dependencies: Ticket 9
Effort: M

---

## Phase 6: Frontend

### Ticket 11 — Vue.js Parcel UI (M)

What: Vue.js 3 + Vite app with Composition API, four components (ParcelForm, ParcelList, ParcelDetail, NotificationHistory), and `useParcelApi` composable.

Acceptance:
- `npm run dev` starts frontend on `localhost:5173`
- Parcel Form: form with fields for recipientName, deliveryAddress, description; submit creates parcel via Parcel Service API
- Parcel List: shows all parcels with status badges (PENDING/ASSIGNED)
- Parcel Detail: click a parcel → sees status, driver, timestamps
- Notification History: select a parcel → shows its notifications
- No Pinia, no UI framework — plain CSS or scoped styles only
- Uses native `fetch` for all API calls

Verification:
```bash
npm run dev --workspace apps/parcel-ui
# Open http://localhost:5173 in browser
# Test: fill form, submit, see parcel in list, click parcel, see notifications
```

Expected Files:
```
apps/parcel-ui/src/
  App.vue
  components/ParcelForm.vue
  components/ParcelList.vue
  components/ParcelDetail.vue
  components/NotificationHistory.vue
  composables/useParcelApi.ts
  main.ts
```

Dependencies: Ticket 3 (Parcel Service API), Ticket 5 (Delivery Service)
Effort: M

---

### Ticket 12 — Frontend Integration: Parcel UI + Full System Flow (S)

What: Wire the Vue frontend to the running ParcelFlow system end-to-end. Parcel form submission → parcel appears in list → auto-assigns driver → status changes → notifications appear. Add simple Playwright component tests for the frontend.

Acceptance:
- User creates a parcel via frontend form
- Parcel appears in list with `PENDING` status
- After ~5 seconds, status changes to `ASSIGNED` with driver name
- Notification cards appear for "parcel created" and "driver assigned"
- End-to-end flow completes without errors

Verification:
```bash
# With all services running:
npm run dev --workspace apps/parcel-ui
# Manual verification: create parcel, watch it transition to ASSIGNED
```

Dependencies: Ticket 11
Effort: S

---

## Phase 7: Full-System E2E Tests

### Ticket 13 — Playwright Full-System E2E Tests (L)

What: Playwright E2E test that runs against the full Docker Compose system. Verifies the complete business journey: create parcel → wait for Kafka → check parcel status → check delivery → check notifications.

Acceptance:
- `npm test:e2e` in the frontend workspace runs a Playwright test
- Test creates a parcel via Parcel Service REST API (or frontend)
- Test polls parcel status until it reaches `ASSIGNED` (bounded, no fixed sleep)
- Test verifies a delivery record exists for the parcel
- Test verifies notification history for the parcel
- Test passes on first run (no flaky sleeps)

Verification:
```bash
npm run test:e2e --workspace apps/parcel-ui
```

Expected Files:
```
apps/parcel-ui/tests/parcelFlow.e2e.ts
apps/parcel-ui/playwright.config.ts
```

Dependencies: Ticket 10 (full lifecycle), Ticket 11 (frontend running)
Effort: L

---

## Phase 8: Observability

### Ticket 14 — Structured Logging + Splunk Forwarder (M)

What: Implement structured JSON logging in all three services. Each log entry includes `timestamp`, `level`, `service`, `message`, `correlationId`, `eventId`, `parcelId`. Add Fluentd container for log forwarding. Add Splunk container as an optional profile.

Acceptance:
- `docker logs parcel-service` shows structured JSON lines (parseable with `jq .`)
- All business events logged: parcel created, event published, event consumed, delivery assigned, notification recorded
- Fluentd container forwards logs to Splunk
- Splunk is optional — starts only with `--profile splunk-dbg`
- Fluentd handles log rotation or truncation

Verification:
```bash
# Without Splunk profile:
docker compose up --build -d
docker logs parcel-service | jq .timestamp    # should work

# With Splunk profile:
docker compose --profile splunk-dbg up --build -d
curl http://localhost:8000/api/search  # Splunk UI accessible
```

Files:
```
packages/*/src/middleware/log.ts
fluentd/
  fluent.conf
  Dockerfile
docker-compose.splunk.yml   (or profiles in main compose)
```

Dependencies: Tickets 1-11 (services running)
Non-goals: No Splunk setup in committed source files (env vars only). No dead-letter queues. No metrics or dashboards.

Effort: M

---

### Ticket 15 — README + Architecture Review (S)

What: Comprehensive README.md with startup instructions, architecture overview, testing guide, and troubleshooting. Link to architecture docs and product definition.

Acceptance:
- `README.md` explains how to start the system (`docker compose up`)
- Lists all service URLs and Swagger endpoints
- Documents testing commands (component tests, E2E tests)
- Explains the event flow with a simple diagram
- Documents the Docker Compose profiles (default, splunk-dbg, kafka-only)
- Links to `docs/definitions/parcelflow.md` and `docs/architecture/`

Verification:
```bash
cat README.md
# Verify all sections readable, commands executable, URLs correct
```

Dependencies: All tickets above
Effort: S

---

## Summary Table

| # | Ticket | Effort | Service | Key Outcome |
|---|--------|--------|---------|-------------|
| 1 | Project Scaffold | S | All | Monorepo + TypeScript + Dockerfile |
| 2 | Docker Compose Infra | S | All | MongoDB + Kafka (KRaft) running |
| 3 | Parcel Service API | M | Parcel | POST/GET/health + Swagger |
| 4 | Parcel Service Tests | S | Parcel | Component tests pass |
| 5 | Delivery Service | M | Delivery | Consumer → assign driver → producer |
| 6 | Delivery Service Tests | S | Delivery | Idempotent delivery tests pass |
| 7 | Notification Service | M | Notification | Consumer both events + CRUD API |
| 8 | Notification Tests | S | Notification | Duplicate handling verified |
| 9 | Parcel Service Kafka Consumer | M | Parcel | Consumes delivery.assigned, sets ASSIGNED |
| 10 | Parcel Integration Tests | M | Parcel | Full lifecycle in tests |
| 11 | Vue.js Parcel UI | M | Frontend | 4 components, fetch-based |
| 12 | Frontend Integration | S | Frontend | Flow works in browser |
| 13 | Full-System E2E Tests | L | All | Playwright test passes end-to-end |
| 14 | Structured Logging + Splunk | M | All | JSON logs, optional Splunk |
| 15 | README + Review | S | All | Startup, test, arch docs |

**Total tickets: 15 | Total effort: ~6M, 7S, 1L**

---

## Implementation Order Notes

1. **Tickets 1–2 first**: You cannot build anything without the monorepo scaffold and running infra.
2. **Ticket 3 before 4**: Parcel API must exist before testing it.
3. **Ticket 5 before 6**: Delivery Service needs a running Parcel Service producing events.
4. **Ticket 7 after 5**: Notification Service needs both `parcel.created` and `delivery.assigned` contracts.
5. **Ticket 9 after 5**: Parcel Service consumes `delivery.assigned` from Delivery Service.
6. **Ticket 10 after 9**: Full lifecycle test in Parcel Service depends on the consumer.
7. **Ticket 11 after 3–5**: Frontend consumes all three service APIs.
8. **Ticket 13 last**: Full-system E2E requires all services running correctly.
9. **Ticket 14 and 15 last**: Observability and documentation are finishing touches.

## Handoff Notes for Each Ticket

- Each ticket is self-contained in its service folder.
- No ticket modifies files in another service's folder (except README at the end).
- Message contracts are in `docs/architecture/message-contracts.md` — the canonical source.
- Docker Compose layout in `docs/architecture/docker-compose-layout.md`.
- Kafka topics are defined and do not change.

---

*Approved / Not Approved? Reply "Approved" to proceed with implementation of Ticket 1.*
