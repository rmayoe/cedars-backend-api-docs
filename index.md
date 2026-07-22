---
title: Home
nav_order: 1
---

# Cedars / HOG — Backend API Requirements

This folder documents the backend APIs the mobile app (Cedars 2.0) needs but does **not
yet have**, derived from the product designs. Each feature is a separate doc so backend
engineers can pick up sections independently. Every doc opens with a plain-English
explanation of the feature — read that first to understand what you are building.

- Start with **[`_CONVENTIONS.md`](./_CONVENTIONS.md)** — the shared contract (transport,
  auth, multi-tenancy, envelope, pagination, errors, naming). Everything below assumes it.
- Every doc is structured the same way: **user actions → API table → endpoint detail
  (params + JSON) → entity field tables → open questions.**

> These are proposals grounded in the designs and the app's existing data layer
> (`ApiClient`, entities, remote datasources). Paths, field names, and shapes are
> suggestions — the "Open questions" in each doc flag where backend input is needed.

## Section docs

| # | Doc | Feature |
| - | --- | ------- |
| 1 | [catch-up-past-service.md](./catch-up-past-service.md) | Past Service / Catch Up — weekly missed-content feed + history |
| 2 | [book-of-the-month.md](./book-of-the-month.md) | Book of the Month — featured book, detail, read, recommendations |
| 3 | [streaming-sermons.md](./streaming-sermons.md) | Streaming / Sermons — live, campuses, ministers, sermon player |
| 4 | [notes.md](./notes.md) | Notes — user CRUD, sermon-attached, favourites, recently deleted |
| 5 | [explore-events.md](./explore-events.md) | Explore / Events — curated hub + all-events list |
| 6 | [testimonies.md](./testimonies.md) | Testimonies — public feed + user submission (moderated) |
| 7 | [birthdays.md](./birthdays.md) | Birthdays — today/upcoming list + send a wish |
| 8 | [discover-search.md](./discover-search.md) | Discover / Search — curated hub + unified cross-content search |

## All endpoints at a glance

Grouped by feature. `Bearer` = requires auth; `Optional` = works signed-out but
personalises when authed; **Required** = user-scoped. See each doc for full detail.

### Catch Up
| Method | Path | Description |
| ------ | ---- | ----------- |
| GET | `/catch-up/current` | Current-week feed (cards + Play) |
| GET | `/catch-up/items/{id}` | Item detail + playable media + resume position |
| GET | `/catch-up/history` | Past items grouped by week, paginated |
| POST | `/catch-up/items/{id}/progress` | Persist watched / resume state |

### Book of the Month
| Method | Path | Description |
| ------ | ---- | ----------- |
| GET | `/books/current` | Current month's featured book |
| GET | `/books` | Browse/list books (`?month`, `?q`, paginate) |
| GET | `/books/{id}` | Book detail |
| GET | `/books/recommended` · `/books/{id}/recommended` | Recommendation rails |
| GET | `/books/{id}/content` | Resolve read/download content (PDF/EPUB) |
| POST · DELETE | `/books/{id}/bookmark` | Save / unsave |
| POST | `/books/{id}/reaction` | Like / thumbs-up |

### Streaming / Sermons
| Method | Path | Description |
| ------ | ---- | ----------- |
| GET | `/stream/live` | Live status + featured stream & campus |
| GET | `/campuses` · `/campuses/{id}` | Campuses (`?featured`), with embedded live stream |
| GET | `/ministers` · `/ministers/{id}` | List/search ministers (`?q`, `?sort`), profile |
| GET | `/sermons` · `/sermons/{id}` | Search sermons (`?campusId`,`?ministerId`,`?q`,`?from`,`?to`); detail with media/notes/scripture |
| GET | `/ministers/{id}/sermons` · `/campuses/{id}/sermons` | Convenience aliases |
| GET | `/stream/library` · `/stream/history` | User library / recently played |
| POST · DELETE | `/sermons/{id}/favorite` · `/ministers/{id}/favorite` | Favorite / follow |
| PUT | `/sermons/{id}/progress` | Upsert playback progress |

### Notes
| Method | Path | Description |
| ------ | ---- | ----------- |
| GET · POST | `/notes` | List (filter/search) / create |
| GET · PATCH · DELETE | `/notes/{id}` | Read / update / soft-delete |

### Explore / Events
| Method | Path | Description |
| ------ | ---- | ----------- |
| GET | `/explore` | Aggregated hub feed (ordered sections) |
| GET | `/events` · `/events/{id}` | Event list (`?from`,`?to`,`?category`,`?campusId`,`?featured`) / detail |
| POST · DELETE | `/events/{id}/rsvp` | RSVP / cancel — **provisional** (no CTA in current design) |

