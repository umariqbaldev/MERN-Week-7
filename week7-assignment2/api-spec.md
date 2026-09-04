# Task Management Platform — REST API Contract

> **Version:** v1 — all routes are prefixed with `/api/v1`
>
> **Base URL:** `https://api.example.com/api/v1`

---

## Table of Contents

1. [Conventions](#1-conventions)
2. [Error Contract](#2-error-contract)
3. [Authentication Endpoints](#3-authentication-endpoints)
4. [Tasks Resource](#4-tasks-resource)
5. [Projects Resource](#5-projects-resource)
6. [Users Resource](#6-users-resource)
7. [Project Members Resource](#7-project-members-resource)
8. [Tags Resource](#8-tags-resource)
9. [Comments Resource](#9-comments-resource)
10. [Idempotent Create (X1)](#10-idempotent-create)
11. [Full Status-Code Table — Tasks (X2)](#11-full-status-code-table--tasks)

---

## 1. Conventions

### 1.1 Versioning

Every route carries the prefix `/api/v1`. When a breaking change is introduced, a new version (`/api/v2`) is deployed alongside the old one. The old version is supported for at least 6 months after the new one is stable.

### 1.2 Access Levels

Every endpoint is labelled with one of the following:

| Level | Meaning |
|---|---|
| **Public** | No authentication required. |
| **Authenticated** | Requires a valid JWT access token in the `Authorization: Bearer <token>` header. |
| **Role-restricted (roles)** | Authenticated, and the caller must hold one of the named roles on the relevant project. |

### 1.3 List Convention (Pagination, Filtering, Sorting)

All list endpoints share the same query-parameter convention. This is defined once and referenced everywhere.

**Query parameters:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `page` | integer | `1` | The page number (1-indexed). |
| `pageSize` | integer | `20` | Rows per page. Minimum `1`, maximum `100`. Values above 100 are clamped to 100. |
| `sort` | string | `createdAt:desc` | Sort field and direction, formatted as `field:asc` or `field:desc`. Allowed fields are documented per resource. |
| `status` | string | *(none)* | Filter by status (tasks only). One of: `todo`, `in_progress`, `done`. |
| `assigneeId` | integer | *(none)* | Filter by assignee (tasks only). |
| `tagId` | integer | *(none)* | Filter by tag (tasks only). |
| `priority` | integer | *(none)* | Filter by priority 1–5 (tasks only). |
| `search` | string | *(none)* | Case-insensitive substring match on the resource's `name` or `title`. |

**Paged response shape:**

Every list endpoint returns this wrapper:

```json
{
  "data": [ ... ],
  "meta": {
    "page": 1,
    "pageSize": 20,
    "total": 142,
    "totalPages": 8
  }
}
```

| Field | Type | Description |
|---|---|---|
| `data` | array | The page of resources. |
| `meta.page` | integer | Current page number. |
| `meta.pageSize` | integer | Rows per page (as requested, clamped to 100). |
| `meta.total` | integer | Total rows matching the filters. |
| `meta.totalPages` | integer | `ceil(total / pageSize)`. |

---

## 2. Error Contract

Every error response, regardless of the endpoint, uses the same shape:

```json
{
  "statusCode": 400,
  "message": "...",
  "error": "Bad Request",
  "timestamp": "2026-09-02T10:30:00.000Z",
  "path": "/api/v1/projects/1/tasks"
}
```

| Field | Type | Description |
|---|---|---|
| `statusCode` | integer | The HTTP status code. |
| `message` | string or string[] | A human-readable explanation. For validation failures (400), this is an array of field-level messages. |
| `error` | string | The HTTP status text (e.g. `"Bad Request"`, `"Unauthorized"`). |
| `timestamp` | string (ISO 8601) | When the error occurred. |
| `path` | string | The request path that produced the error. |

### Worked Examples

**400 Bad Request** — validation failure:

```json
{
  "statusCode": 400,
  "message": ["title must be a non-empty string", "priority must be an integer between 1 and 5"],
  "error": "Bad Request",
  "timestamp": "2026-09-02T10:31:00.000Z",
  "path": "/api/v1/projects/1/tasks"
}
```

**401 Unauthorized** — missing, expired, or invalid JWT:

```json
{
  "statusCode": 401,
  "message": "Access token is missing or expired",
  "error": "Unauthorized",
  "timestamp": "2026-09-02T10:32:00.000Z",
  "path": "/api/v1/projects/1/tasks"
}
```

**403 Forbidden** — authenticated but insufficient role:

```json
{
  "statusCode": 403,
  "message": "You need at least the 'member' role on this project",
  "error": "Forbidden",
  "timestamp": "2026-09-02T10:33:00.000Z",
  "path": "/api/v1/projects/1/tasks"
}
```

**404 Not Found** — resource does not exist:

```json
{
  "statusCode": 404,
  "message": "Project with id 999 not found",
  "error": "Not Found",
  "timestamp": "2026-09-02T10:34:00.000Z",
  "path": "/api/v1/projects/999/tasks"
}
```

---

## 3. Authentication Endpoints

Authentication is public (no JWT required). Phase 4 adds `password_hash` to `users` and a `refresh_tokens` table — these columns never appear in any API response.

---

### 3.1 Register

| | |
|---|---|
| **Method** | `POST` |
| **Path** | `/api/v1/auth/register` |
| **Access** | **Public** |

**Request body:**

```json
{
  "name": "Ayesha Khan",
  "email": "ayesha.khan@example.com",
  "password": "S3cureP@ss!"
}
```

| Field | Type | Rules |
|---|---|---|
| `name` | string | Required, 1–100 characters. |
| `email` | string | Required, valid email format, unique. |
| `password` | string | Required, minimum 8 characters, at least 1 uppercase, 1 lowercase, 1 digit. |

**Success — 201 Created:**

```json
{
  "id": 7,
  "name": "Ayesha Khan",
  "email": "ayesha.khan@example.com",
  "createdAt": "2026-09-02T10:30:00.000Z"
}
```

**Error codes:**

| Code | Cause |
|---|---|
| 400 | Validation failure (missing field, weak password). |
| 409 | Email already registered. |

---

### 3.2 Login

| | |
|---|---|
| **Method** | `POST` |
| **Path** | `/api/v1/auth/login` |
| **Access** | **Public** |

**Request body:**

```json
{
  "email": "ayesha.khan@example.com",
  "password": "S3cureP@ss!"
}
```

**Success — 200 OK:**

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...",
  "refreshToken": "d3f1a2b4-9c8e-4f7a-b123-abcdef012345",
  "expiresIn": 900
}
```

| Field | Type | Description |
|---|---|---|
| `accessToken` | string | JWT, signed, expires in 15 minutes. |
| `refreshToken` | string | Opaque token stored in `refresh_tokens` table, expires in 7 days. |
| `expiresIn` | integer | Access token lifetime in seconds. |

**Error codes:**

| Code | Cause |
|---|---|
| 400 | Validation failure (missing email or password). |
| 401 | Wrong email/password combination. |

> **Note:** A wrong password returns **401**, not 400 or 404. Returning 404 would confirm whether an email is registered; 401 reveals nothing.

---

### 3.3 Refresh

| | |
|---|---|
| **Method** | `POST` |
| **Path** | `/api/v1/auth/refresh` |
| **Access** | **Public** (the refresh token itself is the credential) |

**Request body:**

```json
{
  "refreshToken": "d3f1a2b4-9c8e-4f7a-b123-abcdef012345"
}
```

**Success — 200 OK:**

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIs...(new)...",
  "refreshToken": "e4g2b5c6-0d9f-5g8b-c234-bcdefg123456",
  "expiresIn": 900
}
```

The server revokes the old refresh token and issues a new pair (access + refresh). This is **refresh-token rotation** — a stolen token can only be used once.

**Error codes:**

| Code | Cause |
|---|---|
| 400 | Missing refresh token in body. |
| 401 | Refresh token is invalid, expired, or already revoked. |

---

### 3.4 Logout

| | |
|---|---|
| **Method** | `POST` |
| **Path** | `/api/v1/auth/logout` |
| **Access** | **Authenticated** |

**Request body:**

```json
{
  "refreshToken": "d3f1a2b4-9c8e-4f7a-b123-abcdef012345"
}
```

**Success — 204 No Content** (empty body).

The server deletes (revokes) the refresh token from the `refresh_tokens` table. The access token remains valid until it expires naturally (up to 15 minutes), because JWTs are stateless — the server does not maintain an access-token blacklist.

**Error codes:**

| Code | Cause |
|---|---|
| 400 | Missing refresh token in body. |
| 401 | Access token missing or expired. |

---

## 4. Tasks Resource

Tasks are nested under projects: a task always belongs to exactly one project.

**Sortable fields:** `createdAt`, `priority`, `dueDate`, `title`, `status`.

---

### 4.1 Create Task

| | |
|---|---|
| **Method** | `POST` |
| **Path** | `/api/v1/projects/:projectId/tasks` |
| **Access** | **Role-restricted** — `owner`, `admin`, or `member` on the project |

**Request body:**

```json
{
  "title": "Upgrade PostgreSQL to 16",
  "description": "Past its due date and still open.",
  "status": "todo",
  "priority": 5,
  "assigneeId": 4,
  "dueDate": "2026-09-10",
  "tagIds": [5, 3, 6]
}
```

| Field | Type | Rules |
|---|---|---|
| `title` | string | Required, 1–255 characters. |
| `description` | string \| null | Optional. |
| `status` | string | Optional, default `"todo"`. One of `todo`, `in_progress`, `done`. |
| `priority` | integer | Required, 1–5. |
| `assigneeId` | integer \| null | Optional. Must be a member of the same project. |
| `dueDate` | string (ISO date) \| null | Optional. |
| `tagIds` | integer[] | Optional. Array of existing tag ids. |

**Success — 201 Created:**

```json
{
  "id": 16,
  "title": "Upgrade PostgreSQL to 16",
  "description": "Past its due date and still open.",
  "status": "todo",
  "priority": 5,
  "projectId": 1,
  "assigneeId": 4,
  "dueDate": "2026-09-10",
  "tags": [
    { "id": 5, "name": "urgent" },
    { "id": 3, "name": "backend" },
    { "id": 6, "name": "tech-debt" }
  ],
  "createdAt": "2026-09-02T10:30:00.000Z"
}
```

**Headers:** `Location: /api/v1/projects/1/tasks/16`

**Error codes:**

| Code | Cause |
|---|---|
| 400 | Validation failure, or `assigneeId` is not a project member, or a `tagId` does not exist. |
| 401 | Missing or invalid JWT. |
| 403 | Caller's role is `viewer` (not member/admin/owner). |
| 404 | Project not found. |

---

### 4.2 List Tasks

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/api/v1/projects/:projectId/tasks` |
| **Access** | **Role-restricted** — any project member (`owner`, `admin`, `member`, `viewer`) |

**Query parameters:** see [List Convention](#13-list-convention-pagination-filtering-sorting). Supports `status`, `assigneeId`, `tagId`, `priority`, `search` (on title), and `sort`.

**Success — 200 OK:** paged response (see convention). Each item in `data` has the same shape as the create response.

**Error codes:**

| Code | Cause |
|---|---|
| 401 | Missing or invalid JWT. |
| 403 | Caller is not a member of this project. |
| 404 | Project not found. |

---

### 4.3 Get Task

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/api/v1/projects/:projectId/tasks/:taskId` |
| **Access** | **Role-restricted** — any project member |

**Success — 200 OK:**

Same shape as the create response, plus nested `assignee` and `project` objects:

```json
{
  "id": 16,
  "title": "Upgrade PostgreSQL to 16",
  "description": "Past its due date and still open.",
  "status": "todo",
  "priority": 5,
  "projectId": 1,
  "project": { "id": 1, "name": "Apollo Web Platform" },
  "assigneeId": 4,
  "assignee": { "id": 4, "name": "Daniyal Raza", "email": "daniyal.raza@example.com" },
  "dueDate": "2026-09-10",
  "tags": [
    { "id": 5, "name": "urgent" },
    { "id": 3, "name": "backend" },
    { "id": 6, "name": "tech-debt" }
  ],
  "createdAt": "2026-09-02T10:30:00.000Z"
}
```

**Error codes:**

| Code | Cause |
|---|---|
| 401 | Missing or invalid JWT. |
| 403 | Caller is not a member of this project. |
| 404 | Project or task not found. |

---

### 4.4 Update Task (Partial)

| | |
|---|---|
| **Method** | `PATCH` |
| **Path** | `/api/v1/projects/:projectId/tasks/:taskId` |
| **Access** | **Role-restricted** — `owner`, `admin`, or `member` on the project |

**Request body** (all fields optional — only provided fields are updated):

```json
{
  "title": "Upgrade PostgreSQL to 17",
  "status": "in_progress",
  "priority": 4,
  "assigneeId": 2,
  "dueDate": "2026-10-01",
  "tagIds": [5, 3]
}
```

**Success — 200 OK:** returns the full updated task (same shape as Get Task).

**Error codes:**

| Code | Cause |
|---|---|
| 400 | Validation failure, or `assigneeId` not a project member, or a `tagId` not found. |
| 401 | Missing or invalid JWT. |
| 403 | Caller's role is `viewer`. |
| 404 | Project or task not found. |

---

### 4.5 Delete Task

| | |
|---|---|
| **Method** | `DELETE` |
| **Path** | `/api/v1/projects/:projectId/tasks/:taskId` |
| **Access** | **Role-restricted** — `owner` or `admin` on the project |

**Success — 204 No Content** (empty body).

Deleting a task cascades to its `task_tags` rows and `comments` (via `ON DELETE CASCADE`).

**Error codes:**

| Code | Cause |
|---|---|
| 401 | Missing or invalid JWT. |
| 403 | Caller's role is `member` or `viewer` (must be admin or owner). |
| 404 | Project or task not found. |

---

## 5. Projects Resource

---

### 5.1 Create Project

| | |
|---|---|
| **Method** | `POST` |
| **Path** | `/api/v1/projects` |
| **Access** | **Authenticated** |

**Request body:**

```json
{
  "name": "Apollo Web Platform"
}
```

| Field | Type | Rules |
|---|---|---|
| `name` | string | Required, 1–255 characters. |

**Success — 201 Created:**

```json
{
  "id": 1,
  "name": "Apollo Web Platform",
  "ownerId": 1,
  "createdAt": "2026-09-02T10:30:00.000Z"
}
```

The authenticated user is automatically set as the `owner` and added to `project_members` with role `owner`.

**Headers:** `Location: /api/v1/projects/1`

**Error codes:**

| Code | Cause |
|---|---|
| 400 | Validation failure (missing or empty name). |
| 401 | Missing or invalid JWT. |

---

### 5.2 List Projects

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/api/v1/projects` |
| **Access** | **Authenticated** |

Returns only the projects the authenticated user is a member of.

**Query parameters:** see [List Convention](#13-list-convention-pagination-filtering-sorting). Supports `search` (on name) and `sort` (fields: `createdAt`, `name`).

**Success — 200 OK:** paged response.

**Error codes:**

| Code | Cause |
|---|---|
| 401 | Missing or invalid JWT. |

---

### 5.3 Get Project

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/api/v1/projects/:projectId` |
| **Access** | **Role-restricted** — any project member |

**Success — 200 OK:**

```json
{
  "id": 1,
  "name": "Apollo Web Platform",
  "ownerId": 1,
  "owner": { "id": 1, "name": "Ayesha Khan", "email": "ayesha.khan@example.com" },
  "createdAt": "2026-09-02T10:30:00.000Z"
}
```

**Error codes:**

| Code | Cause |
|---|---|
| 401 | Missing or invalid JWT. |
| 403 | Caller is not a member of this project. |
| 404 | Project not found. |

---

### 5.4 Update Project

| | |
|---|---|
| **Method** | `PATCH` |
| **Path** | `/api/v1/projects/:projectId` |
| **Access** | **Role-restricted** — `owner` or `admin` |

**Request body:**

```json
{
  "name": "Apollo Platform v2"
}
```

**Success — 200 OK:** returns the full updated project.

**Error codes:**

| Code | Cause |
|---|---|
| 400 | Validation failure. |
| 401 | Missing or invalid JWT. |
| 403 | Caller's role is `member` or `viewer`. |
| 404 | Project not found. |

---

### 5.5 Delete Project

| | |
|---|---|
| **Method** | `DELETE` |
| **Path** | `/api/v1/projects/:projectId` |
| **Access** | **Role-restricted** — `owner` only |

**Success — 204 No Content** (empty body).

Deleting a project cascades to `project_members`, `tasks` (and transitively to `task_tags` and `comments`).

**Error codes:**

| Code | Cause |
|---|---|
| 401 | Missing or invalid JWT. |
| 403 | Caller is not the project owner. |
| 404 | Project not found. |

---

## 6. Users Resource

User management is limited: users are created via `/auth/register`. The API provides read-only access and a self-update endpoint.

---

### 6.1 List Users

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/api/v1/users` |
| **Access** | **Authenticated** |

Returns a paginated list of all users (used for assignee pickers, mention autocomplete).

**Query parameters:** see [List Convention](#13-list-convention-pagination-filtering-sorting). Supports `search` (on name or email) and `sort` (fields: `createdAt`, `name`).

**Success — 200 OK:** paged response. Each item:

```json
{
  "id": 1,
  "name": "Ayesha Khan",
  "email": "ayesha.khan@example.com",
  "createdAt": "2026-09-02T10:30:00.000Z"
}
```

**Error codes:**

| Code | Cause |
|---|---|
| 401 | Missing or invalid JWT. |

---

### 6.2 Get User

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/api/v1/users/:userId` |
| **Access** | **Authenticated** |

**Success — 200 OK:** same shape as the list item above.

**Error codes:**

| Code | Cause |
|---|---|
| 401 | Missing or invalid JWT. |
| 404 | User not found. |

---

### 6.3 Update User (Self Only)

| | |
|---|---|
| **Method** | `PATCH` |
| **Path** | `/api/v1/users/:userId` |
| **Access** | **Authenticated** — the caller's id must match `:userId` |

**Request body:**

```json
{
  "name": "Ayesha K."
}
```

| Field | Type | Rules |
|---|---|---|
| `name` | string | Optional, 1–100 characters. |
| `email` | string | Optional, valid email format, must remain unique. |

**Success — 200 OK:** returns the updated user.

**Error codes:**

| Code | Cause |
|---|---|
| 400 | Validation failure. |
| 401 | Missing or invalid JWT. |
| 403 | Caller is trying to update another user's profile. |
| 409 | New email already taken. |

---

### 6.4 Delete User

| | |
|---|---|
| **Method** | `DELETE` |
| **Path** | `/api/v1/users/:userId` |
| **Access** | **Authenticated** — the caller's id must match `:userId` |

**Success — 204 No Content** (empty body).

Fails if the user owns projects (`ON DELETE RESTRICT` on `projects.owner_id`) or has authored comments (`ON DELETE RESTRICT` on `comments.author_id`). The user must transfer ownership and delete their comments first.

**Error codes:**

| Code | Cause |
|---|---|
| 401 | Missing or invalid JWT. |
| 403 | Caller is trying to delete another user. |
| 409 | User still owns projects or has authored comments (FK RESTRICT). |

---

## 7. Project Members Resource

Project members are nested under projects. A member is identified by the composite key `(userId, projectId)`.

---

### 7.1 Add Member

| | |
|---|---|
| **Method** | `POST` |
| **Path** | `/api/v1/projects/:projectId/members` |
| **Access** | **Role-restricted** — `owner` or `admin` on the project |

**Request body:**

```json
{
  "userId": 5,
  "role": "member"
}
```

| Field | Type | Rules |
|---|---|---|
| `userId` | integer | Required. Must be an existing user. |
| `role` | string | Required. One of `owner`, `admin`, `member`, `viewer`. |

**Success — 201 Created:**

```json
{
  "userId": 5,
  "projectId": 1,
  "role": "member",
  "user": { "id": 5, "name": "Hina Saeed", "email": "hina.saeed@example.com" }
}
```

**Error codes:**

| Code | Cause |
|---|---|
| 400 | Validation failure (invalid role, missing userId). |
| 401 | Missing or invalid JWT. |
| 403 | Caller's role is `member` or `viewer`. |
| 404 | Project or user not found. |
| 409 | User is already a member of this project. |

---

### 7.2 List Members

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/api/v1/projects/:projectId/members` |
| **Access** | **Role-restricted** — any project member |

**Query parameters:** see [List Convention](#13-list-convention-pagination-filtering-sorting). Supports `sort` (fields: `role`, `userId`). Filtering by `role` is supported via `?role=admin`.

**Success — 200 OK:** paged response. Each item:

```json
{
  "userId": 5,
  "projectId": 1,
  "role": "member",
  "user": { "id": 5, "name": "Hina Saeed", "email": "hina.saeed@example.com" }
}
```

**Error codes:**

| Code | Cause |
|---|---|
| 401 | Missing or invalid JWT. |
| 403 | Caller is not a member of this project. |
| 404 | Project not found. |

---

### 7.3 Update Member Role

| | |
|---|---|
| **Method** | `PATCH` |
| **Path** | `/api/v1/projects/:projectId/members/:userId` |
| **Access** | **Role-restricted** — `owner` or `admin` (admins cannot promote to owner) |

**Request body:**

```json
{
  "role": "admin"
}
```

**Success — 200 OK:** returns the updated membership.

**Error codes:**

| Code | Cause |
|---|---|
| 400 | Invalid role value. |
| 401 | Missing or invalid JWT. |
| 403 | Insufficient role, or an admin tried to set `role: "owner"`. |
| 404 | Project not found, or user is not a member. |

---

### 7.4 Remove Member

| | |
|---|---|
| **Method** | `DELETE` |
| **Path** | `/api/v1/projects/:projectId/members/:userId` |
| **Access** | **Role-restricted** — `owner` or `admin` (the project owner cannot be removed) |

**Success — 204 No Content** (empty body).

Tasks assigned to the removed user have their `assignee_id` set to `NULL` (via `ON DELETE SET NULL` on the FK, or handled by the service).

**Error codes:**

| Code | Cause |
|---|---|
| 401 | Missing or invalid JWT. |
| 403 | Caller's role is insufficient, or trying to remove the project owner. |
| 404 | Project not found, or user is not a member. |

---

## 8. Tags Resource

Tags are global — they are not scoped to a project.

---

### 8.1 Create Tag

| | |
|---|---|
| **Method** | `POST` |
| **Path** | `/api/v1/tags` |
| **Access** | **Authenticated** |

**Request body:**

```json
{
  "name": "urgent"
}
```

| Field | Type | Rules |
|---|---|---|
| `name` | string | Required, 1–50 characters, unique (case-insensitive). |

**Success — 201 Created:**

```json
{
  "id": 7,
  "name": "urgent"
}
```

**Headers:** `Location: /api/v1/tags/7`

**Error codes:**

| Code | Cause |
|---|---|
| 400 | Validation failure (missing or empty name). |
| 401 | Missing or invalid JWT. |
| 409 | A tag with this name already exists. |

---

### 8.2 List Tags

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/api/v1/tags` |
| **Access** | **Authenticated** |

**Query parameters:** see [List Convention](#13-list-convention-pagination-filtering-sorting). Supports `search` (on name) and `sort` (fields: `name`, `id`).

**Success — 200 OK:** paged response.

**Error codes:**

| Code | Cause |
|---|---|
| 401 | Missing or invalid JWT. |

---

### 8.3 Get Tag

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/api/v1/tags/:tagId` |
| **Access** | **Authenticated** |

**Success — 200 OK:**

```json
{
  "id": 5,
  "name": "urgent"
}
```

**Error codes:**

| Code | Cause |
|---|---|
| 401 | Missing or invalid JWT. |
| 404 | Tag not found. |

---

### 8.4 Update Tag

| | |
|---|---|
| **Method** | `PATCH` |
| **Path** | `/api/v1/tags/:tagId` |
| **Access** | **Authenticated** |

**Request body:**

```json
{
  "name": "critical"
}
```

**Success — 200 OK:** returns the updated tag.

**Error codes:**

| Code | Cause |
|---|---|
| 400 | Validation failure. |
| 401 | Missing or invalid JWT. |
| 404 | Tag not found. |
| 409 | Another tag with this name already exists. |

---

### 8.5 Delete Tag

| | |
|---|---|
| **Method** | `DELETE` |
| **Path** | `/api/v1/tags/:tagId` |
| **Access** | **Authenticated** |

**Success — 204 No Content** (empty body).

Deleting a tag cascades to `task_tags` rows (via `ON DELETE CASCADE`). Tasks themselves are unaffected.

**Error codes:**

| Code | Cause |
|---|---|
| 401 | Missing or invalid JWT. |
| 404 | Tag not found. |

---

## 9. Comments Resource

Comments are nested under tasks, which are nested under projects.

---

### 9.1 Create Comment

| | |
|---|---|
| **Method** | `POST` |
| **Path** | `/api/v1/projects/:projectId/tasks/:taskId/comments` |
| **Access** | **Role-restricted** — any project member (`owner`, `admin`, `member`, `viewer`) |

**Request body:**

```json
{
  "body": "Blocked on the maintenance window."
}
```

| Field | Type | Rules |
|---|---|---|
| `body` | string | Required, 1–5000 characters. |

**Success — 201 Created:**

```json
{
  "id": 11,
  "taskId": 8,
  "authorId": 1,
  "author": { "id": 1, "name": "Ayesha Khan", "email": "ayesha.khan@example.com" },
  "body": "Blocked on the maintenance window.",
  "createdAt": "2026-09-02T11:00:00.000Z"
}
```

**Headers:** `Location: /api/v1/projects/1/tasks/8/comments/11`

**Error codes:**

| Code | Cause |
|---|---|
| 400 | Validation failure (missing or empty body). |
| 401 | Missing or invalid JWT. |
| 403 | Caller is not a member of the project. |
| 404 | Project or task not found. |

---

### 9.2 List Comments

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/api/v1/projects/:projectId/tasks/:taskId/comments` |
| **Access** | **Role-restricted** — any project member |

**Query parameters:** see [List Convention](#13-list-convention-pagination-filtering-sorting). Default sort is `createdAt:asc` (chronological thread). Supports `sort` (fields: `createdAt`).

**Success — 200 OK:** paged response.

**Error codes:**

| Code | Cause |
|---|---|
| 401 | Missing or invalid JWT. |
| 403 | Caller is not a member of the project. |
| 404 | Project or task not found. |

---

### 9.3 Get Comment

| | |
|---|---|
| **Method** | `GET` |
| **Path** | `/api/v1/projects/:projectId/tasks/:taskId/comments/:commentId` |
| **Access** | **Role-restricted** — any project member |

**Success — 200 OK:** same shape as the create response.

**Error codes:**

| Code | Cause |
|---|---|
| 401 | Missing or invalid JWT. |
| 403 | Caller is not a member of the project. |
| 404 | Project, task, or comment not found. |

---

### 9.4 Update Comment

| | |
|---|---|
| **Method** | `PATCH` |
| **Path** | `/api/v1/projects/:projectId/tasks/:taskId/comments/:commentId` |
| **Access** | **Authenticated** — the caller must be the comment's author |

**Request body:**

```json
{
  "body": "Blocked on the maintenance window. Ops confirmed Saturday 02:00."
}
```

**Success — 200 OK:** returns the updated comment.

**Error codes:**

| Code | Cause |
|---|---|
| 400 | Validation failure. |
| 401 | Missing or invalid JWT. |
| 403 | Caller is not the author of this comment. |
| 404 | Project, task, or comment not found. |

---

### 9.5 Delete Comment

| | |
|---|---|
| **Method** | `DELETE` |
| **Path** | `/api/v1/projects/:projectId/tasks/:taskId/comments/:commentId` |
| **Access** | **Role-restricted** — comment author, or project `owner`/`admin` |

**Success — 204 No Content** (empty body).

**Error codes:**

| Code | Cause |
|---|---|
| 401 | Missing or invalid JWT. |
| 403 | Caller is neither the author nor a project owner/admin. |
| 404 | Project, task, or comment not found. |

---

## 10. Idempotent Create

### `POST /api/v1/projects/:projectId/tasks` with `Idempotency-Key`

To prevent duplicate task creation on network retries, the client may include an `Idempotency-Key` header:

```
POST /api/v1/projects/1/tasks
Authorization: Bearer <token>
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{ "title": "Upgrade PostgreSQL to 16", ... }
```

**How it works:**

1. On the first request, the server creates the task normally, stores the `Idempotency-Key` alongside the response in a dedicated `idempotency_keys` table:

   | Column | Value |
   |---|---|
   | `key` | `550e8400-e29b-41d4-a716-446655440000` |
   | `user_id` | 1 |
   | `status_code` | 201 |
   | `response_body` | `{"id": 16, "title": "Upgrade PostgreSQL to 16", ...}` |
   | `created_at` | `2026-09-02T10:30:00.000Z` |
   | `expires_at` | `2026-09-03T10:30:00.000Z` |

2. On a retry with the **same key and same user**, the server returns the stored response with the stored status code. No second task is created. The response includes the header `Idempotent-Replayed: true`.

3. Keys expire after **24 hours**. After expiry, the same key can be reused and will produce a new resource.

4. If a different user sends the same key, it is treated as a new key (keys are scoped to `user_id`).

5. If the idempotency key is provided but the request body differs from the original, the server returns **422 Unprocessable Entity** with the message `"Idempotency-Key reused with a different request body"`.

---

## 11. Full Status-Code Table — Tasks

The table below covers every HTTP status code the tasks resource can return, including `409`.

| Code | Status Text | Cause |
|---|---|---|
| 200 | OK | Successful GET (single task or list) or PATCH. |
| 201 | Created | Successful POST — task created. |
| 204 | No Content | Successful DELETE — task removed. |
| 400 | Bad Request | Validation failure: missing `title`, `priority` out of range, invalid `status` value, `assigneeId` not a project member, `tagId` does not exist, malformed JSON. |
| 401 | Unauthorized | JWT missing, expired, or signature invalid. |
| 403 | Forbidden | The caller is authenticated but their project role is insufficient for this action (e.g. a `viewer` attempting to create a task). |
| 404 | Not Found | The project or task does not exist — or the caller is not a member of the project (see note below). |
| 409 | Conflict | A duplicate resource conflict: for example, attempting to create a task with an `Idempotency-Key` that was already used with a different request body (422), or — more broadly on other resources — a duplicate email on register, a duplicate tag name, or a duplicate `(user_id, project_id)` membership. For tasks specifically, 409 arises if a concurrent update produces a conflict (e.g. optimistic-locking version mismatch). |
| 422 | Unprocessable Entity | The JSON is well-formed but semantically invalid (e.g. `Idempotency-Key` reused with a different body). |
| 500 | Internal Server Error | Unexpected server failure. |

### When to return 404 instead of 403

When a caller requests a task inside a project they do not belong to, the server has two defensible responses:

- **403 Forbidden** — "you are not allowed to see this." This is honest but **leaks information**: it confirms that the project exists. An attacker can enumerate project ids by watching for 403 vs. 404.
- **404 Not Found** — "this project does not exist." This **leaks less**: the attacker cannot distinguish "I am not a member" from "this id is unused." The cost is a slightly confusing error for a legitimate user who mis-typed a project id they do have access to.

**Decision:** return **404** when the caller is not a member of the project, for both the project and any resources nested under it. This follows the principle of least information: do not confirm the existence of a resource the caller has no right to see. We use 403 only when the caller is a confirmed member whose role is too low for the specific action (e.g. a `viewer` trying to create a task).

---

*Document prepared for the CMIT Internship Program, Week 7, Assignment 2.*
