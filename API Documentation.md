# BlockCAD — API Documentation (MVP)

**API Style:** REST  
**Base Path:** `/api`  
**Data Format:** JSON, except file exports  
**Document Status:** Initial draft 

## 1. Overview

BlockCAD is a 3D modeling application that allows users to create 3D shapes, modify their geometry and appearance, save and load projects, search projects, and export models to STL format.

This document defines the proposed API contract for the BlockCAD Minimum Viable Product (MVP). It describes the planned endpoints, authentication requirements, request and response formats, validation rules, and expected HTTP status codes.

### 1.1 MVP Scope

The API will support the following MVP features:
| MVP feature | Main endpoints | Section |
| :--- | :--- | :---: |
| User registration, login and logout | `/auth/*` | 2 |
| Adding basic 3D shapes to a project | `POST /projects/:projectId/shapes` |4 |
| Moving, rotating, and resizing shapes | `PATCH /shapes/:shapeId` | 4 |
| Changing shape colors | `PATCH /shapes/:shapeId` | 4 |
| Saving and loading projects | `/projects`, `PUT /projects/:projectId/shapes` | 3,5 |
| Searching saved projects | `GET /projects?search=` | 3.2 |
| Exporting projects to STL | `GET /projects/:projectId/export` | 6 |

### 1.2 General conventions

- Endpoints use the `/api` base path.
- JSON is used for request and response bodies unless otherwise specified.
- Protected endpoints require authentication.
- The examples use illustrative IDs and timestamps.
- The final documentation must reflect the actual response fields and status codes returned by the backend.
- Each endpoint documents its purpose, authentication requirements, request format, successful response, and expected errors.

### 1.3 Error format

