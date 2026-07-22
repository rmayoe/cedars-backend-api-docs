---
title: Streaming & Sermons
nav_order: 5
---

# Streaming & Sermons

The Stream tab is where people watch and listen to church services and sermons. It is the
largest feature in the app. When a service is being broadcast, the tab shows it live; the
rest of the time it acts as a catalogue of past sermons (audio and video) that a user can
play on demand, download for offline use, favorite, and resume where they left off.

The tab has a few connected sub-flows. The home view shows whatever is live right now (or a
countdown to the next service) plus rails of recent and newly uploaded sermons. The church
runs in more than one physical location ("campuses"), so a user can open a single campus to
see that campus' own live stream, its most-streamed sermons, and its ministers. Ministers
(pastors, speakers, artists) have a searchable directory: a user can scroll or type to find
one, then open that minister's profile to read their bio, follow them, and browse the sermons
they have preached. Opening any sermon — from a rail, a campus, a minister, or a search —
loads a detail/player view with the media, notes, and scripture references for that sermon.

Mechanically, the backend serves mostly read endpoints (live status, campuses, ministers,
sermons) that anyone can call, plus a set of user-scoped endpoints (library, favorites,
follows, and playback progress) that require a signed-in user. When a valid token is present,
the read endpoints additionally hydrate per-user flags such as whether the user has favorited
an item or how far they have played it. A sermon's list rows are lightweight; the full media
URLs and notes are fetched only when the player opens.

> This doc follows the conventions in [`_CONVENTIONS.md`](./_CONVENTIONS.md).

---

## 1. Screens & user actions

### 1a. Stream home — "We are Live"
The landing screen of the Stream tab.

| Area | User action | Data needed |
| ---- | ----------- | ----------- |
| Live hero | Sees the featured/current service with a countdown badge ("We are Live in 20Min :30s") and a live indicator | Live/streaming status + featured stream |
| Service caption | Reads service title ("COMMUNION SERVICE \|\| 6TH MAY, 2026") + campus name + campus logo | Featured campus summary |
| **Join Livestream** button | Taps to open the live player | LiveStream playback URL (`hlsUrl`/`videoUrl`) |
| Your Library card | Opens Downloads, Favorites, Listening History, Ministers | Per-user library collections (see §7) |
| Recently played (tabs: Sermons / Livestreams) | Scrolls their recent sermons/livestreams; taps ❤ to favorite; taps ⋯ for actions | Playback history + per-user favorite flags |
| Newly Uploaded | Horizontal cards (title, date, play) → open sermon | Latest sermons list |
| Featured Campuses | Taps a campus logo → campus screen | Campus list (featured) |

### 1b. Featured campus
Reached by tapping a campus logo. Shows one campus and everything it streams.

| Area | User action | Data needed |
| ---- | ----------- | ----------- |
| Campus header | Sees campus hero image, logo, name, full address | `getCampus(id)` |
| About Us | Reads campus description + social channels + lead pastor | Campus `description`, `socialLinks` |
| Live now (if streaming) | Joins this campus' live stream | Campus embedded `liveStream` |
| Top Streamed Sermons | Taps a sermon → player | `listSermons?campusId=` (sorted by plays) |
| Top Streamed Videos & Livestreams | Taps a video card → player | Sermons/streams filtered to campus, video media |
| Devotionals Shared This Month | Taps a devotional | (cross-links to Devotionals section) |
| Featured Profiles | Taps a minister → profile | Ministers for this campus |
| Featured Campuses | Switches to another campus | Campus list |

### 1c. Ministers list + search
A full directory of ministers.

| Area | User action | Data needed |
| ---- | ----------- | ----------- |
| Ministers list | Scrolls all ministers (avatar, name, campus); taps ❤ to favorite | `listMinisters` (paginated) |
| Sort toggle (↑↓) | Re-orders the list (A–Z / most-streamed) | `?sort=` param |
| Search pill (keyboard) | Types a query to filter ministers live | `listMinisters?q=` |
| Row tap | Opens a minister profile | `getMinister(id)` |

### 1d. Minister profile
A single minister and their sermons.

| Area | User action | Data needed |
| ---- | ----------- | ----------- |
| Minister header | Sees hero image, avatar, name, bio | `getMinister(id)` |
| Connect with … | Taps social links (Instagram/Facebook/Twitter/Mixlr) | Minister `socialLinks` |
| Top Streamed Sermons | Taps a sermon → player; taps ⬇ to download; ❤ to favorite | `listSermons?ministerId=` |
| Featured Profiles | Jumps to another minister | Ministers (featured/related) |
| Featured Campuses | Jumps to a campus | Campus list |

