---
name: file-upload-fullstack
description: Implement end-to-end file uploads with drag-and-drop UI, presigned URLs, direct-to-cloud uploads, progress handling, backend confirmation, and CDN delivery. Use when building scalable upload flows or avoiding server-side proxy uploads. Trigger on file uploads, S3/GCS presigned URLs, direct uploads, multipart uploads, or file storage/CDN setup.
category: "Fullstack"
---

# File Upload Fullstack

Covers the non-obvious parts of a complete upload pipeline: why you never proxy file bytes through your own server, the two-phase commit pattern for upload confirmation, progress tracking without a server round-trip, and the CDN delivery gotchas that break cached files after replacement. Skips basic form handling — assumes a storage bucket exists.

---

## Discovery

Before writing anything, answer:

1. **Storage provider**: AWS S3, GCS, Cloudflare R2, Azure Blob? (presigned URL API differs per provider)
2. **File types and size limits**: Images only, or arbitrary files? Max size? (determines chunking strategy)
3. **Access control**: Public files (CDN-served directly) or private files (signed CDN URLs per request)?
4. **Confirmation pattern**: Does the backend need to know a file was uploaded? (almost always yes — for DB records, processing jobs, virus scanning)
5. **Replacement behavior**: Can files be overwritten, or does each upload get a unique key?

---

## Core Patterns

### 1. Why Direct Upload, Not Proxy

**Never** pipe file bytes through your own server:

```
WRONG: Client → [file bytes] → Your server → [file bytes] → S3
RIGHT: Client → [presign request] → Your server → [presigned URL] → Client → [file bytes] → S3
```

Proxying through your server: doubles bandwidth cost, blocks server threads during large uploads, makes your server a bottleneck, and bypasses CDN. The presigned URL pattern moves bytes directly from client to storage — your server only authorizes the upload.

---

### 2. Presigned URL Generation (Backend)

```ts
// AWS S3 — generate a presigned PUT URL
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { randomUUID } from 'crypto';

const s3 = new S3Client({ region: process.env.AWS_REGION });

interface PresignRequest {
  filename: string;
  contentType: string;
  sizeBytes: number;
}

interface PresignResponse {
  uploadUrl: string;    // PUT to this URL directly from the client
  fileKey: string;      // store this in your DB to reference the file
  publicUrl: string;    // CDN/S3 URL to read the file after upload
}

const MAX_SIZE = 10 * 1024 * 1024; // 10MB
const ALLOWED_TYPES = new Set(['image/jpeg', 'image/png', 'image/webp', 'application/pdf']);

async function generatePresignedUrl(
  userId: string,
  { filename, contentType, sizeBytes }: PresignRequest
): Promise<PresignResponse> {
  // Validate before generating — don't trust the client
  if (!ALLOWED_TYPES.has(contentType)) throw new Error('File type not allowed');
  if (sizeBytes > MAX_SIZE) throw new Error('File too large');

  // Non-obvious: never use the original filename as the S3 key
  // User-controlled filenames can contain path traversal, overwrite other files,
  // or create collisions. Always generate the key server-side.
  const ext = filename.split('.').pop()?.toLowerCase() ?? '';
  const fileKey = `uploads/${userId}/${randomUUID()}.${ext}`;

  const command = new PutObjectCommand({
    Bucket: process.env.S3_BUCKET!,
    Key: fileKey,
    ContentType: contentType,
    ContentLength: sizeBytes,       // enforced by S3 — client can't upload a different size
    // Non-obvious: set metadata here, not after upload (can't add it later without re-uploading)
    Metadata: { uploadedBy: userId },
  });

  const uploadUrl = await getSignedUrl(s3, command, {
    expiresIn: 300, // 5 minutes — short enough to limit abuse, long enough for slow connections
  });

  return {
    uploadUrl,
    fileKey,
    publicUrl: `${process.env.CDN_BASE_URL}/${fileKey}`,
  };
}
```

**Non-obvious**: `ContentLength` in the `PutObjectCommand` is enforced server-side by S3. If the client tries to upload a different number of bytes than declared, S3 rejects it. Always include it.

---

### 3. Frontend — Drag-and-Drop + Direct Upload

