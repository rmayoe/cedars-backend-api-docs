---
title: Notes
nav_order: 6
---

# Notes API

Notes is the feature that lets a person keep their own personal written notes inside the
app. A user can jot down free-form thoughts, or attach a note to a sermon they were
listening to so the note stays linked to that sermon. They can mark any note as a favourite,
search across the text of their own notes, and delete notes they no longer want. Deleted
notes are not gone for good — they move to a "Recently Deleted" area and can be recovered
later.

Under the hood, every note belongs to exactly one user, so a user only ever sees and edits
their own notes. Notes carry a creation timestamp and are typically shown grouped by the
day they were written. Deletion is a soft-delete: instead of erasing the row, the backend
just marks it as deleted (a timestamp), which is what makes recovery possible. This doc
follows the shared patterns in `_CONVENTIONS.md`.

## Screens & user actions

**Notes home**

| Area | User action | API |
| ---- | ----------- | --- |
| Search bar ("Search Notes") | Type to search across the user's own note titles/bodies | `GET /notes?q=` |
| Create button (glass / stylus button, top-right of search) | Start a new note | `POST /notes` |
| Library card — **All Notes** | Browse every note the user owns | `GET /notes` |
| Library card — **Sermon Notes** | Browse only notes attached to a sermon | `GET /notes?sourceType=sermon` |
| Library card — **Favourites Notes** | Browse notes the user favourited | `GET /notes?favourite=true` |
| Library card — **Recently Deleted** | Browse soft-deleted notes | `GET /notes?deleted=true` |
| **Recent Notes** feed | See own notes grouped by date announcements (`09 · JAN 2026 · WEDNESDAY`) | `GET /notes` (sorted by `createdAt` desc; client groups by date) |
| Note card | Tap to open / read a note | `GET /notes/{id}` |
| Note card — attached-source block ("Notes taken while listening to <sermon>") | Deep-link to the source sermon; dismiss (`×`) detaches the source from the note | `PATCH /notes/{id}` (`sourceType`/`sourceId` → null) |

**Note detail / editor** (open a card)

| User action | API |
| ----------- | --- |
| Edit title/body | `PATCH /notes/{id}` |
| Favourite / unfavourite | `PATCH /notes/{id}` (`favourite`) |
| Attach a note to a sermon while listening | `POST /notes` or `PATCH /notes/{id}` with `sourceType`/`sourceId` |
| Delete (soft-delete → Recently Deleted) | `DELETE /notes/{id}` |
| Restore from Recently Deleted | `PATCH /notes/{id}` (`deletedAt` → null) *(see open questions)* |

Notes card content observed in the design: a bold **title** (`FROM STEWARDNESS TO OWNERSHIP`),
a one-line **body preview** (`The Lord God said, "It is not good for the man to be alone…"`),
and — for sermon-linked notes — a **source block** showing a thumbnail, the sermon title
(`Doctrine of The Church`) and the campus/source name (`Cedars House of Grace`). Free-form
notes (e.g. `THE POWER OF COMMUNITY`) render title + body only, with no source block.

## APIs required

| Method | Path | Auth | Description |
| ------ | ---- | ---- | ----------- |
| GET | `/notes` | Bearer | List the authenticated user's notes (filterable, searchable, paginated). |
| POST | `/notes` | Bearer | Create a new note (free-form or attached to a source). |
| GET | `/notes/{id}` | Bearer | Fetch a single note owned by the user. |
| PATCH | `/notes/{id}` | Bearer | Partially update a note (title, body, favourite, source, restore). |
| DELETE | `/notes/{id}` | Bearer | Soft-delete a note (moves it to Recently Deleted). |

## Endpoint detail

### GET /notes

Returns the authenticated user's notes, newest first (`createdAt` desc). The client groups
rows by calendar date for the "Recent Notes" date-announcement headers, so the response
does not need to pre-group — but every row must carry an accurate `createdAt`.

**Query params**

