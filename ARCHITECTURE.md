# Architecture

## Backend structure
The API is organized into:
- Controllers: HTTP endpoints, authorization gates, and status codes.
- DTOs: request and response contracts.
- Services: application and business operations.
- Data: EF Core DbContext and data seeders.
- Models: persistence entities.
- Enums: constrained domain values.
- Security: permission catalog.
- Migrations: database schema evolution.

Request flow:
HTTP Client → Authentication/Authorization → Controller → Service Interface/Implementation → DbContext → SQL Server.

Controllers should remain thin. API contracts use DTOs rather than exposing EF entities directly. Async controller and service operations accept CancellationToken where implemented.

## Multi-tenancy and tenant scope
Stock Pilot uses a shared SQL Server database with company-scoped records. Tenant-owned entities carry CompanyId; composite keys and foreign keys constrain relationships to the same company.

Tenant API routes currently include a client-supplied `companyId`, for example:
- `/api/companies/{companyId}/products`
- `/api/companies/{companyId}/warehouses`
- `/api/companies/{companyId}/stock`
- `/api/companies/{companyId}/stock-transfers`

The route value is not accepted as proof of access. `PermissionService` requires an active membership matching the authenticated user and requested company, and verifies active company/user/membership state. Warehouse-scoped membership is additionally checked against its active assigned warehouse. Service queries also scope records by CompanyId.

This is endpoint-level company and warehouse authorization, but tenant identity is not yet centrally resolved into a trusted tenant context; CompanyId remains an explicit route parameter. Keep the membership check on every tenant-owned operation and add integration coverage for every resource before production exposure.

## Authentication and platform administration
Authentication uses ASP.NET Core Identity with `ApplicationUser : IdentityUser<Guid>` and JWT bearer tokens. JWT validation checks issuer, audience, signing key, and lifetime. Inbound claim mapping is disabled.

Platform Admin is a separate Identity role, not a company membership. The `PlatformAdminOnly` policy protects:
- `CompaniesController` at `/api/platform/companies` (list, get, update, deactivate).
- `PlatformCompaniesController` at `/api/platform/company-provisioning` (provision company and initial moderator; provision a moderator for an existing company).

Company provisioning creates the company and initial `CompanyModerator` membership in a database transaction. It generates a 24-character temporary password and sets `MustChangePassword=true`. A separate operation creates a moderator for an existing active company only when it has no active CompanyModerator. The first successful login returns a five-minute JWT containing `token_use=password_change`; middleware rejects that token on other routes. Successful password change clears the flag and returns a normal JWT.

Bootstrap configuration keys are `PlatformAdmin:Email`, `PlatformAdmin:Password`, and `PlatformAdmin:FullName`. JWT keys are `Jwt:Secret`, `Jwt:Issuer`, `Jwt:Audience`, and `Jwt:ExpirationMinutes`. Keep all secrets outside source control.

## Authorization and role permissions
Tenant-owned controllers inherit `CompanyAuthorizedController`, which has `[Authorize]` and shared helpers for reading the authenticated subject and assigned warehouse claim, then requiring company- or warehouse-scoped permissions.

- Products: `products.view` for reads; `products.create` for creation; `products.update` for update and deactivation.
- Warehouses: `warehouses.view` for reads; `warehouses.manage` for create, update, activate, and deactivate.
- Stock: `stock.view`, `stock.in`, `stock.out`, and `stock.adjust` are checked per operation and warehouse.
- Transfers: `stock.transfer.view`, `stock.transfer.create`, `stock.transfer.post`, and `stock.transfer.cancel` are checked separately. Warehouse-scoped users can only access transfers involving their assigned warehouse.

The permission catalog contains 18 codes. `PermissionSeeder` seeds catalog entries; `CompanyRolePermissionSeeder` grants default permissions per company and role. The latter runs at startup for existing companies and during company provisioning for new companies. It adds missing grants idempotently.

Permission resolution checks an active user, membership, company and relevant warehouse, then an active permission. A user-specific allow/deny override takes precedence over the company's role grant. If neither grants access, the result is deny.

Default grants:
- CompanyModerator: all 18 catalog permissions.
- WarehouseModerator: 8 permissions — product view, warehouse view, stock view/in/out/adjust, transfer view/create.
- WarehouseEmployee: 5 permissions — product view, warehouse view, stock view/in/out.

The CompanyModerator role has no assigned WarehouseId. WarehouseModerator and WarehouseEmployee memberships require an assigned warehouse. A warehouse-scoped user is denied when a requested warehouse differs from that assignment.

## Inventory transactions and concurrency
Stock updates and transfer posting use database transactions. ProductStock.RowVersion is configured as a concurrency token. Transfer posting conditionally changes a Draft transfer to Posted before applying source and destination balance changes and paired movement records; a failed operation rolls back the transaction. Automatic retry behavior and concurrent posting stress coverage are not implemented.

## Current backend structure
```text
src/StockPilot.Api/
├── Controllers/
│   ├── Base/CompanyAuthorizedController.cs
│   ├── AuthController.cs
│   ├── CompaniesController.cs
│   ├── PlatformCompaniesController.cs
│   ├── ProductsController.cs
│   ├── StockController.cs
│   ├── StockTransfersController.cs
│   └── WarehousesController.cs
├── Data/
│   ├── Seeding/
│   └── StockPilotDbContext.cs
├── DTOs/
├── Enums/
├── Migrations/
├── Models/
├── Security/PermissionCatalog.cs
├── Services/
│   ├── Interfaces/
│   └── Implementations/
└── Program.cs
```

## Remaining boundaries
- CompanyId remains in tenant API routes; centralized trusted tenant-context resolution is not implemented.
- Automated API integration tests for authorization and cross-tenant access are not yet present.
- Employee provisioning, membership lifecycle, role administration, and permission-override management endpoints are not implemented.
- Production security review, concurrency stress tests, and deployment hardening remain outstanding.
