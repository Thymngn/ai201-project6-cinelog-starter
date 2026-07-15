# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** 
save_to_watchlist() to add_to_watchlist() in services/watchlist_service.py 
**How I verified:**
I used the editor's find-all-reference.

## Comment 2 — Deduplication
**What I did:** Added a lookup for an existing `WatchlistEntry` with the same `user_id`/`film_id` in `add_to_watchlist()` before creating a new entry, raising a new `AlreadyInWatchlistError` if one is found. Modeled on `add_to_collection()`'s `AlreadyInCollectionError` check in `collection_service.py`, but scoped to `WatchlistEntry` since the watchlist and collection are separate tables — a film can be in both without conflict.
**How I verified:** Ran `pytest tests/ -v` after the change to confirm the existing suite still passed, then had the logic reviewed line-by-line to catch that an early draft mistakenly queried `CollectionEntry`/`AlreadyInCollectionError` instead of the watchlist's own table/exception.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` with `test_add_to_watchlist_nonexistent_film_raises`, following `test_add_to_collection_nonexistent_film_raises` in `test_collection.py` — same `app`/`sample_user` fixtures, asserting `FilmNotFoundError` is raised for a fake UUID film_id.
**How I verified:** Ran `pytest tests/test_watchlist.py -v` — passed.

## Comment 4 — Default visibility
**My position:**
**Reasoning:**
**Tradeoff acknowledged:**

## Comment 5 — Sort order
**My position:**
**Reasoning:**
**Engagement with reviewer's point:**

## Comment 6 — Rebase
**What conflicted:**
**How I resolved it:**
**How I verified no conflict remains:**

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->