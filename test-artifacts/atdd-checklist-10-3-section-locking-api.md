# ATDD Checklist: 10-3-section-locking-api

## Story Info
- **Epic:** 10
- **Story:** 10.3 Section Locking API
- **Module:** Client API
- **Story Key:** 10-3-section-locking-api

## Acceptance Tests

### API Contention (test_proposal_section_locks_contention.py)
- [ ] Concurrent acquire: Exactly one request wins (200), the other loses (423) with lock metadata.

### API Acquire & Release (test_proposal_section_locks.py)
- [ ] Happy path acquire: bid_manager acquires lock on freshly-created section -> 200 + SectionLockAcquireResponse
- [ ] Refresh own lock: same caller re-acquires -> 200, acquired_at unchanged, expires_at pushed forward
- [ ] Foreign live lock: Alice holds, Bob attempts -> 423 with SectionLockConflictDetail
- [ ] Foreign expired lock: Alice holds expired lock, Bob attempts -> 200 (replaces transparently)
- [ ] read_only role acquire: read_only collaborator -> 403
- [ ] Non-collaborator acquire: authenticated company member without collaborator row -> 403
- [ ] Cross-company acquire: company-B caller -> 404
- [ ] Admin bypass acquire: company admin acquires -> 200
- [ ] Self-release: caller releases own lock -> 204 + row deleted
- [ ] Force-release by bid_manager: bid_manager B releases technical_writer A's lock -> 204
- [ ] Force-release by admin: admin releases any user's lock -> 204
- [ ] Non-bid_manager non-holder release: financial_analyst attempts to release legal_reviewer's lock -> 403
- [ ] Idempotent release on missing row: DELETE /lock when no row exists -> 204
- [ ] Invalid section_key: empty string or >255 chars -> 422
- [ ] Audit trail on acquire: assert one audit_log row per acquire (action_type=create), on refresh assert action_type=update
- [ ] Audit trail on release: assert action_type=delete

### API GET Sections List (test_proposal_sections_list.py)
- [ ] Empty proposal: no current_version -> 200 with sections=[]
- [ ] No locks: proposal with 3 sections, no active locks -> 200 with 3 SectionWithLock rows, lock=None
- [ ] One active lock: lock on section B -> 200, sections A and C have lock=None, section B has lock info
- [ ] Expired lock excluded: lock with expires_at in past -> NOT returned
- [ ] read_only role can read: read_only collaborator -> 200
- [ ] Non-collaborator -> 403
- [ ] Cross-company -> 404
- [ ] Malformed content.sections missing: content JSONB without "sections" -> 200 with empty sections + WARNING log

### Service Unit Tests (test_proposal_section_lock_service.py)
- [ ] Atomic UPSERT: fresh acquire inserts row
- [ ] Atomic UPSERT: own-lock refresh preserves acquired_at, pushes expires_at
- [ ] Atomic UPSERT: foreign expired replaces, new acquired_at = NOW()
- [ ] Atomic UPSERT: foreign live -> ForeignLockHeld sentinel returned (or None)
- [ ] force_release: deletes by id
- [ ] get_active_lock: returns None for non-existent, None for expired, row for live
- [ ] list_active_locks_for_proposal: excludes expired rows; joins full_name correctly

### Integration (test_cleanup_expired_section_locks.py)
- [ ] Cleanup task runs: 3 expired locks deleted, 2 live ones remain; structlog metrics captured
- [ ] Cleanup task idempotent: runs a second time, 0 deleted, 2 remain
- [ ] DB error path: service raises exception, task catches and logs ERROR without re-raising

### Route Dependency / RBAC Matrix (test_section_locks_rbac_matrix.py & test_proposals_router_dependency_spec.py)
- [ ] RBAC Matrix: 24 assertions covering 3 endpoints × 8 caller profiles
- [ ] Route dependency snapshot: captures 3 new `/api/v1/proposals/{proposal_id}/sections*` routes
