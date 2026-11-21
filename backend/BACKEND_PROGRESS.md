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

### Stock Transfer Management

Files:
- `DTOs/StockTransfers/CreateStockTransferDto.cs`
- `DTOs/StockTransfers/CreateStockTransferItemDto.cs`
- `DTOs/StockTransfers/StockTransferItemResponseDto.cs`
- `DTOs/StockTransfers/StockTransferResponseDto.cs`
- `Services/Interfaces/IStockTransferService.cs`
- `Services/Implementations/StockTransferService.cs`
- `Controllers/StockTransfersController.cs`

The service is registered in `Program.cs`. No migration was required because the StockTransfer and StockTransferItem entities, relationships, constraints, and indexes were already configured in the DbContext.

Route base: `/api/companies/{companyId}/stock-transfers`

Implemented operations:
- `POST /` — create a multi-item transfer in Draft status without changing stock.
- `GET /` — list transfers scoped by CompanyId.
- `GET /{transferId}` — retrieve transfer details.
- `POST /{transferId}/post` — post a transfer, update source and destination balances, create paired TransferOut/TransferIn movements, and set Posted status.
- `POST /{transferId}/cancel` — cancel Draft transfers only.

Transfer creation rejects identical source/destination warehouses, duplicate products within one transfer, invalid quantities, inactive companies/warehouses/products, and resources outside the company. Transfer numbers use GUID-based identifiers.

Posting uses a database transaction and a conditional Draft-to-Posted update to prevent repeated posting. Stock movements use positive quantities; `MovementType` indicates direction. Each transfer item is linked to its paired TransferOut and TransferIn movements through `StockTransferItemId`. ProductStock row-version concurrency and database unique constraints protect stock integrity. A failed posting rolls back the transfer status and stock changes.

**Manual PowerShell API validation completed:**
- Created a transfer for 30 units — `201 Created`, status Draft.
- Verified stock did not change before posting.
- Posted the transfer — status Posted.
- Source warehouse balance changed from 130 to 100.
- Destination warehouse balance changed from 0 to 30.
- Two linked movements were created: TransferOut (type 5) and TransferIn (type 6), both with positive quantities and the same StockTransferItemId.
- Repeated posting returned `409 Conflict`.
- Created an insufficient-stock transfer for 1000 units; posting was rejected.
- Cancelled the failed Draft transfer — status Cancelled.
- Verified final balances remained 100 in WH-001 and 30 in WH-002.

These are manual PowerShell API integration checks. Automated unit/integration test coverage and concurrent posting stress tests have not been confirmed.
 

### Authentication and identity (implemented)

Files verified in the application repository:
- `Models/ApplicationUser.cs`
- `Models/CompanyMembership.cs`
- `Models/CompanyRolePermission.cs`
- `Models/CompanyUserPermissionOverride.cs`
- `Models/Permission.cs`
- `Enums/CompanyRule.cs` (declares `CompanyRole`)
- `Data/StockPilotDbContext.cs`
- `Data/Seeding/PlatformAdminBootstrapper.cs`
- `Data/Seeding/PermissionSeeder.cs`
- `Security/PermissionCatalog.cs`
- `Services/Implementations/AuthService.cs`
- `Services/Implementations/TokenService.cs`
- `Services/Implementations/PermissionService.cs`
- `Services/Implementations/PlatformCompanyService.cs`
- `Controllers/AuthController.cs`
- `Controllers/PlatformCompaniesController.cs`

Identity:
- `ApplicationUser` extends `IdentityUser<Guid>` and adds `FullName`, `IsActive`, `CreatedAt`, and `MustChangePassword`.
- Identity uses EF Core stores and Guid keys. Email uniqueness is enabled.
- Password policy: at least 10 characters, uppercase, lowercase, digit, and non-alphanumeric character.
- Lockout is enabled after 5 failed attempts for 15 minutes. Email confirmation is not required in the current development configuration.
- `PlatformAdminBootstrapper` creates the `PlatformAdmin` Identity role and optionally bootstraps the first platform admin from configuration keys `PlatformAdmin:Email`, `PlatformAdmin:Password`, and `PlatformAdmin:FullName`. Keep these values in User Secrets locally and a production secret store later.

JWT:
- JWT Bearer validates issuer, audience, signing key, and token lifetime; inbound claim mapping is disabled.
- Configuration keys: `Jwt:Secret` (Base64-encoded key, minimum 32 bytes), `Jwt:Issuer`, `Jwt:Audience`, and `Jwt:ExpirationMinutes`.
- Company-user JWT includes `sub`, `jti`, `membership_id`, optional email, and optional `warehouse_id`.
- Platform Admin JWT includes `sub`, `jti`, `role=PlatformAdmin`, and `account_type=platform_admin`.
- `PlatformAdminOnly` requires an authenticated principal with the `PlatformAdmin` role.
- Temporary-password challenge JWT includes `token_use=password_change`, expires after five minutes, and is rejected by JWT bearer validation on routes other than `/api/auth/change-temporary-password`. The controller also rejects normal JWTs at that endpoint.

Authentication endpoints:
- `POST /api/auth/login` — accepts email/password. Returns `accessToken`, `tokenType`, `expiresAt`, and `requiresPasswordChange`.
- `POST /api/auth/change-temporary-password` — requires the challenge JWT and `currentPassword` + `newPassword`; on success clears `MustChangePassword` and returns a normal JWT.
- Inactive users, missing/inactive memberships, inactive companies, and inactive assigned warehouses are rejected during company-user login and temporary-password change.
- Platform Admin login currently issues a normal platform token and does not use the company-user temporary-password branch.