All errors use the same JSON structure:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "The request contains invalid geometry."
  }
}
```

- `code`: stable, machine-readable string. Clients should rely on it, not on `message`.
- `message`: human-readable description. For validation errors it names the invalid field. It may change and must not be parsed.

### 1.4 Common status codes

| Status | Meaning |
| :--- | :--- |
| 200 OK | Request succeeded and a body is returned |
| 201 Created | Resource created; the `Location` header holds its URL |
| 204 No Content | Request succeeded without a body |
| 400 Bad Request | Malformed JSON or missing body |
| 401 Unauthorized | Authentication is missing, expired or invalid |
| 403 Forbidden | Request refused, for example a missing CSRF header |
| 404 Not Found | Resource does not exist or is not visible to the user |
| 409 Conflict | The request conflicts with the current state |
| 413 Payload Too Large | Request body exceeds 1 MB |
| 415 Unsupported Media Type | `Content-Type` is not `application/json` |
| 422 Unprocessable Entity | Well-formed JSON that fails validation rules |
| 429 Too Many Requests | Rate limit exceeded; see the `Retry-After` header |
| 500 Internal Server Error | Unexpected server error |

### 1.5 Error codes

| Code | Status | Description |
| :--- | :---: | :--- |
| `INVALID_REQUEST` | 400 | Malformed JSON or missing body |
| `UNAUTHENTICATED` | 401 | No valid session |
| `INVALID_CREDENTIALS` | 401 | Email or password is incorrect |
| `FORBIDDEN` | 403 | Request refused, for example a missing CSRF header |
| `NOT_FOUND` | 404 | Resource not found or not visible to the user |
| `EMAIL_ALREADY_REGISTERED` | 409 | Email is already in use |
| `SHAPE_LIMIT_REACHED` | 409 | The project already holds the maximum number of shapes (500) |
| `PAYLOAD_TOO_LARGE` | 413 | Request body exceeds 1 MB |
| `UNSUPPORTED_MEDIA_TYPE` | 415 | `Content-Type` is not `application/json` |
| `VALIDATION_ERROR` | 422 | One or more fields failed validation |
| `NOT_EXPORTABLE` | 422 | The project cannot be exported (see section 6) |
| `RATE_LIMITED` | 429 | Too many attempts |
| `INTERNAL_ERROR` | 500 | Unexpected error |

### 1.6 Pagination and sorting

Collection endpoints that can grow large (`GET /projects`) accept these query parameters. Shape lists are not paginated, because a project holds at most 500 shapes.

| Parameter | Type | Default | Rules |
| :--- | :--- | :--- | :--- |
| `limit` | integer | 20 | 1–100 |
| `offset` | integer | 0 | 0 or greater |
| `sort` | string | `-updatedAt` | `name`, `createdAt` or `updatedAt`; a leading `-` means descending |

Paginated responses include a `pagination` object, for example `{ "total": 42, "limit": 20, "offset": 0 }`. Invalid values return `422 VALIDATION_ERROR`.

## 2. Authentication

Authentication allows users to register, log in, log out, and retrieve their current account information.

**Authentication mechanism:** The final implementation must specify whether authentication uses secure cookies or bearer tokens and document how the frontend sends credentials.

| Method | Endpoint | Purpose | Authentication |
|---|---|---|---|
| POST | `/api/auth/register` | Register a new user | No |
| POST | `/api/auth/login` | Log in a user | No |
| POST | `/api/auth/logout` | Log out the current user | Yes |
| GET | `/api/auth/user` | Get current user information | Yes |

These routes are proposed and must be checked against the actual implementation.

### 2.1 Register a User

**Endpoint:** `POST /api/auth/register`

**Purpose:** Create a new user account. The user is **not** logged in automatically; the client must call the login endpoint afterwards.

**Authentication:** Not required.

**Request body:**

| Field | Type | Required | Rules |
| :--- | :--- | :---: | :--- |
| `name` | string | Yes | 1–50 characters after trimming |
| `email` | string | Yes | Valid email, maximum 254 characters |
| `password` | string | Yes | 8–128 characters |

```json
{
  "name": "Alex",
  "email": "alex@example.com",
  "password": "ExamplePassword123!"
}
```

**Example successful response — `201 Created`:**

```json
{
  "id": "user_123",
  "name": "Alex",
  "email": "alex@example.com"
}
```

**Possible errors:**

| Status | Code | Reason |
| :---: | :--- | :--- |
| 400 | `INVALID_REQUEST` | Body is missing or is not valid JSON |
| 409 | `EMAIL_ALREADY_REGISTERED` | Email is already in use |
| 422 | `VALIDATION_ERROR` | A field is missing or breaks a rule above |
| 429 | `RATE_LIMITED` | Too many registration attempts |
| 500 | `INTERNAL_ERROR` | Unexpected server error |

- `400 Bad Request` — required fields are missing or invalid.
- `409 Conflict` — the email address is already registered.
- `500 Internal Server Error` — unexpected server error.

The actual registration fields, validation rules, and response must be confirmed against the implementation.

### 2.2 Log In

**Endpoint:** `POST /api/auth/login`

**Purpose:** Authenticate a user and establish a session.

**Authentication:** Not required before login. If the client already has a valid session, the old session is replaced by a new one.

**Request body:**

| Field | Type | Required |
| :--- | :--- | :---: |
| `email` | string | Yes |
| `password` | string | Yes |

```json
{
  "email": "alex@example.com",
  "password": "ExamplePassword123!"
}
```

**Example successful response — `200 OK`:**

```json
{
  "user": {
    "id": "user_123",
    "name": "Alex",
    "email": "alex@example.com"
  }
}
```

The example assumes authentication is established through a cookie or another session mechanism. If bearer tokens are used, the actual response must document the token fields.

**Possible errors:**

| Status | Code | Reason |
| :---: | :--- | :--- |
| 400 | `INVALID_REQUEST` | Body is missing or is not valid JSON |
| 401 | `INVALID_CREDENTIALS` | Email or password is incorrect.|
| 422 | `VALIDATION_ERROR` | `email` or `password` is missing or empty |
| 429 | `RATE_LIMITED` | Too many failed attempts; use the `Retry-After` header |
| 500 | `INTERNAL_ERROR` | Unexpected server error |

- `400 Bad Request` — invalid request fields.
- `401 Unauthorized` — credentials are incorrect.
- `500 Internal Server Error` — unexpected server error.

### 2.3 Log Out

**Endpoint:** `POST /api/auth/logout`

**Purpose:** End the current user's session.

**Authentication:** Required.

**Request:** No body required in this proposed design.

**Example successful response — `204 No Content`:**

No response body.

**Possible errors:**

- `401 Unauthorized` — authentication is missing or invalid.
- `500 Internal Server Error` — unexpected server error.

The response status and session-clearing behavior must match the implemented authentication mechanism.

### 2.4 Get Current User

**Endpoint:** `GET /api/auth/user`

**Purpose:** Retrieve information about the authenticated user. The frontend can call this on page load to check whether a session is still valid.

**Authentication:** Required.

**Example request:**

```http
GET /api/auth/user
```

Credentials must be sent using the authentication mechanism selected.

**Example successful response — `200 OK`:**

```json
{
  "id": "user_123",
  "name": "Alex",
  "email": "alex@example.com"
}
```

**Possible errors:**

- `401 Unauthorized` — authentication is missing or invalid.
- `500 Internal Server Error` — unexpected server error.

## 3. Projects

A project represents a saved 3D model and contains its shapes. A project is owned by one user. It contains a collection of shapes (section 4). All project endpoints require authentication, and a user can only reach their own projects.

### Project endpoints

| Method | Endpoint | Purpose | Authentication |
|---|---|---|---|
| POST | `/api/projects` | Create a project | Required |
| GET | `/api/projects` | List projects, optionally filtered by name | Required |
| GET | `/api/projects/:projectId` | Retrieve a project and its shapes | Required |
| PATCH | `/api/projects/:projectId` | Update project details | Required |
| DELETE | `/api/projects/:projectId` | Delete a project | Required |

These endpoints are proposed and must be verified against the backend.

### Project object

| Field | Type | Description |
| :--- | :--- | :--- |
| `id` | string | Unique project ID |
| `name` | string | 1–100 characters after trimming |
| `createdAt` | string | Creation time (ISO 8601, UTC) |
| `updatedAt` | string | Last change to the project or any of its shapes |
| `shapes` | array | Shape objects. Returned only by the retrieve endpoint. |

### 3.1 Create a Project

**Endpoint:** `POST /api/projects`

**Purpose:** Create and save a new project.

**Authentication:** Required.

**Request body:**

```json
{
  "name": "First Model"
}
```

**Example successful response — `201 Created`:**

```json
{
  "id": "project_123",
  "name": "First Model",
  "createdAt": "2026-10-12T18:00:00Z",
  "updatedAt": "2026-10-12T18:00:00Z"
}
```

**Possible errors:**

- `400 Bad Request` — project name is missing or invalid.
- `401 Unauthorized` — authentication is missing or invalid.
- `422 VALIDATION_ERROR - name is missing, empty or longer than 100 characters
- `500 Internal Server Error` — unexpected server error.

