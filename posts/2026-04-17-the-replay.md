---
layout: post
---

# The replay

*April 17, 2026*

*Related to: [gaffer\_ai](https://github.com/samsohn28/gaffer_ai) (currently private — possible data TOS issues, hoping to open it back up soon)*

The optimizer existed for months before I tested it honestly. It had predictions, a planner, a captain-picking routine. What it didn't have was any proof that any of it worked. A system you can't backtest is just a claim dressed up as an algorithm.

So I built the replay simulator. The idea is simple: run the entire 2025–26 season again from GW1, as if the live system had been making decisions the whole time. No peeking at results. No adjusting with hindsight. Just the model's predictions, the planner's choices, and whatever actually happened.

The final number is 2238 actual points — +654 over the FPL average across 32 gameweeks. That's not the conclusion. That's the start of the honest accounting.


## point in time

The hardest problem in backtesting isn't the algorithm. It's the data.

Lookahead bias is sneaky. A rolling 5-GW ICT average sounds harmless until you realize that for GW3, "5-GW rolling" should only include GW1 and GW2. If you compute it across the full season first and then slice by gameweek, you've given the model information it couldn't have known. Injury flags are worse. If a player was ruled out on Thursday but recovered by Saturday, a feature built naively from end-of-season data might mark him available for a deadline he actually missed.

The snapshot system treats every feature vector as a point-in-time photograph. For each gameweek, the model only sees data that existed before that deadline. Rolling averages are computed incrementally. Injury confidence comes from deadline-day scraper runs, not from whatever the status ended up being. Anything that could only be known after the fact is excluded.

Get this wrong and the backtest is fiction. A model that quietly consumed future information will look brilliant on historical data and fall apart in production. This is the part that's easy to do badly and embarrassing to discover later.


## the planner

Picking a good squad for one gameweek is a solved problem. Linear programming handles it in milliseconds: maximize expected points subject to budget, position, and club constraints.

Planning across multiple gameweeks is harder. Every transfer this week closes off options next week. Bring in a premium midfielder now and you might not have the budget to fix your defense later. The planner looks ahead across a five-gameweek horizon, evaluating transfer sequences rather than individual moves.

The combinatorial explosion is real. If you allow up to 2 transfers per GW and have roughly 12 meaningful combinations at each step, you get 12^5 paths across a five-step horizon — around 250,000 to evaluate. Exhaustive search isn't practical once you add the inner LP solve at each node.

Beam search is the tradeoff. At each step, the planner keeps only the top K paths — the K squads with the highest cumulative expected points — and prunes everything else. You lose the guarantee of finding the global optimum. For FPL planning horizons, the beam is close enough.

The -4 hit penalty lives inside all of this. An extra transfer only makes sense if the expected points gain across the horizon clears 4 pts. The planner accounts for it naturally: paths that take unnecessary hits score worse and die in the beam. You can watch the penalty reshape the search landscape in the output — the planner holds free transfers with genuine patience when the expected gain doesn't justify spending them.


## chip timing

Chips are where the simulator earns its keep.

The naive approach is to fire a chip whenever expected points looks high. The problem is that "looks high" is usually just model noise. Wait for a week that truly looks exceptional and you'll wait forever, because the model doesn't have that kind of confidence. Fire on any week that looks good and you'll have used the Wildcard on GW5 when the real opportunity was GW26.

The reservation price system sets a floor. A Wildcard fires only if the expected gain from rebuilding the squad clears 18 points over a normal transfer week. A Free Hit requires 20. The thresholds are tuned by feel rather than derived from theory, but the logic holds: high-variance, irreversible decisions need a higher bar than low-variance ones.

The decay curve handles hoarding. Every gameweek a chip goes unused, its reservation price drops slightly. By the end of the season, an unused chip fires on almost anything, because holding it forever is worse than using it suboptimally. The curve turns a binary use/don't-use question into a gradual one that resolves itself.

In this replay: Free Hit in GW2, Wildcard in GW3, Bench Boost in GW4, Triple Captain in GW7, Wildcard again in GW26. The GW26 Wildcard had the highest single-GW prediction in the season — 109.2 xPts. The ceiling sim, which cherry-picks chip timing with perfect hindsight, would have arranged some of this differently. That gap is the cost of having to commit before the scores come in.


## the output

Each gameweek produces a decision block: transfers in and out, starting XI with expected and actual points, bench, captain, vice-captain, and a summary of what happened. Thirty-two of them.

The files are generated from JSON and published directly to this site as the [replay\_gw{N}.md](/fpl/) pages. The season table in GW32 shows every week: predicted, actual, average, running edge.

| Total | xPts | Actual | FPL Avg | vs Avg |
|---|---|---|---|---|
| Season | 2130.4 | 2238 | 1584 | +654 |

Publishing matters for a different reason than aesthetics. A backtest that lives in a terminal is a number you're asking someone to trust. Turning it into browsable pages makes the reasoning auditable. GW14 was the worst week of the season — -13 vs average. You can open that file and see exactly why: the captain blanked, the model over-predicted a clean sheet that didn't come, and the planner had no free transfers. No hiding.


## what the numbers mean

+654 over 32 gameweeks is honest signal. But it's bounded.

The model's MAE is around 1.4–1.6 points per player per gameweek. With 15 starters and bench contributions, that's roughly 20 points of expected error per week, in either direction. A +654 edge over 32 weeks is large enough to be meaningful. But the replay can't cleanly separate model quality from decision quality. A better model would have predicted more accurately. A worse planner would have scored less with the same predictions. The two are entangled.

What the replay validates is algorithm correctness. If the planner were making systematically bad decisions — hoarding chips past any rational threshold, taking hits when it shouldn't, captaining the wrong player week after week — it would show up in aggregate. The -13 week in GW14 is in there. So is the +80 in GW20. Neither looks like a failure of the algorithm; they look like variance.

The replay doesn't prove the system is good. It proves it's not secretly bad. That's a lower bar than it sounds, and it's the right bar to start with.
