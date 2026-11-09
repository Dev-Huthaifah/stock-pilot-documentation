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
The product behavior is agreed: isolated company workspaces. Current persistence uses CompanyId and composite keys/foreign keys for tenant-owned records. Full authentication, authorization, trusted tenant resolution, and comprehensive tenant-scoped query enforcement are not yet implemented.

Never treat a client-supplied CompanyId as proof of authorization. Resolve tenant context from authenticated server-side identity/membership once authentication is implemented. Test cross-tenant access for every tenant-owned resource.

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
