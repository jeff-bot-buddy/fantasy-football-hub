# `ff-waivers` skill — plan v2 (after 3 adversaries + 2 validators, 2026-11-05)

**Goal:** before each waiver deadline, give the claims to enter, in order, for each of Jeff's 5 leagues separately: add, drop, bid and fallback. Each claim must raise **Jeff's odds of winning that league's title**.

**One currency:** ΔP(title) for each move. It comes from a Monte Carlo of the rest of the season and the playoffs under each league's own rules, with common random numbers (CRN). Recency, positional value, FAAB and blocking all feed this number.

**Honesty rule:** the run never fails *silently*. Every brief states the age of each data source, how many sources it read, and any `RuleBlocked` items. Blocked sources are routed around, never through: no CAPTCHA solving, no bot-check evasion, no paywall bypass.

**Fits in BUILD_PLAN as:** rows W0a and W0b (the v0 skill), then S5b-1 and S5b-2 (full math). See §9.

---

## 1. Verified platform rules that change the math
Verified by validators against the official help centers on 2026-11-05. Each league's actual settings still override these.

| Rule | ESPN | Sleeper | Yahoo |
|---|---|---|---|
| When waivers process | Daily, 3–5 AM ET. Default waiver period is 1 day, and a claim needs a full 24 h before it processes. | Commissioner sets the day and time. Help text cites a 3 AM Wednesday default; the timezone is unverified. | Default "Game Time – Tuesday": waivers end after 11:59 PM PT Tuesday (= 2:59 AM ET Wednesday). 2-day waiver period after a drop. |
| FAAB tie | Waiver tiebreaker order | Rolling priority. $0 bids allowed by default. | Priority type the commissioner chose |
| Does a win cost priority? | **Yes, any win** moves you to the bottom, even with "Reset each week" | Yes, any successful claim moves you to the end | FAB: only when a tie decides it. Reverse standings: **no**. Weekly rolling: yes. |
| Order of one team's claims | UNVERIFIED (test on the first real claim) | **By bid amount**; can reorder only claims with equal bids | Your ordered list. A later claim is **voided** if its drop player is already gone. |
| Seeding tiebreaker | ESPN's own help pages **contradict each other** on the default. Division winners seed first; a tied playoff game goes to the higher seed. | Read setting | Read setting |
| League-median game | Unknown → read setting | Regular season only | Unknown → read setting |
| Gotchas | A claim **fails silently** with a healthy player in IR or an uneven pending trade | — | You can't re-claim your own drop until its waiver period ends |

**NFL 2026 byes (nfl.com, confirmed):**

| Week | Teams on bye |
|---|---|
| 10 | CHI, DEN, PHI, TB |
| 11 | ATL, CLE, GB, LAR, NE, SEA |
| 13 | BAL, IND, LV, NYJ |
| 14 | ARI, DAL |

There are no byes in weeks 12 or 15–18. After week 14, a bench spot kept for bye cover is dead weight and should hold a stash.

## 2. Inputs per league (data contract)
Scoring settings, roster slots, IR slots and position maximums, waiver type and timing, FAAB budget and tiebreak, acquisition limits, **all** rosters, free agents, standings (PF/PA), the remaining schedule, divisions, playoff format (teams, byes, weeks, 1- vs 2-week rounds, reseeding), **seeding tiebreakers**, **median game**, keeper rules (Sleeper), and transactions (winning FAAB bids, who is still active).

An unknown rule shows `RuleBlocked('<question>')` on the card. It is never guessed (CLAUDE.md).

## 3. Cadence (ET)
| Run | When | Does |
|---|---|---|
| **Main** | Tue 1:00 PM | Full run. Set late because this week's CBS (12:03 PM) and ESPN (11:10 AM) columns came out after 9 AM. |
| **Final** | Tue 9:00 PM | News, injury and availability check only. Locks bids before the earliest deadline (Yahoo 2:59 AM, ESPN 3–5 AM). |
| **Wake-up** | Wed 7:00 AM | Waivers have cleared. Runs the "if X wasn't claimed, add Y" list prepared Tuesday night (free agents are first come, first served). |
| **Injury pivots** | Thu/Fri after practice reports | Free-agent replacements only |
| Inactives | Sun 11:30 AM | Belongs to `ff-weekly` (lineups); linked from here |

