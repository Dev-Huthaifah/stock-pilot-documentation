# Database Design

## Core entities
- Company
- Warehouse
- Product
- ProductStock
- StockMovement
- StockTransfer
- StockTransferItem

Enums: `StockMovementType`, `StockTransferStatus`.

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

`Company.Id` is `Guid`. The current model includes company ownership on tenant-owned records, composite company-scoped keys/foreign keys, unique company-scoped warehouse codes and product SKUs, non-negative stock and prices, positive transfer item quantities, and a constraint that source and destination warehouses differ.

## Inventory decisions
- Current balance is stored in ProductStock.
- A transfer can contain multiple products.
- Warehouse transfers are in the first version.
- Transfers are posted directly on approval; no in-transit workflow is planned initially.
- Transfer posting should create transfer-out and transfer-in movement records.

## Enum values
`StockMovementType`: StockIn=1, StockOut=2, AdjustmentIn=3, AdjustmentOut=4, TransferOut=5, TransferIn=6.

`StockTransferStatus`: Draft=1, Posted=2, Cancelled=3.

## Migrations
Known migrations:
- `20261030212054_InitialCreate`
- `20261030214454_AddProductPriceConstraints`

Both were reported applied to the local database. Verify actual migration history and current model before schema changes. This document is a summary; inspect entities, DbContext, migrations, and database history.


## Identity, memberships, and permissions

The DbContext derives from `IdentityDbContext<ApplicationUser, IdentityRole<Guid>, Guid>`. `ApplicationUser` extends IdentityUser with `FullName`, `IsActive`, `CreatedAt`, and `MustChangePassword`.

Current identity and authorization entities:
- `CompanyMembership`: links one user to one company, optional assigned warehouse, company role, active flag, and creation time.
- `Permission`: global permission catalog entry with unique code, name, category, description, and active flag.
- `CompanyRolePermission`: company-specific grant keyed by CompanyId + Role + PermissionId.
- `CompanyUserPermissionOverride`: per-membership allow/deny override keyed by CompanyMembershipId + PermissionId.

A unique index on `CompanyMembership.UserId` currently limits each user to one company membership. A composite foreign key ensures an assigned warehouse belongs to the same company. The membership check constraint requires `CompanyModerator` to have no WarehouseId, while `WarehouseModerator` and `WarehouseEmployee` require a WarehouseId.

Company roles:
- `CompanyModerator = 1`
- `WarehouseModerator = 2`
- `WarehouseEmployee = 3`

The permission catalog currently defines 18 codes: `products.view/create/update`, `warehouses.view/manage`, `stock.view/in/out/adjust`, `stock.transfer.view/create/post/cancel`, `employees.view/create/manage/permissions.manage`, and `reports.view`.

`PermissionSeeder` inserts missing catalog entries idempotently. It does not seed company-role grants. `PermissionService` applies user override first, then company-role grant, and defaults to deny. Default role grants, endpoint-level enforcement, and trusted tenant resolution remain unfinished.

Platform Admin is represented by an ASP.NET Core Identity role (`PlatformAdmin`), not by a `CompanyMembership`.
