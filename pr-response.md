# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used Claude throughout this project for orientation and troubleshooting. Early on, I had it walk through `collection_service.py` and `test_collection.py` to confirm the dedup and testing patterns before writing my own versions in `watchlist_service.py` and `test_watchlist.py`. The most valuable use was during the rebase (Comment 6): after `git rebase origin/main` completed without any conflict in `models.py`, I used AI to help me verify the rebase hadn't silently broken anything rather than assuming "no conflict = success." That check caught that the `WatchlistEntry` class had been dropped entirely from `models.py` during the rebase, which I then fixed by re-adding it with the UUID `film_id` type. I wrote my own reasoning for Comments 4 and 5 independently — the AI wasn't used to draft either argument.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in services/watchlist_service.py, matching the verb_to_noun convention used by add_to_collection(). Updated the one call site in routes/watchlist.py.
**How I verified:** Ran `grep -rn "save_to_watchlist" --include="*.py" .` to confirm no remaining references, and `pytest tests/ -v` to confirm the existing test suite still passes.

## Comment 2 — Deduplication
**What I did:** Added a check in add_to_watchlist() that queries for an existing WatchlistEntry with the same user_id/film_id before creating a new one, raising AlreadyInWatchlistError if found. Mirrors add_to_collection()'s pattern in collection_service.py.
**How I verified:** Added test_add_to_watchlist_duplicate_raises, which adds a film twice and confirms the second call raises AlreadyInWatchlistError and only one entry exists in the DB.

## Comment 3 — Missing test
**What I did:** Created tests/test_watchlist.py with test_add_to_watchlist_nonexistent_film_raises, following test_add_to_collection_nonexistent_film_raises's structure — using a fake ID with app_context() and pytest.raises.
**How I verified:** Ran pytest tests/ -v, confirms all 6 tests pass including the new ones.

## Comment 4 — Default visibility
**My position:** I would not agree with this being set to TRUE from default for some reasons but most are not 
real saftery concern but it is more privacy related.
**Reasoning:** My reasoning for this not being set to TRUE on default is some people might not feel safe having others know what they like to watch or want to watch. This would make them stay to what their friends/peers are watching and make them align their watchlist with their peers, this isn't ideal in my opinion because we need uniqueness and that is what makes us us. Though having the option maybe to suggest movies to others would be a much better idea.
**Tradeoff acknowledged:** The trade off for this woul be limiting what people watch or what they are exposed to. Sometimes it can be hard for people to find what they would want to watch but by making this field TRUE they might get intereted by others watchlist.

## Comment 5 — Sort order
**My position:** I would keep the date added as the sorting order for this.
**Reasoning:** My reasoning for keeping this sorted by date added is because a good amount of people put items stuff on their watchlist and forget but if they recently added someting to their list, they are more likely to watch as they would be more interested because it is a new thing and new things are usually exciting to people but having it alphabetically could be beneficial to easily finding stuff and that is only in the case of if you know what you would already want to watch.
**Engagement with reviewer's point:** This option would keep people engaged the most becasuse the biggest providers like netflix use this approach, I can also say that I have had movies I added to my watchlist and not watched as the excitment for them fades away the more the time elapsed.

## Comment 6 — Rebase
**What conflicted:** Running `git rebase origin/main` produced a conflict only in `.gitignore` (main added `.pytest_cache/` to its ignore list). However, the rebase silently dropped the `WatchlistEntry` model entirely from `models.py` — since main's version of that file predated the WatchlistEntry class, git replayed main's version of the file for one commit without flagging a conflict, because the changes didn't touch the same lines.
**How I resolved it:** Resolved the `.gitignore` conflict by keeping both ignore patterns. After the rebase completed, I noticed `models.py` no longer contained `WatchlistEntry` and confirmed via `grep -rn "WatchlistEntry"` that `watchlist_service.py` and `test_watchlist.py` still referenced it. I re-added the `WatchlistEntry` class to `models.py`, changing `film_id` from `db.Integer` to `db.String(36)` to match `Film.id`'s new UUID type post-refactor. I also updated the fake film_id in `test_add_to_watchlist_nonexistent_film_raises` from an integer placeholder to a UUID-shaped string for consistency.
**How I verified no conflict remains:** Ran `pytest tests/ -v` — all 6 tests pass. Confirmed the app boots cleanly with `python app.py` (no ImportError). Ran `git log --oneline` to confirm the branch history is a clean linear sequence on top of main's UUID refactor commit, with no merge commits.

## PR Description
## What this adds
Adds a watchlist feature so users can save films they want to watch later. Includes a `WatchlistEntry` model, service functions (`add_to_watchlist`, `get_watchlist`), and REST endpoints (`GET /watchlist/<user_id>`, `POST /watchlist/<user_id>/add`).

## Changes from review
- Renamed `save_to_watchlist()` to `add_to_watchlist()` to match the project's verb_to_noun convention
- Added deduplication so a film can't be added to the same watchlist twice (raises `AlreadyInWatchlistError`)
- Added a test for adding a nonexistent film (`FilmNotFoundError`)
- Rebased on main to pick up the integer-to-UUID film ID migration; restored `WatchlistEntry.film_id` as a UUID string to match

## Design decisions

**Default visibility (`public`):** I'd default watchlist entries to private rather than public. A watchlist reveals ongoing taste and intent in a way a completed collection doesn't, and defaulting to public risks nudging people toward conforming their list to what peers are watching rather than what they're actually curious about. The tradeoff is losing some of the social discovery value a public-by-default list would offer — that could be recovered later with an opt-in "suggest to friends" feature instead of a default.

**Sort order:** I'd keep watchlist entries sorted by date added (descending) rather than alphabetically. Most people add films to a watchlist in a moment of interest, and that interest tends to fade with time — surfacing recent adds first keeps the list matched to what someone's actually likely to watch next, similar to how Netflix and other streaming platforms default. Alphabetical sort is better suited to someone who already knows exactly what they're looking for, which is a secondary use case here.

## How to test manually
1. Start the app: `python app.py`
2. Create a user and film via the existing endpoints (or directly via a Python shell using `create_app`/`db`)
3. Add a film to the watchlist:

POST /watchlist/<user_id>/add

Body: { "film_id": "<uuid>" }

4. Confirm a 201 response with the new entry
5. Repeat the same POST with the same `film_id` — confirm it fails with `AlreadyInWatchlistError` rather than creating a duplicate
6. Retry with a made-up UUID for `film_id` — confirm it raises `FilmNotFoundError`
7. View the watchlist: `GET /watchlist/<user_id>` — confirm the film appears with `date_added` and `public` fields
8. Run the automated test suite: `pytest tests/ -v` — all 6 tests should pass