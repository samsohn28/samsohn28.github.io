---
layout: default
---

# First Things First

*March 3, 2026*

Yesterday's post was mostly about me — how I got into football, how FPL became a habit, and why I'm building this. Consider it the backstory. From here on out, the focus shifts to the tool itself: what it does, how it's built, and what I'm learning along the way.

The vision is a competitive FPL tool — a machine learning model capable of going toe-to-toe with the best managers in the game. At its core will be a deep neural network trained on a rich set of features: player form, fixture difficulty, home and away splits, underlying stats like xG and xA, injury history, rotation risk, price trends, and more. But it won't stop at what the FPL API serves up. The model will also pull in weather data to account for conditions on matchday, scan press conference quotes for hints about team selection and fitness, monitor bookmaker odds and prediction markets for crowd-sourced signals, and track social media trends to catch breaking news before it moves the market. The goal is a model that doesn't just regurgitate last week's points but synthesizes everything — from a manager's post-training presser to a rain forecast in Manchester — to surface the sharpest possible recommendations for transfers, captaincy, and squad structure week after week. And eventually, it shouldn't need me to tune it: after each gameweek it will evaluate its own predictions against the actual results, identify where it went wrong, and automatically recalibrate its feature weights — a self-repairing system that gets sharper as the season progresses.

But, first things first. Before any of that, I built a simple heuristic bot as a starting point. It pulls live data from the FPL API, scores each player using a weighted formula across a handful of key metrics — recent form, fixture difficulty, points per game, and availability — and outputs a recommended squad and captain pick. No ML, no external signals, just deterministic rules. The point wasn't to win the overall. It was to get the data pipeline working, understand the shape of the problem, and have a functional baseline to benchmark everything that comes next.

The pipeline has three stages: Ingestion, Heuristic, and Optimization.

**Ingestion** fetches and normalizes data from the [FPL API](https://fantasy.premierleague.com/api/bootstrap-static/) — player stats, fixture lists, team information — and stores it for downstream use. This is the foundation everything else depends on. Each player comes back as a JSON object; here's a trimmed example for Arsenal's David Raya:

```json
{
  "web_name": "Raya",
  "element_type": 1,
  "now_cost": 60,
  "total_points": 115,
  "points_per_game": "4.0",
  "form": "2.7",
  "expected_goals": "0.00",
  "expected_assists": "0.06",
  "chance_of_playing_next_round": null,
  "minutes": 2610
}
```

`element_type` maps to position (1 = GK), `now_cost` is in tenths of a million (60 = £6.0m), and `form` is a rolling 4-gameweek average. These are the raw ingredients the heuristic layer works with.

**Heuristic** applies a weighted scoring formula to rank players by position across key metrics: recent form and points per game, adjusted for availability. The output is a ranked list of candidates at each position within budget. The formula is:

$$\text{weighted score} = (0.6 \times \text{points per game}) + (0.4 \times \text{form})$$

$$\text{expected points} = \text{weighted score} \times \text{availability}$$

Points per game is the season-long average; form is FPL's own 4-gameweek rolling average. Availability converts a player's injury probability (0–100 or null) to a 0–1 multiplier, so a player listed as 75% fit takes a proportional hit to their expected output.

**Optimization** takes those scores and finds the highest-value squad that satisfies all of FPL's constraints. At its core this is a variant of the classic knapsack problem — given a set of items (players) each with a value (expected points) and a weight (cost), find the combination that maximizes total value without exceeding a weight limit (£100m). The twist is that FPL adds multiple constraints on top of the budget: exactly 2 GKPs, 5 DEFs, 5 MIDs, and 3 FWDs, plus no more than 3 players from any single club. A greedy approach won't cut it — too many competing constraints. Instead the optimizer uses [PuLP](https://coin-or.github.io/pulp/), a linear programming library, to model every player as a binary decision variable (selected or not) and let a solver find the mathematically optimal 15. A second pass then picks the best starting XI from that squad, applying FPL's formation rules, and assigns the captaincy to the highest-scoring starter. Each stage is modular, which will make it straightforward to swap in the ML model later without rebuilding everything from scratch.

Here's what the output looks like when you run it:

```
==========================================================
  GAFFER AI — OPTIMIZED SQUAD
==========================================================

  STARTING XI
   (C)  Ellborg                SUN   £ 4.0m  xPts:  6.80
        B.Fernandes            MUN   £10.0m  xPts:  5.94
        Semenyo                MCI   £ 8.2m  xPts:  5.90
        Palmer                 CHE   £10.6m  xPts:  5.82
        Dewsbury-Hall          EVE   £ 5.0m  xPts:  5.78
        Iwobi                  FUL   £ 6.3m  xPts:  4.72
        Gabriel                ARS   £ 7.1m  xPts:  5.62
        Virgil                 LIV   £ 6.1m  xPts:  5.58
        Hill                   BOU   £ 4.1m  xPts:  4.84
        Ballard                SUN   £ 4.6m  xPts:  4.84
        João Pedro             CHE   £ 7.6m  xPts:  5.16

  BENCH
        Henderson              CRY   £ 5.0m  xPts:  4.64
        Senesi                 BOU   £ 4.9m  xPts:  4.62
        Ekitike                LIV   £ 9.1m  xPts:  4.66
        Thiago                 BRE   £ 7.2m  xPts:  4.30

==========================================================
  Total squad cost:  £99.8m  (budget: £100.0m)
  Starting XI xPts:  67.80  (incl. captain bonus)
  Captain:           Ellborg
==========================================================
```

And the full squad is written to `data/processed/` as a JSON file for downstream use:

```json
{
  "generated_at": "20260303_235036",
  "budget": 100.0,
  "total_cost": 99.8,
  "captain": "Ellborg",
  "starting_xi_expected_points": 67.8,
  "starting_xi": [ ... ],
  "bench": [ ... ]
}
```

Not a bad first squad — £99.8m spent, 67.8 projected points for the XI including the captain bonus.

Two things I learned building this. First, Claude Code is really, really good. I went from nothing to a working pipeline in about an hour. It's not just autocomplete — it understands the problem, suggests the right abstractions, and catches mistakes before they compound. If you haven't tried it, you should. Second, and more sobering: if getting a simple rule-based bot off the ground still required this much thought about data shapes, constraint modeling, and pipeline design, then building a proper ML model is going to be a lot harder than I originally imagined. The heuristic bot is a foundation, but it's also a reality check. The gap between this and what I actually want to build is wide. Good thing I'm not doing it alone.


