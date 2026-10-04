# LLM SQL Eval — Cricket Text-to-SQL Benchmark

This repository is an **experimental evaluation pipeline** for comparing language models on text-to-SQL questions over IPL cricket data. It builds a local SQLite database, provides its schema to candidate models through OpenRouter, executes generated SQL against the local database, and compares the resulting rows with stored reference results.

> **Scope and status:** This checkout evaluates an IPL 2021–2024 dataset. It is not a production service and is not, by itself, an evaluation on TBS/Asian Games data. The checked-in benchmark has 20 questions, all labelled `hard` or `brutal`. The checked-in `eval_results.csv` is a small benchmark snapshot, not a production model recommendation. Review the limitations below before using its scores to make a deployment decision.

## What it does

1. Filters IPL match and ball-by-ball CSV data to match dates in 2021–2024 and writes `ipl_2021_2024.db`.
2. Extracts the SQLite schema into `schema.sql` for the model prompt.
3. Keeps a 20-question gold set and reference results in `golden_dataset.csv`.
4. Calls five configured OpenRouter model slugs with the same basic text-to-SQL prompt.
5. Cleans the model response, executes the SQL locally, and compares the result to the stored gold result.
6. Writes per-question outcomes and generated SQL to `eval_results.csv`.

The runner uses `temperature=0` and `max_tokens=800`. Each normal run attempts 5 models × 20 questions = **100 model calls**, excluding retries or manual tests.

## Repository layout

```text
.
├── data/
│   ├── matches.csv                  # Source match data
│   └── deliveries.csv               # Source delivery data
├── ipl_2021_2024.db                 # Built SQLite database
├── schema.sql                       # Prompt schema generated from the DB
├── db.py                            # Filter CSVs and build SQLite database
├── schema_extractor.py              # Inspect and export schema; optional rename experiment
├── golden_dataset_generator.py     # 20 reference questions/queries; validates query execution
├── golden_dataset.csv               # Gold questions and expected result JSON; read by main.py
├── make_golden_dataset.py           # Writes optional golden_hard.csv from the generator
├── evaluator.py                     # Result comparison logic
├── model_openrouter_slug.py         # Candidate model display names and OpenRouter slugs
├── first_test.py                    # One-question API smoke test (separate model config)
├── main.py                          # Main model-evaluation runner
└── eval_results.csv                 # Checked-in result snapshot; overwritten by main.py
```

All scripts expect to be run from the repository root because paths are relative to the current working directory.

## Data and schema

`db.py` reads `data/matches.csv` and `data/deliveries.csv`. It parses `matches.date`, selects rows whose parsed calendar year is one of **2021, 2022, 2023, or 2024**, and retains deliveries whose `match_id` belongs to one of those selected matches. The filter uses the date, not the text in `season`, because the source `season` values may vary in format.

The checked-in database contains:

- **279 matches**
- **67,303 delivery rows**
- Seasons represented: 2021 (60 matches), 2022 (74), 2023 (74), and 2024 (71)

The intended join is:

```sql
matches.id = deliveries.match_id
```

The tables currently have no declared primary-key or foreign-key constraints. `db.py` reports orphan deliveries as a sanity check and creates indexes on `deliveries.match_id` and `matches.id`.

The code comments identify the source as a Kaggle IPL complete dataset, but this repository does not document a canonical source URL or license. Verify the source's terms and permissions before redistributing the files or using them in a commercial product. The IPL data is a useful prototype, but it does not establish performance on Asian Games cricket data, a TBS production schema, or actual website questions.

## Requirements

- Python 3.10+ is recommended.
- An OpenRouter API key is required for `first_test.py` and `main.py`.
- This repository does not currently include a `requirements.txt` or lock file.

Install the dependencies from the repository root:

```bash
python -m venv .venv

# macOS / Linux
source .venv/bin/activate

# Windows PowerShell alternative:
# .venv\Scripts\Activate.ps1

python -m pip install --upgrade pip
python -m pip install pandas python-dotenv langchain-openrouter
```

Create a `.env` file in the repository root:

```dotenv
OPENROUTER_API_KEY=your_openrouter_api_key
```

`.env` is ignored by Git. Do not commit or share API keys. The exact model slugs in `model_openrouter_slug.py` must be available to your OpenRouter account and may need updating; model identifiers, availability, limits, and prices change over time.

## How to run

### 1. Rebuild the SQLite database (optional)

The repository already contains `ipl_2021_2024.db`. To recreate it from the CSVs:

```bash
python db.py
```

This writes/overwrites `ipl_2021_2024.db` in the repository root and prints match/delivery counts, orphan count, and date range. Back up the database first if you have made local changes to it.

### 2. Extract the schema

```bash
python schema_extractor.py
```

By default, this prints table DDL, column information, row counts and sample rows, and writes `schema.sql`. The optional `RENAME` experiment is disabled by default. If enabled, it creates a separate renamed database/schema; that renamed setup is **not wired into `main.py` or the current gold SQL**, so it should not be used for a normal evaluation without adapting the rest of the pipeline.

