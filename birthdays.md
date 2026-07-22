---
title: Birthdays
nav_order: 9
---

# Birthdays

Birthdays is the part of the app that shows which church members are celebrating a birthday
today, this week, or this month. It pulls members from the directory, figures out whose
birthday falls inside the chosen time window, and presents them as a simple list. The point
is to make it easy for the community to notice and acknowledge each other's birthdays.

A user opens the feature and sees the members with upcoming birthdays. They can narrow the
list by time window (today, this week, this month) and, in churches with more than one
campus, by campus. Tapping a member opens a small profile, from which the user can send that
member a birthday wish — a short message that starts from a default greeting and can be
edited before it is sent. The "celebrate" step shows the member's photo, a header, the
editable message field (pre-filled with the default greeting), and a send action. Behind the
scenes the app calls a few endpoints: one to fetch the grouped list of birthdays, one to
fetch a member's basic profile, and one to record and deliver the wish.

This doc follows `_CONVENTIONS.md`.

## Screens & user actions

| Screen | User action |
| ------ | ----------- |
| Birthdays list | View members with birthdays, grouped by **Today** and **Upcoming** (this week / this month). |
| Birthdays list | Toggle scope filter: **Today / This week / This month**. |
| Birthdays list | (Multi-campus) filter by campus. |
| Birthdays list | See each member's avatar, name, birthday date, campus, and a "today" highlight. |
| Birthdays list | Tap a member to open their basic profile. |
| Celebrate overlay | Compose a birthday message in a text field (default greeting pre-filled). |
| Celebrate overlay | Tap **Send Message** to deliver the wish to that member. |
| Celebrate overlay | Dismiss without sending (close button). |

## APIs required

| Method | Path | Auth | Description |
| ------ | ---- | ---- | ----------- |
| GET | `/birthdays` | Bearer | List members with birthdays, grouped into `today` / `upcoming`. Supports scope + campus filters. |
| GET | `/members/{id}` | Bearer | Basic public profile for a member (name, avatar, campus, birthday). |
| POST | `/birthdays/{memberId}/wishes` | Bearer | Send a birthday wish/message to a member. |
| GET | `/birthdays/{memberId}/wishes` | Bearer | *(Optional)* List wishes a member has received (if a wall of wishes is shown). |

## Endpoint detail

### GET `/birthdays`

Returns members whose birthday falls within the requested scope, split into a `today`
bucket and an `upcoming` bucket so the client can render the two groups directly.

**Query params**

| Param | Type | Required | Default | Description |
| ----- | ---- | -------- | ------- | ----------- |
| `scope` | enum `today` \| `week` \| `month` | no | `month` | Window of birthdays to return. `today` = today only; `week` = current calendar week; `month` = current calendar month. |
| `campusId` | string | no | — | Restrict to a single campus. Omit for all campuses the tenant exposes. |
| `page` | int | no | `1` | Pagination (mainly relevant for `month`). |
| `perPage` | int | no | `20` | Page size. |

**Response** — `200 OK`

```json
{
  "data": {
    "today": [
      {
        "userId": "usr_7Hk2",
        "name": "Toheeb Adewale",
        "avatar": "https://cdn.hog.app/avatars/usr_7Hk2.jpg",
        "birthday": "07-22",
        "campus": { "id": "cmp_lag", "name": "Lagos" },
        "isToday": true,
        "hasWished": false
      }
    ],
    "upcoming": [
      {
        "userId": "usr_9Qm4",
        "name": "Grace Okonkwo",
        "avatar": "https://cdn.hog.app/avatars/usr_9Qm4.jpg",
        "birthday": "07-25",
        "campus": { "id": "cmp_lag", "name": "Lagos" },
        "isToday": false,
        "hasWished": false
      },
      {
        "userId": "usr_3Zp1",
        "name": "David Eze",
        "avatar": null,
        "birthday": "07-30",
        "campus": { "id": "cmp_abv", "name": "Abuja" },
        "isToday": false,
        "hasWished": true
      }
    ]
  },
  "meta": { "page": 1, "perPage": 20, "total": 3, "scope": "month" }
}
```

Notes:
- When `scope=today`, `upcoming` may be empty (or omitted).
- `upcoming` should be sorted by nearest birthday first.
- `hasWished` reflects whether the **authenticated user** has already sent a wish to that
  member in the current birthday cycle (drives a "sent" state on the celebrate action).

### GET `/members/{id}`

Basic profile shown when a birthday member is tapped. Keep this lightweight and
privacy-safe (no email/phone unless the member opted in).

