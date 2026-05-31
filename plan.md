# Sub2API Feature Plan

## Context

Current backup support is focused on PostgreSQL full backups created by Sub2API itself:

- Admin UI can configure S3-compatible storage, create backups, schedule backups, download backup files, delete records, and restore completed records.
- Backup records are stored in application settings under `backup_records`.
- Backup files are `pg_dump` SQL dumps compressed as `.sql.gz` and stored in S3-compatible object storage.
- There is no UI or API for importing an existing backup file into the backup list.

## Goal

Add backup import capabilities so administrators can restore from backup artifacts that were not created in the current running instance, such as manually uploaded `.sql.gz` files, files copied from another S3 bucket/prefix, or backups downloaded before a migration.

## Phase 1: Import Existing S3 Backup

### User Story

As an administrator, I can register an existing S3 object as a backup record, then use the existing restore flow to restore it.

### Scope

- Add an admin API endpoint to import an existing object:
  - `POST /api/v1/admin/backups/import`
  - Request fields:
    - `s3_key`: required S3 object key.
    - `file_name`: optional display name. If empty, derive from `s3_key`.
    - `size_bytes`: optional. If empty, fetch object metadata if possible.
    - `expires_at`: optional.
  - Validate that S3 config exists.
  - Validate that the object exists with `HeadObject`.
  - Create a `BackupRecord` with `status=completed`, `backup_type=postgres`, `triggered_by=imported`.

### Backend Changes

- Extend `BackupObjectStore` with a metadata/head method for objects.
- Implement the method in `S3BackupStore`.
- Add `ImportBackupRecord` to `BackupService`.
- Add handler method in `backend/internal/handler/admin/backup_handler.go`.
- Register route in `backend/internal/server/routes/admin.go`.
- Keep restore password confirmation unchanged.

### Frontend Changes

- Add an "Import backup" button in `frontend/src/views/admin/BackupView.vue`.
- Add a modal with fields for S3 object key, optional file name, optional expiry.
- Add API client method in `frontend/src/api/admin/backup.ts`.
- Add i18n strings for Chinese and English.
- Refresh backup list after successful import.

### Tests

- Service test: import fails when S3 config is missing.
- Service test: import fails when S3 object does not exist.
- Service test: import creates a completed record and preserves S3 key.
- Handler/API test: password is still required only for restore, not import.
- Frontend test: import modal submits request and refreshes list.

## Phase 2: Upload Local Backup File

### User Story

As an administrator, I can upload a local `.sql.gz` backup file from the browser and then restore from it.

### Scope

- Add file upload endpoint:
  - `POST /api/v1/admin/backups/upload`
  - Accept multipart file.
  - Upload the file to configured S3 storage.
  - Create a completed backup record.
- Enforce file extension and size limit.
- Keep restore flow unchanged.

### Notes

- This should not load large files fully into memory.
- Current S3 upload implementation reads the whole object into memory, so streaming or multipart upload should be implemented before enabling large browser uploads.

## Phase 3: Safer Restore Workflow

### Improvements

- Show a stronger restore confirmation dialog with target database name and backup file name.
- Add optional "create pre-restore backup" checkbox.
- Add restore progress states in the record.
- Add restore audit details: operator ID, started time, finished time, error message.
- Disable concurrent backup and restore combinations when they conflict.

## Phase 4: Backup Coverage Expansion

### Candidate Features

- Redis backup support for runtime/cache data that should survive migration.
- Application data directory backup for logs, generated files, and local resources.
- Full instance export bundle that includes PostgreSQL, Redis, and selected `/app/data` files.
- Cross-instance migration helper: export on old instance, import on new instance.

## Open Questions

- Should imported records use `triggered_by=imported`, or should `triggered_by` remain limited to `manual` and `scheduled`?
- Should import allow arbitrary S3 keys, or restrict to the configured prefix?
- Should local file upload be available only in self-hosted mode?
- Should restore automatically create a pre-restore backup by default?
- Is Redis state considered disposable, or does it contain data that must be covered by backups?

## Implementation Order

1. Implement S3 object import because it reuses the existing restore path and has the smallest risk.
2. Improve S3 upload internals to support streaming or multipart uploads.
3. Add local file upload after upload memory usage is safe.
4. Add safer restore UX and audit trail.
5. Evaluate Redis and full-instance backup after database import is stable.
