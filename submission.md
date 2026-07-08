# Submission Doc

## AI Usage

During this project, I used AI tools to accelerate debugging, understand the responsibilities of unfamiliar code modules, and trace data flow anomalies. I began by feeding Gemini unknown files (such as models.py) and asked it to summarize what each module was responsible for, detailing its main functions and what each one did.
When pytest failed with stack traces like AssertionError: assert 4 == 5 or assert 1 == 2, I asked the AI to trace the symptoms back to the logical source within the code layer. The AI was incredibly helpful in contextualizing implicit language errors and date mechanics. It immediately pointed out that a Python array slice of [:-1] inherently trims the final index of a collection, which helped me instantly connect why the last song of a playlist was missing from the backend data payload.
While Gemini laid a good foundation for my understanding, its initial code fixes were sometimes incomplete, so I had to step in. While fixing the playlist indexing error, the AI provided a conceptual code block that accidentally omitted an explicit return keyword at the end of the array builder. When I blindly ran the tests, they crashed with a worse error (TypeError: object of type 'NoneType' has no len()). I had to inspect the code manually, notice that the function was dropping execution flow, and restore the proper return statement myself.

## Codebase Map

### models.py
**What it is:**
Defines 5 SQLAlchemy models: User, Song, Playlist, PlaylistSong, and Notification. The PlaylistSong table is a join table that adds an order column — songs in a playlist have an explicit position, not just insertion order.

### streak_service.py
**What it is:**
A service module responsible for managing and calculating user listening streaks.

**Main Functions:**
- record_listening_event(user_id, song_id): Registers a new music playback history record and triggers a streak validation check for the user.
- update_listening_streak(user, now): Evaluates consecutive active days, maintaining or incrementing the streak count if a user listens daily, or resetting it to 1 if a calendar day is skipped.
- get_streak(user_id): Fetches a user's current consecutive day streak integer directly from their profile.

### search_service.py
**What it is:**
A service module responsible for lookup operations and discovering music tracks.

**Main Functions:**
- search_songs(query): Queries the song database to find track records matching a title or artist name using case-insensitive partial matching, returning them along with descriptive category tags.
- get_song(song_id): Fetches a single music track record by its unique database identifier or throws an error if it doesn't exist.

### notification_service.py
**What it is:**
A service module responsible for orchestrating user interactions and generating social activity notifications.

**Main Functions:**
- create_notification(user_id, notification_type, body): Logs a new alert message entry targeting a specific user account in the database.
- add_to_playlist(playlist_id, song_id, added_by_user_id): Records a song insertion into a playlist and pushes a notification alert to the user who originally shared that song.
- rate_song(user_id, song_id, score): Saves or updates a 1-to-5 review score for a track and handles corresponding database persistence updates.
- get_notifications(user_id, unread_only): Pulls a chronological feed of alerts for a user profile, with an optional filter to isolate unread notifications.
- mark_as_read(notification_id): Updates a specific notification entry's flag state to mark it as read by the recipient.

### playlist_service.py
**What it is:**
A service module responsible for creating, exploring, and managing user-curated music playlists.

**Main Functions:**
- create_playlist(name, created_by_user_id, is_collaborative): Provisions a new playlist associated with a specific creator account and structural sharing rules.
- get_playlist_songs(playlist_id): Fetches the sequential tracks mapped to a specific collection sorted chronologically by their entry position.
- get_playlist(playlist_id): Retrieves the metadata profile describing a compilation (excluding its full song list).
- get_user_playlists(user_id): Aggregates a listing of all collections designed and established by a specific user profile.

### feed_service.py
**What it is:**
A service module responsible for generating real-time and historical social network timelines.

**Main Functions:**
- get_friends_listening_now(user_id): Builds a deduplicated overview listing friends who have listened to tracks within an active 24-hour time threshold.
- get_activity_feed(user_id, limit): Compiles a raw history timeline showing recent track playbacks from an account's friend network, capped at a specific row length.

### test_streaks.py
**What it is:**
An automated test suite module using pytest to validate listening streak rule expectations.

**Main Functions:**
- app() & user(app): Setup fixtures initializing an isolated database configuration for test isolation.
- test_streak_starts_at_1_for_new_user(): Validates that first-time playbacks correctly initialize a user's streak value to 1.
- test_streak_increments_on_consecutive_day(): Ensures listening on consecutive days successfully increments the tracked count.
- test_streak_does_not_double_count_same_day(): Checks that duplicate playbacks within the same calendar day leave the count unchanged.
- test_streak_resets_after_skipped_day(): Verifies that skipping a calendar day accurately resets a running streak back to 1.
- test_streak_increments_on_sunday(): Confirms that streak counts accurately survive weekend boundaries moving from Saturday to Sunday.

