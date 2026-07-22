---
title: Catch Up / Past Service
nav_order: 3
---

# Catch Up / Past Service

Catch Up is the part of the app where a member of a church can watch or listen to services
they were not able to attend. Churches record their regular gatherings (Sunday services,
midweek prayers, sermons, and similar events) as audio or video. This feature collects those
recordings and presents them to each signed-in user so they can view anything they missed. The
app tracks, per user, which recordings they have already finished and where they stopped in
each one, so they can pick up exactly where they left off and so watched items stop being
suggested.

The experience has two parts. First, the app shows the current week's recordings — the services
and moments the user has not yet caught up on for the week they are in right now. The user picks
one and plays it; as they watch, the app periodically saves their playback position back to the
server, and marks the recording as finished once they reach the end. Second, the app shows a
history of everything from earlier weeks, grouped by week and dated, which the user scrolls
through and loads more of as they go; tapping any past recording opens its full detail so it can
be played. Behind the scenes, each catch-up entry is a lightweight, per-user pointer to an
existing sermon or service recording, with that user's watched/resume state layered on top,
rather than a separate copy of the media.

This doc follows the shared contract in [`_CONVENTIONS.md`](./_CONVENTIONS.md): REST/JSON over
HTTPS, Bearer-JWT auth, tenant scoping via the token (or `X-Tenant`), list responses wrapped in a
`{ "data": [...], "meta": {...} }` envelope, camelCase keys, ISO-8601 UTC timestamps, integer
`durationSeconds`, and absolute media URLs.

## Screens & user actions

The screen has two logical regions: the **current-week catch-up** (top) and the **catch-up
history** (below the divider).

| Region | Element | User action | Data needed |
| ------ | ------- | ----------- | ----------- |
| Header | Title "CATCH UP" + week badge "WK14" | Read which week they are catching up on | Current week label/number |
| Header | Subtitle "Stay updated with services and key moments you may have missed this week." | Static copy (may be tenant-configurable) | — |
| This week | Horizontal carousel of numbered **Player Cards** (e.g. `1. THANKSGIVING SUNDAY SERVICE — Sun, 29 March 2026`) with a thumbnail | Swipe through the items missed this week; tap a card to select/preview | List of catch-up items for the current week |
| This week | Large **Play** button | Play the selected / first catch-up item (audio or video) | Playable media URL + resume position |
| History | "Catch Up History" heading | Section label | — |
| History | **Date-announcement** group header per week: big week label `wk'10`, `JAN 2026`, `SUN – MON` | Scan history by week | Week groups with label + date range |
| History | **Stream/Sermon list rows** under each week: thumbnail, title (`Sunday Service \|\| Family Gathering`), date (`Sun, 05 April 2026`), optional minister (`Pastor John Kofi`) | Tap a row to open that past service / sermon detail | Per-item metadata + link to sermon/service |
| History | Scrolling past the bottom loads older weeks | Paginate history | Cursor / page of older week groups |

Primary user journeys:

1. **Browse this week's catch-up cards** — see everything missed in the current week.
2. **Play a catch-up item** — start playback (and resume where left off).
3. **Browse catch-up history grouped by week/date** — scroll dated groups of past streams.
4. **Open a past service / sermon** — tap a history row to view its detail.
5. **(Implied) Mark as watched / save progress** — so cards drop off and playback resumes.

## APIs required

| Method | Path | Auth | Description |
| ------ | ---- | ---- | ----------- |
| GET | `/catch-up/current` | Bearer | Current-week catch-up: week meta + ordered list of items missed this week (powers the cards + Play). |
| GET | `/catch-up/items/{id}` | Bearer | Full detail for a single catch-up item, including playable media and resume position. |
| GET | `/catch-up/history` | Bearer | Past catch-up items grouped by week/date, paginated (powers "Catch Up History"). |
| POST | `/catch-up/items/{id}/progress` | Bearer | Persist playback progress / mark an item watched. |

