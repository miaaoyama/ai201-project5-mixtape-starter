# Project 5 – Mixtape Bug Hunt Submission

## Codebase Map

### Main Files and Responsibilities

* **app.py** – Creates the Flask application, initializes the database, and registers the application's routes.
* **models.py** – Defines the SQLAlchemy models and relationships used throughout the application, including users, songs, playlists, notifications, ratings, and listening events.
* **routes/**

  * `songs.py` – Handles song sharing, searching, and rating endpoints.
  * `playlists.py` – Handles playlist creation and playlist song management.
  * `users.py` – Handles user profiles, notifications, and listening streaks.
  * `feed.py` – Handles the Friends Listening Now feed and activity feed.
* **services/**

  * `streak_service.py` – Updates and retrieves listening streaks.
  * `feed_service.py` – Builds the Friends Listening Now and activity feeds.
  * `search_service.py` – Performs song search operations.
  * `notification_service.py` – Creates and retrieves notifications.
  * `playlist_service.py` – Retrieves playlists and playlist songs.
* **tests/** – Contains automated tests that verify playlist, search, and streak functionality.
* **seed_data.py** – Populates the database with sample data for testing.

### Data Flow Example – Rating a Song

1. The user submits a request to rate a song.
2. The request is handled by the route in `routes/songs.py`.
3. The route calls `notification_service.rate_song()`.
4. The service validates the user, song, and rating.
5. The rating is saved to the database.
6. If appropriate, a notification is created for the user who originally shared the song.
7. The updated rating is returned to the route, which sends the response to the client.

### Pattern Observed

The project follows a layered architecture. Route files are responsible for handling HTTP requests and responses, while nearly all business logic lives inside the `services` directory. Models represent the database structure, making each layer responsible for a single purpose.

---

# Root Cause Analysis

## Issue #1 – My listening streak keeps resetting

**How I reproduced it:** I ran `pytest tests/test_streaks.py`. The failing test simulated a user listening on Saturday, June 15, 2024, and then again on Sunday, June 16, 2024. The streak stayed at 1 instead of increasing to 2.

**How I found the root cause:** I traced the issue from the test failure to `tests/test_streaks.py`, then opened `services/streak_service.py` and inspected `update_listening_streak()`. The suspicious line was the condition `elif days_since_last == 1 and today.weekday() != 6:` because Sunday is represented as `6` by Python’s `weekday()` method.

**The root cause:** The function only incremented the streak if one day had passed and the current day was not Sunday. Because Sunday has `weekday()` value `6`, Saturday-to-Sunday listening was incorrectly sent to the reset branch.

**Your fix and side-effect check:** I removed the unnecessary Sunday condition so any consecutive-day listen increments the streak. I reran `pytest tests/test_streaks.py` to check same-day, first-listen, missed-day, and Sunday boundary behavior. All streak tests passed.

---

## Issue #4 – I got notified when a friend added my song to a playlist but not when they rated it

**How I reproduced it:** I compared the two related behaviors in `services/notification_service.py`. `add_to_playlist()` created a notification when someone interacted with another user’s shared song, but `rate_song()` only saved the rating and returned it.

**How I found the root cause:** I followed the route described in the README: rating a song goes through `routes/songs.py` and then `notification_service.rate_song()`. I compared that function line-by-line with `add_to_playlist()`, which was the working notification path. The missing step was that `rate_song()` never called `create_notification()`.

**The root cause:** The rating flow saved or updated the `Rating` record but did not notify the original song sharer. The notification architecture existed, but the rating path did not use it.

**Your fix and side-effect check:** I added notification creation after the rating is committed, only when the rater is not the original song sharer. This matches the playlist notification pattern and avoids self-notifications. I reran `pytest tests/` to confirm existing playlist, search, and streak behavior still passed.

---

## Issue #5 – The last song in a playlist never shows up

**How I reproduced it:** I ran `pytest tests/test_playlists.py`. The failing test seeded a playlist with five songs, but `get_playlist_songs()` returned only four. The returned titles stopped at `Track 4` instead of including `Track 5`.

**How I found the root cause:** I traced the failure from `tests/test_playlists.py` to `services/playlist_service.py`, specifically `get_playlist_songs()`. The query correctly retrieved songs in playlist position order, but the return statement used `songs[:-1]`.

**The root cause:** The slice `songs[:-1]` returns every item except the last one. This caused the final playlist song to be omitted every time, even though the database query returned it.

**Your fix and side-effect check:** I changed the return statement to iterate over `songs` instead of `songs[:-1]`. I reran `pytest tests/test_playlists.py` to confirm all songs were returned and still ordered correctly. All playlist tests passed.


## Testing

After applying all fixes, I ran:

`pytest tests/`

Result:

**13 tests passed.**

---

