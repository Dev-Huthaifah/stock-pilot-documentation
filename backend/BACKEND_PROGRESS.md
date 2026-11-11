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
- `PATCH /{warehouseId}/activate` — reactivate a warehouse by setting `IsActive=true`; returns `204 No Content`.

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

### Stock management

Files:
- `DTOs/Stock/StockInDto.cs`
- `DTOs/Stock/StockOutDto.cs`
- `DTOs/Stock/StockAdjustmentDto.cs`
- `DTOs/Stock/StockResponseDto.cs`
- `DTOs/Stock/StockMovementResponseDto.cs`
- `Services/Interfaces/IStockService.cs`
- `Services/Implementations/StockService.cs`
- `Controllers/StockController.cs`

The service is registered in `Program.cs`. No migration was required; ProductStock, StockMovement, their relationships, constraints, indexes, and ProductStock.RowVersion were already configured in the DbContext.

Route base: `/api/companies/{companyId}/stock`

Implemented operations:
- `GET /` — list stock balances for the company, with optional `warehouseId` and `productId` filters.
- `GET /movements` — list stock movements, with optional `warehouseId` and `productId` filters; newest movements are returned first.
- `POST /in` — add stock.
- `POST /out` — remove stock, rejecting an operation that would make the balance negative.
- `POST /adjustment` — apply a signed quantity difference. Positive values increase stock; negative values decrease stock; zero is rejected.

Stock updates and movement creation are performed in a database transaction. The service validates that the company is active and that the product and warehouse belong to the company and are active. ProductStock uses its configured row-version concurrency token. A concurrent first insert for a product/warehouse balance is protected by the unique index; the current implementation reports the handled concurrency/unique-key conflict rather than automatically retrying it. StockMovement.Quantity is always positive; MovementType records the direction. CreatedBy remains null until authentication and user identity are implemented.

**Manual PowerShell API validation completed:**
- Stock In 100 — balance became 100.
- Stock In 50 more — same ProductStock row was updated to 150.
- Stock Out 30 — balance became 120.
- Stock Out 200 with insufficient balance — `409 Conflict`; negative stock was prevented.
- Adjustment +20 — balance became 140.
- Adjustment -10 — balance became 130.
- Adjustment 0 — `409 Conflict`.
- Stock movement query returned the expected five movements: StockIn 100, StockIn 50, StockOut 30, AdjustmentIn 20, AdjustmentOut 10. Movement quantities were positive.
- Final tested balance for the temporary product `STOCK-TEST-001` in warehouse `WH-001` was 130.
- Warehouse `WH-001` was reactivated through the new activate endpoint and verified with `IsActive=true`.

These are manual API integration checks using PowerShell `Invoke-RestMethod`; automated unit/integration test coverage has not been confirmed.

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
- Stock transfer service/controller and transfer API operations are not yet implemented.
- Stock posting uses transactions and row-version concurrency, but automatic retry behavior for concurrency conflicts is not implemented.
- Stock operations currently accept CompanyId from the route; trusted tenant resolution and authorization remain pending.
- Automated unit/integration tests have not been confirmed.

## Next backend tasks
1. Implement Stock Transfer DTOs, service interface/implementation, and controller using the existing StockTransfer/StockTransferItem models and DbContext constraints.
2. Implement atomic multi-item transfer posting: validate source/destination warehouses and available balances; update both warehouse balances; create paired TransferOut/TransferIn movements; transition Draft to Posted exactly once.
3. Add transfer cancellation rules (Draft only unless a deliberate reversal workflow is designed) and verify duplicate-post protection under repeated/concurrent requests.
4. Create a second active warehouse and run manual API tests for single-item and multi-item transfers, insufficient source stock, same-warehouse rejection, and repeated posting.
5. Design Authentication, Platform Admin, company membership, Main Moderator, roles, permissions, and trusted server-side tenant resolution.
6. Add automated unit/integration tests, consistent error handling, and authorization.

## Next-session instruction
Inspect the actual repository and working-tree state first. Company, Warehouse, Product, and Stock Management have been implemented; Warehouse activation and the Stock In/Out/Adjustment/movement-query flows have passed manual PowerShell API checks. The latest Stock Management changes were built and pushed by the user. Next, implement Stock Transfer Management using the existing StockTransfer and StockTransferItem models and configured composite relationships. Do not create the second test warehouse until beginning transfer testing. Preserve atomic multi-item posting, positive movement quantities with direction represented by MovementType, non-negative stock, and protection against duplicate posting. Authentication, trusted tenant resolution, and automated tests remain outstanding.
