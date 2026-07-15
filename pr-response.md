# PR Response Doc — CineLog Watchlist Feature

## AI Usage
Used Claude Code throughout this project as a reviewer/guide, not as the author of the core logic:
- **Codebase orientation:** Had it explain what `add_to_watchlist()` did before each change, and locate all call sites of `save_to_watchlist()` before renaming, so I knew the blast radius before editing.
- **Reviewing my own code, not writing it:** For Comment 2 (deduplication), I wrote the duplicate-check logic myself. AI caught two real bugs in my draft before I committed: (1) I had queried `CollectionEntry`/`AlreadyInCollectionError` instead of `WatchlistEntry`/a new watchlist-specific exception, which meant the check silently did nothing for actual watchlist duplicates; (2) a leftover copy-pasted error message that said "already in this user's collection" instead of "watchlist." I fixed both myself after it pointed them out.
- **Stress-testing the design arguments (Comments 4 & 5):** For Comment 4 (default visibility), I asked for the strongest argument on both sides of `public=True` vs `public=False` before picking a position, so I wasn't just defending my first instinct. I wrote the final reasoning myself in my own words. For Comment 5 (sort order), I agreed with the reviewer's reasoning directly and used AI mainly to confirm the code change matched the existing `get_collection()` pattern.
- **Rebase debugging:** During Comment 6's rebase, my first conflict resolution in `models.py` silently deleted the entire `WatchlistEntry` model instead of merging it in with the corrected UUID type. AI caught this by running the test suite and tracing the resulting `ImportError` back to the dropped model, which I then restored with the correct `String(36)` UUID foreign key.
- **Commit hygiene:** Asked for the correct Conventional Commit prefix (`feat`/`fix`/`refactor`/`test`/`docs`) for each change based on `CONTRIBUTING.md`'s rules, and for help planning the final interactive rebase (which commits to reword vs. fixup) to get one logical change per commit.

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

### What this feature does
Adds a watchlist to CineLog so users can save films they want to watch later, separate from their collection of films already watched. It introduces a `WatchlistEntry` model, an `add_to_watchlist()` / `get_watchlist()` service layer, and two endpoints: `POST /watchlist/<user_id>/add` to save a film, and `GET /watchlist/<user_id>` to view it.

### Design decisions
- **Default visibility (`public=True`):** Watchlists default to public so friends and family can discover what a user wants to watch and share interest around it — this is what makes the feature social rather than a private, unused list. Users who prefer privacy can change this per-entry in settings. Tradeoff: some users may want their watchlist hidden by default (e.g. due to genre), and that group has to opt out rather than opt in.
- **Sort order (date added, descending):** `get_watchlist()` returns films most-recently-added first, rather than alphabetically. A watchlist is inherently time-ordered — users care most about what they just queued up — and this also matches `get_collection()`'s existing `date_added` sort, keeping the two features consistent.

### Manual testing
1. Start the app: `python app.py`
2. Create a user and a film (or use existing seed data), noting their UUIDs.
3. Add a film to the watchlist:
   ```
   POST /watchlist/<user_id>/add
   Body: { "film_id": "<film_uuid>" }
   ```
   Confirm a `201` response with the new entry.
4. View the watchlist: `GET /watchlist/<user_id>` — confirm the film appears.
5. Repeat step 3 with the same `film_id` — confirm it now returns an error instead of creating a duplicate.
6. Add a second film, then `GET /watchlist/<user_id>` again — confirm the most recently added film appears first (date-added order, not alphabetical).
7. Try `POST /watchlist/<user_id>/add` with a nonexistent `film_id` — confirm a `FilmNotFoundError`-driven error response, not a raw database error.

## Git Log Screenshot

![Git log showing conventional commits](https://github.com/Thymngn/ai201-project6-cinelog-starter/blob/feature/watchlist/Screenshot%202026-07-15%20000037.png)
