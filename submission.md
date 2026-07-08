## AI Usage

I used AI (Claude) twice during debugging, in different roles:

1. **Docstring-vs-implementation comparisons.** For each of the three services I was debugging, I asked the AI to read the function's docstring alongside its body and flag anywhere the code didn't match the stated contract, rather than asking it "what's the bug." That's what produced the `diagnosis.txt` format (docstring says / code does / bug is on line N / fix is). It's how the streak issue and the notification issue both got pinpointed quickly — in both cases the mismatch between the documented behavior and the actual conditional/missing block was the bug itself, not a red herring.

2. **Explaining the `playlist_entries` join.** I wasn't confident I understood why `get_playlist_songs` joined through the association table instead of just using `playlist.songs`, so I asked the AI to explain what the join + `order_by(asc(position))` was doing. It correctly explained the join, but its first answer also guessed that the bug might be in the join condition itself (suggesting the `song_id` join key could be wrong). I checked that against `models.py` directly — the `playlist_entries` table's columns and the join in `playlist_service.py` matched exactly — and ran the test suite before and after to confirm the join wasn't the problem. The AI's initial hypothesis was wrong; the actual bug turned out to be the `[:-1]` slice a few lines below, which I found by reading the rest of the function myself once the join was ruled out.


## Codebase Map

``app.py``

Flask application factory. Creates the app, configures the SQLAlchemy database, and registers the blueprints defined in `routes/`. This is the entry point — it doesn't contain business logic itself.

``models.py``

Defines the SQLAlchemy models for all entities: User, Tag, Song, ListeningEvent, Rating, Playlist, and Notification. Three association tables back the many-to-many relationships: `friendships` (symmetric user-to-user), `song_tags` (song-to-tag), and `playlist_entries` (playlist-to-song, which also carries `position`, `added_by`, and `added_at` columns — so songs in a playlist have an explicit order and an audit trail of who added them, not just insertion order). Ratings are their own table (`Rating`), not a column on Song, with a unique constraint on `(user_id, song_id)` so a user can only have one rating per song.

``routes/songs.py``, ``routes/playlists.py``, ``routes/users.py``, ``routes/feed.py``

Each route file is a Flask blueprint that maps HTTP endpoints to service calls. Routes are thin: they parse the request body/query params, call exactly one service function, and format the response as JSON (translating `ValueError` from the service layer into a 400/404). No business logic lives in the routes layer.

``services/streak_service.py``

Handles listening-streak logic. Three functions: `record_listening_event` (creates a `ListeningEvent` row and triggers a streak update), `update_listening_streak` (the actual streak rules — increments on a consecutive calendar day, resets to 1 if a day was skipped, no-ops if already counted today), and `get_streak` (read-only lookup). `record_listening_event` is the write path, `update_listening_streak` is where the streak math lives, `get_streak` is the read path.

``services/playlist_service.py``

Handles playlist creation and retrieval. Four functions: `create_playlist` (inserts a new Playlist row), `get_playlist_songs` (joins through `playlist_entries` to return songs in position order), `get_playlist` (metadata only, no songs), and `get_user_playlists` (all playlists a given user created). `create_playlist` is the only function that writes; the other three are read-only queries.

``services/notification_service.py``

