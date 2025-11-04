# Valorant Prop Model Documentation

## vlr_patchpool.csv

Pipeline begins with a helper "patchpool file". Inputs into this code block are a roundstats csv, player stats csv, and match stats csv.

The **round stats csv** provides game_ids, map level keys, and round level results per team. With this information, a per-team, per-game round differential is built that is used for flagging map wins, and capturing margin of victory, a feature used downstream. Round differentials per map are aggregated later at a series level.

**Player stats csv** contains essential information such as player team, game_id again as a key to link players/rosters to maps and subsequently matches, and the player names themselves. We can then build rosters at a match level, using the team names to delineate between both teams involved. This is used downstream to calculate roster stability metrics.

The **match stats csv** contains lot of essential context: the game (map) id core key, match datetime, picks and bans (parsed into team, priority specific columns), match patch, map name, team names, competition name, and match_best_of, containing the best of format. We consolidate important information that delineates a match: the date/time, team1 and team 2 names, the competition name (tournament), and whether it is a best of 3 or 5. From there, we attach map ids (ordered game_ids), the relevant pick and ban columns, and attach per map round-differentials. Map names are backfilled and patch information is standardized and added in.

Within the script itself, some other important additions include consecutive game-id enforcement at a match level, which ensures matches with forfeits/missing maps are skipped and row filters for data quality (ensuring a full 7 map pool, presence of patch/round differential, 2+ map picks). There also is creation of roster stability features, that indicate how long different ratios of the roster (3/5 players, 4/5 players, 5/5 players) have been together on the same team which is derived from historical series ordering.

## vlr_matchstats_with_matchid

Adds match_id from patchpool into original matchstats file.

## vlr_features

Cleaned, player-map level dataset enriched with temporal, roster, context features. After loading and cleaning, this code merges match context into individual player rows. Basic features are then computed: patch indexes (patch information converted into integer that is easy to parse downstream), total rounds played, and kills per round. Junk maps are filtered out (ie less than 13 rounds, the min rounds to win a valorant match without a forfeit). 

After sorting by player handle and datetime, distinct match timestamps are identified at a player level to avoid dupes from the same match, then days_since_last_match is computed, the difference between current match time and previous distinct match time for that player. Margin of victory is also assigned at a player level based on team1/team 2 round differentials. Roster continuity features computed at a match level in patchpool are recomputed at a player-map level.

## map_preds

Map prediction engine that takes patchpool, player-level features and takes historical picks/bans, performance patterns to predict map/pick bans in the future.

Activemappool, assumed from rows based on patch/datetime, is utilized as a list of available maps to predict. A lookup is built that maps match_datetime to patch index, allowing weighing of past matches by the distance of a match, and team histories are built chronologically to see pick/ban tendencies over time.

`compute_team_map_scores` iterates over a team's past matches before a prediction date, and applies weights that downweigh older patches, older dates, and factors in roster confidence, which is higher if core ratios (3/4/5 of 5 players) are stable for longer periods of time. Different weights were gridsearched and assigned to historical picks/bans based on order: the first pick a team makes is worth much more than its 3rd pick, and similar for bans; and bans are worth less than picks as a map tendency predictor. Contributions are collected from these pb_weights as well as performance history (round differential/margin of victory on a map) as well.

Valid prediction contexts are filtered for next: a loop looks at each unique MatchID, and only keeps valid 7 map pools, teams that have at least 5 matches played, and the number of matches played matches the best_of format (2 for bo3s, 3 for bo5s to match kill prediction markets offered elsewhere).

For each context deemed valid, scores are computed for both teams involved, added per map, and softmax normalized to the number of maps actually played. Capping is used to clip probabilities at 1 (otherwise teams that historically tend to both pick the same map with high priority could have map probabilities that add to over 1 despite normalization), with redistribution of probability mass occurring in such cases. The final result is the probability of each map across the 7 map pool.

## agent_probs

Agent prediction engine; begins with ingestion of the features csv and the map preds per-match, per map probabilities.

Additional behavior includes active roster selection, which uses the last match's roster that played together, up to 5 players, and for each prediction context it tries this exact last roster, and otherwise falls back to filling missing spots with the most recent unique players.

Probabilities are then built; at each row in map preds, a player set is chosen for each team. 3 channels are computed concerning an agent tendency, all past-only with decay:

1. Player-specific history on the map (team-specific first, falls back to any-team history if requisite sample size not met)
2. Team-specific history
3. Meta history (what other teams are running composition wise on that map)

