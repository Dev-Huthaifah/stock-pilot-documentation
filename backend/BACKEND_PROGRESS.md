# Backend Progress and Handoff

## Stack
ASP.NET Core Web API, C#/.NET 10, EF Core 10, SQL Server; LocalDB for local development.

## Paths
- Solution: `StockPilot.slnx`
- API: `src/StockPilot.Api/StockPilot.Api.csproj`
- DbContext: `src/StockPilot.Api/Data/StockPilotDbContext.cs`
- Models: `src/StockPilot.Api/Models/`
- Migrations: `src/StockPilot.Api/Migrations/`

## Implemented
### Persistence
Seven core entities exist: Company, Warehouse, Product, ProductStock, StockMovement, StockTransfer, StockTransferItem. DbContext configures relationships, company-scoped composite keys/foreign keys, indexes, and check constraints. ProductStock.RowVersion is configured as a row-version concurrency token.

Migrations `InitialCreate` and `AddProductPriceConstraints` were reported applied locally. The database health endpoint previously returned `{"status":"Connected"}`.

### Company management
Files created:
- `DTOs/Companies/CreateCompanyDto.cs`
- `DTOs/Companies/UpdateCompanyDto.cs`
- `DTOs/Companies/CompanyResponseDto.cs`
- `Services/Interfaces/ICompanyService.cs`
- `Services/Implementations/CompanyService.cs`
- `Controllers/CompaniesController.cs`

Operations: create, get by ID, list, update, deactivate. Route base: `/api/platform/companies`. Company deactivation sets IsActive=false. The service is registered in Program.cs.

**Important:** `Company.Id` is `Guid`, not `int`. Use Guid consistently for company IDs.

## Last known validation
`dotnet build StockPilot.slnx` succeeded after correcting the Guid/int mismatch. API previously ran at `http://localhost:5130`; verify current configuration.

## Recent commits (snapshot)
- `ef95fd6` — `finalize inventory data model and constraints`
- `71e7c12` — `implement company management service`

At documentation time, both commits were local and not pushed to GitHub. Working tree was clean after `71e7c12`. Re-check:
```powershell
git status --short
git log -5 --oneline
git remote -v
```

## Known unfinished work / limitations
- Company endpoints are not yet protected by authentication/authorization; do not expose publicly in production.
- Platform Admin identity/authorization is not implemented.
- Main Moderator invitation/account setup is not implemented.
- Company membership, roles, and permissions are not implemented.
- Trusted server-side tenant resolution is pending.
- Validation and consistent error handling need strengthening.
- Warehouse, Product, Stock, Movement, and Transfer services/controllers are not yet implemented.
- Full transaction/concurrency handling for stock posting is pending.
- Automated unit/integration tests have not been confirmed.

## Next backend tasks
1. Implement Warehouse DTOs, service interface, implementation, and controller.
2. Scope every warehouse operation by CompanyId. Explicit CompanyId used during early development is not secure tenant identity; replace it with authenticated tenant context later.
3. Implement Product management.
4. Design Authentication, Platform Admin, company membership, Main Moderator, roles, and permissions.
5. Implement inventory operations, movements, and atomic multi-item transfer posting.
6. Add validation, error handling, tests, and authorization.

## Next-session instruction
Inspect actual repository state first. Review CompanyService, ICompanyService, CompaniesController, Program.cs, DbContext, and entity definitions. Then implement Warehouse management following the DTO → Service → Controller pattern.
