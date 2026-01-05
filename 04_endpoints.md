Rendben 👍
Az alábbiakban **elkészítettem a teljes `docs/04_endpoints.md` fájlt**, **végig tiszta Markdown formátumban**, úgy, hogy:

* közvetlenül bemásolható a repo-ba,
* pontosan tükrözi a jelenlegi implementációt,
* fejlesztőknek szól (nem marketing, nem user guide),
* megfelel annak, amit az issue + Slack alapján kértek.

---

```md
# API Endpoints Documentation

This document provides a detailed overview of all available backend API endpoints
implemented under the `/api/v1` prefix.

All endpoints are grouped by functional domain and follow RESTful conventions.

---

## Base Path and Versioning

All endpoints are available under the following base path:

```

/api/v1

````

The API is versioned to support future backward-compatible changes.
New versions will be introduced under `/api/v2`, `/api/v3`, etc.

---

## Health

### GET `/api/v1/health/`

Simple health check endpoint.

**Authentication:** Not required

**Response:**
- `200 OK`
```json
{
  "status": "ok"
}
````

---

## Users

### GET `/api/v1/users/all`

Returns all users in the system.

**Authentication:** Not required

**Response:**

* `200 OK`
* Response model: `UserResponse[]`

---

### POST `/api/v1/users/`

Creates a new user.

**Authentication:** Not required

**Request body:**

* `UserCreate`

  * name: string
  * email: string
  * password: string

**Responses:**

* `201 Created` – User successfully created
* `409 Conflict` – Email already registered
* `422 Unprocessable Entity` – Validation error

**Notes:**

* Passwords are hashed before being stored.
* A user-specific directory is created on the filesystem.

---

### GET `/api/v1/users/{user_uid}`

Returns a user by unique identifier.

**Authentication:** Not required

**Path parameters:**

* `user_uid` – User unique identifier

**Responses:**

* `200 OK`
* `404 Not Found` – User not found

---

## Authentication & Authorization

### POST `/api/v1/auth/register`

Registers a new user and sends an email verification link.

**Authentication:** Not required

**Request body:**

* `UserCreate`

**Responses:**

* `201 Created` – User registered successfully
* `409 Conflict` – Email already registered
* `422 Unprocessable Entity` – Validation error

**Side effects:**

* Password is hashed before storage.
* User is created with `is_verified = false`.
* Verification email is sent asynchronously.
* Email verification token has a limited validity (configured via environment variables).

---

### GET `/api/v1/auth/verify`

Verifies a user's email address.

**Authentication:** Not required

**Query parameters:**

* `token` – Email verification token

**Responses:**

* `200 OK` – Email verified or already verified
* `400 Bad Request` – Invalid or expired token
* `404 Not Found` – User not found

---

### POST `/api/v1/auth/resend-verification`

Resends the email verification link.

**Authentication:** Not required

**Request body:**

* `ResendVerificationRequest`

  * email: string

**Responses:**

* `200 OK`

**Security notes:**

* Anti-enumeration protection is applied.
* The same response is returned regardless of user existence.
* A cooldown period limits how frequently emails can be resent.

---

### POST `/api/v1/auth/login`

Authenticates a user and issues JWT tokens.

**Authentication:** Not required

**Request body:**

* `LoginRequest`

  * email: string
  * password: string

**Responses:**

* `200 OK` – Login successful
* `401 Unauthorized` – Invalid credentials
* `403 Forbidden` – Email not verified

**Notes:**

* Issues access and refresh JWT tokens.
* Tokens are stored in HTTP cookies.
* Refresh tokens are persisted in the database.

---

### POST `/api/v1/auth/refresh`

Refreshes access and refresh tokens.

**Authentication:** Required (refresh token cookie)

**Responses:**

* `200 OK`
* `401 Unauthorized` – Invalid, missing, or revoked refresh token

**Notes:**

* Implements refresh token rotation.
* Old refresh tokens are revoked on use.

---

### POST `/api/v1/auth/logout`

Logs out the current user.

**Authentication:** Required (refresh token cookie)

**Responses:**

* `200 OK`

**Side effects:**

* Refresh token is revoked.
* Authentication cookies are deleted.

---

### GET `/api/v1/auth/me`

Returns information about the currently authenticated user.

**Authentication:** Required (access token)

**Responses:**

* `200 OK`

```json
{
  "user_uid": "...",
  "email": "..."
}
```

---

## Jobs

### GET `/api/v1/jobs`

Returns all jobs associated with the authenticated user.

**Authentication:** Required

**Query parameters:**

* `include_archived` (boolean, default: true)

**Responses:**

* `200 OK`
* Response model: `JobOut[]`

---

### POST `/api/v1/jobs`

Creates a new job.

**Authentication:** Required

**Request body:**

* `JobCreateIn`

**Responses:**

* `201 Created`
* `500 Internal Server Error` – Job creation failed

**Side effects:**

* Job directory is created on the filesystem.
* Database and filesystem operations are coordinated.

---

### POST `/api/v1/jobs/{job_uid}/archive`

Archives a job.

**Authentication:** Required

**Authorization:** Job owner only

**Responses:**

* `200 OK`
* `403 Forbidden` – Not the job owner
* `404 Not Found` – Job not found

**Notes:**

* Operation is idempotent.

---

### POST `/api/v1/jobs/{job_uid}/transfer-ownership`

Transfers job ownership to another user.

**Authentication:** Required

**Authorization:** Job owner only

**Request body:**

* `TransferOwnershipIn`

  * newOwnerUId: string

**Responses:**

* `200 OK`
* `400 Bad Request` – Target user not verified
* `403 Forbidden` – Not the job owner
* `404 Not Found` – Job or target user not found

---

### PATCH `/api/v1/jobs/{job_uid}`

Renames a job.

**Authentication:** Required

**Authorization:** Job owner only

**Request body:**

* `JobRenameIn`

**Responses:**

* `200 OK`
* `403 Forbidden`
* `404 Not Found`

---

### POST `/api/v1/jobs/{job_uid}/assets`

Creates a new asset for a job.

**Authentication:** Required

**Responses:**

* `201 Created`
* `403 Forbidden` – No access to job
* `404 Not Found` – Job not found

**Side effects:**

* Asset directory structure is created on the filesystem.

---

## File Uploads & Asset Processing

### POST `/api/v1/jobs/{job_uid}/assets/upload`

Uploads a new asset (video + skeleton).

**Authentication:** Required

**Authorization:** Roles: EDIT, OWNER, ADMIN

**Request:**

* Multipart form-data:

  * `video` – Video file
  * `skeleton` – JSON file
  * `start_hls` (optional, query parameter)

**Responses:**

* `201 Created`
* `400 Bad Request` – Invalid file type or missing video
* `403 Forbidden`
* `404 Not Found`
* `500 Internal Server Error`

**Constraints:**

* Video size limit: 5 GB
* Skeleton JSON size limit: 200 MB

---

### POST `/api/v1/jobs/{job_uid}/assets/{asset_uid}/upload`

Re-uploads files to an existing asset.

**Authentication:** Required

**Authorization:** Roles: EDIT, OWNER, ADMIN

**Responses:**

* `201 Created`
* `403 Forbidden`
* `404 Not Found`
* `409 Conflict` – Asset not in uploadable state

**Notes:**

* Operation is retry-safe.
* Asset status is updated accordingly.

---

### POST `/api/v1/jobs/{job_uid}/assets/{asset_uid}/process/hls`

Starts HLS transcoding for an asset.

**Authentication:** Required

**Authorization:** Roles: EDIT, OWNER, ADMIN

**Responses:**

* `202 Accepted`
* `400 Bad Request` – Missing source video
* `403 Forbidden`
* `404 Not Found`
* `409 Conflict` – Asset already processing
* `500 Internal Server Error`

**Side effects:**

* FFmpeg subprocess is started.
* HLS output is written under the asset directory.
* Processing state and logs are stored in metadata files.

---

```

---