A Sunday-night free-agent sweep (adding before rivals can bid) depends on whether each platform locks players onto waivers once their game starts. That is unverified: read it per league.

Data is archived by **target week**: `data/waivers/2026-w10/`. The target week is the next week with unplayed games on the ESPN scoreboard, not the Sleeper week label.

## 4. Pipeline
```
P0 preflight ─ P1 fetch scripts (deterministic, cached, date-checked) ─ P2 league math ─┐
               P3 research agents R1–R3 (parallel, read saved pages)  ─────────────────┤
                                                                       P4 merge + claim portfolio
                                                                       P5 cold adversary R4 → validator R5
                                                                       P6 cards (chat + HTML) + archive + scorecard
```

## 5. Data layer: scripts for anything structured, agents only for reading
Every fetch uses curl with an honest named user agent (`fantasy-football-hub/0.1`). WebFetch summarizes, misses embedded JSON and caches, so it is never used for primary data. Raw pages are saved to `data/waivers/<wk>/raw/` and hashed.

| Script | Endpoint (all tested 200 today) | Gives | Traps found |
|---|---|---|---|
| `espn_public` | `lm-api-reads…/leaguedefaults/3?view=kona_player_info` + `x-fantasy-filter`, no cookies | % owned, Δ%, ESPN projections weeks 1–18 incl. playoffs, raw stat lines (can be re-scored) | Mixes 2025 and 2026 data: filter seasonId=2026, statSourceId, statSplitTypeId. A bye week shows a 0.0 projection. |
| `espn_league` / `sleeper_league` / Yahoo | league views (S1–S3) | rosters, free agents, settings, transactions | ESPN private leagues need cookies; Yahoo needs OAuth (screenshot until S3) |
| `sleeper_usage` | `api.sleeper.app/stats/nfl/2026/{wk}`, `/projections/nfl/2026/{wk}` (unofficial) | snaps (`off_snp`/`tm_off_snp`), targets, RZ, raw projections (RotoWire) | `last_modified` is a bulk re-save time: **snapshot our own copy every Tuesday** |
| `nflverse` | hard-coded release-asset URLs: `snap_counts_2026.csv`, `injuries_2026.csv`, `stats_player_week_2026.csv` | usage and injuries (through week 8) | This session's proxy blocks the release *listing* pages, but the asset downloads work |
| `injuries` | `site.api.espn.com/…/nfl/injuries` | news-style injury feed | **Capped at 25 per team** (8.7 MB): a player's absence doesn't mean he's healthy. Status comes from kona, Sleeper or nflverse. |
| `odds` | Action Network scoreboard for weeks N+1..N+3; ESPN scoreboard (DraftKings) for N+1 | spreads and totals → implied team totals | ESPN has lines for N+1 only. Action Network is undocumented: cache it and record fetch time. |
| `fp_waiver` | FantasyPros `waiver-wire-{ppr,half-point-ppr,}-overall.php` → `ecrData` | 11-expert ranks, owned %, $ tag | Use the URL that matches the league's scoring. `?week=N+1` returns empty, so require `week==target && count>0`. |
| `fbg_faab` | Footballguys waiver hub | FAAB % per player | — |
| `trending` | Sleeper trending adds and drops | demand signal | — |
| `discover` | Google News RSS | this week's article URLs | Search results mix seasons; use them to find URLs only |

**Date gate (all items):** keep an item only if its `datePublished` falls after the previous slate ended and its season is 2026. A URL slug is never trusted: we saw 2025 articles behind 2026-looking slugs and a reused ESPN URL.

**Player resolution:** availability is checked **in each platform's own player list** (that league's free-agent list), never through Sleeper's crosswalk. Sleeper has an ESPN ID for 0 of 305 active rookies and for only 24.6% of players on an NFL team. Research names are matched by name + team + position, with `active==true` and collision detection. An unresolved name is **excluded and listed**, never guessed. Availability is re-checked at the final run.

