---
title: Conventions
nav_order: 2
---

# Backend API Docs — Shared Conventions

> Read this first. Every section doc in this folder follows these conventions so the
> backend team sees one consistent contract. This document describes how the **Cedars /
> HOG** Flutter app expects the API to behave, derived from the existing data layer.

## App architecture (for context)

The app is a Flutter client with a clean data layer:

- **Transport:** REST/JSON over HTTPS, via `dio`. One `ApiClient` wraps every call.
- **Response envelope:** Handlers return either a bare array or an object envelope.
  The client already unwraps `data`, `items`, or `results`. Prefer:
  ```json
  { "data": <payload>, "meta": { "page": 1, "perPage": 20, "total": 134 } }
  ```
  For lists, wrap rows so pagination can be added without breaking clients.
- **Auth:** Bearer token (JWT) attached by an auth interceptor on every authenticated
  request: `Authorization: Bearer <token>`.
- **Multi-tenancy:** The app is white-labelled across brands/tenants (e.g. Cedars, HOG).
  Every resource is tenant-scoped. Assume the tenant is derived from the auth token or a
  `X-Tenant` header — **do not** require the client to pass a tenant id in the path unless
  a resource is explicitly cross-tenant.
- **Entities are all-nullable on the wire.** The client mirrors the response defensively;
  a missing field must never crash the client. Still, mark which fields are *expected*.

## Documentation format for each section

Each section `.md` must contain, in this order:

1. **Title + one-line summary** of the feature.
2. **Screens & user actions** — a short bullet list, or table, of what the user does on
   each screen (derived from the product design). Keep it action-oriented.
3. **APIs required** — a table of endpoints. Use the columns:

   | Method | Path | Auth | Description |
   | ------ | ---- | ---- | ----------- |

4. **Endpoint detail** — for each non-trivial endpoint: query/path params (table),
   request body (if any), and a representative JSON response example.
5. **Entities** — one subsection per entity with a field table:

   | Field | Type | Nullable | Description |
   | ----- | ---- | -------- | ----------- |

6. **Notes / open questions for backend** — anything ambiguous in the design, pagination,
   sorting, filtering, caching, or realtime needs.

## Shared conventions

- **IDs:** string (UUID or slug). Use `id` consistently.
- **Timestamps:** ISO-8601 UTC strings (`createdAt`, `updatedAt`, `publishedAt`).
- **Media:** absolute URLs (`image`, `thumbnail`, `audioUrl`, `videoUrl`, `hlsUrl`).
- **Durations:** integer seconds (`durationSeconds`), plus optional display string.
- **Pagination:** `?page=1&perPage=20`, response `meta` as above. Cursor pagination is
  fine for feeds — note it if you use it.
- **Filtering/search:** `?q=`, `?month=`, `?from=`, `?to=`, `?campusId=`, `?ministerId=`.
- **Naming:** JSON keys camelCase preferred (the existing API mixes snake_case and
  camelCase; note new endpoints should be camelCase).
- **Errors:** standard envelope `{ "error": { "code": "NOT_FOUND", "message": "..." } }`
  with proper HTTP status codes.
- **User-generated content** (notes, testimonies, prayer/promise): scope to the
  authenticated user; support create/update/delete and list-mine.

## Endpoint naming

Prefix feature endpoints clearly and RESTfully, e.g. `/sermons`, `/ministers`,
`/campuses`, `/books`, `/devotionals`, `/events`, `/testimonies`, `/birthdays`,
`/notes`, `/catch-up`, `/search`.
