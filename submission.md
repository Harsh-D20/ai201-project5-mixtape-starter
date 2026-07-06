# Submission

## Codebase Map

| File | Responsibility |
|---|---|
| `app.py` | Flask application factory (`create_app`). Configures the DB URI/secret key, initializes the shared `db = SQLAlchemy()` instance, registers all four blueprints under their URL prefixes (`/songs`, `/playlists`, `/users`, `/feed`), and calls `db.create_all()`. This is the single entry point (`flask run` targets `app:create_app`). |
| `models.py` | All SQLAlchemy models (`User`, `Song`, `Tag`, `ListeningEvent`, `Rating`, `Playlist`, `Notification`) plus three raw association tables for many-to-many relationships: `friendships` (self-referential, symmetric), `song_tags`, and `playlist_entries` (which also carries `position`, `added_by`, `added_at` — ordering/audit metadata for playlist membership). Every model has a `to_dict()` used to serialize it straight into a JSON response. IDs are UUID strings, not integers. |
| `routes/songs.py` | Blueprint for `/songs`: search, get one song, rate a song, log a listening event. Delegates to `search_service`, `notification_service`, `streak_service`. |
| `routes/playlists.py` | Blueprint for `/playlists`: create a playlist, get playlist metadata, list a playlist's songs, add a song to a playlist. Delegates to `playlist_service` and `notification_service`. |
| `routes/users.py` | Blueprint for `/users`: get a user, get their streak, list/mark notifications. Delegates to `streak_service` and `notification_service` (and queries `User` directly for the plain profile lookup). |
| `routes/feed.py` | Blueprint for `/feed`: "friends listening now" and general activity feed, both scoped to `/feed/<user_id>/...`. Delegates entirely to `feed_service`. |
| `services/search_service.py` | Song search by title/artist substring match, including tags. |
| `services/streak_service.py` | Listening-streak math: records a `ListeningEvent` and updates `User.listening_streak`/`last_listened_at` based on how many calendar days have passed since the last listen. |
| `services/feed_service.py` | Builds the "listening now" (last 24h, deduplicated per friend) and "activity" (last N events, unfiltered by recency) feeds from friends' `ListeningEvent` rows. |
| `services/playlist_service.py` | Playlist CRUD-ish logic: create a playlist, fetch its metadata, fetch its songs in position order, list a user's playlists. |
| `services/notification_service.py` | Creates `Notification` rows and is the cross-cutting service that reacts to actions elsewhere in the app (rating a song, adding a song to a playlist) by notifying the relevant user. Also handles read/unread retrieval. |
| `seed_data.py` | Standalone script (`python seed_data.py`) that drops and recreates all tables, then populates users, friendships, songs, tags, playlists, listening events, ratings, and notifications for local testing. |
| `tests/test_streaks.py`, `tests/test_search.py`, `tests/test_playlists.py` | Pytest suites per service, each using an in-memory SQLite DB via a `create_app({"TESTING": True, ...})` fixture. |

### Data flow: adding a song to a playlist triggers a notification

1. **Request in:** `POST /playlists/<playlist_id>/songs` with JSON body `{"song_id": ..., "added_by": ...}` hits `add_song()` in `routes/playlists.py`.
2. **Route-level validation:** the route only checks that `song_id` and `added_by` are present in the body — no DB access yet.
3. **Route calls service:** it calls `notification_service.add_to_playlist(playlist_id, song_id, added_by)` — notice the *playlist* route delegates to the *notification* service, not `playlist_service`, because appending a song is entangled with notifying the original sharer.
4. **Service does the real work** (`services/notification_service.py::add_to_playlist`):
   - Loads the `Song`, the adding `User`, and the `Playlist` from the DB, raising `ValueError` (→ mapped to a 400 by the route) if any is missing.
   - If the song isn't already in `playlist.songs`, appends it (writing a row into the `playlist_entries` association table) and commits.
   - If the person adding the song is *not* the same person who originally shared it (`song.shared_by != added_by_user_id`), it calls `create_notification(...)`.
5. **Notification is created:** `create_notification` builds a `Notification` row (`user_id=song.shared_by`, `notification_type="song_added_to_playlist"`, a human-readable `body`) and commits it.
6. **Response out:** control returns to the route, which responds `{"message": "Song added to playlist"}, 201`.
7. **Read path:** later, the original sharer hits `GET /users/<user_id>/notifications`, which calls `notification_service.get_notifications`, querying `Notification` rows for that user (optionally filtered to unread) ordered newest-first.

### Patterns noticed

- **Strict three-layer architecture:** routes never touch `db`/models directly (except the one trivial `GET /users/<user_id>` lookup) and never contain business logic — they parse input, call exactly one or two service functions, and translate `ValueError` into a 4xx JSON response. All logic and all `db.session.commit()` calls live in `services/`.
- **Services are organized by domain, not strictly 1:1 with blueprints.** Most services map cleanly to one blueprint (`search_service` ↔ songs, `feed_service` ↔ feed, `streak_service` ↔ users/songs), but `notification_service` is cross-cutting — it's called from both `routes/songs.py` (on rating) and `routes/playlists.py` (on adding a song), because "notify someone" is a side effect of actions owned by other features rather than a feature of its own with its own route for *creating* notifications (only for *reading* them, via `routes/users.py`).
- **Services call other services, not just models.** `notification_service.add_to_playlist` imports from `services.playlist_service` (for `Playlist`/song membership context) — the layering is routes → services → models, but *within* the services layer, cross-imports happen for shared side effects rather than duplicating logic.
- **Shared, module-level `db` object.** `db = SQLAlchemy()` is defined once in `app.py` and imported everywhere else (`from app import db`) rather than passed around — this is the consistent pattern for DB access across all services and routes.
- **Consistent error convention:** every service raises a plain `ValueError` with a human-readable message for "not found" / "invalid input" cases, and every route catches `ValueError` and turns it into `jsonify({"error": str(e)}), <4xx>` — there's no custom exception hierarchy.
- **Consistent serialization convention:** every model has a `to_dict()` and routes/services always return dicts (or lists of dicts) built from `to_dict()`, never a raw model instance, keeping JSON shaping colocated with the model definition.
- **UUID primary keys everywhere**, generated in Python (`generate_uuid()` in `models.py`) rather than DB-assigned integers, which is why tests and seed data create real rows first to get IDs rather than assuming small sequential integers.