The project must be associated with the authenticated user on the server.

### 3.2 List Projects and Search

**Endpoint:** `GET /api/projects`

**Purpose:** Retrieve the authenticated user's projects, optionally filtered by name.

**Authentication:** Required.

**Query parameters:**

- `search` (optional string) — filters projects by name.
- If omitted, the endpoint returns the user's projects without a search filter.

**Example request — list all projects:**

```http
GET /api/projects
```

**Example request — search projects:**

```http
GET /api/projects?search=First
```

**Example successful response — `200 OK`:**

```json
{
  "projects": [
    {
      "id": "project_123",
      "name": "First Model",
      "createdAt": "2026-10-12T18:00:00Z",
      "updatedAt": "2026-10-12T18:00:00Z"
    }
  ]
}
```

**Possible errors:**

- `401 Unauthorized` — authentication is missing or invalid.
- `500 Internal Server Error` — unexpected server error.

The example response is illustrative. The actual response structure and search behavior should be discussed.

### 3.3 Retrieve a Project

**Endpoint:** `GET /api/projects/:projectId`

**Purpose:** Load a saved project and its associated shapes.

**Authentication:** Required.

**Path parameters:**

- `projectId` — ID of the project to retrieve.

**Example request:**

```http
GET /api/projects/project_123
```

**Example successful response — `200 OK`:**

```json
{
  "id": "project_123",
  "name": "First Model",
  "shapes": [
    {
      "id": "shape_456",
      "name": "Custom Sphere",
      "geometry": {
        "type": "sphere",
        "radius": 1,
        "segments": 16,
        "rings": 8
      },
      "color": "#FF0000",
      "position": [0, 0, 0],
      "rotation": [0, 0, 0],
      "scale": [1, 1, 1]
    }
  ]
}
```

**Possible errors:**

