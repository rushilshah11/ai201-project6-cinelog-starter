# PR Response Doc — CineLog Watchlist Feature

## AI Usage
I used Claude (an AI coding assistant) throughout this project, in a few
distinct ways:

- **Codebase orientation.** Before touching any review comment, I had it
  read `models.py`, `services/collection_service.py`, and
  `tests/test_collection.py` in full and summarize the naming
  convention (verb_to_noun), the two-layer dedup pattern
  (app-level check + DB `UniqueConstraint`), and the test fixture
  structure. I verified this against the actual code rather than taking
  the summary at face value — e.g. I confirmed `AlreadyInCollectionError`
  really is checked via a `filter_by().first()` call before insert, not
  just described that way.
- **Pattern replication, not generation.** For Comment 2 (dedup) and
  Comment 3 (test), I asked it to explain what
  `add_to_collection()`/`test_add_to_collection_nonexistent_film_raises`
  do step by step, then wrote the watchlist equivalents myself against
  that understanding, rather than asking it to generate the dedup logic
  or the test directly.
- **Self-critiquing the design arguments (Comments 4 & 5).** While
  drafting the visibility and sort-order positions, I had it push back
  on my own draft before I finalized it. For Comment 4, the pushback was
  that a "public by default, per-entry toggle" design is a real
  consent/disclosure problem if there's no UI surfacing the default —
  I hadn't originally framed the tradeoff that sharply, and I folded
  that directly into the "Tradeoff acknowledged" section rather than
  leaving my position as a one-sided justification. For Comment 5, the
  pushback was that recency-first ordering doesn't serve someone
  browsing/searching a *large* watchlist by title — I agreed it's a
  legitimate point but not one that changes the default, so I captured
  it in "Engagement with reviewer's point" as a scoped-out future
  enhancement (`?sort=title`) instead of ignoring it.
- **Git conflict/history archaeology (Comment 6 + commit cleanup).**
  During the `main` rebase and the later interactive rebase to clean up
  commit messages, I used it to diff conflicting sides of `models.py`
  (`git show :2:` vs `:3:`) and to trace exactly which commit in history
  first introduced the `WatchlistEntry` model (`git log --all -- models.py`),
  since that turned out to matter for splitting a bundled commit
  correctly rather than guessing at the split point.

Where this stopped: I didn't ask it to write the dedup check, the test,
or the Comment 4/5 arguments outright — those are my own reasoning and
code, with AI used to check my understanding of existing patterns and
to stress-test my conclusions before committing to them.

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

### What this adds
A watchlist feature: users can save films they want to watch later,
separate from their collection of films they've already watched.

- `WatchlistEntry` model (`models.py`) — one row per user/film pair,
  with `date_added` and a `public` visibility flag.
- `POST /watchlist/<user_id>/add` — add a film to a user's watchlist.
  Body: `{ "film_id": "<uuid>" }`. Returns 404 if the film doesn't
  exist, and prevents duplicate entries for the same user/film pair.
- `GET /watchlist/<user_id>` — return a user's watchlist, sorted by
  date added (newest first).

### Design decisions
- **Visibility default (`public=True`):** kept `public` defaulting to
  `True`. A watchlist that's private by default is a watchlist no one
  else can discover, which undercuts the reason to build a social
  "want to watch" feature in the first place — and unlike
  `CollectionEntry`, a watchlist entry is just an aspiration, not a
  logged rating, so it's lower-stakes to share. The real risk is that
  users won't realize their entries are visible by default without a
  UI affordance that makes that clear at add-time — that's flagged as
  follow-up work, not something this PR's schema alone resolves. Full
  reasoning in Comment 4 above.
- **Sort order (date added, newest first):** `get_watchlist()` sorts by
  `date_added` descending instead of alphabetically by title. A
  watchlist is a queue of intent — users check it to decide what to
  watch next, and recency is a stronger signal than title for that.
  This also matches `get_collection()`'s existing "newest first"
  convention instead of introducing a second, inconsistent sort order
  in the same API. Full reasoning in Comment 5 above.

### Manual testing steps
1. Start the app (`flask run`, or however the project is normally run
   locally) with a fresh/empty database.
2. Create a user and a couple of films (via the existing
   `/collection` or direct DB seeding, since there's no dedicated user/
   film-creation endpoint in this PR).
3. Add a film to the watchlist:
   `POST /watchlist/<user_id>/add` with body `{ "film_id": "<uuid>" }`.
   Expect `201` and the created entry back, with `"public": true`.
4. Try adding the same film again for the same user. Expect a
   duplicate-prevention error rather than a second row being created
   (verify via `GET /watchlist/<user_id>` still shows one entry).
5. Add a second, different film to the same user's watchlist.
   `GET /watchlist/<user_id>` should return both films with the most
   recently added one first.
6. Try adding a film with a nonexistent `film_id` (e.g. a random UUID).
   Expect a "film not found" error, not a 500 or a raw DB error.
7. Run the automated suite to confirm both the new watchlist tests and
   the existing collection tests still pass:
   `pytest tests/ -v`

## Screenshot

![Screenshot](Screenshot%202026-07-09%20at%207.24.14%20PM.png)

If you can't see screenshot: 

e91e944 (HEAD -> feature/watchlist, origin/feature/watchlist) docs: add pr-response.md with review responses and design decisions
a1d0650 fix: default watchlist sort order to date added (newest first)
1153c09 test: add test for nonexistent film_id in add_to_watchlist
37ccd59 fix: update WatchlistEntry film_id to UUID after main branch refactor
7abed2c fix: add deduplication check to prevent duplicate watchlist entries
5349881 fix: rename save_to_watchlist to add_to_watchlist per naming convention
6d45c75 fix: update film retrieval method to use db.session.get in collection and watchlist services
c645987 feat: add watchlist model and add_to_watchlist endpoint
bbe206c (origin/main, origin/HEAD, main) Merge pull request #2 from ascherj/chore/add-gitignore
718a9a8 chore: add .gitignore for generated files
07ca580 refactor: migrate film IDs from integer to UUID
014ae54 feat: initial CineLog API with film collection feature