### 1e. Previous sermon — detail / player
Opened from any sermon row or "Newly Uploaded"/"Top Streamed" card.

| Area | User action | Data needed |
| ---- | ----------- | ----------- |
| Player | Plays audio or video, scrubs, pauses | `getSermon(id)` → `media` with `audioUrl`/`videoUrl`/`hlsUrl` |
| Metadata | Reads title, minister, campus, date, series/week | Sermon fields |
| Notes / description | Reads sermon notes and scripture references | `notes`, `scriptureRefs` |
| Download ⬇ | Saves for offline | Downloadable media URL |
| Favorite ❤ | Toggles favorite | Favorite endpoint (§7) |
| Resume | Continues from last position | Playback progress (§8) |

---

## 2. APIs required

| Method | Path | Auth | Description |
| ------ | ---- | ---- | ----------- |
| GET | `/stream/live` | Optional | Current live/streaming status + the featured live stream and its campus. |
| GET | `/campuses` | Optional | List campuses (supports `?featured=true`). |
| GET | `/campuses/{id}` | Optional | One campus, embedding its current `liveStream` if any. |
| GET | `/ministers` | Optional | List / search ministers (`?q=`, `?campusId=`, `?sort=`). |
| GET | `/ministers/{id}` | Optional | One minister; optionally embeds their top sermons. |
| GET | `/ministers/{id}/sermons` | Optional | Sermons by a minister (convenience alias of `/sermons?ministerId=`). |
| GET | `/sermons` | Optional | List / search sermons with filters (`?q`, `?campusId`, `?ministerId`, `?from`, `?to`, `?sort`, `?type`). |
| GET | `/sermons/{id}` | Optional | One sermon with full media (`audioUrl`/`videoUrl`/`hlsUrl`), notes, scripture refs. |
| GET | `/campuses/{id}/sermons` | Optional | Sermons for a campus (alias of `/sermons?campusId=`). |
| GET | `/stream/library` | **Required** | The signed-in user's library (downloads, favorites, history, followed ministers). |
| POST | `/sermons/{id}/favorite` | **Required** | Favorite a sermon (idempotent). |
| DELETE | `/sermons/{id}/favorite` | **Required** | Un-favorite a sermon. |
| POST | `/ministers/{id}/favorite` | **Required** | Follow/favorite a minister. |
| DELETE | `/ministers/{id}/favorite` | **Required** | Unfollow a minister. |
| PUT | `/sermons/{id}/progress` | **Required** | Upsert playback progress for the current user (optional; see §8). |
| GET | `/stream/history` | **Required** | Recently played (sermons + livestreams), most-recent first. |

> Anonymous browsing is expected (the app can show public catalogue before login), so the
> read endpoints are `Optional` auth — but when a valid token is present the response should
> hydrate per-user flags (`isFavorite`, `progress`). Library/favorite/progress/history are
> user-scoped and therefore require auth.

---

## 3. Endpoint detail

### GET `/stream/live`
Drives the hero + countdown on the Stream home. Returns the currently featured stream and
whether it is live now, about to go live, or offline (VOD only).

Response:
```json
{
  "data": {
    "status": "upcoming",
    "startsAt": "2026-05-06T18:00:00Z",
    "countdownSeconds": 1230,
    "stream": {
      "id": "ls_9f2c",
      "title": "COMMUNION SERVICE || 6TH MAY, 2026",
      "isLive": false,
      "protocol": "hls",
      "hlsUrl": "https://stream.cedars.app/live/main/index.m3u8",
      "videoUrl": null,
      "youtubeUrl": null,
      "thumbnail": "https://cdn.cedars.app/live/communion.jpg",
      "campus": {
        "id": "cmp_cedars_hog",
        "name": "Cedars House of Grace",
        "slug": "CEDARS",
        "image": "https://cdn.cedars.app/campus/cedars-logo.png"
      },
      "socialCaption": "Follow us on other social media channels — Instagram: /wearecedars …"
    }
  }
}
```
`status` ∈ `live` | `upcoming` | `offline`. When `offline`, `stream` may be `null` (app hides
the hero) or hold the most recent VOD.

### GET `/campuses`
| Param | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `featured` | boolean | no | Only campuses flagged for the "Featured Campuses" row. |
| `q` | string | no | Filter by name/location. |
| `page`, `perPage` | int | no | Pagination. |