### 3. Validate the reference SQL

```bash
python golden_dataset_generator.py
```

This runs the 20 SQL statements defined in `golden_dataset_generator.py` against `ipl_2021_2024.db` and reports whether each query executes. This checks executability, **not** whether the natural-language question and reference SQL express the correct cricket/business semantics.

### 4. Optionally write `golden_hard.csv`

```bash
python make_golden_dataset.py
```

Despite the comments inside the script referring to other script names, the command above is the actual file in this repository. It writes `golden_hard.csv` from the 20-query generator. **`main.py` does not read this file.** The main runner reads the checked-in `golden_dataset.csv`.

### 5. Make a one-question API smoke test

```bash
python first_test.py
```

This sends one question—“Who scored the most runs in 2024?”—using the separately hard-coded OpenRouter model `openai/gpt-4o`, prints the raw response, and does not execute or score it. This is separate from the five-model list used by `main.py`.

### 6. Run the full evaluation

```bash
python main.py
```

The runner reads `schema.sql`, `golden_dataset.csv`, `ipl_2021_2024.db`, and `MODELS` from `model_openrouter_slug.py`. It calls each configured model for each gold question, executes generated SQL locally, prints a scoreboard, and writes detailed rows to `eval_results.csv`.

> **Cost warning:** Running `main.py` makes paid model requests. Check current provider pricing, account limits, model availability, and expected token usage first. The current runner does not estimate or enforce a monthly cost ceiling.

`eval_results.csv` is opened for writing, so a run **overwrites the checked-in result file**. Copy or rename it first if you want to preserve the existing snapshot.

## Candidate models

The current `model_openrouter_slug.py` configures these five display-name/slug pairs:

| Display name    | OpenRouter slug in this checkout |
| --------------- | -------------------------------- |
| GPT-5.6 Terra   | `openai/gpt-5.6-terra-pro`       |
| Kimi K3         | `moonshotai/kimi-k3`             |
| Grok 4.5        | `x-ai/grok-4.5`                  |
| Claude Sonnet 5 | `anthropic/claude-sonnet-5`      |
| MiniMax M3      | `minimax/minimax-m3`             |

These are the strings configured in the code, not a guarantee that the endpoints are currently valid, available, or priced as expected. Verify them before running. This project currently selects candidates by a manually configured list; it does not automatically query a leaderboard or use a cost-based filtering stage.

## Benchmark format

`golden_dataset.csv` has 20 rows and these columns:

