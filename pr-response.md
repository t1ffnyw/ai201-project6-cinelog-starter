# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I mainly used AI to double check my work on this project. After each edit, I asked the AI to double check to see if there were any errors caused by my change. I used the AI to understand certain functions like `add_to_collection()` and how to write pytests. I also used it to draft and refine the written responses in pr-response.md, for example brainstorming tradeoffs for default visibility. I also used the AI tool a lot to help me through the git rebase process.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` to match the project's `verb_to_noun` naming convention. I updated the function definition in `services/watchlist_service.py`, the import and call in `routes/watchlist/watchlist.py`. To find every call site, I searched the entire repo for `save_to_watchlist` and the only hits were those three files, and all have been updated to `add_to_watchlist`.

**How I verified:** Ran a project-wide search for `save_to_watchlist` and confirmed zero remaining references. Started the app and confirmed the watchlist add endpoint still works with the renamed function.

## Comment 2 — Deduplication
**What I did:** Read `add_to_collection()` in `services/collection_service.py` to see how collection handles duplicates. It queries for an existing `CollectionEntry` with the same `user_id` and `film_id` before inserting, and raises `AlreadyInCollectionError` if one is found. I applied the same pattern in `add_to_watchlist()`: after confirming the film exists, it queries `WatchlistEntry` with `filter_by(user_id=user_id, film_id=film_id)`, raises `AlreadyInCollectionError` if a match exists, and only then creates the new entry. I reused `AlreadyInCollectionError` from `collection_service.py` rather than defining a new exception class, keeping error handling consistent across features.

**How I verified:** Reviewed the logic side-by-side with `add_to_collection()` to confirm the query-then-raise-then-insert flow matches. Manually tested by POSTing the same `film_id` twice to `/watchlist/<user_id>/add` and confirming the second request is rejected instead of creating a duplicate row.

## Comment 3 — Missing test
**What I did:** Added `tests/test_watchlist.py` with a test for the nonexistent-film case. I used `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py` as my model — it sets up the same in-memory SQLite fixtures (`app`, `sample_user`), passes a fake `film_id` that doesn't exist in the database, and asserts that `pytest.raises(FilmNotFoundError)` is triggered. My test `test_add_to_watchlist_nonexistent_film_raises` follows the same structure, calling `add_to_watchlist()` instead of `add_to_collection()`.

**How I verified:** Ran `pytest tests/test_watchlist.py -v` and confirmed `test_add_to_watchlist_nonexistent_film_raises` passes. Also ran `pytest tests/ -v` to confirm all tests pass.

## Comment 4 — Default visibility
**My position:** Keep public=True
**Reasoning:** CineLog is a community app. Collections are already fully public (no visibility field). Watchlists are forward-looking and socially useful. People can share watchlists and compare, prompting more conversation and connection.
**Tradeoff acknowledged:** Some users will want private watchlists. Public-by-default exposes intent before we ship a way to change it. Private-by-default would be safer for privacy but weaker for discovery and inconsistent with how collections work. I chose sharing-first for v1 and would add a visibility toggle as a follow-up if needed.



## Comment 5 — Sort order
**My position:** Change get_watchlist() to sort by date_added descending (newest first), matching get_collection()
**Reasoning:** A watchlist is a running log of what someone saved to watch later — users usually care most about what they added recently, not alphabetical order. Collection already sorts by date_added descending, and there is even a test for that in test_get_collection_returns_newest_first. Using the same sort for watchlists keeps behavior consistent across user lists and matches how most film apps present “saved for later” content.
**Engagement with reviewer's point:** I agree with your preference. You’re right that most users want to see recent additions first, and we should document the decision rather than leave sort order implicit. I changed get_watchlist() to match get_collection(). 

## Comment 6 — Rebase
**What conflicted:** I rebased `feature/watchlist` onto `origin/main` with `git rebase origin/main`. The only file Git paused on was `.gitignore` — both `main` and my branch had added it. I also expected a `models.py` conflict because `main` migrated film IDs from integer to UUID, but Git never flagged it: none of my watchlist commits modified `models.py`, so Git kept `main`'s version silently. That version had UUID `Film` and `CollectionEntry` but no `WatchlistEntry` at all, even though my service and route code still imported it.

**How I resolved it:** For `.gitignore`, I kept the merged version with all ignore rules from both sides (including `.pytest_cache/`). For the UUID issue, I manually re-added `WatchlistEntry` to `models.py` with `film_id` as `String(36)` to match `CollectionEntry`, and updated watchlist docstrings/comments to use UUID instead of integer. I committed that as `fix: update WatchlistEntry film_id to UUID after main branch refactor`. I aborted an accidental `git pull` merge afterward — `CONTRIBUTING.md` says to rebase, not merge.

**How I verified no conflict remains:** `git status` shows a clean branch with no unmerged paths. Ran `pytest tests/ -v` — all 5 tests pass. Confirmed `WatchlistEntry` exists in `models.py` with UUID `film_id` and that watchlist endpoints accept UUID `film_id` values.

## PR Description

**Feature overview:** This PR adds a watchlist feature to CineLog. Users can save films they want to watch later, view their watchlist, and see metadata like `date_added` and `public` visibility on each entry. New endpoints: `GET /watchlist/<user_id>` and `POST /watchlist/<user_id>/add`.

**Design decisions:**
- **Naming:** Service function follows `verb_to_noun` convention — `add_to_watchlist()`, matching `add_to_collection()`.
- **Deduplication:** Adding a film already on the watchlist raises `AlreadyInCollectionError`, same pattern as collection.
- **Default visibility (`public=True`):** Intentional choice for a community app. Collections are already fully visible; watchlists are shareable discovery signal. Tradeoff: users who want private lists cannot toggle visibility yet — a follow-up if needed.
- **Sort order (`date_added` desc):** Newest additions first, matching `get_collection()`. Users care most about recent saves, not alphabetical order.
- **Post-rebase UUID alignment:** After rebasing onto `main`, `WatchlistEntry.film_id` uses UUID strings to match the film ID migration on `main`.

**Manual testing:**
1. Start the app: `python app.py`
2. Add a film to a user's watchlist:
   `POST /watchlist/<user_id>/add` with body `{ "film_id": "<uuid>" }` → expect `201`
3. View the watchlist:
   `GET /watchlist/<user_id>` → expect the film listed, newest first
4. Add the same film again → expect duplicate rejected (409 if route handles `AlreadyInCollectionError`, or error from service layer)
5. Add a nonexistent `film_id` → expect `FilmNotFoundError` / 404
6. Run automated tests: `pytest tests/ -v`