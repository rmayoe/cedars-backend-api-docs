---
title: Explore & Events
nav_order: 7
---

# Explore & Events

The Explore tab is the app's discovery hub. Instead of a single list, it pulls together
several separate content strips — updates, events, birthdays, testimonies, and Bible-streak
leaders — and stacks them into one feed the user scrolls through top to bottom. The backend
decides which strips appear and in what order; a user opening the tab simply sees a curated
mix of what's happening in their church, taps into whatever interests them, and can follow a
"see more" link on any strip to view that strip's full list.

This doc covers the Events strip specifically. Events means the church's featured and upcoming
gatherings: the hub shows a highlighted event plus a few upcoming ones, a "see more" link opens
the complete list of events for that campus, and tapping any event opens its detail page. On
the composition side, one endpoint builds the whole hub by returning an ordered list of strips
(each strip carries a small preview and a link to its full list), and the Events-specific
endpoints below serve the event list and single-event detail. The other strips (birthdays,
testimonies, streak leaders, updates) are documented separately.

This doc follows the shared rules in `_CONVENTIONS.md` (response envelope, auth, tenancy, IDs,
timestamps, media).

## Screens & user actions

**Explore / Main hub**

| Region | User action | Backing data |
| ------ | ----------- | ------------ |
| Header "EXPLORE" + avatar | Open profile | (auth/profile — other doc) |
| **Updates** strip (3 square thumbnails) | Tap an update to open it | `updates` section of hub feed |
| **Events Happening this Month** + "Discover more, Don't miss out." + chevron | Tap chevron → all events; tap the featured card → event detail | featured event (`events` section) |
| Featured **Video Announcement card** ("NIGHT OF WORSHIP", 21 DEC) | Open event detail (play announcement/video) | one `Event` (featured) |
| **Right at home / This Month at Cedars House of Grace** + chevron | Tap chevron → all events for the campus; tap a card → event detail | `events` list (campus-scoped) |
| Two **Event Cards** (cover, day/month badge, title, date) | Open event detail | `Event` rows |
| **Who's Celebrating?** (birthdays) | — | separate doc |
| **Testimonies shared by others** | — | separate doc |
| **Bible Streak Leaders** | — | separate doc |
| Bottom tab bar (Home, Stream, Explore, Notes, Search) | Navigate | — |

**Explore / Events "More" — all events**

| Region | User action | Backing data |
| ------ | ----------- | ------------ |
| Back button | Return to hub | — |
| Header "Right at home / EVENTS AT CEDARS HOUSE OF GRACE" + campus logo | Shows the campus this list is scoped to | `Campus` context |
| **Upcoming Events** heading | — | — |
| Featured **Video Announcement card** ("YOUTH NIGHT", 01 FEB) | Open event detail | featured `Event` |
| **Event card grid** (2-col: Madam Cash, Prayer Night, Youth Social, Freedom Come) | Tap a card → event detail; scroll for pagination | `events` list |

Notes on behaviour:
- Every section header on the hub has a chevron ("see more") that deep-links into a filtered
  list. For Events the target is `GET /events` scoped to the same campus/month.
- Event cards render a **day/month date badge** (e.g. `21` / `DEC`) plus a full human date
  string (e.g. "FRIDAY, 10TH OCTOBER 2026"). The badge is derived client-side from `startAt`;
  the long string can also be derived client-side, but a preformatted `dateLabel` is welcome.
- The featured card is labelled "Video Announcement card" — it can carry a video/announcement
  in addition to a still cover image (see `videoUrl` / `ctaUrl` on `Event`).
- No explicit RSVP/register/add-to-calendar control is visible in the current design. The RSVP
  endpoint below is specified as **provisional** so the contract exists if a CTA is added; see
  open questions.

## APIs required

