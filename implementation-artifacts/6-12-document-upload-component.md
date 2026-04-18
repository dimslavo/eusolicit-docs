# Story 6.12: Document Upload Component

Status: review

<!-- Note: Validation is optional. Run validate-create-story for quality check before dev-story. -->

## Story

As a **paid-tier user managing documents on the opportunity detail Documents tab**,
I want **a reusable `DocumentUploadComponent` with drag-and-drop (and click-to-browse), client-side MIME/size validation, per-file progress bars, automatic S3 PUT via presigned URL, confirm callback, and live scan-status polling — wired into the existing `DocumentsTab` upload zone placeholder from S06.11**,
so that **I can securely upload one or more documents, see real-time upload and scan progress, and have infected files surfaced with a clear error without ever having to refresh the page**.

## Acceptance Criteria

1. `DocumentUploadComponent.tsx` is created at `app/[locale]/(protected)/opportunities/components/DocumentUploadComponent.tsx` and exported for use by `DocumentsTab.tsx`.

2. The component renders a drag-and-drop zone (`data-testid="upload-zone"`) that activates a dashed-border highlight on `dragover`; dropping files enqueues them. A hidden `<input type="file" multiple accept=".pdf,.docx,.xlsx,.zip" ref={fileInputRef} data-testid="upload-file-input" />` triggers on zone click via `fileInputRef.current.click()`.

