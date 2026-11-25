# Database Design

## Core entities
- Company
- Warehouse
- Product
- ProductStock
- StockMovement
- StockTransfer
- StockTransferItem

Enums: `StockMovementType`, `StockTransferStatus`, and `CompanyRole`.

## Responsibilities
| Entity | Purpose |
|---|---|
| Company | Customer company/workspace |
| Warehouse | A company's warehouse |
| Product | A company's product |
| ProductStock | Stored quantity per product and warehouse |
| StockMovement | Inventory movement history |
| StockTransfer | Transfer header |
| StockTransferItem | Product/quantity within a transfer |
| ApplicationUser | ASP.NET Core Identity user |
| CompanyMembership | User membership, company role, and optional warehouse assignment |
| Permission | Global permission catalog entry |
| CompanyRolePermission | Company-specific permission grant for a role |
| CompanyUserPermissionOverride | Per-membership allow/deny override |

`Company.Id` is `Guid`. Tenant-owned records carry CompanyId. The model uses company-scoped composite keys/foreign keys, unique company-scoped warehouse codes and product SKUs, non-negative stock and prices, positive transfer item quantities, and a constraint that source and destination warehouses differ.

## Inventory decisions
- Current balance is stored in ProductStock.
- A transfer can contain multiple products.
- Warehouse transfers are in the first version.
- Transfers are created in Draft status and posted directly; no in-transit workflow is implemented.
- Posting deducts source stock, adds destination stock, and creates paired TransferOut and TransferIn movement records.
- StockMovement.Quantity is positive; MovementType identifies the direction.

## Enum values
`StockMovementType`: StockIn=1, StockOut=2, AdjustmentIn=3, AdjustmentOut=4, TransferOut=5, TransferIn=6.

`StockTransferStatus`: Draft=1, Posted=2, Cancelled=3.

`CompanyRole`:
- CompanyModerator=1
- WarehouseModerator=2
- WarehouseEmployee=3

## Identity and membership constraints
The DbContext derives from `IdentityDbContext<ApplicationUser, IdentityRole<Guid>, Guid>`. ApplicationUser extends IdentityUser with FullName, IsActive, CreatedAt, and MustChangePassword.

A unique index on `CompanyMembership.UserId` currently limits each user to one company membership. A composite foreign key ensures an assigned warehouse belongs to the same company. A check constraint requires CompanyModerator to have no WarehouseId, while WarehouseModerator and WarehouseEmployee require one.

Platform Admin is an ASP.NET Core Identity role (`PlatformAdmin`), not a CompanyMembership.

## Permission catalog
The global catalog contains 18 codes:
- Products: `products.view`, `products.create`, `products.update`
- Warehouses: `warehouses.view`, `warehouses.manage`
- Stock: `stock.view`, `stock.in`, `stock.out`, `stock.adjust`
- Transfers: `stock.transfer.view`, `stock.transfer.create`, `stock.transfer.post`, `stock.transfer.cancel`
- Employees: `employees.view`, `employees.create`, `employees.manage`, `employees.permissions.manage`
- Reports: `reports.view`

## Default company-role grants
`PermissionSeeder` inserts missing global catalog entries. `CompanyRolePermissionSeeder` grants role permissions for a company. It runs for all existing companies at application startup and is also called when a company or its initial moderator is provisioned. It checks for existing grants and inserts only missing entries; it does not delete or overwrite existing grants.

| Role | Default grants | Scope |
|---|---:|---|
| CompanyModerator | 18 | All catalog permissions |
| WarehouseModerator | 8 | Products/warehouses/stock view; stock in/out/adjust; transfer view/create |
| WarehouseEmployee | 5 | Products/warehouses/stock view; stock in/out |

The current seed set creates 31 role-permission grants per company. User-specific overrides are evaluated before role grants; an explicit allow or deny override takes precedence. If there is no override or role grant, permission evaluation denies access.

## Migrations
Known migrations:
- `20261030212054_InitialCreate`
- `20261030214454_AddProductPriceConstraints`

Both were reported applied to the local database. Verify actual migration history and current model before schema changes. This document is a summary; inspect entities, DbContext, migrations, and database history.
