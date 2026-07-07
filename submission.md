# Mixtape — Submission

## AI usage

I used an AI coding assistant (Claude, via Claude Code) throughout this project. Here is
specifically how I used it, what it helped me understand, and — importantly — where it was
wrong or incomplete and I had to verify things myself.

**Environment & running the app.** I got stuck at the start: my `python -m venv` had been
interrupted and left a half-built venv with a locked `python.exe`, and I kept trying to activate
it with bash-style commands (`source`, `activate.bat`) that don't work in PowerShell. The AI
diagnosed that a leftover Python process was still holding the file, walked me through stopping
that process and recreating the venv, and gave me the correct PowerShell syntax
(`.venv\Scripts\Activate.ps1`, and `$env:FLASK_APP = "app:create_app"` instead of the
`FLASK_APP=... flask run` form). When my browser showed a 404 and then a 405, it explained these
weren't my mistakes — the app has no `/` route, and `/playlists/` is POST-only — and that the
project is a JSON API, not a website. It gave me working GET URLs built from real seeded IDs so I
could actually see responses.

**Understanding the code.** I asked it to explain things I was unsure about: what `uuid` is and
why the models use it for IDs, the difference between a data model and a database table, whether
the models are "OOP with attributes and methods" (they are), and the execution order of a request
(browser → route → service → model → back). I checked each explanation against the actual code and
they held up.

**Codebase map.** I had it read through all the files and draft the codebase map — file
responsibilities, the "add song to playlist → notify sharer" data flow, and the architectural
patterns. I reviewed the claims against the files rather than taking them on faith.

**Bug hunting — where the AI was wrong and I had to verify.** This was where AI help was most
useful *and* most fallible:

- Its first list of bugs included "`rate_song` never sends a notification." After the assignment's
  hint about "inconsistent duplicates," this turned out **not** to be the intended bug — it's a
  missing feature, not a triggerable defect — so we replaced it.
- Its first guess for the duplicates bug was **flat-out wrong**: it predicted `search_songs` would
  return duplicate rows because of an `outerjoin` on the tags table. When we actually ran it,
  SQLAlchemy's ORM had already de-duplicated the entities, so **no** duplicates appeared. That
  failed prediction is what pushed us to keep looking, and the real "inconsistent duplicates" bug
  turned out to be in `add_to_playlist` (a notification created *outside* the dedupe guard).
- Lesson I took from this: the AI's reasoning about database/ORM behavior was plausible but not
  reliable until executed. **Every bug and every fix in this submission was reproduced by actually
  running code against the seeded database**, not accepted on the AI's word.

**Things I double-checked myself.** When I asked whether the playlist slice should be `[:-2]`, the
AI correctly explained it should be *no* slice at all, and I confirmed that with a small list
example. I verified each fix with the test suite (`pytest`) and with boundary cases on both sides
of the bug. I also noticed — with the AI flagging it — that the reproduction runs left extra
notification rows in the dev database, because `create_notification` commits internally, so that
data can be reset with `python seed_data.py`.

**A bug the AI found that's out of scope.** While reproducing Issue #3, it surfaced a fourth
defect: adding a brand-new song raises `IntegrityError` because the relationship append never sets
`playlist_entries.position`. I documented it as an observation but left it unfixed, since it's
outside the three assigned issues.

---

# Codebase Map

Mixtape is a Flask + SQLAlchemy JSON API for sharing songs with friends. Users share
songs, add them to collaborative playlists, rate them, log listens (which build a daily
"streak"), and see what friends are listening to. There is **no HTML front end** — every
endpoint returns JSON.

---

## Top-level files

