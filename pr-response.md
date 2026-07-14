# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` naming convention used by `add_to_collection()`. Updated the single call site in `routes/watchlist/watchlist.py` (`add_film` view). I searched the codebase with `grep -r "save_to_watchlist"` before and after the change — it initially found exactly two matches (the definition and the one call site) and zero matches after the rename, confirming no stragglers were left.

**How I verified:** Ran `grep -r "save_to_watchlist"` across the repo (0 results after rename) and the full test suite:
```
$ python -m pytest tests/ -v
tests/test_collection.py::test_add_to_collection_creates_entry PASSED    [ 25%]
tests/test_collection.py::test_add_to_collection_duplicate_raises PASSED [ 50%]
tests/test_collection.py::test_add_to_collection_nonexistent_film_raises PASSED [ 75%]
tests/test_collection.py::test_get_collection_returns_newest_first PASSED [100%]
============================== 4 passed in 2.02s ==============================
```

## Comment 2 — Deduplication
**What I did:** Added an `AlreadyInWatchlistError` exception (mirroring `AlreadyInCollectionError`) and a pre-check in `add_to_watchlist()`: after confirming the film exists, it queries `WatchlistEntry.query.filter_by(user_id=user_id, film_id=film_id).first()` and raises `AlreadyInWatchlistError` if a row is already found, before constructing a new entry. This is the exact pattern `add_to_collection()` in `services/collection_service.py` uses. I also updated `routes/watchlist/watchlist.py`'s `add_film` view to catch `FilmNotFoundError` (→404) and `AlreadyInWatchlistError` (→409), matching how `routes/collection.py::add_film` handles the analogous cases — previously the watchlist route didn't catch `FilmNotFoundError` at all, so a bad film_id would have produced an unhandled 500 instead of a clean error response.

**How I verified:** Read `services/collection_service.py::add_to_collection` and `routes/collection.py::add_film` first to confirm the exact dedup/error-handling pattern before writing my version. Ran the full test suite (still green, since watchlist-specific tests are added in Comment 3):
```
$ python -m pytest tests/ -v
4 passed in 1.08s
```

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py` and added `test_add_to_watchlist_nonexistent_film_raises`, modeled directly on `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py`: same `app`/`sample_user` fixture structure (in-memory SQLite, per-test app context), same hardcoded nonexistent-id string (`"00000000-0000-0000-0000-000000000000"`), and the same `pytest.raises(FilmNotFoundError)` assertion pattern. I only ported the `sample_user` fixture (no `sample_film`), since the test's whole point is that no matching film exists.

**How I verified:** Ran the new test in isolation, then the full suite:
```
$ python -m pytest tests/test_watchlist.py -v
tests/test_watchlist.py::test_add_to_watchlist_nonexistent_film_raises PASSED [100%]
1 passed in 1.19s

