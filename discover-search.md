---
title: Discover & Search
nav_order: 10
---

# Discover & Search

Discover & Search is the app's global discovery hub combined with a single unified search
that spans every kind of content the app holds — ministers, sermons, notes, testimonies,
devotionals, livestreams, campuses, scriptures, and books. Instead of hunting through each
feature separately, a user has one place to explore what's available and one search box that
looks across all of it at once. When a user opens Discover without searching, they see a
curated hub: trending and suggested content plus category shortcuts they can browse. When
they start typing, they get a query, and the results come back grouped by content type — a
section of matching ministers, a section of matching sermons, and so on — with only the
types that actually have matches shown. Tapping any result deep-links straight into that
item's own screen, and previous queries are kept as recent searches the user can re-run.

Under the hood this is a read-mostly feature. The backend indexes each content type into a
searchable store, then answers three main calls: one that returns the curated Discover hub,
one that runs a query and returns results grouped by type (or a flat, paginated list when
narrowed to a single type), and a lightweight autocomplete call for suggestions as the user
types. Recent-search history is optional and may live either on the device or server-side.
Everything is tenant-scoped, and each result is a thin, uniform projection of the underlying
item — just enough to display it — with the full object fetched from the owning feature once
the user taps through.

This doc follows the shared conventions in `_CONVENTIONS.md` (envelope, auth, tenancy, IDs,
timestamps, media, pagination, and errors).

---

## Screens & user actions

| Screen / state | User action | Data needed |
| -------------- | ----------- | ----------- |
| **Discover (idle / empty)** | Opens Discover from the bottom toolbar search pill. Sees curated hub: trending content, category shortcuts, and suggested items. | `GET /discover` |
| **Discover (idle, returning user)** | Sees their **recent searches** and can tap one to re-run it, or clear one / clear all. | `GET /search/recent` (or local store) |
| **Typing** | Types in the bottom search field. Sees live autocomplete suggestions as they type. | `GET /search/suggestions?q=` |
| **Results** | Submits a query (e.g. "Ken Abidjan"). Sees a heading "Your search results for '<query>'" and results **grouped into typed sections**: Ministers, Sermons, Notes, Testimonies, Devotionals, Livestreams, Campus, Scriptures, Book Of The Month. Each section renders that content type's native card. | `GET /search?q=` |
| **Tap a result** | Taps any row/card → deep-links to that item's detail screen (minister, sermon player, note, testimony, devotional, livestream, campus, scripture, book). | `deeplink` on each `SearchResult` |
| **Row actions** | On sermon/minister/livestream rows, taps the heart (favourite) or the "more" (…) menu. Favouriting is handled by the owning feature's endpoints, not search. | (see feature docs) |
| **Filter by type** | (Implied) narrow results to a single content type — e.g. "see all Sermons". | `GET /search?q=&type=sermon&page=` |
| **Cancel** | Taps the close (✕) button → returns to the screen they came from before entering Discover. | client-side only |

Notes on the observed design:
- The results screen is a single scroll of typed sections; only sections **with matches**
  are shown ("Only related search result should pop up" — annotation on the frame).
- Each typed section reuses the content type's existing list/card component (audio-card row
  for ministers & sermons, note card, testimony card, check-in/devotional card, livestream
  card, circular campus avatar, scripture catch-up card, book-of-the-month card).
- The search input lives in the **bottom toolbar** (a floating "Search" pill), not a top bar.

---

## APIs required

| Method | Path | Auth | Description |
| ------ | ---- | ---- | ----------- |
| GET | `/discover` | Bearer | Curated Discover hub: trending, category shortcuts, and suggested content sections. |
| GET | `/search` | Bearer | Unified search across all content types. Grouped results by default; `type` narrows to one. |
| GET | `/search/suggestions` | Bearer | Lightweight autocomplete suggestions for the current partial query. |
| GET | `/search/recent` | Bearer | The authenticated user's recent search terms (server-side history, if used). |
| POST | `/search/recent` | Bearer | Record a search term into the user's recent history. |
| DELETE | `/search/recent/{id}` | Bearer | Remove one recent-search entry. |
| DELETE | `/search/recent` | Bearer | Clear all recent searches for the user. |