| File | What it does |
|------|--------------|
| `app.py` | The **application factory**. `create_app()` configures the app (SQLite DB at `sqlite:///mixtape.db`, secret key), initializes SQLAlchemy, registers the four blueprints under their URL prefixes (`/songs`, `/playlists`, `/users`, `/feed`), and calls `db.create_all()` to build the tables from the models. Also runnable directly for local dev (`app.run(debug=True)`). |
| `models.py` | Defines the **data models** (SQLAlchemy classes → database tables) and the plain **association tables**. This is the single source of truth for the app's data shape. |
| `seed_data.py` | Populates the database with sample users, songs, playlists, tags, friendships, and listening events so the endpoints return real data during development. |
| `requirements.txt` | Dependencies: Flask, Flask-SQLAlchemy, SQLAlchemy, python-dotenv, pytest. |
| `tests/` | Pytest suite for the services/routes. |
| `instance/` | Holds the generated `mixtape.db` SQLite file (Flask's instance folder). |

---

## The data layer (`models.py`)

**5 models** (each becomes a table, each has a UUID string `id` and a `to_dict()` serializer):

- **`User`** — `username`, `email`, `listening_streak`, `last_listened_at`. Owns relationships to shared songs, ratings, listening events, notifications, and playlists. Also has a **self-referential many-to-many** `friends` relationship.
- **`Song`** — `title`, `artist`, `album`, `genre`, `shared_by` (FK → user), `share_note`. Has many `ratings`, `listening_events`, and `tags`.
- **`Tag`** — just `id` + unique `name`. Linked to songs many-to-many.
- **`ListeningEvent`** — one row per "user listened to song at time T". This is the raw material for both streaks and the friend feeds.
- **`Rating`** — `user_id`, `song_id`, `score` (1–5). A `UniqueConstraint("user_id", "song_id")` enforces **one rating per user per song** at the DB level.
- **`Notification`** — `user_id` (recipient), `notification_type`, `body`, `read` flag.

**3 association (join) tables**, written as `db.Table(...)` rather than full models because they only link IDs:

- **`friendships`** — links `user_id` ↔ `friend_id` (both FKs to `user`).
- **`song_tags`** — links `song_id` ↔ `tag_id`.
- **`playlist_entries`** — links `playlist_id` ↔ `song_id`, but **adds extra columns**: `position` (explicit song ordering), `added_by`, and `added_at`. So playlist order is stored explicitly, not left to insertion order.

---

## The route layer (`routes/`)

Each file is a Flask **blueprint** registered under a URL prefix in `app.py`. Routes are
deliberately **thin**: parse input, call a service, format the JSON response, translate
`ValueError` into a 400/404.

| File | Prefix | Endpoints |
|------|--------|-----------|
| `routes/songs.py` | `/songs` | `GET /search?q=`, `GET /<id>`, `POST /<id>/rate`, `POST /<id>/listen` |
| `routes/playlists.py` | `/playlists` | `POST /` (create), `GET /<id>`, `GET /<id>/songs`, `POST /<id>/songs` (add song) |
| `routes/users.py` | `/users` | `GET /<id>`, `GET /<id>/streak`, `GET /<id>/notifications`, `POST /notifications/<id>/read` |
| `routes/feed.py` | `/feed` | `GET /<id>/listening-now`, `GET /<id>/activity` |

---

## The service layer (`services/`)

All **business logic and database queries** live here. Routes never touch the DB directly
(the one exception is `routes/users.py`, which does a simple `db.session.get(User, ...)` inline).

| File | Responsibility |
|------|----------------|
| `services/search_service.py` | `search_songs()` (case-insensitive title/artist match via `ilike`), `get_song()`. |
| `services/playlist_service.py` | Create playlists, fetch a playlist, fetch its ordered songs (`get_playlist_songs`), list a user's playlists. |
| `services/notification_service.py` | `create_notification()`, `add_to_playlist()` (adds song **and** notifies the sharer), `rate_song()` (upsert a rating), plus `get_notifications()` and `mark_as_read()`. |
| `services/streak_service.py` | `record_listening_event()` logs a listen and updates the streak; `update_listening_streak()` holds the consecutive-day rules; `get_streak()` reads it. |
| `services/feed_service.py` | `get_friends_listening_now()` (friends active in the last 24h, deduped to one song each) and `get_activity_feed()` (most recent N friend listens). |

---

## Data flow — adding a song to a playlist triggers a notification

This is the clearest end-to-end "an action notifies someone" flow in the app.

1. **HTTP request:** `POST /playlists/<playlist_id>/songs` with JSON body `{"song_id": ..., "added_by": ...}`.
2. **Router:** Flask matches the `/playlists` prefix → the playlists blueprint → `add_song()` in `routes/playlists.py`.
3. **Route (thin):** `add_song()` pulls `song_id` and `added_by` from the JSON, validates they're present (else `400`), then calls the service function `add_to_playlist(playlist_id, song_id, added_by)`.
4. **Service (logic):** `add_to_playlist()` in `services/notification_service.py`:
   - looks up the song, the adding user, and the playlist — raising `ValueError` (→ `400`) if any is missing;
   - appends the song to `playlist.songs` (writing a row into the `playlist_entries` join table) and commits;
   - **if the person adding the song is not the original sharer** (`song.shared_by != added_by`), it calls `create_notification(...)` targeting `song.shared_by`, with type `"song_added_to_playlist"` and a human-readable body like *"darius added your song 'X' to the playlist 'Y'."*
5. **`create_notification()`** inserts a `Notification` row for the sharer and commits.
6. **Back up the stack:** the route returns `{"message": "Song added to playlist"}` with `201`.
7. **Later**, the sharer sees it via `GET /users/<id>/notifications`, which flows route → `get_notifications()` → query the `Notification` table → JSON.

**Data flow — rating a song** (for contrast): `POST /songs/<id>/rate` → `routes/songs.py::rate()` → `notification_service.rate_song()`, which validates the score is 1–5, then **upserts**: if a `Rating` already exists for that user+song it overwrites the score, otherwise it inserts a new one (backed by the DB `UniqueConstraint`).

---

## Patterns I noticed

1. **Strict layered architecture: route → service → model.** Routes only parse input and format output; every query and business rule lives in a service; models define the data. To understand any endpoint, read the matching service function.

2. **`ValueError` is the standard "not found / bad input" signal.** Services raise `ValueError` with a message; routes catch it and turn it into a `404` (for lookups) or `400` (for bad input). No exceptions leak to the client.

3. **App factory + blueprints.** `create_app()` lets tests build a fresh app with test config; blueprints keep each resource's routes in its own file and give them a shared URL prefix.

4. **UUID string primary keys everywhere** (`generate_uuid()` → `uuid4()`), so IDs are globally unique and reveal nothing (no sequential `1, 2, 3`).

5. **Two kinds of "link" tables.** Simple pairings (`friendships`, `song_tags`) are plain `db.Table`s; a link that needs its own attributes (`playlist_entries`, with `position`/`added_by`/`added_at`) still uses `db.Table` but carries extra columns rather than being promoted to a full model.

6. **`ListeningEvent` is a shared foundation.** One event table feeds three separate features — streaks (`streak_service`), "listening now" (24h window), and the activity feed (recent N) — each just querying it differently.

7. **Notifications are side effects of other actions**, not a standalone feature — they're created inside `add_to_playlist` rather than via their own endpoint.

---

## Root cause analysis — three bugs

Each bug below was **reproduced by running the service functions directly against the seeded
database** inside an app context, comparing observed behavior to the docstring's promise.

### Issue #1 — Listening streak resets on Sundays (should increment)

- **Location:** `services/streak_service.py::update_listening_streak()`
- **Faulty line:** `elif days_since_last == 1 and today.weekday() != 6:` — `weekday() == 6` is Sunday, so a listen on a consecutive day that *lands on a Sunday* fails this branch and falls through to the `else`, which **resets the streak to 1**.
- **Condition to trigger:** the current listen must be (a) exactly one calendar day after the last listen **and** (b) on a **Sunday**. Any other day works correctly, which is why the bug is easy to miss.
- **How I reproduced it:** Took a seeded user, set `listening_streak = 5` and `last_listened_at =` Saturday `2026-07-04`, then called `update_listening_streak(user, sunday)` with `sunday = 2026-07-05` (a real Sunday, `weekday() == 6`).
  - **Expected:** streak → `6`. **Actual:** streak → `1`.
  - **Control:** the identical scenario shifted one day earlier (Friday → Saturday) correctly produced `6`, confirming Sunday is the trigger.
- **Fix:** removed the stray day-of-week condition — `elif days_since_last == 1 and today.weekday() != 6:` → `elif days_since_last == 1:`. A single-line change at the root cause; no other logic touched. A consecutive-day listen now increments regardless of weekday.
- **Verification (both sides of the boundary):**
  - Sat → Sun (the bug): now `6` ✓; Sun → Mon: `6` ✓ (Sunday no longer special on either side).
  - Same day (Sun → Sun): unchanged at `5` ✓; gap of 2 days: resets to `1` ✓; first-ever listen: `1` ✓; ordinary weekday consecutive: increments ✓.
  - Existing streak test suite (`pytest -k streak`): **5 passed**.

### Issue #2 — Viewing a playlist silently drops the last song

- **Location:** `services/playlist_service.py::get_playlist_songs()`
- **Faulty line:** `return [song.to_dict() for song in songs[:-1]]` — the `[:-1]` slice removes the final element even though the docstring says it "returns all songs in the playlist."
- **Condition to trigger:** **any** playlist containing at least one song. No special state needed — it is off-by-one on every call. (A playlist with exactly one song returns an empty list.)
- **How I reproduced it:** Called `get_playlist_songs()` for the seeded playlist **"Late Night Vibes"**, which has **7** rows in `playlist_entries`.
  - **Expected:** `7` songs. **Actual:** `6` songs (the last one by `position` is missing).
  - Reachable over HTTP via `GET /playlists/<id>/songs`.
- **Fix:** removed the slice — `return [song.to_dict() for song in songs[:-1]]` → `return [song.to_dict() for song in songs]`. Single-line change at the root cause; the ordering query above it was already correct and left untouched.
- **Verification (both sides of the boundary):**
  - All three seeded playlists now return the full `7` songs, in `position` order, with the last-by-position song included ✓.
  - Empty playlist → `0` songs ✓ (no crash). Single-song playlist → `1` song ✓ — previously this returned an empty list, the worst case of the off-by-one.
  - Existing playlist test suite (`pytest -k playlist`): **3 passed**.

### Issue #3 — Re-adding a song creates duplicate notifications (inconsistent)

- **Location:** `services/notification_service.py::add_to_playlist()`
- **Root cause:** the song append is guarded by `if song not in playlist.songs:`, but the `create_notification(...)` call sits **outside** that guard. So the playlist membership is idempotent (deduped), while the notification is **not** — every call fires another notification. The two side effects are inconsistent.
- **Condition to trigger:** the song is **already in the playlist** *and* the person adding it is **not** the original sharer (`song.shared_by != added_by_user_id`), so the notification branch runs.
- **How I reproduced it:** Picked an existing `(playlist, song)` pair from `playlist_entries` — **"Late Night Vibes"** / **"Midnight Drive"** — with an adder who is not the sharer, then called `add_to_playlist()` three times in a row, counting both the playlist entries and the sharer's `song_added_to_playlist` notifications after each call:

  | call | song in playlist | notifications for sharer |
  |------|------------------|--------------------------|
  | start | ×1 | 2 |
  | after add #1 | ×1 | 3 |
  | after add #2 | ×1 | 4 |
  | after add #3 | ×1 | 5 |

  - **Expected:** re-adding an already-present song is a no-op → no new notification. **Actual:** the song stays deduped at ×1, but a new notification is created on every call.
- **Fix:** moved the `create_notification(...)` block **inside** the `if song not in playlist.songs:` guard (a re-indent, no logic rewrite), so the notification is a side effect of an *actual* addition rather than of every call. Now the notification and the playlist membership are consistent — both happen only when the song is newly added.
- **Verification:**
  - Re-adding an already-present song three times: song stays ×1 and notifications stay **flat** ✓ (previously climbed by one each call).
  - Full test suite (`pytest`): **13 passed** — no related functionality broken.
  - **Boundary I could not exercise end-to-end:** the "genuinely new song → exactly one notification" path is blocked by the *separate, pre-existing* `IntegrityError` documented below (append via the relationship never sets `playlist_entries.position`). My change only re-indents the notification and does not touch that code path, so it neither fixes nor worsens it. That defect is out of scope for these three issues and would need its own fix.

> **Additional observation (found while reproducing #3):** adding a song that is *not yet* in a
> playlist raises `IntegrityError: NOT NULL constraint failed: playlist_entries.position`,
> because `playlist.songs.append(song)` inserts through the relationship without supplying the
> required `position` / `added_by` columns. This is a separate defect in the same function and
> means genuinely new songs cannot be added via this endpoint at all.

### Patterns among the bugs

All three are **logic errors hidden behind correct-looking structure**, not crashes or typos: an
off-by-one slice, a stray day-of-week condition, and a side effect placed outside its guard.
Each one contradicts its own docstring, which is what made them findable by reading the code and
then reproducing against the seed data.

---

## Commit history

Each bug fix is an isolated commit on the `bugfix/mixtape` branch, with a meaningful `Fix Issue #N:`
message — no fixes bundled together:

![git log --oneline on the bugfix/mixtape branch showing one commit per bug fix](![git-log.png](image.png))
