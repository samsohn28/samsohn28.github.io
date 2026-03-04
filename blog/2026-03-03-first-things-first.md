---
layout: post
---

# First Things First

*March 3, 2026*

*Related to: [gaffer\_ai](https://github.com/samsohn28/gaffer_ai)*

Yesterday's post was the backstory of how I got into football, how FPL became a habit, and why I'm building this. Today, the focus shifts to the tool itself: what it does, how it's built, and what I'm learning along the way.

The goal is to build a competitive FPL (Fantasy Premier League) tool. It is a machine learning model designed to compete with the best managers in the game. At its core is a deep neural network trained on a wide range of data: player form, fixture difficulty, home/away splits, and underlying stats like xG (expected goals) and xA (expected assists). But it won't just rely on the FPL API (Application Programming Interface). The model will also pull in weather data, scan press conference quotes for team news, and monitor bookmaker odds for extra signals. Instead of just looking at last week's points, the system combines everything from a manager’s Friday presser to a rain forecast in Manchester to give better recommendations for transfers and captaincy. Eventually, it shouldn't need manual tuning. After each gameweek, it will check its predictions against the actual results, find the errors, and automatically update its feature weights. This creates a self-repairing system that gets more accurate as the season goes on.

Before any of that, I built a simple heuristic bot as a starting point. It pulls live data from the FPL API and scores each player using a weighted formula across a few key metrics. These include recent form, points per game, and availability. It then outputs a recommended squad and captain pick. There is no ML (machine learning) or external signals involved, just deterministic rules. The goal wasn't to win the overall rank. Instead, it was to get the data pipeline working, understand the shape of the problem, and have a functional baseline to benchmark everything that comes next.

The pipeline has three stages: Ingestion, Heuristic, and Optimization.

**Ingestion** fetches and normalizes data from the [FPL API](https://fantasy.premierleague.com/api/bootstrap-static/) and stores it for downstream use. Each player comes back as a JSON (JavaScript Object Notation) object, here's a trimmed example for David Raya:

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

`element_type` maps to position (1 = GK (Goalkeeper)), `now_cost` is in tenths of a million (60 = £6.0m), and `form` is a rolling 4-gameweek average. These are the raw ingredients the heuristic layer works with.

**Heuristic** applies a weighted scoring formula to rank players by position across key metrics. The output is a ranked list of candidates at each position within budget. The formula is:

<div>
$$\text{weighted score} = (0.6 \times \text{points per game}) + (0.4 \times \text{form})$$

$$\text{expected points} = \text{weighted score} \times \text{availability}$$
</div>

Points per game is the season-long average; form is FPL's own 4-gameweek rolling average. Availability converts a player's injury probability (0–100 or null) to a 0–1 multiplier, so a player listed as 75% fit takes a proportional hit to their expected output.

**Optimization** takes those scores and finds the highest-value squad that satisfies all of FPL's constraints. At its core this is a variant of the classic [knapsack problem](https://en.wikipedia.org/wiki/Knapsack_problem): given a set of items (players) each with a value (expected points) and a weight (cost), find the combination that maximizes total value without exceeding a weight limit (£100m). The twist is that FPL adds multiple constraints on top of the budget: exactly 2 GKPs (Goalkeepers), 5 DEFs (Defenders), 5 MIDs (Midfielders), and 3 FWDs (Forwards), plus no more than 3 players from any single club. Since there are too many competing constraints, we can't use a greedy approach. Instead the optimizer uses [PuLP](https://coin-or.github.io/pulp/), a linear programming library, to model every player as a binary decision variable (selected or not) and let a solver find the mathematically optimal 15. A second pass then picks the best starting XI from that squad, applying FPL's formation rules, and assigns the captaincy to the highest-scoring starter. Each stage is modular, which will make it straightforward to swap in the ML model later without rebuilding everything from scratch.

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

Seeing Melker Ellborg as captain makes me feel like a first-time player again. Apparently, it’s not all about form and PPG (points per game). Still, with £99.8m spent and 67.8 projected points for the XI, it is a solid start for a first squad.

Some key takeaways from today. First, if you're like me, starting is the hardest part. This is your sign to just do the thing. Second, Claude Code is really good. If you haven't tried it yet, you should. Last thing, sometimes the daily grind can make you forget what's important. As Roy Kent said, "You did all this because it was fun. So fuck your feelings, fuck your overthinking, fuck all the bullshit. Go back out there and have some fucking fun."


