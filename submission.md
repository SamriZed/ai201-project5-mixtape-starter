# Codebase map

- **`app.py`** — Flask application factory. Creates the Flask app, initializes SQLAlchemy, registers all four blueprints (songs, playlists, users, feed), and sets up the database.
- **`models.py`** — Eight SQLAlchemy models defining the domain entities and relationships.

### Database Models
1. **`User`** — Represents an app user. Tracks username, email, listening streak, and last listening timestamp. Has relationships to shared songs, ratings, listening events, notifications, playlists, and friends (many-to-many via `friendships` table).
2. **`Song`** — Represents a song shared by a user. Stores title, artist, album, genre, sharer ID, and an optional share note. Has tags (many-to-many via `song_tags`), ratings, and listening events.
3. **`Tag`** — Simple lookup table for song tags (genre/mood classifications).
4. **`ListeningEvent`** — Records when a user listened to a song. Has user_id, song_id, and listened_at timestamp. Used by feed service and streak tracking.
5. **`Rating`** — User's 1–5 score for a song. Has a unique constraint on (user_id, song_id) so each user can rate each song once.
6. **`Playlist`** — User-created collection of songs. Stores name, creator, creation timestamp, and collaborative flag. Songs are stored via the `playlist_entries` join table (which adds position ordering and tracks who added each song).
7. **`Notification`** — Alerts sent to users when friends interact with their content. Has type (e.g., "song_added_to_playlist"), body text, creation timestamp, and read flag.
8. **`friendships`** (association table) — Symmetric many-to-many linking users as friends.

### Routes (Request Handlers)
Each route immediately delegates to a service function, handling only input validation and response formatting.

- **`routes/songs.py`** — Four endpoints:
  - `GET /songs/search?q=<query>` → `search_songs()` 
  - `GET /songs/<id>` → `get_song()`
  - `POST /songs/<id>/rate` → `rate_song()` (user rates a song)
  - `POST /songs/<id>/listen` → `record_listening_event()` (user listens, updates streak)

- **`routes/playlists.py`** — Four endpoints:
  - `POST /playlists/` → `create_playlist()`
  - `GET /playlists/<id>` → `get_playlist()`
  - `GET /playlists/<id>/songs` → `get_playlist_songs()`
  - `POST /playlists/<id>/songs` → `add_to_playlist()` (also triggers notification)

- **`routes/users.py`** — Four endpoints:
  - `GET /users/<id>` → retrieves user metadata
  - `GET /users/<id>/streak` → `get_streak()`
  - `GET /users/<id>/notifications` → `get_notifications()`
  - `POST /users/notifications/<id>/read` → `mark_as_read()`

- **`routes/feed.py`** — Two endpoints:
  - `GET /feed/<user_id>/listening-now` → `get_friends_listening_now()`
  - `GET /feed/<user_id>/activity` → `get_activity_feed()`

### Services (Business Logic)
All domain logic lives here; services are called by routes and by other services.

- **`services/feed_service.py`** — Builds user feeds from friend activity:
  - `get_friends_listening_now(user_id)` — Returns friends who listened in the last 24 hours, one song per friend, most recent first.
  - `get_activity_feed(user_id, limit=20)` — Returns the most recent N listening events from all friends (no time filter).

- **`services/notification_service.py`** — Notifications and interactions:
  - `create_notification(user_id, type, body)` — Creates a raw notification.
  - `add_to_playlist(playlist_id, song_id, added_by_user_id)` — Adds a song to a playlist and notifies the original song sharer.
  - `rate_song(user_id, song_id, score)` — Saves or updates a rating (1–5). Creates or updates a Rating record.
  - `get_notifications(user_id, unread_only=False)` — Retrieves notifications, optionally filtered to unread.
  - `mark_as_read(notification_id)` — Marks a notification as read.

- **`services/playlist_service.py`** — Playlist CRUD and song management:
  - `create_playlist(name, created_by_user_id, is_collaborative=True)` — Creates a new playlist.
  - `get_playlist_songs(playlist_id)` — Returns all songs in a playlist, ordered by position.
  - `get_playlist(playlist_id)` — Returns playlist metadata.
  - `get_user_playlists(user_id)` — Returns all playlists created by a user.

- **`services/search_service.py`** — Song search:
  - `search_songs(query)` — Searches for songs by title or artist (case-insensitive).
  - `get_song(song_id)` — Retrieves a single song by ID.