Each channel uses patch decay, time decay, roster decay weights with margin of victory multiplier. Via gridsearch tuned player history to weight the most (0.6), other 2 at 0.2. These probabilities are scaled by map probabilities, and then normalized to the number of maps actually played in the series. There is also a role suppression component to make sure probabilities align with compositions that make sense. There is a low-probability cutoff to redistribute mass from player-agent-map combos below 10% (often generic team/meta weight additions that don't make sense in a player-level context but are generically evenly distributed across a team to fit historical expected compositions).

## agent_coefs

Agent specific coefficients. Used to represent agent-specific kill impacts at a map level.

This block starts by loading the features csv, then filters to valid rows and computes kills per round. Via an eyetest I determined cypher to be the optimal base agent, to be used as a baseline for kpr agent adjustments (closest to neutral amongst agents). There is time windowing every 7 days, with coefficients in backtesting using data strictly before that date. Coef rows factor in patch/recency decay.

Global WLS (weighted least squares) regression is calculated next at an agent level, where the model asks how well each agent does across all maps relative to cypher. This is primarily used as a fallback in the case an agent doesn't have a significant enough sample size at a map level. If there is enough data, there is map WLS at an agent level again comparing to cypher. Shrinkage is used as another sample size guardrail, where agent impacts are shrank towards its global number based on data. There is a final pass at end_date+1 day to produce latest coefficients for the separate future preds pipeline.

## vlr_playerstats_with_matchid

Adds match_id from patchpool into original playerstats file.

## actual_kills

Used for backtesting prediction model accuracy; looks at first 2 maps of bo3s, 3 maps of bo5s like predictions/betting markets.

## neutral_kpr

Leak safe neutral (agent-agnostic) prediction of KPR at a player, matchid level for backtesting.

Inputs include the features csv, agent coefs csv, and map preds csv. For each map_name, player_agent combo in features, we merge in the most recent past agent_map_coef before the computation date, with rows with no past agent coefficient dropped. Neutral KPR residuals are subsequently computed by deducting agent_map_coefs from the computed kprs, creating this agent-agnostic effect. After clipping min/max kprs, match level aggregation is performed by weighing per-map neutral baselines by total_rounds_played within a match, and only outputting 1 row to reflect a single match.

## kill_preds

Backtest for per-player predictions at a match level. Inputs are agent probs for match/player/agent probabilities, features csv for player history related to decay weights, agent coefs, neutral kpr at a match/player level. This outputs kill predictions per player, matchid combination, and a trained xgboost model to be used downstream in the future predictions script.

Core logic starts with building a map-agent index for coefficients; a lookup is used at a row level using past-only coefs. A leakproof neutral player baseline is then brought in. Patch, recency, and core confidence decay processes are incorporated here. Neutral kpr/agent additional kpr are scaled to match level values by multiplying these respective values (neutral kpr, agent coefs) each by final_series_prob (matching betting markets, 2 for bo3, 3 for bo5), and rounds per map (historical value). XGB training then ensues on the feature set, which includes the aforementioned patch, recency, core confidence decay, neutral kpr/agent coef player-match values, and shrink_w_scaled, which looks at how stable a player's history is based on sample size. Upon validation a corrective bias is applied.

## future_map_preds

Rather than using the patchpool file for input information, a helper file for matches offered at other bookmakers is fed in/utilized to predict maps for upcoming matches. Patchpool/features are used to build team histories before the dates in the helper file. Outputs are predicted probabilities over the full pool, without map_was_played truth labels. Shares the same scoring function, normalization logic, same outputs.

## future_agent_probs

Ingests features, future map preds, future juicer out helper to predict leakage-proof, future agent probabilities. Logic prefers the exact last members of a team who played together.

## future_neutral_kpr

Ingests features, agent coefs, future map and agent preds to produce neutral kpr baselines for upcoming slates. This version is matchidless, and uses future map probabilities rather than historically realized maps.

## future_preds

Similar logic to backtesting kill preds, but with futurified versions of agent probs, neutral kpr, joins/keys on datetime, team names, player handles instead of matchid. Leakproof lookups on decays, coefs, neutral kpr that guarantees no forward lookaheads. Decays/shrink calculated the same. Future script doesn't train, instead loading the trained XGB model and bias from backtest pipeline, which it applies to future rows. Same final output in terms of kill series projections.
