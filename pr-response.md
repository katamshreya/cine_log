# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:**
Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`
to match the project's `verb_to_noun` naming convention already used in
`collection_service.py` (e.g., `add_to_collection`). Updated the docstring's first
line to match, and updated the only call site in `routes/watchlist/watchlist.py`
(both the import statement and the function call inside `add_film`).

**How I verified:**
Ran `git grep "save_to_watchlist"` across the repo to confirm no remaining
references to the old function name — result was empty. Ran the full test
suite (`pytest tests/ -v`) and confirmed all 4 existing tests still pass.
Committed as `fix: rename save_to_watchlist to add_to_watchlist per naming convention`.

## Comment 2 — Deduplication
**What I did:**
Added a `AlreadyInWatchlistError` exception class to `services/watchlist_service.py`,
defined locally in the file following the same pattern as `collection_service.py`'s
own exceptions (`FilmNotFoundError`, `AlreadyInCollectionError`, etc.). In
`add_to_watchlist()`, added a check after the film-existence check and before
creating the entry: query for an existing `WatchlistEntry` matching the same
`user_id` and `film_id`; if found, raise `AlreadyInWatchlistError` instead of
inserting a duplicate row.

**How I verified:**
Compared the implementation directly against `add_to_collection()`'s dedup
logic in `collection_service.py` to confirm the same structure (existence
check → dedup check → create). Wrote a new test,
`test_add_to_watchlist_duplicate_raises`, mirroring
`test_add_to_collection_duplicate_raises`, which adds a film to the watchlist,
attempts to add it again, asserts `AlreadyInWatchlistError` is raised, and
confirms only one entry exists in the database afterward. Ran the full test
suite (`pytest tests/ -v`) — all 6 tests pass.

## Comment 3 — Missing test
**What I did:**
Created `tests/test_watchlist.py`, following the same fixture and structure
pattern as `tests/test_collection.py`. Wrote
`test_add_to_watchlist_nonexistent_film_raises`, directly modeled on
`test_add_to_collection_nonexistent_film_raises`, which asserts that calling
`add_to_watchlist()` with a film_id that doesn't exist raises
`FilmNotFoundError`. Used an integer fake ID (`999999`) rather than a UUID
string, since `Film.id` is still `db.Integer` on this branch prior to the
Comment 6 rebase — will need to revisit this once film IDs move to UUID.

**How I verified:**
Ran `pytest tests/test_watchlist.py -v` to confirm the test passes on its own,
then ran the full suite (`pytest tests/ -v`) to confirm no other tests broke.

## Comment 4 — Default visibility
**My position:**
I chose to keep `public=True` as the default for new watchlist entries.

**Reasoning:**
Unlike a collection entry, which represents a film a user has already watched and
functions more like a personal record, a watchlist entry is inherently
forward-looking and social. I think this default best matches the primary
purpose of a watchlist as a discovery and sharing feature. Many users build
watchlists to track films they plan to watch and may want to share those lists
with friends or other users without having to manually change the visibility of
every new entry. Using `public=True` minimizes friction for that common workflow
by making the default behavior align with sharing.

**Tradeoff acknowledged:**
The main downside is that users who expect watchlists to be private by default
could unintentionally expose entries they intended to keep personal. Setting the
default to `False` would better prioritize privacy and require users to
explicitly opt into sharing, but it would also add an extra step for users whose
primary goal is maintaining a public watchlist. I think the better long-term
solution would be to let users choose a default visibility preference in their
account settings, but given the current design, I believe `public=True` provides
the smoother default experience.

## Comment 5 — Sort order
**My position:**
I would change the watchlist to sort by date added rather than alphabetically.

**Reasoning:**
A watchlist is typically an evolving list of films a user intends to watch, so I
think preserving the order in which items were added better reflects how users
interact with the feature. Recently added films are often the ones users are
most interested in revisiting, so showing them first makes it easier to continue
using the watchlist without searching. Alphabetical ordering is predictable, but
it is less useful when users are actively adding new titles over time.

**Engagement with reviewer's point:**
I agree with the maintainer's reasoning that insertion order provides more
meaningful context than alphabetical order for this type of feature. While
alphabetical sorting can help locate a specific title, users can already search
for films before adding them, and a future search or filter feature could
address that use case more effectively. For the default view, I think sorting
by date added better reflects user intent and provides a more natural
experience.

## Comment 6 — Rebase
**What conflicted:**
While my `feature/watchlist` branch was open, main merged a refactor migrating
`Film.id` from an integer primary key to a UUID string (`db.String(36)`).
I ran `git fetch origin` and `git rebase origin/main`, which completed without
git reporting a textual conflict. However, this was misleading: after the
rebase, `WatchlistEntry` was missing entirely from `models.py`, and my test
suite failed on import (`ImportError: cannot import name 'WatchlistEntry'
from 'models'`). The refactor and my original watchlist model changes touched
non-overlapping lines in the file, so git didn't flag a conflict even though
the result was semantically broken.

**How I resolved it:**
I re-added the `WatchlistEntry` model to `models.py`, this time defining
`film_id` as `db.String(36)` (matching the new UUID-based `Film.id` and
`CollectionEntry.film_id`) instead of the old `db.Integer`. I also added the
missing `watchlist_entries` relationship on both `User` and `Film` to mirror
the existing `collection_entries` relationships. Separately, I updated
`tests/test_watchlist.py`, which had used an integer placeholder
(`fake_film_id = 999999`) for its nonexistent-film test — I changed this to a
fake UUID string (`"00000000-0000-0000-0000-000000000000"`) to match the new
column type and stay consistent with `test_collection.py`'s equivalent test.

**How I verified no conflict remains:**
Ran the full test suite (`pytest tests/ -v`) after each change — all 6 tests
pass, including both watchlist tests exercising `add_to_watchlist()` against
the UUID-based schema. Confirmed with `git log --oneline` that the branch
history remains linear relative to origin/main, with no merge commits created
during the rebase. Started the app locally (`python app.py`) to confirm it
boots without errors after the model changes.

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->