All endpoints are tenant-scoped from the auth token. "Current week" and per-item `watched`/resume
state are resolved **for the authenticated user**.

## Endpoint detail

### GET `/catch-up/current`

Returns the week being caught up on plus the ordered list of items the user missed this week. Drives
the header badge, the carousel of Player Cards, and the Play button.

Query params:

| Param | Type | Required | Default | Description |
| ----- | ---- | -------- | ------- | ----------- |
| `week` | string | no | current | Override which week to load (e.g. `2026-W14`). Omit for "this week". |
| `includeWatched` | boolean | no | `false` | If `true`, also return items the user already watched (default hides them). |

Response:

```json
{
  "data": {
    "week": {
      "id": "2026-W14",
      "label": "WK14",
      "weekNumber": 14,
      "year": 2026,
      "startDate": "2026-03-29",
      "endDate": "2026-04-04"
    },
    "subtitle": "Stay updated with services and key moments you may have missed this week.",
    "items": [
      {
        "id": "ci_9f2a",
        "position": 1,
        "title": "THANKSGIVING SUNDAY SERVICE",
        "date": "2026-03-29",
        "dateLabel": "Sun, 29 March 2026",
        "thumbnail": "https://cdn.hog.app/catchup/ci_9f2a/thumb.jpg",
        "mediaType": "video",
        "durationSeconds": 5220,
        "durationLabel": "1h 27m",
        "watched": false,
        "resumePositionSeconds": 0,
        "sermonId": "srm_1042",
        "serviceId": "svc_2210"
      }
    ]
  },
  "meta": { "total": 3 }
}
```

Notes: `items` is pre-ordered by `position` (the `1.` / `2.` numbering on the cards). The Play
button plays `items[selectedIndex]` (default the first) using `GET /catch-up/items/{id}`.

### GET `/catch-up/items/{id}`

Path params:

| Param | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `id` | string | yes | Catch-up item id. |

Response:

```json
{
  "data": {
    "id": "ci_9f2a",
    "title": "THANKSGIVING SUNDAY SERVICE",
    "date": "2026-03-29",
    "dateLabel": "Sun, 29 March 2026",
    "thumbnail": "https://cdn.hog.app/catchup/ci_9f2a/thumb.jpg",
    "mediaType": "video",
    "videoUrl": "https://cdn.hog.app/catchup/ci_9f2a/master.m3u8",
    "hlsUrl": "https://cdn.hog.app/catchup/ci_9f2a/master.m3u8",
    "audioUrl": null,
    "durationSeconds": 5220,
    "durationLabel": "1h 27m",
    "watched": false,
    "resumePositionSeconds": 480,
    "sermonId": "srm_1042",
    "serviceId": "svc_2210",
    "minister": "Pastor John Kofi",
    "weekId": "2026-W14"
  }
}
```

### GET `/catch-up/history`

Returns past catch-up items grouped by week/date, newest first. Each group corresponds to one
date-announcement header (`wk'10 · JAN 2026 · SUN – MON`) with its list rows.

Query params:

| Param | Type | Required | Default | Description |
| ----- | ---- | -------- | ------- | ----------- |
| `cursor` | string | no | — | Opaque cursor for the next page of older week groups (from `meta.nextCursor`). |
| `perPage` | integer | no | `10` | Number of **week groups** per page. |
| `from` | date | no | — | Only groups on/after this date (ISO `YYYY-MM-DD`). |
| `to` | date | no | — | Only groups on/before this date. |
| `q` | string | no | — | Free-text search across item titles / ministers. |

Response (cursor pagination — the history is an unbounded feed):

