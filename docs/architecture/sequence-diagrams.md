# ParcelFlow — Sequence Diagrams

## Primary Flow: Create Parcel → Delivery → Notification

```
User     ParcelSvc     Kafka        DeliverySvc     Kafka        NotificationSvc
 │         │            │               │             │                │
 │──POST /parcels────────────────────────────────────────────────────────
 │         │──validate payload───────────│
 │         │──insert parcel (MONGO)──────│
 │         ││                            │
 │         ││                            │
 │         │──publish parcel.created───────►│
 │         │                            │──consume parcel.created─────│
 │         │                            │       │assign driver (deterministic)
 │         │                            │       │insert delivery (MONGO)
 │         │                            │       │publish delivery.assigned───|
 │         │                            │       │                            │──consume parcel.created
 │         │──consume delivery.assigned──┤       │                            │       │insert notification
 │         │──update parcel status───────┤       │                            │       │──consume delivery.assigned
 │         │                            │       │                            │       │insert notification
 │         │                            │       │                            │
 │──GET /parcels/parcelId───────────────────────│                            │
 │◄──parcel (ASSIGNED)─────────────────────│
 │         │                            │       │                            │──GET /notifications...
 │◄──delivery assigned notification─┘       │                            │       │◄──notifications
 │◄──parcel details + status───────────────│
 │
```

## Detail: Deterministic Driver Assignment

```
┌────────────────────────────────────────────────────┐
│ Delivery Service: Assignment Strategy               │
├────────────────────────────────────────────────────┤
│ Drivers (hard-coded array, index-modulo):           │
│   [0] "Alex Chen"                                    │
│   [1] "Sam Okonkwo"                                  │
│   [2] "Maria García"                                 │
│                                                     │
│ For parcel N:                                        │
│   driverIndex = hash(parcelId) % 3                  │
│   driver = drivers[driverIndex]                      │
│                                                     │
│ This gives consistent results per parcelId           │
│ without any external routing logic.                  │
└────────────────────────────────────────────────────┘
```

## Health Check Pattern (All Services)

```
User         Service
 │            │
 │──GET /health───────────────────│
 │◄──200 OK──────────────────────│
 │    { "status": "healthy",
 │      "service": "parcel-service",
 │      "components": {
 │        "mongo": { "status": "ok" },
 │        "kafka": { "status": "ok" }
 │      }
 │    }
 │
 │──GET /api-docs─────────────────│
 │◄──200 UI───────────────────────│   (OpenAPI/Swagger page)
 │
```

## Kafka Consumer Error Recovery (Simplified)

```
Consumer     MongoDB/Kafka
  │            │
  │──consume event────────────────│
  │   process payload            │
  │   │
  │   │──insert to MongoDB───────│
  │   │   fail ─────────────────►│   { "error": "duplicate key" }
  │   │   OK────────────────────│
  │   │                        │
  │   │──publish next event──────│
  │   │   fail ─────────────────│   ConnectionRefusedError
  │   │   OK────────────────────│
  │                           │
  │   handle error ──► log it ──│
  │   do NOT re-throw on dup     │
  │   bounded retries: max 3,    │
  │   1s between tries          │
  │   then give up & log         │
```

**Key principle:** On duplicate (unique index violation), log and treat as success. On transient failure, retry up to 3 times then log. On permanent failure (bad data), log and skip. No dead-letter queue for this learning project.
