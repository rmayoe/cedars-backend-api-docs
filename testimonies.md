---
title: Testimonies
nav_order: 8
---

# Testimonies

Testimonies is the part of the app where members share and read "praise reports" —
short stories about answered prayers or good things happening in their lives. It has two
sides: a public feed that any signed-in member can scroll and read, and a submission flow
where a member writes and posts their own story. Think of it as a lightweight, read-mostly
social feed scoped to a single church/tenant.

A typical user opens the feature, browses recent testimonies from other members, and taps
one to read the full text. When they want to contribute, they write a title and body,
optionally pick a category or post anonymously, and submit. Submissions are
user-generated content, so they do not go live immediately: each new testimony is created
in a `pending` state and only shows up in the public feed after a moderator reviews and
approves it. Reading is open to all members; writing is gated by that approval step, and a
member can always see the status of their own submissions.

This doc follows the shared conventions in `_CONVENTIONS.md`.

## Screens & user actions

| Area | User action |
| ---- | ----------- |
| Header (`Testimonies`, back button) | Navigate into / out of the feature. |
| Hero — "LETS HEAR YOUR PRAISE REPORT" + **Share** button | Tap **Share** to open the submit-a-testimony flow. |
| Section "Testimonies from others this week" | Read the sub-copy "Be inspired by their journeys"; a curated/recent feed. |
| Testimony card (avatar, uppercase title, author name, body preview) | Scroll the feed; tap a card to read the full testimony. |
| (Submit flow, opened from Share) | Compose title + body, choose a category, optionally attach media, optionally post anonymously, then submit for approval. |
| (Implied) "My submissions" | View the status of testimonies I have submitted (pending / approved / rejected). |
| (Implied) React | Like / react to a testimony to encourage the author. |

Notes on the design:
- Each card shows an **avatar**, an uppercase **title**, an **author name**, and a
  truncated **body preview** — the feed endpoint must return all four.
- The hero is a fixed call-to-action; it is not data-driven (copy can be static or
  tenant-configured).

## APIs required

| Method | Path | Auth | Description |
| ------ | ---- | ---- | ----------- |
| GET | `/testimonies` | Bearer | Public feed of **approved** testimonies. Supports `?category`, `?q`, pagination. |
| GET | `/testimonies/{id}` | Bearer | Full detail of a single approved testimony. |
| POST | `/testimonies` | Bearer | Submit a new testimony. Enters moderation (`status = pending`). |
| GET | `/testimonies/mine` | Bearer | List the authenticated user's own submissions, any status. |
| POST | `/testimonies/{id}/reactions` | Bearer | Add / toggle a reaction (e.g. a "like") on a testimony. *(optional)* |
| DELETE | `/testimonies/{id}/reactions` | Bearer | Remove the current user's reaction. *(optional)* |
| DELETE | `/testimonies/{id}` | Bearer | Author withdraws their own submission. *(optional)* |

All paths are tenant-scoped (tenant derived from the auth token / `X-Tenant` header).

---

## Endpoint detail

### GET `/testimonies`

Returns the public feed. Only `status = approved` rows are visible here.

**Query params**

| Param | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `category` | string | no | Filter by category slug (e.g. `healing`, `finances`, `relationships`). |
| `q` | string | no | Free-text search over title / body / author name. |
| `page` | integer | no | Page number, default `1`. |
| `perPage` | integer | no | Page size, default `20`. |
| `sort` | string | no | `recent` (default) or `popular` (by `reactionCount`). |

**Response**

```json
{
  "data": [
    {
      "id": "tst_9f2a1c",
      "title": "God did it",
      "body": "GOD declared that it was not good for man to be alone; this was not because Adam lacked work or purpose, but because he lacked relationship. This year He brought me a helpmeet and restored my joy...",
      "excerpt": "GOD declared that it was not good for man to be alone; this was not because Adam lacked work or purpose, but",
      "category": "relationships",
      "authorName": "Ralph Edward",
      "userId": "usr_5521",
      "isAnonymous": false,
      "avatar": "https://cdn.hog.app/avatars/usr_5521.jpg",
      "media": [],
      "status": "approved",
      "reactionCount": 42,
      "hasReacted": false,
      "createdAt": "2026-07-18T09:12:00Z",
      "approvedAt": "2026-07-18T14:30:00Z"
    }
  ],
  "meta": { "page": 1, "perPage": 20, "total": 134 }
}
```

Notes:
- `excerpt` is a server-truncated preview for the card. If omitted, the client will
  truncate `body` itself, so it is optional but preferred.
- `hasReacted` is scoped to the authenticated user (only meaningful if reactions ship).

### GET `/testimonies/{id}`

**Path params**

| Param | Type | Description |
| ----- | ---- | ----------- |
| `id` | string | Testimony id. |

Returns a single `Testimony` object (same shape as a feed row, with the full `body` and
all `media`). Returns `404` if the testimony does not exist or is not `approved` and the
requester is not its author.

```json
{
  "data": {
    "id": "tst_9f2a1c",
    "title": "God did it",
    "body": "GOD declared that it was not good for man to be alone...",
    "category": "relationships",
    "authorName": "Ralph Edward",
    "userId": "usr_5521",
    "isAnonymous": false,
    "avatar": "https://cdn.hog.app/avatars/usr_5521.jpg",
    "media": [
      { "type": "image", "url": "https://cdn.hog.app/testimonies/tst_9f2a1c/1.jpg" }
    ],
    "status": "approved",
    "reactionCount": 42,
    "hasReacted": true,
    "createdAt": "2026-07-18T09:12:00Z",
    "approvedAt": "2026-07-18T14:30:00Z"
  }
}
```

### POST `/testimonies`

