# AGENTS.md — NFML Loop

## Purpose

A research loop over NFL play-by-play data. Each cycle:

1. An LLM proposes football hypotheses as structured specs.
2. Code tests them.
3. Every result is logged.
4. Earlier findings are retested on newer data.
5. The record of which question types confirm guides the next proposals.

The goal is to surface possibly significant, football-meaningful relationships, and to learn over time which questions are worth asking. Speed to a useful result matters more than infrastructure.

## Guardrails (the complete list)

These are the only required safeguards. Do not add others unless asked.

1. **Code computes every number.** The LLM proposes specs and writes interpretations from computed results only. It never states a number that is not in a result.
2. **The play is the unit of analysis; uncertainty is clustered by `game_id`.** Never report standard errors that treat plays as independent. When a comparison has fewer than 20 games, use the game-level sign-flip test and say so in the result.
3. **Multiple testing.** Apply Benjamini-Hochberg across every test in a run. Default q = 0.10, set in `loop.yaml`. A test that fails to run enters the family with p = 1; the family never shrinks.
4. **Labels.** `flagged`: BH-adjusted p ≤ q. `lead`: raw p ≤ 0.10 but not flagged. `null`: everything else. Only a forward test can set `confirmed` or `faded`.
5. **Log every test** in the ledger, including nulls and failures. Never delete or edit past ledger rows; append corrections.
6. **Point-in-time features.** Any baseline used in a forward test (opponent strength, league averages) is computed only from plays before the cutoff week. No lookahead.
7. **Player identity uses GSIS IDs only.** Names are display fields. Unresolved IDs stay null; never fill them from names.
8. **Missing values never become zero.**

## Out of scope unless explicitly requested

- Immutable snapshot stores, manifests, semantic fingerprints, supersession links, revision or removal detection
- Frozen or immutable configuration wrappers
- Evidence gates that block computation; finality or availability state machines
- Canonical table contracts beyond what a feature needs
- UI, directed-analysis toolkit, scheduling, cloud deployment

## Data

- Source: `nflreadpy.load_pbp`.
- Cache one file per season: `data/pbp/{season}.parquet` plus `data/pbp/{season}.json` holding `pulled_at`, nflreadpy version, row count, game count, max week, and sha256.
- Past seasons are pulled once. The current season is re-pulled whole on each run.
- Keep all raw columns, including player ID columns. Population filters live in specs, never in the loader.
- Use Polars for data work. Convert to pandas only at the statsmodels boundary.

## Configuration

`loop.yaml` holds q, thresholds, batch budgets, seeds, and model names. It is loaded into a plain dataclass. Each run records a hash of the loaded config.

## Ledger

DuckDB at `data/ledger.duckdb`. Tables: `runs`, `specs`, `results`, `confirmations`, `scoreboard`.

Each result row records: spec ID, run ID, seasons and max week used, data sha256s, method, estimate, interval, raw p, BH-adjusted p, play count, game count, label, code commit, and seed.

## Working rules

- Make small changes with focused tests. Test on synthetic data with a known signal and a known null.
- No multi-round review loops. Run one review only when asked.
- Allowed dependencies: polars, numpy, scipy, statsmodels, pandas, pyarrow, duckdb, pyyaml, pydantic, anthropic, nflreadpy. Ask before adding others.
- Code ported from `nfl-discovery-engine` keeps a comment at the top naming its source file.
- A change is done when its tests pass and the calibration suite still passes.