- `401 Unauthorized` — authentication is missing or invalid.
- `404 Not Found` — the project does not exist or is not accessible to the user.
- `500 Internal Server Error` — unexpected server error.

This example assumes the project endpoint returns its shapes in the same response. 

### 3.4 Update a Project

**Endpoint:** `PATCH /api/projects/:projectId`

**Purpose:** Update project information, such as its name.

**Authentication:** Required.

**Path parameters:**

- `projectId` — ID of the project to update.

**Request body:**

```json
{
  "name": "Updated Model"
}
```

Only the supplied fields are updated.

**Example successful response — `200 OK`:**

```json
{
  "id": "project_123",
  "name": "Updated Model",
  "updatedAt": "2026-10-12T18:30:00Z"
}
```

**Possible errors:**

- `400 Bad Request` — invalid project data.
- `401 Unauthorized` — authentication is missing or invalid.
- `404 Not Found` — the project does not exist or is not accessible to the user.
- `500 Internal Server Error` — unexpected server error.

### 3.5 Delete a Project

**Endpoint:** `DELETE /api/projects/:projectId`

**Purpose:** Delete a project and its associated data.

**Authentication:** Required.

**Path parameters:**

- `projectId` — ID of the project to delete.

**Example request:**

```http
DELETE /api/projects/project_123
```

**Example successful response — `204 No Content`:**

No response body.

**Possible errors:**

- `401 Unauthorized` — authentication is missing or invalid.
- `404 Not Found` — the project does not exist or is not accessible to the user.
- `500 Internal Server Error` — unexpected server error.

The backend must define whether deleting a project also deletes its shapes and any related files.

## 4. Shapes and Geometry

Every shape belongs to one project. All shapes share the same properties, and the geometry object changes with the shape type. All shape endpoints require authentication and check that the parent project belongs to the user.

### Shape object

| Field | Type | Description |
| :--- | :--- | :--- |
| `id` | string | Unique shape ID (read-only) |
| `projectId` | string | Parent project ID (read-only) |
| `name` | string | 1–100 characters. Defaults to the type name, for example `Sphere`. |
| `geometry` | object | Shape type and its parameters (see Geometry types below) |
| `color` | string | Hex color `#RRGGBB`. Stored in uppercase. Default `#CCCCCC`. |
| `position` | number[3] | `[x, y, z]` in scene units. Default `[0, 0, 0]`. |
| `rotation` | number[3] | `[x, y, z]` in degrees, applied in X, Y, Z order. Default `[0, 0, 0]`. |
| `scale` | number[3] | `[x, y, z]` factors, each greater than 0 and at most 1000. Default `[1, 1, 1]`. |

### Common properties include:

- `name` — shape name.
- `geometry` — shape type and geometry-specific parameters.
- `color` — color value, such as a hexadecimal color.
- `position` — `[x, y, z]` coordinates.
- `rotation` — rotation values for the three axes.
- `scale` — scale values for the three axes.

### Geometry types

The API accepts only the types below. Each type is generated from parameters by the 3D engine. Custom meshes with raw vertices and faces are not part of the MVP.

| `type` | Parameters | Limits |
| :--- | :--- | :--- |
| `box` | `width`, `height`, `depth` | Each greater than 0 and at most 1000 |
| `sphere` | `radius`, `segments`, `rings` | `radius` greater than 0 and at most 1000; `segments` integer 3–64; `rings` integer 2–64 |
| `cylinder` | `radius`, `height`, `segments` | `radius` and `height` greater than 0 and at most 1000; `segments` integer 3–64 |
| `cone` | `radius`, `height`, `segments` | `radius` and `height` greater than 0 and at most 1000; `segments` integer 3–64 |

For example, a procedural sphere can store its radius and segment counts, while a custom mesh can store vertices and faces. These different geometry structures can be stored in the same shape record, provided the backend validates them according to their type.

### Shape endpoints

| Method | Endpoint | Purpose | Authentication |
|---|---|---|---|
| POST | `/api/projects/:projectId/shapes` | Create a shape in a project | Required |
| GET | `/api/projects/:projectId/shapes` | List a project's shapes | Required |
| GET | `/api/shapes/:shapeId` | Retrieve a specific shape | Required |
| PATCH | `/api/shapes/:shapeId` | Update a shape | Required |
| DELETE | `/api/shapes/:shapeId` | Delete a shape | Required |


