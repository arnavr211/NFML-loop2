# NFML Loop — build prompts

Use a fresh Codex chat for each step, in the new repo with AGENTS.md at its root. Move to the next step only when the current one's tests pass.

---

## Step 0 — Housekeeping (you, in the old repo, no Codex)

The useful kernels (`local_app/context.py`, `local_app/regression.py`) are uncommitted on `codex/2026-week1-ingest`. Commit them before anything else so they can't be lost:

```bash
cd ~/Desktop/NFML/nfl-discovery-engine
git checkout codex/2026-week1-ingest
git add -A
git commit -m "Local app and enrichment work before loop split"
git push -u origin codex/2026-week1-ingest
```

The `loop-v2` branch is no longer needed. Delete it if you created it:

```bash
git branch -D loop-v2
git push origin --delete loop-v2
```

Create the new repo, e.g. `nfml-loop`, beside the old one, and put AGENTS.md at its root.

---

## Step 1 — Scaffold and data layer

```
Read AGENTS.md first; it governs this repo.

Set up the project and the data layer:

1. pyproject.toml with uv, Python 3.11+, the allowed dependencies from
   AGENTS.md, plus pytest and ruff. Commit uv.lock. Root-anchored
   .gitignore for data/.
2. loop.yaml with: q (0.10), lead_threshold (0.10),
   small_cluster_threshold (20), bootstrap_resamples (999), seed,
   batch_size (20). Load it into a plain dataclass and compute a config
   hash.
3. src/nfml_loop/data.py:
   - pull_season(season): nflreadpy.load_pbp, write
     data/pbp/{season}.parquet and its .json metadata (see AGENTS.md).
   - load_plays(seasons): read cached parquet files, concatenate with
     pl.concat(how="diagonal_relaxed"), validate unique
     (game_id, play_id), and raise an error naming any requested season
     that is missing. No filtering.
   - refresh(current_season): re-pull the current season whole.
4. Pull 2016-2025 and the current 2026 season. Use the latest nflreadpy.
   If it rejects the in-progress 2026 season, report the error and stop;
   do not work around it.
5. Tests on a small synthetic parquet: concatenation, duplicate-key
   rejection, missing-season error.

Report: rows, games, and max week per season.
```

---

## Step 2 — Statistics core and calibration

```
Read AGENTS.md first.

Build src/nfml_loop/stats.py. Port these from
~/Desktop/NFML/nfl-discovery-engine, keeping a source-file comment on
each, as plain functions taking arrays/frames and ordinary arguments
(no MethodRequest, AnalysisDataset, or registry):
- _cluster_bootstrap from analysis/methods/implementations.py
- _benjamini_hochberg from analysis/validation/gate.py, changed so a
  None p-value counts as p = 1 instead of being dropped
- the game-level sign-flip logic from local_app/regression.py, with the
  number of flips as a parameter instead of hardcoded 999

Then write:
1. compare(df, group_col, outcome, controls, cluster_col="game_id"):
   OLS of outcome on C(group) + controls via statsmodels, with standard
   errors clustered by game. Returns estimate, 90% interval, raw p,
   play count, game count, and which inference method was used. When
   either group has fewer than small_cluster_threshold games, use the
   sign-flip test for p instead, and record that.
2. opponent_baseline(df, cutoff_week=None): port add_opponent_context
   from local_app/context.py, adding cutoff_week. When set, the
   baseline uses only plays from weeks before the cutoff.
3. label(results, q): applies BH across the run and assigns flagged,
   lead, or null per AGENTS.md.

Tests:
- Synthetic known signal is detected; synthetic known null is not
  flagged more than about q of the time across 200 simulations.
- Clustered SEs are wider than naive SEs on synthetic data with
  within-game correlation.
- opponent_baseline with a cutoff never uses later weeks.

Calibration suite (tests/test_calibration.py, marked to run on real
data): on 2016-2025, early-down dropbacks have higher EPA than
early-down runs, controlling for down, distance, and field position.
If this fails, stop and report; the stack cannot be trusted.
```

---

## Step 3 — Specs, runs, and the ledger