```json
{
  "data": [
    { "id": "cmp_cedars_hog", "name": "Cedars House of Grace", "slug": "CEDARS",
      "image": "https://cdn.cedars.app/campus/cedars-logo.png",
      "location": "Lekki, Lagos", "isLive": true, "featured": true }
  ],
  "meta": { "page": 1, "perPage": 20, "total": 6 }
}
```

### GET `/campuses/{id}`
| Param | Type | In | Description |
| ----- | ---- | -- | ----------- |
| `id` | string | path | Campus id or slug. |

Embeds the campus' current live stream (if any) so the campus screen can offer "Join live"
without a second call.
```json
{
  "data": {
    "id": "cmp_cedars_hog",
    "name": "Cedars House of Grace",
    "slug": "CEDARS",
    "image": "https://cdn.cedars.app/campus/cedars-logo.png",
    "heroImage": "https://cdn.cedars.app/campus/cedars-hero.jpg",
    "address": "Second Roundabout, Lekki, Block 2, Plot 3, Okunde Bluewater Zone, Off Remi Olowude St, Lekki",
    "location": "Lekki, Lagos",
    "description": "Lorem ipsum dolor sit amet consectetur…",
    "leadPastor": { "id": "min_eboda", "name": "Gbeminiyi Eboda" },
    "socialLinks": {
      "instagram": "https://instagram.com/wearecedars",
      "facebook": "https://facebook.com/wearecedars",
      "twitter": "https://twitter.com/wearecedars_",
      "mixlr": "https://wearecedars.mixlr.com/"
    },
    "coordinate": { "lat": 6.4413, "lng": 3.5426 },
    "liveStream": null,
    "featured": true
  }
}
```

### GET `/ministers`
Powers both the directory and the search pill.

| Param | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `q` | string | no | Free-text search on minister name (live filter as the user types). |
| `campusId` | string | no | Ministers at a given campus (for "Featured Profiles"). |
| `featured` | boolean | no | Only featured ministers. |
| `sort` | enum | no | `name_asc` (default), `name_desc`, `most_streamed` — backs the ↑↓ toggle. |
| `page`, `perPage` | int | no | Pagination. |

```json
{
  "data": [
    { "id": "min_marvin", "name": "Marvin McKinney",
      "avatar": "https://cdn.cedars.app/ministers/marvin.jpg",
      "campus": { "id": "cmp_alpha", "name": "Alpha Cathedral" },
      "isFavorite": false, "sermonCount": 24 },
    { "id": "min_eboda", "name": "Gbeminiyi Adetola",
      "avatar": "https://cdn.cedars.app/ministers/eboda.jpg",
      "campus": { "id": "cmp_cedars_hog", "name": "Cedars Cathedral" },
      "isFavorite": true, "sermonCount": 58 }
  ],
  "meta": { "page": 1, "perPage": 20, "total": 41 }
}
```

### GET `/ministers/{id}`
| Param | Type | In | Description |
| ----- | ---- | -- | ----------- |
| `id` | string | path | Minister id. |
| `includeSermons` | boolean | query | When `true`, embed up to `sermonsLimit` top sermons (default 5) so the profile renders in one call. |

```json
{
  "data": {
    "id": "min_marvin",
    "name": "Marvin McKinney",
    "avatar": "https://cdn.cedars.app/ministers/marvin.jpg",
    "heroImage": "https://cdn.cedars.app/ministers/marvin-hero.jpg",
    "bio": "Lorem ipsum dolor sit amet consectetur…",
    "role": "Pastor",
    "campus": { "id": "cmp_alpha", "name": "Alpha Cathedral" },
    "socialLinks": {
      "instagram": "https://instagram.com/wearecedars",
      "facebook": "https://facebook.com/wearecedars",
      "twitter": "https://twitter.com/wearecedars_",
      "mixlr": "https://wearecedars.mixlr.com/"
    },
    "isFavorite": false,
    "sermonCount": 24,
    "sermons": [ /* Sermon objects, present only when includeSermons=true */ ]
  }
}
```

### GET `/sermons`
The catalogue behind "Recently played", "Newly Uploaded", "Top Streamed Sermons", campus
lists, minister lists, and any search.

