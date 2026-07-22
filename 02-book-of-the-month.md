---
title: Book of the Month
nav_order: 4
---

# Book of the Month

Book of the Month is a reading feature that highlights one recommended book each
calendar month. The app editorially picks a featured book for the current month;
users see it on the home screen, open it to read its summary, author biography,
and publication details (page count, language, publisher, ISBN), and then read or
download the book itself. Alongside the featured pick, users can browse
recommended and top-rated books and look back at what was featured in previous
months. Users can also save (bookmark) a book and give it a thumbs-up.

Under the hood it is a single `Book` resource. Each book is tagged with the month
it is featured for, so "the current book" is just the book whose month matches
today. The actual book file (a PDF or EPUB, or a link to a hosted reader) is
fetched through a separate content endpoint, which lets the backend gate access
and hand out expiring/signed URLs without changing the book record itself. Reads,
bookmarks, likes, and browsing are all served from the same `Book` resource.

The feature already has partial app scaffolding. This doc follows the shared
`_CONVENTIONS.md`.

## Screens & user actions

| Screen | User action | Drives |
| ------ | ----------- | ------ |
| Home — "Book of the Month" carousel | See the current featured book (cover, author, title, blurb) | `GET /books/current` |
| Home — carousel | Scroll a horizontal rail of featured / recommended / top-rated books | `GET /books/current`, `GET /books/recommended` |
| Home — book card | Tap "Read now" (available) or see "Unavailable to read" (disabled) | `readUrl` / `isReadable` on the book |
| Home — book card | Tap the bookmark to save a book | `POST /books/{id}/bookmark` |
| Home — book card | Tap the whole card to open detail | `GET /books/{id}` |
| Book detail | View cover, title, author, and the Pages / Ratings / Month strip | `GET /books/{id}` |
| Book detail | Read Summary, "About the author", and publication details (Published, Language, Publisher, ISBN) | `GET /books/{id}` |
| Book detail | Tap "Read Now" (pinned bar) to open/download the book content | `GET /books/{id}/content` |
| Book detail | Tap the thumbs-up to like/rate the book | `POST /books/{id}/reaction` |
| Book detail | Tap share (toolbar) | client-side share of the book's public URL |
| Book detail | Browse the "Recommended for you" carousel | `GET /books/{id}/recommended` |
| (Past months) | Browse previous months' books | `GET /books?month=YYYY-MM` / `GET /books` |

## APIs required

| Method | Path | Auth | Description |
| ------ | ---- | ---- | ----------- |
| GET | `/books/current` | Bearer | The current month's featured Book of the Month. |
| GET | `/books/{id}` | Bearer | Full detail for one book. |
| GET | `/books` | Bearer | List/browse books; filter by month, search, paginate. |
| GET | `/books/recommended` | Bearer | Recommended books for the home rail (global). |
| GET | `/books/{id}/recommended` | Bearer | Books recommended alongside a given book (detail rail). |
| GET | `/books/{id}/content` | Bearer | Resolve the readable/downloadable content (PDF/EPUB or read URL). |
| POST | `/books/{id}/bookmark` | Bearer | Bookmark (save) a book for the current user. |
| DELETE | `/books/{id}/bookmark` | Bearer | Remove a bookmark. |
| POST | `/books/{id}/reaction` | Bearer | Like / thumbs-up (or clear) a book. |

> Legacy note: the previous app used `GET /monthly_books` and
> `GET /monthly_books/current` (snake_case fields: `cover_image`,
> `featured_images`, `date`). New endpoints should be camelCase per the
> conventions; keep the old paths working or provide a migration alias.

## Endpoint detail

### GET /books/current

Returns the single featured book for the current month (tenant-scoped). Used by
the home carousel's lead card.

Query params:

| Param | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `month` | string `YYYY-MM` | no | Override "current" to fetch a specific month's featured book. Defaults to the server's current month for the tenant. |

Response:

