---
layout: post
---

# The Training Ground

*March 6, 2026*

*Related to: [gaffer\_ai](https://github.com/samsohn28/gaffer_ai)*

Welcome to pre-season training! With only 8 fixtures left on Arsenal's Premier League calendar, I'm using the final stretch of this title run to do two things: watch the gunners lift the trophy, and get our baby robot gaffer match-fit in the process. As I covered in my [last post](/blog/2026-03-03-first-things-first), the brains of this operation is a machine learning model — and if that means nothing to you, don't worry. Here's Claude's ELI5:

> Imagine you're trying to get good at guessing how many sweets are in a jar. You don't know the formula, but every time you guess, someone tells you if you were too high or too low. After enough guesses, you start to notice things — bigger jars mean more sweets, rounder sweets pack tighter — and your guesses get better. Training a model is the same idea. You show it thousands of past gameweeks and say: "here's what this player looked like before the match, and here's how many points they scored." It guesses, checks how wrong it was, and nudges itself in the right direction. Do that enough times and it starts to pick up on what actually matters — form, opponent strength, position — without anyone telling it the rules. Then you point it at next weekend's fixtures and let it have a go.

Originally I wanted my machine learning model to be a deep neural network, but it turns out that was only aspirational. What I actually built was an XGBoost regressor. These gradient-boosted trees are what you reach for when you only have a few thousand rows of mixed data. The good news is that XGBoost is a legitimate choice here, not a compromise. The bad news is that some of the things I was most excited about (injury signals, ownership data, bookmaker odds, etc) are completely inert in this first version. More on that later.


## The Data

Yesterday, I spent the better part of the evening building a "data lake". I tried to source as much data as I could from as many different sources as I could find. I didn't think too much about whether it was useful and I didn't really care how clean the data was (or even really look much at it). It was hungry hungry hippos, except all of the balls feed into one bucket. I call all this raw data bronze-level data. It's not really that useful by itself, so, we need to clean and combine it.

After all that feasting, the model trains on three main sources: FPL's own historical data, Understat, and ClubElo ratings. The FPL data is the backbone providing per-player, per-gameweek history covering points scored, cost, form, ICT index, fixture context, and player flags like set piece and penalty taker responsibility. Understat adds shot-level stats like xG and xA per match. ClubElo contributes a single strength rating per club at each point in time, built on the Elo system with adjustments for home field advantage, goal difference, and inter-league competition. We clean the data up a bit (convert strings to floats, drop unnecessary columns, etc) and then write it into parquet files to get to our silver-level data.

We take our three disparate silver tables and fuse them together into one wide feature table — one row per player per gameweek. This is our gold-level data. Each row gets the full treatment: rolling stats from FPL history, ClubElo ratings for both the player's team and their opponent, and xG/xA figures joined in from Understat. All of the historical rows covering GW1 through GW29 and have a real `total_points` value which is what the model trains on. Next-GW rows have one entry per active player for the upcoming gameweek, with everything filled in except the points, because the match hasn't happened yet. Those are what we hand to the model down the line during when it's time to make a pick.


## Features

Features are the inputs the model actually sees when it makes a prediction. Things like a player's recent form, their cost, or the strength of their opponent. The more informative the features, the better the model can do its job.

One thing you have to be careful about is data leakage. A leak happens when a feature contains information that wouldn't actually be available at prediction time. In other words, it cheats. For example, `xG_match` tells us how many expected goals a player generated in a specific match. If we trained on that, the model would learn to rely on something it can never know in advance. When you then ask it to predict next week's scores, it would be flying blind on its most important input. Leaky models look great on paper and fall apart in production.

So, after dropping our leakage columns (`xG_match`, `xA_match`, `goals_match`, etc) and identifier columns, the model trains on 26 features:

| Feature | Source |
|---|---|
| `cost` | Rolling value/10 at that GW |
| `ppg` | Cumulative points/games before the GW |
| `form` | Rolling 5-GW points average |
| `ict_index_form` | Rolling 5-GW ICT index average |
| `is_home` | Fixture |
| `opponent_team_id` | Fixture |
| `elo_for`, `elo_against` | ClubElo historical snapshots |
| `is_penalty_taker`, `is_set_piece_taker` | FPL elements flags |
| `chance_of_playing`, `injury_confidence` | FPL + injuries CSV |
| `position_*` | One-hot encoded (GKP/DEF/MID/FWD) |
| `selected_pct`, `price_rise_probability` | Null for historical rows |
| `defensive_contribution_per90`, etc. | Null for historical rows |
| `clean_sheet_prob`, `goalscorer_prob` | Null for historical rows (odds pipeline broken) |

<br>

The honest version of that table: 14 features have real values in training data, and 12 are null for every historical row. Those 12 null features are exactly the signals I was most excited about: `selected_pct` and `price_rise_probability` for ownership, `clean_sheet_prob` and `goalscorer_prob` for bookmaker odds, `chance_of_playing` and `injury_confidence` for injury signals. The issue is that they're difficult to find historical data for (at least for free). They exist in the schema, they show up in next-GW prediction rows, but they've never seen a training example. The model has no idea how to use them yet.

The time split is worth calling out too. Rather than a random 70/15/15, the data is sorted chronologically by gameweek and split in order: the first 70% of gameweeks form the training set, the next 15% validation, the last 15% test. This matters because FPL data has a temporal structure. A random split would let early-season rows bleed into the test set, making the model look better than it is. The time-series split is more honest.


## Training

Remember the sweets in a jar? This is where that actually happens.

The model doesn’t start from scratch. It opens with a base guess, usually just the average points of every player in the training set. Something like: "I’ll say 2 points for everyone." Wildly wrong, obviously, but it’s a starting point.

From there, it asks a simple question: "What did I miss?" It builds a small decision tree to predict its own mistakes. Something like: if a player played more than 60 minutes against a weak side, I was probably too low. It adds that correction to its base guess, then immediately looks at the new gap. Then it builds another tree to fix that. Then another. Then another.

Each tree is deliberately shallow and deliberately humble — scaled down by a learning rate so no single rule gets to throw its weight around. The idea is patience. Small steps, stacked up, compounded over hundreds of rounds. By tree 500, the model has assembled something genuinely complex out of individually simple pieces: a map of subtle interactions that no one wrote down, that no one designed, that just emerged from enough reps. The kind of thing where it quietly learned that away games against elite defences are where mid-table attackers go to die.


## Results

There are two main things we're looking for: feature weights and the MAE (mean absolute error). During training, models adjust how much weight they put into each feature to guide their predictions. The MAE then tells us how far off those predictions were.

Here's where our model puts its weight:

| Rank | Feature | Gain |
|---|---|---|
| 1 | `ict_index_form` | 154.8 |
| 2 | `form` | 67.5 |
| 3 | `ppg` | 56.1 |
| 4 | `is_home` | 48.6 |
| 5 | `position_DEF` | 47.2 |
| 6 | `elo_against` | 46.6 |
| 7 | `is_set_piece_taker` | 45.7 |
| 8 | `position_MID` | 44.8 |
| 9 | `elo_for` | 43.3 |
| 10 | `position_GKP` | 39.5 |

<br>

**ICT index is #1.** `form` and `ppg` come in at 2 and 3. Turns out the folks at FPL know what they're doing. ICT index is our strongest single signal, which makes sense: recent attacking involvement helps predict future points.

**Position is pulling its weight.** `position_DEF`, `position_MID`, and `position_GKP` all appear in the top 10. The model is picking up on the fact that defenders and keepers score very differently from midfielders and forwards — and it's learned to care about it.

**`is_home` and `is_set_piece_taker` register strongly.** Home advantage and set piece responsibility are well-established FPL factors, so seeing them surface is a sign the model is healthy.

**Elo ratings matter, but cost doesn't.** `elo_against` and `elo_for` both sit in the top 10, so opponent and team strength remain reliable signals. Surprisingly, `cost` does not appear on the table. This suggests that once you account for form, position, and fixture context, price isn't adding much on top.

So those are the levers it's pulling. But how well does it actually perform? The model lands at **Val MAE: 1.625** and **Test MAE: 1.559** — on average, about one and a half points off the real result. Here's how that compares to two simple baselines:

| Predictor | Val MAE | Test MAE |
|---|---|---|
| Predict global mean (1.87 pts) | 1.934 | 1.902 |
| Predict per-player historical mean | 1.677 | 1.702 |
| XGBoost model | 1.625 | 1.559 |

<br>

Good news and bad news. The good news is, it beat both baselines. The bad news is it only barely beat them — particularly the per-player mean, which requires zero ML. The root cause is the points distribution: 80% of rows score 0–2 pts, only 10% score 6+. Any model learns to predict low for everyone, which gives decent MAE but is useless for FPL where you only care about identifying the 10% who have a big week. The fix? Better features.

So, I just spent the last 3 evenings building my robot to tell me exactly what the pub down the street has known forever. Great.


## What Comes Next

The gaps are real. The model still doesn't know if a player is injured or if their team is likely to keep a clean sheet, two things any FPL manager checks before making a transfer. Until that historical data gets backfilled, it's making decisions with one hand tied behind its back.

But I still feel like a proud parent. This is what football dads must feel on the touchline, watching their kid do something clever for the first time — a little surprise, a little pride, and the quiet hope that one day they might teach you a thing or two.