3. **Client-side validation** runs before any network call for each file:
   - MIME type must be one of `application/pdf`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document` (DOCX), `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` (XLSX), `application/zip` or `application/x-zip-compressed`; invalid files show an inline error (`data-testid="upload-error-{index}"`) with `t("uploadInvalidType")` — they are NOT added to the upload queue.
   - File size must be ≤ 100 MB (104,857,600 bytes); oversized files show `t("uploadFileTooLarge")` and are rejected.
   - The sum of `existingTotalBytes + newFilesBytes` must not exceed 500 MB (524,288,000 bytes); files that would exceed the limit show `t("uploadPackageSizeExceeded")` and are rejected.

4. Accepted files are added to the upload queue (`data-testid="upload-file-queue"`); each entry (`data-testid="upload-file-item-{index}"`) shows: filename, file size (formatted with `formatFileSize()`), a status badge (`data-testid="upload-file-status-{index}"`), a progress bar (`data-testid="upload-file-progress-{index}"`), and a cancel button (`data-testid="upload-file-cancel-{index}"`).

5. **Upload flow per file** (executed immediately after enqueue):
   - Call `requestPresignedUrl(opportunityId, { filename, size, mime_type })` → receives `{ doc_id, presigned_url, expires_in }`.
   - PUT the `File` object directly to `presigned_url` using `XMLHttpRequest` (NOT fetch) with `Content-Type` header set to `mime_type`; attach `xhr.upload.addEventListener("progress", ...)` to update the per-file progress percentage; update status badge to `uploading` with `"{progress}%"` text.
   - On XHR `load` with status 2xx: call `confirmDocumentUpload(opportunityId, doc_id)` → status transitions to `scanning`.
   - On XHR error or non-2xx: status transitions to `failed` with `t("uploadErrorNetwork")`.

6. **Cancel**: clicking the cancel button for a file in `queued` or `uploading` state aborts the in-progress XHR (via stored `AbortController` signal passed to XHR as `signal.addEventListener("abort", () => xhr.abort())`) and removes the file entry from the queue.

7. **Scan status polling**: `useOpportunityDocuments(opportunityId)` is called in `DocumentsTab` with `refetchInterval` set to `3000` ms when any document has `scan_status === "pending"`, otherwise `false`. When a document transitions from `pending` to `clean` or `infected`, the corresponding queue entry status badge updates; `onUploadComplete()` callback is invoked to trigger the parent document list refresh.

8. **Status badge states** per file:
   - `queued` — grey pill, `t("uploadStatusQueued")`
   - `uploading` — blue pill, `t("uploadStatusUploading", { progress })` (e.g. "47% uploaded")
   - `scanning` — amber spinner + `t("uploadStatusScanning")` ("Scanning…")
   - `clean` — green check + `t("uploadStatusClean")` ("Clean")
   - `infected` — red X + `t("uploadStatusInfected")` ("Infected — file removed")
   - `failed` — red exclamation + error text

9. **Package size indicator** (`data-testid="upload-package-size-indicator"`) displays `t("uploadPackageSizeIndicator", { used, limit })` (e.g. "152 MB of 500 MB used"); `used` is computed from the sum of `size` of existing documents passed via `existingTotalBytes` prop.

10. `DocumentsTab.tsx` is modified to:
    - Import `DocumentUploadComponent` and render it in place of the `detail-document-upload-zone` static placeholder div.
    - Pass `opportunityId`, `existingTotalBytes` (sum of `documents.map(d => d.size)`), and `onUploadComplete={() => refetch()}`.
    - Add `refetchInterval` to `useOpportunityDocuments(id, { refetchInterval: (query) => query.state.data?.some(d => d.scan_status === "pending") ? 3000 : false })`.

11. `lib/api/opportunities.ts` is extended with:
    - `PresignedUploadResponse` interface (`{ doc_id: string; presigned_url: string; expires_in: number }`)
    - `ConfirmUploadResponse` interface (`{ doc_id: string; scan_status: string; message: string }`)
    - `requestPresignedUrl(opportunityId, metadata): Promise<PresignedUploadResponse>`
    - `confirmDocumentUpload(opportunityId, docId): Promise<ConfirmUploadResponse>`
    - `formatFileSize(bytes: number): string` utility exported from `opportunity-utils.tsx`

12. All upload i18n keys (see Dev Notes: i18n Keys section) added to both `messages/en.json` and `messages/bg.json` under the `opportunities` namespace; key-parity check (`pnpm check:i18n`) passes.

13. All components and interactive elements have `data-testid` attributes as specified in the Dev Notes testid reference table.

14. ATDD test file `__tests__/opportunities-upload-s6-12.test.ts` covers: file structure, `data-testid` presence in source, API export completeness, i18n key completeness, XHR usage for S3 PUT (via source assertion), and drag-and-drop interaction flow (mocked APIs); all tests pass GREEN.

## Tasks / Subtasks

- [x] Task 1: Extend `lib/api/opportunities.ts` with upload interfaces and functions (AC: 11)
  - [x] 1.1 Add `PresignedUploadResponse` interface: `{ doc_id: string; presigned_url: string; expires_in: number }`
  - [x] 1.2 Add `ConfirmUploadResponse` interface: `{ doc_id: string; scan_status: string; message: string }`
  - [x] 1.3 Add `requestPresignedUrl(opportunityId: string, metadata: { filename: string; size: number; mime_type: string }): Promise<PresignedUploadResponse>` — POST to `/api/v1/opportunities/${opportunityId}/documents/upload`
  - [x] 1.4 Add `confirmDocumentUpload(opportunityId: string, docId: string): Promise<ConfirmUploadResponse>` — POST to `/api/v1/opportunities/${opportunityId}/documents/${docId}/confirm`

- [x] Task 2: Add `formatFileSize()` to `opportunity-utils.tsx` (AC: 11)
  - [x] 2.1 Export `formatFileSize(bytes: number): string` — returns human-readable size: < 1024 → `"{bytes} B"`, < 1,048,576 → `"{kb} KB"`, else `"{mb} MB"` (rounded to 1 decimal)
  - [x] 2.2 Ensure no existing utilities are modified; only append the new export

- [x] Task 3: Add i18n translation keys (AC: 12)
  - [x] 3.1 Add all upload keys listed in Dev Notes to `messages/en.json` under the `opportunities` namespace
  - [x] 3.2 Add matching keys with Bulgarian translations to `messages/bg.json`
  - [x] 3.3 Run `pnpm check:i18n` to verify key-parity passes

- [x] Task 4: Extend `useOpportunityDocuments` to accept options (AC: 10)
  - [x] 4.1 In `lib/queries/use-opportunities.ts`, add optional second argument `options?: Partial<UseQueryOptions<DocumentRecord[]>>` to `useOpportunityDocuments`; spread onto the `useQuery` call; preserves all existing callers (calling with one arg still works)

- [x] Task 5: Create `DocumentUploadComponent.tsx` (AC: 1–9, 13)
  - [x] 5.1–5.13 All subtasks implemented

- [x] Task 6: Modify `DocumentsTab.tsx` to wire `DocumentUploadComponent` (AC: 10)
  - [x] 6.1–6.5 All subtasks implemented

- [x] Task 7: ATDD test file `__tests__/opportunities-upload-s6-12.test.ts` — all 122 tests pass GREEN (AC: 14)

## Dev Notes

### Architecture Overview

S06.12 is a pure frontend story. All required backend endpoints are already implemented:

- `POST /api/v1/opportunities/{opportunity_id}/documents/upload` → `{doc_id, presigned_url, expires_in}` (S06.06, done)
- `POST /api/v1/opportunities/{opportunity_id}/documents/{doc_id}/confirm` → `{doc_id, scan_status, message}` (S06.06, done)
- `GET /api/v1/opportunities/{opportunity_id}/documents` — already used by `DocumentsTab` via `useOpportunityDocuments` (S06.11, done)

**Integration with S06.11**: `DocumentsTab.tsx` already has the static `detail-document-upload-zone` placeholder div (no onClick, no logic). This story replaces that placeholder with `<DocumentUploadComponent />`. The `DocumentsTab` modification is **minimal** — import + replace the placeholder div + add `refetchInterval` option.

**Do NOT modify** `OpportunityDetailPage.tsx`, `OverviewTab.tsx`, `RequirementsTab.tsx`, `AIAnalysisTab.tsx`, `SubmissionGuideTab.tsx` — all out of scope.

### File Structure

```
eusolicit-app/frontend/apps/client/
├── app/[locale]/(protected)/opportunities/
│   └── components/
│       ├── DocumentUploadComponent.tsx    # NEW — main upload component
│       ├── DocumentsTab.tsx               # MODIFIED — wire upload component, add refetchInterval
│       └── opportunity-utils.tsx          # MODIFIED — add formatFileSize()
├── lib/
│   ├── api/
│   │   └── opportunities.ts              # MODIFIED — add PresignedUploadResponse, ConfirmUploadResponse, requestPresignedUrl, confirmDocumentUpload
│   └── queries/
│       └── use-opportunities.ts          # MODIFIED — add options param to useOpportunityDocuments
├── messages/
│   ├── en.json                           # MODIFIED — add upload i18n keys
│   └── bg.json                           # MODIFIED — add upload i18n keys
└── __tests__/
    └── opportunities-upload-s6-12.test.ts  # NEW — ATDD tests