Handles notification creation and retrieval, and is where cross-user interactions get surfaced. Five functions: `create_notification` (the shared low-level insert), `add_to_playlist` (adds a song to a playlist, then notifies the song's original sharer if someone else added it), `rate_song` (saves/updates a `Rating` row, then notifies the song's original sharer if someone else rated it), `get_notifications` (query for a user's notifications, optionally unread-only), and `mark_as_read` (flips the `read` flag). `add_to_playlist` and `rate_song` share the same shape: mutate the underlying data, then call `create_notification` if the actor isn't the song's original sharer.

``services/search_service.py``

Handles song search. `search_songs` does a case-insensitive `ILIKE` match against title or artist and returns matching songs (with tags). `get_song` is a single-song lookup by ID.

``services/feed_service.py``

Handles the "friends" surfaces. `get_friends_listening_now` returns each friend's most recent listening event within a rolling 24-hour window (`RECENT_THRESHOLD`), deduplicated to one entry per friend. `get_activity_feed` is similar but ignores recency entirely — it just returns the most recent N events from friends, capped by `limit`.

## Data Flow

**A user rates a song:** `POST /songs/<song_id>/rate` in `routes/songs.py` calls `notification_service.rate_song(user_id, song_id, score)`. That function looks up the `Song` and `User`, checks for an existing `Rating` row for that (user, song) pair (updating it if present, otherwise inserting a new one), commits, and then — if the rater isn't the person who originally shared the song — calls `create_notification` to notify `song.shared_by`. There's no separate rating "event" model; the Rating table itself is the record, and the notification is a side effect of saving it.

**A user adds a song to a playlist:** `POST /playlists/<playlist_id>/songs` in `routes/playlists.py` calls `notification_service.add_to_playlist(playlist_id, song_id, added_by_user_id)`. That function appends the song to `playlist.songs` (the many-to-many `playlist_entries` relationship) if it isn't already present, commits, and then — following the same pattern as `rate_song` — notifies the song's original sharer if they weren't the one who added it.

Pattern I noticed: every route delegates immediately to a service function. The routes do input parsing and response formatting; all business logic lives in `services/`. Within `services/`, `notification_service.py` in particular follows a consistent shape across its two "interaction" functions: mutate state, commit, then conditionally notify.

## Root Cause Analysis

``Issue #1 — My listening streak keeps resetting``

**Reproduction:** Seed a user with no listening history. Call `record_listening_event` (or `POST /songs/<id>/listen`) on a Saturday, then again on the following Sunday. Expected: streak goes 1 → 2 (two consecutive calendar days). Actual: streak drops back to 1 on the Sunday call, as if a day had been skipped.

**Navigation strategy:** `tests/test_streaks.py` already had a test for this exact scenario (`test_streak_increments_on_sunday`), so I ran `pytest tests/test_streaks.py` first rather than guessing. That test was the only failure. It pointed straight at `update_listening_streak`, so I opened `streak_service.py` and read the docstring above the function: "If the user listened yesterday: streak increments by 1" — no exception mentioned for any particular day. Then I read the actual conditional and found `elif days_since_last == 1 and today.weekday() != 6:` — an extra clause comparing today's weekday to 6 (Sunday) that isn't described anywhere in the spec. Since `days_since_last == 1` was already true in the failing test, the only thing that could route execution to the `else` (reset) branch was that added weekday check.

**Root Cause + Fix:** The streak code was checking `today.weekday() != 6` as part of the condition to increment the streak. Because Python's `date.weekday()` returns 6 for Sunday, this meant that any time the current listen fell on a Sunday — even if the user had listened the day before — the `elif` failed and execution fell through to the `else` branch, which unconditionally resets the streak to 1. The bug only manifests on the specific day of the week (Sunday) that closes out a streak, which is why it read as "the streak keeps resetting" rather than "the streak never increments" — it worked fine Monday–Saturday and only broke on the Sunday transition. The fix was to remove the `and today.weekday() != 6` clause entirely, so incrementing is governed purely by `days_since_last == 1`, matching the docstring.

**Side-effect check:** Reran the full `tests/test_streaks.py` suite (not just the one that had failed) to confirm the other four cases were untouched by the change: new-user-starts-at-1, same-day-no-double-count, and skip-a-day-resets-to-1 all still passed. I also re-read the other two branches (`days_since_last == 0` and the `else`) to confirm the edit only touched the `elif` condition and didn't change when those branches fire.

``Issue #4 — I got notified when a friend added my song to a playlist but not when they rated it``

**Reproduction:** User A shares a song. User B adds that song to a playlist — User A gets a notification. Separately, User B instead rates that song via `POST /songs/<id>/rate` — User A's notifications list (`get_notifications`) stays empty even though the rating was saved successfully (visible via the Rating's own record).

