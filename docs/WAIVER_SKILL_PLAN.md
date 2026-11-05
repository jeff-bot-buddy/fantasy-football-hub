# `ff-waivers` skill — plan v1 (pre-adversary)

**Goal:** every week, for each of Jeff's 5 leagues separately, name the adds (with drop and bid) that most raise **Jeff's odds of winning that league's title**, backed by league math plus a fleet of research agents reading 10+ sources.

**North-star metric:** ΔP(title) per move, from a Monte Carlo of the rest of the season and playoffs under each league's own rules. Everything else (recency weights, positional value, FAAB) feeds that one number, so moves in different leagues can be compared.

**Where it fits:** this is BUILD_PLAN row **S5b** (waiver ranker + skill), pulled forward as a v0 that runs on whatever adapters exist. It does not replace S1–S4b; it consumes them.

## 0. Facts checked today (2026-11-05, from this cloud container)
| Check | Result |
|---|---|
| Sleeper `state/nfl` | 200 — season 2026, **week 9**, regular season |
| Sleeper trending adds API | 200, no auth |
| ESPN fantasy read host `lm-api-reads.fantasy.espn.com` | 200 (private leagues still need `espn_s2` + `SWID`) |
| ESPN public scoreboard `site.api.espn.com` | 200, includes DraftKings spread + total per game |
| FantasyPros, Fantasy Footballers, RotoWire, NBC Sports (Rotoworld), CBS, Yahoo Sports, Footballguys, Fantasy Life | 200 (home/listing pages; article bodies and paywalls not yet tested) |
| Yahoo Fantasy API | 401 — needs OAuth (S3) |
| Reddit | 403 "Blocked" (site blocks datacenter IPs) |
| Pro-Football-Reference | 403 Cloudflare challenge — **excluded; we do not solve bot checks** |
| GitHub nflverse releases | 403 from session scope — fixable with `add_repo nflverse/nflverse-data` (read) |