### 4.1 Create a Shape

**Endpoint:** `POST /api/projects/:projectId/shapes`

**Purpose:** Add a shape to an existing project.

**Authentication:** Required.

**Path parameters:**

- `projectId` — ID of the project receiving the shape.

**Request body:**

```json
{
  "name": "Custom Sphere",
  "geometry": {
    "type": "sphere",
    "radius": 1,
    "segments": 16,
    "rings": 8
  },
  "color": "#FF0000",
  "position": [0, 0, 0],
  "rotation": [0, 0, 0],
  "scale": [1, 1, 1]
}
```

This example defines a red sphere with a radius of 1 at the origin. The 3D engine generates the sphere geometry from its parameters.

**Example successful response — `201 Created`:**

```json
{
  "id": "shape_456",
  "projectId": "project_123",
  "name": "Custom Sphere",
  "geometry": {
    "type": "sphere",
    "radius": 1,
    "segments": 16,
    "rings": 8
  },
  "color": "#FF0000",
  "position": [0, 0, 0],
  "rotation": [0, 0, 0],
  "scale": [1, 1, 1]
}
```

**Possible errors:**

- `400 Bad Request` — required fields are missing or geometry is invalid.
- `401 Unauthorized` — authentication is missing or invalid.
- `404 Not Found` — the project does not exist or is not accessible to the user.
- `413 Payload Too Large` — geometry exceeds the allowed size.
- `500 Internal Server Error` — unexpected server error.

### 4.2 List Shapes

**Endpoint:** `GET /api/projects/:projectId/shapes`

**Purpose:** Retrieve all shapes belonging to a project.

**Authentication:** Required.

**Path parameters:**

- `projectId` — ID of the project.

**Example request:**

```http
GET /api/projects/project_123/shapes
```

**Example successful response — `200 OK`:**

```json
{
  "shapes": [
    {
      "id": "shape_456",
      "projectId": "project_123",
      "name": "Custom Sphere",
      "geometry": {
        "type": "sphere",
        "radius": 1,
        "segments": 16,
        "rings": 8
      },
      "color": "#FF0000",
      "position": [0, 0, 0],
      "rotation": [0, 0, 0],
      "scale": [1, 1, 1]
    }
  ]
}
```

**Possible errors:**

- `401 Unauthorized` — authentication is missing or invalid.
- `404 Not Found` — the project does not exist or is not accessible to the user.
- `500 Internal Server Error` — unexpected server error.

### 4.3 Retrieve a Shape

**Endpoint:** `GET /api/shapes/:shapeId`

**Purpose:** Retrieve one shape by its ID.

**Authentication:** Required.

**Path parameters:**

- `shapeId` — ID of the shape.

**Example request:**

```http
GET /api/shapes/shape_456
```

**Example successful response — `200 OK`:**

```json
{
  "id": "shape_456",
  "projectId": "project_123",
  "name": "Custom Sphere",
  "geometry": {
    "type": "sphere",
    "radius": 1,
    "segments": 16,
    "rings": 8
  },
  "color": "#FF0000",
  "position": [0, 0, 0],
  "rotation": [0, 0, 0],
  "scale": [1, 1, 1]
}
```

**Possible errors:**

- `401 Unauthorized` — authentication is missing or invalid.
- `404 Not Found` — the shape does not exist or its project is not accessible to the user.
- `500 Internal Server Error` — unexpected server error.

### 4.4 Update a Shape

**Endpoint:** `PATCH /api/shapes/:shapeId`

**Purpose:** Update a shape's geometry or properties.

**Authentication:** Required.

**Path parameters:**

- `shapeId` — ID of the shape to update.

**Request body:**

```json
{
  "color": "#0000FF",
  "position": [2, 0, 0]
}
```

Only the supplied fields are updated. The example changes the shape's color to blue and moves it along the X axis.

**Example successful response — `200 OK`:**

```json
{
  "id": "shape_456",
  "projectId": "project_123",
  "name": "Custom Sphere",
  "geometry": {
    "type": "sphere",
    "radius": 1,
    "segments": 16,
    "rings": 8
  },
  "color": "#0000FF",
  "position": [2, 0, 0],
  "rotation": [0, 0, 0],
  "scale": [1, 1, 1]
}
```

