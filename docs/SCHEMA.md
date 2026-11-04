# Common data model (stub — filled in S1–S4b)

- `League{platform, id, season, name, scoring{stat→pts}, roster_slots, keeper{count, cost_rule, deadline}?, waiver{type: FAAB|rolling|reverse, budget, process_time, waiver_period_days, tiebreak}, pickup_limits, trade_deadline, trade_review_days, playoff_weeks}`
- `Team{league_id, team_id, owner, is_jeff, roster[player_hub_id], faab_remaining}`
- `Player{hub_id, espn_id, sleeper_id, yahoo_id, name, pos, nfl_team, status, bye}`
- `Matchup{league_id, week, home, away, proj, actual}`
- `Transaction{league_id, ts_utc, kind, players}`
- `Move{id, kind: lineup|add_drop|claim|trade_offer, league_id, payload, status: proposed|approved|sent|confirmed|failed|rejected, created_utc}`

Checklist to capture per league (S4a): superflex/2QB, IDP, TE premium, fractional PPR, bonuses, IR slots, position maxes, per-game lineup lock, waiver timing, playoff end week (17 vs 18).
