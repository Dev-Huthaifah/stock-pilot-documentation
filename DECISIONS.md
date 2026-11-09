# Decision Log

## D-001 — Multi-tenant workspaces
**Status:** Agreed. Each customer company has its own logically isolated, company-named Stock Pilot workspace.

## D-002 — Platform Admin and Main Moderator
**Status:** Product decision agreed; implementation pending. Platform Admin creates companies and invites the Main Moderator. Main Moderator manages company users and permissions.

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
