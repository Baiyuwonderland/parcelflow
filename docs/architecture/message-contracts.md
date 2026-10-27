# ParcelFlow — Message Contracts

## Shared Envelope Schema (TypeScript)

All Kafka messages use this envelope structure:

```typescript
interface KafkaEventEnvelope {
  eventId:       string;       // UUID v4
  eventType:     EventType;    // "parcel.created" | "delivery.assigned"
  eventVersion:  number;       // 1
  correlationId: string;       // UUID v4 — links related events
  occurredAt:    string;       // ISO-8601 with Z suffix
  payload:       object;       // Event-specific payload (described below)
}
```

## Topic: `parcel.created`

**Publisher:** Parcel Service
**Subscribers:** Delivery Service, Notification Service

```typescript
interface ParcelCreatedPayload {
  parcelId:      string;       // MongoDB ObjectId string
  recipientName: string;       // e.g. "John Morris"
  deliveryAddress: string;     // e.g. "123 Main St, Sydney NSW"
  description:   string;       // e.g. "Small electronics package"
  status:        PENDING;      // Always "PENDING" on creation
  customerId?:   string;       // Optional future field
}
```

**Validation Rules (sender must enforce):**
- `recipientName`: 1–200 chars, no empty strings
- `deliveryAddress`: 1–500 chars, no empty strings
- `description`: 1–1000 chars

**Error Handling:** If MongoDB insert fails, log error and do NOT publish the event. Parcel is not created. If Kafka publish fails, log error — the parcel may be persisted without the event (documented limitation, see docs/definitions/parcelflow.md).

## Topic: `delivery.assigned`

**Publisher:** Delivery Service
**Subscribers:** Parcel Service, Notification Service

```typescript
interface DeliveryAssignedPayload {
  parcelId:     string;       // Matches the parcel's id
  driverName:   string;       // One of the three hard-coded drivers
  driverId?:    string;       // Optional
  estimatedEta?: string;      // e.g. "2 hours"
  status:       ASSIGNED;     // Always "ASSIGNED"
}
```

**Validation Rules (sender must enforce):**
- `parcelId`: must be non-empty
- `driverName`: one of `["Alex Chen", "Sam Okonkwo", "Maria García"]`

**Duplicate Protection:** Delivery Service stores deliveries by `parcelId`. If the same `parcelId` is seen again, the assignment is a no-op. Use a unique index on `parcelId` in the `deliveries` collection.

## Event Flow Summary

```
Event Type           Direction                          Duplication Guard
─────────────────────────────────────────────────────────────────────────────
parcel.created       Parcel → Delivery, Notification    Unique index on eventId
delivery.assigned    Delivery → Parcel, Notification    Unique index on parcelId (deliveries),
                                                          eventId (notifications)
```

## Correlation Tracking

Each `parcel.created` gets a `correlationId`. All downstream events and database records for that parcel use the same `correlationId`. This enables filtering by `parcelId` or `correlationId` in logs and queries.

Example trace for one parcel:

```
parcel.created  correlationId=abc-123  parcelId=parcel-001
  ↓
delivery.assigned correlationId=abc-123  parcelId=parcel-001  driverId=driver-002
  ↓
notification (parcel created)   correlationId=abc-123
notification (delivery assigned) correlationId=abc-123
```

## JSON Schema for Validation

Both service schemas and frontend JSON can reference these contracts. The actual runtime validation is done in TypeScript — the JSON schemas serve as the machine-readable contract for testing.

See the TypeScript interface blocks above as the canonical contract.