```json
{
  "data": [
    {
      "week": {
        "id": "2026-W10",
        "label": "wk'10",
        "weekNumber": 10,
        "year": 2026,
        "monthYearLabel": "JAN 2026",
        "dayRangeLabel": "SUN - MON",
        "startDate": "2026-01-04",
        "endDate": "2026-01-05"
      },
      "items": [
        {
          "id": "ci_71bd",
          "title": "Sunday Service || Family Gathering",
          "date": "2026-04-05",
          "dateLabel": "Sun, 05 April 2026",
          "thumbnail": "https://cdn.hog.app/catchup/ci_71bd/thumb.jpg",
          "minister": null,
          "mediaType": "video",
          "durationSeconds": 4980,
          "watched": true,
          "resumePositionSeconds": 4980,
          "sermonId": "srm_0990",
          "serviceId": "svc_2101"
        },
        {
          "id": "ci_71be",
          "title": "Monday Morning Prayer with Rev",
          "date": "2026-04-06",
          "dateLabel": "Mon, 06 April 2026",
          "thumbnail": "https://cdn.hog.app/catchup/ci_71be/thumb.jpg",
          "minister": null,
          "mediaType": "audio",
          "durationSeconds": 1800,
          "watched": false,
          "resumePositionSeconds": 0,
          "sermonId": "srm_0991",
          "serviceId": "svc_2102"
        },
        {
          "id": "ci_71bf",
          "title": "Wednesday Midweek Motivation",
          "date": "2026-04-08",
          "dateLabel": "Wed, 08 April 2026",
          "thumbnail": "https://cdn.hog.app/catchup/ci_71bf/thumb.jpg",
          "minister": "Pastor John Kofi",
          "mediaType": "video",
          "durationSeconds": 2400,
          "watched": false,
          "resumePositionSeconds": 0,
          "sermonId": "srm_0992",
          "serviceId": "svc_2103"
        }
      ]
    }
  ],
  "meta": { "nextCursor": "eyJ3ayI6IjIwMjYtVzA5In0", "perPage": 10 }
}
```

Notes: the design shows only 3 rows per week group, but assume `items` can be any length. Tapping a
row opens the linked sermon/service (`sermonId` / `serviceId`) — see cross-reference below.

### POST `/catch-up/items/{id}/progress`

Persists playback progress so cards fall off "this week" and playback resumes. Called periodically
during playback and on completion.

Request body:

```json
{
  "positionSeconds": 480,
  "durationSeconds": 5220,
  "watched": false
}
```

Response:

```json
{
  "data": {
    "id": "ci_9f2a",
    "watched": false,
    "resumePositionSeconds": 480,
    "updatedAt": "2026-07-22T10:15:00Z"
  }
}
```

Notes: server may auto-flip `watched` to `true` once `positionSeconds` passes a completion
threshold (e.g. ≥ 95% of `durationSeconds`), even if the client sends `watched: false`.

## Entities

### CatchUpItem

A single missed service / key moment, either in the current week feed or in history.

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `id` | string | no | Catch-up item id. |
| `position` | integer | yes | Order within the current-week carousel (the `1.`, `2.` numbering). Absent in history. |
| `title` | string | no | Display title, e.g. "THANKSGIVING SUNDAY SERVICE". |
| `date` | string (date) | no | ISO date of the service (`YYYY-MM-DD`). |
| `dateLabel` | string | yes | Preformatted display date, e.g. "Sun, 29 March 2026". Client may format from `date`. |
| `thumbnail` | string (url) | yes | Absolute thumbnail image URL. |
| `mediaType` | string | yes | `video` or `audio`. Drives which player + which URL to use. |
| `videoUrl` | string (url) | yes | Progressive/MP4 video URL (item detail only). |
| `hlsUrl` | string (url) | yes | HLS manifest for adaptive streaming (item detail only). |
| `audioUrl` | string (url) | yes | Audio stream URL for audio items (item detail only). |
| `durationSeconds` | integer | yes | Total duration in seconds. |
| `durationLabel` | string | yes | Human duration, e.g. "1h 27m". |
| `watched` | boolean | yes | Whether the authed user has finished it. Default `false`. |
| `resumePositionSeconds` | integer | yes | Last playback position for the authed user; `0` if unstarted. |
| `minister` | string | yes | Speaker/minister label, e.g. "Pastor John Kofi" (shown on some history rows). |
| `sermonId` | string | yes | Reference to the underlying Sermon (see below), if this item is a sermon. |
| `serviceId` | string | yes | Reference to the underlying Service, if this item is a service stream. |
| `weekId` | string | yes | Id of the CatchUpWeek this item belongs to. |

