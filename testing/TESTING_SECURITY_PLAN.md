# Testing and Security Plan

**Status:** A successful backend build has been reported; a complete automated test suite and security review are not confirmed.

## Testing
- Unit tests for service rules, validation, stock calculations, transfer rules, and permission evaluation.
- Integration tests for endpoints, EF persistence, migrations, transactions, and concurrency.
- Multi-tenant tests: Company A cannot read, update, deactivate, or reference Company B's resources; client-supplied CompanyId cannot change tenant identity.
- Inventory invariants: no unintended negative stock, no partial transfer posting, no duplicate posting, valid source/destination, and consistent balances/movements.

## Production security requirements
Authentication and authorization; separate Platform Admin and company roles; trusted server-side tenant resolution; input validation; safe error responses; rate limiting as appropriate; secret management; HTTPS; audit important administrative/inventory operations; backup and restore testing; dependency review.

A feature is not done merely because it compiles. It needs relevant tests, tenant-boundary checks, error handling, and documentation.


## Implemented authentication behavior and manual verification

The following behavior has been implemented and manually verified through PowerShell against the local API:
- Platform Admin login returns a normal Bearer JWT with `RequiresPasswordChange=false`.
- Company Admin login with a generated temporary password returns `RequiresPasswordChange=true` and a five-minute challenge token.
- The challenge token receives `401 Unauthorized` when used against `GET /api/platform/companies`.
- After successful temporary-password change, the endpoint returns a normal JWT with `RequiresPasswordChange=false`.
- A fresh login with the new password succeeds and returns `RequiresPasswordChange=false`.
- A company-user JWT receives `403 Forbidden` from the Platform Admin-only companies endpoint.
- The user reported successful `dotnet build StockPilot.slnx` and a successful test run earlier in the authentication workstream. The permission-service suite previously reported 17 passing tests; re-run the current suite before relying on that count.

## Remaining security gaps (do not treat as production-ready)

- Warehouse, product, stock, and stock-transfer controllers do not yet consistently require authentication or call `IPermissionService`.
- The permission catalog is seeded, but company-role default grants are not seeded.
- Route-supplied CompanyId remains a scoping input; trusted tenant context must be derived from authenticated membership.
- The legacy `POST /api/platform/companies` action still creates a company through `ICompanyService` without provisioning its initial CompanyModerator. Prefer the company-provisioning route and remove or disable the legacy create action before production.
- Platform Admin accounts currently bypass the company-user temporary-password branch in `AuthService.LoginAsync`; define a separate platform-admin password rotation policy if required.
- Challenge tokens are short-lived and endpoint-restricted; automated integration tests should cover expiry, replay after successful password change, wrong current password, inactive user/company/membership/warehouse, and token use on all protected route groups.
- No comprehensive API authorization, tenant-isolation, or production security review has been completed.
