# PR Response Doc — CineLog Watchlist Feature

## AI Usage

Most of my AI use was for orientation and verification. At the start, I had it map the responsibilities and dependencies in `models.py`, `services/collection_service.py`, and `tests/test_collection.py`. I read those files myself and checked the summary against them before using it to trace the existing collection patterns.

For Comment 2, I asked AI to walk through the deduplication in `add_to_collection()` so I could follow the same order of checks in the watchlist service. For Comments 4 and 5, I started with my own positions and used AI as a counterargument check. That led me to address the privacy expectation created by a public default and to consider oldest-first as a real alternative to alphabetical sorting. I kept the public default and chose newest-first after weighing those tradeoffs.

The rebase was where the verification mattered most. I had the agent run the tests after Git reported a clean rebase, and those tests exposed a semantic conflict that Git had not marked: the watchlist service still imported `WatchlistEntry`, but the rebased model was missing. I reviewed the proposed UUID-compatible fix before applying it. I also used the agent to run project-wide reference searches and inspect what each commit actually changed before rewriting the history. 

## Comment 1 — Rename
**What I did:** I renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`. A project-wide search showed that the only call site was in `routes/watchlist/watchlist.py`, so I updated its import and function call too. The new name matches `add_to_collection()` and the `verb_to_noun` rule in `CONTRIBUTING.md`.

**How I verified:** Before editing, an exact-name search found three executable references: the definition, the route import, and the route call. After the rename, a source-only search found no uses of the old name and found those same three places under `add_to_watchlist`. At that point, the existing test suite also passed all four tests.

## Comment 2 — Deduplication
**What I did:** I followed the pattern in `add_to_collection()`. After confirming that the film exists, `add_to_watchlist()` now queries `WatchlistEntry` using both `user_id` and `film_id`. If that pair already exists, it raises `AlreadyInWatchlistError` before adding or committing another row.

**How I verified:** Before the fix, I called `add_to_watchlist()` twice with the same user and film in an in-memory database and confirmed that it created two rows. After the fix, the second call raises `AlreadyInWatchlistError`, and a database count confirms that the original row is the only one left. At that point, the existing test suite still passed all four tests.

## Comment 3 — Missing test
**What I did:** I created `tests/test_watchlist.py` and used `test_add_to_collection_nonexistent_film_raises` as the model. The new test creates an isolated app and user, passes a nonexistent UUID to `add_to_watchlist()`, and checks that the service raises `FilmNotFoundError` instead of reaching the database insert.

**How I verified:** I ran `tests/test_watchlist.py` by itself and the new test passed. I then ran the full suite to check that the new fixture and test did not interfere with the collection tests; all five tests passed.

## Comment 4 — Default visibility
**My position:** I would keep `public=True` as the default for new watchlist entries.

**Reasoning:** CineLog describes itself as a community film tracking app, so sharing is part of the product. Public watchlists let users see what their friends plan to watch, find films through people they follow, and trade recommendations. Making entries public by default fits that social purpose without requiring every user to configure sharing before their watchlist is useful to anyone else.

**Tradeoff acknowledged:** Some users will treat their watchlist as a personal queue and may not expect their viewing interests to be visible. That matters more because the current endpoint does not offer a visibility choice. If a visibility control is added later, users should be able to choose when they add a film, and the default should be clearly shown rather than buried in a setting. A client could also confirm that the new entry is public after it is created. I still think public is the better default for CineLog, but it should not be an invisible one.

## Comment 5 — Sort order
**My position:** I agreed with the reviewer and changed `get_watchlist()` to sort by `date_added` descending, with the most recently added films first.

**Reasoning:** Recent additions are more likely to reflect what the user is interested in watching now, and putting them first makes those films easy to find. Alphabetical order is predictable, but the title of a film says nothing about when or why the user added it.

**Engagement with reviewer's point:** I also considered sorting oldest-first. That would be better than alphabetical order because it would treat the watchlist like a queue and surface films the user has postponed the longest. I decided against it because old entries can become stale while newly added films get buried at the bottom. The reviewer's newest-first suggestion is the better default for how users are likely to return to an active watchlist.

## Comment 6 — Rebase
**What conflicted:** The rebase replayed all six feature commits without textual conflict markers, but the branch still had a semantic conflict with `main`. `main` migrated `Film.id` and `CollectionEntry.film_id` to UUID strings and removed `WatchlistEntry`, while the rebased service still imported that model. The first post-rebase test run failed during collection with `ImportError: cannot import name 'WatchlistEntry' from 'models'`.

**How I resolved it:** I restored `WatchlistEntry` in the rebased `models.py`, changed its `film_id` foreign key to `db.String(36)` to match `Film.id`, and restored the `Film.watchlist_entries` relationship used by `get_watchlist()` through `entry.film`. I also replaced the remaining integer-ID wording in the service and route documentation with UUID examples.

**How I verified no conflict remains:** I confirmed that `origin/main` is an ancestor of the branch and that there are no merge commits between `origin/main` and `HEAD`. The full test suite passes all five tests. I also created two UUID-backed watchlist entries with controlled timestamps and called `get_watchlist()`; it returned both films newest-first, and every returned film ID was a 36-character UUID.

## PR Description

### Summary

This PR adds watchlists to CineLog. A user can add a film through `POST /watchlist/<user_id>/add` and retrieve the watchlist through `GET /watchlist/<user_id>`. The service checks that the film exists, prevents the same user from adding the same film twice, and returns watchlist metadata with each film.

The watchlist model now uses the same UUID film IDs as `main`. Results are ordered by `date_added` descending, so the newest additions appear first.

### Design decisions

New watchlist entries remain public by default. CineLog is a community film app, so this makes watchlists useful for sharing interests and recommendations without requiring extra setup. The tradeoff is that some users may expect a watchlist to be private. The current endpoint does not expose a visibility choice, so clients should make the public default clear rather than leaving it implicit.

I changed the original alphabetical ordering to newest-first. Recent additions are more likely to represent what a user wants to watch now. I also considered oldest-first because it would make the watchlist behave more like a queue, but older entries can become stale while recent interests get buried at the bottom.

### Testing

The full test suite passes all five tests:

```bash
pytest tests/ -v
```

I also manually exercised the HTTP routes with two films. Both add requests returned `201`, the list request returned `200`, both entries had `public: true`, and the film added second appeared first.

### Manual test steps

1. Install the dependencies and create test records:

   ```bash
   pip install -r requirements.txt
   python -m flask --app app:create_app shell
   ```

2. In the Flask shell, create one user and two films, then copy the three printed UUIDs:

   ```python
   from uuid import uuid4
   from app import db
   from models import Film, User

   suffix = uuid4().hex[:8]
   user = User(
       username=f"pr-test-{suffix}",
       email=f"pr-test-{suffix}@example.com",
   )
   older_film = Film(title="Older Choice")
   newer_film = Film(title="Newer Choice")
   db.session.add_all([user, older_film, newer_film])
   db.session.commit()
   print(user.id, older_film.id, newer_film.id, sep="\n")
   exit()
   ```

3. Start the API in one terminal:

   ```bash
   python app.py
   ```

4. In another terminal, replace the values below with the three UUIDs printed by the shell:

   ```bash
   USER_ID="<user UUID>"
   OLDER_FILM_ID="<Older Choice UUID>"
   NEWER_FILM_ID="<Newer Choice UUID>"

   curl -X POST "http://127.0.0.1:5000/watchlist/$USER_ID/add" \
     -H "Content-Type: application/json" \
     -d "{\"film_id\":\"$OLDER_FILM_ID\"}"

   curl -X POST "http://127.0.0.1:5000/watchlist/$USER_ID/add" \
     -H "Content-Type: application/json" \
     -d "{\"film_id\":\"$NEWER_FILM_ID\"}"

   curl "http://127.0.0.1:5000/watchlist/$USER_ID"
   ```

5. Confirm that both add responses contain `"public": true` and that the final list shows `Newer Choice` before `Older Choice`.