**Navigation strategy:** Since `add_to_playlist` clearly worked, I opened `notification_service.py` and read it top to bottom, comparing `add_to_playlist` against `rate_song` line by line since they're described as parallel "interaction" features in the module docstring. `add_to_playlist` ends with an `if song.shared_by != added_by_user_id: create_notification(...)` block. `rate_song` had no equivalent block — it committed the Rating and returned immediately. I confirmed this wasn't just a different code path by grepping for `create_notification` across the file and finding only one call site (inside `add_to_playlist`), which told me the rating flow was missing the feature outright rather than implementing it differently.

**Root Cause + Fix:** `rate_song` persists the score to the `Rating` table but never calls `create_notification` — the function is simply missing the notify step that its sibling function `add_to_playlist` has. This isn't a broken condition, it's an absent one: the docstring says "Save a user's rating for a song" and that's all the original code did, but the actual feature (per the issue and the app's own pattern) requires notifying the original sharer the same way adding-to-playlist does. The fix mirrors `add_to_playlist`'s existing pattern: after `db.session.commit()`, check `if song.shared_by != user_id` and call `create_notification(user_id=song.shared_by, notification_type="song_rated", body=f"{rater.username} rated your song '{song.title}'.")`.

**Side-effect check:** I checked both branches of the existing `if existing / else` block in `rate_song` — rating a song for the first time and updating a previously-submitted rating — to confirm the new notification fires in both cases (it sits after the branch, so it does). I also checked the case where a user rates their own shared song and confirmed the `song.shared_by != user_id` guard suppresses the self-notification, exactly matching how `add_to_playlist` already avoids notifying someone about their own action.

``Issue #5 — The last song in a playlist never shows up``

**Reproduction:** Create a playlist and add 5 songs to it in order. Call `GET /playlists/<id>/songs`. Expected 5 songs back; actual response contains only the first 4 — the 5th song added (last by position) is missing from the list, though it's confirmed present in the `playlist_entries` table.

**Navigation strategy:** `tests/test_playlists.py::test_playlist_returns_all_songs` asserts `len(songs) == 5` with a comment noting "Bug causes this to return 4" — running `pytest tests/test_playlists.py` reproduced that failure immediately. I opened `playlist_service.get_playlist_songs` and read the SQLAlchemy query first, since that's the part most likely to under-fetch: it joins `Song` to `playlist_entries` on `song_id`, filters by `playlist_id`, and orders by `position` — nothing wrong there, and the query has no `LIMIT`. The bug had to be after the query, in `return [song.to_dict() for song in songs[:-1]]` — `songs[:-1]` slices off the last element of an already-fully-correct, already-correctly-ordered list.

**Root Cause + Fix:** The query itself returns every song in the playlist in the right order — the join and the `order_by(asc(position))` are both correct. The bug is purely the `[:-1]` slice in the return statement, which drops the last element of the list regardless of playlist size (which is also why the empty-playlist case wasn't affected — slicing an empty list still returns an empty list, so that edge case masked the bug in casual testing). The fix was to iterate over `songs` instead of `songs[:-1]`.

**Side-effect check:** Reran `test_playlist_returns_songs_in_order` to confirm the ordering guarantee still held once the 5th song was included (not just that the count was now 5, but that "Track 5" landed last in the list, not first). I also reran `test_empty_playlist_returns_empty_list` to make sure removing the slice didn't change behavior for a playlist with zero songs, since an empty list sliced with `[:-1]` and a full empty list both look the same and I wanted to be sure the fix didn't rely on that coincidence anywhere else.

## Commit History

Work is on the `bugfix/mixtape` branch, as three separate commits (one per fix), each using conventional commit format:

| Commit | Message |
|---|---|
| `b9810e5` | `fix: correct Sunday boundary condition in streak reset logic` |
| `204a176` | `fix: correct dictionary to iterate through in get playlist function` |
| `294c628` | `fix: implemented notification feature in rate_song function` |

Screenshot: ![alt text](<Screenshot 2026-07-07 184659.png>)