## 1. Inputs per league (data contract)
| Input | Why | Source (v0 → v1) |
|---|---|---|
| Scoring settings (PPR/half/std, 4 vs 6 pt pass TD, TE premium, bonuses, superflex, IDP, K/DST rules) | re-score every projection per league | ESPN `mSettings`, Sleeper `scoring_settings`, Yahoo settings (paste until S3) |
| Roster slots, bench, IR slots, position maxes | legal lineups, drop logic | same |
| Waiver type (FAAB / rolling / reverse standings), budget, process day+time, waiver period after drops, FAAB tie-break, acquisition limits | bid/priority logic, deadline countdown | same |
| All rosters (not just Jeff's) | true availability, rivals' needs, FAAB competition | same |
| Free agents + rostered % | candidate pool | ESPN `kona_player_info` (+`x-fantasy-filter`), Sleeper = all players − rostered, Yahoo paste |
| Standings, PF/PA, remaining schedule, divisions | playoff/title sim | league endpoints |
| Playoff format: teams in, byes, weeks (15–17 vs 16–18), seeding **tiebreakers** | sim rules | settings; unknown → `RuleBlocked` |
| Keeper rules (Sleeper) | keeper value in drop/add | `docs/LEAGUES.md` |
| Transaction history (FAAB winning bids, who adds what) | league bid model | transactions endpoints |
| Injury status, byes | lineup holes | platform player data + news agent |

**Rule:** if a rule the math needs is unknown, the league card shows `RuleBlocked('<question>')` for that piece instead of a guess (CLAUDE.md).

## 2. Cadence (ET)
| Run | When | Purpose |
|---|---|---|
| **Main** | Tuesday 9:00 AM ET (after MNF, before earliest waiver process) | full run: all leagues, fleet, bids |
| **Pre-process check** | 2 h before each league's waiver process time | injury/news delta only; revise bids |
| **FA pass** | Thursday 10 AM + Saturday 6 PM ET | first-come free agents after waivers clear; streamers; injury pivots |
| Countdown | always shown | per-league waiver process time and trade deadline |

## 3. Pipeline
```
P0 Preflight (script) → P1 Ingest (script) → P2 League math (script, rules/) ─┐
                                   P3 Research fleet (6 agents, parallel) ────┤
                                                                               ├→ P4 Merge + rank (script)
                                                                               ├→ P5 Adversary + validator (2 agents)
                                                                               └→ P6 Brief (chat + HTML) + archive JSON
```
- **P0:** resolve NFL week from Sleeper `state/nfl`; load league configs; compute each league's phase (contending / bubble / locked / eliminated) and waiver deadline.
- **P1:** pull the §1 inputs; hash-skip if unchanged; label every input with its age. Fallback when an adapter is missing or a cookie expired: ask Jeff for a roster + waiver screenshot for that league only, and mark the card "manual data".
- **P2:** §4 math, deterministic and unit-tested.
- **P3:** §5 research fleet.
- **P4:** join math with research features; produce ranked adds per league.
- **P5:** a cold adversary agent tries to break each league's top 3; a validator agent checks the adversary's claims against sources; orchestrator keeps, drops or re-ranks.
- **P6:** §6 output; save `data/waivers/2026-wk09/{state,research,recs}.json` for backtesting.

## 4. League math (pure JS in `rules/`, 100% coverage)
1. **League-scored projections** μ, σ per player per week for N+1, N+2, N+3 and the playoff weeks. v0 uses each platform's own projection (already in league scoring on ESPN; Sleeper projections re-scored with `scoring_settings`); v1 blends consensus and market (S8).
2. **Lineup optimizer** `L*(roster, week)` = best legal lineup points, respecting slots, flex eligibility, byes, OUT/IR, and already-locked players.
3. **Move value:** for add *a* / drop *d*, ΔL_w = L*(R+a−d, w) − L*(R, w).
4. **Weekly win prob:** P_w = Φ((μ_me − μ_opp) / √(σ_me² + σ_opp²)), using the actual scheduled opponent each week.
5. **Season + playoff Monte Carlo** (seeded RNG, `S.sim.n` runs): simulate remaining regular-season weeks with each team's lineup distribution, apply the league's real seeding tiebreakers (H2H, points for, division, etc. — read from settings), then the bracket. Outputs: P(playoffs), P(bye), P(title), "magic number" (wins needed for ≥90% playoff odds), and whether points-for matters (PF tiebreak and close race → ceiling plays matter more).
6. **Recency weighting** (Jeff's rule: prioritize the next 3 weeks in recency order): short-term value = Σ r_w · ΔP_w for w = N+1..N+3 with `S.waiver.recency_weights` (default 0.5 / 0.3 / 0.2), scaled by playoff leverage λ = ∂P(playoffs)/∂(one more win) from the sim. Playoff-week value = P(playoffs) × Σ ΔP_w over playoff weeks × `S.waiver.playoff_weight`. Final rank uses ΔP(title); the recency score is shown as the explanation.
7. **Phase rules:** bubble team → weight next 3 weeks hardest; locked-in team → weight playoff weeks, handcuffs, playoff-week byes; eliminated → nothing (or keeper value only in Sleeper); underdog vs strong field → prefer higher-variance adds (the sim captures this if σ is per-player).
8. **Positional value:** replacement level = best available FA at that position in *this* league (deeper leagues → each add worth more). Penalize spending on positions with a deep stream pool (e.g. 8 playable DSTs on the wire).
9. **Drop selection:** lowest ΔP(title) cost; protect handcuffs of Jeff's starters (`S.waiver.protect_handcuffs`), keeper-eligible Sleeper players (keeper value term), and IR-eligible players (move to IR instead).
10. **FAAB bid:** value-based ceiling (ΔP(title) × budget-per-title-point given weeks left) vs market estimate (FantasyPros FAAB %, Sleeper trending adds, this league's past winning bids, rivals who need the position *and* have budget). Show *bid* and *walk-away max*. Rolling/reverse waivers: claim only if value > expected value of keeping priority (`S.waiver.priority_threshold`).
11. **Roster limits:** acquisition limits, position maxes, lineup-lock (cannot drop a player whose game started, platform-specific) — read, never assumed.

## 5. Research fleet (P3)
6 research agents run in parallel (Sonnet to save limits), each owning a source cluster, each returning strict JSON `{player, pos, team, source, url, published_utc, rank_or_tier, faab_pct?, role_note, injury_note, confidence}`.

| Agent | Sources | Gives |
|---|---|---|
| A. Waiver columns I | FantasyPros waiver rankings + FAAB %, The Fantasy Footballers waiver show/article, ESPN (Field Yates / Mike Clay) | ranked adds per position, FAAB % |
| B. Waiver columns II | Yahoo (Andy Behrens), CBS (Jamey Eisenberg), NFL.com, Footballguys, RotoBaller, PFF free articles | ranked adds per position |
| C. News + injuries | NBC Sports EDGE (Rotoworld), RotoWire news, official injury reports (Wed–Fri), team beat reporters via search | role changes, injuries, depth chart, return timelines |
| D. Usage | FantasyPros snap counts, Fantasy Life utilization (free part), nflverse (if added) | snap %, routes, target share, carries, red-zone share, trend |
| E. Market | ESPN scoreboard odds (DraftKings spread/total → implied team totals N+1..N+3 where posted), Sleeper trending adds/drops, player props for top candidates where reachable | implied totals, crowd demand |
| F. Schedule | defense vs position (FantasyPros), bye matrix, playoff-week (15–17) schedule strength | matchup grades for N+1..N+3 and playoffs |

**Source rules**
- **Assumption:** "roto" = both NBC Sports EDGE (Rotoworld) and RotoWire. **Assumption:** "Fantasy Football advice" = the r/fantasyfootball community; it blocks this container, so it is read only through search snippets at low weight. Jeff to confirm if a specific site was meant.
- **Fallback chain per source** (never a single point of failure): direct fetch → RSS / AMP / print view → web search for this week's article title (snippets) → syndicated copy (Yahoo/MSN/Apple News, podcast show notes, YouTube description) → Wayback snapshot → mark `unavailable` with reason.
- **Quorum:** a run is "full" with ≥ `S.research.quorum_min` (default 6) of the 10+ sources fresh (≤ `S.research.max_age_hours`, default 72); otherwise the brief says "partial research: n/12" at the top. It never silently degrades.
- **Hard line:** no CAPTCHA solving, no bot-check evasion, no paywall bypass. A blocked source is routed around via the chain above, not through.
- **Consensus score** (script, `rules/consensus.js`): weighted Borda across sources (`S.research.source_weights`, recency-decayed), plus a disagreement index. High disagreement = flagged, not averaged away.

## 6. Output (mobile-first)
**Top: 30-second table**
| League | Rec (seed) | Playoff % / Title % | Waivers clear (ET) | #1 move | Bid / priority |
|---|---|---|---|---|---|

**Per-league card**
- Status line: record, PF rank, playoff odds, magic number, tiebreak note, next 3 opponents.
- Needs grid: positions × weeks N+1..N+3 + playoffs (OK / thin / hole), showing byes and injuries.
- **Ranked adds (max 5):** Add · Pos/Team · Drop · Bid (max) · ΔTitle % · Why (one line) · Biggest risk · Consensus (k/12 sources).
- Streamers by week (QB/TE/K/DST), stash/handcuff for playoffs, **don't-chase** list (hype adds the math rejects), contingencies ("if X is claimed, pivot to Y").
- Data freshness + research coverage footer.

Delivered as a chat brief plus an HTML artifact page; JSON archived for the weekly scorecard (did recommended adds outscore the consensus #1? did bids overpay vs winning bids?).

## 7. Files
```
.claude/skills/ff-waivers/SKILL.md          orchestration steps
.claude/skills/ff-waivers/agents/*.md       prompts: research A–F, adversary, validator
.claude/skills/ff-waivers/sources.json      URLs, fallback chains, weights, parse hints
tools/waivers/run.js                        P0–P2, P4, P6 driver
rules/waivers.js  rules/playoffs.js  rules/consensus.js   pure math, 100% coverage
fixtures/waivers/                           scrubbed league + research fixtures
```

## 8. Phasing
| Version | Needs | Works |
|---|---|---|
| v0 | S0 done (league URLs, Sleeper username, ESPN cookies) | Sleeper + ESPN live, Yahoo via screenshot; full research fleet; platform projections; simple sim |
| v1 | S4a, S4b, S5a | league-scored projections, optimizer, full title-odds sim, FAAB model |
| v2 | S7, S9 | site cards, approve queue, deep-link moves |

## 9. Needs from Jeff
S0 items (5 league URLs, Sleeper username, ESPN `espn_s2`/`SWID`), Yahoo league screenshot until S3, confirm "Fantasy Football advice" source, main-run time, optional Odds API key.