```ts
// uploadFile.ts — the two-step upload client
async function uploadFile(file: File, userId: string): Promise<string> {
  // Step 1: get presigned URL from your backend
  const { uploadUrl, fileKey, publicUrl } = await fetch('/api/uploads/presign', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      filename:    file.name,
      contentType: file.type,
      sizeBytes:   file.size,
    }),
  }).then(r => r.json());

  // Step 2: PUT directly to S3 — no auth headers, no JSON, just the raw file
  const uploadRes = await fetch(uploadUrl, {
    method: 'PUT',
    body: file,                           // raw File object, not FormData
    headers: { 'Content-Type': file.type }, // must match what was presigned
  });

  if (!uploadRes.ok) throw new Error(`Upload failed: ${uploadRes.status}`);

  // Step 3: confirm with backend (covered in pattern 4)
  await confirmUpload(fileKey);

  return publicUrl;
}
```

**Upload progress** — `fetch` doesn't expose progress. Use `XMLHttpRequest` for progress events, or the newer `ReadableStream` approach:

```ts
function uploadWithProgress(
  uploadUrl: string,
  file: File,
  onProgress: (pct: number) => void
): Promise<void> {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();
    xhr.open('PUT', uploadUrl);
    xhr.setRequestHeader('Content-Type', file.type);

    xhr.upload.addEventListener('progress', (e) => {
      if (e.lengthComputable) onProgress(Math.round((e.loaded / e.total) * 100));
    });

    xhr.addEventListener('load', () =>
      xhr.status >= 200 && xhr.status < 300 ? resolve() : reject(new Error(`${xhr.status}`))
    );
    xhr.addEventListener('error', () => reject(new Error('Network error')));
    xhr.send(file);
  });
}
```

**Drag-and-drop zone** — the non-obvious events:

```ts
// Must prevent default on dragover, not just drop, or the browser opens the file
dropZone.addEventListener('dragover', (e) => {
  e.preventDefault(); // required — otherwise drop event never fires
  e.dataTransfer!.dropEffect = 'copy';
});

dropZone.addEventListener('drop', (e) => {
  e.preventDefault();
  const files = Array.from(e.dataTransfer!.files);
  // Non-obvious: e.dataTransfer.files is not a real array — must spread or Array.from
  handleFiles(files);
});
```

---

### 4. Two-Phase Commit — Upload Confirmation

**The problem**: the client uploaded a file directly to S3. Your backend has no idea it happened. Without confirmation, you can't:
- Create the DB record linking the file to an entity
- Trigger processing jobs (resize, virus scan, transcode)
- Prevent orphaned files (presigned URL generated but upload never completed)

```ts
// Backend: confirmation endpoint
router.post('/api/uploads/confirm', async (req, res) => {
  const { fileKey, entityId, entityType } = req.body;

  // Verify the file actually exists in S3 before writing the DB record
  // Non-obvious: clients can call this with any fileKey — verify ownership
  const expectedPrefix = `uploads/${req.user.id}/`;
  if (!fileKey.startsWith(expectedPrefix)) {
    return res.status(403).json({ error: 'Forbidden' });
  }

  // Check file exists in S3 (optional but prevents ghost DB records)
  try {
    await s3.send(new HeadObjectCommand({ Bucket: process.env.S3_BUCKET!, Key: fileKey }));
  } catch {
    return res.status(404).json({ error: 'File not found in storage' });
  }

  // Write DB record
  const attachment = await db.attachment.create({
    data: { fileKey, entityId, entityType, uploadedBy: req.user.id },
  });

  // Trigger async processing if needed
  await queue.add('process-upload', { fileKey, attachmentId: attachment.id });

  res.json({ attachmentId: attachment.id });
});
```

**Non-obvious**: always verify the `fileKey` starts with the uploading user's prefix. Without this check, any authenticated user can "confirm" a file belonging to another user, hijacking their upload.

---

### 5. Client-Side Validation Before Presigning

Validate on the client to fail fast, but never trust it on the backend. Client validation is UX; backend validation is security.

```ts
const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/webp'];
const MAX_SIZE_MB = 10;

function validateFile(file: File): string | null {
  if (!ALLOWED_TYPES.includes(file.type)) {
    return `File type not supported. Use: ${ALLOWED_TYPES.join(', ')}`;
  }
  if (file.size > MAX_SIZE_MB * 1024 * 1024) {
    return `File must be under ${MAX_SIZE_MB}MB`;
  }
  // Non-obvious: file.type is set by the browser from the extension — not the actual bytes.
  // A user can rename malware.exe to malware.jpg and file.type will be 'image/jpeg'.
  // Real content-type validation must happen on the backend via magic bytes or virus scan.
  return null;
}
```

