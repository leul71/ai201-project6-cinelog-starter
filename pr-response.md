# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

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
<!-- Written at the end — feature overview, design decisions, manual testing steps -->