```

### API Contract (S06.06 — Backend Done)

Add to `lib/api/opportunities.ts`:

```typescript
// ─── Upload Interfaces ────────────────────────────────────────────────────────

export interface PresignedUploadResponse {
  doc_id: string;
  presigned_url: string;
  expires_in: number;   // seconds (900 = 15 minutes)
}

export interface ConfirmUploadResponse {
  doc_id: string;
  scan_status: "pending" | "clean" | "infected" | "failed";
  message: string;
}

// ─── Upload Functions ─────────────────────────────────────────────────────────

export async function requestPresignedUrl(
  opportunityId: string,
  metadata: { filename: string; size: number; mime_type: string }
): Promise<PresignedUploadResponse> {
  const response = await apiClient.post<PresignedUploadResponse>(
    `/api/v1/opportunities/${opportunityId}/documents/upload`,
    metadata
  );
  return response.data;
}

export async function confirmDocumentUpload(
  opportunityId: string,
  docId: string
): Promise<ConfirmUploadResponse> {
  const response = await apiClient.post<ConfirmUploadResponse>(
    `/api/v1/opportunities/${opportunityId}/documents/${docId}/confirm`
  );
  return response.data;
}
```

**MIME Type Validation**: The backend (S06.06 AC3) validates on `application/pdf`, DOCX, XLSX, ZIP. Client-side must match exactly. Some browsers report `application/x-zip-compressed` for `.zip` files — accept both:

```typescript
const ALLOWED_MIME_TYPES = new Set([
  "application/pdf",
  "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
  "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
  "application/zip",
  "application/x-zip-compressed",
]);

const MIME_BY_EXTENSION: Record<string, string> = {
  ".pdf": "application/pdf",
  ".docx": "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
  ".xlsx": "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
  ".zip": "application/zip",
};