All endpoints are tenant-scoped (tenant derived from the token / `X-Tenant`). See open
questions on whether recent searches are stored server-side at all.

---

## Endpoint detail

### GET `/discover`

Returns the curated hub shown when Discover is opened with no active query. A list of
`DiscoverSection`s, each with a `type` that tells the client how to render it, plus a flat
list of `SearchResult`-shaped items (or category chips).

**Query params**

| Param | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `limit` | int | no | Max items per section (default e.g. 10). |

**Response**

```json
{
  "data": {
    "sections": [
      {
        "id": "trending",
        "type": "trending",
        "title": "Trending now",
        "items": [
          {
            "type": "sermon",
            "id": "sm_9f2",
            "title": "There Is No Secret Recipe",
            "subtitle": "Cedars House of Grace",
            "image": "https://cdn.hog.app/sermons/9f2/thumb.jpg",
            "deeplink": "hog://sermons/sm_9f2"
          },
          {
            "type": "minister",
            "id": "mn_204",
            "title": "Rev. Gbeminiyi Eboda",
            "subtitle": "Cedars Cathedral",
            "image": "https://cdn.hog.app/ministers/204.jpg",
            "deeplink": "hog://ministers/mn_204"
          }
        ]
      },
      {
        "id": "categories",
        "type": "categories",
        "title": "Browse",
        "categories": [
          { "id": "sermons",      "label": "Sermons",     "icon": "https://cdn.hog.app/ic/sermons.png",     "deeplink": "hog://sermons" },
          { "id": "ministers",    "label": "Ministers",   "icon": "https://cdn.hog.app/ic/ministers.png",   "deeplink": "hog://ministers" },
          { "id": "books",        "label": "Books",       "icon": "https://cdn.hog.app/ic/books.png",       "deeplink": "hog://books" },
          { "id": "events",       "label": "Events",      "icon": "https://cdn.hog.app/ic/events.png",      "deeplink": "hog://events" },
          { "id": "devotionals",  "label": "Devotionals", "icon": "https://cdn.hog.app/ic/devotionals.png", "deeplink": "hog://devotionals" },
          { "id": "testimonies",  "label": "Testimonies", "icon": "https://cdn.hog.app/ic/testimonies.png", "deeplink": "hog://testimonies" }
        ]
      },
      {
        "id": "suggested",
        "type": "suggested",
        "title": "Suggested for you",
        "items": [
          {
            "type": "book",
            "id": "bk_31",
            "title": "Kingdom Woman",
            "subtitle": "Tony Evans",
            "image": "https://cdn.hog.app/books/31.jpg",
            "deeplink": "hog://books/bk_31"
          }
        ]
      }
    ]
  }
}
```

### GET `/search`

Unified search. **Default (no `type`)** returns grouped results — one group per content
type that has matches. **With `type`** returns a flat, paginated list of just that type.

**Query params**

| Param | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `q` | string | yes | The search query. Trim; treat empty as 400 or fall back to `/discover`. |
| `type` | string | no | Narrow to a single content type: `minister`, `sermon`, `note`, `testimony`, `devotional`, `livestream`, `campus`, `scripture`, `book`, `event`. Omit for grouped results. |
| `page` | int | no | 1-based page (only meaningful with `type`). Default 1. |
| `perPage` | int | no | Page size when `type` is set. Default 20. |
| `limitPerGroup` | int | no | When grouped, cap results shown per group (default e.g. 5); the client "see all" jumps to `type`-scoped calls. |

**Response — grouped (no `type`)**

