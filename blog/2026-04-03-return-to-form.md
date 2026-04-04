---
layout: post
---

# Return to Form

*April 3, 2026*

*Related to: [gaffer\_ai](https://github.com/samsohn28/gaffer_ai)*

A month off. No posts, no commits — just sun, food, and Arsenal quietly going about their business without me worrying about it. When I came back and looked at the codebase, I found a system that could technically produce predictions but needed too much hand-holding: manual CSV files, a broken injuries pipeline, and a prediction engine cheerfully forecasting points for players who were ruled out for the season. Time to get back to work.


## Rebuilding the Ingestion Layer

The old injuries setup was embarrassing. It relied on a manually maintained CSV and a half-finished Gmail scraper. Every time a new injury dropped, someone — me — had to update a spreadsheet. Unacceptable for a system trying to be autonomous.

The replacement is `injuries_scraper.py`, a live web scraper that pulls from premierinjuries.com on every run. No spreadsheets, no stale data. The old `injuries_from_csv.py` and `premier_injuries.py` are gone. The scraper uses `cloudscraper` to get past the site's bot detection, which is a new dependency but means I never have to open a CSV again.

`my_team.py` is new too. It hits the public FPL API to pull your actual current squad: who you own, what chips you have available, how many free transfers you're sitting on, and your full transfer history. The model now knows what it's working with before it makes any recommendations.

`ingest_all.py` runs all of it in one command. One line, all sources, fresh data.


## Idempotency: Stop Running Things Twice

Any serious data pipeline should be idempotent — run it multiple times and you get the same result without breaking anything. The new `_cache.py` module has a shared `is_fresh()` helper that checks each output file's modification time against today's date. All six ingestion scripts now bail out early if the data is current. Pass `--force` if you actually need a refresh. Without this, running a script twice would hammer external APIs and potentially overwrite files mid-pipeline.


## A Single Command for the Processing Pipeline

Getting from raw data to a gold table used to mean running silver and gold scripts in the right order and hoping nothing had broken quietly in between. `process_all.py` sorts this out: silver → gold in sequence, stops if silver fails. The error shows up rather than getting buried in whatever the gold script produces downstream.


## The Optimizer Learns to Read its Own Notes

Two things here.

`transfer_optimizer.py` now saves its recommendations to `data/processed/transfer_recommendations.json` after each run. Before, those suggestions existed only in a terminal window that you'd close and never think about again.

`pick_team.py` — which picks your starting XI and captain from your actual squad — now reads those saved recommendations before it runs. So if the model has told you to bring in a player but you haven't made the transfer yet, it plans around the squad you're about to have rather than the one you currently own. Which is the whole point.


## Fixing the Obvious Bug

The model was predicting 5.22 expected points for Trevoh Chalobah.

Chalobah has a `chance_of_playing` of 0. He is ruled out. Any FPL manager with a pulse would skip straight past him. The model was ignoring the flag entirely and happily projecting him as a viable starter.

The fix in `predict.py` is exactly what you'd expect: players with `chance_of_playing = 0` get zeroed out before predictions are generated. Not clever. Just correct.


## Where Things Stand

There was also a pre-existing `NameError` in `understat_scraper.py` — a reference to an undefined variable called `threshold` — that blew up any run that hit it. Fixed.

The system is in much better shape than it was this morning. Ingestion is live and automated. The pipeline runs in two commands. The optimizer knows what squad you actually own, and it's no longer tipping players who are on a physio table.

First day back and it was mostly plumbing. That's usually how it goes — the preseason friendlies aren't glamorous, but you have to do them. Seven gameweeks left. Let's see what this thing can do.