| Column             | Meaning                                                                                                                                                         |
| ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`               | Question identifier                                                                                                                                             |
| `difficulty`       | `hard` or `brutal` in this set                                                                                                                                  |
| `question`         | Natural-language question sent to the model                                                                                                                     |
| `gold_sql`         | Reference SQL text; retained in the CSV, but the main scoring loop compares against the stored result JSON rather than re-running this field for each candidate |
| `gold_result_json` | JSON object containing expected `columns` and `rows`                                                                                                            |
| `n_rows`           | Expected number of result rows                                                                                                                                  |
| `order_sensitive`  | Whether result row order is significant; only question 12 is marked `TRUE` in this checkout                                                                     |

The checked-in set contains **8 hard** and **12 brutal** questions. There are no easy or medium questions, no explicit train/dev/holdout split, and no built-in unanswerable/clarification cases. Treat it as a small, challenging prototype set—not a statistically strong estimate of production accuracy.

## How scoring works

`main.py` performs the following steps for each question and model:

1. Builds a prompt with a fixed system instruction and `schema.sql` plus the question in the user message.
2. Calls the model and applies `clean_sql()` to strip common Markdown fences or leading text.
3. Runs the resulting SQL using `pandas.read_sql_query()` against `ipl_2021_2024.db`.
4. Reconstructs the expected DataFrame from `gold_result_json` and calls `evaluate_one()`.
5. Writes `model`, question ID, difficulty, correctness, reason, and cleaned generated SQL to `eval_results.csv`.

`evaluator.py` compares **result values**, not exact SQL strings. It ignores column labels. When the two outputs have the same number of columns, it compares canonicalized row values, normally ignoring row order. Numeric values are rounded to four decimal places for comparison. When `order_sensitive=True`, row order is retained.

The evaluator also has a permissive rule for a one-column difference: it converts cells within each row to sets and accepts the smaller row's values as a subset of the larger row's values. This is intended to allow a name-only result to match a name-plus-count result, but it applies generically and can accept outputs that do not satisfy the question's requested projection. Set conversion can also lose duplicate values within a row. Therefore the current score is best described as **the repository's value-comparison execution score**, not a strict measure of full semantic answer correctness.

## Checked-in evaluation snapshot

The repository's `eval_results.csv` contains **100 rows**: five models × 20 questions. The results currently stored there are:

| Model           | Correct | Score |    Hard (8) |   Brutal (12) |
| --------------- | ------: | ----: | ----------: | ------------: |
| Grok 4.5        |   18/20 |   90% |  8/8 (100%) | 10/12 (83.3%) |
| Claude Sonnet 5 |   17/20 |   85% |  8/8 (100%) |    9/12 (75%) |
| GPT-5.6 Terra   |   16/20 |   80% | 7/8 (87.5%) |    9/12 (75%) |
| MiniMax M3      |   13/20 |   65% | 7/8 (87.5%) |    6/12 (50%) |
| Kimi K3         |   11/20 |   55% |   6/8 (75%) |  5/12 (41.7%) |

These are transcribed from the checked-in CSV, not a fresh API run. The CSV does not contain a run timestamp, immutable provider model version, prompt/schema/database hashes, token usage, latency, price, cache usage, or retry metadata. Also, the permissive comparison logic and the small, all-hard set can affect the ranking. Do not treat the table alone as a production selection or cost comparison.

## Prompt caching status

The code repeats a stable system instruction and schema across model calls, but this repository does **not** currently implement explicit prompt caching or record cache writes, cache hits, cache TTL, or cache-related costs. Repeated prompt text by itself does not demonstrate that a provider cache is active or that a cost saving occurred.

A future caching experiment should keep stable instructions/schema/examples in a cacheable prefix, place the current question after that prefix, confirm each model/provider's cache requirements and pricing, and measure cache-hit rate and cost separately. The current `eval_results.csv` cannot be used to claim caching savings.

## Important limitations and known issues

### Dataset and business fit

- The data is IPL 2021–2024; it does not establish performance on Asian Games cricket or TBS production data.
- The data provenance/license is not fully documented in this repository. Confirm permissions before external or commercial use.
- The database has no declared primary/foreign-key constraints; the model is given DDL but not an explicit relationship/semantic glossary beyond the field names.
- A 20-question set, with only hard/brutal labels, is too small and skewed to represent a broad user distribution or reliably separate close model scores.

### Gold-query semantics need domain review

Before calling the stored answers “gold,” review the cricket definitions with a domain/data expert. In particular:

- Question 10 calculates bowler economy using `SUM(total_runs)`. Depending on the source's extras representation, `total_runs` may include byes/leg-byes that are not charged to the bowler under standard cricket scoring.
- Question 11 uses `SUM(total_runs)` for bowling average, which has the same potential extras issue.
- Question 16 counts delivery rows as balls faced for batters who were never dismissed; standard balls-faced statistics exclude wides, so this may need an explicit wide-ball exclusion.
- Tie-breaking behavior for questions that request one “top” result should be reviewed and made deterministic where the intended answer requires a single row.

`golden_dataset_generator.py` confirms its 20 SQL statements execute on the bundled DB, but successful execution alone does not validate those semantics.

### SQL execution and safety

- `clean_sql()` is a regex-based cleanup function, not a SQL parser or policy validator.
- Generated SQL is sent to `pandas.read_sql_query()` on a normal SQLite connection. There is no read-only URI, SQL AST/authorizer allowlist, query timeout, row limit, or resource budget in `main.py`.
- Do not connect this runner to a production database or execute arbitrary/untrusted model output outside the isolated local benchmark DB.
- The current runner catches query exceptions but does not save detailed SQL error text to the results CSV.

### Evaluation and observability

- Column names are ignored, and the extra/missing-column rule is permissive; this can overstate answer correctness.
- Values are rounded to four decimal places, with no configurable per-column numeric tolerance.
- The runner does not record generation or end-to-end latency, input/output token counts, price, estimated monthly cost, cache hits/writes, retries, model version, or prompt/schema/database hashes.
- There is no automatic leaderboard filtering, cost-ceiling gate, user-level product/retention experiment, or production safety layer.
- The optional renamed-schema experiment in `schema_extractor.py` is not integrated with the current benchmark.
- The generator/maker comments mention script names that are not present; use the actual command names documented above.

## Recommended next steps

1. Verify the configured OpenRouter model IDs and current prices; record immutable model/version identifiers for each run.
2. Review the gold questions and reference SQL with a cricket-domain expert, especially economy, bowling average, balls faced, and top-result ties.
3. Add representative Asian Games/TBS data and naturalistic questions before using this as a final TBS model benchmark.
4. Expand and stratify the benchmark; reserve a held-out test set and add easy/medium, ambiguity, no-result, and edge cases.
5. Tighten result comparison: validate required columns/aliases, row multiplicity, order rules, and per-column numeric tolerances; remove generic subset acceptance unless a case explicitly permits it.
6. Add a SQL safety layer, read-only database connection, timeout, row/resource limits, and tests for unsafe outputs.
7. Record latency, tokens, cache usage, errors/retries, model and prompt versions, and cost per correct answer. Add a reproducible dependency lock/`requirements.txt`.
8. Evaluate prompt caching as a separate, measured experiment; it is not present in this code today.
9. Treat model selection and retention impact as separate questions: benchmark SQL behavior first, then test user benefit through a controlled product experiment.