| Param | Type | Required | Description |
| ----- | ---- | -------- | ----------- |
| `q` | string | no | Full-text search (title, minister, description, scripture). |
| `campusId` | string | no | Filter to a campus. |
| `ministerId` | string | no | Filter to a minister. |
| `type` | enum | no | `sermon` \| `livestream` \| `video` \| `audio` — backs the Sermons/Livestreams tab. |
| `from` | date | no | ISO date lower bound (`publishedAt >= from`). |
| `to` | date | no | ISO date upper bound. |
| `sort` | enum | no | `recent` (default), `most_streamed`, `oldest`. |
| `featured` | boolean | no | Only featured/"Newly Uploaded". |
| `page`, `perPage` | int | no | Pagination. |

```json
{
  "data": [
    {
      "id": "srm_1042",
      "title": "Monday Morning Prayer with Rev",
      "minister": { "id": "min_eboda", "name": "Rev. Gbeminiyi Eboda" },
      "campus": { "id": "cmp_cedars_hog", "name": "Cedars House of Grace" },
      "thumbnail": "https://cdn.cedars.app/sermons/1042.jpg",
      "type": "audio",
      "durationSeconds": 2415,
      "durationLabel": "40:15",
      "publishedAt": "2026-04-26T07:00:00Z",
      "seriesLabel": "WK14",
      "playCount": 1820,
      "isFavorite": true
    }
  ],
  "meta": { "page": 1, "perPage": 20, "total": 214 }
}
```
List rows are lightweight (no full media URLs / notes) — fetch `/sermons/{id}` when opening
the player.

### GET `/sermons/{id}`
Full detail for the player. Returns every media rendition available so the client can pick
audio vs video and the right streaming protocol.

| Param | Type | In | Description |
| ----- | ---- | -- | ----------- |
| `id` | string | path | Sermon id. |

```json
{
  "data": {
    "id": "srm_1042",
    "title": "Winning in Life",
    "description": "Lorem ipsum dolor sit amet consectetur…",
    "minister": { "id": "min_eboda", "name": "Rev. Gbeminiyi Eboda",
                  "avatar": "https://cdn.cedars.app/ministers/eboda.jpg" },
    "campus": { "id": "cmp_cedars_hog", "name": "Cedars House of Grace",
                "image": "https://cdn.cedars.app/campus/cedars-logo.png" },
    "thumbnail": "https://cdn.cedars.app/sermons/1042.jpg",
    "type": "video",
    "publishedAt": "2026-04-26T07:00:00Z",
    "seriesLabel": "WK14",
    "durationSeconds": 2415,
    "durationLabel": "40:15",
    "playCount": 1820,
    "isFavorite": true,
    "isDownloadable": true,
    "media": {
      "audioUrl": "https://cdn.cedars.app/sermons/1042.mp3",
      "videoUrl": "https://cdn.cedars.app/sermons/1042-720p.mp4",
      "hlsUrl": "https://stream.cedars.app/vod/1042/index.m3u8",
      "youtubeUrl": null,
      "mimeType": "video/mp4",
      "fileSizeBytes": 84213760
    },
    "notes": "Full sermon notes / outline in markdown or html…",
    "scriptureRefs": [
      { "reference": "Philippians 4:13", "book": "Philippians", "chapter": 4, "verse": "13" },
      { "reference": "Romans 8:37", "book": "Romans", "chapter": 8, "verse": "37" }
    ],
    "progress": { "positionSeconds": 640, "completed": false, "updatedAt": "2026-07-20T21:04:00Z" }
  }
}
```
`progress` is present only for authenticated users who have played the sermon (see §8).

### GET `/stream/library` *(auth)*
Backs the "Your Library" card. Returns the four collections (or counts + first page each).
```json
{
  "data": {
    "downloads": { "count": 12, "items": [ /* Sermon summaries */ ] },
    "favorites": { "count": 30, "items": [ /* Sermon summaries */ ] },
    "history":   { "count": 88, "items": [ /* Sermon summaries, most-recent first */ ] },
    "ministers": { "count": 5,  "items": [ /* Minister summaries */ ] }
  }
}
```

### POST / DELETE `/sermons/{id}/favorite`, `/ministers/{id}/favorite` *(auth)*
Idempotent toggles; no request body. Return the updated resource or `{ "data": { "isFavorite": true } }`.

---

