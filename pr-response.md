# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** 
save_to_watchlist() to add_to_watchlist() in services/watchlist_service.py 
**How I verified:**
I used the editor's find-all-reference.

## Comment 2 — Deduplication
**What I did:**
create AlreadyInWatchlistError(Exception) in watchlist_service. Add a check for existing film in watchlist following collection_service format.
**How I verified:**
Run pytest /tests -v and all the test passed

## Comment 3 — Missing test
**What I did:**
Created `tests/test_watchlist.py` with `t
est_add_to_watchlist_nonexistent_film_raises`, following
`test_add_to_collection_nonexistent_film_raises` in `test
_collection.py` — same `app`/`sample_user` fixtures, asse
rting `FilmNotFoundError` is raised for a fake UUID film_
id.
**How I verified:** 
Ran `pytest tests/test_watchlist.py -
v` — passed.

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default.
**Reasoning:** Primarily the discovery factor and sociality — default
ing to public allows others to discover and share interest among frie
nds and family. Making it public by default is what makes the app fee
l social. Users who want privacy can change it in settings.
**Tradeoff acknowledged:** The tradeoff is privacy — some users may w
ant to hide their watchlist, for example because of the genre.

## Comment 5 — Sort order
**what i did**
services/watchlist_service.py, in get_watchlist(): I swapped Film.title.asc() → WatchlistEntry.date_added.desc()
**My position:** Switch to date-added order (descending — most recent first).
**Reasoning:** A watchlist is inherently time-ordered — users add films as they discover them and want to see what they most recently queued up, not an alphabetical index. This also brings get_watchlist() in line with get_collection(), which already sorts by CollectionEntry.date_added.desc(), making the two similar features behave consistently.
**Engagement with reviewer's point:** Agreed with the reviewer's reasoning outright — alphabetical sort was the wrong default for a "what do I want to watch" list, and there's no strong counter-argument for keeping it.

## Comment 6 — Rebase
**What conflicted:** Two things. First, `.gitignore` had an add/add conflict — both `main` (via the `chore: add .gitignore` commit) and my branch created the file independently with slightly different content; resolved by merging both lists together. Second, and more significantly, `models.py` had a conflict between `main`'s UUID refactor (`Film.id`/`CollectionEntry.film_id` changed from `Integer` to `String(36)`) and my branch's addition of the `WatchlistEntry` model (which still used `db.Integer` for `film_id`, since it was written pre-refactor). My first attempt at resolving this conflict accidentally dropped the `WatchlistEntry` class entirely instead of merging it in with the corrected type, which silently broke every import of `services/watchlist_service.py` (it imports `WatchlistEntry` from `models`).
**How I resolved it:** Re-added the `WatchlistEntry` model to `models.py` with `film_id` as `db.Column(db.String(36), db.ForeignKey("film.id"), ...)` to match the UUID refactor, added a `watchlist_entries` relationship/backref on `Film` (matching the existing `collection_entries` pattern) so `entry.film` resolves correctly in `get_watchlist()`, and updated the stale `film_id (int)` docstring in `add_to_watchlist()` to `film_id (str): UUID of the film.`
**How I verified no conflict remains:** Ran `pytest tests/ -v` — all 5 tests pass, including the new `test_watchlist.py` test, confirming the model import and the full `add_to_watchlist`/`get_watchlist` flow work against the UUID schema. Also confirmed with `grep -n "Integer" models.py services/watchlist_service.py` that the only remaining `Integer` columns are `Film.year` and `CollectionEntry.rating`, which are correctly integers (not IDs).

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->