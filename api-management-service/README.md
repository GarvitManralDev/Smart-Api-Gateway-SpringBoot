# API Management Service

The **control plane** of Gatekeeper. This is where the gateway is configured:
which organizations (tenants) use it, who can manage each organization, and
which APIs each organization exposes through the gateway.

It does not forward any traffic itself. The
[smart-gateway-service](../smart-gateway-service/) reads the active routes
from here and does the actual routing.

**Who uses it**

- **Platform admin:** runs the platform. Onboards organizations and gives each one an admin.
- **Tenant admin:** manages a single organization and its API routes.
- **Gateway service:** the gateway itself, fetching the routes it should serve.

Runs on `http://localhost:8081`. It comes with a web UI at the root URL and a
Postman collection in [`postman/`](postman/gatekeeper.postman_collection.json).

---

## Screenshots

> Add images to `docs/screenshots/` with these names and they will show up here.

| Screen | |
|---|---|
| Login | ![Login](docs/screenshots/login.png) |
| Overview | ![Overview](docs/screenshots/overview.png) |
| Organizations | ![Organizations](docs/screenshots/organizations.png) |
| Routes | ![Routes](docs/screenshots/routes.png) |
| Route versions | ![Route versions](docs/screenshots/route-versions.png) |
| Audit records | ![Audit records](docs/screenshots/audit-records.png) |

---

## APIs

### General

| API | Purpose |
|---|---|
| `GET /health` | Check that the service and its database are up |
| `POST /api/auth/login` | Log in and get a token for the other APIs |

### Platform admin: organizations

| API | Purpose |
|---|---|
| `POST /api/platform/organizations` | Onboard a new organization |
| `GET /api/platform/organizations` | See all organizations on the platform |
| `GET /api/platform/organizations/{id}` | View one organization |
| `PUT /api/platform/organizations/{id}` | Rename an organization or change its slug |
| `PATCH /api/platform/organizations/{id}/status` | Activate or suspend an organization |
| `DELETE /api/platform/organizations/{id}` | Remove an organization from the platform |

### Platform admin: tenant admins

| API | Purpose |
|---|---|
| `POST /api/platform/organizations/{id}/tenant-admin` | Give an organization its admin account |
| `GET /api/platform/organizations/{id}/tenant-admin` | See who manages an organization |
| `PUT /api/platform/organizations/{id}/tenant-admin` | Change an admin's login or enable/disable them |
| `DELETE /api/platform/organizations/{id}/tenant-admin` | Remove an organization's admin |

### Tenant admin: own organization

| API | Purpose |
|---|---|
| `GET /api/tenant/organization` | View your own organization |
| `PUT /api/tenant/organization` | Rename your organization |

### Tenant admin: routes

| API | Purpose |
|---|---|
| `POST /api/tenant/routes` | Expose a new backend API through the gateway |
| `GET /api/tenant/routes` | See all of your organization's routes |
| `GET /api/tenant/routes/{id}` | View one route |
| `PUT /api/tenant/routes/{id}` | Change a route's settings |
| `PATCH /api/tenant/routes/{id}/status` | Turn a route on or off in the gateway |
| `DELETE /api/tenant/routes/{id}` | Remove a route |
| `GET /api/tenant/route-versions/{routeId}` | See the change history of a route |
| `GET /api/tenant/audit-records` | See who changed what in your organization, and when |

### Gateway

| API | Purpose |
|---|---|
| `GET /api/gateway/routes` | Give the gateway the list of routes it should currently serve |
