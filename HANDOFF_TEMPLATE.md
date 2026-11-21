# Stock Pilot — New Chat Handoff

Copy this into a new conversation and update the placeholders.

I am continuing development of **Stock Pilot**, a multi-tenant inventory and product management platform.

## Workstream
`[Backend / Frontend / AI / Testing & Security / Deployment / Integration]`

## Repositories
- Application: https://github.com/Dev-Huthaifah/stock-pilot
- Documentation: https://github.com/Dev-Huthaifah/stock-pilot-documentation

## Read first
`PROJECT_CONTEXT.md`, `ARCHITECTURE.md`, `TECH_STACK.md`, `DECISIONS.md`, and the relevant workstream progress file.

## Current Git state
- Branch: `[branch]`
- Latest application commit: `edf0841` — temporary password flow and company admin provisioning (verify current remote head before continuing).
- Pushed: `[yes/no]`
- Working tree clean: `[yes/no]`

## Current implementation
Implemented: inventory/company/warehouse/product/stock/transfer services; ASP.NET Core Identity; JWT login for Platform Admin and company users; PlatformAdminOnly policy; company provisioning with generated temporary password; forced password-change challenge token (5-minute lifetime); password change and normal JWT issuance; CompanyMembership and permission catalog/evaluation service. Manual auth checks confirmed challenge token rejection (401) on platform companies, company JWT rejection (403) from Platform Admin-only routes, and successful password change/re-login. Permission catalog is seeded, but default company-role grants are not. Tenant-owned inventory controllers are not yet consistently protected by authorization.

## Known issues
Remaining: apply authorization and permission checks to warehouse/product/stock/transfer endpoints; trusted tenant resolution instead of trusting route CompanyId; seed default CompanyRolePermission grants; remove/disable legacy POST /api/platform/companies that can create a company without its initial moderator; add comprehensive auth/tenant integration tests; decide platform-admin password rotation and secure temporary-credential delivery.

## Immediate next task
Implement idempotent default company-role permission grants and enforce IPermissionService on every tenant-owned endpoint, deriving company and warehouse scope from authenticated membership.

## Working preferences
- Respond in Arabic unless I request another language.
- Work in coherent implementation batches, not one file per message.
- Inspect existing code before proposing changes.
- Do not invent file contents, test results, commits, or deployment status.
- Preserve agreed architecture unless a reasoned change is discussed.
- Keep tenant isolation and security central.
- Ask for build/test output before claiming success.
- Update documentation after meaningful milestones.
