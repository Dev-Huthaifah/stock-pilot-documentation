# Deployment Plan

**Status: Not started. Production hosting has not been selected.**

Decisions required: cloud/hosting provider, API hosting, production SQL Server, frontend hosting, domains/TLS, secret management, logging/monitoring, backup/recovery, CI/CD, and release approvals.

Plan separate local development, test/staging, and production environments. Do not use LocalDB in production. Production secrets must be stored securely outside source control.

Release checklist:
- Build and automated tests pass.
- Migrations reviewed and safely applied.
- Authentication and authorization enabled.
- Tenant-isolation tests pass.
- HTTPS configured.
- Secrets absent from repository and logs.
- Backups and restore procedure verified.
- Monitoring and rollback procedure documented.


## Current authentication state

Local API authentication is implemented with ASP.NET Core Identity and JWT Bearer, including Platform Admin policy protection and a temporary-password onboarding flow for company moderators. This is not yet a production security sign-off: inventory controllers still need consistent authentication and permission enforcement, trusted tenant resolution is pending, and secret rotation/production secret storage and automated security tests remain to be designed and verified.
