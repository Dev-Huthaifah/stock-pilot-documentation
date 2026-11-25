# Decision Log

## D-001 — Multi-tenant workspaces
**Status:** Agreed. Each customer company has its own logically isolated, company-named Stock Pilot workspace.

## D-002 — Platform Admin and Company Moderator
**Status:** Implemented for platform provisioning. Platform Admin is an ASP.NET Core Identity role protected by the `PlatformAdminOnly` policy. Platform Admin can provision a company together with its initial `CompanyModerator`, or create a moderator for an existing active company that has no active moderator. Provisioning generates a temporary password and forces a password change at first login. Full company-user management and invitation delivery are not implemented.

## D-003 — Shared database with company-scoped records
**Status:** Implemented in the current model and enforced at tenant endpoints through active membership and CompanyId checks. CompanyId remains an explicit route parameter; centralized trusted tenant-context resolution is still a future improvement.

## D-004 — Stored stock balance
**Status:** Agreed and implemented. ProductStock stores current quantity. Stock-changing workflows update balances and movement history within database transactions.

## D-005 — Multi-item transfers
**Status:** Agreed and implemented. One transfer may contain multiple product items.

## D-006 — Direct transfer posting
**Status:** Implemented. Transfers are created as Draft. Posting applies source deduction and destination addition directly and creates paired TransferOut/TransferIn movements; no in-transit state is used.

## D-007 — Company deactivation
**Status:** Implemented. Deactivation sets IsActive=false rather than physically deleting the company.

## D-008 — DTO and service layers
**Status:** Implemented. Controllers use DTOs and services rather than exposing EF entities directly.

## D-009 — Identity and JWT authentication
**Status:** Implemented. ASP.NET Core Identity uses Guid keys and EF Core storage. JWT bearer authentication validates issuer, audience, signing key, and lifetime. Platform Admin and company-user tokens have distinct claims and authorization boundaries. JWT and bootstrap credentials are configuration secrets and must not be committed.

## D-010 — Temporary password onboarding
**Status:** Implemented and manually verified. Company provisioning generates a 24-character temporary password and sets `MustChangePassword=true`. First login returns a five-minute password-change-only JWT. The token is rejected on other endpoints. Changing the password clears the flag and returns a normal JWT.

## D-011 — One company membership per user
**Status:** Implemented in the current schema. A unique index on `CompanyMembership.UserId` limits a user to one company membership. Warehouse-scoped roles require an assigned warehouse; CompanyModerator must have no assigned warehouse.

## D-012 — Permission evaluation precedence
**Status:** Implemented. The permission catalog contains 18 codes. Permission evaluation requires an active user, membership, company, and relevant active warehouse/permission. A user-specific allow/deny override takes precedence over the company's role grant; if neither exists, access is denied.

## D-013 — Default company-role permission grants
**Status:** Implemented. `CompanyRolePermissionSeeder` grants 18 permissions to CompanyModerator, 8 to WarehouseModerator, and 5 to WarehouseEmployee. The seeder is idempotent, seeds existing companies at application startup, and is called during company provisioning for new companies. Existing grants are preserved; only missing grants are added.

## D-014 — Endpoint authorization and warehouse scope
**Status:** Implemented for current company inventory endpoints. Tenant controllers inherit `[Authorize]` from `CompanyAuthorizedController` and check permissions per operation. Warehouse-scoped users are restricted to their assigned warehouse for warehouse/stock/transfer operations. Platform company-management and provisioning endpoints require `PlatformAdminOnly`. Automated API integration coverage and centralized tenant-context resolution remain outstanding.