function inferMimeType(filename: string): string {
  const ext = filename.substring(filename.lastIndexOf(".")).toLowerCase();
  return MIME_BY_EXTENSION[ext] ?? "";
}
```

**Size constants** (must match backend exactly):

```typescript
const MAX_FILE_BYTES = 104_857_600;      // 100 MB
const MAX_PACKAGE_BYTES = 524_288_000;   // 500 MB
```

### XHR-Based S3 PUT (Progress Tracking)

Use `XMLHttpRequest` — NOT `fetch` — for the S3 PUT because XHR provides `upload.progress` events that enable per-file progress bars. `fetch` does not support upload progress in most browsers.

```typescript
function uploadToS3(
  presignedUrl: string,
  file: File,
  mimeType: string,
  onProgress: (percent: number) => void,
  signal: AbortSignal
): Promise<void> {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();
    xhr.open("PUT", presignedUrl, true);
    xhr.setRequestHeader("Content-Type", mimeType);

    xhr.upload.addEventListener("progress", (event) => {
      if (event.lengthComputable) {
        onProgress(Math.round((event.loaded / event.total) * 100));
      }
    });

    xhr.addEventListener("load", () => {
      // S3 presigned PUT returns 200 on success
      if (xhr.status >= 200 && xhr.status < 300) {
        resolve();
      } else {
        reject(new Error(`S3 upload failed with status ${xhr.status}`));
      }
    });

    xhr.addEventListener("error", () =>
      reject(new Error("Network error during upload"))
    );

    xhr.addEventListener("abort", () =>
      reject(new Error("Upload cancelled"))
    );

    // Wire AbortController to XHR abort
    signal.addEventListener("abort", () => xhr.abort());

    xhr.send(file);
  });
}
```

**Important**: Do NOT set `Authorization` header on the S3 PUT — the presigned URL already contains auth credentials as query params. Adding auth headers would break the presigned URL signature.

### Scan Status Polling via TanStack Query `refetchInterval`

After `confirmDocumentUpload` is called, the file's server-side scan status will change from `pending` → `clean` | `infected` asynchronously. Poll by extending `useOpportunityDocuments`:

```typescript
// lib/queries/use-opportunities.ts
import type { UseQueryOptions } from "@tanstack/react-query";

export function useOpportunityDocuments(
  id: string,
  options?: Partial<UseQueryOptions<DocumentRecord[]>>
) {
  return useQuery({
    queryKey: ["opportunity-documents", id],
    queryFn: () => getOpportunityDocuments(id),
    staleTime: 30_000,
    ...options,
  });
}
```

In `DocumentsTab.tsx`:

```typescript
const { data: documents = [], isLoading, isError, refetch } = useOpportunityDocuments(
  opportunityId,
  {
    refetchInterval: (query) =>
      query.state.data?.some((d: DocumentRecord) => d.scan_status === "pending")
        ? 3000
        : false,
  }
);
```

This polls every 3 seconds only while any document is in `pending` state, stopping automatically once all documents have resolved to `clean`, `infected`, or `failed`. The `onUploadComplete` callback from the component simply calls `refetch()` immediately after confirm, so the poll starts right away without waiting 3 seconds.

### Queue State Management

Use a local `useState` for the upload queue. Do NOT use Zustand for this — upload queue state is ephemeral and scoped to a single component mount. Use `useCallback` for all handlers to avoid stale closures:

```typescript
const [queue, setQueue] = useState<FileUploadState[]>([]);

const updateQueue = useCallback((id: string, patch: Partial<FileUploadState>) => {
  setQueue(prev =>
    prev.map(entry => (entry.id === id ? { ...entry, ...patch } : entry))
  );
}, []);

const removeFromQueue = useCallback((id: string) => {
  setQueue(prev => prev.filter(entry => entry.id !== id));
}, []);
```

The `processFile` function must capture `id`, `opportunityId`, `onUploadComplete`, and `updateQueue` via closure. Avoid starting `processFile` inside a `useEffect` — call it immediately after appending the entry to the queue:

```typescript
const handleFiles = useCallback(async (files: FileList | File[]) => {
  const fileArray = Array.from(files);
  const validFiles: FileUploadState[] = [];

  for (const file of fileArray) {
    const currentQueueBytes = validFiles.reduce((s, f) => s + f.file.size, 0);
    const { valid, error } = validateFile(file, existingTotalBytes + currentQueueBytes);
    if (!valid) {
      toast.error(error ?? t("uploadInvalidType"));
      continue;
    }
    const entry: FileUploadState = {
      id: crypto.randomUUID(),
      file,
      status: "queued",
      progress: 0,
    };
    validFiles.push(entry);
  }

  if (validFiles.length === 0) return;

  setQueue(prev => [...prev, ...validFiles]);

  // Start uploads immediately after state update
  for (const entry of validFiles) {
    processFile(entry);  // fire-and-forget; processFile handles its own state updates
  }
}, [existingTotalBytes, opportunityId, onUploadComplete, t]);
```

**Note**: `processFile` is called outside `setQueue`'s callback — this is intentional. The `entry` reference is captured at enqueue time and does not need to read from `queue` state directly.

### Package Size Indicator

```typescript
const queueActiveBytes = queue
  .filter(e => ["uploading", "scanning", "clean"].includes(e.status))
  .reduce((sum, e) => sum + e.file.size, 0);