### CatchUpWeek (history group)

A week grouping in the current feed and history. Renders the date-announcement header.

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `id` | string | no | Week id, ISO week form recommended, e.g. `2026-W14`. |
| `label` | string | no | Short display label, e.g. "WK14" (header) / "wk'10" (history). |
| `weekNumber` | integer | yes | Numeric week of year. |
| `year` | integer | yes | Calendar year of the week. |
| `monthYearLabel` | string | yes | e.g. "JAN 2026" — small header line in history. |
| `dayRangeLabel` | string | yes | e.g. "SUN - MON" — the day span covered by the group. |
| `startDate` | string (date) | yes | First day of the week window (ISO). |
| `endDate` | string (date) | yes | Last day of the week window (ISO). |
| `items` | CatchUpItem[] | yes | Items in this week (history response). |

### Sermon / Service (referenced)

`CatchUpItem.sermonId` / `serviceId` point at the existing Sermon and Service resources documented
elsewhere. A catch-up item is largely a lightweight, user-scoped pointer at one of those with
playback/watched state layered on top. Minimum fields the client relies on when opening a row:

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `id` | string | no | Sermon/Service id (target of the tap). |
| `title` | string | no | Title of the sermon/service. |
| `minister` | string | yes | Speaker/minister. |
| `hlsUrl` / `audioUrl` | string (url) | yes | Playable media, if not already on the catch-up item. |
| `publishedAt` | string (datetime) | yes | When it aired/was published. |

> Backend decision: expose catch-up as its own resource that *references* `/sermons` and services
> (as above), rather than duplicating full sermon payloads into the catch-up feed. Keeps the feed
> light and lets the row tap deep-link into the existing sermon/service detail.

## Notes / open questions for backend

- **Pagination of history.** History is an unbounded feed, so cursor pagination is specified
  (`meta.nextCursor`). Confirm whether the page unit should be **week groups** (assumed here) or
  individual items — grouping by week complicates item-count-based paging near group boundaries.
- **What "week" means.** Is a "week" an ISO week (Mon–Sun), a Sun–Mon liturgical/service week (the
  header shows "SUN – MON"), or a tenant-configured window? This affects `week.id`, `startDate`,
  `endDate`, and which items land in "this week" vs. history. Needs a canonical, tenant-configurable
  definition.
- **Watched-state persistence.** Confirm progress is stored **per user** server-side (not just
  client-local) so it syncs across devices and drives which cards appear in `/catch-up/current`.
  Define the completion threshold that flips `watched` to `true`, and whether re-watching a watched
  item can bring it back into the current-week feed.
- **Current-week population rule.** Should `/catch-up/current` include everything published this
  week, only items the user hasn't watched (`includeWatched=false` default), or a curated subset?
  Clarify how the numbered `position` ordering is decided (chronological vs. editorial).
- **Labels vs. raw values.** The design shows preformatted strings ("Sun, 29 March 2026", "WK14",
  "JAN 2026"). We return both raw (`date`, `weekNumber`) and `*Label` fields; confirm the backend
  will localize/format labels or whether the client should format from raw values.
- **Media auth.** Are media URLs (`hlsUrl`, `videoUrl`, `audioUrl`) signed/expiring per request, or
  static CDN URLs? If signed, the client must fetch item detail immediately before playback.
- **Subtitle copy.** Is the header subtitle static app copy or tenant-configurable content returned
  by the API (currently returned in `/catch-up/current`)?
- **Empty states.** Response shape when there is nothing to catch up on this week, and when history
  is empty — confirm `data.items: []` / `data: []` rather than `null`.
