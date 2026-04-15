# Profiles API (Data Persistence)

A TypeScript + Express API that:

- accepts a name,
- calls three upstream APIs (`Genderize`, `Agify`, `Nationalize`),
- classifies age group and top nationality,
- stores the result in SQLite,
- exposes CRUD-style endpoints for profile management.

## External APIs Used

- Genderize: `https://api.genderize.io?name={name}`
- Agify: `https://api.agify.io?name={name}`
- Nationalize: `https://api.nationalize.io?name={name}`

No API keys are required.

## Core Behavior

- **Create profile** from a name using all three upstream APIs.
- **Deduplicate by name** (case-insensitive): if a profile already exists for that name, the existing record is returned and no new record is created.
- **Persist data** in SQLite database at `data/profiles.db`.
- **Use UUID v7** for all profile IDs.
- **Store timestamps** as UTC ISO 8601 strings (`created_at`).
- **Enable CORS** with `Access-Control-Allow-Origin: *`.

## Classification Rules

- **Age group** from Agify age:
  - `0-12` -> `child`
  - `13-19` -> `teenager`
  - `20-59` -> `adult`
  - `60+` -> `senior`
- **Nationality** from Nationalize:
  - choose the `country` item with the highest `probability`
  - save both `country_id` and `country_probability`

## Data Model

Each stored profile contains:

- `id` (UUID v7)
- `name` (lowercased, unique case-insensitive)
- `gender`
- `gender_probability`
- `sample_size` (Genderize `count`)
- `age`
- `age_group`
- `country_id`
- `country_probability`
- `created_at` (UTC ISO 8601)

## API Base URL

`http://localhost:3021`

## Error Format

All errors use:

```json
{ "status": "error", "message": "<error message>" }
```

## Endpoints

### 1) Create Profile

`POST /api/profiles`

Request body:

```json
{ "name": "ella" }
```

Success (`201 Created`) when new profile is created:

```json
{
  "status": "success",
  "data": {
    "id": "019d91a4-7c3c-7fc9-b5d2-a82cc21b5811",
    "name": "ella",
    "gender": "female",
    "gender_probability": 0.99,
    "sample_size": 97517,
    "age": 53,
    "age_group": "adult",
    "country_id": "CM",
    "country_probability": 0.09677289106552395,
    "created_at": "2026-04-15T14:56:09.278Z"
  }
}
```

Success (`200 OK`) when profile already exists for the same name:

```json
{
  "status": "success",
  "message": "Profile already exists",
  "data": {
    "id": "019d91a4-7c3c-7fc9-b5d2-a82cc21b5811",
    "name": "ella",
    "gender": "female",
    "gender_probability": 0.99,
    "sample_size": 97517,
    "age": 53,
    "age_group": "adult",
    "country_id": "CM",
    "country_probability": 0.09677289106552395,
    "created_at": "2026-04-15T14:56:09.278Z"
  }
}
```

### 2) Get Single Profile

`GET /api/profiles/{id}`

Success (`200 OK`):

```json
{
  "status": "success",
  "data": {
    "id": "019d91a4-7c3c-7fc9-b5d2-a82cc21b5811",
    "name": "ella",
    "gender": "female",
    "gender_probability": 0.99,
    "sample_size": 97517,
    "age": 53,
    "age_group": "adult",
    "country_id": "CM",
    "country_probability": 0.09677289106552395,
    "created_at": "2026-04-15T14:56:09.278Z"
  }
}
```

### 3) Get All Profiles

`GET /api/profiles`

Optional query parameters (all case-insensitive):

- `gender`
- `country_id`
- `age_group`

Example:

`GET /api/profiles?gender=male&country_id=NG`

Success (`200 OK`):

```json
{
  "status": "success",
  "count": 2,
  "data": [
    {
      "id": "id-1",
      "name": "emmanuel",
      "gender": "male",
      "age": 25,
      "age_group": "adult",
      "country_id": "NG"
    },
    {
      "id": "id-2",
      "name": "sarah",
      "gender": "female",
      "age": 28,
      "age_group": "adult",
      "country_id": "US"
    }
  ]
}
```

### 4) Delete Profile

`DELETE /api/profiles/{id}`

Success: `204 No Content`

## Validation and Error Cases

- `400 Bad Request`: missing or empty `name`
  - message: `"Missing or empty name"`
- `422 Unprocessable Entity`: invalid `name` type
  - message: `"Invalid type"`
- `404 Not Found`: profile not found
  - message: `"Profile not found"`
- `500 Internal Server Error`: unexpected server failures
  - message: `"Server failure"`

### Upstream Invalid Data (Do Not Store)

If upstream APIs return structurally invalid classification data, API responds with `502` and does not insert any row.

- Genderize `gender: null` or `count: 0`
  - `{ "status": "error", "message": "Genderize returned an invalid response" }`
- Agify `age: null`
  - `{ "status": "error", "message": "Agify returned an invalid response" }`
- Nationalize empty `country`
  - `{ "status": "error", "message": "Nationalize returned an invalid response" }`

## Quick Start

### 1) Install dependencies

```bash
npm install
```

### 2) Run in development

```bash
npm run dev
```

Default port is `3021`.  
Override with `PORT` when needed:

```bash
# PowerShell
$env:PORT=3000; npm run dev
```

### 3) Build and run production

```bash
npm run build
npm start
```

## cURL Examples

Create:

```bash
curl -X POST "http://localhost:3021/api/profiles" \
  -H "Content-Type: application/json" \
  -d "{\"name\":\"ella\"}"
```

List with filters:

```bash
curl "http://localhost:3021/api/profiles?gender=female&age_group=adult"
```

Get by ID:

```bash
curl "http://localhost:3021/api/profiles/<profile-id>"
```

Delete:

```bash
curl -X DELETE "http://localhost:3021/api/profiles/<profile-id>"
```

## Project Structure

- `src/server.ts`: API routes, upstream integration, validation, UUID v7 generation, SQLite bootstrap.
- `data/profiles.db`: runtime SQLite database file (auto-created, ignored by git).