```
Read AGENTS.md first.

Build the run pipeline:

1. src/nfml_loop/spec.py: a pydantic Spec model with question,
   rationale, question_type, population_a, population_b (column-to-
   value filters), outcome, controls, method ("compare"), and
   min_games. Validate that every referenced column exists in the data.
2. src/nfml_loop/ledger.py: DuckDB at data/ledger.duckdb with the tables
   in AGENTS.md. Append-only.
3. src/nfml_loop/run.py: run_batch(specs) loads the needed seasons,
   builds each population, labels plays as group A or B, calls
   compare(), catches failures (logged with p = 1), applies BH across
   the batch, writes everything to the ledger, and writes a markdown
   report to reports/{run_id}.md. The report leads with flagged
   results, then leads, then a count of nulls and failures. Each result
   gets one plain sentence written from its numbers by a template, not
   an LLM.
4. CLI: `nfml-loop run --specs path/to/specs.json`.
5. Write 10 example specs in examples/specs.json covering different
   question types (situational split, cross-season team comparison,
   after-event sequence, home/away, quarter). Run them.

Tests: a failed spec still counts in the BH family; the ledger never
rewrites rows; a spec referencing a missing column fails validation.
```

---

## Step 4 — Forward confirmation and the scoreboard

```
Read AGENTS.md first.

Build src/nfml_loop/confirm.py:

1. For every flagged or lead result whose data ended at week W, rerun
   the same spec on plays from weeks after W only, with any baseline
   computed with cutoff_week = W + 1. Record in the confirmations
   table: confirmed (same direction and p <= 0.10), faded (opposite
   direction or estimate within half its original interval of zero),
   or pending (fewer than min_games new games).
2. Results that compare fixed past seasons cannot be confirmed forward.
   Mark their question type as descriptive and exclude them from
   forward confirmation.
3. Scoreboard: per question_type and method, count tested, flagged,
   confirmed, faded, and pending, and compute the confirmation rate.
4. CLI: `nfml-loop confirm` and `nfml-loop scoreboard`.

Tests on synthetic weekly data: a real effect confirms; an effect
injected only in early weeks fades; lookahead is impossible.
```

---

## Step 5 — Backtest over past seasons

```
Read AGENTS.md first.

Build src/nfml_loop/backtest.py to simulate the weekly loop on
2016-2025 before using it live:

1. For each season: run spec batches on weeks 1-9 with every baseline
   cut off at week 10, then run confirm.py on weeks 10-18.
2. Specs come from a template generator for now: crossing question
   types, teams, and situations from a fixed list in loop.yaml. No LLM
   yet.
3. Also run a shuffled-label control: same specs, with group labels
   permuted at the game level. Its flag rate shows the false-flag
   baseline.
4. Write reports/backtest.md: per question type, flag rate, confirmation
   rate, and fade rate, with the shuffled control beside them. Write the
   scoreboard to the ledger as the starting history for the live loop.

Report the five question types with the highest confirmation rate and
the five with the lowest.
```

---

## Step 6 — LLM proposer

```
Read AGENTS.md first.

Build src/nfml_loop/propose.py:

1. Allocation: Thompson sampling over question types, using a
   Beta(confirmed + 1, faded + 1) posterior from the scoreboard. Of
   batch_size slots, reserve 20% for question types with fewer than 5
   tests, so new areas keep getting tried.
2. For each allocated slot, call the Claude API (model name from
   loop.yaml). Give it: the data dictionary of available columns, the
   question type to fill, the scoreboard, the last 30 confirmed and
   faded findings, and the instruction to propose a football-grounded
   hypothesis with a rationale, not to repeat tested specs.
3. Require JSON output matching Spec; reject and log invalid output.
   Skip specs identical to ones already in the ledger.
4. CLI: `nfml-loop cycle` runs refresh, confirm, propose, run_batch,
   and writes the report.

The LLM never sees raw data or computes numbers; it sees only the
column list, scoreboard, and past results.
```

---

## Later

- Weekly scheduling of `nfml-loop cycle`, after a few manual cycles look right.
- An exploratory-code lane for questions specs can't express.
- Reconnecting the directed-analysis toolkit from the old repo to this ledger.