- **`services/streak_service.py`** — Listening streaks (consecutive days):
  - `record_listening_event(user_id, song_id)` — Creates a ListeningEvent and updates the user's streak.
  - `update_listening_streak(user, now)` — Increments streak if listened yesterday, resets if a day is skipped, starts at 1 if new.
  - `get_streak(user_id)` — Returns current streak count.

---

## Data Flow Example: Adding a Song to a Playlist and Triggering a Notification

**Scenario:** User Alice adds a song that Bob originally shared to her collaborative playlist.

1. **HTTP Request:** `POST /playlists/playlist-123/songs`
   - Body: `{ "song_id": "song-456", "added_by": "alice-id" }`
   
2. **Route Handler** (`routes/playlists.py` → `add_song()`):
   - Extracts `playlist_id`, `song_id`, `added_by` from request.
   - Validates that all required fields are present.
   - Calls `notification_service.add_to_playlist(playlist_id, song_id, added_by)`

3. **Business Logic** (`services/notification_service.py` → `add_to_playlist()`):
   - Fetches the Song record for `song_id` ("song-456").
   - Fetches the User record for `added_by` (Alice).
   - Fetches the Playlist record for `playlist_id`.
   - Adds the song to the playlist: `playlist.songs.append(song)` and commits.
   - **Notification Logic:** Checks if `song.shared_by` (Bob) ≠ `added_by` (Alice).
   - Since Bob is different, calls `create_notification()` to create a notification for Bob.
   - Notification type: `"song_added_to_playlist"`, body: `"Alice added your song 'Song Title' to the playlist 'playlist-123'."`

4. **Database State After:**
   - A row is inserted into `playlist_entries` joining the song to the playlist.
   - A row is inserted into `notifications` for Bob with type `"song_added_to_playlist"` and read=False.

5. **HTTP Response:** `{ "message": "Song added to playlist" }` (201 Created)

---

## Root Cause Analysis
### How I reproduced the Bug

---

### Bug #1 — Listening streak keeps resetting
*Service:* `services/streak_service.py`

- **Symptom (reported behavior):**
  The strak functions normally on all days except on Sunday. It resets to 1 in sunday even if the user listened to a song on Saturday.

- **How I reproduced it:**
 I run the test_streaks.py and saw it failed on Sunday. Instead of incrementing it resets.

- **How I found the root cause:**
  I opened `services/streak_service.py` and read `update_listening_streak()` — the only place the streak value changes. Line 73 was the giveaway: `elif days_since_last == 1 and today.weekday() != 6:`. `weekday() == 6` is Sunday, so a valid Saturday→Sunday listen skips the increment and drops into the `else` that resets to 1. The docstring never mentions Sundays, which confirmed this was the real cause, not intended behavior.

- **Root cause:**
     elif days_since_last == 1 and today.weekday() != 6:

- **My fix and side-effect check:**
  I removed the `and today.weekday() != 6` clause so line 73 reads `elif days_since_last == 1:`. A one-day gap now increments on any weekday. I re-ran `tests/test_streaks.py` — the Sunday test passes and the other four still pass, so normal-day behavior is unchanged.

---

### Bug #2 — Friends Listening Now shows people from yesterday
*Service:* `services/feed_service.py`

- **Symptom (reported behavior):**
  A person 23 hours ago appears, which is technically within 24 hours, but the feature is meant to show only people who are actively listening now. The feed should not include friends whose last listening event was yesterday.

- **How I reproduced it:**
    I created a repro_feed.py in my root directory and added a friend who listened to a song 23 hours ago. I then called the get_friends_listening_now() function and saw that the friend was included in the results.

- **Root cause:**
    RECENT_THRESHOLD = timedelta(hours=24)
      cutoff = datetime.now(timezone.utc) - RECENT_THRESHOLD

---

### Bug #3 — Same song shows up twice in search
*Service:* `services/search_service.py`

- **Symptom (reported behavior):**
    The same song appears twice in search results when it has multiple tags. The search query returns one row per tag, so multi-tag songs are duplicated.

- **How I reproduced it:**
  I created a test case with a song that has multiple tags and executed the search query.

- **Root cause:**
  .outerjoin(song_tags, Song.id == song_tags.c.song_id)

---

### Bug #5 — Last song in a playlist never shows up
*Service:* `services/playlist_service.py`

- **Symptom (reported behavior):**
  last song in a playlist never shows up when calling `get_playlist_songs()`. The function returns all songs except the last one, which is a bug.

- **How I reproduced it:**
    I created a playlist with multiple songs and called `get_playlist_songs()`. I noticed that the last song was missing from the returned list.

- **Root cause:**
  The issue is in the `get_playlist_songs()` function where the query is slicing the results, excluding the last song.


---




