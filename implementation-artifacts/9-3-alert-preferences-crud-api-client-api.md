# Story 9.3: Alert Preferences CRUD API (Client API)

Status: ready-for-dev

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **user**,
I want **to manage my alert preferences via an API**,
so that I can configure my CPV sectors, regions, budget, and alert frequency (immediate/daily/weekly) to receive relevant opportunity notifications.

## Acceptance Criteria

1. **CRUD Endpoints exist** — Implement endpoints under `/api/v1/alerts/preferences`: `POST` (create), `GET` (list user's preferences), `PUT /{id}` (update), `DELETE /{id}` (delete), `PATCH /{id}/toggle` (enable/disable).
2. **CPV Validation** — `cpv_sectors` (text[]) validated against known CPV codes. Invalid codes return 422.
3. **Region Validation** — `regions` (text[]) validated against EU member states. Invalid regions return 422.
4. **Budget Range** — `budget_min` and `budget_max` (decimal) enforced: `min >= 0`, `max > min`.
5. **Deadline Proximity** — `deadline_days_ahead` (int, 1-90).
6. **Limit Enforcement** — Maximum 5 alert preferences per user. A 6th create attempt returns 409.
7. **Toggle Endpoint** — `PATCH /{id}/toggle` flips `is_active` boolean without requiring full payload.
8. **Auth & Scoping** — Endpoints require authentication. Users can only access their own preferences. Accessing another user's preference returns 404 (not 403, to avoid existence leakage).
9. **Database** — Alembic migration for `client.alert_preferences` with composite index on `(user_id, is_active)`.

## Tasks / Subtasks

- [ ] Task 1: Create Alembic migration for `client.alert_preferences`
  - [ ] 1.1 Create table with `id` (UUID PK), `user_id` (UUID FK), `cpv_sectors` (ARRAY(String)), `regions` (ARRAY(String)), `budget_min` (Numeric), `budget_max` (Numeric), `deadline_days_ahead` (Integer), `schedule` (Enum: immediate/daily/weekly), `is_active` (Boolean).
  - [ ] 1.2 Add composite index on `(user_id, is_active)` for efficient active-preference lookup by the Notification Service.
- [ ] Task 2: Define Pydantic Schemas in `schemas/alert_preferences.py`
  - [ ] 2.1 Request/Response models with validators for CPV codes, EU Regions, Budget ranges, and Deadline days.
- [ ] Task 3: Implement CRUD Endpoints in `api/v1/alert_preferences.py`
  - [ ] 3.1 `POST` to create (check limit 5 -> 409).
  - [ ] 3.2 `GET` to list.
  - [ ] 3.3 `PUT /{id}` to update (404 if not found/owned).
  - [ ] 3.4 `DELETE /{id}` to remove (404 if not found/owned).
  - [ ] 3.5 `PATCH /{id}/toggle` to update `is_active` (404 if not found/owned).
- [ ] Task 4: ATDD/API Tests
  - [ ] 4.1 Write P0 API tests verifying validation rules (CPV, Region, Limits) and 404 scope isolation.

## Dev Notes

- Reference `test-design-epic-09.md` for P0 test requirements: "Validation rules (CPV, Region, Limits)".
- Store `client.alert_preferences` in the client schema to be read by the Notification Service later.
- Return 404 for preferences owned by other users (not 403, to avoid leaking existence).
