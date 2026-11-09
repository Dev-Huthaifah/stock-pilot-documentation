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