## Bug Fix 1
### How I reproduced the error

`tests/test_streaks.py` already contains a test for this exact scenario:

```python
def test_streak_increments_on_sunday(app, user):
    """
    Listening on Saturday and then Sunday should increment the streak.
    """
    ...
    update_listening_streak(u, saturday)
    assert u.listening_streak == 1
    update_listening_streak(u, sunday)
    assert u.listening_streak == 2  # Should increment, not reset
```

Running it confirmed the bug:

```
$ pytest tests/test_streaks.py -v
tests/test_streaks.py::test_streak_starts_at_1_for_new_user PASSED
tests/test_streaks.py::test_streak_increments_on_consecutive_day PASSED
tests/test_streaks.py::test_streak_does_not_double_count_same_day PASSED
tests/test_streaks.py::test_streak_resets_after_skipped_day PASSED
tests/test_streaks.py::test_streak_increments_on_sunday FAILED

E       assert 1 == 2
E        +  where 1 = <User ...>.listening_streak
```

To see the real-world impact (not just one boundary case), I also wrote a small script that simulates a user listening every single day for two weeks, straight through, and prints the streak after each day. If the streak logic were correct, it should climb to 14 with no drops:

```
2024-06-10 (Monday   ) -> streak = 1
2024-06-11 (Tuesday  ) -> streak = 2
2024-06-12 (Wednesday) -> streak = 3
2024-06-13 (Thursday ) -> streak = 4
2024-06-14 (Friday   ) -> streak = 5
2024-06-15 (Saturday ) -> streak = 6
2024-06-16 (Sunday   ) -> streak = 1   <-- dropped despite listening every day
2024-06-17 (Monday   ) -> streak = 2
2024-06-18 (Tuesday  ) -> streak = 3
2024-06-19 (Wednesday) -> streak = 4
2024-06-20 (Thursday ) -> streak = 5
2024-06-21 (Friday   ) -> streak = 6
2024-06-22 (Saturday ) -> streak = 7
2024-06-23 (Sunday   ) -> streak = 1   <-- dropped again
```

This matches the reported behavior: the streak resets even when the user never actually skips a day — it happens like clockwork, once a week.

### How I found the root cause

The docstring for `update_listening_streak` spells out the intended rules explicitly:

- No prior listen → streak starts at 1
- Already listened today → no change
- Listened yesterday → streak increments by 1
- More than one day has passed → streak resets to 1

There is no day-of-week exception mentioned anywhere in the spec. Reading the function body against these four rules line by line, the branch that handles "listened yesterday" stood out:

```python
elif days_since_last == 1 and today.weekday() != 6:
    user.listening_streak += 1
else:
    user.listening_streak = 1
```

`days_since_last == 1` is exactly the "listened yesterday" condition from the docstring, but it's gated behind `today.weekday() != 6` (Python's `date.weekday()` returns `6` for Sunday). Every time "today" is a Sunday, this extra clause makes the condition `False` regardless of `days_since_last`, so execution falls into the `else` branch and the streak is unconditionally reset to 1 — even though the user listened on consecutive days.

### Root cause

`services/streak_service.py`, line 73 (before fix):

```python
elif days_since_last == 1 and today.weekday() != 6:
```

An erroneous `and today.weekday() != 6` condition was added to the "listened yesterday" branch. This has no basis in the documented streak rules and causes the streak-increment path to be skipped every Sunday, incorrectly routing execution into the reset branch instead. The result: any user who listens every day still has their streak reset to 1 once a week, on Sunday.

### How I solved it and tested for side effects

**Fix** — remove the erroneous weekday condition so the branch matches the documented rule ("listened yesterday → increment"):

```python
elif days_since_last == 1:
    user.listening_streak += 1
```

**Testing:**

1. Re-ran the two-week daily-listening simulation script — the streak now climbs monotonically through Sundays with no drops:
   ```
   2024-06-15 (Saturday ) -> streak = 6
   2024-06-16 (Sunday   ) -> streak = 7
   ...
   2024-06-22 (Saturday ) -> streak = 13
   2024-06-23 (Sunday   ) -> streak = 14
   ```
2. Ran `pytest tests/test_streaks.py -v` — all 5 tests pass, including `test_streak_increments_on_sunday`.
3. Ran the full suite (`pytest tests/ -v`) to check for side effects in unrelated areas:
   - `tests/test_search.py` — all 5 tests pass (unaffected, different module).
   - `tests/test_playlists.py` — 1 of 3 tests pass; the other two (`test_playlist_returns_all_songs`, `test_playlist_returns_songs_in_order`) fail, but this is **pre-existing and unrelated** — it's Issue #5 ("the last song in a playlist never shows up"), which lives entirely in `services/playlist_service.py`, a file this change never touches. Confirmed via `git diff --stat` that only `services/streak_service.py` was modified.

No other code path reads `today.weekday()` or depends on this branch, and the change is a strict one-line removal of an incorrect condition, so there is no risk beyond the streak-update logic itself.