## 4. Entity — `LiveStream`

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `id` | string | no | Stream id. |
| `title` | string | no | Service/stream title, e.g. "COMMUNION SERVICE \|\| 6TH MAY, 2026". |
| `isLive` | boolean | no | Currently broadcasting. |
| `status` | enum | no | `live` \| `upcoming` \| `offline`. |
| `startsAt` | string (ISO-8601) | yes | Scheduled start (for the countdown). |
| `countdownSeconds` | int | yes | Convenience seconds-until-start; server-computed. |
| `protocol` | enum | yes | `hls` \| `rtmp` \| `youtube` \| `mixlr` — how to play (see open questions). |
| `hlsUrl` | string (URL) | yes | HLS manifest for live playback. |
| `videoUrl` | string (URL) | yes | Progressive MP4 fallback. |
| `youtubeUrl` | string (URL) | yes | YouTube watch/embed URL if the stream is on YouTube. |
| `thumbnail` | string (URL) | yes | Hero/poster image. |
| `campus` | Campus (summary) | yes | Campus that owns the stream. |
| `socialCaption` | string | yes | Free-text social-follow blurb shown under the caption. |

## 5. Entity — `Campus`
Aligns with the app's existing `BranchModel` (`slug` is load-bearing for tenancy — the same
backend serves every brand; the running app keeps only campuses whose `slug` it claims).

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `id` | string | no | Campus id. |
| `name` | string | no | Display name, e.g. "Cedars House of Grace". |
| `slug` | string | no | Tenant/brand key, uppercase (`CEDARS`, `HILLTOP`). |
| `image` | string (URL) | yes | Logo. |
| `heroImage` | string (URL) | yes | Large header image on the campus screen. |
| `location` | string | yes | Short location, e.g. "Lekki, Lagos". |
| `address` | string | yes | Full postal address. |
| `description` | string | yes | "About Us" body. |
| `leadPastor` | Minister (summary) | yes | Lead pastor. |
| `socialLinks` | map<string,string> | yes | Platform → URL (only platforms the campus has). |
| `coordinate` | { lat, lng } | yes | Map pin. |
| `email` | string | yes | Contact email. |
| `website` | string (URL) | yes | Campus website. |
| `contacts` | string[] | no | Phone numbers (empty array, never null). |
| `isLive` | boolean | no | Whether this campus is streaming now. |
| `liveStream` | LiveStream | yes | Embedded on `getCampus`; the campus' current stream. |
| `featured` | boolean | no | Appears in "Featured Campuses". |

## 6. Entity — `Minister`

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `id` | string | no | Minister id. |
| `name` | string | no | Display name, e.g. "Marvin McKinney". |
| `avatar` | string (URL) | yes | Circular thumbnail. |
| `heroImage` | string (URL) | yes | Large header image on the profile. |
| `bio` | string | yes | Biography / description. |
| `role` | string | yes | Title, e.g. "Pastor", "Artist". |
| `campus` | Campus (summary) | yes | Home campus (name shown under the row). |
| `socialLinks` | map<string,string> | yes | "Connect with …" links (instagram/facebook/twitter/mixlr). |
| `sermonCount` | int | yes | Number of sermons (for sorting/labels). |
| `isFavorite` | boolean | no | Whether the current user follows/favorites them (auth only; default false). |
| `featured` | boolean | no | Appears in "Featured Profiles". |
| `sermons` | Sermon[] | yes | Embedded only when `includeSermons=true`. |

## 7. Entity — `Sermon` (+ `SermonMedia`)
Supersedes the legacy `Episode` model (`id`, `title`, `minister`, `date`, `url`→audio,
`content`→description, `image`, `duration`). New fields are camelCase and structured.

### `Sermon`
| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `id` | string | no | Sermon id. |
| `title` | string | no | Sermon title. |
| `description` | string | yes | Short description/summary (list + detail). |
| `minister` | Minister (summary) | yes | Speaker. |
| `campus` | Campus (summary) | yes | Owning campus. |
| `thumbnail` | string (URL) | yes | Cover art. |
| `type` | enum | no | `audio` \| `video` \| `livestream`. |
| `publishedAt` | string (ISO-8601) | no | Publish/recorded date (the "SUN 26TH APRIL '26" label). |
| `seriesLabel` | string | yes | Series/week tag, e.g. "WK14". |
| `durationSeconds` | int | yes | Length in seconds. |
| `durationLabel` | string | yes | Display duration, e.g. "40:15". |
| `playCount` | int | yes | Stream count (drives "Top Streamed" sort). |
| `isFavorite` | boolean | no | Current-user favorite flag (auth only; default false). |
| `isDownloadable` | boolean | no | Whether offline download is allowed. |
| `media` | SermonMedia | yes | Playback URLs; present on `getSermon`, omitted from list rows. |
| `notes` | string | yes | Full sermon notes/outline (markdown or html). |
| `scriptureRefs` | ScriptureRef[] | yes | Bible references cited in the sermon. |
| `progress` | Progress | yes | Current user's playback position (auth only). |