```json
{
  "data": {
    "id": "kingdom-stewardship",
    "title": "Kingdom Stewardship",
    "author": {
      "id": "tony-evans",
      "name": "Tony Evans",
      "bio": "Dr. Tony Evans is the founder and senior pastor of Oak Cliff Bible Fellowship..."
    },
    "coverImage": "https://cdn.hog.app/books/kingdom-stewardship/cover.jpg",
    "description": "Managing all of life under God's rule. Over 1 million books sold in the Kingdom series.",
    "summary": "Kingdom Stewardship explores what it means to manage every area of life...",
    "month": "2026-07",
    "monthLabel": "July 2026",
    "format": "pdf",
    "readUrl": "https://cdn.hog.app/books/kingdom-stewardship/read.pdf",
    "contentUrl": "https://api.hog.app/books/kingdom-stewardship/content",
    "isReadable": true,
    "pageCount": 240,
    "language": "English",
    "publisher": "Cedars House Print Media Company",
    "isbn": "978-686743-02836353",
    "ratingCount": 240,
    "averageRating": 4.6,
    "bookmarked": false,
    "liked": false,
    "publishedAt": "2026-08-24T00:00:00Z",
    "createdAt": "2026-06-01T09:00:00Z",
    "updatedAt": "2026-07-01T09:00:00Z"
  }
}
```

If no book is set for the month, return `200` with `{ "data": null }` (the card
renders an empty/unavailable state) rather than a `404`.

### GET /books/{id}

Path params:

| Param | Type | Description |
| ----- | ---- | ----------- |
| `id` | string | Book id or slug. |

Returns a full `Book` (same shape as above). Powers the detail screen: title
block (Pages / Ratings / Month), the summary + "About the author" card, and the
Published / Language / Publisher / ISBN rows.

### GET /books

Browse and filter books — used for past months' books and any "all books" list.

Query params:

| Param | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `month` | string `YYYY-MM` | no | Only books featured in this month. |
| `from` | string `YYYY-MM` | no | Range start (inclusive) for month browsing. |
| `to` | string `YYYY-MM` | no | Range end (inclusive). |
| `q` | string | no | Full-text search over title/author. |
| `authorId` | string | no | Filter by author. |
| `sort` | string | no | `month` (default, newest first), `rating`, `title`. |
| `page` | int | no | Default `1`. |
| `perPage` | int | no | Default `20`. |

Response (list envelope):

```json
{
  "data": [
    { "id": "kingdom-woman", "title": "Kingdom Woman", "author": { "id": "tony-evans", "name": "Tony Evans" }, "coverImage": "https://cdn.hog.app/books/kingdom-woman/cover.jpg", "month": "2026-06", "monthLabel": "June 2026", "isReadable": true, "averageRating": 4.4 }
  ],
  "meta": { "page": 1, "perPage": 20, "total": 34 }
}
```

List rows may return a lightweight `Book` (omit `summary`, `contentUrl`,
`isbn`, `publisher`); the client hydrates the rest from `GET /books/{id}`.

### GET /books/recommended  &  GET /books/{id}/recommended

The home "Recommended for you" / "Top Rated this year" rails, and the detail
screen's recommendation carousel. Both return a list of (lightweight) `Book`
rows in the same envelope as `GET /books`.

Query params:

| Param | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `limit` | int | no | Max rows (default `10`). |

```json
{ "data": [ { "id": "kingdom-man", "title": "Kingdom Man", "author": { "name": "Tony Evans" }, "coverImage": "https://cdn.hog.app/books/kingdom-man/cover.jpg", "isReadable": true } ] }
```

### GET /books/{id}/content

Resolves the actual readable content for the "Read Now" action. Kept separate
from the book record so hosting/DRM/signing can evolve without changing the
`Book` shape, and so access can be gated (entitlement, expiring URL).

Response:

```json
{
  "data": {
    "format": "pdf",
    "url": "https://cdn.hog.app/books/kingdom-stewardship/read.pdf?token=eyJ...&expires=1690000000",
    "expiresAt": "2026-07-22T13:00:00Z",
    "sizeBytes": 5242880,
    "downloadable": true
  }
}
```

- Return `403` with the standard error envelope when the book is not readable
  for this user/tenant (the card shows "Unavailable to read").
- `format` is one of `pdf`, `epub`, `external` (a hosted-reader URL).

### POST /books/{id}/bookmark  /  DELETE /books/{id}/bookmark

No body. Scoped to the authenticated user. Returns the updated book or
`{ "data": { "bookmarked": true } }`.

### POST /books/{id}/reaction

The detail-screen thumbs-up. Body:

```json
{ "type": "like" }
```

`type`: `like` or `none` (to clear). Returns `{ "data": { "liked": true, "ratingCount": 241 } }`.

## Entities

