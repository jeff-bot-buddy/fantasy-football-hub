# fantasy-football-hub — build plan

## Context
Jeff has 5 fantasy football leagues spread over three platforms: 3 on ESPN, 1 on Sleeper and 1 on Yahoo. He wants one site that:
- understands each league's rules and scoring,
- recommends waivers and trades for each league, and
- makes the moves he approves on the real sites.

This is a separate project from Draftpocalypse. It gets its own repo, `jeff-bot-buddy/fantasy-football-hub`, cloned next to draftpocalypse rather than inside it. It reuses Draftpocalypse's stack and session workflow.

**Decisions so far:**
- **Repo name:** `fantasy-football-hub`.
- **Approval:** every move needs Jeff's approval, with a dry-run preview first.
- **Order:** waivers and lineups come first, then trades, and moves on the real sites come last.

**Constraints:**
- **Timing:** today is 2026-11-04, about week 9 of the NFL season. Most trade deadlines fall in mid-to-late November, so the fast path has to deliver value inside about 6 sessions.
- **No Chrome here.** This cloud session has no Chrome tools, so the open league tabs can't be reached from it. Anything that needs Chrome is done either by Jeff (copying cookies, about 5 minutes) or in a session run from the Claude desktop app with Claude in Chrome.
- **No repo creation here.** This session can't create GitHub repos, so Jeff creates the empty private repo and I attach it with `add_repo`.

## Architecture (reused from Draftpocalypse)
| Layer | Choice | Reused from |
|---|---|---|
| Rules and recommendation engine | Plain JS in `rules/` with no I/O, `node --test`, 100% coverage, `// covers R-…` tags, settings `S.*` | `draftpocalypse/rules/settings.js`, `rules/errors.js`, `tests/` |
| Ingest | One pure parser per platform in `ingest/{espn,sleeper,yahoo}.js` plus a thin fetch wrapper; schema checks with `need()`; skip a run when the hash is unchanged | `draftpocalypse/apps-script/ingest/common.js`, the pattern in `jobs.js` |
| Scheduler | Cloudflare Worker cron runs the reads and builds `site.json`, so credentials stay in the Worker only | New. Apps Script is skipped because Google IPs may be blocked by ESPN fantasy hosts. **Assumption**, to be checked in S2. |
| Site | Vanilla JS on Cloudflare Pages, mobile-first, with a last-good JSON fallback | `draftpocalypse/site/`, `tools/deploy_site.js` |
| Approve and execute | Cloudflare Worker with a KV approval queue. It holds platform credentials as Worker secrets and sends approved moves. | Pattern from `draftpocalypse/workers/heartbeat/`, HMAC sessions from `apps-script/auth.js` |
| Weekly skill | `.claude/skills/ff-weekly/`: a script does the math, then the model makes the calls | `draftpocalypse/.claude/skills/roster-scout/` |

**Common data model** (`docs/SCHEMA.md`): `League{platform, id, scoring{stat→pts}, roster_slots, waiver{type: FAAB|rolling|reverse, budget, process_time}, trade_deadline, playoff_weeks}`, `Team`, `Player{hub_id, espn_id, sleeper_id, yahoo_id}`, `Matchup`, `Transaction`, `Move{kind, league, payload, status}`. Player IDs are linked across platforms with Sleeper's `/players/nfl` dump, which includes `espn_id` and `yahoo_id`. **Assumption:** Sleeper's ID coverage is good enough for this. It will be checked against all 5 rosters.