| Method | Path | Auth | Description |
| ------ | ---- | ---- | ----------- |
| GET | `/explore` | Bearer | Aggregated hub feed: ordered sections (updates, events, birthdays, testimonies, streak-leaders) for the current tenant/campus. |
| GET | `/events` | Bearer | Paginated event list with `from`/`to`/`category`/`campusId`/`featured` filters. Backs both the hub Events strip and the "More" list. |
| GET | `/events/{id}` | Bearer | Single event detail. |
| POST | `/events/{id}/rsvp` | Bearer | **Provisional.** Register/RSVP the authenticated user for an event, if a CTA is introduced. |
| DELETE | `/events/{id}/rsvp` | Bearer | **Provisional.** Cancel the user's RSVP. |

## Endpoint detail

### GET /explore

Returns the hub as an ordered list of typed sections so the client can render strips in the
order the backend decides (curated). The client renders only section `type`s it knows and
skips the rest. Each section carries a lightweight `items` preview plus a `seeMore` deep-link
target the chevron uses.

Query params:

| Param | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `campusId` | string | no | Scope the feed to one campus. Defaults to the user's home campus (from token/profile) when omitted. |

Response example (Events strips abbreviated to the relevant sections):

```json
{
  "data": {
    "sections": [
      {
        "type": "updates",
        "title": "Updates",
        "items": [ /* update previews — see updates doc */ ]
      },
      {
        "type": "events",
        "title": "Events Happening this Month",
        "subtitle": "Discover more, Don't miss out.",
        "layout": "featured",
        "seeMore": { "label": null, "endpoint": "/events?featured=true" },
        "items": [
          {
            "id": "evt_9f2a",
            "title": "Night of Worship",
            "coverImage": "https://cdn.hog.app/events/night-of-worship.jpg",
            "videoUrl": "https://cdn.hog.app/events/night-of-worship.m3u8",
            "startAt": "2026-10-10T19:00:00Z",
            "dateLabel": "FRIDAY, 10TH OCTOBER 2026",
            "category": "worship",
            "campusId": "cmp_cedars"
          }
        ]
      },
      {
        "type": "events",
        "title": "This Month at Cedars House of Grace",
        "subtitle": "Right at home",
        "layout": "grid",
        "seeMore": {
          "label": null,
          "endpoint": "/events?campusId=cmp_cedars&from=2027-12-01&to=2027-12-31"
        },
        "items": [
          {
            "id": "evt_1c04",
            "title": "The Return of Madam Cash",
            "coverImage": "https://cdn.hog.app/events/madam-cash.jpg",
            "startAt": "2027-12-12T18:00:00Z",
            "dateLabel": "Fri, 12 December 2027",
            "category": "drama",
            "campusId": "cmp_cedars"
          },
          {
            "id": "evt_1c05",
            "title": "At His Feet",
            "coverImage": "https://cdn.hog.app/events/at-his-feet.jpg",
            "startAt": "2027-01-04T17:00:00Z",
            "dateLabel": "Wed, 4 January 2027",
            "category": "worship",
            "campusId": "cmp_cedars"
          }
        ]
      }
    ]
  }
}
```

### GET /events