### `SermonMedia`
| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `audioUrl` | string (URL) | yes | Progressive audio (mp3). At least one of audio/video/hls expected. |
| `videoUrl` | string (URL) | yes | Progressive video (mp4). |
| `hlsUrl` | string (URL) | yes | HLS manifest for adaptive streaming. |
| `youtubeUrl` | string (URL) | yes | YouTube URL if hosted there. |
| `mimeType` | string | yes | e.g. `audio/mpeg`, `video/mp4`. |
| `fileSizeBytes` | int | yes | For download progress/size display. |

### `ScriptureRef`
| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `reference` | string | no | Human-readable ref, e.g. "Philippians 4:13". |
| `book` | string | yes | Book name. |
| `chapter` | int | yes | Chapter number. |
| `verse` | string | yes | Verse or verse-range, e.g. "13" or "13-15". |

## 8. Entity — `Progress` (playback, optional)
Enables cross-device resume from "Listening History".

| Field | Type | Nullable | Description |
| ----- | ---- | -------- | ----------- |
| `positionSeconds` | int | no | Last playback position. |
| `completed` | boolean | no | Whether the user finished it. |
| `updatedAt` | string (ISO-8601) | no | When progress was last saved. |

**PUT `/sermons/{id}/progress`** body:
```json
{ "positionSeconds": 640, "completed": false }
```
Server upserts per (user, sermon) and returns the stored `Progress`. The client will call
this on pause, seek, and app background (debounced), so it must be cheap and idempotent.

---

## 9. Notes / open questions for backend

1. **Streaming protocol.** The live hero + "Join Livestream" and VOD players need one
   agreed protocol. Design references Mixlr, YouTube, and a generic live view. Please confirm
   whether live is **HLS**, **RTMP** (needs a player), a **YouTube embed**, or **Mixlr** — the
   `protocol` field and which of `hlsUrl`/`videoUrl`/`youtubeUrl` is authoritative depend on it.
   Ideally standardise VOD on **HLS** (`hlsUrl`) with an MP3/MP4 fallback.
2. **Live vs VOD identity.** Is a livestream that has ended promoted into the `/sermons`
   catalogue (a "previous sermon"), or does it stay a separate `LiveStream`? The Recently
   Played tabs (Sermons / Livestreams) imply both live and recorded items co-exist — confirm
   whether `type=livestream` sermons and `LiveStream` are the same records or distinct.
3. **Search backend.** `?q=` on `/ministers` and `/sermons` should be server-side full-text
   (title, minister, description, scripture) so the search pill filters live. The legacy API
   exposed `/sermons/search?search=` returning bare id arrays — please replace with a unified
   `?q=` returning full rows + `meta`. Is a single global `/search` (across ministers, sermons,
   campuses) also wanted, or per-resource only?
4. **Sort semantics.** "Top Streamed" needs a reliable `playCount`; the ↑↓ toggle on the
   Ministers list needs `sort=name_asc|name_desc|most_streamed`. Confirm play counts are tracked
   server-side and whether they are tenant-scoped.
5. **Progress sync.** Playback progress (§8) is marked optional. Do you want per-user
   cross-device resume, or is local-only acceptable for v1? If server-side, confirm the debounce
   contract and whether `history` is derived from progress writes.
6. **Library collections.** Downloads are inherently client-side (offline files), but
   Favorites, History, and followed Ministers should be server-backed for cross-device parity.
   Confirm whether `/stream/library` should return Downloads at all or only the server-owned
   three.
7. **Featured curation.** `featured` on campuses/ministers/sermons and the "Newly Uploaded" /
   "Top Streamed" / "Devotionals Shared This Month" rails — are these editorially curated
   (admin-flagged) or purely algorithmic (recency / play count)? This affects whether we need
   dedicated `?featured=true` flags or just `sort=` + `limit`.
8. **Tenancy of ministers/sermons.** Campuses map to `BranchModel.slug`. Are ministers and
   sermons filtered by the same tenant/brand token, and can a minister belong to multiple
   campuses (the list shows one campus per row, but "Featured Profiles" spans campuses)?
9. **Notes format.** Confirm whether `sermon.notes` is markdown, HTML, or plain text, and
   whether scripture refs are structured (as modelled) or must be parsed from the notes body.
