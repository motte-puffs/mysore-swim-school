# API Contract (Phase 1 – Single Source of Truth)

> **Status:** 🟡 Awaiting User Approval · Created: 2026‑11‑17

---

## 1. Base URL
All endpoints are prefixed with:
```
http://<host>/api/v1
```
*During local development the host is `localhost` (served by Nginx). In production it will be the real domain (e.g., `https://swimacademy.in`).*

---

## 2. Authentication
| Route type | Requirement |
|------------|--------------|
| **Public** (`/public/**`) | No authentication (open to all visitors). Rate limiting applies. |
| **Admin** (`/admin/**`) | HTTP Basic Authentication. Header: `Authorization: Basic <base64(username:password)>`. Username/password are taken from environment variables `ADMIN_USERNAME` / `ADMIN_PASSWORD`. |

---

## 3. Error Response Format (RFC 7807 Problem Detail)
Whenever the server returns an error (4xx/5xx), the payload is JSON with the following fields:
```json
{
  "type": "https://swimacademy.in/errors/<slug>",
  "title": "<Human readable title>",
  "status": <http status code>,
  "detail": "<detailed error message>",
  "instance": "<request path>"
}
```
**Common `slug` values**:
- `duplicate-enrollment`
- `batch-full`
- `batch-program-mismatch`
- `invalid-status-transition`
- `age-group-mismatch`
- `validation-failed`
- `not-found`
- `rate-limit-exceeded`

---

## 4. Public Endpoints
| Method | Path | Description | Request Body | Response Body | Success Code | Errors (common) |
|--------|------|-------------|--------------|---------------|--------------|----------------|
| `GET` | `/public/coaches` | List all active coaches. | – | `CoachResponse[]` | `200` | `429` (rate limit) |
| `GET` | `/public/programs` | List programs, optional filters `ageGroup` (`TODDLER`/`KIDS`/`TEENS`/`ADULTS`) and `skillLevel` (`BEGINNER`/`INTERMEDIATE`/`ADVANCED`). | – | `ProgramResponse[]` | `200` | `429` |
| `GET` | `/public/programs/{id}` | Program details **including read‑only batch schedule**. | – | `ProgramDetailResponse` (program fields + `batches: BatchInfo[]`) | `200` | `404`, `429` |
| `GET` | `/public/academy-info` | Key‑value metadata (address, phone, map iframe URL, etc.). | – | `AcademyInfoResponse` (object of `key: value`) | `200` | `429` |
| `POST` | `/public/enrollments` | Submit a new enrollment (admin‑gated). | `EnrollmentRequest` (see schema) | `EnrollmentResponse` (`referenceId`, `status: "PENDING"`, `createdAt`) | `201` | `400` (validation), `409` (duplicate), `422` (age‑group mismatch) |
| `GET` | `/public/enrollments/{referenceId}` | Retrieve status of a specific enrollment (no reason shown). | – | `EnrollmentStatusResponse` (`referenceId`, `status`, optionally `programId`, `batchId`) | `200` | `404` |
| `POST` | `/public/inquiries` | Submit a contact‑us inquiry. | `InquiryRequest` | `InquiryResponse` (`id`, `createdAt`) | `201` | `400` |

### 4.1 Request / Response Schemas (public)
```json
// EnrollmentRequest
{
  "applicantType": "PARENT_CHILD" | "ADULT_SELF",
  "studentName": "string",               // required, non‑empty
  "studentDob": "yyyy-MM-dd",            // optional, ISO date (validated if present)
  "guardianName": "string",             // required when applicantType = PARENT_CHILD
  "phone": "string",                     // required, 10‑12 digits (regex ^\d{10,12}$)
  "email": "string",                     // optional, valid email format
  "emergencyContact": "string",          // optional
  "medicalNotes": "string",              // optional
  "swimmingBackground": "string",        // optional
  "programId": number                     // required, must refer to an existing program
}
```
```json
// EnrollmentResponse (after POST)
{
  "referenceId": "SWM-20261117-1A2B3C4D",
  "status": "PENDING",
  "studentName": "…",
  "program": { "id": 3, "name": "Kids Beginner" },
  "createdAt": "2026-11-17T17:30:00Z"
}
```
```json
// EnrollmentStatusResponse (GET by referenceId)
{
  "referenceId": "SWM-20261117-1A2B3C4D",
  "status": "PENDING" | "CONFIRMED" | "REJECTED",
  "programId": 3,
  "batchId": null | number
}
```
```json
// InquiryRequest
{
  "name": "string",
  "email": "string" optional,
  "phone": "string",
  "message": "string"
}
```
```json
// InquiryResponse
{
  "id": 42,
  "createdAt": "2026-11-17T18:00:00Z"
}
```
```json
// CoachResponse (public list)
{
  "id": 5,
  "name": "Jane Doe",
  "photoUrl": "https://cdn.example.com/coaches/jane.jpg",
  "bio": "…",
  "specialties": "Toddler water safety, Freestyle",
  "certifications": "Level 2 Coach",
  "experienceYears": 8,
  "active": true
}
```
```json
// ProgramResponse (list)
{
  "id": 3,
  "name": "Kids Beginner",
  "ageGroup": "KIDS",
  "skillLevel": "BEGINNER",
  "syllabus": "…",
  "sessionCount": 12,
  "packageDurationDays": 90,
  "feePaise": 250000,
  "active": true
}
```
```json
// ProgramDetailResponse (single program + batch info)
{
  "id": 3,
  "name": "Kids Beginner",
  "ageGroup": "KIDS",
  "skillLevel": "BEGINNER",
  "syllabus": "…",
  "sessionCount": 12,
  "packageDurationDays": 90,
  "feePaise": 250000,
  "active": true,
  "batches": [
    {
      "id": 7,
      "name": "Morning Batch – Mon/Wed/Fri",
      "startTime": "07:00",
      "endTime": "08:00",
      "daysOfWeek": "MON,WED,FRI",
      "batchStartDate": "2026-12-01",
      "batchEndDate": "2027-02-28",
      "capacity": 15,
      "active": true
    }
  ]
}
```
```json
// AcademyInfoResponse (key‑value map)
{
  "address": "123 Swim Lane, City, State",
  "phone": "+91‑9876543210",
  "whatsappLink": "https://wa.me/919876543210",
  "mapIframeUrl": "<iframe src=\"https://www.google.com/maps/embed?...\" ...></iframe>",
  "facilityHighlights": "Olympic‑size pool, heated water, ..."
}
```
---