Backs both the hub Events strip (via each section's `seeMore.endpoint`) and the "More" /
all-events list. Sorted by `startAt` ascending by default (upcoming first).

Query params:

| Param | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `from` | ISO-8601 date/datetime | no | Only events with `startAt` >= this. Defaults to "now" (upcoming only) — pass an explicit `from` to include past events. |
| `to` | ISO-8601 date/datetime | no | Only events with `startAt` <= this. Used for "this month" strips. |
| `category` | string (slug) | no | Filter by category slug, e.g. `worship`, `drama`, `prayer`, `youth`. |
| `campusId` | string | no | Scope to one campus. Omit for tenant-wide. |
| `featured` | boolean | no | When `true`, only events flagged for the featured/announcement card. |
| `q` | string | no | Free-text search over title/description. |
| `page` | integer | no | 1-based page. Default `1`. |
| `perPage` | integer | no | Page size. Default `20`. |

Response example:

```json
{
  "data": [
    {
      "id": "evt_7b11",
      "title": "Youth Night",
      "description": "A night for the young people of the house.",
      "coverImage": "https://cdn.hog.app/events/youth-night.jpg",
      "videoUrl": "https://cdn.hog.app/events/youth-night.m3u8",
      "startAt": "2026-02-01T18:00:00Z",
      "endAt": "2026-02-01T21:00:00Z",
      "dateLabel": "FRIDAY, 1ST FEBRUARY 2026",
      "venue": "Cedars House of Grace, Main Auditorium",
      "campusId": "cmp_cedars",
      "category": "youth",
      "featured": true,
      "ctaUrl": null,
      "rsvp": { "enabled": false, "count": 0, "userStatus": null }
    },
    {
      "id": "evt_1c04",
      "title": "The Return of Madam Cash",
      "description": "A live stage drama.",
      "coverImage": "https://cdn.hog.app/events/madam-cash.jpg",
      "videoUrl": null,
      "startAt": "2027-12-12T18:00:00Z",
      "endAt": null,
      "dateLabel": "Fri, 12 December 2027",
      "venue": "Cedars House of Grace",
      "campusId": "cmp_cedars",
      "category": "drama",
      "featured": false,
      "ctaUrl": null,
      "rsvp": { "enabled": false, "count": 0, "userStatus": null }
    }
  ],
  "meta": { "page": 1, "perPage": 20, "total": 24 }
}
```

### GET /events/{id}

Path params:

| Param | Type | Description |
| ----- | ---- | ----------- |
| `id` | string | Event id. |

Returns a single `Event` (same shape as a list row) with the full `description`, `endAt`,
`venue`, geo, and — when applicable — `rsvp` and `ctaUrl` populated. Returns `404` with the
standard error envelope if not found or not visible to the tenant.

### POST /events/{id}/rsvp — provisional

Only relevant if an RSVP/register CTA is added to the event detail (none shown in the current
design). Registers the authenticated user; user is derived from the bearer token.

Request body (all optional):

```json
{ "status": "going", "guests": 1, "note": "Bringing two friends" }
```

| Field | Type | Description |
| ----- | ---- | ----------- |
| `status` | string | `going` \| `interested` \| `not_going`. Defaults to `going`. |
| `guests` | integer | Additional guests the user is bringing. |
| `note` | string | Optional message to organisers. |

Response: the updated `rsvp` block for the event.

```json
{ "data": { "enabled": true, "count": 143, "userStatus": "going", "guests": 1 } }
```

`DELETE /events/{id}/rsvp` cancels the RSVP and returns the updated `rsvp` block with
`userStatus: null`.

## Entities

### Event

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `id` | string | no | Stable event id (UUID or slug). |
| `title` | string | no | Event name (e.g. "Night of Worship"). |
| `description` | string | yes | Full description. May be absent in list previews. |
| `coverImage` | string (URL) | yes | Absolute URL to the poster/cover art shown on cards. |
| `videoUrl` | string (URL) | yes | Announcement/promo video (HLS or MP4) for the "video announcement" card. |
| `startAt` | ISO-8601 UTC | no | Event start. Drives the day/month badge and sort order. |
| `endAt` | ISO-8601 UTC | yes | Event end, when known. |
| `dateLabel` | string | yes | Preformatted human date (e.g. "FRIDAY, 10TH OCTOBER 2026"). Convenience; client can derive from `startAt` if absent. |
| `venue` | string | yes | Human-readable location/venue name. |
| `location` | object | yes | Optional structured geo `{ address, latitude, longitude, googleMapUrl }` — mirrors `Campus` geo shape. |
| `campusId` | string | yes | Owning campus id. Null for tenant-wide events. |
| `category` | string (slug) | yes | Category slug, e.g. `worship`, `drama`, `prayer`, `youth`. See taxonomy open question. |
| `featured` | boolean | yes | `true` when eligible for the featured/announcement card. Defaults `false`. |
| `ctaUrl` | string (URL) | yes | External call-to-action (register/tickets/livestream). Null when none. |
| `rsvp` | object | yes | RSVP state, see below. Present only when RSVP is enabled for the event. |
| `createdAt` | ISO-8601 UTC | yes | Record created. |
| `updatedAt` | ISO-8601 UTC | yes | Record updated. |

#### Event.rsvp (provisional)

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `enabled` | boolean | no | Whether RSVP/registration is open for this event. |
| `count` | integer | yes | Total confirmed attendees. |
| `userStatus` | string | yes | The caller's status: `going` \| `interested` \| `not_going` \| `null`. |
| `guests` | integer | yes | Additional guests the caller registered. |

### ExploreSection

The hub is an ordered list of these. The client renders known `type`s and skips unknown ones.

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `type` | string | no | Section kind: `updates` \| `events` \| `birthdays` \| `testimonies` \| `streak_leaders`. |
| `title` | string | no | Bold section heading (e.g. "Events Happening this Month"). |
| `subtitle` | string | yes | Secondary line (e.g. "Discover more, Don't miss out.", "Right at home"). |
| `layout` | string | yes | Presentation hint: `featured` (single big card) \| `grid` \| `carousel` \| `list`. |
| `seeMore` | object | yes | Chevron target: `{ label, endpoint }` where `endpoint` is a ready-to-call path (e.g. `/events?campusId=…&from=…&to=…`). Null hides the chevron. |
| `items` | array | no | Preview items for the strip. Element shape depends on `type` (for `events`, an `Event` preview). |

## Notes / open questions for backend

- **Hub composition — curated vs derived.** Are the hub sections and their order fixed
  server-side (curated), or derived from rules (e.g. "events this month", "top 5 streak
  leaders")? The client treats `/explore` as authoritative ordered sections either way, but
  the team should confirm whether editors can pin/reorder/hide sections per tenant/campus.
- **Featured selection.** How is `featured` decided — an editorial flag, or "next N upcoming"?
  Can more than one event be featured at once (the hub shows one big card; the "More" screen
  also shows one)? Confirm tie-breaking/ordering when multiple are featured.
- **Recurring events.** Several posters imply repeat gatherings (Prayer Night, Youth Night).
  Are these modelled as recurrence rules (RRULE) that expand into instances, or as discrete
  one-off rows? If recurring, does `/events` return expanded occurrences (each with its own
  `startAt`) or series objects? This affects `from`/`to` filtering semantics.
- **Timezones.** All timestamps are UTC per conventions, but events are inherently local to a
  campus. Should the response include the event's IANA timezone (e.g. `Africa/Lagos`) so the
  client renders the correct local time and date badge? Right now `dateLabel` sidesteps this —
  confirm whether the badge/label must reflect campus-local or device-local time.
- **Categories taxonomy.** `category` is a free slug in the examples (`worship`, `drama`,
  `prayer`, `youth`). Is there a fixed, backend-owned taxonomy? If so, expose
  `GET /events/categories` (or embed the list in `/explore`) so the client can build a filter
  UI and localise labels, rather than hard-coding slugs.
- **RSVP / ticketing.** No RSVP/register/add-to-calendar control appears in the current design. The
  RSVP endpoints and `Event.rsvp` are specified provisionally. Confirm whether RSVP is in
  scope; if events use external ticketing/registration instead, `ctaUrl` covers it and the
  RSVP endpoints can be dropped. If in scope, clarify capacity/waitlist, guests, and whether
  cancellation is allowed.
- **Add to calendar.** If we want an "add to calendar" action, the client can build an ICS/
  intent from `startAt`/`endAt`/`venue`, but confirm whether the backend should instead expose
  a `calendarUrl`/`.ics` per event for correct timezone + recurrence handling.
- **Campus scoping default.** For `/explore` and `/events` with no `campusId`, should results
  default to the user's home campus (as assumed here) or be tenant-wide? The "More" screen is
  clearly campus-scoped ("EVENTS AT CEDARS HOUSE OF GRACE").
- **Pagination for the hub strips.** `/explore` returns only a preview per section; the full
  list comes from each section's `seeMore.endpoint`. Confirm the preview item count per strip
  is backend-controlled (design shows 1 featured, then 2 grid cards) rather than fixed client
  side.
- **Updates strip.** The three square thumbnails under "Updates" look story/announcement-like.
  Confirm whether these are events, a separate "announcements" resource, or media stories —
  they are treated here as a distinct `updates` section owned by another doc.