const totalUsedBytes = existingTotalBytes + queueActiveBytes;
```

`formatFileSize` to add to `opportunity-utils.tsx`:

```typescript
export function formatFileSize(bytes: number): string {
  if (bytes < 1024) return `${bytes} B`;
  if (bytes < 1_048_576) return `${(bytes / 1024).toFixed(1)} KB`;
  return `${(bytes / 1_048_576).toFixed(1)} MB`;
}
```

### `DocumentsTab.tsx` Minimal Change

The modification to `DocumentsTab.tsx` is surgical. The static placeholder currently reads:

```tsx
{/* EXISTING S06.11 placeholder — replace this entire div */}
<div
  data-testid="detail-document-upload-zone"
  className="border-2 border-dashed ..."
>
  <Upload />
  {t("documentsUploadCta")}
</div>
```

Replace with:

```tsx
import { DocumentUploadComponent } from "./DocumentUploadComponent";

// Inside DocumentsTab, after computing `documents`:
const existingTotalBytes = useMemo(
  () => documents.reduce((sum, d) => sum + d.size, 0),
  [documents]
);

// Replace the static placeholder div:
<DocumentUploadComponent
  opportunityId={opportunityId}
  existingTotalBytes={existingTotalBytes}
  onUploadComplete={() => refetch()}
