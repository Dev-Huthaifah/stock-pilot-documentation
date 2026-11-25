# Testing and Security Plan

## Latest automated test result
The latest local run of `dotnet test .\StockPilot.slnx` completed successfully:
- Total: 27
- Passed: 27
- Failed: 0
- Skipped: 0

The current automated suite includes permission-service tests and company-role permission seeder tests using EF Core InMemory. This is unit-level coverage; it is not a substitute for HTTP/API integration tests against SQL Server.

## Manual authorization verification
The following endpoint checks were performed locally with PowerShell and JWTs:

### Warehouse scope
- CompanyModerator could list all company warehouses.
- WarehouseModerator could list only the assigned Main Warehouse.
- WarehouseEmployee could list only the assigned Secondary Warehouse.
- Warehouse-scoped users were denied access to stock in another warehouse.
- WarehouseEmployee could not adjust stock (403), create transfers (403), list transfers (403), or create/update/deactivate products (403).
- WarehouseEmployee could record stock-in and stock-out operations in the assigned warehouse.
- WarehouseModerator could adjust stock and create a transfer involving the assigned warehouse.
- WarehouseModerator could not post or cancel a transfer (403).
- WarehouseModerator could not manage warehouses (create/update/deactivate); those operations returned 403.

### Company boundary
Cross-company product, warehouse, stock, stock-movement, and transfer requests were rejected with 403 in the tested scenarios, including attempts to use a different company's CompanyId.

### CompanyModerator operations
CompanyModerator successfully created, updated, and deactivated products, and created, updated, deactivated, and reactivated warehouses in its company.

These are manual checks, not automated integration tests. They demonstrate the tested scenarios only and do not constitute a complete security review.

## Automated test priorities
- HTTP/API authorization tests for unauthenticated requests and each role/permission combination.
- Cross-company isolation tests for every tenant-owned endpoint and resource.
- Warehouse-scope tests for list, get-by-ID, stock operations, movements, and transfers.
- Platform Admin policy tests, provisioning transaction tests, and temporary-password challenge-token expiry/replay tests.
- SQL Server integration tests for migrations, constraints, transactions, row-version concurrency, and concurrent first stock inserts.
- Transfer posting concurrency/stress tests and rollback verification.
- Regression tests for inactive users, memberships, companies, warehouses, and permissions.

## Production security requirements
- Keep JWT signing keys and Platform Admin bootstrap credentials in a secret store; never commit them.
- Require HTTPS in production and configure secure deployment defaults.
- Review rate limiting, safe error responses, audit logging for administrative and inventory operations, backup/restore, and dependency vulnerabilities.
- Keep CompanyId membership validation and assigned-warehouse checks on every tenant-owned operation. CompanyId in a route is a selector, not authorization.
- Review whether centralized tenant-context resolution should replace route-only tenant selection.
- Complete a security review before exposing the API publicly.

A feature is not done merely because it compiles. It needs relevant tests, tenant-boundary checks, error handling, and documentation.
