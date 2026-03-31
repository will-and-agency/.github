# File Upload & Attachment System

## Overview

Attachments support uploading files (images, videos, audio, PDFs, CSVs) to tasks, questions, answers, and projects. Files are stored in an S3-compatible object storage (OVHcloud in production, MinIO locally). The backend never handles file bytes directly — uploads and downloads go straight between the client and storage.

## Architecture

src/storage.rs       — StorageBackend trait + S3Storage implementation
src/dto/attachments  — request/response types, validation constants
src/handlers/attachment.rs — all attachment handlers
src/routes/attachment.rs   — route definitions
tests/common/mod.rs  — FakeStorage (tests only, never ships)



### StorageBackend trait

Abstracts over real and fake storage. Production uses `S3Storage` (AWS SDK v2, `force_path_style=true` for OVHcloud/MinIO compatibility). Tests use `FakeStorage` (in-memory HashSet, no network calls).

```rust
trait StorageBackend {
    presigned_upload_url(key, mime_type)      // PUT URL for client upload
    presigned_download_url(key, expires_secs) // GET URL returned in responses
    object_exists(key)                        // HEAD check used in confirm
    delete_object(key)                        // called on replace + delete
}
Upload Flow
Every file upload follows three steps, all triggered automatically by the client with no user interaction between them:


1. POST .../attachments/init
       ↓ backend validates, generates file_key, creates DB row (pending),
         returns presigned PUT URL (15 min)

2. PUT <upload_url>   ← client uploads directly to storage (no backend involved)
       ↓

3. POST .../attachments/{id}/confirm
       ↓ backend HEAD-checks file exists in storage,
         marks row as ready, runs singleton replacement if applicable,
         returns AttachmentDto with presigned GET URL (1 hour)
The pending row stays in the DB until confirm is called. If the upload fails and confirm is never called, the row stays pending forever (cleanup job TBD).

Presigned GET URLs
Every AttachmentDto response includes a url field — a short-lived presigned GET URL valid for 1 hour (configurable in src/dto/attachments.rs). The client uses this URL directly in <img>, <video>, or <audio> tags. No auth header is needed to use the URL.

Security model: auth happens when calling the API (JWT required). The URL itself is public for its lifetime. Anyone with the URL can access the file for up to 1 hour — equivalent security to the proxy+token approach, with zero backend overhead for file delivery.

For video/audio: if the URL expires mid-playback, the player should catch the error, call the GET attachments endpoint to get a fresh URL, and resume from the current timestamp.

Singleton Roles
Some roles only allow one active attachment per parent at a time. When a second upload of the same singleton role is confirmed, the previous attachment is automatically deleted from both storage and the DB.

Role	Singleton	Used on
cover_image	Yes	task, question, project
intro_video	Yes	task
userlist	Yes	project
answer	Yes	answer
task_attachment	No	task
question_attachment	No	question
Replace-on-confirm guarantee: the old attachment stays ready until the new one is fully uploaded and confirmed. No gap in availability.

Validation
Enforced in the handler before any DB write or storage call:

Role must be one of ALLOWED_ROLES
MIME type must be one of ALLOWED_MIME_TYPES (video, image, audio, pdf, csv, json)
File size must be between 1 byte and 500 MB
API Endpoints
All routes require a valid JWT (Authorization: Bearer <token>).

Task attachments

GET    /projects/{project_id}/tasks/{task_id}/attachments
GET    /projects/{project_id}/tasks/{task_id}/attachments/cover-image
POST   /projects/{project_id}/tasks/{task_id}/attachments/init
POST   /projects/{project_id}/tasks/{task_id}/attachments/{id}/confirm
Question attachments

GET    /projects/{project_id}/tasks/{task_id}/questions/{question_id}/attachments
POST   /projects/{project_id}/tasks/{task_id}/questions/{question_id}/attachments/init
POST   /projects/{project_id}/tasks/{task_id}/questions/{question_id}/attachments/{id}/confirm
Answer attachments

GET    /projects/{project_id}/tasks/{task_id}/questions/{question_id}/answers/{answer_id}/attachments
POST   /projects/{project_id}/tasks/{task_id}/questions/{question_id}/answers/{answer_id}/attachments/init
POST   /projects/{project_id}/tasks/{task_id}/questions/{question_id}/answers/{answer_id}/attachments/{id}/confirm
Project attachments

GET    /projects/{project_id}/attachments
GET    /projects/{project_id}/attachments/cover-image
POST   /projects/{project_id}/attachments/init
POST   /projects/{project_id}/attachments/{id}/confirm
Delete (any attachment)

DELETE /projects/{project_id}/attachments/{id}
The project_id is used for authorization — the backend verifies the attachment belongs to that project by walking the parent chain (attachment → task/question/answer → project).

Request / Response
POST .../init — body

{
  "role": "cover_image",
  "mime_type": "image/jpeg",
  "file_size_bytes": 204800
}
POST .../init — response (201)

{
  "attachment_id": "uuid",
  "upload_url": "https://storage/bucket/key?X-Amz-Signature=...",
  "file_key": "attachments/{parent_id}/{uuid}"
}
AttachmentDto (returned by GET, confirm)

{
  "id": "uuid",
  "task_id": "uuid | null",
  "question_id": "uuid | null",
  "answer_id": "uuid | null",
  "project_id": "uuid | null",
  "role": "cover_image",
  "file_key": "attachments/...",
  "mime_type": "image/jpeg",
  "file_size_bytes": 204800,
  "processing_status": "ready",
  "created_at": "2026-03-31T...",
  "url": "https://storage/bucket/key?X-Amz-Signature=..."
}
Local Development (MinIO)
Start MinIO with:


docker compose up -d minio minio_init
S3 API: http://localhost:9000
Web console: http://localhost:9001 (login: minioadmin / minioadmin)
Bucket attachments is created automatically with private access
Environment variables (.env):


S3_ENDPOINT=http://localhost:9000
S3_REGION=us-east-1
S3_BUCKET=attachments
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
Testing

cargo test attachment -- --test-threads=1
Tests use FakeStorage — no MinIO required. simulate_uploaded(key) inserts a key into the fake storage's in-memory set, standing in for an actual client PUT.


