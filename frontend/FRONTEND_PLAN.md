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


## Backend authentication contract now available

- `POST /api/auth/login`: accepts email and password. Response includes `accessToken`, `tokenType`, `expiresAt`, and `requiresPasswordChange`.
- If `requiresPasswordChange=true`, treat the returned token as a restricted challenge token. Do not call normal company APIs with it. Route the user to a forced password-change screen.
- `POST /api/auth/change-temporary-password`: requires the challenge Bearer token and a body containing `currentPassword` and `newPassword`. A successful response contains a normal JWT and `requiresPasswordChange=false`.
- Normal company JWTs and Platform Admin JWTs are separate authorization contexts. Platform Admin endpoints require the `PlatformAdmin` role.
- The backend currently returns a generated temporary password once in the platform company-provisioning response. A secure invitation/email delivery flow is not implemented; the platform UI must display the credential once and handle it as a secret.

The frontend must not treat hiding UI elements as authorization. Inventory API endpoints still need consistent backend authentication, permission enforcement, and trusted tenant resolution before frontend integration is considered secure.