## 5. Admin Endpoints (protected)
| Method | Path | Description | Request Body | Response Body | Success Code | Errors |
|--------|------|-------------|--------------|---------------|--------------|--------|
| `GET` | `/admin/enrollments` | List enrollments (filterable). | – | `EnrollmentResponse[]` (full payload) | `200` | `401` |
| `GET` | `/admin/enrollments/{id}` | Retrieve full enrollment record. | – | `EnrollmentFullResponse` (all fields) | `200` | `401`, `404` |
| `PUT` | `/admin/enrollments/{id}/approve` | Approve and assign batch. | `{ "batchId": number }` | `EnrollmentFullResponse` (status = `CONFIRMED`, dates populated) | `200` | `401`, `404`, `422` (batch‑program mismatch), `409` (batch full) |
| `PUT` | `/admin/enrollments/{id}/reject` | Reject enrollment. | `{ "reason": "string" }` | `{ "id": number, "status": "REJECTED", "rejectionReason": "string" }` | `200` | `401`, `404`, `409` (invalid status) |
| `GET/POST/PUT/DELETE` | `/admin/coaches`, `/admin/programs`, `/admin/batches` | Standard CRUD for catalog. | Varies per method | Varies | `200`/`201`/`204` | `401`, `404`, `400` |
| `PUT` | `/admin/academy-info/{key}` | Update a single key‑value entry. | `{ "value": "string" }` | Updated key/value object | `200` | `401`, `404` |

### 5.1 Admin DTO examples
```json
// ApproveRequest
{ "batchId": 7 }
```
```json
// RejectRequest
{ "reason": "Applicant did not meet age criteria" }
```
---

## 6. Validation Rules (summary)
| Field | Rule |
|-------|------|
| `applicantType` | Must be `PARENT_CHILD` or `ADULT_SELF`.
| `studentName` | Non‑empty string, max 150 chars.
| `studentDob` | If present, must be a valid ISO date; server will compute age‑group compatibility.
| `guardianName` | Required when `applicantType = PARENT_CHILD`.
| `phone` | 10‑12 digits, regex `^\d{10,12}$`.
| `email` | If present, must match email format.
| `programId` | Positive integer, must refer to an existing active program.
| `batchId` (admin approve) | Must belong to the same program as the enrollment, and batch must be active.
| `feePaise` | Stored as BIGINT; front‑end divides by 100 for display.
| `status` | One of `PENDING`, `CONFIRMED`, `REJECTED` (no other values).

---

## 7. Rate Limiting (public only)
| Endpoint | Limit |
|----------|-------|
| `POST /public/enrollments` | 5 requests per 10 min per IP.
| `POST /public/inquiries` | 10 requests per 10 min per IP.
| `GET /public/**` | 60 requests per minute per IP.

When limit is exceeded, server returns `429 Too Many Requests` with an RFC 7807 payload (`type: "https://swimacademy.in/errors/rate-limit-exceeded"`).

---

## 8. Coordination Guidelines
* **Any change** to endpoint paths, request/response field names, or status values **must** be made **first** in this `API_CONTRACT.md` file.
* After editing, open a pull request; **both developers must approve** before merging.
* Once merged:
  - Dev 1 updates DTO classes, validation annotations, and controller signatures.
  - Dev 2 updates TypeScript interfaces in `frontend/src/types/`, Zod schemas in `frontend/src/schemas/`, and mock JSON.
* Mock data (`frontend/src/mock/api-mocks.json`) is generated from the seed data in the contract; keep it in sync by running `npm run generate-mocks` after any contract change.
* The shared `docker-compose.yml` uses the same base URL for both services; changing the base path requires updating this contract **and** the env variable `VITE_API_BASE_URL`.

---

## 9. Open Decisions (still pending from the original request)
> [!IMPORTANT]
> **Academy branding** – official name, tagline, and primary brand colors for Tailwind config.
>
> [!IMPORTANT]
> **Re‑application policy** – after a rejection, should the applicant be allowed to submit a new enrollment for the same program? (Current contract permits it because duplicate‑block excludes `status = REJECTED`).

Please confirm these two items so the UI can reflect the brand and the backend logic can be finalised.

---

*This file is stored as an artifact and will be updated automatically if you request changes.*