```json
{
  "data": {
    "query": "Ken Abidjan",
    "groups": [
      {
        "type": "minister",
        "title": "Ministers",
        "total": 3,
        "results": [
          {
            "type": "minister",
            "id": "mn_11",
            "title": "Marvin McKinney",
            "subtitle": "Alpha Cathedral",
            "image": "https://cdn.hog.app/ministers/11.jpg",
            "deeplink": "hog://ministers/mn_11"
          },
          {
            "type": "minister",
            "id": "mn_12",
            "title": "Gbeminiyi Adetola",
            "subtitle": "Cedars Cathedral",
            "image": "https://cdn.hog.app/ministers/12.jpg",
            "deeplink": "hog://ministers/mn_12"
          }
        ]
      },
      {
        "type": "sermon",
        "title": "Sermons",
        "total": 8,
        "results": [
          {
            "type": "sermon",
            "id": "sm_51",
            "title": "Monday Morning Prayer with Rev",
            "subtitle": "Rev. Gbeminiyi Eboda",
            "image": "https://cdn.hog.app/sermons/51/thumb.jpg",
            "deeplink": "hog://sermons/sm_51",
            "meta": { "durationSeconds": 2145, "audioUrl": "https://cdn.hog.app/sermons/51/audio.m3u8" }
          }
        ]
      },
      {
        "type": "note",
        "title": "Notes",
        "total": 3,
        "results": [
          {
            "type": "note",
            "id": "nt_7",
            "title": "The Power Of Community",
            "subtitle": "“For where two or three gather in my name...” - Matthew 18:20",
            "image": null,
            "deeplink": "hog://notes/nt_7"
          }
        ]
      },
      {
        "type": "testimony",
        "title": "Testimonies",
        "total": 1,
        "results": [
          {
            "type": "testimony",
            "id": "ts_88",
            "title": "God Did It",
            "subtitle": "Ralph Edward",
            "image": "https://cdn.hog.app/users/ralph.jpg",
            "deeplink": "hog://testimonies/ts_88"
          }
        ]
      },
      {
        "type": "devotional",
        "title": "Devotionals",
        "total": 2,
        "results": [
          {
            "type": "devotional",
            "id": "dv_3",
            "title": "The Importance Of Community",
            "subtitle": "HEB 10:24-25 (NKJV)",
            "image": null,
            "deeplink": "hog://devotionals/dv_3"
          }
        ]
      },
      {
        "type": "livestream",
        "title": "Livestreams",
        "total": 2,
        "results": [
          {
            "type": "livestream",
            "id": "ls_2",
            "title": "There Is No Secret Recipe",
            "subtitle": "Cedars House of Grace",
            "image": "https://cdn.hog.app/livestreams/2/thumb.jpg",
            "deeplink": "hog://livestreams/ls_2"
          }
        ]
      },
      {
        "type": "campus",
        "title": "Campus",
        "total": 3,
        "results": [
          {
            "type": "campus",
            "id": "cm_1",
            "title": "Cedars Cathedral",
            "subtitle": null,
            "image": "https://cdn.hog.app/campuses/1.jpg",
            "deeplink": "hog://campuses/cm_1"
          }
        ]
      },
      {
        "type": "scripture",
        "title": "Scriptures",
        "total": 2,
        "results": [
          {
            "type": "scripture",
            "id": "sc_19",
            "title": "Genesis 2:18",
            "subtitle": "It is not good for the man to be alone.",
            "image": "https://cdn.hog.app/scriptures/19.jpg",
            "deeplink": "hog://scriptures/sc_19"
          }
        ]
      },
      {
        "type": "book",
        "title": "Book Of The Month",
        "total": 2,
        "results": [
          {
            "type": "book",
            "id": "bk_31",
            "title": "Kingdom Woman",
            "subtitle": "Tony Evans",
            "image": "https://cdn.hog.app/books/31.jpg",
            "deeplink": "hog://books/bk_31"
          }
        ]
      }
    ]
  },
  "meta": { "query": "Ken Abidjan", "totalResults": 26 }
}
```

**Response — single type (`type=sermon`)** — flat + paginated:

