# Decision Log

## D-001 — Multi-tenant workspaces
**Status:** Agreed. Each customer company has its own logically isolated, company-named Stock Pilot workspace.

## D-002 — Platform Admin and Company Moderator
**Status:** Partially implemented. Platform Admin is an ASP.NET Core Identity role protected by the `PlatformAdminOnly` policy. Platform Admin can provision a company together with its initial `CompanyModerator`, or create a moderator for an existing company that has no active moderator. Provisioning generates a temporary password and forces a password change at first login. Full company-user management and invitation delivery are not implemented.

## D-003 — Shared database with company-scoped records
**Status:** Current model implemented in part; full enforcement pending. Existing schema uses CompanyId and composite keys/foreign keys. Application-level tenant authorization is still required.

## D-004 — Stored stock balance
**Status:** Agreed and represented in model. ProductStock stores current quantity. Stock-changing workflows must update balances and movement history consistently.

## D-005 — Multi-item transfers
**Status:** Agreed and represented in model. One transfer may contain multiple product items.

## D-006 — Direct transfer posting
**Status:** Agreed at workflow level. Posting applies source deduction and destination addition directly; no in-transit state initially.

## D-007 — Company deactivation
**Status:** Implemented in initial Company service. Deactivation sets IsActive=false rather than physically deleting the company.

## D-008 — DTO and service layers
**Status:** Implemented for Company management. Controllers use DTOs and services rather than exposing EF entities directly.


## D-009 — Identity and JWT authentication
**Status:** Implemented in the API. ASP.NET Core Identity uses Guid keys and EF Core storage. JWT bearer authentication validates issuer, audience, signing key, and lifetime. Platform Admin and company-user tokens have distinct claims and authorization boundaries. JWT and bootstrap credentials are configuration secrets and must not be committed.

## D-010 — Temporary password onboarding
**Status:** Implemented and manually verified. Company provisioning generates a 24-character temporary password and sets `MustChangePassword=true`. First login returns a five-minute password-change-only JWT. The token is rejected on other endpoints. Changing the password clears the flag and returns a normal JWT.

## D-011 — One company membership per user
**Status:** Implemented in the current schema. A unique index on `CompanyMembership.UserId` limits a user to one company membership. Warehouse-scoped roles require an assigned warehouse; CompanyModerator must have no assigned warehouse.

## D-012 — Permission evaluation precedence
**Status:** Permission catalog and evaluation service implemented; default role grants and endpoint integration pending. A user-specific allow/deny override takes precedence over the company's role grant; if neither exists, access is denied.
