# PR Response Doc — CineLog Watchlist Feature

> **Note on process:** The upstream `ascherj/ai201-project6-cinelog-starter` repo has issues
> disabled and its two open PRs (from other students' forks) carry no review comments, so there
> was no live `@dev-lead` thread to pull from directly. The six comments addressed below are
> taken from the assignment brief itself, which specifies each one's substance precisely enough
> to act on: rename, deduplication, missing test, default visibility, sort order, and the
> UUID-migration rebase.

## AI Usage
<!-- Fill in at the end -->

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in
`services/watchlist_service.py` to match the project's `verb_to_noun` convention
(`add_to_collection`, `remove_from_collection`, `get_collection`). `save_to_*` doesn't fit that
pattern and reads as a different verb entirely — "save" implies persistence semantics, "add"
implies membership in a collection, which is what's actually happening.

**How I verified:** Grepped the whole repo for `save_to_watchlist` before and after the change —
only two hits existed: the definition in `watchlist_service.py` and the single call site + import
in `routes/watchlist/watchlist.py`. Updated both, re-grepped to confirm zero remaining references,
then ran the full test suite to confirm nothing else depended on the old name.

## Comment 2 — Deduplication
**What I did:** Added an `AlreadyInWatchlistError` exception and a duplicate check in
`add_to_watchlist()`, directly mirroring `add_to_collection()`'s pattern in
`services/collection_service.py`: query for an existing `WatchlistEntry` with the same
`user_id`/`film_id` pair before inserting, and raise if one exists. I also wrapped the route
handler in `routes/watchlist/watchlist.py` with the same `try/except → 404/409` structure that
`routes/collection.py` uses, so the API surface is consistent between the two features (this also
meant handling `FilmNotFoundError` at the route level for the first time — previously an
unhandled 404 case there would have surfaced as a raw 500).

**How I verified:** Added `test_add_to_watchlist_duplicate_raises` (see Comment 3) which asserts
the second call raises `AlreadyInWatchlistError` and that exactly one row exists afterward. Ran
`pytest tests/ -v` — all tests green.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py`, modeled directly on `tests/test_collection.py`:
same `app`/`sample_user`/`sample_film` fixture structure (in-memory SQLite, fresh per test), same
three-test shape required by `CONTRIBUTING.md` (happy path, duplicate/conflict, nonexistent ID).
`test_add_to_watchlist_nonexistent_film_raises` is the direct equivalent of
`test_add_to_collection_nonexistent_film_raises` — same fake-ID string, same assertion that
`FilmNotFoundError` is raised rather than a DB integrity error.

**How I verified:** Ran `pytest tests/test_watchlist.py -v` in isolation (3 passed), then the full
suite `pytest tests/ -v` (7 passed) to confirm no interference with the existing collection tests.

## Comment 4 — Default visibility
**My position:** Keep `public=True` as the default on `WatchlistEntry`, but ship the explicit
`public` parameter on the add endpoint (see Stretch below) so the default is a convenience, not
the only option.

**Reasoning:** The default isn't being decided in a vacuum — it has a precedent already set by
this exact codebase. `CollectionEntry` has no privacy concept at all: every entry returned by
`GET /collection/<user_id>` is visible to anyone who can call that endpoint with a user's ID.
`WatchlistEntry.public` is a *new* concept this feature introduces, and defaulting it to `True`
simply matches the behavior collection already has — it doesn't introduce a new default privacy
posture, it preserves the existing one. Given CineLog's README frames it as a "community film
tracking app," a default that keeps activity visible (what you've watched, what you're planning
to watch) is consistent with what the product already does elsewhere, and it's what makes a
"community" feature legible in the first place — private-by-default lists don't feed a community
feed.

**Tradeoff acknowledged:** A watchlist is not the same thing as a collection, and the reviewer's
underlying concern is legitimate even if the literal default matches precedent: a watchlist
signals *intent* rather than *completed action*. Watchlisting something reveals what you're
curious about before you've committed to it publicly — that's a more sensitive signal than "I
watched and rated this," and some users may not want that visible by default (e.g., a niche or
personal-interest title they haven't decided whether to share yet). I'm not dismissing that risk
— I'm addressing it by making the field explicitly settable at creation time (Comment 4 stretch:
the `public` parameter on `POST /watchlist/<user_id>/add`) rather than only fixable after the
fact. That gives privacy-conscious callers a first-class way to opt out immediately, without
silently changing the default behavior for everyone else based on one contributor's guess at what
users want. If usage data later shows most users flip it to `False` immediately, that's the
signal to revisit the default — not something to decide from first principles mid-PR.

*Stress-tested counterargument:* the strongest pushback here is that "collection has no privacy
concept" isn't really precedent for a *deliberate* public default — collection just never
considered privacy, so citing it as justification is weaker than I initially framed it. It's also
true that shipping an optional `public` param doesn't fully solve the problem if most callers
never pass it — default effects are real, and most entries will land on `True` regardless of
whether the option exists. I still land on `True` as the default because CineLog's stated identity
is a community app where visible activity is the point, but I'm holding that position on the
product-framing argument alone now, not on the (weaker) collection-precedent one.

## Comment 5 — Sort order
**My position:** I disagree with sorting `get_watchlist()` by `date_added` to match
`get_collection()`, and I'm keeping the current alphabetical order (`Film.title.asc()`).

**Reasoning:** Collection and watchlist answer different questions for the user, and I don't
think "for consistency" is a strong enough reason to force them into the same ordering when their
underlying use cases point in different directions. `get_collection()` is a *log* — it's
inherently chronological, and "what did I just watch" is the natural question a newest-first order
answers. A watchlist is a *working list you pick from*, not a timeline: the user's actual question
when opening it is "what are my options," not "what did I add most recently." Sorting a pick-from
list by insertion time is also actively misleading here, because there's no priority field on
`WatchlistEntry` — nothing in the model says films are added in the order the user intends to
watch them. An add-time sort would visually imply a priority queue that doesn't exist, which is a
worse failure mode than "inconsistent with collection."

**Engagement with reviewer's point:** The consistency argument isn't wrong in general — an app
where every list behaves differently for no reason is genuinely harder to use, and if I'm missing
user research showing people expect watchlist and collection to share an ordering convention, I'd
defer to that data over my own reasoning. But absent that evidence, I'd rather have each list's
sort order match its actual use pattern (browsing/picking vs. logging/reviewing) than harmonize
them for the sake of uniformity. If CineLog later adds an explicit "priority" or manual-reorder
feature to the watchlist, that would be the natural point to revisit this — a user-set order would
beat both alphabetical and add-time. That's a future extension, not something this PR needs to
solve.

*Stress-tested counterargument:* a fair challenge is that alphabetical is just as arbitrary as
add-time from the model's perspective — neither actually encodes "what to watch next" — so my
"add-time falsely implies priority" argument cuts both ways, and cross-feature muscle memory
(users learning "newest-first" once and reusing that mental model everywhere) has real value I
was underweighting. I still prefer alphabetical because it's *stable* — a user's position in the
list doesn't shift every time they add something new, which matters for a list you return to and
scan repeatedly — while collection's newest-first order is fine precisely because you don't need a
stable position in a log you're not re-scanning the same way.

## Comment 6 — Rebase
**What conflicted:** Ran `git fetch origin` then `git rebase origin/main`. Git only reported one
textual conflict — an add/add conflict on `.gitignore` (both branches added the file
independently; resolved by taking the union of both lists, since neither strictly subsumed the
other — main's version added `.pytest_cache/` that mine didn't have).

The real conflict wasn't textual, and git didn't flag it: `main`'s
`refactor: migrate film IDs from integer to UUID` commit rewrote `models.py` wholesale — Film.id
and CollectionEntry.film_id became `db.String(36)` UUIDs — but that refactor commit predates the
watchlist feature entirely, so its diff never touched (and had no way to know about) the
`WatchlistEntry` class. Since none of the watchlist branch's own commits ever *added* that class
(it was already present in the branch's initial commit, before any of the six review comments'
work even started), replaying the watchlist commits on top of main's rewritten `models.py` reused
main's version of the file for every non-conflicting hunk — which silently dropped
`WatchlistEntry` altogether. Rebasing succeeded and reported no conflicts, but the moment I ran
the suite afterward, `tests/test_watchlist.py` failed to even import: `ImportError: cannot import
name 'WatchlistEntry' from 'models'`.

**How I resolved it:** Re-added the `WatchlistEntry` class to `models.py`, using
`db.String(36)`/UUID for `film_id` to match the post-refactor `CollectionEntry` pattern instead of
the original `db.Integer`. Updated the stale `film_id (int)` docstrings in
`services/watchlist_service.py` and the `{"film_id": <int>}` request-body examples in
`routes/watchlist/watchlist.py` to reflect UUIDs, and updated both files' module docstrings (which
still said "feature/watchlist branch" / pre-refactor) to match the current, post-rebase state.
Committed as a single `fix:` commit separate from the rebase itself, since it's a distinct logical
change (restoring dropped functionality + updating types), not part of any individual comment's
fix.

**How I verified no conflict remains:** `pytest tests/ -v` — all 12 tests pass post-rebase.
Booted the app via `create_app()` against an in-memory SQLite DB to confirm the models import and
`db.create_all()` succeeds without error. Confirmed `git log --oneline origin/main..HEAD --merges`
returns nothing (no merge commits — a pure linear rebase).

## Stretch — remove_from_watchlist()
**What I did:** Added `NotInWatchlistError` and `remove_from_watchlist(user_id, film_id)` to
`services/watchlist_service.py`, mirroring `remove_from_collection()` exactly: look up the
matching entry, raise if it doesn't exist, otherwise delete and commit. Added the
`DELETE /watchlist/<user_id>/remove` route in `routes/watchlist/watchlist.py`, matching the
existing `DELETE /collection/<user_id>/remove` route's request/response shape (`{"film_id": ...}`
body, 200 on success, 404 with an error message on `NotInWatchlistError`). Added
`test_remove_from_watchlist_removes_entry` and `test_remove_from_watchlist_not_present_raises` to
`tests/test_watchlist.py`.

While writing the first real test that exercised `get_watchlist()` (the sort-order test, added
just after this), I found that `Film` had no relationship pointing back to `WatchlistEntry` —
`entry.film.to_dict()` in `get_watchlist()` raised `AttributeError` because only
`collection_entries` had a backref defined, not `watchlist_entries`. This was a real, shippable bug
in the untested code — nothing had ever called `get_watchlist()` in a test before mine. Fixed by
adding `watchlist_entries = db.relationship("WatchlistEntry", backref="film", lazy=True)` to the
`Film` model, committed separately as its own `fix:`.

## Stretch — additional edge-case test
**What I chose and why:** I wrote `test_get_watchlist_sorts_alphabetically`, which adds two films
in the *reverse* of alphabetical order and asserts `get_watchlist()` still returns them
alphabetically. I picked this case because Comment 5's design decision (keeping alphabetical sort
over the reviewer's suggested date-added order) had no test locking it in — a future contributor
could "fix" the sort back to date-added without any test failing, silently reverting a deliberate
decision. Adding films in reverse-alphabetical insertion order also rules out the test passing by
coincidence (i.e., alphabetical and insertion order happening to match).

## Stretch — visibility toggle
**What I did:** Added an optional `public` parameter to `add_to_watchlist(user_id, film_id,
public=True)`, passed through to the `WatchlistEntry` constructor, and threaded it through the
`POST /watchlist/<user_id>/add` route as an optional `public` key in the request body
(`data.get("public", True)`) — so the endpoint's default matches the service's default exactly.
Added `test_add_to_watchlist_respects_explicit_public_false` and
`test_add_to_watchlist_defaults_to_public_true` to cover both branches. This is what makes the
Comment 4 position actually livable: callers who disagree with the default don't have to accept
it — they can set `public: false` at creation time instead of adding-then-editing.

## Commit history
<!-- git log --oneline output, added after Milestone 4 -->

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->
