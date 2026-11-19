# Gatekeeper: Spring API Gateway

A multi-tenant API gateway built with Spring Boot as a set of microservices.
Organizations (tenants) register their backend APIs as routes. The gateway
then sits in front of those backends and handles routing, security, rate
limiting and header rules, and collects analytics on the traffic.

The project is split along the classic gateway lines:

- **Control plane:** where the gateway is configured (tenants, users, routes).
- **Data plane:** the gateway itself, which forwards live traffic using that configuration.
- **Analytics:** records and reports on what flowed through the gateway.

```
   Admins (UI / Postman)                         End users
           │                                         │
           ▼                                         ▼
┌──────────────────────┐   active routes   ┌──────────────────────┐
│ api-management-      │ ────────────────► │ smart-gateway-       │ ──► tenant backends
│ service  (control)   │                   │ service  (data)      │
└──────────┬───────────┘                   └──────────┬───────────┘
           │                                          │ traffic events
         MySQL                                        ▼
                                           ┌──────────────────────┐
                                           │ analytics-service    │
                                           └──────────┬───────────┘
                                                    MongoDB
```

## Services

| Service | Port | Role | Main storage | Status | Docs |
|---|---|---|---|---|---|
| `api-management-service` | 8081 | Control plane: organizations, tenant admins, routes, versioning, audit | MySQL | Implemented | [README](api-management-service/README.md) |
| `smart-gateway-service` | 8080 | Data plane: proxies requests to tenant backends | Redis | Skeleton (health endpoint only) | [folder](smart-gateway-service/) |
| `analytics-service` | 8082 | Collects and reports gateway traffic | MongoDB | Skeleton (health endpoint only) | [folder](analytics-service/) |

Each service is its own Spring Boot application with its own `pom.xml`, and
can be built and run on its own.

## Tech stack

- Java 21, Spring Boot 4.1 (Web MVC, Security, OAuth2 Resource Server, Validation)
- MySQL + Spring Data JPA + Flyway (management)
- Redis (gateway)
- MongoDB (analytics)
- Kafka, for communication between services (planned)
- JWT (HS256) authentication

## Getting started

Prerequisites: Java 21, plus the storage each service needs (MySQL, Redis, MongoDB).

Run any service from its folder:

```bash
cd api-management-service
./mvnw spring-boot:run
```

Each service exposes `GET /health`. See the service's own README for setup,
configuration and its API.

## Repository layout

```
gatekeeper/
├── api-management-service/   control plane
├── smart-gateway-service/    data plane
└── analytics-service/        analytics
```
