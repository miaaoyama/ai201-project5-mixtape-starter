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

## Issue #1 – Listening streak resets on Sunday

**Issue:** Listening streaks reset when a user listened on Saturday and again on Sunday.

**Root Cause:** The streak logic contained an additional condition that prevented streaks from incrementing whenever the current day was Sunday (`today.weekday() != 6`).

**How I Reproduced It:** Running `pytest tests/test_streaks.py` caused the Sunday streak test to fail.

**Fix:** Removed the unnecessary Sunday check so the streak increments whenever the user listened exactly one day earlier.

**Verification:** All streak tests passed after the change.

---

## Issue #4 – Song rating notifications were missing

**Issue:** Users received notifications when their songs were added to playlists but not when someone rated their songs.

**Root Cause:** The `rate_song()` function saved ratings but never created a notification for the original song owner.

**How I Reproduced It:** Comparing `add_to_playlist()` with `rate_song()` showed that only the playlist function created notifications.

**Fix:** Added notification creation after saving a rating whenever someone other than the original owner rated the song.

**Verification:** All project tests continued to pass after the change.

---

## Issue #5 – Final playlist song missing

**Issue:** The final song in every playlist was omitted.

**Root Cause:** `get_playlist_songs()` returned `songs[:-1]`, which slices off the final element of the list.

**How I Reproduced It:** Running `pytest tests/test_playlists.py` showed that only four songs were returned instead of five.

**Fix:** Returned the complete list by removing the slice.

**Verification:** All playlist tests passed after the change.

---

## Testing

After applying all fixes, I ran:

`pytest tests/`

Result:

**13 tests passed.**

---

## AI Assistance

I used Claude to help understand the project structure, trace the flow between routes and services, explain unfamiliar functions, and discuss possible root causes. I reproduced each bug before making changes and verified each fix by running the project's automated tests.
