# ParcelFlow — Docker Compose Layout

## docker-compose.yml (Main)

```yaml
# services run with: docker compose up --build -d
# or: docker compose up --build -d --profile splunk-dbg

services:

  # MongoDB — shared physical instance, logical separation via collections
  mongodb:
    image: mongo:7
    container_name: pf-mongodb
    ports:
      - "27017:27017"
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: ${MONGO_ROOT_PASSWORD:-changeme}
    volumes:
      - vol-mongo:/data/db
      - ./init-mongo.js:/docker-entrypoint-initdb.d/init-mongo.js:ro
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongosh --quiet
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - parcelflow-net

  # Kafka — KRaft mode, single broker
  kafka:
    image: confluentinc/cp-kafka:7.7.1
    container_name: pf-kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_CLUSTER_ID: "LzUyLXl3U1CpJnGJLWIfSA"
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: controller,broker
      KAFKA_LISTENERS: INTERNAL://0.0.0.0:29092,CONTROLLER://0.0.0.0:29093,EXTERNAL://0.0.0.0:9092
      KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka:29092,EXTERNAL://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INTERNAL:PLAINTEXT,CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:29093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
    healthcheck:
      test: kafka-topics.sh --bootstrap-server localhost:9092 --list
      interval: 10s
      timeout: 5s
      retries: 5
    depends_on:
      - mongodb
    networks:
      - parcelflow-net

  # Parcel Service
  parcel-service:
    build:
      context: .
      dockerfile: Dockerfile.node
      target: parcel-service
    container_name: pf-parcel-service
    ports:
      - "3001:3001"
    environment:
      SERVICE_NAME: parcel-service
      EXPRESS_PORT: 3001
      MONGO_URL: mongodb://admin:${MONGO_ROOT_PASSWORD:-changeme}@mongodb:27017/parcels
      KAFKA_BROKERS: kafka:9092
      NODE_ENV: development
    depends_on:
      kafka:
        condition: service_healthy
      mongodb:
        condition: service_healthy
    restart: on-failure
    networks:
      - parcelflow-net

  # Delivery Service
  delivery-service:
    build:
      context: .
      dockerfile: Dockerfile.node
      target: delivery-service
    container_name: pf-delivery-service
    ports:
      - "3002:3002"
    environment:
      SERVICE_NAME: delivery-service
      EXPRESS_PORT: 3002
      MONGO_URL: mongodb://admin:${MONGO_ROOT_PASSWORD:-changeme}@mongodb:27017/deliveries
      KAFKA_BROKERS: kafka:9092
      NODE_ENV: development
    depends_on:
      kafka:
        condition: service_healthy
      mongodb:
        condition: service_healthy
    restart: on-failure
    networks:
      - parcelflow-net

  # Notification Service
  notification-service:
    build:
      context: .
      dockerfile: Dockerfile.node
      target: notification-service
    container_name: pf-notification-service
    ports:
      - "3003:3003"
    environment:
      SERVICE_NAME: notification-service
      EXPRESS_PORT: 3003
      MONGO_URL: mongodb://admin:${MONGO_ROOT_PASSWORD:-changeme}@mongodb:27017/notifications
      KAFKA_BROKERS: kafka:9092
      NODE_ENV: development
    depends_on:
      kafka:
        condition: service_healthy
      mongodb:
        condition: service_healthy
    restart: on-failure
    networks:
      - parcelflow-net

  # Vue.js Frontend
  parcel-ui:
    build:
      context: ./apps/parcel-ui
      dockerfile: Dockerfile.dev
    container_name: pf-vue-ui
    ports:
      - "5173:5173"
    environment:
      API_BASE_URL: http://parcel-service:3001
      REACT_APP_WS_URL: ws://parcel-service:3001
    depends_on:
      parcel-service:
        condition: service_healthy
    restart: on-failure
    networks:
      - parcelflow-net

# Optional Splunk stack
x-splunk-env: &splunk-env
  SPLUNK_PASSWORD: ${SPLUNK_PASSWORD:-changeme}
  SPLUNK_HEC_TOKEN: ${SPLUNK_HEC_TOKEN:-abc123token}
  HTTP_RECEIVER_PORT: 8200
  TCP_RECEIVER_PORT: 9886

services:
  fluentd:
    <<: *splunk-env
    image: fluent/fluentd:v1.17-1
    container_name: pf-fluentd
    ports:
      - "24224:24224"
    volumes:
      - ./fluentd:/fluentd/etc
      - vol-fluentd:/fluentd/log
    depends_on:
      - mongodb
      - parcel-service
      - delivery-service
      - notification-service
    networks:
      - parcelflow-net

  splunk:
    <<: *splunk-env
    image: splunk/splunk:latest
    container_name: pf-splunk
    ports:
      - "8200:8200"
      - "9886:9886"
      - "8000:8000"
    volumes:
      - vol-splunk:/opt/splunk/var
    profiles:
      - splunk-dbg
    networks:
      - parcelflow-net

volumes:
  vol-mongo:
  vol-kafka:
  vol-fluentd:
  vol-splunk:

networks:
  parcelflow-net:
    driver: bridge
```

## docker-compose.kafka-only.yml

Dev-only file for Kafka work without bringing up all services:

```yaml
# Usage: docker compose -f docker-compose.yml -f docker-compose.kafka-only.yml up
version: '3.8'

services:
  kafka:
    profiles:
      - kafka-only
    extends:
      file: docker-compose.yml
      service: kafka
  mongodb:
    profiles:
      - kafka-only
    extends:
      file: docker-compose.yml
      service: mongodb
```

## Init Script: `init-mongo.js`

Creates default databases and indexes on first startup:

```javascript
// init-mongo.js — runs once on first MongoDB container startup
db = db.getSiblingDB('admin');
db.createUser({
  user: 'admin',
  pwd: process.env.MONGO_INITDB_ROOT_PASSWORD || 'changeme',
  roles: [{ role: 'root', db: 'admin' }]
});

// Create databases (physical MongoDB creates them on first write)
db.getSiblingDB('parcels').createCollection('parcels', {
  timeseries: false
});
db.getSiblingDB('deliveries').createCollection('deliveries');
db.getSiblingDB('notifications').createCollection('notifications');
```

## Dockerfile.node (Shared Base)

Multi-stage build for each target:

```dockerfile
FROM node:22-slim AS base

WORKDIR /app

# Install deps
COPY packages/parcel-service/package.json ./packages/parcel-service/package.json
COPY packages/delivery-service/package.json ./packages/delivery-service/package.json
COPY packages/notification-service/package.json ./packages/notification-service/package.json
COPY package.json ./
RUN npm install

COPY packages/parcel-service ./packages/parcel-service
COPY packages/delivery-service ./packages/delivery-service
COPY packages/notification-service ./packages/notification-service

FROM base AS parcel-service
RUN npm run build parcel-service
EXPOSE 3001
CMD ["node", "dist/index.js"]

FROM base AS delivery-service
RUN npm run build delivery-service
EXPOSE 3002
CMD ["node", "dist/index.js"]

FROM base AS notification-service
RUN npm run build notification-service
EXPOSE 3003
CMD ["node", "dist/index.js"]
```

This is a simplified layout. Each service's Dockerfile can also be self-contained after the build step, but sharing a multi-stage approach keeps the monorepo clean.
