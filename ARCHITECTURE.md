# Architecture

## Backend direction
ASP.NET Core Web API organized into:
- Controllers: HTTP endpoints and status codes.
- DTOs: request/response contracts.
- Services: application and business operations.
- Data: EF Core DbContext and persistence configuration.
- Models: entities.
- Enums: constrained domain values.
- Migrations: schema evolution.

Expected request flow:
HTTP Client → Controller → Service Interface/Implementation → DbContext → SQL Server.

Controllers should remain thin. Do not expose EF entities directly as API contracts.

## Multi-tenancy
The product behavior is agreed: isolated company workspaces. Current persistence uses CompanyId and composite keys/foreign keys for tenant-owned records. ASP.NET Core Identity, JWT authentication, Platform Admin authorization, company memberships, and a permission-evaluation service are implemented. However, tenant-owned inventory controllers still do not consistently enforce authentication or permission checks, and CompanyId remains a route parameter. Trusted tenant resolution and comprehensive endpoint-level enforcement are not complete.

Never treat a client-supplied CompanyId as proof of authorization. Resolve tenant context from authenticated server-side identity/membership once authentication is implemented. Test cross-tenant access for every tenant-owned resource.


## Authentication, provisioning, and authorization

Authentication uses ASP.NET Core Identity with `ApplicationUser : IdentityUser<Guid>` and JWT bearer tokens. The API validates issuer, audience, signing key, and token lifetime. Company-user tokens carry the subject, token ID, membership ID, and assigned warehouse ID when applicable. Platform Admin tokens carry the `PlatformAdmin` role and a platform account-type claim.

Platform Admin is a separate Identity role, not a company membership. The `PlatformAdminOnly` policy protects platform company-management and provisioning controllers. Platform Admin bootstrap reads `PlatformAdmin:Email`, `PlatformAdmin:Password`, and `PlatformAdmin:FullName` from configuration; secrets must remain outside source control.

Company provisioning creates a company and its initial `CompanyModerator` membership in one database transaction. It generates a 24-character temporary password and marks the user `MustChangePassword=true`. A separate provisioning operation can create a moderator for an existing company when it has no active CompanyModerator. The first successful login returns a five-minute JWT containing `token_use=password_change`; that token is rejected on other routes and accepted only by the temporary-password-change endpoint. Successful password change clears the flag and returns a normal JWT.

The permission catalog contains 18 permission codes. `PermissionSeeder` seeds the catalog only; it does not grant role permissions. `PermissionService` checks active user, membership, company, warehouse assignment, active permission, then user override before company-role grant; default is deny. Default role grants and endpoint integration are still pending.

**Security boundary:** the existence of Identity, JWT, and `PermissionService` does not mean all business APIs are protected. Warehouse, product, stock, and transfer controllers currently lack consistent `[Authorize]` and permission enforcement. Route-supplied `companyId` is not a trusted tenant identity. Do not expose these APIs publicly until endpoint authorization and tenant resolution are completed.

## Transactions and concurrency
Stock updates and transfer posting must be atomic. ProductStock.RowVersion is configured as a concurrency token. Full transaction/concurrency behavior remains to be implemented and tested in inventory workflows.

## Current backend structure
```text
src/StockPilot.Api/
├── Controllers/
├── Data/
├── DTOs/
│   ├── Companies/
│   ├── Warehouses/
│   └── Products/
├── Enums/
├── Migrations/
├── Models/
├── Services/
│   ├── Interfaces/
│   └── Implementations/
└── Program.cs
```
This is the intended structure; some features/folders may be incomplete.
