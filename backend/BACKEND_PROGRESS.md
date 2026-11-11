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
Files:
- `DTOs/Companies/CreateCompanyDto.cs`
- `DTOs/Companies/UpdateCompanyDto.cs`
- `DTOs/Companies/CompanyResponseDto.cs`
- `Services/Interfaces/ICompanyService.cs`
- `Services/Implementations/CompanyService.cs`
- `Controllers/CompaniesController.cs`

Operations: create, get by ID, list, update, deactivate. Route base: `/api/platform/companies`. Company deactivation sets `IsActive=false`. The service is registered in Program.cs.

**Important:** `Company.Id` is `Guid`, not `int`. Use Guid consistently for company IDs.

### Warehouse management
Files:
- `DTOs/Warehouses/CreateWarehouseDto.cs`
- `DTOs/Warehouses/UpdateWarehouseDto.cs`
- `DTOs/Warehouses/WarehouseResponseDto.cs`
- `Services/Interfaces/IWarehouseService.cs`
- `Services/Implementations/WarehouseService.cs`
- `Controllers/WarehousesController.cs`

The service is registered in `Program.cs`. No migration was required; the Warehouse entity and its database configuration already existed.

Route base: `/api/companies/{companyId}/warehouses`

Implemented operations:
- `POST /` — create a warehouse for an active company.
- `GET /` — list warehouses scoped by CompanyId.
- `GET /{warehouseId}` — retrieve a warehouse scoped by CompanyId and warehouse ID.
- `PUT /{warehouseId}` — update name, code, and address.
- `DELETE /{warehouseId}` — soft-deactivate by setting `IsActive=false`; returns `204 No Content`.

Warehouse codes are checked for duplicates within the company, and the database's unique company/code index remains the final integrity constraint. DTO validation includes required name/code and maximum lengths. Names and codes are trimmed; blank addresses are normalized to null.

### Product management
Files:
- `DTOs/Products/CreateProductDto.cs`
- `DTOs/Products/UpdateProductDto.cs`
- `DTOs/Products/ProductResponseDto.cs`
- `Services/Interfaces/IProductService.cs`
- `Services/Implementations/ProductService.cs`
- `Controllers/ProductsController.cs`

The service is registered in `Program.cs`. No migration was required; the Product entity, company-scoped alternate key, unique company/SKU index, price precision, and non-negative price check constraints already existed in `StockPilotDbContext`.

Route base: `/api/companies/{companyId}/products`

Implemented operations:
- `POST /` — create a product for an active company.
- `GET /` — list products scoped by CompanyId; inactive products are excluded by default.
- `GET /{productId}` — retrieve a product scoped by CompanyId and product ID.
- `PUT /{productId}` — update SKU, name, description, unit, cost price, and selling price.
- `DELETE /{productId}` — soft-deactivate by setting `IsActive=false`; returns `204 No Content`.
- `GET /?includeInactive=true` — include inactive products in the list.

Product DTOs validate required fields and maximum lengths. CostPrice and SellingPrice are non-negative decimals. The service trims SKU, name, and unit, and normalizes blank descriptions to null. SKU uniqueness is checked within the company. Product IDs and CompanyIds are not accepted from the request body.

**Current behavior and limitations:**
- `GetAllAsync` does not verify that the company exists; an unknown CompanyId currently returns an empty list.
- `GetByIdAsync` can return an inactive product. The list excludes inactive products by default unless `includeInactive=true`.
- Deactivation is idempotent: deactivating an already inactive product still succeeds.
- The service checks for duplicate SKU before saving, but a concurrent request can still race. The database unique index is the final integrity constraint; translating a resulting unique-index `DbUpdateException` into `409 Conflict` remains to be implemented.
- The update operation currently does not reject an inactive product; it can still be updated if its ID and CompanyId match.

## Tenant isolation note
CompanyId is currently supplied explicitly in the route for early development and testing. It is a scoping parameter, not proof of identity or authorization. Production requests must derive the tenant from a trusted authenticated server-side context.

## Last known validation
- `dotnet build StockPilot.slnx` succeeded after implementing Company, Warehouse, and Product Management.
- API was tested locally at `http://localhost:5130`.
- Manual PowerShell API tests succeeded for warehouse create, list, get by ID, update, and deactivate.
- A follow-up GET confirmed the deactivated warehouse remains retrievable with `IsActive=false`. This is the current soft-delete behavior; Warehouse GET endpoints do not currently filter inactive warehouses.
- Manual PowerShell API tests for Product Management all succeeded with expected results:
  - Create product — `201 Created`.
  - List products — `200 OK`.
  - Create another product with the same company-scoped SKU — `409 Conflict`.
  - Update product — `200 OK`.
  - Deactivate product — `204 No Content`.
  - Verify soft delete — inactive product is hidden by default and returned when `includeInactive=true`, with `IsActive=false`.
- The Product tests were executed with PowerShell `Invoke-RestMethod` / `Invoke-WebRequest`; Swagger was not used.
- These were manual API integration checks; automated unit/integration test coverage has not been confirmed.
- No database migration was needed for Product Management.
- Verify the current API configuration and database state before relying on the local environment.

## Recent commits
- `7ef199b` — `implement warehouse management CRUD`
- `71e7c12` — `implement company management service`
- `ef95fd6` — `finalize inventory data model and constraints`

At the time of the previous documentation update, commit `7ef199b` was present on `origin/main`. Re-check local state when continuing:
```powershell
git status --short
git log -5 --oneline
git remote -v
```

## Known unfinished work / limitations
- Company, warehouse, and product endpoints are not yet protected by authentication/authorization; do not expose publicly in production.
- Platform Admin identity/authorization is not implemented.
- Main Moderator invitation/account setup is not implemented.
- Company membership, roles, and permissions are not implemented.
- Trusted server-side tenant resolution is pending.
- Duplicate-SKU race-condition exception handling is pending.
- Product stock, stock movement, and stock transfer services/controllers are not yet implemented.
- Full transaction/concurrency handling for stock posting is pending.
- Automated unit/integration tests have not been confirmed.

## Next backend tasks
1. Inspect `ProductStock.cs` and `StockMovement.cs`, including their relationships and constraints, before implementing inventory operations.
2. Design Product Stock queries and stock-in/stock-out operations; do not expose arbitrary direct quantity editing.
3. Implement inventory movements with atomic quantity updates, non-negative stock enforcement, and appropriate concurrency handling.
4. Implement atomic multi-item stock transfer posting with appropriate transaction and concurrency handling.
5. Design Authentication, Platform Admin, company membership, Main Moderator, roles, and permissions.
6. Add validation, consistent error handling, automated tests, and authorization.

## Next-session instruction
Inspect the actual repository and working-tree state first. Review `ProductStock.cs`, `StockMovement.cs`, their DbContext configuration, and the existing service/controller patterns. Then design Product Stock and inventory movement operations around auditable stock movements, atomic updates, company scoping, and concurrency safety. Do not assume ProductStock CRUD should allow direct quantity edits.