**Possible errors:**

- `400 Bad Request` — invalid property values or geometry.
- `401 Unauthorized` — authentication is missing or invalid.
- `404 Not Found` — the shape does not exist or its project is not accessible to the user.
- `413 Payload Too Large` — updated geometry exceeds the allowed size.
- `500 Internal Server Error` — unexpected server error.

### 4.5 Delete a Shape

**Endpoint:** `DELETE /api/shapes/:shapeId`

**Purpose:** Delete a shape from its project.

**Authentication:** Required.

**Path parameters:**

- `shapeId` — ID of the shape to delete.

**Example request:**

```http
DELETE /api/shapes/shape_456
```

**Example successful response — `204 No Content`:**

No response body.

**Possible errors:**

- `401 Unauthorized` — authentication is missing or invalid.
- `404 Not Found` — the shape does not exist or its project is not accessible to the user.
- `500 Internal Server Error` — unexpected server error.

## 5. Saving and Loading Projects

A project is stored as a project record plus its shapes. The frontend can save in two ways.

| Use | How | Endpoints |
| :--- | :--- | :--- |
| Autosave after each edit (move, rotate, resize, recolor) | One small request per change | `POST`, `PATCH` and `DELETE` on shapes (see Shapes and Geometry) |
| Explicit Save button | One request that stores the whole model | `PUT /api/projects/:projectId/shapes` (5.1) |
| Load | One request that returns the project and all its shapes | `GET /api/projects/:projectId` (5.2) |

Every change to a shape updates the project's `updatedAt`. When two browser tabs edit the same project, the last write wins in the MVP.

### 5.1 Replace All Shapes of a Project

**Endpoint:** `PUT /api/projects/:projectId/shapes`

**Purpose:** Store the complete model in one atomic operation: either every change is applied or none.

**Authentication:** Required.

**Path parameters:**

- `projectId` — ID of the project to save.

**Request body:**

```json
{
  "shapes": [
    {
      "id": "shape_456",
      "geometry": { "type": "sphere", "radius": 1, "segments": 16, "rings": 8 },
      "color": "#0000FF",
      "position": [2, 0, 0]
    },
    {
      "geometry": { "type": "box", "width": 2, "height": 1, "depth": 1 },
      "color": "#00AA00"
    }
  ]
}
```

**How the shapes are saved:**

- A shape with the `id` of an existing shape in this project is updated.
- A shape without an `id` is created.
- An existing shape that is missing from the request is deleted.
- Each shape in the request is treated as a complete shape. Properties that are left out take their default values, not their previous values.
- Shapes are returned in the order they were sent.
- A project can hold at most 500 shapes.

**Example successful response — 200 OK:**

```json
{
  "updatedAt": "2026-10-12T18:45:00Z",
  "shapes": [
    {
      "id": "shape_456",
      "projectId": "project_123",
      "name": "Sphere",
      "geometry": { "type": "sphere", "radius": 1, "segments": 16, "rings": 8 },
      "color": "#0000FF",
      "position": [2, 0, 0],
      "rotation": [0, 0, 0],
      "scale": [1, 1, 1]
    },
    {
      "id": "shape_789",
      "projectId": "project_123",
      "name": "Box",
      "geometry": { "type": "box", "width": 2, "height": 1, "depth": 1 },
      "color": "#00AA00",
      "position": [0, 0, 0],
      "rotation": [0, 0, 0],
      "scale": [1, 1, 1]
    }
  ]
}
```

**Possible errors:**

| Status | Code | Reason |
| :---: | :--- | :--- |
| 400 | `INVALID_REQUEST` | Body is missing or is not valid JSON |
| 401 | `UNAUTHENTICATED` | Authentication is missing or invalid |
| 404 | `NOT_FOUND` | The project does not exist or is not accessible to the user, or a supplied shape `id` is not part of this project |
| 409 | `SHAPE_LIMIT_REACHED` | The request contains more than 500 shapes |
| 413 | `PAYLOAD_TOO_LARGE` | Request body exceeds 1 MB |
| 422 | `VALIDATION_ERROR` | A shape is invalid. The message names the invalid field and its array position, for example `shapes[1].geometry.width`. |
| 500 | `INTERNAL_ERROR` | Unexpected server error |

### 5.2 Loading a Project

