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

**How I reproduced it:** I ran `pytest tests/test_streaks.py`. The failing test created a Saturday listening event followed by a Sunday listening event. The streak stayed at 1 instead of increasing to 2.

## Issue #4 – Song rating notifications were missing

**How I reproduced it:** I compared the working playlist notification path with the rating path. `add_to_playlist()` created a notification after a user added someone else's song, but `rate_song()` only saved the rating and returned it. The missing condition was rating a song shared by another user.

## Issue #5 – Final playlist song missing

**How I reproduced it:** I ran `pytest tests/test_playlists.py`. The playlist test seeded a playlist with 5 songs, but `get_playlist_songs()` returned only 4 songs. The returned titles stopped at Track 4 instead of including Track 5.

## Testing

After applying all fixes, I ran:

`pytest tests/`

Result:

**13 tests passed.**

---

## AI Assistance

I used Claude to help understand the project structure, trace the flow between routes and services, explain unfamiliar functions, and discuss possible root causes. I reproduced each bug before making changes and verified each fix by running the project's automated tests.
