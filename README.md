# Flight Booking — Microservices Platform

A Spring Boot microservices system for searching flights, managing seat inventory, and creating passenger bookings, built with Domain-Driven Design, CQRS, and an event-driven architecture.

## Table of Contents

- [Description](#description)
- [Business Problem](#business-problem)
- [Features](#features)
- [Architecture](#architecture)
- [Technologies](#technologies)
- [Project Structure](#project-structure)
- [Microservices Overview](#microservices-overview)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Environment Variables](#environment-variables)
- [Running Locally](#running-locally)
- [Running with Docker](#running-with-docker)
- [Database](#database)
- [API Overview](#api-overview)
- [Testing](#testing)
- [Build](#build)
- [Development Workflow](#development-workflow)
- [Future Improvements](#future-improvements)
- [License](#license)

## Description

Flight Booking is a reference implementation of a flight-reservation platform decomposed into independently deployable Spring Boot microservices. Each service owns its own data, exposes a REST API for commands/queries, and communicates with other services over gRPC (synchronous lookups) and RabbitMQ (asynchronous domain events), following the transactional outbox pattern for reliable messaging.

## Business Problem

Airlines and travel agencies need to manage aircraft, airports, flight schedules, seat inventory, passenger records, and bookings as independently scalable capabilities. This project models that domain as four bounded contexts — **Flight**, **Passenger**, **Booking**, and a shared **API Gateway** — so that seat availability, passenger management, and booking creation can evolve, scale, and fail independently while staying consistent through domain events.

## Features

- Aircraft, airport, flight, and seat management (Flight service)
- Passenger registration and lookup (Passenger service)
- Booking creation with real-time seat and passenger validation via gRPC
- Event-driven seat reservation and flight-status propagation via RabbitMQ
- CQRS read/write segregation: PostgreSQL for writes, MongoDB for reads
- Transactional outbox for reliable event publishing
- Centralized authentication/authorization via Keycloak (OAuth2/JWT) behind an API Gateway
- Full observability stack: OpenTelemetry, Prometheus, Grafana, Loki, Tempo
- Database schema versioning with Flyway

## Architecture

The system follows **Hexagonal (Ports & Adapters) Architecture** combined with **Domain-Driven Design** and **CQRS** at the service level:

- **Domain layer** — Aggregates (`Flight`, `Seat`, `Aircraft`, `Airport`, `Passenger`, `Booking`), value objects, and domain events, free of framework dependencies.
- **Application layer** — Vertical-slice "features" (e.g. `createflight`, `reserveseat`, `createbooking`) implemented as Command/Query handlers via an internal mediator (CQRS), each with its own validator and controller.
- **Infrastructure/adapters** — JPA repositories (write side, PostgreSQL), MongoDB read repositories (query side), gRPC servers/clients for cross-service lookups, RabbitMQ publishers/consumers, Keycloak security adapters.
- **Shared kernel (`buildingblocks`)** — Cross-cutting building blocks: mediator abstractions, domain event dispatching, exception handling, outbox processor, JPA/R2DBC/Mongo configuration, Keycloak integration, OpenTelemetry, Swagger, and validation utilities, published as a shared Maven module.

Services communicate:
- **Synchronously** via gRPC (e.g. Booking → Flight/Passenger for validation at booking time).
- **Asynchronously** via RabbitMQ integration events (e.g. `FlightUpdated`, `SeatReserved`, `PassengerCreated`), published through the outbox pattern to guarantee at-least-once delivery.

## Technologies

- **Language/Runtime:** Java, Maven
- **Framework:** Spring Boot, Spring Data JPA, Spring Data MongoDB, Spring Data R2DBC, Spring Cloud Gateway
- **Messaging:** RabbitMQ
- **RPC:** gRPC / Protocol Buffers
- **Databases:** PostgreSQL (write model), MongoDB (read model)
- **Migrations:** Flyway
- **Security:** Keycloak, OAuth2 Resource Server, JWT
- **Observability:** OpenTelemetry Collector, Prometheus, Grafana, Loki, Tempo
- **API Docs:** Swagger / OpenAPI
- **CI:** GitHub Actions, Release Drafter

## Project Structure

```
flight-booking/
├── deployments/
│   ├── docker-compose/           # Infrastructure composition (DBs, MQ, observability)
│   └── configs/                  # Grafana, Prometheus, Loki, Tempo, OTel configs
├── src/
│   ├── buildingblocks/           # Shared kernel: mediator, events, persistence, security
│   ├── apigateway/                # Spring Cloud Gateway + Keycloak
│   └── services/
│       ├── flight/                # Aircraft, Airport, Flight, Seat bounded context
│       ├── passenger/              # Passenger bounded context
│       └── booking/                # Booking bounded context (orchestrates the above)
├── booking.rest                   # HTTP client requests for manual API testing
└── README.md
```

## Microservices Overview

| Service | Responsibility | Write DB | Read DB | Sync Port | gRPC Port |
|---|---|---|---|---|---|
| **flight** | Aircraft, airport, flight, seat inventory | PostgreSQL | MongoDB | REST | gRPC server |
| **passenger** | Passenger registration/lookup | PostgreSQL | MongoDB | REST | gRPC server |
| **booking** | Booking creation, orchestration | PostgreSQL | MongoDB | REST | gRPC client (flight, passenger) |
| **apigateway** | Routing, authentication | — | — | REST | — |

## Prerequisites

- Java 21+ (JDK)
- Maven 3.9+ (or use the included `mvnw` wrapper)
- Docker & Docker Compose
- PostgreSQL 15+, MongoDB 6+, RabbitMQ (provided via `deployments/docker-compose`)
- A running Keycloak realm for authentication

## Installation

```bash
git clone <repository-url>
cd flight-booking
./src/buildingblocks/mvnw -f src/buildingblocks/pom.xml install
```

Build each service:

```bash
./src/services/flight/mvnw -f src/services/flight/pom.xml clean install
./src/services/passenger/mvnw -f src/services/passenger/pom.xml clean install
./src/services/booking/mvnw -f src/services/booking/pom.xml clean install
./src/apigateway/mvnw -f src/apigateway/pom.xml clean install
```

## Configuration

Each service ships `application.yml` (defaults) and `application-dev.yml` (local overrides) under `src/main/resources`, covering datasource, MongoDB, RabbitMQ, gRPC, OpenTelemetry, and Keycloak JWT settings.

## Environment Variables

| Variable | Description | Example |
|---|---|---|
| `SPRING_DATASOURCE_URL` | PostgreSQL JDBC URL | `jdbc:postgresql://localhost:5432/flight_write` |
| `SPRING_DATA_MONGODB_URI` | MongoDB connection URI | `mongodb://localhost:27017/flight_read` |
| `SPRING_RABBITMQ_HOST` | RabbitMQ host | `localhost` |
| `KEYCLOAK_JWK_SET_URI` | Keycloak JWKS endpoint | `http://localhost:8080/realms/keycloak-realm/protocol/openid-connect/certs` |
| `OTEL_COLLECTOR_ENDPOINT` | OpenTelemetry collector endpoint | `http://localhost:4317` |

## Running Locally

1. Start infrastructure: `docker compose -f deployments/docker-compose/docker-compose.infrastructure.yaml up -d`
2. Run Flyway-managed migrations automatically on service startup.
3. Start each service with its `dev` profile, e.g.:
   ```bash
   ./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
   ```
4. Use `booking.rest` (REST Client format) to exercise the APIs.

## Running with Docker

```bash
docker compose -f deployments/docker-compose/docker-compose.infrastructure.yaml up -d
```

This provisions PostgreSQL, MongoDB, RabbitMQ, Keycloak, and the observability stack (Prometheus, Grafana, Loki, Tempo, OTel Collector) with pre-provisioned Grafana dashboards and datasources.

## Database

- **Write model:** PostgreSQL, one schema per service, versioned with Flyway (`V1__Init_*_Tables.sql`).
- **Read model:** MongoDB, one database per service, populated via domain-event projections for fast, denormalized queries.
- **Outbox table:** `persist_messages`, used by the shared outbox processor for reliable event publishing.

## API Overview

Each service exposes REST endpoints for its vertical-slice features (e.g. `POST /aircrafts`, `POST /flights`, `GET /flights/available`, `POST /seats/reserve`, `POST /passengers`, `POST /bookings`). Cross-service lookups (seat/passenger validation during booking) happen over gRPC, defined in the `.proto` contracts under each service's `src/main/proto`. See `booking.rest` for concrete request examples.

## Testing

- **Unit tests** — domain and handler logic in isolation.
- **Integration tests** — persistence and messaging with Testcontainers.
- **End-to-end tests** — full request lifecycle per service.

Run tests per module:

```bash
./mvnw test
```

## Build

```bash
./mvnw clean package
```

## Development Workflow

1. Branch from `main`.
2. Implement a vertical slice (domain → application handler → adapter → tests).
3. Open a PR; GitHub Actions runs the build; Release Drafter tracks changelog entries.
4. Merge once green, then deploy.

## Future Improvements

- Saga/process-manager orchestration for multi-step booking compensation
- Rate limiting and circuit breakers at the API Gateway
- Contract testing between services
- Kubernetes manifests / Helm charts for production deployment

## License

This project is licensed under the MIT License.