---

### 6. CDN Delivery and Cache Invalidation

**Public files**: point CDN origin at your S3 bucket. Files are served at edge.

**Private files**: generate signed CDN URLs (CloudFront, Cloudflare) per request — never expose the S3 URL directly.

```ts
// CloudFront signed URL (private files)
import { getSignedUrl } from '@aws-sdk/cloudfront-signer';

function getSignedCdnUrl(fileKey: string, expiresInSeconds = 3600): string {
  return getSignedUrl({
    url: `${process.env.CDN_BASE_URL}/${fileKey}`,
    keyPairId: process.env.CLOUDFRONT_KEY_PAIR_ID!,
    dateLessThan: new Date(Date.now() + expiresInSeconds * 1000).toISOString(),
    privateKey: process.env.CLOUDFRONT_PRIVATE_KEY!,
  });
}
```

**Cache invalidation on file replacement** — the most commonly missed step:

```ts
// If you allow overwriting a file at the same key, the CDN will serve the old version
// until the TTL expires (potentially hours or days).

// Option A: always use unique keys (UUID-based) — no invalidation needed, old URL just 404s
const fileKey = `uploads/${userId}/${randomUUID()}.${ext}`; // ← preferred

// Option B: invalidate after overwrite
import { CloudFrontClient, CreateInvalidationCommand } from '@aws-sdk/client-cloudfront';

const cf = new CloudFrontClient({});
await cf.send(new CreateInvalidationCommand({
  DistributionId: process.env.CLOUDFRONT_DIST_ID!,
  InvalidationBatch: {
    CallerReference: Date.now().toString(),
    Paths: { Quantity: 1, Items: [`/${fileKey}`] },
  },
}));
// Non-obvious: invalidations cost $0.005 per path after the free tier — use sparingly
```

---

### 7. Multipart Upload for Large Files (>100MB)

Standard presigned PUT has a 5GB limit and no resumability. For large files, use multipart:

```ts
// Backend: initiate multipart upload
import { CreateMultipartUploadCommand, UploadPartCommand,
         CompleteMultipartUploadCommand } from '@aws-sdk/client-s3';

async function initiateMultipartUpload(fileKey: string, contentType: string) {
  const { UploadId } = await s3.send(new CreateMultipartUploadCommand({
    Bucket: process.env.S3_BUCKET!,
    Key: fileKey,
    ContentType: contentType,
  }));
  return UploadId;
}

// Backend: generate presigned URL per part (client uploads each part directly)
async function presignPart(fileKey: string, uploadId: string, partNumber: number) {
  return getSignedUrl(s3, new UploadPartCommand({
    Bucket: process.env.S3_BUCKET!,
    Key: fileKey,
    UploadId: uploadId,
    PartNumber: partNumber,   // 1-indexed, 1–10000
  }), { expiresIn: 3600 });
}

// Backend: complete after all parts uploaded
async function completeMultipartUpload(
  fileKey: string,
  uploadId: string,
  parts: { ETag: string; PartNumber: number }[]
) {
  await s3.send(new CompleteMultipartUploadCommand({
    Bucket: process.env.S3_BUCKET!,
    Key: fileKey,
    UploadId: uploadId,
    MultipartUpload: { Parts: parts }, // ETags come from the PUT response headers per part
  }));
}
```

**Non-obvious**: the client must capture the `ETag` response header from each part's PUT request. Without it, `CompleteMultipartUpload` can't be called. Parts must be ≥5MB each (except the last), or S3 rejects the completion.

---

## Output

Produce:
- `api/uploads.ts` — presign endpoint + confirm endpoint with ownership check + HeadObject verification
- `uploadFile.ts` — two-step client (presign → PUT → confirm) with progress via XHR
- `FileDropZone.tsx` — drag-and-drop component with client-side validation
- `cdn.ts` — signed CDN URL generator for private files

Flag clearly in comments:
- Which validations are UX-only (client) vs security boundaries (backend)
- The ownership check on confirmation and why it's required
- Cache invalidation cost and when to use unique keys instead
- Multipart thresholds (>100MB, parts ≥5MB)