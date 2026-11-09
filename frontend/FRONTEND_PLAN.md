# Frontend Plan

**Status: Not started; framework not selected.**

Expected product surfaces:
- Platform Admin: manage companies and invite Main Moderators.
- Company workspace: company-branded experience.
- Main Moderator: manage company users, roles, and permissions.
- Operations: products, warehouses, stock balances, movements, transfers.
- Dashboard/reporting.
- Later AI assistant interface.

UX requirements: clear active-company identity, permission-aware navigation, responsive desktop/mobile layouts, accessible forms, and clear loading/empty/error states.

Coordinate API routes, DTOs, authentication/session handling, tenant context, and authorization behavior with the backend. Client-side role checks are not a security boundary; the backend must enforce permissions.
