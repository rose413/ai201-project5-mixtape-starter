## Codebase Map

### Top-Level Structure

The app is a Flask API organized into three layers: route handlers, service functions, and SQLAlchemy models. All business logic lives in `services/`; routes only parse input and format responses.

Pattern I noticed: every route function is essentially a one-liner — it extracts arguments from the request, calls a single service function, and returns the result as JSON. There's no business logic in any route file. This makes it easy to find where a feature actually lives: look at the route to find the service function, then read the service function for everything that matters.

```
app.py          — Flask app factory; registers blueprints; initializes DB
models.py       — All SQLAlchemy models and association tables
routes/         — Blueprint definitions (input parsing, HTTP responses)
services/       — Business logic (queries, writes, domain rules)
seed_data.py    — Populates the database with test users, songs, and friendships
```

---

### models.py

Defines 7 models and 3 association tables. All primary keys are UUID strings.

| Model | Purpose |
|-------|---------|
| `User` | A registered user. Tracks `listening_streak` and `last_listened_at` for the streak feature. Has a many-to-many self-referential `friends` relationship via the `friendships` table. |
| `Song` | A song shared by a user. Stores title, artist, album, genre, and a `shared_by` foreign key pointing to the user who originally added it. |
| `Tag` | A label (e.g. genre or mood) that can be attached to many songs. |
| `ListeningEvent` | A record that a specific user listened to a specific song at a specific time. Drives both streak updates and the social feed. |
| `Rating` | A 1–5 score a user gives a song. Enforces a unique constraint on `(user_id, song_id)` so each user can only rate a song once. |
| `Playlist` | A named, optionally collaborative collection of songs. Songs are linked via `playlist_entries`, which stores position, who added it, and when. |
| `Notification` | An in-app alert for a user (e.g., someone rated their shared song or added it to a playlist). Stores `notification_type`, `body`, and a `read` boolean. |

Association tables:
- **`friendships`** — symmetric many-to-many between users
- **`song_tags`** — many-to-many between songs and tags
- **`playlist_entries`** — many-to-many between playlists and songs, with extra columns for `position`, `added_by`, and `added_at`

---

### app.py

Creates the Flask app via `create_app()`. Configures a SQLite database (default: `mixtape.db`, overridable via environment variable), initializes SQLAlchemy, creates all tables, and registers these four blueprints:

| Prefix | Blueprint |
|--------|-----------|
| `/songs` | `songs_bp` |
| `/playlists` | `playlists_bp` |
| `/users` | `users_bp` |
| `/feed` | `feed_bp` |

---

### Routes

**`routes/songs.py`** — song discovery and interaction

| Endpoint | What it does |
|----------|-------------|
| `GET /songs/search?q=` | Delegates to `search_service.search_songs(query)` |
| `GET /songs/<song_id>` | Delegates to `search_service.get_song(song_id)` |
| `POST /songs/<song_id>/rate` | Delegates to `notification_service.rate_song(user_id, song_id, score)` |
| `POST /songs/<song_id>/listen` | Delegates to `streak_service.record_listening_event(user_id, song_id)` |

**`routes/playlists.py`** — playlist management

| Endpoint | What it does |
|----------|-------------|
| `POST /playlists/` | Delegates to `playlist_service.create_playlist(name, created_by, is_collaborative)` |
| `GET /playlists/<id>` | Delegates to `playlist_service.get_playlist(playlist_id)` |
| `GET /playlists/<id>/songs` | Delegates to `playlist_service.get_playlist_songs(playlist_id)` |
| `POST /playlists/<id>/songs` | Delegates to `notification_service.add_to_playlist(playlist_id, song_id, added_by)` |

**`routes/users.py`** — user profiles and notifications

| Endpoint | What it does |
|----------|-------------|
| `GET /users/<user_id>` | Queries the `User` model directly and returns the user dict |
| `GET /users/<user_id>/streak` | Delegates to `streak_service.get_streak(user_id)` |
| `GET /users/<user_id>/notifications` | Delegates to `notification_service.get_notifications(user_id, unread_only)` |
| `POST /users/notifications/<notification_id>/read` | Delegates to `notification_service.mark_as_read(notification_id)` |

**`routes/feed.py`** — social feed

| Endpoint | What it does |
|----------|-------------|
| `GET /feed/<user_id>/listening-now` | Delegates to `feed_service.get_friends_listening_now(user_id)` |
| `GET /feed/<user_id>/activity` | Delegates to `feed_service.get_activity_feed(user_id, limit)` |

---

### Services

**`services/search_service.py`**
Handles read-only song queries. `search_songs` runs a case-insensitive `LIKE` on song title and artist. `get_song` fetches a single song by ID and raises `ValueError` if it doesn't exist.

**`services/notification_service.py`**
Handles all write operations that may produce notifications. `rate_song` creates or updates a `Rating` record (the unique constraint means repeat ratings update rather than insert). `add_to_playlist` adds a song to a playlist and then notifies the song's original sharer if they're not the one doing the adding. `create_notification` is the shared helper both use. `get_notifications` and `mark_as_read` support the notification inbox in the users routes.