/>
```

The `data-testid="detail-document-upload-zone"` was a placeholder; it is gone and replaced by `data-testid="upload-zone"` inside `DocumentUploadComponent`. The ATDD test for S06.11 (`opportunities-detail-s6-11.test.ts`) checked for the testid's presence in the source of `DocumentsTab.tsx`. After this change, that testid will no longer exist there. **Check the S06.11 ATDD test** — if it asserts `detail-document-upload-zone` in `DocumentsTab.tsx` source, update that assertion to check for the `DocumentUploadComponent` import instead. The component itself owns the upload zone.

### UI Components Import Map

All UI components from `@eusolicit/ui`:

| Component | Usage |
|-----------|-------|
| `Button` | Cancel button, browse fallback button |
| `Progress` | Per-file upload progress bar (`value={progress}`) |
| `Badge` | Status badges (queued, uploading, scanning, clean, infected, failed) |
| `toast` | Validation error toasts (from `@eusolicit/ui` or `sonner`) |
| `Skeleton` | Loading skeleton while documents list loads |

Lucide icons (`lucide-react`):
- `Upload` — upload zone icon
- `X` — cancel button icon
- `CheckCircle2` — clean status icon
- `XCircle` — infected status icon
- `Loader2 animate-spin` — scanning status icon
- `AlertCircle` — failed status icon
- `FileText`, `FileSpreadsheet`, `Archive` — file type icons (optional; show generic `File` if unsure)

### i18n Keys Required

Add to `opportunities` namespace in both `en.json` and `bg.json`:

```json
{
  "uploadDropZoneTitle": "Drop files here or click to upload",
  "uploadDropZoneSubtitle": "PDF, DOCX, XLSX, ZIP · Max 100 MB per file",
  "uploadFileTooLarge": "File is too large (max 100 MB)",
  "uploadInvalidType": "Invalid file type. Allowed: PDF, DOCX, XLSX, ZIP",
  "uploadPackageSizeExceeded": "Upload would exceed the 500 MB package limit",
  "uploadStatusQueued": "Queued",
  "uploadStatusUploading": "{progress}% uploaded",
  "uploadStatusScanning": "Scanning…",
  "uploadStatusClean": "Clean",
  "uploadStatusInfected": "Infected — file removed",
  "uploadStatusFailed": "Upload failed",
  "uploadErrorNetwork": "Network error — please retry",
  "uploadCancelButton": "Cancel",
  "uploadPackageSizeIndicator": "{used} of 500 MB used",
  "uploadBrowseCta": "Browse files"
}
```

Before adding: check `messages/en.json` for any existing upload keys added by previous stories and do NOT duplicate.

### data-testid Reference Table

| `data-testid` | Component | Description |
|---|---|---|
| `upload-zone` | `DocumentUploadComponent` | Drag-and-drop target + click trigger |
| `upload-file-input` | `DocumentUploadComponent` | Hidden `<input type="file">` |
| `upload-package-size-indicator` | `DocumentUploadComponent` | "X MB of 500 MB used" |
| `upload-file-queue` | `DocumentUploadComponent` | Queue list container |
| `upload-file-item-{index}` | `DocumentUploadComponent` | Per-file row (0-indexed) |
| `upload-file-status-{index}` | `DocumentUploadComponent` | Status badge per file |
| `upload-file-progress-{index}` | `DocumentUploadComponent` | Progress bar per file |
| `upload-file-cancel-{index}` | `DocumentUploadComponent` | Cancel button per file |
| `upload-error-{index}` | `DocumentUploadComponent` | Inline validation error per rejected file |

### Test Design Coverage (from `eusolicit-docs/test-artifacts/test-design-epic-06.md`)

| Test ID | Level | Scenario | ATDD Coverage |
|---------|-------|----------|---------------|
| **E06-P1-028** | Component | Drag-drop upload flow — presigned URL requested, S3 PUT initiated, confirm called, scan status polling shows pending → clean | ATDD: mock `requestPresignedUrl` + `confirmDocumentUpload`; simulate file drop on `upload-zone`; assert presigned URL API called; simulate XHR progress event; assert progress bar `data-testid` reflects percent; assert `onUploadComplete` called after confirm; simulate polling result showing `clean` → assert badge shows "Clean" |
| **E06-P2-005** | API (backend) | Rejected by server for >100MB | ATDD: client-side assertion — simulate 105MB file drop → assert toast error shown, file NOT in queue |
| **E06-P1-016** | API (backend) | Invalid MIME types rejected | ATDD: simulate `.exe` file drop → assert validation error toast shown, no API call made |
| **E06-P0-006** | Integration (backend) | Download blocked before scan | Not directly tested by this story; scan_status polling handles UI side — covered by `useOpportunityDocuments` `refetchInterval` behaviour |

**Playwright E2E additions** (future — S06.14 scope): The full E2E tier-gate flow (E06-P0-010) will cover the Documents tab once S06.14 global interceptor is in place. No new Playwright tests are required for this story specifically — the component Vitest tests provide sufficient coverage at this tier.

### Critical Mistakes to Prevent

1. **Do NOT use `fetch` for the S3 PUT** — XHR is mandatory for upload progress events. `fetch` with `ReadableStream` for upload progress is experimental and not widely supported. Use `XMLHttpRequest` as shown in Dev Notes.

2. **Do NOT send Authorization header to S3** — The presigned URL already embeds AWS auth credentials as query parameters. Any additional `Authorization` header on the PUT request will corrupt the signature and cause a 403 from S3.

3. **Do NOT start polling inside the component** — scan status polling is done via `refetchInterval` on `useOpportunityDocuments` in `DocumentsTab`, not by a `setInterval` or a separate polling hook inside `DocumentUploadComponent`. The component calls `onUploadComplete()` after confirm; `DocumentsTab` handles the refetch.

4. **Do NOT recreate `detail-document-upload-zone` testid** — The S06.11 static placeholder with `data-testid="detail-document-upload-zone"` is replaced entirely. The new testid is `data-testid="upload-zone"` inside `DocumentUploadComponent`. If the S06.11 ATDD test asserts that testid in `DocumentsTab.tsx`, update it to assert the `DocumentUploadComponent` import instead.

5. **Do NOT use `any` type** — TypeScript strict mode is enforced. Use `unknown` for JSONB fields; use proper typing for XHR event and `FileList`.

6. **Queue index for testid** — Use the array index (0-based) for `data-testid="upload-file-item-{index}"` rather than `doc_id` (which doesn't exist until the presigned URL API responds). This makes ATDD assertions possible before network calls complete.

7. **MIME type fallback for ZIP** — Browsers inconsistently report MIME for `.zip`: Chrome uses `application/zip`, others use `application/x-zip-compressed`, some report empty string. Always run `inferMimeType(file.name)` when `file.type` is empty or unrecognised. Use the inferred MIME for both validation and the API request body.

8. **`existingTotalBytes` must come from the parent** — Do NOT fetch the document list inside `DocumentUploadComponent`. The parent `DocumentsTab` already has documents from `useOpportunityDocuments`. Pass the sum as a prop. This avoids duplicate API calls.

9. **Single file input, multiple files** — the `<input type="file" multiple />` is always present but hidden. Clicking the zone triggers `fileInputRef.current.click()`. This is more robust than multiple inputs and works consistently across drag-and-drop and click-to-browse.

### Project Structure Notes

- `DocumentUploadComponent.tsx` goes in `app/[locale]/(protected)/opportunities/components/` — same as S06.11 tab components. This component is tightly coupled to the document upload API and is NOT a generic UI primitive for `packages/ui`.
- All TypeScript strict mode; no `any`; use `unknown` for typed-as-any API fields
- `"use client"` is required: XHR, drag events, `useState`, `useCallback`, `useRef`
- File input `accept` attribute: `.pdf,.docx,.xlsx,.zip` (extension-based; browsers show type hints in file picker)
- `crypto.randomUUID()` is available in all modern browsers and Node 19+; safe to use without polyfill

### References

- Story 6.11 (previous story): `eusolicit-docs/implementation-artifacts/6-11-opportunity-detail-page-tabbed-layout.md` — `DocumentsTab.tsx` static placeholder to replace; established `DocumentRecord` interface; `useOpportunityDocuments` hook; all file paths; `opportunity-utils.tsx` patterns
- Story 6.6 (document upload API): `eusolicit-docs/implementation-artifacts/6-6-document-upload-api-s3-clamav.md` — exact API request/response schema, MIME type list, size limits, scan_status states
- Story 6.7 (document download API): `eusolicit-docs/implementation-artifacts/6-7-document-download-api.md` — 422 for pending/infected scan states
- Epic 6 test design: `eusolicit-docs/test-artifacts/test-design-epic-06.md` — E06-P1-028 (component drag-drop), E06-P2-005 (file size validation), E06-P1-016 (MIME validation)
- Existing opportunities API: `eusolicit-app/frontend/apps/client/lib/api/opportunities.ts`
- Existing opportunity utilities: `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/opportunities/components/opportunity-utils.tsx`
- Existing DocumentsTab: `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/opportunities/components/DocumentsTab.tsx`
- Existing use-opportunities: `eusolicit-app/frontend/apps/client/lib/queries/use-opportunities.ts`
- i18n messages: `eusolicit-app/frontend/apps/client/messages/en.json` + `bg.json`

## Dev Agent Record

### Agent Model Used

claude-opus-4-5

### Debug Log References

None — implementation completed in a single pass.

### Completion Notes List

- All 122 ATDD tests pass GREEN on first run (after fixing 2 regex-alignment issues in source assertions)
- S6.11 ATDD test (238 tests) continues to pass after updating the `detail-document-upload-zone` assertion to `DocumentUploadComponent` import assertion
- Pre-existing failures in s3-8, s3-9, s11-11, s11-13, s12-4 test files are unrelated to this story
- XHR `uploadToS3` uses `Promise`-based wrapper with `signal.addEventListener("abort", () => xhr.abort())` for AbortController integration
- ✅ Resolved review finding [Patch]: Cancel-while-queued — added `cancelledIdsRef` (useRef<Set<string>>) checked at each async boundary in `processFile`; `handleCancel` adds entry ID to set before abort/removal
- ✅ Resolved review finding [Patch]: Package size validation — `handleFiles` now computes `alreadyQueuedBytes` from existing queue entries (status !== "failed") and adds to validation total; `queue` added to deps array
- ✅ Resolved review finding [Patch]: Side effect in setQueue — moved `abort()` call outside the state updater; entry lookup now reads from `queue` state directly; `queue` added to `handleCancel` deps array
- ✅ Resolved review finding [Patch]: Duplicate formatFileSize — removed local `formatFileSize` from `DocumentsTab.tsx`; replaced with `import { formatFileSize } from "./opportunity-utils"`
- All 122 S6.12 + 238 S6.11 ATDD tests pass GREEN after review patches (2026-04-17)
- i18n parity: 638 keys match in both en.json and bg.json

### File List

- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/opportunities/components/DocumentUploadComponent.tsx` — NEW
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/opportunities/components/DocumentsTab.tsx` — MODIFIED
- `eusolicit-app/frontend/apps/client/app/[locale]/(protected)/opportunities/components/opportunity-utils.tsx` — MODIFIED (added formatFileSize export)
- `eusolicit-app/frontend/apps/client/lib/api/opportunities.ts` — MODIFIED (added PresignedUploadResponse, ConfirmUploadResponse, requestPresignedUrl, confirmDocumentUpload)
- `eusolicit-app/frontend/apps/client/lib/queries/use-opportunities.ts` — MODIFIED (added options param to useOpportunityDocuments)
- `eusolicit-app/frontend/apps/client/messages/en.json` — MODIFIED (added 15 upload i18n keys)
- `eusolicit-app/frontend/apps/client/messages/bg.json` — MODIFIED (added 15 upload i18n keys in Bulgarian)
- `eusolicit-app/frontend/apps/client/__tests__/opportunities-detail-s6-11.test.ts` — MODIFIED (updated detail-document-upload-zone assertion)

## Senior Developer Review

### Review Findings

Review date: 2026-04-17 | Layers: Blind Hunter, Edge Case Hunter, Acceptance Auditor | Dismissed: 10

- [x] [Review][Patch] **Cancel-while-queued uploads proceed silently (AC6 violation)** — When the user clicks cancel before `requestPresignedUrl` resolves, `entry.abort` is `undefined` (AbortController is only created after the presigned URL returns). The entry is visually removed from the queue, but `processFile` continues executing: the presigned URL request completes, `uploadToS3` runs, `confirmDocumentUpload` fires, and `onUploadComplete()` triggers a refetch — causing the "cancelled" document to appear in the document list. **Fix:** Track cancelled IDs in a `useRef<Set<string>>` and check `cancelledIds.current.has(entry.id)` at each async boundary in `processFile`. [DocumentUploadComponent.tsx:260-312, 365-376] ✅ Resolved 2026-04-17

- [x] [Review][Patch] **Package size validation ignores in-flight queue bytes (AC3 gap)** — `handleFiles` computes `currentQueueBytes` only from files in the *current batch*. It does not include bytes from files already in the `queue` state array (e.g., files uploading from a prior batch). Since `existingTotalBytes` only reflects server-confirmed documents (updated on refetch), a user can exceed the 500 MB package limit by dropping files in multiple rapid batches before prior uploads are confirmed. **Fix:** In `handleFiles`, compute `alreadyQueuedBytes` from `queue.filter(e => e.status !== "failed").reduce(...)` and add it to the validation total. Add `queue` to the `useCallback` deps array. [DocumentUploadComponent.tsx:316-361] ✅ Resolved 2026-04-17

- [x] [Review][Patch] **Side effect inside `setQueue` updater function (handleCancel)** — `handleCancel` calls `entry.abort()` inside the `setQueue` updater callback. React state updater functions must be pure — React may invoke them multiple times (Strict Mode, concurrent rendering). Calling `abort()` multiple times is idempotent for `AbortController`, so this won't cause a runtime crash, but it violates React's contract. **Fix:** Read the entry from the current `queue` state *before* calling `setQueue`, call `abort()` outside, then update state. [DocumentUploadComponent.tsx:365-376] ✅ Resolved 2026-04-17

- [x] [Review][Patch] **Duplicate `formatFileSize` in `DocumentsTab.tsx` (AC11 intent)** — `DocumentsTab.tsx` defines its own local `formatFileSize` (line 63) instead of importing the shared export from `opportunity-utils.tsx`. Both implementations produce identical results, but maintaining two copies creates a maintenance hazard and contradicts the spec intent that `formatFileSize` be a shared utility. **Fix:** Replace the local function with `import { formatFileSize } from "./opportunity-utils";`. [DocumentsTab.tsx:63-69] ✅ Resolved 2026-04-17

- [x] [Review][Defer] **Orphaned S3 files on confirm failure** — If `confirmDocumentUpload` fails after a successful S3 PUT, the file exists on S3 but is never confirmed or scanned. No recovery path exists for the user. Requires server-side orphan cleanup or client-side confirm retry logic. — deferred, pre-existing architectural gap

- [x] [Review][Defer] **No cleanup on component unmount** — Navigating away from the Documents tab mid-upload leaves XHR requests running. `onUploadComplete` fires on stale closures. Requires `useEffect` cleanup with `AbortController` tracking. — deferred, pre-existing pattern

- [x] [Review][Defer] **0-byte files pass client-side validation** — An empty file (0 bytes) passes all validation checks and uploads to S3. The spec does not require a minimum file size check, and the server should handle this. — deferred, not specified in AC

- [x] [Review][Defer] **Drag-and-drop flickering on child elements** — `onDragLeave` fires when hovering over child elements inside the drop zone (icon, text, button), causing rapid toggling of the `isDragging` highlight. Fix: use a `dragCounter` ref with `dragEnter`/`dragLeave` pair. — deferred, UX polish

- [x] [Review][Defer] **`removeFromQueue` is defined but never called (dead code)** — `handleCancel` uses inline filtering instead of calling the `removeFromQueue` helper. — deferred, code quality