```json
{
  "data": [
    {
      "type": "sermon",
      "id": "sm_51",
      "title": "Monday Morning Prayer with Rev",
      "subtitle": "Rev. Gbeminiyi Eboda",
      "image": "https://cdn.hog.app/sermons/51/thumb.jpg",
      "deeplink": "hog://sermons/sm_51"
    }
  ],
  "meta": { "query": "prayer", "type": "sermon", "page": 1, "perPage": 20, "total": 8 }
}
```

**Empty result** — return `200` with empty `groups: []` (grouped) or empty `data: []`
(typed) so the client can show an empty state; do not 404.

### GET `/search/suggestions`

Fast autocomplete for the partial query. Returns lightweight suggestions only (label +
optional type/deeplink) — not full result cards. Should be cheap and debounced client-side.

**Query params**

| Param | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `q` | string | yes | Partial query (may be 1–2 chars). |
| `limit` | int | no | Max suggestions (default e.g. 8). |

**Response**

```json
{
  "data": [
    { "text": "Ken Abidjan", "type": null, "deeplink": null },
    { "text": "Kenneth Hagin", "type": "minister", "id": "mn_301", "deeplink": "hog://ministers/mn_301" },
    { "text": "Kingdom Woman", "type": "book", "id": "bk_31", "deeplink": "hog://books/bk_31" }
  ]
}
```

`type`/`id`/`deeplink` are optional: a suggestion may be a pure query-completion string
(tap → run a full search) or an entity hit (tap → deep-link straight to the item).

### Recent searches (server-side variant)

Only needed if recent searches are stored server-side rather than on-device. See open
questions — the client can implement this entirely locally.

**GET `/search/recent`**

```json
{
  "data": [
    { "id": "rs_1", "query": "Ken Abidjan", "searchedAt": "2026-07-21T18:04:00Z" },
    { "id": "rs_2", "query": "prayer",      "searchedAt": "2026-07-20T09:12:00Z" }
  ]
}
```

**POST `/search/recent`** — body:

```json
{ "query": "Ken Abidjan" }
```

Returns the created/updated entry (upsert by normalized query; refresh `searchedAt` if it
already exists):

```json
{ "data": { "id": "rs_1", "query": "Ken Abidjan", "searchedAt": "2026-07-22T10:00:00Z" } }
```

**DELETE `/search/recent/{id}`** — removes one entry → `204`.
**DELETE `/search/recent`** — clears all for the user → `204`.

---

## Entities

### SearchResult

The universal shape the client renders in every group and in Discover sections. It is a thin
projection over each content type so one component family can render mixed content; the
owning feature's detail endpoint provides the full object after the user taps through.

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `type` | string | no | Content type: `minister`, `sermon`, `note`, `testimony`, `devotional`, `livestream`, `campus`, `scripture`, `book`, `event`. Drives which card the client uses. |
| `id` | string | no | Stable id of the underlying item (UUID or slug). |
| `title` | string | no | Primary line (minister name, sermon title, note title, book title…). |
| `subtitle` | string | yes | Secondary line (campus/church, minister name, verse ref, author, testimony author…). |
| `image` | string (URL) | yes | Absolute thumbnail/avatar URL. Null for text-only results (some notes/devotionals). |
| `deeplink` | string | yes | App deep-link/route to open on tap (e.g. `hog://sermons/{id}`). If null, client routes by `type` + `id`. |
| `meta` | object | yes | Optional type-specific extras the card needs inline (e.g. `durationSeconds`, `audioUrl`, `publishedAt`, `isLive`). Keep minimal. |

### DiscoverSection

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `id` | string | no | Section key (e.g. `trending`, `categories`, `suggested`). |
| `type` | string | no | Render hint: `trending`, `categories`, `suggested`, `continue`, `featured`. |
| `title` | string | yes | Display heading for the section. |
| `items` | SearchResult[] | yes | Content items (present for content sections). |
| `categories` | Category[] | yes | Category chips (present when `type = categories`). |

