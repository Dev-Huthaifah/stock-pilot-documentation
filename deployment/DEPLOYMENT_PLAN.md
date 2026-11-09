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
