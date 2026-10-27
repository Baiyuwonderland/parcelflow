# ParcelFlow — System Architecture

## High-Level Diagram

```
┌─────────────┐    ┌──────────────┐    ┌─────────────────┐    ┌──────────────┐    ┌──────────────────┐
│  Parcel Svc  │    │  Delivery Svc│    │Notification Svc │    │   Vue Frontend│    │  Splunk (opt-in) │
│  Port :3001  │    │  Port :3002  │    │   Port :3003    │    │  Port :5173   │    │  Port :8200/9886 │
└──────┬───────┘    └──────┬───────┘    └───────┬─────────┘    └──────┬───────┘    └──────────────────┘
       │                   │                      │                      │
       │     ┌─────────────┴──────────────────────┴─────────────────────┐
       │     │                      Kafka (KRaft)                        │
       │     │  - parcel.created                                        │
       │     │  - delivery.assigned                                     │
       │     └─────────────────────────────────────────────────────────┘
       │                   │                      │
 ┌──────┴───────┐    ┌─────┴───────┐      ┌──────┴─────────┐
 │  MongoDB     │    │  MongoDB    │      │   MongoDB       │
 │  parcels DB  │    │ deliveries │      │  notifications  │
 └──────────────┘    └─────────────┘      └────────────────┘
```

## Folder Structure (Monorepo)

```
ParcelFlow/
├── docker-compose.yml
├── docker-compose.kafka-only.yml     # Standalone Kafka dev
├── Dockerfile.node                  # Shared Node base image
├── package.json                     # Root monorepo package
├── packages/
│   ├── parcel-service/              # Service A
│   │   ├── src/
│   │   │   ├── routes/parcels.ts
│   │   │   ├── services/parcelService.ts
│   │   │   ├── services/parcelRepository.ts
│   │   │   ├── events/ParcelCreatedEvent.ts
│   │   │   ├── events/ParcelConsumer.ts
│   │   │   ├── db/index.ts
│   │   │   ├── kafka/producer.ts
│   │   │   ├── kafka/consumer.ts
│   │   │   ├── middleware/log.ts
│   │   │   └── index.ts             # Express app entry
│   │   ├── tests/
│   │   │   └── parcel.test.ts
│   │   ├── openapi/
│   │   │   └── schema.yaml
│   │   ├── package.json
│   │   └── tsconfig.json
│   ├── delivery-service/            # Service B
│   │   ├── src/
│   │   │   ├── routes/deliveries.ts
│   │   │   ├── services/deliveryService.ts
│   │   │   ├── services/deliveryRepository.ts
│   │   │   ├── events/DeliveryAssignedEvent.ts
│   │   │   ├── events/ParcelCreatedConsumer.ts
│   │   │   ├── db/index.ts
│   │   │   ├── kafka/consumer.ts
│   │   │   ├── kafka/producer.ts
│   │   │   ├── middleware/log.ts
│   │   │   └── index.ts
│   │   ├── tests/
│   │   │   └── delivery.test.ts
│   │   ├── openapi/schema.yaml
│   │   ├── package.json
│   │   └── tsconfig.json
│   └── notification-service/        # Service C
│       ├── src/
│       │   ├── routes/notifications.ts
│       │   ├── services/notificationService.ts
│       │   ├── services/notificationRepository.ts
│       │   ├── events/ParcelCreatedConsumer.ts
│       │   ├── events/DeliveryAssignedConsumer.ts
│       │   ├── db/index.ts
│       │   ├── kafka/consumer.ts
│       │   ├── middleware/log.ts
│       │   └── index.ts
│       ├── tests/
│       │   └── notification.test.ts
│       ├── openapi/schema.yaml
│       ├── package.json
│       └── tsconfig.json
├── apps/
│   └── parcel-ui/
│       ├── src/
│       │   ├── App.vue
│       │   ├── components/ParcelForm.vue
│       │   ├── components/ParcelList.vue
│       │   ├── components/ParcelDetail.vue
│       │   ├── components/NotificationHistory.vue
│       │   ├── composables/
│       │   │   └── useParcelApi.ts
│       │   └── main.ts
│       ├── tests/
│       │   └── parcelFlow.e2e.ts
│       ├── package.json
│       ├── tsconfig.json
│       └── vite.config.ts
├── docs/
│   ├── definitions/parcelflow.md
│   └── architecture/
├── CLAUDE.md
└── README.md
```

## Container Layout

| Service          | Image                    | Ports | Volume           | Env Vars                |
|------------------|--------------------------|-------|------------------|-------------------------|
| parcel-service   | parcelflow/parcel-svc    | 3001  | /app/data        | MONGO_URL, KAFKA_BROKER |
| delivery-service | parcelflow/delivery-svc  | 3002  | /app/data        | MONGO_URL, KAFKA_BROKER |
| notification-svc | parcelflow/notification-svc| 3003 | /app/data        | MONGO_URL, KAFKA_BROKER |
| mongodb          | mongo:7                  | 27017 | vol-mongo         | MONGO_INITDB_ROOT…       |
| kafka            | confluentinc/kafka-gpl   | 9092  | vol-kafka         | KAFKA_CFG_…              |
| fluentd          | fluent/fluentd           | 24224 | vol-fluentd       | SPLUNK_HEC_TOKEN         |
| splunk           | splunk/splunk            | 8200/9886| vol-splunk      | SPLUNK_PASSWORD          |

## Shared Configs

- `Dockerfile.node` — pinned Node 22 LTS slim, multi-stage build
- Root `package.json` — workspaces for `packages/*` and `apps/*`
- All services share the same base Dockerfile; service-specific config injected at runtime via env vars