**Category** (inline in a `categories` section)

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `id` | string | no | Category key (`sermons`, `ministers`, `books`…). |
| `label` | string | no | Display label. |
| `icon` | string (URL) | yes | Absolute icon URL. |
| `deeplink` | string | yes | Where the chip navigates (usually a content list route). |

### SearchGroup (in a grouped `/search` response)

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `type` | string | no | The content type this group holds (matches `SearchResult.type`). |
| `title` | string | no | Section heading shown in the UI ("Ministers", "Sermons"…). |
| `total` | int | yes | Total matches of this type (so the client can show "see all"). |
| `results` | SearchResult[] | no | The (capped) matches for this type. |

### SearchSuggestion

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `text` | string | no | The suggestion / completion string to display and, if tapped as a query, to search. |
| `type` | string | yes | Content type when the suggestion is a concrete entity; null for a plain query completion. |
| `id` | string | yes | Entity id when `type` is set. |
| `deeplink` | string | yes | Deep-link to open directly when the suggestion is an entity. |

### RecentSearch (server-side only)

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `id` | string | no | Entry id (for delete). |
| `query` | string | no | The stored query text. |
| `searchedAt` | string (ISO-8601) | no | When last searched (for ordering, newest first). |

---

## Notes / open questions for backend

1. **Search engine.** Which backing store? Options in rough order of quality: DB `LIKE`/
   `ILIKE` (simplest, no typo tolerance, poor ranking) → Postgres full-text search
   (`tsvector`, good enough for launch) → external engine (Algolia / Elasticsearch / Meili
   / Typesense) for typo tolerance, ranking and instant autocomplete. `/search/suggestions`
   in particular benefits from a real search engine. Please confirm the target.
2. **Which content types are indexed?** The design shows ministers, sermons, notes,
   testimonies, devotionals, livestreams, campuses, scriptures, books; the brief also names
   events. Confirm the exact indexed set and whether **user-private** content (a user's own
   notes) is searchable — and if so, that it is scoped to the authenticated user only.
3. **Ranking / ordering.** How are groups ordered on the grouped screen (fixed order vs by
   relevance/score)? And within a group — relevance, recency, or popularity? Should there be
   cross-type relevance (best overall matches first) or purely per-type?
4. **Grouped vs paginated contract.** Confirm the two-mode shape: grouped (capped per type,
   no paging) with no `type`, and flat+paginated with `type`. Is `limitPerGroup` acceptable,
   and what's the default cap the "see all" affordance kicks in at?
5. **Recent searches storage — local vs server.** Simplest is fully on-device (no endpoints).
   Server-side is only worth it if history should sync across devices. Please decide; the
   `/search/recent` endpoints above are optional and can be dropped if local-only.
6. **Autocomplete behaviour.** Min query length before calling `/search/suggestions`?
   Suggested debounce interval? Should suggestions mix query-completions and entity hits, or
   entities only? Any per-tenant popular-query seeding for the empty field?
7. **Typo tolerance / synonyms.** Is fuzzy matching required (e.g. "Ken Abidjan" →
   "Kenneth"), and are there curated synonyms/aliases (minister nicknames, book short
   titles, campus abbreviations)?
8. **Empty-state content.** What should render when Discover has no curated data, when a
   query returns nothing, and when the user has no recent searches? Confirm whether `/discover`
   should always return at least categories so the screen is never blank.
9. **Deep-links.** Confirm the deep-link scheme/route format the app uses so `deeplink` values
   match the router; otherwise the client falls back to routing by `type` + `id`.
10. **Tenancy & caching.** Discover/curation is per-tenant — confirm derivation from token
    vs `X-Tenant`. `/discover` is cacheable (short TTL / CDN); search responses should not be
    cached across users if they include user-private content.
11. **Query logging / analytics.** Should the backend log queries (for trending + zero-result
    analysis)? If so, note PII handling for user-typed text.