**Sources:**

| Use | Sources |
|---|---|
| Read by agents (11) | FantasyPros article, The Fantasy Footballers (free article only; see decision D2), NBC Sports EDGE / Rotoworld, RotoWire (facts only; its analysis is paywalled), ESPN, Yahoo, CBS, NFL.com, Footballguys, RotoBaller, PFF (main article) |
| Data feeds (5) | Sleeper, ESPN APIs, nflverse, Action Network, FantasyPros `ecrData` |
| Dropped | Fantasy Life (paywall), Reddit (blocks this container), Pro-Football-Reference (Cloudflare challenge), Wayback (429), syndicated copies (stale duplicates) |

**Quorum:** the run is "full" when the required core is fresh: platform data, FantasyPros, injuries/news and odds. It must also have ≥6 *independent* columns read. Anything less shows "PARTIAL n/11" at the top of the brief. Consensus is counted per player, only across the sources that covered him. It is shown as a flag, not used as a ranking input, because columnists echo each other.

## 6. Agents (per run)
| Agent | Model | Job | Returns |
|---|---|---|---|
| R1 Columns A | Sonnet | FantasyPros, Footballers, ESPN, Yahoo, CBS (saved pages) | `{name, team, pos, src, url, published, rank, faab_hint, note}` |
| R2 Columns B | Sonnet | NFL.com, Footballguys, RotoBaller, PFF, NBC analysis | same |
| R3 News + roles | Sonnet | The top ~30 candidates across all leagues: NBC player news, RotoWire facts, ESPN news, practice reports, beat reporters | **role scenarios** `{s, π_s, points path μ_w,s, resolves_week}` |
| R4 Adversary | Opus, cold | Tries to break each league's claim list: stale tag, temporary role, starter returning, overbid, lock timing, already rostered | objections with evidence |
| R5 Validator | Opus, cold | Checks R4's claims against the saved pages and the APIs | keep / drop / re-rank |

5 agents per run. Everything structured is a script, so it is repeatable, testable and cheap.

