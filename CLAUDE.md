# fantasy-football-hub — notes for every Claude session

One site for Jeff's 5 fantasy football leagues (3 ESPN, 1 Sleeper keeper, 1 Yahoo): understand each league's rules, recommend lineups/waivers/trades, and execute moves Jeff approves. Separate project from `draftpocalypse` (sibling folder); reuse its patterns, never its code paths.

## Start of every session
1. Read `BUILD_PLAN.md` → **Session plan**; take the first row whose status isn't `done`. Do only that row. If it won't fit one usage window, split it and update the table first.
2. Read `docs/LEAGUES.md` (once it exists) for any rule you touch. Never guess an unknown league rule: implement the signature, throw `RuleBlocked('<question>')`, leave a skipped test.
3. Check the row's "Needs from Jeff" column and say what's missing at the start, not the end.

## End of every session
- `npm test` green.
- Set the row's status to `done` (or `partial: <what's left>`), add one line to `docs/CHANGELOG.md`, commit, push to `origin main`.
- If git has no identity, commit with `git -c user.name="Jeff" -c user.email="jeffbiestek@gmail.com"`; don't change global config.

## Conventions
- Rules: plain JS in `rules/`, no I/O. Every test names its rule id (`// covers R-XXX-n`). 100% coverage on `rules/**`.
- Knobs come from Settings (`S.*`), never literals.
- Store UTC, compute in America/New_York, show ET.
- Unit tests never call live APIs; use scrubbed `fixtures/` (no SWID, espn_s2, tokens, or member emails).
- Secrets live only in Cloudflare Worker secrets/KV (local dev: `.env`, gitignored). Never commit credentials.
- Moves on ESPN/Sleeper/Yahoo: only after Jeff approves in the queue; dry-run preview first; human-paced; log to `data/moves.log.jsonl`. Never try to bypass CAPTCHAs or bot checks — hand the move back to Jeff with a deep link.
- Windows machine: PowerShell or Git Bash; Node v24 LTS; Python runs with `-I`.