**Platform reality** (from my own knowledge, to be re-verified in each platform's first session):
| Platform | Read | Write (moves) | Auth |
|---|---|---|---|
| ESPN ×3 | Unofficial v3 JSON (`lm-api-reads.fantasy.espn.com …/ffl/seasons/2026/segments/0/leagues/{id}?view=mSettings,mRoster,mMatchup,kona_player_info`) | Unofficial `lm-api-writes…/transactions`, which is what the ESPN website itself calls | `espn_s2` and `SWID` cookies |
| Sleeper | Official read-only API `api.sleeper.app/v1`, no auth needed | No public write API. Either the app's private GraphQL with a token (spike first) or Playwright in Jeff's Chrome as a fallback. Highest risk of the three. | Session token |
| Yahoo | Official Fantasy Sports API | **Officially supported**: add/drop, waiver claims, trade proposals, lineup | OAuth2 developer app (Jeff registers it, about 5 minutes) |

**Risk flag:** automating writes on ESPN and Sleeper uses undocumented endpoints. That is a terms-of-service grey area, and the endpoints can break without notice. Mitigations:
- Only moves Jeff has approved are sent.
- A human-paced rate is used.
- No bulk actions.
- Every move is logged in `data/moves.log.jsonl`.

The project will not try to get around CAPTCHAs or bot detection. If a site challenges a request, the move is handed back to Jeff with a one-tap deep link.

## Projections and recommendations (Opus rows)
- **Projections:** start from platform projections (ESPN `kona_player_info` and Sleeper's projections endpoint). Then blend in **betting-market implied stats**: player props such as yards and anytime TD, turned into expected fantasy points under each league's own scoring. Blend weights are `S.*` settings.
  - **Assumption:** the props come from The Odds API, which needs a paid key from Jeff. The free tier is probably too small for full NFL props, but that isn't verified.
- **Waivers:** for each league, rank free agents by points over the worst starter Jeff would drop at that position, adjusted for bye weeks, schedule and playoff-week schedule. Suggest a FAAB bid from league budget context.
- **Lineups:** the best legal lineup for each league's roster slots, with an OUT/bye check before kickoff.
- **Trades:** rest-of-season value plus Jeff's positional needs, using the opponent's needs to find mutually good offers. Reuse the `fantasy-trade-evaluator` skill's market-value and adversary steps.
- **Adversary step:** every weekly run includes a check that tries to break the top calls, for example a stale injury tag, a backup getting the snaps, or Thursday-night lock timing.

## Adversary changes already applied
The review's platform facts come from model knowledge and are not verified. Each one gets checked in the first session for its platform.
- **Trades moved earlier:** the trade evaluator now comes right after waivers and lineups. Trade-proposal writes are left out of 2026; Jeff sends offers by hand from a prepared offer. The real deadlines (including ESPN's trade review period) are read in S2 and S3.
- **Keeper league:** the Sleeper league is a keeper league, so `League.keeper{count, cost_rule, deadline}` is added along with a keeper value term in trade value.
- **Rules checklist added to S4:**
  - Superflex/2QB, IDP, TE premium, fractional PPR and bonuses.
  - IR slots, position maximums and pickup limits.
  - Per-game lineup locks: the optimizer leaves locked players fixed.
  - Waiver timing: process day and time, Yahoo's roughly 2-day waiver period, FAAB tie-breaks.
  - Fantasy playoffs ending week 17 vs. 18.
- **Rows split:** the old S4, S5 and S9 were each too big for one session and are now two rows each.
- **Credentials in the Worker only:** the Worker also runs the reads and keeps the Yahoo refresh token in KV. Fixtures are scrubbed of SWID and member names. Cloudflare Access protects the Approve button.
- **Player ID crosswalk:** a manual map for defenses (team codes) and a name+team+position fallback. Sleeper's `/players/nfl` is cached and pulled at most once a day.
- **ESPN free agents:** `kona_player_info` needs the `x-fantasy-filter` header. Before any write is built, the real write requests are recorded from the ESPN site.

## Session plan (one row per session; mechanical rows run on Sonnet to save limits)
| # | Session | Model | Delivers (done when) | Needs from Jeff | Status |
|---|---|---|---|---|---|
| S0 | Jeff setup (about 15 min, no Claude) | — | Empty private repo `jeff-bot-buddy/fantasy-football-hub`; a list of the 5 league URLs | Create the repo. For each league, paste its URL. Copy `espn_s2` and `SWID` from Chrome DevTools → Application → Cookies (espn.com). Note your Sleeper username. | in progress |
| S1 | Scaffold + Sleeper read | Sonnet | Repo with `CLAUDE.md`, `BUILD_PLAN.md`, `docs/SCHEMA.md`, `npm test`, settings plus errors copied from Draftpocalypse; Sleeper adapter (league, rosters, matchups, players dump) with fixtures and tests | Sleeper username | todo |
| S2 | ESPN read ×3 | Sonnet | ESPN adapter covering settings, rosters, matchups, free agents (`x-fantasy-filter`) and projections for all 3 leagues; scrubbed fixtures; real trade deadlines; a smoke test from a datacenter IP | `espn_s2` and `SWID` stored as secrets (I give you the exact steps) | todo |
| S3 | Yahoo read | Sonnet | OAuth2 refresh-token flow and a Yahoo adapter with fixtures | Register the Yahoo dev app and do a one-time Allow click | todo |
| S4a | Scoring engine | Opus | `rules/scoring.js` maps ESPN, Yahoo and Sleeper stat vocabularies to one scheme and scores any stat line under any of the 5 leagues; the S4 rules checklist is part of the schema; a past week's points match the platform's to 0.01 | — | todo |
| S4b | League docs + ID crosswalk | Sonnet | Generated `docs/LEAGUES.md` (scoring, slots, keeper, waiver type and time, trade deadline, playoffs); player ID crosswalk with a manual defense map | Confirm `LEAGUES.md` is right (about 5 min) | todo |
| S5a | Projections + lineup optimizer | Opus | League-scored projections; best legal lineup for each league that works around locked players; OUT/bye alerts | — | todo |
| S5b | Waiver ranker + `ff-weekly` skill v1 | Opus | Waiver list and FAAB bid per league, with adversary checks; the weekly run works from chat | — | todo |
| S6 | Trade evaluator | Opus | Targets and fair offers per league (keeper-aware for Sleeper), deadline countdown, evaluates a pasted trade; offers are sent by hand | — | todo |
| S7 | Site v1 (read-only) + Worker refresh | Sonnet | Cloudflare Pages dashboard: 5 league cards, lineup alerts, waivers and trade ideas per league; a Worker cron runs the reads | Cloudflare project (same account as Draftpocalypse) | todo |
| S8 | Market blend | Opus | Odds API props turned into league-scored projections, with a backtest on weeks 1–9 that is reported honestly | Odds API key (optional) | todo |
| S9a | Approval queue | Sonnet | KV queue, **Approve / Reject**, dry-run preview, audit log, Cloudflare Access | Set up Cloudflare Access (about 5 min) | todo |
| S9b | Yahoo writes | Sonnet | Yahoo lineup, add/drop and waiver claim through the queue (XML bodies, `fspt-w` permission), tested first on an empty slot | Approve the first live test move | todo |
| S10 | ESPN writes | Opus | ESPN lineup and add/drop only, using request formats recorded from the real site; any challenge is handed back to Jeff | Watch the first test move | todo |
| S11 | Sleeper writes spike | Opus | Proves either private GraphQL or local Playwright for lineup and add/drop; falls back to a deep link if both fail | Run this session from the desktop app with Claude in Chrome | todo |
| S12 | Hardening | Sonnet | Cookie and token expiry alerts, retries, skill v2, playoff-week mode | — | todo |

**Fast path:** after S6 (8 sessions), lineup, waiver and trade calls for all 5 leagues already work from chat, before the site or any moves on the real sites exist.

## Workflow conventions (copied into the new `CLAUDE.md`)
- **Start of session:** take the first row whose status isn't `done`, do only that row, and raise "Needs from Jeff" up front.
- **End of session:** `npm test` green, row status updated, one line in `docs/CHANGELOG.md`, commit, then push to `main`.
- **Time:** store UTC, compute in America/New_York, show ET. Settings are never written as literal values.
- **Tests:** unit tests use `fixtures/` only, never live APIs.
- **Secrets:** credentials are never committed. They live only in Worker secrets and KV (plus a local `.env` for dev, which `.gitignore` covers). Fixtures are scrubbed before they are committed.

## Verification
- **Each read row:** fixture tests pass, plus one live smoke run (`node tools/smoke.js --league <id>`) whose team names and rosters match what Jeff sees on the site.
- **S4a:** the points calculated for Jeff's starters in one past week match each platform's official box score to within 0.01, in all 5 leagues.
- **S5a and S8:** backtest on weeks 1–9 using mean absolute error against actual points; report whether the market blend beats platform projections, with the honest result whichever way it goes.
- **Write rows:** dry-run payload shown, then one approved low-stakes live move, then confirmation by reading the real site back.