**Path params**

| Param | Type | Description |
| ----- | ---- | ----------- |
| `id` | string | Member/user id (matches `userId` from `/birthdays`). |

**Response** — `200 OK`

```json
{
  "data": {
    "userId": "usr_7Hk2",
    "name": "Toheeb Adewale",
    "avatar": "https://cdn.hog.app/avatars/usr_7Hk2.jpg",
    "birthday": "07-22",
    "campus": { "id": "cmp_lag", "name": "Lagos" },
    "bio": "Media team volunteer.",
    "isBirthdayToday": true
  }
}
```

### POST `/birthdays/{memberId}/wishes`

Sends a birthday message from the authenticated user to `memberId`. Backs the **Send
Message** button on the celebrate overlay.

**Path params**

| Param | Type | Description |
| ----- | ---- | ----------- |
| `memberId` | string | Recipient member's `userId`. |

**Request body**

```json
{
  "message": "Happy birthday Toheeb! 🎉 Have a blessed year ahead."
}
```

| Field | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `message` | string | yes | Free-text wish. Server should enforce a max length and sanitize. A sensible default greeting is pre-filled client-side; user may edit it. |

**Response** — `201 Created`

```json
{
  "data": {
    "id": "wsh_5Abc",
    "from": { "userId": "usr_self", "name": "Mayowa A." },
    "to": { "userId": "usr_7Hk2", "name": "Toheeb Adewale" },
    "message": "Happy birthday Toheeb! 🎉 Have a blessed year ahead.",
    "createdAt": "2026-07-22T09:14:00Z"
  }
}
```

Errors: `404 NOT_FOUND` (member/id invalid), `409 CONFLICT` (already wished this cycle, if
duplicate wishes are disallowed), `422` (empty/too-long message).

## Entities

### BirthdayMember

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `userId` | string | no | Member/user id. Used to open the profile and to send a wish. |
| `name` | string | no | Display name. |
| `avatar` | string (URL) | yes | Absolute avatar URL; null → client shows initials/placeholder. |
| `birthday` | string `MM-DD` | no | Month-day only (no year — see open questions). |
| `campus` | object `{ id, name }` | yes | Member's campus; null for single-campus tenants. |
| `isToday` | bool | no | True if the birthday is today (in the tenant/member timezone). Drives the highlight. |
| `hasWished` | bool | yes | Whether the authenticated user already sent a wish this cycle. |

### BirthdayWish

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `id` | string | no | Wish id. |
| `from` | object `{ userId, name }` | no | Sender (the authenticated user). |
| `to` | object `{ userId, name }` | no | Recipient (birthday member). |
| `message` | string | no | The wish text. |
| `createdAt` | string (ISO-8601 UTC) | no | When the wish was sent. |

## Notes / open questions for backend

- **Privacy / opt-in.** Do members opt in to having their birthday shown publicly? If so,
  `/birthdays` must exclude opted-out members, and there should be a settings toggle
  (likely owned by the profile/settings feature, not this one).
- **Year vs month-day.** Design shows only a celebratory name, not an age. Assume the API
  stores birthdays as `MM-DD` only (or withholds the year) to avoid exposing age. Confirm
  whether age/year is ever needed.
- **Grouping boundaries & timezone.** Define "today", "this week" (calendar week vs rolling
  7 days), and "this month" precisely, and in which timezone (tenant vs member local). This
  affects both `isToday` and which bucket a member lands in.
- **Wish delivery.** Is a wish delivered in-app only (a received-wishes list/wall), or also
  as a push notification / message to the recipient? The overlay only proves compose+send;
  confirm the recipient-side surface.
- **Duplicate wishes.** Can a user send multiple wishes to the same member in one cycle, or
  is it one-per-cycle (drives `hasWished` and the `409` case)?
- **Birthday notifications.** Should users get a push/notification on mornings when someone
  they know (or everyone at their campus) has a birthday? Out of scope for these endpoints
  but likely a related requirement.
- **Default message.** The compose field is pre-filled — is the default greeting
  client-side static text, or should it come from the server (tenant-configurable)?
- **Pagination.** `today` is typically small; `month` can be large — paginate `upcoming`
  and keep `today` unpaginated, or paginate the whole list? Current design assumes the
  former.
- **Reactions.** The task mentions "react"; the design only shows a text message. If a
  lightweight emoji reaction is wanted separately from a full message, consider a `type`
  field on the wish (`message` | `reaction`) plus an `emoji` field.
