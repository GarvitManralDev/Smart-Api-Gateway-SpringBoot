# API Management Service

The **control plane** of the Gatekeeper API gateway. This service does not proxy
any traffic itself. It is where people **configure** the gateway: which
organizations (tenants) exist, who can log in for them, and which API routes
each tenant exposes. The separate `smart-gateway-service` (the **data plane**)
is meant to fetch the active routes from here and actually forward requests.

```
                    ┌──────────────────────────────┐
  Platform admin ──►│                              │
  Tenant admin  ───►│   api-management-service     │──► MySQL
  (browser UI or    │   (this service, port 8081)  │    gatekeeper_management
   Postman)         │                              │
                    └──────────────┬───────────────┘
                                   │ GET /api/gateway/routes
                                   │ (role GATEWAY_SERVICE)
                    ┌──────────────▼───────────────┐
  End users ───────►│   smart-gateway-service      │──► tenant backends
                    │   (data plane)               │
                    └──────────────────────────────┘
```

---

## Contents

1. [Tech stack](#tech-stack)
2. [Running it](#running-it)
3. [How a request flows through the code](#how-a-request-flows-through-the-code)
4. [Package layout](#package-layout)
5. [Roles and who can do what](#roles-and-who-can-do-what)
6. [Authentication: login and JWTs](#authentication-login-and-jwts)
7. [Multi-tenancy: how tenants are kept apart](#multi-tenancy-how-tenants-are-kept-apart)
8. [The data model](#the-data-model)
9. [Features in detail](#features-in-detail)
10. [API reference](#api-reference)
11. [The browser UI](#the-browser-ui)
12. [Database migrations (Flyway)](#database-migrations-flyway)
13. [Spring concepts used here](#spring-concepts-used-here)
14. [Try it end to end](#try-it-end-to-end)
15. [Known gaps and things to notice](#known-gaps-and-things-to-notice)

---

## Tech stack

| Piece | What it does here |
|---|---|
| Java 21, Spring Boot 4.1 | Application framework |
| Spring Web MVC | REST controllers (`@RestController`) and serving the static UI |
| Spring Data JPA + Hibernate | Maps Java entities to MySQL tables; generates queries from method names |
| MySQL | Stores everything (`gatekeeper_management` database) |
| Flyway | Creates and seeds the schema from SQL files on startup |
| Spring Security + OAuth2 Resource Server | Checks the JWT on every request and enforces roles |
| Nimbus JOSE (via Spring Security) | Signs and verifies JWTs with HMAC-SHA256 |
| Bean Validation (`jakarta.validation`) | `@NotBlank`, `@Size`, etc. on request DTOs |
| Lombok | Generates getters/setters/constructors (`@Getter`, `@Setter`, ...) |
| Kafka starter | In `pom.xml`, **not used yet** |

---

## Running it

**Prerequisites:** Java 21 and a MySQL server on `localhost:3306`.

1. Create the database (Flyway creates the tables, but not the database itself):
   ```sql
   CREATE DATABASE gatekeeper_management;
   ```
2. Check `src/main/resources/application.properties`:
   ```properties
   server.port=8081
   spring.datasource.url=jdbc:mysql://localhost:3306/gatekeeper_management
   spring.datasource.username=root
   spring.datasource.password=...
   spring.jpa.hibernate.ddl-auto=validate   # Hibernate only checks the schema, never changes it
   spring.flyway.enabled=true               # Flyway owns the schema
   security.jwt.expiration-seconds=3600     # tokens last 1 hour
   security.jwt.secret=...                  # HMAC key; must be at least 32 bytes for HS256
   ```
3. Start it:
   ```bash
   ./mvnw spring-boot:run
   ```
4. Open <http://localhost:8081> for the UI, or <http://localhost:8081/health> to check it is up.

**Seeded accounts** (created by `V3__seed_platform_accounts.sql`):

| Username | Role | Password |
|---|---|---|
| `platform-admin` | `PLATFORM_ADMIN` | stored only as a BCrypt hash in V3 (the course gives the plain text) |
| `gateway-service` | `GATEWAY_SERVICE` | same as above |

Tenant admins are not seeded. A platform admin creates them through the API.

**Postman:** import `postman/gatekeeper.postman_collection.json`. It has every
endpoint, and the Login request saves the token for you.

---

## How a request flows through the code

Take `POST /api/tenant/routes` (a tenant creating a route) as an example:

```
HTTP request
   │
   ▼
Spring Security filter chain  (config/SecurityConfig)
   │  1. reads "Authorization: Bearer <jwt>"
   │  2. verifies the signature and expiry with JwtDecoder
   │  3. turns the "role" claim into authority ROLE_TENANT_ADMIN
   │  4. checks /api/tenant/** requires TENANT_ADMIN, otherwise 401/403
   ▼
Controller  (controller/TenantRouteController.create)
   │  - JSON body → RouteRequest DTO
   │  - @Valid runs the annotations on RouteRequest; a failure returns 400
   ▼
Service  (service/RouteService.create)        ← @Transactional: one DB transaction
   │  - TenantContext.getOrganization(): which org is the caller?
   │  - business rules (path must start with /<slug>/, valid URL, ...)
   │  - builds the ApiRoute entity
   │  - saves the route, a version snapshot, and an audit record
   ▼
Repositories  (repository/*Repository)   ← Spring Data generates the SQL
   ▼
MySQL
   │
   ▼
The entity is converted to a RouteResponse DTO → JSON → 201 Created
```

Every feature follows this **Controller → Service → Repository → Entity** shape.

- **Controllers** only handle HTTP: URLs, status codes, request and response bodies.
- **Services** hold the business rules and transactions.
- **Repositories** handle database access.
- **DTOs** (`dto/requestDto`, `dto/responseDto`) are what goes over the wire. Entities
  never leave the service directly, so database details (like password hashes)
  can't leak into a response.

---

## Package layout

```
in.garvit.gatekeeper.management
├── ApiManagementServiceApplication   main(); @SpringBootApplication starts everything
├── config/
│   └── SecurityConfig                URL rules, JWT encoder/decoder, BCrypt
├── controller/                       REST endpoints (HTTP layer)
│   ├── AuthenticationController        POST /api/auth/login
│   ├── HealthController                GET  /health
│   ├── PlatformOrganizationController  /api/platform/organizations/**
│   ├── TenantOrganizationController    /api/tenant/organization
│   ├── TenantRouteController           /api/tenant/routes, route-versions, audit-records
│   └── GatewayConfigurationController  /api/gateway/routes
├── service/                          business logic
│   ├── AuthenticationService           checks the password, issues the JWT
│   ├── OrganizationService             organizations + tenant admins
│   ├── RouteService                    routes, versions, audit listing
│   ├── AuditService                    writes audit records
│   ├── TenantContext                   "which organization is calling?"
│   └── HealthService                   SELECT 1 against MySQL
├── repository/                       Spring Data JPA interfaces
├── entity/                           JPA entities (one per table)
├── dto/requestDto/                   incoming JSON shapes + validation rules
├── dto/responseDto/                  outgoing JSON shapes (+ static from(entity) mappers)
└── exception/
    └── ApiException                  RuntimeException that carries an HTTP status

src/main/resources
├── application.properties
├── db/migration/                     Flyway SQL (V1 schema, V2 roles, V3 accounts)
└── static/                           browser UI: index.html, styles.css, app.js
```

---

## Roles and who can do what

There are three roles, stored in the `roles` table and seeded by `V2`.

| Role | Who | Belongs to an org? | URL area |
|---|---|---|---|
| `PLATFORM_ADMIN` | Runs the whole platform | No (`organization_id` is NULL) | `/api/platform/**` |
| `TENANT_ADMIN` | Manages one organization | Yes, exactly one | `/api/tenant/**` |
| `GATEWAY_SERVICE` | The data-plane gateway (a machine, not a person) | No | `/api/gateway/**` |

These URL rules live in `SecurityConfig`:

```java
.requestMatchers("/", "/index.html", "/styles.css", "/app.js", "/health", "/api/auth/login").permitAll()
.requestMatchers("/api/platform/**").hasRole("PLATFORM_ADMIN")
.requestMatchers("/api/tenant/**").hasRole("TENANT_ADMIN")
.requestMatchers("/api/gateway/**").hasRole("GATEWAY_SERVICE")
.anyRequest().authenticated()
```

`hasRole("X")` checks for an authority named `ROLE_X`. The `ROLE_` prefix is added in
`jwtAuthenticationConverter()`.

A platform admin **cannot** call tenant endpoints, and the reverse is also true. Roles
don't stack.

---

## Authentication: login and JWTs

This service is both the **issuer** of tokens and the **resource server** that checks
them. There are no server-side sessions (`SessionCreationPolicy.STATELESS`), so every
request must carry its token.

**1. Login** (`AuthenticationService.login`):
1. Find the user by username. If there's no such user, return 401 "Invalid username or password".
2. Reject the login if the user is disabled or the password doesn't match. BCrypt compares
   the plain password with the stored hash, so the real password is never stored.
3. If the user belongs to an organization that is not `ACTIVE`, return 403.
4. Build the JWT:
   ```json
   {
     "iss": "gatekeeper-api-management",
     "sub": "acme-admin",
     "role": "TENANT_ADMIN",
     "organizationId": 1,            // only for tenant admins
     "iat": 1763540000,
     "exp": 1763543600
   }
   ```
   It is signed with HS256 using `security.jwt.secret`.
5. Return `{ token, tokenType: "Bearer", expiresInSeconds, role, organizationId }`.

**2. Every later request:**
1. The client sends `Authorization: Bearer <token>`.
2. The `JwtDecoder` bean verifies the signature with the same secret and checks `exp`.
3. `jwtAuthenticationConverter()` reads the `role` claim and creates a `JwtAuthenticationToken`
   with authority `ROLE_<role>`.
4. That token is stored in `SecurityContextHolder`. Any code can then read it during the
   request, which is how `TenantContext` and `AuditService` know who the caller is.

Because the same secret both signs and verifies (symmetric HMAC), anyone with the secret
can create valid tokens. Keep it out of git in a real deployment.

---

## Multi-tenancy: how tenants are kept apart

All tenants share the same tables. Every tenant-owned row has an `organization_id` column,
and isolation is enforced **in the service layer**.

**`TenantContext.getOrganization()`** is the key piece:
1. Read the current `JwtAuthenticationToken` from `SecurityContextHolder`.
2. Take the `organizationId` claim from the token. The org id **never** comes from the URL
   or the request body, so a tenant can't ask for another tenant's data by changing an id.
3. Load the organization and require its status to be `ACTIVE`. If a platform admin suspends
   an org, its tenant admin is locked out immediately, even with a valid token.

Every query then filters by that org. For example:

```java
apiRouteRepository.findByIdAndOrganizationId(id, organization.getId())
```

If tenant A asks for route 7 and route 7 belongs to tenant B, the query finds nothing and
the response is 404 "Route not found". That 404 doesn't reveal that route 7 exists at all.

**Path namespacing:** a route's `pathPattern` must start with `/<org-slug>/`, for example
`/acme-corp/orders/**`. Two tenants can never claim the same public path on the gateway.

---

## The data model

```
roles 1───* management_users *───0..1 organizations 1───* api_routes 1───* route_header_rules
                                             │                 └──* route_allowed_methods
                                             ├──* route_configuration_versions
                                             └──* control_plane_audit_records
```

| Table / Entity | Purpose | Notable details |
|---|---|---|
| `organizations` / `Organization` | A tenant | `name` and `slug` unique; `status` is an enum stored as text; `deleted_at` for soft delete |
| `roles` / `Role` | The three roles | Seeded by V2 |
| `management_users` / `ManagementUser` | Anyone who can log in | `organization_id` is NULL for platform/gateway accounts; **unique**, so at most one tenant admin per org |
| `api_routes` / `ApiRoute` | One gateway route | `(organization_id, route_id)` unique; `configuration_version` counts changes |
| `route_allowed_methods` | GET/POST/... allowed on a route | Not an entity: an `@ElementCollection` of strings on `ApiRoute` |
| `route_header_rules` / `RouteHeaderRule` | Add/remove headers on request or response | `@OneToMany(cascade = ALL, orphanRemoval = true)`: saved and deleted together with the route |
| `route_configuration_versions` / `RouteConfigurationVersion` | History: a full JSON snapshot of the route after every change | Not a foreign key to `api_routes`, so history **survives deleting the route** |
| `control_plane_audit_records` / `ControlPlaneAuditRecord` | Who did what, when, and how long it took | Written by `AuditService` |

**Organization status lifecycle** (`OrganizationStatus` enum):

```
create ──► PENDING ──► ACTIVE ◄──► SUSPENDED
                │         │            │
                └─────────┴────────────┴──► DELETED   (only through DELETE, a soft delete)
```

- New organizations start as `PENDING`. Their tenant admin **can't log in** until a platform admin sets them to `ACTIVE`.
- `PATCH .../status` refuses `DELETED`. You must use `DELETE`, which sets `deleted_at`.
- Deleted organizations are hidden by the `...StatusNot(DELETED)` queries, but the rows stay in the database.

**Route field meanings** (`RouteRequest`):

| Field | Meaning |
|---|---|
| `routeId` | Your stable string id, e.g. `orders-api`. Can't change after creation. Version history is keyed by it |
| `name` | Display name |
| `pathPattern` | Public path on the gateway; must start with `/<slug>/` |
| `targetBaseUrl` | Backend to forward to; must be `http` or `https` |
| `targetPathPrefix` | Path prefix on the backend; must start with `/` |
| `requestsPerMinute` | Rate limit (optional, at least 1) |
| `responseTimeoutMs` | Backend timeout (required, at least 1) |
| `idempotencyEnabled` | Flag for the gateway's idempotency feature |
| `active` | Only active routes are sent to the gateway |
| `allowedMethods` | Non-empty set; each must be a real HTTP method; stored uppercased |
| `headerRules` | List of `{direction: REQUEST\|RESPONSE, action: ADD\|REMOVE, headerName, headerValue}`; `ADD` needs a value |

Validation happens in two layers:
1. **Annotations** on the DTO (`@NotBlank`, `@Size`, `@Min`), checked by `@Valid` in the controller.
2. **Business rules** in `RouteService.validate()` (slug prefix, URL scheme, methods, header rules).

---

## Features in detail

### Organizations (platform admin): `OrganizationService`
- **Create:** checks the name and slug are unique (case-insensitive), saves with status `PENDING`, and writes an audit record.
- **List / get:** never return `DELETED` orgs.
- **Update:** changes the name and slug. `checkUnique` lets an org keep its own name.
- **Change status:** any status except `DELETED`.
- **Delete:** a soft delete.

### Tenant admin (platform admin): `OrganizationService`
- One per organization. A second one gets 409 Conflict, and the DB unique constraint backs this up.
- Usernames are unique across the whole system.
- The password is BCrypt-hashed before saving. Responses (`TenantAdminResponse`) never contain the hash.
- Update replaces the username, password and enabled flag. Delete is a real (hard) delete.

### Own organization (tenant admin)
- `GET /api/tenant/organization`: see your org.
- `PUT /api/tenant/organization`: rename it. Tenants can change only the name, not the slug or status.

### Routes (tenant admin): `RouteService`
Every change bumps `configurationVersion` and stores a snapshot:

| Action | Version | `changeType` | Audit action |
|---|---|---|---|
| Create | 1 | `CREATED` | `CREATE_ROUTE` |
| Update | +1 | `UPDATED` | `UPDATE_ROUTE` |
| Activate / deactivate | +1 | `ACTIVATED` / `DEACTIVATED` | `ACTIVATE_ROUTE` / `DEACTIVATE_ROUTE` |
| Delete | +1 (final snapshot) | `DELETED` | `DELETE_ROUTE` |

`apply()` copies the request onto the entity. For header rules it **clears the list and
rebuilds it**. Because of `orphanRemoval = true`, Hibernate deletes the old rule rows by
itself.

### Version history
`GET /api/tenant/route-versions/{routeId}` returns all snapshots for that **string**
`routeId`, newest first. Each snapshot is the route serialized to JSON with Jackson's
`ObjectMapper` at that moment. Use it to see exactly what the config was at any version,
even after the route is deleted.

### Audit trail: `AuditService`
Every write records:
- the **actor** (username from the JWT, or `system`)
- the **action**, **resource type** and **resource id**
- the **organization**
- the **duration** (in ms, measured from `startedAt = System.nanoTime()` at the start of the service method)
- the **result**
- the **`X-Correlation-ID`** request header, if the caller sent one. This lets you link a gateway request to the config change it caused.

Tenants read their own audit trail through `GET /api/tenant/audit-records`.

### Gateway feed
`GET /api/gateway/routes` (role `GATEWAY_SERVICE`) returns every route that is `active`
**and** belongs to an `ACTIVE` organization, across all tenants. This is what the
data-plane gateway will poll to build its routing table.

### Health
`GET /health` is public. It runs `SELECT 1` against MySQL and returns
`{status: "UP", database: "UP" | "DOWN", timestamp}`.

---

## API reference

Base URL: `http://localhost:8081`

| Method | Path | Role | Body | Success |
|---|---|---|---|---|
| GET | `/health` | public | | 200 |
| POST | `/api/auth/login` | public | `{username, password}` | 200 |
| POST | `/api/platform/organizations` | PLATFORM_ADMIN | `{name, slug}` | 201 |
| GET | `/api/platform/organizations` | PLATFORM_ADMIN | | 200 |
| GET | `/api/platform/organizations/{id}` | PLATFORM_ADMIN | | 200 |
| PUT | `/api/platform/organizations/{id}` | PLATFORM_ADMIN | `{name, slug}` | 200 |
| PATCH | `/api/platform/organizations/{id}/status` | PLATFORM_ADMIN | `{status}` | 200 |
| DELETE | `/api/platform/organizations/{id}` | PLATFORM_ADMIN | | 204 |
| POST | `/api/platform/organizations/{id}/tenant-admin` | PLATFORM_ADMIN | `{username, password, enabled}` | 201 |
| GET | `/api/platform/organizations/{id}/tenant-admin` | PLATFORM_ADMIN | | 200 |
| PUT | `/api/platform/organizations/{id}/tenant-admin` | PLATFORM_ADMIN | `{username, password, enabled}` | 200 |
| DELETE | `/api/platform/organizations/{id}/tenant-admin` | PLATFORM_ADMIN | | 204 |
| GET | `/api/tenant/organization` | TENANT_ADMIN | | 200 |
| PUT | `/api/tenant/organization` | TENANT_ADMIN | `{name}` | 200 |
| POST | `/api/tenant/routes` | TENANT_ADMIN | RouteRequest | 201 |
| GET | `/api/tenant/routes` | TENANT_ADMIN | | 200 |
| GET | `/api/tenant/routes/{id}` | TENANT_ADMIN | | 200 |
| PUT | `/api/tenant/routes/{id}` | TENANT_ADMIN | RouteRequest | 200 |
| PATCH | `/api/tenant/routes/{id}/status` | TENANT_ADMIN | `{active}` | 200 |
| DELETE | `/api/tenant/routes/{id}` | TENANT_ADMIN | | 204 |
| GET | `/api/tenant/route-versions/{routeId}` | TENANT_ADMIN | | 200 |
| GET | `/api/tenant/audit-records` | TENANT_ADMIN | | 200 |
| GET | `/api/gateway/routes` | GATEWAY_SERVICE | | 200 |

`{id}` in `/routes/{id}` is the numeric database id. `{routeId}` in `/route-versions/` is
the string id like `orders-api`.

---

## The browser UI

`src/main/resources/static/` is served at the root by Spring Boot, with no controller
needed:

- `index.html`: the login screen and the app shell. It has sections for Overview,
  Organizations, Routes, My organization and Audit records, plus dialogs for editing.
- `app.js`: plain JavaScript. It calls the same REST API with `fetch`, keeps the JWT, and
  shows or hides sections based on the role returned by login.
- `styles.css`: the styling.

These three files and `/` are in the `permitAll()` list. Without that, the browser couldn't
load the page before logging in. If you add or rename a static file, add it to that list
too. That mismatch is why the CSS once failed to load.

---

## Database migrations (Flyway)

On startup Flyway runs any SQL file in `db/migration` that hasn't been applied yet, in
version order, and records each one in its `flyway_schema_history` table:

| File | Does |
|---|---|
| `V1__create_api_management_schema.sql` | Creates all 8 tables, keys and constraints |
| `V2__seed_management_roles.sql` | Inserts the 3 roles |
| `V3__seed_platform_accounts.sql` | Inserts `platform-admin` and `gateway-service` with BCrypt hashes |

Rules:
- **Never edit a migration that has already run.** Flyway stores a checksum and will refuse
  to start if the file changes. Add a new `V4__...sql` instead.
- `ddl-auto=validate` makes Hibernate check at startup that the entities match the tables.
  If you add an entity field without a matching migration, the app fails to start.

---

## Spring concepts used here

| Concept | Where | Why it matters |
|---|---|---|
| **Beans and dependency injection** | `@Service`, `@Component`, `@Configuration`/`@Bean`, constructor parameters | Spring creates one instance of each annotated class and passes it into constructors. A class without an annotation isn't a bean, which caused the "required a bean of type RouteService" startup error |
| **Constructor injection** | Every service and controller | Dependencies are `final` and explicit. `HealthService` uses Lombok's `@RequiredArgsConstructor` to generate the constructor |
| **Derived queries** | `findByIdAndOrganizationId`, `findAllByStatusNot`, `existsBySlugIgnoreCase`, ... | Spring Data reads the method name and writes the SQL. No implementation class exists |
| **`@Transactional`** | Service methods | All DB work in the method commits together or rolls back together. `readOnly = true` on reads is an optimization |
| **Dirty checking** | `update`, `changeStatus` in the services | These methods never call `save()`. The entity was loaded inside a transaction, so Hibernate notices the changed fields and writes them at commit |
| **Lazy vs eager loading** | `@ManyToOne(fetch = LAZY)` | Related rows are loaded only when accessed. `ManagementUser.role` is `EAGER` because login always needs it |
| **Bean Validation** | `@Valid @RequestBody`, DTO annotations | Bad input is rejected with 400 before your code runs |
| **Security filter chain** | `SecurityConfig.securityFilterChain` | Runs before every controller; handles auth and URL rules |
| **SecurityContextHolder** | `TenantContext`, `AuditService` | Per-request (per-thread) storage of who is logged in |
| **Static resources** | `src/main/resources/static` | Served automatically at `/` |

---

## Try it end to end

Using Postman (`postman/gatekeeper.postman_collection.json`) or the UI:

1. **Health:** `GET /health` should return `database: UP`.
2. **Log in as platform admin:** set `username=platform-admin` and its password, then send Login.
3. **Create an org:** `{ "name": "Acme Corp", "slug": "acme-corp" }`. It starts as `PENDING`.
4. **Create its tenant admin:** `{ "username": "acme-admin", "password": "ChangeMe123" }`.
5. Log in as `acme-admin`. This **fails with 403** because the org is still `PENDING`.
6. **Activate the org:** `PATCH .../status` with `{ "status": "ACTIVE" }`.
7. **Log in as `acme-admin`** again. It works now.
8. **Create a route** with `pathPattern` `/acme-corp/orders/**`. You get version 1.
9. **Update it** and **deactivate it**, then look at `GET /api/tenant/route-versions/orders-api`. You should see versions 3, 2 and 1.
10. **Check the trail** with `GET /api/tenant/audit-records`.
11. **Log in as `gateway-service`** and call `GET /api/gateway/routes`. Only active routes of active orgs appear, so re-activate the route to see it.

---

## Known gaps and things to notice

These are worth knowing while learning. Some may be fixed in later lectures.

- **No exception handler.** Services throw `ApiException(HttpStatus.NOT_FOUND, ...)` and
  similar, but there's no `@RestControllerAdvice` to turn that into a response. Spring's
  default handling treats any unhandled `RuntimeException` as **500 Internal Server
  Error**. So "Route not found" or "Invalid username or password" currently come back as
  500, not 404 or 401. Adding a `@RestControllerAdvice` with
  `@ExceptionHandler(ApiException.class)` that returns `exception.getStatus()` would fix it.
- **Copy-pasted audit messages.** In `OrganizationService`, `changeStatus` and `delete`
  record action `UPDATE_ORGANIZATION` with details "Organization name and slug updated".
- **Kafka is a dependency but unused.** It's probably intended for later, for example
  pushing route changes to the gateway instead of the gateway polling.
- **Secrets in `application.properties`.** The DB password and JWT secret are in a file
  committed to git. Fine for a course, but real deployments use environment variables.
- **The gateway side isn't wired yet.** Nothing in `smart-gateway-service` calls
  `/api/gateway/routes` yet.
- **Only the default test exists.** `ApiManagementServiceApplicationTests` just checks that
  the application context starts, and it needs a running MySQL to do so.
