# Project Context

## Product
Stock Pilot is a web-based inventory and product management system for shops, wholesalers, retailers, distributors, and SMEs.

## Multi-company model
- Each customer company gets its own logically isolated workspace, named for the company (for example, Oliva Stock Pilot).
- A Platform Admin manages customer companies on the platform.
- The Platform Admin creates a company and invites its designated Main Moderator.
- The Main Moderator manages that company's users, roles, and permissions.
- Company users access only the resources their permissions allow.
- A company must never access another company's data.

Authentication is now implemented with ASP.NET Core Identity and JWT. Platform Admin accounts are bootstrapped from configuration secrets; company administrators are provisioned with generated temporary passwords and must change them at first login. The current company roles are CompanyModerator, WarehouseModerator, and WarehouseEmployee. The permission catalog and permission-evaluation service exist. Trusted tenant resolution and complete endpoint-level authorization are still unfinished; subscription billing and custom domains remain undecided.

## Planned capabilities
Company management, users and roles, warehouses, products, stored stock balances, stock movements, multi-item transfers, reporting, and later AI-assisted workflows.

## Development method
Use one application repository and separate focused workstreams/conversations for backend, frontend, AI, testing/security, deployment, and integration. Maintain this documentation as the durable project record.