Loading needs no separate endpoint. `GET /api/v1/projects/:projectId` returns the project together with all its shapes, including each shape's geometry and properties, so the frontend can rebuild the whole model from one response. See the Projects section for the request and response format.

Export uses the last saved state of a project, so the frontend must save the project before it requests an STL export.

The proposed API saves a project record separately from its shapes. The frontend may therefore create a project first and then create, update, or delete its shapes through the shape endpoints.

If the frontend instead saves the entire model in one request, a project-level save endpoint and its complete request and response format must be documented.

Loading must return enough information for the frontend to reconstruct the project, including each shape's geometry and properties.

## 6. STL Export

**Endpoint:** `POST /api/projects/:projectId/export`

**Purpose:**  Generates an STL file from the saved state of a project. Changes the user has not saved are not included, so the frontend must save before it exports. Export does not change any data, so it uses GET and a plain download link works with the session cookie.

**Authentication:** Required.

**Path parameters:**

- `projectId` — ID of the project to export.

**Request body:** No body is required in this proposed design.

**Example request:**

```http
POST /api/projects/project_123/export
```

**Expected successful response:**

The server returns the generated STL file for download. The final implementation must document:

- The actual HTTP success status.
- The response content type, typically `model/stl`.
- The filename and `Content-Disposition` behavior.
- Whether the file is returned directly or through a download URL.

**Possible errors:**

- `401 Unauthorized` — authentication is missing or invalid.
- `404 Not Found` — the project does not exist or is not accessible to the user.
- `400 Bad Request` — the project contains invalid or non-exportable geometry.
- `500 Internal Server Error` — export fails unexpectedly.

The export workflow, file format, and error behavior must be verified against the implementation.

## 7. Authorization and Validation

### Project access

Under the MVP design, users may access and modify only their own projects.

The backend must:

1. Identify the authenticated user through the chosen authentication mechanism.
2. Associate newly created projects with that user.
3. Check project ownership before returning or modifying a project.
4. Check access to the parent project before creating, retrieving, updating, or deleting a shape.
5. Restrict search results to projects the user is authorized to access.
6. Avoid relying on a user ID supplied by the frontend to determine access permissions.

An inaccessible project or shape may return `404 Not Found` to avoid revealing whether another user's private resource exists. 

### Input validation

The backend should validate:

- Required fields and data types.
- Project names and other text fields.
- Supported geometry types and their required parameters.
- Sphere radius and segment counts when spheres are supported.
- Vertex and face structure when custom meshes are supported.
- Color values and position, rotation, and scale arrays.
- Geometry size limits to prevent excessive resource use.

Only geometry types supported by the implemented 3D engine should be accepted.


## 8. Future Features — Outside the MVP

The following features are not part of the MVP API unless they are implemented and explicitly added to the current API reference.

| Feature | Proposed endpoint | Purpose |
|---|---|---|
| Two-factor authentication | `POST /api/auth/2fa/setup` | Begin setting up a second authentication factor |
| Two-factor authentication | `POST /api/auth/2fa/verify` | Verify a second authentication factor |
| Project sharing | `POST /api/projects/:projectId/shares` | Grant another user access to a project |
| Project visibility | `PATCH /api/projects/:projectId/visibility` | Set project visibility to public or private |

The 3D canvas is primarily a frontend feature and does not necessarily require its own endpoint. Geometry editing, saving, loading, and STL export are covered by the proposed MVP routes.

## 9. Decisions to Confirm Before Finalizing

1. **Implemented routes:** Verify all endpoint paths, HTTP methods, and authentication requirements against the backend.
2. **Authentication:** Confirm cookie-based sessions or bearer tokens and document the exact frontend behavior.
3. **Project saving:** Decide whether the frontend saves projects and shapes separately or submits the entire model through one project-level endpoint.
4. **Geometry format:** Confirm supported shape types and their geometry fields, including whether custom vertices and faces are supported.
5. **Response formats:** Confirm the exact fields, IDs, timestamps, and response wrappers returned by each endpoint.
6. **Search scope:** Confirm whether search covers project names only and whether results are filtered by ownership.
7. **STL export:** Confirm the file response format, filename, and whether export is synchronous or uses a background job.
8. **Error behavior:** Verify status codes and response bodies for validation, authentication, authorization, and missing resources.