$ python -m pytest tests/ -v
tests/test_collection.py::test_add_to_collection_creates_entry PASSED    [ 20%]
tests/test_collection.py::test_add_to_collection_duplicate_raises PASSED [ 40%]
tests/test_collection.py::test_add_to_collection_nonexistent_film_raises PASSED [ 60%]
tests/test_collection.py::test_get_collection_returns_newest_first PASSED [ 80%]
tests/test_watchlist.py::test_add_to_watchlist_nonexistent_film_raises PASSED [100%]
5 passed in 0.86s
```

## Comment 4 — Default visibility
**My position:** I'm changing the default to `public=False` (private by default).

**Reasoning:** I checked whether CineLog currently has any feature that reads a watchlist entry's `public` flag — it doesn't. There's no follow/friend model, no "browse other users' watchlists" route, nothing in `routes/` or `services/` that ever queries `WatchlistEntry.public`. Today, that field is inert data with no consuming feature. Given that, defaulting it to `True` exposes every new user's want-to-watch list — which can reveal more about a person than a "watched" list does (e.g. wanting to watch something out of step with their public persona, or a spoiler-sensitive queue) — with zero corresponding product benefit, since nothing surfaces it to anyone. A safe default in the absence of an enforcing feature is the private one: it costs nothing today (no feature currently depends on `public=True`), and it means that if/when a social "see what your friends want to watch" feature ships later, users have to affirmatively opt in to being visible, rather than discovering after the fact that a list they never meant to share was public the whole time. That ordering — build the feature, then let users opt in — is safer than the reverse.

**Tradeoff acknowledged:** The obvious cost is that if CineLog's roadmap for watchlists actually is "quiet social discovery" (the way the app's collections feature already frames it as a "community film tracking app"), then a private default adds friction: the eventual feature would need users to go flip a toggle before it'd have any content to show, so we'd launch that hypothetical feature into an empty, opt-in-only surface instead of one that's populated by default. I'm accepting that friction because it only matters once such a feature exists — nothing in this codebase suggests one is imminent — and I'd rather ship a conservative default now than retroactively narrow an already-public default once real user data exists (that's the harder migration).

## Comment 5 — Sort order
**My position:** I agree — switching `get_watchlist()` from alphabetical (`Film.title.asc()`) to date-added, newest first (`WatchlistEntry.date_added.desc()`).

**Reasoning:** Beyond the reviewer's general point about recency, there's a concrete consistency argument specific to this codebase: `get_collection()` in `services/collection_service.py` already sorts its list newest-first by `date_added`. A user who logs a watched film and then checks their watchlist would see one list ordered by recency and the other ordered alphabetically, for no functional reason — that's an inconsistency a user would have to learn and remember, not something that helps them. Matching the existing collection convention removes that inconsistency and gives CineLog one predictable "how are my lists ordered" answer instead of two.

**Engagement with reviewer's point:** The reviewer's framing — "most users want to see what they added recently" — describes the watchlist specifically as a queue: you add a film meaning to get to it, and the ones you added most recently are the most likely to still be top-of-mind (a film you added six months ago and forgot about is lower-value at the top than something you added yesterday and are actively planning to watch). Alphabetical sort actively works against that use case — it puts "Alien" above whatever you added an hour ago, permanently, regardless of when it was added. I don't have a case for alphabetical that beats that, and the collection-consistency point above only reinforces it, so I'm implementing the reviewer's suggestion rather than proposing an alternative.

## Comment 6 — Rebase
**What conflicted:** Ran `git fetch origin && git rebase origin/main`. Two conflicts came up:
1. **`.gitignore`** (add/add): both `main` (via a separate PR, `chore: add .gitignore for generated files`) and my branch added a `.gitignore` independently, with the same 9 base lines plus my own extra entries (`.claude`, `CLAUDE.md`, `p6_docs/`).
2. **`models.py`** (modify/delete): `main`'s `refactor: migrate film IDs from integer to UUID` commit changed `Film.id` and `CollectionEntry.film_id` from `Integer` to `String(36)` — and, because `main`'s history never had the watchlist feature, that same commit's baseline simply didn't contain a `WatchlistEntry` class at all. My later commit changing `WatchlistEntry.public`'s default (`public=True` → `public=False`) touched a line inside that now-missing class, so git flagged it as a conflict rather than silently reintroducing stale integer-based code.

**How I resolved it:** For `.gitignore`, kept the shared base lines plus my extra entries (both sides' content, no information lost). For `models.py`, restored the `WatchlistEntry` class and updated `film_id` from `db.Column(db.Integer, db.ForeignKey("film.id"))` to `db.Column(db.String(36), db.ForeignKey("film.id"))` to match the post-refactor `Film.id` type, keeping `public=False` from my Comment 4 decision. I also fixed a stale docstring in `services/watchlist_service.py::add_to_watchlist` that still described `film_id` as `(int)` — updated to `(str): UUID of the film.` to match the rebased model.

**How I verified no conflict remains:** `git rebase --continue` completed with "Successfully rebased and updated refs/heads/feature/watchlist." `git log --oneline --merges origin/main..HEAD` returns nothing — no merge commits were introduced by the rebase. Ran the full test suite after resolving each conflict and again after the rebase finished:
```
$ python -m pytest tests/ -v
tests/test_collection.py::test_add_to_collection_creates_entry PASSED    [ 20%]
tests/test_collection.py::test_add_to_collection_duplicate_raises PASSED [ 40%]
tests/test_collection.py::test_add_to_collection_nonexistent_film_raises PASSED [ 60%]
tests/test_collection.py::test_get_collection_returns_newest_first PASSED [ 80%]
tests/test_watchlist.py::test_add_to_watchlist_nonexistent_film_raises PASSED [100%]
5 passed in 1.25s
```

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->