**`services/streak_service.py`**
Tracks daily listening habits. `record_listening_event` writes a `ListeningEvent` and calls `update_listening_streak`. The streak logic is: first listen ever → streak = 1; listening again on the same day → no change; listening the next calendar day (excluding Sundays) → streak increments; a gap of 2+ days → streak resets to 1. `get_streak` returns the user's current streak count.

**`services/feed_service.py`**
Generates the social feed from `ListeningEvent` records. `get_friends_listening_now` queries events from the past 24 hours for all of the user's friends, then deduplicates to one entry per friend (their most recent song). `get_activity_feed` returns the 20 most recent events across all friends with no time restriction.

**`services/playlist_service.py`**
Handles playlist reads and creation. `create_playlist` inserts a new `Playlist` row. `get_playlist_songs` returns songs in their stored position order. `get_playlist` returns playlist metadata without the song list. `get_user_playlists` returns all playlists a user has created.

---

### Data Flow: Adding a Song to a Playlist Triggers a Notification

`POST /playlists/<playlist_id>/songs` in `routes/playlists.py` calls `notification_service.add_to_playlist()`. That function looks up the song to find its original sharer, inserts a new row into `playlist_entries` (recording position, who added it, and when), then checks whether the person adding the song is different from the person who originally shared it. If they're different, it creates a `Notification` record for the sharer. The notification just sits in the database until the sharer calls `GET /users/<user_id>/notifications`.

One thing worth noting: the function that adds a song to a playlist lives in `notification_service`, not `playlist_service`. That's because the add-to-playlist action is the trigger — the notification is the reason it's grouped there. `playlist_service` only handles reads and creation; any write that produces a side effect belongs to `notification_service`.

Second data flow worth noting: `POST /songs/<song_id>/listen` in `routes/songs.py` calls `streak_service.record_listening_event()`. That function writes a `ListeningEvent` record, then immediately calls `update_listening_streak()` on the same user. The streak logic checks how many calendar days have passed since `last_listened_at`: same day means no change, next day increments, any bigger gap resets to 1. Sundays are explicitly excluded from breaking a streak. The streak counter lives directly on the `User` row — there's no separate streak model, just two columns (`listening_streak` and `last_listened_at`) that get updated in place.

Key design decision: listening and rating are tracked separately. `ListeningEvent` records drive the streak and the social feed. `Rating` records (a separate model with a unique constraint on `user_id + song_id`) are a distinct social signal that triggers a notification back to the original sharer. The two tables have nothing to do with each other at the data layer.

## Root Cause Analysis

---

### Issue 1: My listening streak keeps resetting

**How I reproduced it**
Using nova's user ID, I called `POST /songs/<song_id>/listen` with today's date, then checked `GET /users/<user_id>/streak`. The streak returned 1 even when the user had an existing multi-day streak and had listened the day before. The issue consistently reproduced whenever the listen happened to fall on a Sunday.

**How I found the root cause**
The route `POST /songs/<song_id>/listen` delegates to `streak_service.record_listening_event`, so I opened `services/streak_service.py`. I read `update_listening_streak` and focused on the conditional block starting at line 70. The middle branch on line 73 — `elif days_since_last == 1 and today.weekday() != 6` — had two conditions joined by `and`. The second condition (`today.weekday() != 6`) immediately stood out: it gates the increment on today not being Sunday. That meant any listen on a Sunday would fall through to `else` and reset the streak.

**The root cause**
The Sunday exclusion was applied to the wrong branch. The intent was: if a user skips Sunday and listens on Monday, the two-day gap should not reset the streak. Instead, the code blocks the streak from incrementing when today is Sunday — so listening on Sunday after Saturday resets the streak to 1 instead of incrementing it. Additionally, the Monday-after-Saturday case (`days_since_last == 2, today.weekday() == 0`) is never handled; it also falls to `else` and resets.

**Fix and side-effect check**
The fix is to split the branch into two cases: `days_since_last == 1` always increments (consecutive days, no Sunday exception needed here), and `days_since_last == 2 and today.weekday() == 0` also increments (Monday after Saturday, Sunday excused). Everything else resets. After the fix I verified that same-day listens still return early (`days_since_last == 0`) and that a two-day gap on a non-Monday still resets correctly.

---

### Issue 2: Friends Listening Now shows people from yesterday

**How I reproduced it**
I called `GET /feed/<aaliya_user_id>/listening-now`. Aaliya's only friend is kenji. Kenji had a seed event from 26 hours ago. With a 24-hour threshold, kenji appeared in the "listening now" feed even though he had not listened for over a day. The feed is supposed to show only friends who listened in the last 30 minutes.

**How I found the root cause**
The route delegates to `feed_service.get_friends_listening_now`. I opened `services/feed_service.py` and saw the constant on line 13: `RECENT_THRESHOLD = timedelta(hours=24)`. The cutoff was calculated correctly using that constant, but the constant itself was wrong — 24 hours instead of 30 minutes.