### Data flow
user rates a song: POST /songs/<id>/rate in routes/songs.py calls notification_service.notify_song_rated(). That function creates a Notification record for the song's original sharer. There's no separate rating model — the rating is stored directly on the Song.

Pattern I noticed: every route delegates immediately to a service function. The routes do input parsing and response formatting; all business logic lives in services/."

## Bug Fixes

### Bug 1 (Streak Service)

**Issue number and title:**
Issue #1 - My listening streak keeps resetting.

**How you reproduced it:** Ran the automated test suite using pytest tests/test_streaks.py. The specific test case test_streak_increments_on_sunday failed with assert 1 == 2, proving that listening on Saturday followed by Sunday reset the counter rather than incrementing it.

**How you found the root cause:** Investigated services/streak_service.py to examine how consecutive calendar days are calculated. The logic relied on Python's datetime.weekday() method to track the turn of the week.

**The root cause:** Python's datetime.weekday() returns 6 for Sunday, but the streak calculation code was checking weekday() == 0 to detect the boundary of a new week, which actually matches Monday. Because of this, a streak update on a Sunday was treated as an out-of-order mid-week entry, preventing the streak from rolling over and incrementing across the weekend.

**Fix:** Updated the week boundary evaluation from datetime.weekday() == 0 to utilize datetime.isoweekday() == 7. The ISO convention maps Sunday to 7 cleanly, allowing the conditional block to accurately recognize the Sunday transition.

**Side-effect check:** Executed pytest tests/test_streaks.py to ensure test_streak_increments_on_sunday passes and confirmed that weekday streak increments remain uncorrupted.

### Bug 2 (Feed Service)

**Issue number and title:**
Issue #2 - Friends Listening Now shows people from yesterday.

**How you reproduced it:** 
Started the local application server and observed the network payloads or test logs returning listening events older than the 24-hour window on the "Friends Listening Now" endpoint.

**How you found the root cause:**
Traced the data flow within services/feed_service.py inside the get_friends_listening_now(user_id) function. Examined how cutoff was initialized and evaluated against the database entity field ListeningEvent.listened_at.

**The root cause:**
The application calculated the time window using a timezone-aware object (datetime.now(timezone.utc) - RECENT_THRESHOLD), while the backend SQLite database stores ListeningEvent.listened_at timestamps as naive datetimes. This timezone-aware vs. timezone-naive mismatch caused SQLAlchemy's filter expression (ListeningEvent.listened_at >= cutoff) to evaluate incorrectly, letting events older than 24 hours leak into the active feed.

**Fix:**
Modified the cutoff calculation to use datetime.utcnow() - RECENT_THRESHOLD. This ensures that both sides of the database comparison use matching, naive UTC datetime structures.

**Side-effect check:**
Verified the endpoint payload to ensure only events within the true 24-hour window are displayed, and ran the broader feed test suite to confirm no regression in the general activity timeline.

### Bug 5 (Playlist Service)

**Issue number and title:**
Issue #5 - The last song in a playlist never shows up.

**How you reproduced it:** 
Ran the automated test suite using pytest tests/test_playlists.py. The test test_playlist_returns_all_songs failed with an AssertionError: assert 4 == 5, confirming the final item was dropping off.

**How you found the root cause:**
Navigated to services/playlist_service.py and traced the data flow of the get_playlist_songs(playlist_id) function. Looking closely at the return statement, the slice syntax immediately stood out as the precise location of the error.

**The root cause:**
In services/playlist_service.py, the get_playlist_songs function fetches all matching song tracks sequentially from the database but returns them using a Python list slice: return [song.to_dict() for song in songs[:-1]]. The [:-1] slice explicitly excludes the final element of the evaluated array, causing the application to always omit the last track of any playlist.

**Fix:**
Removed the [:-1] slice parameter entirely, changing the statement to return [song.to_dict() for song in songs].

**Side-effect check:**
Reran pytest tests/test_playlists.py to confirm both test_playlist_returns_all_songs and test_playlist_returns_songs_in_order now pass perfectly without breaking any adjacent playlist functionality.

**Screenshot of git log --oneline**
![Screenshot 2026-07-07 at 11.08.44 PM.png](Screenshot%202026-07-07%20at%2011.08.44%20PM.png)