### Book

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `id` | string | no | Book id or slug. |
| `title` | string | no | Book title (e.g. "Kingdom Stewardship"). |
| `author` | Author | yes | Embedded author (see below). May be a bare string in legacy responses. |
| `coverImage` | string (URL) | yes | Absolute cover art URL. |
| `description` | string | yes | Short blurb shown on the card. |
| `summary` | string | yes | Long summary shown on the detail card ("Summary of …"). |
| `month` | string `YYYY-MM` | no | The month this book is featured for. Selection key for `current`. |
| `monthLabel` | string | yes | Display form, e.g. "July 2026" (Month strip on detail). |
| `format` | string | yes | `pdf` \| `epub` \| `external`. |
| `readUrl` | string (URL) | yes | Direct read URL when unsigned/public; prefer `contentUrl` for gated content. |
| `contentUrl` | string (URL) | yes | Endpoint to resolve signed/gated content (`/books/{id}/content`). |
| `isReadable` | bool | yes | Whether "Read now" is enabled vs. "Unavailable to read". Defaults false if absent. |
| `pageCount` | int | yes | Pages (detail "Pages" stat). |
| `language` | string | yes | e.g. "English" (detail card). |
| `publisher` | string | yes | Publisher name (detail card). |
| `isbn` | string | yes | ISBN (detail card). |
| `ratingCount` | int | yes | Number of ratings ("Ratings" stat). |
| `averageRating` | number | yes | 0–5 average. |
| `bookmarked` | bool | yes | Whether the current user has saved this book. |
| `liked` | bool | yes | Whether the current user has thumbs-upped this book. |
| `publishedAt` | ISO-8601 | yes | Publication date ("Published" detail row). |
| `createdAt` | ISO-8601 | yes | Record created. |
| `updatedAt` | ISO-8601 | yes | Record updated. |

### Author

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `id` | string | yes | Author id or slug. |
| `name` | string | no | Display name (e.g. "Tony Evans"). |
| `bio` | string | yes | Biography shown in the detail card's "About the author" section. |
| `avatar` | string (URL) | yes | Optional author photo. |

### BookContent (from `/books/{id}/content`)

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `format` | string | no | `pdf` \| `epub` \| `external`. |
| `url` | string (URL) | no | Signed/expiring URL to the file or hosted reader. |
| `expiresAt` | ISO-8601 | yes | When a signed `url` stops working. |
| `sizeBytes` | int | yes | File size for download progress. |
| `downloadable` | bool | yes | Whether offline download is permitted (vs. stream-only). |

## Notes / open questions for backend

- **Content hosting & DRM.** How is the book file served — a static CDN PDF, a
  signed/expiring URL, or a hosted third-party reader? This determines whether
  `contentUrl` returns a file or a web-reader URL, and whether the client can
  cache/download for offline. `format: external` covers the hosted-reader case.
- **Format.** PDF vs. EPUB drives the in-app reader choice. Confirm one primary
  format; if both can exist, the client keys off `format`.
- **Reading-progress tracking.** The design has no progress UI yet, but a
  `GET/PUT /books/{id}/progress` (last page/percentage per user) is a likely
  near-term need — flag whether to reserve it now.
- **"Month" selection.** Is exactly one book featured per calendar month, and is
  "current" derived from server date or an explicit editorial flag
  (`isCurrent`)? The `?month=YYYY-MM` param assumes one-per-month; confirm.
- **Recommended vs. Top Rated.** The home rail shows both "Recommended For You"
  and "Top Rated this year". Are these two distinct feeds (curated vs.
  rating-sorted) or one endpoint with a `sort`/`type` param? Confirm whether
  `/books/recommended` should accept `?type=recommended|top-rated`.
- **Availability semantics.** `isReadable=false` renders "Unavailable to read".
  Is unavailability per-tenant, per-entitlement (paid), or time-gated (not yet
  released)? Affects whether `/content` returns `403` or the book is filtered
  out entirely.
- **Reactions vs. ratings.** The thumbs-up is a binary like in the UI, but the
  detail strip shows a numeric "Ratings" count. Confirm whether these are the
  same signal (likes counted as ratings) or a star rating is coming later.
- **Author as object vs. string.** Legacy `author` was a plain string. New
  responses should embed an `Author` object to power the "About the author"
  section; the client tolerates both.
- **Pagination for past months.** Confirm page/perPage vs. cursor for the
  browse-by-month history if the catalogue is large.