| Param | Type | Default | Description |
| ----- | ---- | ------- | ----------- |
| `q` | string | — | Full-text search over the user's own note `title` and `body`. |
| `sourceType` | string | — | Filter by source kind. `sermon` powers the "Sermon Notes" tab. Also reserved: `devotional`, `service`, `book`, `null`/`none` for free-form only. |
| `sermonId` | string | — | Filter to notes attached to a specific sermon (all notes I took on that sermon). |
| `sourceId` | string | — | Generic form of `sermonId` for non-sermon sources. |
| `favourite` | boolean | — | `true` → only favourited notes (Favourites tab). |
| `deleted` | boolean | `false` | `true` → only soft-deleted notes (Recently Deleted). Omitted/`false` excludes deleted notes. |
| `from` | ISO date | — | Only notes with `createdAt >= from`. |
| `to` | ISO date | — | Only notes with `createdAt <= to`. |
| `page` | int | `1` | Page number. |
| `perPage` | int | `20` | Page size. |

**Response** `200 OK`

```json
{
  "data": [
    {
      "id": "note_9f2c1a",
      "userId": "usr_1a2b3c",
      "title": "FROM STEWARDNESS TO OWNERSHIP",
      "body": "The Lord God said, \"It is not good for the man to be alone. I will make a helper suitable for him.\"",
      "preview": "The Lord God said, \"It is not good for the man to be alone…\"",
      "favourite": false,
      "sourceType": "sermon",
      "sourceId": "srm_44de21",
      "sermonId": "srm_44de21",
      "source": {
        "id": "srm_44de21",
        "type": "sermon",
        "title": "Doctrine of The Church",
        "subtitle": "Cedars House of Grace",
        "thumbnail": "https://cdn.cedars.app/sermons/srm_44de21/thumb.jpg"
      },
      "createdAt": "2026-01-09T18:32:00Z",
      "updatedAt": "2026-01-09T18:40:12Z",
      "deletedAt": null
    },
    {
      "id": "note_71bb08",
      "userId": "usr_1a2b3c",
      "title": "THE POWER OF COMMUNITY",
      "body": "\"For where two or three gather in my name, there am I with them.\"",
      "preview": "\"For where two or three gather in my name…\"",
      "favourite": true,
      "sourceType": null,
      "sourceId": null,
      "sermonId": null,
      "source": null,
      "createdAt": "2026-01-08T09:05:00Z",
      "updatedAt": "2026-01-08T09:05:00Z",
      "deletedAt": null
    }
  ],
  "meta": { "page": 1, "perPage": 20, "total": 42 }
}
```

### POST /notes

Create a note. `title` and `body` are the minimum. Attach to a source by sending
`sourceType` + `sourceId` (the app sends these when the user creates a note while listening
to a sermon).

**Request body**

```json
{
  "title": "THE ROLE OF COMPASSION",
  "body": "No act of kindness, no matter how small, is ever wasted.",
  "favourite": false,
  "sourceType": "sermon",
  "sourceId": "srm_44de21"
}
```

**Response** `201 Created` — the full created `Note` object (same shape as a `GET /notes` row,
with the resolved `source` block expanded when `sourceId` is present).

### GET /notes/{id}

**Path params**

| Param | Type | Description |
| ----- | ---- | ----------- |
| `id` | string | The note id. Must belong to the authenticated user. |

**Response** `200 OK` — a single `Note` (full `body`, resolved `source`). `403`/`404` if the
note is not owned by the caller.

### PATCH /notes/{id}

Partial update — send only the fields that change. Used for editing content, toggling
favourite, detaching a source (the `×` on the source block), and restoring a deleted note.

**Request body examples**

Edit content:
```json
{ "title": "THE IMPORTANCE OF DIVERSITY", "body": "\"Unity in diversity is a strength.\"" }
```

Toggle favourite:
```json
{ "favourite": true }
```

Detach the source (dismiss the "Notes taken while listening to…" block):
```json
{ "sourceType": null, "sourceId": null }
```

Restore from Recently Deleted:
```json
{ "deletedAt": null }
```

**Response** `200 OK` — the updated `Note`.

### DELETE /notes/{id}

Soft-delete. The note moves to **Recently Deleted** (sets `deletedAt`) rather than being
purged, so it stays retrievable via `GET /notes?deleted=true` and restorable via `PATCH`.

**Response** `204 No Content` (or `200` with the tombstoned `Note` including `deletedAt`).
See open questions for hard-delete / auto-purge policy.

## Entities

