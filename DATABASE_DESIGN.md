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