## 7. League math (pure JS in `rules/`, 100% coverage, seeded RNG)
1. **Projections:** for each player and week (N+1..N+3 plus playoff weeks), raw stat lines from ESPN *and* Sleeper/RotoWire are re-scored under each league's scoring. Two independent sets let us flag a move whose sign flips between them.
2. **σ:** fitted from 2026 weeks 1–8 projection error by position and projection band. Validators found σ/μ ≈ 0.6–0.9 for WR/RB/TE projected 4–12 points (right-skewed) and ≈ 0.42 for QB. A normal is fine at team level: stacking correlation changes team σ by only 3–5%.
3. **Lineup optimizer** `L*(roster, w)`: legal slots and flex, byes, OUT/IR, locked players.
4. **Season and playoff sim:** team-level, CRN, the real schedule, median game, divisions, seeding tiebreakers from settings, byes (a bye ≈ doubles title odds: 0.55² vs 0.55³), 1- or 2-week rounds, reseeding. Outputs: P(playoffs), P(bye), P(title), and week sensitivities g_w = ∂P(title)/∂μ_w and h_w = ∂P(title)/∂σ²_w.
5. **Move value:** ΔP ≈ Σ_w (g_w·Δμ_w + h_w·Δσ²_w).
   - Δμ_w is **scenario-weighted with a cut option** (if the role fails, Jeff cuts him).
   - It is measured against the **replacement path** F_w (what Jeff could stream or claim later anyway; fitted from this league's weeks 1–8 free-agent pickups).
   - It is measured against the **rival counterfactual**: P(title | Jeff adds) − P(title | likeliest rival adds). That rival is chosen by need, budget, activity, and being a next-3 or playoff opponent. This is what values blocking adds.
   - All ~750 add/drop pairs are screened with the linear formula; the top 5 per league are fully re-simulated at 10k CRN runs. Validator: CRN SE ≈ 0.07–0.14 pp, versus ≈ 0.35–0.58 pp with independent draws.
   - **"Hold"** when the best ΔP is under `S.waiver.hold_pp` (default 0.2 pp).
6. **Jeff's recency rule** (next 3 weeks in recency order): by default the sim's g_w decides how much each week counts, and the card shows each move's "value by week" split. In the toy sims a bubble team gets 47–56% of its value from playoff weeks; an 8-1 team gets 99%. A fixed 0.5/0.3/0.2 would mis-rank those cases. The fixed weights stay as the v0 fallback and as a mode switch: `S.waiver.mode = 'title' | 'recency'` (**decision D1**).
7. **Phase overrides:**
   - P(playoffs) > 95% → playoff weeks plus the weeks 13–14 race for a bye.
   - Eliminated → keeper value only (Sleeper), otherwise nothing.
   - Underdog → h_w rewards variance.
8. **Drops:** lowest ΔP cost. A favorite keeps its own handcuffs; an underdog targets rivals' handcuffs. The run also scans for IR-eligible free agents who return by week 15 (they cost no roster spot) and treats bye-cover bench spots as dead after week 14.
9. **FAAB (first-price sealed bid):**
   - Value of a claim: V = ΔP(add) − ΔP(fallback if lost).
   - Price of a dollar: λ$ = ∂V_k(B)/∂B, from a small dynamic program over the remaining waiver runs. It falls to 0 by the last run that can matter and rises after the trade deadline.
   - Rival bids F(b): only rivals who need the position and are active. Lognormal around the FantasyPros $ tag / Footballguys % × this league's inflation (from past winning bids), capped at each rival's public remaining budget.
   - **Bid = argmax F(b)(V − λ$·b)**. Walk-away max = min(V/λ$, top rival budget + $1). The card shows P(win claim).
10. **Priority waivers:** claim if V > the value of keeping priority, using the platform cost table in §1. Keeping priority is worth 0 on Yahoo reverse standings; claiming still costs it on ESPN, even with a weekly reset.
11. **Claim portfolio:** up to `S.waiver.max_claims` (3) claims per league, found by enumerating add × drop × bid with each success/fail combination re-solved. "Only if #1 fails" uses the same drop player. On Sleeper the fallback bid must be ≤ the first claim's bid, because Sleeper orders claims by bid.
12. **Keeper (Sleeper):** `S.keeper.future_weight` puts next year in this year's ΔP units. `RuleBlocked` until the keeper-cost rule is in LEAGUES.md.
13. **Trade chips (v1, needs S6):** until each deadline, a trending add whose market value exceeds his value to Jeff is worth adding to sell.
14. **Matchups:** defense-vs-position is heavily regressed (year-to-year r ≈ .16–.27, 4for4). Playoff-week strength of schedule gets low weight. Weather is used only inside 5 days.

**Validation:**
- **Matchup calibration** on 2026 weeks 1–8: z = (actual − projected margin)/σ_diff should have variance ≈ 1. Brier score versus the platform's own win probability.
- **Logged weekly:** projection snapshot, scenarios, ΔP, bid, P(win), result, winning bid, add vs drop points over 3 weeks, whether he was started.
- **Weekly scorecard:** claim-win calibration, overpay, and points versus the consensus #1.

## 8. Output (phone, Tuesday 11 PM)
**Top table, sorted by deadline:**

| League | Clears (ET) | Do | ΔTitle | Title % |
|---|---|---|---|---|

**Per league:**
```
ESPN·Bros  6-3 (#4)  PO 71% · Title 11.2%   ⏱ Wed 3:00a ET (27h)  $38 left
ENTER IN THIS ORDER
1 +J.Doe RB NYJ  −K.Roe WR  $14 (max $19)  win≈60%  +1.1pp
2 +A.Poe RB TEN  −K.Roe WR  $6   ↳ only if #1 fails (same drop)
3 +BUF DST       −NYJ DST   $0   (wk10 stream)
All hit: Title 11.2→12.6% (±0.3) · Conf MED (sign holds 2/2 projection sets)
Why #1: starter Q-ham; 68% snaps wk9. Risk: starter back wk11 (40%) → value ×0.5
Value: 35% wk10–12 · 65% playoffs · Holes: wk10 TE (bye) · Skip: X (hype, +0.1pp)
Wed 7am if unclaimed: Y · [Open ESPN claims ↗]
```
**Footer:** sources n/11 fresh · data ages · RuleBlocked list · adversary changes.

The needs grid and magic number are cut (they don't fit a phone). The run also writes a JSON archive and the weekly scorecard.

## 9. Build rows (proposed for BUILD_PLAN)
| Row | Model | Delivers |
|---|---|---|
| W0a | Sonnet | `sources.json` registry; fetch scripts in §5 with date gate, caching and honest user agent; name→platform resolver with collision checks; fixtures and tests |
| W0b | Opus | `SKILL.md` and agent prompts R1–R5; team-level CRN sim (schedule, median, tiebreakers, byes); recency-mode fallback; card renderer; **first live Tuesday run** (Yahoo by screenshot) |
| S5b-1 | Opus | role scenarios with cut option, replacement path F_w, rival counterfactual, claim portfolio |
| S5b-2 | Opus | FAAB auction with λ$ program and rival bid model; weeks 1–8 calibration; scorecard |

W0a/W0b go right after S2 (ESPN read), so 4 of 5 leagues are live. **Assumption:** if S0 is finished by Sat 11/7, S1 → S2 → W0a → W0b can make the Tue 11/10 run (week 10). Otherwise Tue 11/17. About 8 waiver runs remain through week 17.

## 10. Decisions and inputs needed from Jeff
- **D1:** use sim-driven week weights (recommended), or keep fixed 0.5/0.3/0.2 recency weights?
- **D2:** are you a Footclan member? The Footballers' structured rankings JSON is marked `toolLocked`, so only their free article is used unless you have access.
- **D3:** "Fantasy Football advice": which site did you mean? Reddit blocks this container.
- **D4:** OK to slot W0a/W0b after S2?
- S0 items: the 5 league URLs, Sleeper username, ESPN `espn_s2`/`SWID`. A Yahoo screenshot until S3. Odds API key optional (player props only).

## Adversary log (what changed and why)
| Finding | Source | Verdict | Action |
|---|---|---|---|
| Recency formula double-counts and contradicts the rank | A1, A3 | Accepted | Sim-driven g_w; recency is now display + mode switch (D1) |
| Value vs "rival gets him" (blocking) | A1 | Accepted | Rival counterfactual in §7.5 |
| Handcuffs/stashes valued ≈0 by μ+σ | A1, A3 | Accepted | Role scenarios with cut option |
| FAAB "budget-per-title-point" incoherent | A3 | Accepted | First-price auction + λ$ program |
| Ranked list ≠ what you enter | A3 | Accepted | Claim portfolio with conditional fallbacks |
| "No byes weeks 15–18"; median game; ESPN priority loss | A1, A3 | **Validated** | §1 tables |
| Scripts > agents for structured data; curl, not WebFetch | A2 | Accepted | §5; fleet 6 → 3 research agents + 2 checkers |
| Date/season traps in URLs and search | A2 | Validated | Date gate |
| Sleeper ID crosswalk fails for waiver-pool players | A2 | **Validated** (worse than claimed) | Resolve in each platform's own list |
| nflverse "blocked" | A2 | Validated as fixable | Asset URLs work |
| FantasyPros is STD-only | A2 | **Refuted** | PPR/half pages exist |
| "Antonio Williams" collision | A2 | **Refuted** (one is inactive) | Filter on `active` |
| ESPN WAF challenges a Chrome UA | A2 | **Not reproduced** | Honest UA anyway |
| Sleeper past projections leak results | A3 | **Refuted** (bulk re-save, values genuine) | Snapshot our own anyway |
| Run 2 h before every process | plan v1 | Cut (Jeff asleep) | Tue 9 PM final run |
| Shrink sources to 3, drop consensus | A1 | Partly rejected | Jeff asked for 10+; consensus kept as a flag |
