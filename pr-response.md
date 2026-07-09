# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in
`services/watchlist_service.py` to match the project's verb_to_noun
convention already used by `add_to_collection()`. Updated the one call
site in `routes/watchlist/watchlist.py` (both the import line and the
call inside `add_film()`).

**How I verified:** Ran `grep -rn "save_to_watchlist" --include="*.py" .`
across the whole repo (not just the two files I expected to touch) both
before and after the edit — zero hits remained afterward, confirming no
call site was missed (e.g. in tests, which didn't exist for watchlist
yet at this point).

## Comment 2 — Deduplication
**What I did:** Read `add_to_collection()` in
`services/collection_service.py` as the model pattern: it defines a
dedicated `AlreadyInCollectionError`, does a `filter_by(user_id,
film_id).first()` lookup before inserting, and raises if a match is
found; `CollectionEntry` also has a `UniqueConstraint("user_id",
"film_id")` as a DB-level backstop. I mirrored both layers for the
watchlist: added `AlreadyInWatchlistError` and the same pre-insert
existence check to `add_to_watchlist()` in
`services/watchlist_service.py`, and added a matching
`UniqueConstraint("user_id", "film_id", name="unique_user_film_watchlist")`
to `WatchlistEntry` in `models.py`.

**How I verified:** Verified by code inspection against the collection
service's exact pattern (exception class, check-then-raise order,
constraint naming convention). Have not yet run the test suite myself —
a dedicated dedup test for the watchlist path (duplicate insert raises
`AlreadyInWatchlistError`) is still outstanding and should be added
alongside the Comment 3 test file.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py`. Used
`test_add_to_collection_nonexistent_film_raises` in
`tests/test_collection.py` as the direct model: copied its `app` and
`sample_user` fixtures verbatim (same in-memory SQLite setup, same
`app_context()` teardown), then wrote
`test_add_to_watchlist_nonexistent_film_raises` with the same fake
UUID-format id (`"00000000-0000-0000-0000-000000000000"`) and the same
`pytest.raises(FilmNotFoundError)` assertion structure, calling
`add_to_watchlist()` instead of `add_to_collection()`.

**How I verified:** Confirmed `FilmNotFoundError` is importable from
`services.watchlist_service` (it's brought into that module's namespace
via `from services.collection_service import FilmNotFoundError`), same
as the collection test imports it from its own service module. Have not
run `pytest tests/test_watchlist.py -v` myself yet — asked to have that
run before committing.

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default for `WatchlistEntry`.

**Reasoning:** I'm optimizing for the social-discovery loop that makes a
watchlist feature worth building in the first place. A watchlist that
defaults to private is, for most users, a watchlist no one ever sees —
and a "want to watch" list only creates value for the app (as opposed
to a private notes file) if it's visible enough for friends to browse
it for recommendations. Apps in this space (letterboxd-style logging)
bootstrap their social graph precisely by making activity visible by
default; if visibility requires an opt-in toggle most users never find,
the feature quietly becomes a private list with social plumbing no one
uses — an empty-network problem. I'd also note the *content* being
shared here is lower-stakes than what `CollectionEntry` holds: a
watchlist entry is an aspiration ("I want to see this"), not a rating
or a logged opinion about a film you've actually watched. That makes
public-by-default a reasonable fit for this specific entity, even if
I wouldn't necessarily default an entry with a personal rating attached
to public.

**Tradeoff acknowledged:** The real cost is a consent/surprise problem,
not a data-sensitivity one: a user could add a film to their watchlist
without realizing it's visible to others by default, especially since
there's no onboarding UI yet to disclose this. The `public` field
already gives per-entry control, so the mechanism for privacy exists —
but a control nobody knows about doesn't actually protect anyone. If
this ships without a UI affordance that surfaces the default (e.g., a
visible toggle at add-time, not just an editable flag buried in a
settings screen), I'd consider that a real gap worth flagging as
follow-up work rather than something this default silently resolves.

## Comment 5 — Sort order
**My position:** Agreed with the maintainer — implementing date-added
descending (newest first) as the default, replacing the alphabetical
sort.

**Reasoning:** A watchlist is a queue of intent, not a reference
catalog. Users check it to decide what to watch *next*, and what they
added most recently is the strongest signal of current interest —
alphabetical order requires scanning the whole list regardless of size
to find anything recent. This also removes a needless asymmetry with
`get_collection()`, which already sorts newest-first; a user moving
between "what I've watched" and "what I want to watch" shouldn't have
to reorient to two different mental models for the same kind of list.

**Engagement with reviewer's point:** The maintainer's core claim —
"most users want to see what they added recently" — is the same
reasoning that already justified `get_collection()`'s sort order, so
agreeing here is really just closing a consistency gap I should have
caught myself rather than a close call. The one place I'd push back if
asked: alphabetical sort has real value for a *large* watchlist someone
is trying to browse or search rather than triage (e.g., "did I already
add Paddington 2?"). I don't think that justifies a different default,
but it's a legitimate argument for exposing sort order as a future
query parameter (`?sort=title`) rather than assuming recency is correct
for every use case forever. That's out of scope for this PR.

## Comment 6 — Rebase
**What conflicted:** Ran `git fetch origin && git rebase origin/main`.
Two things came up:
1. An add/add conflict on `.gitignore` — both `main` (via a separate
   PR merged there) and one of my own commits had independently added
   a `.gitignore` file. Not a real disagreement: main's version was a
   strict superset of mine (it additionally ignored `.pytest_cache/`).
2. A conflict in `models.py` on the commit that added the
   `UniqueConstraint` to `WatchlistEntry` (Comment 2's dedup work).
   Git left the file as the "ours" version with no `<<<<<<<` markers
   and just flagged the path unresolved, rather than producing a
   textual conflict — so I diffed the two sides directly
   (`git show :2:models.py` vs `git show :3:models.py`) to see exactly
   what the incoming hunk was meant to add.
3. The conflict that *didn't* show up as a git conflict but was the
   actual point of this exercise: `WatchlistEntry.film_id` was still
   `db.Column(db.Integer, ...)`, left over from before main's
   int→UUID refactor. Because `WatchlistEntry` doesn't exist on
   `main` at all, git had nothing to diff it against, so this
   type mismatch would have silently made it through the rebase
   undetected if I'd only fixed the textual conflicts.

**How I resolved it:** Overwrote `.gitignore` with main's version
(`git show origin/main:.gitignore > .gitignore`). Manually re-added the
`UniqueConstraint("user_id", "film_id", name="unique_user_film_watchlist")`
block to `WatchlistEntry` in `models.py`. Then, separately from the git
conflict, changed `WatchlistEntry.film_id` from `db.Integer` to
`db.String(36)` to match the now-UUID `Film.id`, and swept the
watchlist code for other stale integer references — updated the
`film_id (int)` docstring in `add_to_watchlist()` to `film_id (str):
UUID of the film.`, and the example request body comment in
`routes/watchlist/watchlist.py` from `{ "film_id": <int> }` to
`{ "film_id": "<uuid>" }`. Ran `git add` on the resolved files and
`git rebase --continue`, which replayed the remaining two commits
(test file, sort-order) without further conflicts.

**How I verified no conflict remains:** `git status` reported "working
tree clean" with no rebase in progress. `git log --oneline --graph`
shows a single linear line of 6 commits sitting directly on main's tip
— no merge commit was created by the rebase (the one merge commit
visible, for the `.gitignore` PR, predates and is unrelated to this
branch). Then re-grepped the watchlist code
(`grep -rn "\bint\b|integer|Integer" services/watchlist_service.py
routes/watchlist/watchlist.py tests/test_watchlist.py`) — zero hits —
and confirmed in `models.py` that `WatchlistEntry.film_id` is
`db.String(36)` with its own `UniqueConstraint`, mirroring
`CollectionEntry`.

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->