Submit a new testimony. The server sets `status = pending`, `userId` and `authorName`
from the auth context, and `createdAt`. The row does **not** appear in `GET /testimonies`
until a moderator approves it.

**Request body**

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `title` | string | yes | Short headline (shown uppercase on the card). |
| `body` | string | yes | Full testimony text. |
| `category` | string | no | Category slug. Server may default to `general`. |
| `isAnonymous` | boolean | no | If `true`, hide author identity in the public feed. Default `false`. |
| `mediaIds` | string[] | no | Ids of pre-uploaded media assets (see open questions). |

```json
{
  "title": "God restored my family",
  "body": "For three years we prayed, and this month...",
  "category": "family",
  "isAnonymous": false,
  "mediaIds": ["media_a12", "media_a13"]
}
```

**Response** — `201 Created`

```json
{
  "data": {
    "id": "tst_new0012",
    "title": "God restored my family",
    "body": "For three years we prayed, and this month...",
    "category": "family",
    "authorName": "Mayor A.",
    "userId": "usr_0001",
    "isAnonymous": false,
    "media": [],
    "status": "pending",
    "reactionCount": 0,
    "createdAt": "2026-07-22T10:04:00Z",
    "approvedAt": null
  }
}
```

The client should show a "Thanks — your testimony is awaiting approval" confirmation
based on `status = pending`.

### GET `/testimonies/mine`

Lists the authenticated user's own submissions across **all** statuses so they can track
moderation progress. Same row shape as the feed, plus a `rejectionReason` when
`status = rejected`.

**Query params:** `page`, `perPage` (as above); optional `status` filter.

```json
{
  "data": [
    {
      "id": "tst_new0012",
      "title": "God restored my family",
      "status": "pending",
      "category": "family",
      "reactionCount": 0,
      "createdAt": "2026-07-22T10:04:00Z",
      "approvedAt": null,
      "rejectionReason": null
    }
  ],
  "meta": { "page": 1, "perPage": 20, "total": 1 }
}
```

### POST `/testimonies/{id}/reactions` *(optional)*

Adds or toggles the current user's reaction. Kept simple (single "like"-style reaction);
extend with a `type` field if multiple reaction kinds are needed later.

**Request body** (optional)

```json
{ "type": "like" }
```

**Response**

```json
{ "data": { "id": "tst_9f2a1c", "reactionCount": 43, "hasReacted": true } }
```

`DELETE /testimonies/{id}/reactions` removes it and returns the decremented count.

---

## Entities

### Testimony

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `id` | string | no | Stable id (UUID or slug). |
| `title` | string | no | Headline; rendered uppercase on the card. |
| `body` | string | no | Full testimony text. |
| `excerpt` | string | yes | Server-truncated preview for feed cards. Optional. |
| `category` | string | yes | Category slug (e.g. `healing`, `finances`, `family`, `general`). |
| `authorName` | string | yes | Display name of the author. `null` / masked when `isAnonymous`. |
| `userId` | string | yes | Id of the submitting member. Omitted/masked when anonymous. |
| `isAnonymous` | boolean | no | Whether the author is hidden in public views. |
| `avatar` | string (URL) | yes | Author avatar; absolute URL. Null when anonymous or unset. |
| `media` | Media[] | yes | Attached images/video. Empty array when none. |
| `status` | enum | no | `pending` \| `approved` \| `rejected`. |
| `reactionCount` | integer | yes | Total reactions. Defaults to `0`. |
| `hasReacted` | boolean | yes | Whether the current user has reacted. Only if reactions ship. |
| `rejectionReason` | string | yes | Populated only on `mine` rows with `status = rejected`. |
| `createdAt` | string (ISO-8601) | no | When the member submitted it. |
| `approvedAt` | string (ISO-8601) | yes | When a moderator approved it; `null` while pending/rejected. |

### Media

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `type` | enum | no | `image` \| `video`. |
| `url` | string (URL) | no | Absolute asset URL. |
| `thumbnail` | string (URL) | yes | Optional poster/thumbnail for video. |

---

## Notes / open questions for backend

- **Moderation workflow.** Who approves submissions (staff/admin role, per-campus
  moderators)? Is there a separate admin API/dashboard, or should the app expose an
  in-app moderation surface for authorized users? Confirm the exact status set —
  is there an intermediate `in_review`, and can approved testimonies be un-published?
- **Anonymity.** When `isAnonymous = true`, the backend must strip `authorName`,
  `userId`, and `avatar` from all public responses (`/testimonies`, `/testimonies/{id}`)
  while still tracking ownership internally for `/testimonies/mine`. Confirm this masking
  happens server-side.
- **Media attachments.** The design shows avatars but no inline testimony media. If media
  is supported, confirm the upload flow: a two-step `POST /media` → returns `mediaId`,
  then reference `mediaIds` on submit? What are the size/type/count limits?
- **Reactions.** Are reactions in scope for v1, and is a single "like" enough or are
  multiple reaction types (e.g. praise, amen) required? Should the feed default sort be
  `recent` or `popular`?
- **Comments.** Not present in the design. Are threaded comments/encouragements planned?
  If so they warrant a separate `/testimonies/{id}/comments` resource.
- **Reporting / abuse.** Should approved testimonies be reportable by members
  (`POST /testimonies/{id}/reports`) to re-enter moderation?
- **Feed scoping.** The section header says "Testimonies from others this week." Is the
  home feed time-boxed (this week) and is there a separate all-time browse view? Confirm
  whether a `from`/`to` or `period` filter is needed, and whether the feed should exclude
  the current user's own approved testimonies.
- **Categories.** Is the category list fixed/server-driven? If dynamic, a
  `GET /testimonies/categories` lookup may be needed to populate the submit-flow picker
  and feed filters.