### Note

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `id` | string | no | Note id (UUID or slug). |
| `userId` | string | no | Owner. Server-set from the auth token; never trust a client-supplied value. |
| `title` | string | yes | Note heading (rendered bold/uppercase in the card). |
| `body` | string | yes | Full note content. Plain text today; may become rich text (see open questions). |
| `preview` | string | yes | Server-truncated single-line snippet for the card. Optional — client can derive from `body`. |
| `favourite` | boolean | no | Whether the user favourited the note (Favourites tab). Default `false`. |
| `sourceType` | string | yes | Kind of attached source: `sermon` (seen in design), reserved `devotional` / `service` / `book`. `null` for free-form notes. |
| `sourceId` | string | yes | Id of the attached source record. `null` for free-form. |
| `sermonId` | string | yes | Convenience alias of `sourceId` when `sourceType == "sermon"`; also the `?sermonId=` filter key. Backend may expose only `sourceType`/`sourceId` and let the client derive this. |
| `source` | object | yes | Resolved/expanded source for rendering the card block. `null` for free-form. See below. |
| `createdAt` | ISO-8601 | no | Creation time (UTC). Drives the date-announcement grouping in "Recent Notes". |
| `updatedAt` | ISO-8601 | no | Last edit time (UTC). |
| `deletedAt` | ISO-8601 | yes | Soft-delete timestamp. `null` = active; non-null = in Recently Deleted. |

### Note.source (embedded)

Denormalised snapshot of the attached source so the card renders without a second request.

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `id` | string | no | Source record id (e.g. the sermon id). |
| `type` | string | no | Source kind — mirrors `sourceType` (`sermon`, …). |
| `title` | string | yes | Source title (e.g. `Doctrine of The Church`). |
| `subtitle` | string | yes | Secondary line (e.g. campus / church `Cedars House of Grace`). |
| `thumbnail` | string (URL) | yes | Absolute thumbnail URL shown left of the source title. |

## Notes / open questions for backend

- **Rich text vs plain text.** The design shows short single-line previews; unknown whether
  the editor allows formatting. Decide now whether `body` is plain text, Markdown, or a rich
  structured format (e.g. Delta/HTML) — it affects storage, search, and the `preview` field.
  Recommend plain text or Markdown for v1.
- **Source attachment is optional and polymorphic.** The design confirms notes can be
  free-form (`source == null`) or attached to a **sermon**. Confirm the full set of
  attachable source types (sermon only, or also devotional/service/book). If more than
  sermon, prefer the generic `sourceType` + `sourceId` pair over a sermon-specific column;
  keep `sermonId`/`?sermonId=` as an alias for the common case.
- **`source` denormalisation vs join.** Confirm whether the server embeds a `source` snapshot
  (title/subtitle/thumbnail) on each note or the client must fetch the sermon separately.
  Embedding is strongly preferred so the feed renders in one call; decide whether the snapshot
  is frozen at attach time or always reflects the live source.
- **Soft-delete lifecycle.** "Recently Deleted" implies retention. Confirm auto-purge window
  (e.g. 30 days), whether `DELETE` is idempotent, and whether a separate hard-delete /
  empty-trash endpoint (e.g. `DELETE /notes/{id}?permanent=true`) is needed.
- **Grouping.** The client groups by `createdAt` calendar date for date-announcement headers.
  Confirm grouping key is `createdAt` (not `updatedAt`) and the timezone used for the day
  boundary (device local vs tenant timezone vs UTC).
- **Search scope.** Confirm `?q=` searches only the caller's own notes (title + body) and
  respects the `deleted`/`favourite`/`sourceType` filters when combined.
- **Offline / sync & conflict handling.** Notes are created/edited on-device, so define a
  sync strategy: client-generated ids (idempotent create), `updatedAt`-based last-write-wins
  vs a version/ETag for optimistic concurrency, and how conflicting edits are resolved.
- **Sharing / export.** Not shown in this node, but likely future scope. Flag whether notes
  are strictly private to the user, or will support sharing/export (PDF, share sheet); this
  affects the access model beyond simple owner-scoping.
- **Pagination.** `GET /notes` uses `page`/`perPage`. For a long-scrolling feed, cursor
  pagination keyed on `createdAt` may be preferable — confirm which the backend will support.
