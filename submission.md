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