Company provisioning:
- `POST /api/platform/company-provisioning` — PlatformAdminOnly; creates a company and initial `CompanyModerator` in a database transaction. Generates a 24-character temporary password and sets `MustChangePassword=true`. The response contains the temporary password once.
- `POST /api/platform/company-provisioning/{companyId}/admin` — PlatformAdminOnly; creates a moderator for an existing active company if it has no active `CompanyModerator`. It rejects unknown/inactive companies, duplicate email, or an existing active moderator.
- Both operations create a company membership with `CompanyModerator`, no assigned warehouse, and active status.

Membership and permissions:
- Roles: `CompanyModerator=1`, `WarehouseModerator=2`, `WarehouseEmployee=3`.
- A unique index on `CompanyMembership.UserId` currently allows one company membership per user.
- Check constraint: CompanyModerator must have no WarehouseId; WarehouseModerator and WarehouseEmployee must have one.
- The 18-code permission catalog is seeded idempotently by `PermissionSeeder`.
- `PermissionService` verifies active user/membership/company and assigned warehouse, then checks active permission, user override, and company-role grant in that order; default is deny.
- **Not yet implemented:** default `CompanyRolePermission` grants for new companies. Permission evaluation is implemented and has 17 passing unit tests reported earlier, but inventory controllers do not yet consistently call the permission service.

Manual authentication verification (PowerShell, local API):
- Platform Admin login returned Bearer JWT with `RequiresPasswordChange=false`.
- Newly provisioned company moderator login with temporary password returned `RequiresPasswordChange=true`.
- Challenge JWT used against `GET /api/platform/companies` returned `401 Unauthorized`.
- Temporary-password change succeeded and returned a normal JWT with `RequiresPasswordChange=false`.
- A fresh login with the new password succeeded and returned `RequiresPasswordChange=false`.
- Company-user JWT used against `GET /api/platform/companies` returned `403 Forbidden`.
- The user reported successful `dotnet build StockPilot.slnx` during this work. Re-run current build and test suite before the next milestone; do not infer a fresh test result from the manual HTTP checks.

### Current authorization gaps — critical

- `CompaniesController` is protected by `PlatformAdminOnly`, and `PlatformCompaniesController` is protected by the same policy.
- The current `POST /api/platform/companies` action still creates a company through the older `ICompanyService` path without creating an initial moderator. Remove or disable this legacy create action; use `POST /api/platform/company-provisioning` for provisioning.
- `WarehousesController`, `ProductsController`, `StockController`, and `StockTransfersController` currently have no controller-level `[Authorize]` attribute and do not visibly call `IPermissionService`. These routes must not be treated as protected.
- CompanyId is still supplied in tenant API routes. Derive and validate tenant context from the authenticated membership before production use; warehouse-scoped users must be restricted to their assigned warehouse.
- Do not expose the inventory API publicly until endpoint authorization, permission checks, and cross-tenant tests are complete.

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
- Platform Admin Identity role, JWT authentication, company memberships, roles, permission catalog, and permission evaluation are implemented.
- Company moderator provisioning and temporary-password onboarding are implemented; email/invitation delivery is not implemented.
- Default company-role permission grants and endpoint-level permission enforcement are pending.
- Trusted server-side tenant resolution is pending.
- Duplicate-SKU race-condition exception handling is pending.
- Stock posting and transfer posting use transactions and row-version concurrency, but automatic retry behavior for concurrency conflicts is not implemented. Concurrent posting stress tests remain outstanding.
- Stock operations currently accept CompanyId from the route; trusted tenant resolution and authorization remain pending.
- Automated unit/integration tests have not been confirmed.

## Next backend tasks
1. Add an idempotent seeder for default CompanyRolePermission grants; define least-privilege grants for CompanyModerator, WarehouseModerator, and WarehouseEmployee.
2. Remove or disable the legacy POST /api/platform/companies action so all new company creation provisions an initial CompanyModerator.
3. Protect WarehousesController, ProductsController, StockController, and StockTransfersController with authentication and per-operation IPermissionService checks.
4. Derive tenant identity from authenticated membership and enforce assigned-warehouse scope; never trust route CompanyId as authorization.
5. Add automated auth/provisioning integration tests, challenge-token expiry/replay tests, endpoint authorization tests, and cross-tenant isolation tests.
6. Add employee provisioning, membership lifecycle, role management, and permission-override management endpoints.
7. Review concurrency edge cases, consistent API error handling, audit logging, and security before frontend integration.
8. Prepare localization-ready API validation/error responses where needed; keep user-facing Arabic/English presentation and RTL layout in the frontend.

## Next-session instruction
Inspect the actual repository and working-tree state first. Company, Warehouse, Product, Stock, and Stock Transfer Management have been implemented. Stock Transfer manual PowerShell checks confirmed Draft creation without stock changes, successful posting with paired movements, repeated-post rejection (409), insufficient-stock rejection, Draft cancellation, and unchanged balances after failed posting. The user reported committing the main application project; verify the commit and remote state rather than assuming a particular commit hash. Authentication, JWT, Platform Admin bootstrap, company moderator provisioning, temporary-password onboarding, company membership, permission catalog, and permission evaluation now exist. Next, seed default company-role permissions, remove the legacy company-creation path, enforce permissions on all tenant-owned controllers, and derive tenant/warehouse scope from authenticated membership. Automated auth/integration and tenant-isolation tests remain outstanding. The frontend is planned to be Arabic-first; backend data storage already supports Unicode, while localization-ready validation/error responses should be addressed consistently without duplicating translated fields in domain models.