### Testimonies
| Method | Path | Description |
| ------ | ---- | ----------- |
| GET | `/testimonies` · `/testimonies/{id}` | Approved feed / detail |
| POST | `/testimonies` | Submit (enters moderation, `status=pending`) |
| GET | `/testimonies/mine` | My submissions (any status) |
| POST · DELETE | `/testimonies/{id}/reactions` | React / unreact *(optional)* |

### Birthdays
| Method | Path | Description |
| ------ | ---- | ----------- |
| GET | `/birthdays` | Members with birthdays, grouped today/upcoming (`?scope`,`?campusId`) |
| GET | `/members/{id}` | Basic member profile |
| POST · GET | `/birthdays/{memberId}/wishes` | Send / list wishes |

### Discover / Search
| Method | Path | Description |
| ------ | ---- | ----------- |
| GET | `/discover` | Curated hub (trending, categories, suggestions) |
| GET | `/search` | Unified cross-content search (grouped; `?type` narrows) |
| GET | `/search/suggestions` | Autocomplete |
| GET · POST · DELETE | `/search/recent` | Server-side recent searches *(if not local-only)* |

## Cross-cutting entities (shared across sections)

Several entities appear in more than one feature. Define these **once** and reference them
everywhere to avoid drift:

| Entity | Owned by | Also used by |
| ------ | -------- | ------------ |
| **Sermon** (+ `SermonMedia`, `ScriptureRef`) | Streaming | Catch Up (items reference sermons), Notes (attach to sermon), Search |
| **Campus / Branch** | Streaming | Explore/Events, Birthdays, Catch Up (service origin), Search. Aligns with existing `BranchModel`/`BranchEntity` (`id`,`name`,`slug`,`socialMedia`,`coordinate`) |
| **Minister** | Streaming | Search, Sermon (author) |
| **Member / User profile** | Birthdays (`/members/{id}`) | Testimonies (author), Notes (owner), Explore. Aligns with existing `UserModel`/`UserEntity` |
| **Book** (+ `Author`, `BookContent`) | Book of the Month | Search |
| **Event** | Explore/Events | Search, Explore hub |
| **Testimony** | Testimonies | Explore hub strip, Search |
| **SearchResult** (`type`,`id`,`title`,`subtitle`,`image`,`deeplink`) | Discover/Search | wraps all of the above for the unified index |

**Recommendation:** stand up **Sermon**, **Campus**, **Minister**, and **Member/User**
first — they are the backbone the other features (Catch Up, Notes, Search, Explore) hang
off of.

## Cross-cutting open questions for the backend team

Recurring themes raised independently across multiple docs:

1. **Media delivery & signing** — streaming protocol for live + VOD (HLS / RTMP / YouTube /
   Mixlr embed?), and whether `audioUrl`/`videoUrl`/`content` URLs are signed & expiring.
   Affects Catch Up, Streaming, Book content. *(Docs 1, 2, 3)*
2. **Playback / reading progress** — is per-user resume/watched/read state persisted
   server-side (cross-device) or local-only? Same shape needed for Catch Up, Sermons, Books.
   *(Docs 1, 2, 3)*
3. **Search backend** — DB `LIKE` vs full-text vs external (Algolia/Elasticsearch); which
   content types are indexed; private-note scoping; ranking & typo tolerance. *(Doc 8)*
4. **Curated hub composition** — are `/explore` and `/discover` sections editor-curated or
   derived? Who configures ordering per tenant/campus? *(Docs 5, 8)*
5. **User-generated content moderation** — Testimonies (and any note/wish sharing) approval
   workflow, anonymity handling, abuse reporting. *(Doc 6)*
6. **Multi-tenancy & campus scoping** — every list (events, birthdays, ministers, sermons)
   needs a clear default scope: tenant-wide vs the user's home campus, and how `?campusId`
   overrides it. *(Docs 3, 5, 7)*
7. **Privacy / opt-in** — birthdays and member profiles must respect a visibility/opt-in
   flag; year vs month-day only for birthdays. *(Doc 7)*
8. **Recurring/timezone semantics** — events recurrence and timezone; birthday grouping
   boundaries ("this week"/"this month"); catch-up "week" boundary (WK14). *(Docs 1, 5, 7)*

---

*Derived from the product designs; no app code was modified. Each section doc follows
[`_CONVENTIONS.md`](./_CONVENTIONS.md).*
