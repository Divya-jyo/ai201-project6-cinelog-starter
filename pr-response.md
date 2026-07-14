# PR Response Doc - CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end -->

## Comment 1 - Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's verb_to_noun naming convention (consistent with `add_to_collection()`). Updated both references in `routes/watchlist/watchlist.py` (the import and the call site).
**How I verified:** Ran `findstr /s /i "save_to_watchlist" *.py` across the whole project before and after the change - confirmed all three references were found initially, and the search returned empty afterward, confirming no leftover references. Ran the full test suite (`pytest tests/ -v`) - all 4 tests still passed.

## Comment 2 - Deduplication
**What I did:** Added a duplicate check in add_to_watchlist() before creating a new WatchlistEntry, following the same pattern as add_to_collection() in collection_service.py - query for an existing entry with the same user_id and film_id, and raise an error if found. Defined a new AlreadyInWatchlistError exception class locally in watchlist_service.py rather than importing an equivalent from collection_service.py, since watchlist and collection are conceptually separate features and I didn't want to couple their error handling together.
**How I verified:** Ran the full test suite (pytest tests/ -v) to confirm nothing broke on import. Will add a dedicated test for this behavior as a stretch goal if time allows.

## Comment 3 - Missing test
**What I did:** Created tests/test_watchlist.py and wrote test_add_to_watchlist_nonexistent_film_raises, mirroring test_add_to_collection_nonexistent_film_raises from test_collection.py. Reused the same app and sample_user fixture pattern. Used a fake integer film_id (99999) instead of a fake UUID since Film.id is still an integer at this point in the branch history (pre-rebase). Along the way I also caught and fixed a bug where add_to_collection() had accidentally been changed to raise AlreadyInWatchlistError instead of AlreadyInCollectionError.
**How I verified:** Ran pytest tests/test_watchlist.py -v to confirm the new test passes, then ran the full suite (pytest tests/ -v) - all 5 tests pass.

## Comment 4 - Default visibility
**My position:** Watchlists should default to public (public=True), keeping the current behavior.
**Reasoning:** CineLog is a social film-tracking app - the value of the platform comes from seeing what friends are watching and want to watch. Defaulting watchlists to public matches how people naturally use a social app like this and supports discovery (friends can see what you're excited about and get recommendations from each other). Requiring users to manually opt in to sharing would add friction and likely reduce the social engagement that makes the feature useful in the first place.
**Tradeoff acknowledged:** Some users may not immediately realize their watchlist is visible to others by default, which could feel like a surprise if they were expecting a private "notes to self" list. This is a real cost, but it's mitigated by the fact that the public field already exists on WatchlistEntry, meaning users (or future UI) can toggle it to private per-item if they want more control.

## Comment 5 - Sort order
**My position:** I'd like to keep the current alphabetical order (Film.title.asc()) rather than switching to date-added order.
**Reasoning:** A watchlist is meant to be referenced over time, and it can grow to include many films. Alphabetical order makes it much easier to scan for a specific title - for example, checking whether you've already added a movie before adding it again, or finding something specific you remember wanting to watch. Date-added order is great for a short list where you mainly care about "what did I just add," but as the list grows, newer entries bury older ones that a user may still be planning to watch, making it harder to locate a specific film.
**Engagement with reviewer's point:** I agree that most users likely want to see recently added films easily accessible, but I don't think that requires making it the default sort - it could instead be offered as a secondary sort option in the future (e.g., a query parameter) without changing the default. For now, I'm proposing we keep alphabetical as the default since it optimizes for findability as the list scales, while the maintainer's date-added preference optimizes for recency at the cost of long-term scannability.

## Comment 6 - Rebase
**What conflicted:** During git rebase upstream/main, there was a direct conflict in .gitignore (both branches added one independently), which I resolved by merging both sets of entries. After the rebase completed, running the test suite revealed a deeper issue: the WatchlistEntry model had been silently dropped from models.py during the rebase - it wasn't flagged as a text conflict because main's refactor commit didn't touch the exact same lines, but the end result was that WatchlistEntry no longer existed after rebasing.
**How I resolved it:** I re-added the WatchlistEntry class to models.py, updating film_id from db.Integer to db.String(36) to match the new UUID-based Film.id, following the same pattern already used by CollectionEntry. I also added the missing watchlist_entries relationship on the Film model, and updated a stale docstring in watchlist_service.py that still referenced the old integer film_id.
**How I verified no conflict remains:** Ran git status to confirm no unmerged files remained after git rebase --continue completed successfully. Then ran the full test suite (pytest tests/ -v) - initially this failed with an ImportError for WatchlistEntry, which confirmed the missing model. After restoring it, all 5 tests passed.

## PR Description
<!-- Written at the end -->