**The root cause**
`RECENT_THRESHOLD` was set to `timedelta(hours=24)` instead of `timedelta(minutes=30)`. Any friend who listened within the past 24 hours was included in "Friends Listening Now," meaning people who stopped listening yesterday were still shown as active. The deduplication logic (keeping only the most recent event per friend) was correct; only the time window was wrong.

**Fix and side-effect check**
Changed line 13 to `RECENT_THRESHOLD = timedelta(minutes=30)`. The `get_activity_feed` function intentionally has no time restriction (it returns the 20 most recent events regardless of age), so it was unaffected. I confirmed the filter and deduplication logic downstream of the cutoff was untouched.

---

### Issue 3: The same song keeps showing up twice in search

**How I reproduced it**
I called `GET /songs/search?q=rap`. Songs in the seed data are tagged with "rap" but no song has "rap" in its title or artist name. The result was an empty list — tag-based searches returned nothing. When searching by title (e.g. `?q=Crown Heights`), SQLAlchemy 2.0 deduplicated the rows silently, hiding the duplicate issue at the response level.

**How I found the root cause**
I opened `services/search_service.py` and read the query on lines 25–35. The `.outerjoin(song_tags, Song.id == song_tags.c.song_id)` on line 27 joins the association table, which multiplies each song into N rows (one per tag) in the SQL result. However, the `.filter()` on lines 28–33 only checks `Song.title` and `Song.artist` — the joined `song_tags` table is never referenced in the filter. The join serves no purpose for the current filter but creates duplicate rows in the underlying SQL. Without `.distinct()`, any backend or ORM version that doesn't silently deduplicate would return duplicate song entries.

**The root cause**
The `outerjoin` on `song_tags` was added to enable tag-based search, but the corresponding `Tag.name.ilike(...)` filter condition was never included. This left the join present but unused in the WHERE clause. Because the join fans out one row per tag per song, a song with three tags produces three SQL rows. In SQLAlchemy 2.0 the ORM identity map happens to collapse these, but the query is structurally incorrect: it is missing both the tag filter and a `.distinct()` guard.

**Fix and side-effect check**
Added `.distinct()` on line 34 to prevent duplicate rows from reaching the result list regardless of SQLAlchemy version behavior. The filter logic for title and artist was not changed. I verified that songs with no tags (which produce a single NULL row via the outer join) are still returned correctly, and that `get_song` (the other function in the same file) was unaffected.

---

### Issue 4: I got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it**
I called `POST /songs/<song_id>/rate` with a different user's `user_id` and a score, then checked `GET /users/<original_sharer_id>/notifications`. The rating was saved correctly (the endpoint returned 201 and the Rating record existed), but the sharer's notification inbox remained empty. By contrast, calling `POST /playlists/<id>/songs` for the same song did produce a notification.

**How I found the root cause**
Both rating and playlist-adding route through `notification_service.py`. I compared `add_to_playlist` (lines 35–70) and `rate_song` (lines 73–110) side by side. `add_to_playlist` explicitly calls `create_notification(...)` after the write. `rate_song` creates or updates the `Rating` record and commits — and then stops. There is no call to `create_notification` anywhere in `rate_song`.

**The root cause**
`rate_song` never calls `create_notification`. The function correctly handles the upsert logic for the Rating record but is missing the notification step that `add_to_playlist` has. The song's original sharer (`song.shared_by`) is never notified when someone rates their song.

**Fix and side-effect check**
After the rating upsert and before returning, add a check: if `rater.id != song.shared_by`, call `create_notification` for `song.shared_by` with type `"song_rated"` and an appropriate message. This mirrors the pattern already used in `add_to_playlist`. I confirmed that the self-rating case (user rates their own shared song) correctly skips the notification, and that the existing rating upsert logic (update if exists, insert if not) was not changed.

---

### Issue 5: The last song in a playlist never shows up

**How I reproduced it**
I called `GET /playlists/<seeded_playlist_id>/songs` on "Late Night Vibes," which the seed data populates with 7 songs at positions 1–7. The response returned `count: 6` and was missing the 7th song. Every playlist tested was consistently short by exactly one song.

**How I found the root cause**
The route delegates to `playlist_service.get_playlist_songs`. I opened `services/playlist_service.py` and read the function. The query on lines 58–64 is correct: it joins `playlist_entries`, filters by playlist ID, and orders by position ascending. The problem was on line 66, the return statement: `return [song.to_dict() for song in songs[:-1]]`. The `[:-1]` slice drops the last element of the list before returning it.

**The root cause**
The list slice `songs[:-1]` removes the final song from every playlist response. A playlist with 7 songs returns 6; a playlist with 1 song returns 0. The slice has no conditional logic — it unconditionally discards the last entry regardless of playlist size.

**Fix and side-effect check**
Changed `songs[:-1]` to `songs` on line 66. I confirmed that the query ordering (ascending by position) was already correct, so removing the slice returns all songs in the right order. `get_playlist` (metadata only, no songs) and `create_playlist` were unaffected.