# Text-to-SQL Model Evaluation — Concept Notes

**Project context:** A read-only text-to-SQL feature for TBS's cricket/streaming website. The business hypothesis is that useful cricket-stat questions encourage meaningful return visits and support the site's advertising business. The immediate technical goal is to choose a model that is accurate, safe, responsive, deployable, and within a monthly cost ceiling.

**Status:** This is an evaluation plan and analysis of the handwritten `text2sql.pdf` plus the prompt and 11-step roadmap supplied with it. It is **not** a benchmark report and does not select a winning model: no fixed golden dataset, live model runs, verified prices, or production measurements were provided.

---

## 1. Executive summary

The overall direction in the notes is sound: use a public leaderboard to discover candidates, then compare a small set on a custom, cricket-specific benchmark using the target schema and a fixed database. The final decision should be based on measured quality and production constraints—not on a leaderboard rank alone.

The most important refinements are:

1. **Make the test data match the product.** The notes identify an IPL Kaggle dataset, apparently IPL 2020–2024, while the product context is Asian Games cricket on TBS. IPL can be a useful prototype, but it is not a sufficient final benchmark unless the production question types, data definitions, team/player names, schema, and query patterns are representative of the Asian Games experience.
2. **Treat correctness and safety as gates.** A model that exceeds the cost ceiling, emits unsafe SQL, or misses the minimum correctness target is not a winner, regardless of its average score. Rank only the models that pass those gates.
3. **Reconcile the cost assumptions before shortlisting.** The notes contain both roughly 30,000 and 50,000 questions per day. Under the handwritten illustrative rates of $10 per million input tokens and $50 per million output tokens, 50,000 daily queries at 400 input and 100 output tokens cost about **$13,500/month**, or **₹12.825 lakh/month** at ₹95/USD. A ₹3 lakh/month ceiling is therefore about 4.3 times lower than this estimate. Prompt caching can reduce repeated input-token cost, but it does not reduce output-token charges. Under those same assumptions, output tokens alone exceed ₹3 lakh/month at either 30,000 or 50,000 queries per day.
4. **Resolve the prompt and example SQL before caching it.** The supplied example's `SUM(total_runs)` may not mean “runs conceded by the bowler” in cricket, and the prompt's example/question/schema ordering is confusing. Caching a flawed example makes it consistently available; it does not make it correct.
5. **Freeze the whole experiment.** Model ID/version, prompt, schema, database snapshot, generation settings, evaluator, and cost rates must be recorded. Otherwise, differences between model runs may be caused by changed inputs rather than the model.
6. **Use the leaderboard as a filter, not the decision-maker.** The notes correctly question direct selection from an online leaderboard. Use it to get to a manageable shortlist of about 5–10 candidates, then run the custom evaluation.

### The proposed high-level flow

```text
Production-relevant data + fixed database snapshot
        ↓
Introspected, documented schema and cricket definitions
        ↓
Expert-reviewed question / SQL / expected-result benchmark
        ↓
Fixed prompt and safety policy
        ↓
Candidate model calls → SQL validation → isolated read-only execution
        ↓
Result comparison + latency, cost, reliability and safety measurements
        ↓
Budget/correctness/safety gates → choose a feasible model → controlled launch
```

---

## 2. Project framing and success criteria

### 2.1 Separate the business goal from the model benchmark

The business hypothesis is that asking useful questions about cricket statistics will make the site more valuable and may increase repeat use, session depth, or ad opportunities. A model benchmark can assess whether the feature answers questions correctly and quickly. **It cannot establish that the feature increases retention or revenue.** That requires a later product experiment, such as a controlled A/B test.

A sensible sequence is:

- **Model evaluation:** correctness, safe behavior, cost, latency, reliability, and deployability.
- **Product pilot:** user task success, question abandonment, repeat usage, satisfaction, return visits/retention, and ad-related guardrails.
- **Business decision:** whether the measured user benefit justifies the operating cost and product trade-offs.

Optimize for *useful* engagement and successful tasks, not time-on-site by itself. Artificial friction could raise time-on-site while making the product worse.

### 2.2 Write the measurable requirements before choosing a model

The PDF identifies six useful requirement headings. Turn them into measurable criteria:

| Requirement in the notes | What to define before the run |
|---|---|
| **Task** | Which questions must be supported? Which tables/fields and SQLite dialect? Is this text-to-SQL only, or must the system also explain/render the result? |
| **Cost** | Confirm the currency and whether the ceiling is **₹3 lakh/month**. Clarify whether it covers only model tokens or also cache writes/storage, retries, infrastructure, taxes, monitoring, and support. Establish daily average and event-day peak query volumes. |
| **Latency** | The notes suggest about 2–3 seconds. State whether this is model-generation latency or total time from submit to displayed result, and whether the target is p50, p95, or both. Interactive products usually need a percentile target, not only an average. |
| **Context window** | Confirm the full prompt fits with room for the generated SQL. The current two-table schema is small, but schema documentation, examples, and a future production schema can make this material. |
| **Deployment** | Confirm API availability, service terms, concurrency/rate limits, region/data handling, uptime, versioning, and whether the provider is approved. “Use an API key” is not a complete deployment plan. |
| **Correctness** | Define the answer-equivalence rules, minimum quality threshold, handling of ambiguity/no-answer cases, and what counts as a safety failure. The notes correctly say correctness matters most. |

### 2.3 Assumptions and decisions still open

The notes are enough to design the evaluation, but the following items should be confirmed before interpreting results:

- Is the final target database Asian Games cricket data, IPL data, or both?
- Are the 30,000 and 50,000 daily-query figures alternative estimates, or does one represent normal traffic and one peak traffic?
- Is the ₹3 lakh cap a hard monthly inference-cost ceiling? Is it inclusive of taxes and non-LLM services?
- Is the latency target 2–3 seconds p95 for the complete user request, or for SQL generation alone?
- Which exact API model IDs do the handwritten names refer to, and are they currently accessible under approved terms?
- Does the “economy rate” business definition use bowler-conceded runs or all `total_runs`? What is the approved tie-break for equal economies?
- Should the initial gold set contain 50 cases, 60 cases, or a smaller 20-case pilot followed by a larger benchmark?
- What should the application do when a question is ambiguous, unsupported, or asks for data outside the user's access rights?

---

## 3. Analyze the supplied prompt and cricket SQL example

### 3.1 Prompt layout and prompt-caching implication

The supplied text contains a system instruction, then a question and its SQL, and then a `USER` section containing the schema. If the question/SQL is intended as a few-shot example, label it explicitly as an example and put the example's schema, question, and answer together in a coherent order. Otherwise, the model can mistake the example for the current request or imitate its semantics when it should not.

A useful arrangement is:

```text
SYSTEM — stable/cacheable prefix
  Role and SQLite output rules
  Safety and schema-use rules
  Cricket/business definitions
  The fixed schema (if it is the same for all requests)
  Clearly delimited, reviewed example(s), if needed

USER — dynamic suffix
  The user's current question
```

The system prompt should not contain the *current* user's question as static text. Only the fixed instructions, stable schema/definitions, and reviewed examples are candidates for a reusable cache prefix. Put the new question after the cacheable prefix.

For the standardized benchmark, it is reasonable to require a single SQL statement for answerable questions. In the actual product, do not force an invented SQL query for an ambiguous, unsupported, or unauthorized question. Give the product a defined clarification/refusal path—such as a structured status outside the SQL executor—and never execute a model's clarification text as SQL.

### 3.2 Important semantic issue in the economy-rate example

The example query calculates:

```sql
6.0 * SUM(total_runs) / legal_ball_count
```

The legal-ball idea is broadly reasonable for the stated rule—exclude wides and no-balls from the ball count—but the numerator needs a domain decision. In standard cricket scorekeeping, a bowler's runs conceded generally include runs off the bat and bowler-attributable extras such as wides/no-balls, but exclude byes and leg-byes. `total_runs` commonly includes all runs on the delivery, including extras that are not charged to the bowler. Therefore, the example can rank bowlers incorrectly if `total_runs` includes byes/leg-byes or other non-bowler runs.

The provided `deliveries` columns do not include separate `wide_runs`, `noball_runs`, `bye_runs`, and `legbye_runs` fields. The evaluator must confirm that the dataset's `extras_type` and `extra_runs` fields are sufficient to reconstruct the approved definition. Do not assume that the dataset always has only one extras category per row without checking its documentation and real records.

The original query also has a nondeterministic result if two bowlers tie on economy. Specify the tie policy in the gold label, and add a deterministic secondary ordering if that is the intended behavior. The question should state whether the calculation covers all seasons/matches in the database; the current form implies that it does.

The following is an **illustrative candidate only** for a dataset where each row's `extras_type` accurately distinguishes bowler-charged wides/no-balls from other extras. It needs review by a cricket-domain expert against the actual data dictionary before becoming a gold answer:

```sql
WITH bowler_totals AS (
    SELECT
        bowler,
        SUM(
            CASE
                WHEN LOWER(COALESCE(extras_type, '')) NOT IN ('wides', 'noballs')
                THEN 1 ELSE 0
            END
        ) AS legal_balls,
        SUM(
            COALESCE(batsman_runs, 0) +
            CASE
                WHEN LOWER(COALESCE(extras_type, '')) IN ('wides', 'noballs')
                THEN COALESCE(extra_runs, 0)
                ELSE 0
            END
        ) AS bowler_runs_conceded
    FROM deliveries
    WHERE bowler IS NOT NULL
    GROUP BY bowler
)
SELECT
    bowler,
    6.0 * bowler_runs_conceded / legal_balls AS economy
FROM bowler_totals
WHERE legal_balls >= 500
ORDER BY economy ASC, legal_balls DESC, bowler COLLATE NOCASE ASC
LIMIT 1;
```

If the approved metric intentionally uses **all team runs** rather than standard bowler-conceded runs, retain `SUM(total_runs)` but explicitly name and document that metric. The model should not be expected to guess the organization's preferred cricket definition.

Other details to check in the example:

- Are the stored values exactly `wides` and `noballs`, or are there variants such as different spelling/case? Normalize and test against observed values.
- Is each row exactly one delivery event for the intended count? Check penalty/dead-ball records and dataset conventions.
- Are empty or null bowler names possible?
- Does “at least 500 legal balls” mean aggregate across every season, or only the currently selected tournament/period?
- Are ties resolved by more legal balls, alphabetic order, or an accepted multi-row result? The choice must be explicit.

### 3.3 The schema needs relationship documentation

The supplied DDL shows `matches` and `deliveries`, but does not declare primary-key/foreign-key constraints. The intended relationship should be stated explicitly for the model and the evaluator:

```text
matches.id = deliveries.match_id
```

Also document the row grain (one match row; one delivery/event row), the meaning of `inning`, the extras vocabulary, and any date/time normalization. Schema text without business definitions can be syntactically clear and still produce semantically wrong SQL.

---

## 4. Detailed 11-step roadmap

The 11 steps below follow the order supplied by the user. They also incorporate the PDF's supporting notes on leaderboards, token costs, caching, human-created gold data, and result matching.

### Step 1 — Filter and load the database

**Purpose:** Create a reproducible, queryable database snapshot that represents the questions the feature is supposed to answer.

**Actions:**

1. Record the source, license/usage permissions, download date, file names, and source version. Keep raw input files unchanged.
2. The notes point to a Kaggle IPL dataset and mention IPL 2020–2024. First verify that the files actually cover those seasons and can be legally used for this purpose.
3. Build the SQLite tables expected by the prompt. If the downloaded CSV column names or row structure differ, document each mapping rather than silently changing meaning.
4. Check row counts, duplicate IDs, missing values, malformed dates/numbers, unexpected extras labels, and `deliveries.match_id` values without a matching `matches.id`.
5. Record the database file hash, build script/version, table counts, and a database snapshot identifier. The same snapshot must be used for every candidate model.
6. Consider indexes for benchmark execution and safe latency testing, for example on `deliveries(match_id)`, `deliveries(bowler)`, and `matches(season, date)`. Do not let one model's tests alter the database.
7. Use a read-only benchmark copy, not a production write connection.

**Critical representativeness check:** IPL is not Asian Games cricket. Differences in tournament, team/player naming, match format, data coverage, and business definitions can change both the questions and the SQL. Use IPL as an initial pipeline/prototype dataset if necessary; before selecting a production model, add a representative Asian Games/TBS schema and data snapshot (subject to data permission), or clearly label the conclusion as an IPL-proxy result.

**Deliverables:** Immutable raw data, a reproducible SQLite build, a data-quality report, and a versioned database snapshot.

**Done when:** A fresh build produces the same schema, counts, validation checks, and database hash, and all benchmark SQL runs against it.

### Step 2 — Extract and document the schema

**Purpose:** Give every model the same accurate view of tables, columns, types, and relationships.

**Actions:**

- Introspect SQLite metadata (`sqlite_master`, `PRAGMA table_info`, and, if present, foreign keys/indexes) rather than manually copying a potentially stale schema.
- Generate the `CREATE TABLE` statements and a concise schema summary. Keep the join path explicit: `matches.id = deliveries.match_id`.
- Add domain definitions for ambiguous fields such as `total_runs`, `extra_runs`, `extras_type`, `is_wicket`, `inning`, and `over`/`ball`.
- Check that the model's schema contains only approved tables and fields; do not expose private or unrelated tables.
- If the schema is small and stable, it can be included in the fixed prompt prefix. For a larger schema, consider only passing relevant tables, but test that schema selection does not omit required joins.

The handwritten note names a `schema-extractor.py`. That is a good artifact to build, but it should produce a versioned schema file and a human-reviewed business glossary—not just raw DDL.

**Deliverables:** A machine-generated schema file, a human-reviewed glossary, and a schema version/hash.

**Done when:** Every candidate receives identical schema text and the stated relationships match the database.

### Step 3 — Build the golden dataset

**Purpose:** Create the trusted, human-reviewed questions and answer expectations against which every model is evaluated.

**Actions:**

1. Gather realistic questions from analysts, cricket/statistics experts, product staff, and—when available—anonymized, approved user-question patterns. Do not assume leaderboard benchmark questions match TBS users.
2. Cover the target intents: simple filters; player/team totals; season/time filters; joins; grouping; rankings/top-N; per-innings comparisons; rates/averages; ties; null/no-result cases; and multi-condition questions.
3. Make each question sufficiently specified. Explicitly annotate ambiguous or unsupported questions rather than treating multiple valid interpretations as model failures.
4. For each answerable question, record a reference SQL query, the expected result on the frozen database, relevant semantic requirements, and an explanation of the intended business meaning.
5. Have a domain expert author or review the SQL and a second reviewer independently validate it. Resolve disagreements before freezing the benchmark. An LLM can assist with drafting, but its output must not become “gold” without human verification.
6. Tag cases by difficulty and intent, and split into development and held-out test sets. Do not use holdout questions as few-shot examples or repeatedly tune the prompt against them.

**The question-count discrepancy in the notes:** The PDF says “30–50 questions” in one place, gives `10 easy + 20 medium + 30 hard` elsewhere (which totals 60), and later mentions making 20 queries. These can be reconciled as separate stages:

- **20-question pilot/smoke set:** debug ingestion, prompt, and evaluator; not enough to declare a production winner.
- **Benchmark v1:** choose a fixed total and write it down. For example, 50 cases could be 10 easy / 20 medium / 20 hard. If the handwritten 10/20/30 distribution is intentional, use 60 and call it 60. Do not report “30–50” while actually scoring 60.
- **Held-out expansion:** grow the representative test set before making a close call between models. A small set is useful for screening but has wide uncertainty; a few percentage points of difference on 20–60 cases may be noise.

A balanced set should include both common questions and harder, high-risk semantics. Hard examples are valuable, but do not let rare edge cases outweigh normal user traffic without an explicit product reason.

**Deliverables:** A versioned question set with reference SQL, expected outputs, labels, review notes, and development/holdout splits.

### Step 4 — Validate the golden dataset

**Purpose:** Ensure the benchmark is correct before using it to judge models.

**Validation checklist:**

- Every reference query parses and runs in the target SQLite version and against the frozen database.
- A human reviewer confirms the result matches the question, not just that SQL executes.
- Validate important calculations independently where feasible (small hand-calculated examples, an alternate query, or a targeted fixture database).
- Confirm expected columns, ordering, row multiplicity, nulls, ties, date ranges, and numeric precision are handled intentionally.
- For questions where order matters (for example, “top five”), compare in order. For questions where order is not requested, compare as a multiset/set according to the product contract; do not accidentally make row ordering a failure.
- Use a documented numeric tolerance for floating point values such as economy rates.
- Check that different SQL formulations that express the same intent can pass by result equivalence. Exact SQL-string equality is not a suitable primary correctness measure.
- Recognize that the same database can conceal a logical bug if the wrong query happens to return the same rows. Human semantic review and, where possible, counterfactual/fixture tests help address this.
- Mark unanswerable/ambiguous cases separately and define whether the expected behavior is a clarification, refusal, or safe fallback.

**Deliverables:** Review sign-off, validation log, and a frozen golden-set version.

### Step 5 — Store the golden dataset as CSV

CSV is a practical portable manifest, as the notes propose. Quote fields using standard CSV escaping; SQL and JSON result cells may contain commas, quotes, or line breaks. Keep the frozen database snapshot and any build scripts alongside the CSV—the CSV alone cannot reproduce the benchmark.

Suggested columns:

```text
case_id,split,question,difficulty,intent,answerable,
golden_sql,expected_columns_json,expected_rows_json,
order_sensitive,numeric_tolerance,semantic_notes,
database_version,reviewer_1,reviewer_2
```

Example shape for a result cell (illustrative only; not a result from the supplied database):

```text
"[{""bowler"":""Example Player"",""economy"":4.21}]"
```

Do not fill expected rows with invented examples. Generate them from the reviewed reference SQL on the frozen database, inspect them, and preserve the database version/hash that produced them. The handwritten PDF names `golden-dataset-generator.py`; it can automate CSV generation after the human-reviewed questions and reference SQL are ready.

**Deliverables:** UTF-8 `golden_dataset.csv`, database snapshot, and a data manifest/version.

### Step 6 — Check the full flow

**Purpose:** Catch plumbing errors before spending money on 5–10 model runs across every benchmark case.

Run a small end-to-end smoke test on a few representative cases:

1. Load the frozen case and schema.
2. Call one candidate through the same adapter the full evaluation will use.
3. Capture the raw model response and token/latency metadata.
4. Parse the output without silently rewriting its meaning.
5. Validate the SQL against the policy.
6. Execute it on a read-only, isolated SQLite snapshot with time/row limits.
7. Compare columns and results to the expected result.
8. Save a result record and confirm that errors are visible and correctly classified.

Also test malformed SQL, markdown/code fences, multiple statements, a write attempt, an ambiguous question, an empty result, timeout, provider rate limit, and a cache miss/hit if caching is enabled. No model output should reach a production database during evaluation.

**Deliverables:** A smoke-test report and a working end-to-end evaluation path.

### Step 7 — Pick candidate models

The notes correctly propose using an LLM leaderboard to filter rather than directly choose a model. The PDF names LLM Stats as one source and lists candidates resembling:

- GPT “5-6 Tera” (handwritten label; verify exact model ID)
- Kimi K3
- Muse Spark 2.1
- Grok 4.5
- Claude Sonnet 5
- MiniMax M3

Treat these as **candidate labels copied from the notes**, not verified current API names, endorsements, or availability claims. The handwriting also includes a leaderboard table with values for an index/score, price, and throughput, but the columns/figures are not reliable enough to use as benchmark evidence. Recheck each exact model ID, provider, date, token price, context limit, throughput/latency, API status, rate limits, and deployment terms on the actual evaluation date.

Shortlist 5–10 models that are plausibly feasible. Include a strong model as an accuracy reference if practical, even if it may ultimately be over budget. Filter on:

- text-to-SQL/reasoning ability and compatible SQLite behavior;
- likely monthly cost at realistic volume, including output, cache, and retries;
- p95 latency and capacity at expected concurrency;
- context capacity for the complete prompt and schema;
- API/access and legal/security requirements;
- service reliability and version stability.

**Why not select the “best” model or the top leaderboard entry directly?** Leaderboards measure particular tasks, prompts, versions, and scoring methods. An overall coding/reasoning score does not guarantee accurate cricket SQL against this schema, reliable handling of domain definitions, safe read-only behavior, low p95 latency, or affordable cost. A very capable model can be a useful upper-bound comparison, but it is not automatically the production choice.

### Step 8 — Build the evaluator

**Purpose:** Score each model with consistent, auditable rules.

Keep these metrics separate rather than collapsing them immediately into one number:

| Metric | Definition / why it matters |
|---|---|
| **Execution success** | Candidate SQL parses and executes successfully under the allowed SQLite environment. Syntax failures/timeouts count as failures, not silently dropped cases. |
| **Result accuracy** | Candidate output matches the expected result under the dataset's order, duplicate, null, string, and numeric-tolerance rules. This is the main task measure, with human review for semantics. |
| **Safety compliance** | No writes, DDL, multiple statements, unauthorized tables/columns, unbounded expensive queries, or policy violations. Treat serious safety failure as a hard fail. |
| **Answerability behavior** | Correctly handles explicitly ambiguous, unsupported, or unanswerable cases via the defined product route. Do not score these as ordinary SQL generation if the expected behavior is clarification. |
| **Latency** | Record time to first token and full generation; separately measure SQL validation/execution and end-to-end time. Report p50 and p95, not only mean. |
| **Cost** | Actual input, output, cached-read, cache-write, and retry usage × the exact model's rates. Also report cost per successful/correct answer. |
| **Reliability** | API errors, timeouts, rate limits, incomplete output, retry rate, and repeated-run variability. |
| **Diagnostic SQL quality** | Syntax, dialect, join correctness, predicate/aggregation behavior, and query complexity. Helpful for explaining failures, but not a substitute for result/semantic correctness. |

A basic result accuracy is:

```text
correct cases / all benchmark cases in scope
```

State the denominator clearly. If a case times out, produces invalid SQL, or returns an incorrect result, it should normally count as incorrect for end-to-end accuracy. Report easy/medium/hard and intent-level results as well as the overall score. Use confidence intervals or bootstrap intervals, and avoid claiming a meaningful winner when scores overlap substantially on a small set.

The PDF's final page appears to have handwritten numbers beside model names (roughly 80, 55, 90, 85, 65), but no metric name, denominator, model version, or run log is attached. Do **not** present those figures as measured accuracy or use them to select a model until their provenance is established.

### Step 9 — Build the orchestrator

**Purpose:** Make provider differences manageable and ensure all models receive comparable inputs.

Create one adapter per provider behind a common interface, conceptually:

```text
generate_sql(model_id, prompt_version, schema_version, question, settings)
    → raw_text, usage, timing, provider_status, request_id
```

The orchestrator should:

- use the same prompt, schema, DB snapshot, generation settings, and case order policy for the first apples-to-apples comparison;
- store exact model ID/version, prompt hash, schema hash, database hash, timestamp, run ID, raw response, and provider request ID;
- record input/output/cache token counts and separate latency phases when available;
- enforce concurrency and provider quotas; classify 429/5xx/timeouts; and make retries explicit and measurable;
- avoid hidden SQL “repair” or manual cleanup that improves one candidate. If a standard cleanup rule (such as removing a surrounding code fence) is allowed, apply it identically and record it;
- support a second, separately labelled phase for provider-specific prompt optimization, if desired. Do not mix optimized prompts for some models with the shared-prompt baseline and call it a fair comparison;
- store secrets in server-side secret management. Never place API keys in browser JavaScript, a public CSV, or source control.

Suggested run artifacts include `model_runs.jsonl` or a results database plus a human-readable `results.csv` and summary report.

### Step 10 — Run all candidate models

**Purpose:** Produce a fair, repeatable comparison.

- Freeze the benchmark, prompt, schema, model IDs, settings, and pricing snapshot before running.
- Run the same held-out cases through each candidate. Randomize or interleave model order where practical so a provider outage, traffic peak, or temporary network condition does not favor one model.
- Use the same temperature/seed policy and output limit. For SQL generation, low randomness is typically desirable; record the actual settings rather than assuming defaults.
- Use the same read-only evaluator and result-comparison code for every model.
- Log all attempts, including failures and retries. Report first-attempt results and production-like retry-adjusted results separately if retries are part of the planned product behavior.
- Run an initial pass once per case, then repeat a subset (or all cases if affordable) to estimate variability and reliability.
- Measure both normal load and a realistic concurrency/peak-load condition. Tokens-per-second or leaderboard throughput alone is not the same as application p95 latency.
- If evaluating prompt caching, measure uncached, cache-write, and cache-hit paths separately. Keep correctness/safety comparisons equivalent and record cache hit rate.

### Step 11 — Make the decision

Do not select a winner by a single opaque “blended” leaderboard number. Use **hard gates first**, then rank the candidates that remain.

1. **Safety gate:** no disallowed SQL execution; any serious violation triggers rejection or remediation and retest.
2. **Quality gate:** meets the predeclared minimum for result/semantic correctness and acceptable performance on hard/domain-specific cases.
3. **Budget gate:** forecast total monthly spend, including cache, output tokens, expected retries, and realistic volume, stays within the confirmed ceiling with headroom for peaks.
4. **Latency/reliability gate:** meets defined p95 end-to-end latency and service reliability requirements.
5. **Deployment gate:** exact endpoint, terms, access, region/privacy, rate limits, and operational support are acceptable.
6. **Choose from the feasible set:** compare accuracy, cost per correct answer, p95, reliability, and operational fit. A Pareto view is useful: prefer models not dominated by another model that is cheaper, more accurate, and faster.
7. **Controlled launch:** canary or limited release, monitor failures/cost/latency, and separately run a user experiment to test the retention hypothesis.

If no model passes, do not choose the least-bad model and hope. Reduce the per-query cost, improve schema/prompt/data, add a deterministic path or routing strategy, renegotiate the budget, or narrow the supported question scope and rerun the evaluation.

---

## 5. Cost model and arithmetic in the notes

### 5.1 Reproduce the handwritten illustrative estimate

The handwritten notes appear to assume:

- approximately **400 input tokens** per request;
- approximately **100 output tokens** per request;
- **$10 per million input tokens**;
- **$50 per million output tokens**;
- **₹95 per USD**;
- a monthly calculation of 30 days.

With those assumptions:

```text
Input per query  = 400 × $10 / 1,000,000 = $0.004
Output per query = 100 × $50 / 1,000,000 = $0.005
Total per query                                = $0.009
```

The $0.009 estimate is internally consistent with those assumed prices and token counts. It is an **illustrative price sheet**, not a current quote: verify the exact model ID, current official input/output rates, cache rates, billing unit, and currency at evaluation time. Token counts must come from the complete actual prompt and model/API usage metadata, not only a third-party token calculator estimate.

| Traffic assumption | Requests/month | Input tokens/month | Output tokens/month | Model cost at the handwritten rates | Approx. INR at ₹95/USD | Compared with ₹3L |
|---:|---:|---:|---:|---:|---:|---:|
| 30,000/day × 30 | 900,000 | 360M | 90M | $8,100 | ₹769,500 (₹7.695L) | 2.57× the ceiling |
| 50,000/day × 30 | 1,500,000 | 600M | 150M | $13,500 | ₹1,282,500 (₹12.825L) | 4.28× the ceiling |

The PDF seems to contain both about 30K and about 50K daily questions. Reconcile those with analytics. Until then, use low/base/peak scenarios rather than selecting one as fact. Event-day traffic may be much higher than a normal average.

### 5.2 A key budget finding: caching input alone cannot solve the stated gap

At 50,000 questions/day, under the same assumptions:

```text
Output-only cost = 150M output tokens × $50/M = $7,500/month
                ≈ ₹712,500/month (₹7.125L)
```

At 30,000 questions/day, output alone is:

```text
90M × $50/M = $4,500/month ≈ ₹427,500/month (₹4.275L)
```

Both output-only figures exceed a ₹3 lakh/month cap. Prompt caching discounts repeated **input** tokens; it does not discount generated SQL/output tokens. Therefore, with these rates and volumes, caching alone cannot bring the spend under ₹3 lakh. You would also need some combination of a lower-output-price model, fewer generated tokens, lower query volume, deterministic/template handling for common queries, or a tested routing strategy. A very short SQL-only output and a strict output-token limit can help, but must not truncate valid SQL.

At 50K/day, the ₹3 lakh cap allows about ₹0.20/query, or $0.002105/query at ₹95/USD. At 30K/day it allows about ₹0.333/query, or $0.003509/query. The illustrative $0.009/query is above both limits.

The notes' estimate that caching might reduce roughly ₹12 lakh to around ₹6 lakh should be treated as an unverified hypothesis. With the illustrative rates, 50K/day input spend is $6,000/month and output spend is $7,500/month. Even if all input tokens were cacheable and input spend fell close to $600/month at a $1/M read rate, output remains $7,500/month, before cache-write charges. The result is still about $8,100/month (roughly ₹7.7L), and real savings are usually lower because the user question is dynamic, cache writes/refreshes cost money, and the entire input may not qualify for caching.

### 5.3 Complete monthly-cost formula

Use the actual candidate's rate card and measured usage. A practical estimate is:

```text
Monthly total = model input cost
              + model output cost
              + cache-write cost
              + cache-read cost
              + retry/fallback cost
              + other service/infrastructure costs
              + applicable tax/FX effects
```

For token charges, a more explicit model is:

```text
C = (uncached_input_tokens × input_rate
   + cache_write_tokens × cache_write_rate
   + cache_read_tokens × cache_read_rate
   + output_tokens × output_rate) / 1,000,000
   + other per-request charges
```

Use the provider's actual usage fields to avoid counting cache tokens twice. Estimate both **cost per request** and **cost per correct/successful answer**. A slightly cheaper model that needs more retries or gives many wrong answers can be more expensive per useful answer.

Do not forget:

- full schema, system prompt, examples, tool/response wrappers, and question tokens;
- actual output length and model-specific tokenizer behavior;
- retries, fallback calls, and multiple-stage routing;
- cache-write refreshes, cache TTL/storage rules, and cache miss rates;
- expected traffic growth and live-event spikes;
- currency conversion changes and taxes if the ceiling is a local-currency budget.

---

## 6. Prompt caching: correct use and limits

The notes' caching idea is relevant because the same instructions and schema may be sent for many questions. At a high level, a provider can retain the attention key/value state for a stable prompt prefix so repeated requests do not need to reprocess that prefix in the same way. In a hosted API, the provider normally manages the KV cache; the application generally controls the stable prefix/cache markers and observes cache usage rather than manipulating KV tensors directly.

### Recommended cacheable content

- stable text-to-SQL instructions and SQLite dialect rules;
- stable `matches`/`deliveries` schema and join relationship;
- reviewed cricket definitions and a small number of high-quality examples;
- fixed safety/output-format rules.

### Keep outside the stable prefix

- the current user question;
- user-specific or permission-specific information unless correctly isolated by cache key/version;
- changing data/schema details that should not be stale;
- secrets or private information that must not be reused across requests.

### What caching can and cannot do

- **Can:** reduce repeated prefix processing/input charges when the provider recognizes a cache hit; sometimes reduce prefill latency.
- **Cannot:** reduce output-token charges; make a wrong schema or incorrect example right; expand the context window; guarantee a cache hit; substitute for a database result cache or a text-to-SQL evaluation.
- **May not apply:** if the stable prefix is below the provider/model's minimum cacheable-token threshold. The notes estimate roughly 392–400 input tokens; that may be below some services' minimums. Verify the exact model/API's requirements rather than assuming caching is active.

The handwritten cache-rate table appears to list a base input rate of $10/M, a 5-minute cache write of $12.50/M, a 1-hour cache write of $20/M, and a cache hit/read of $1/M. These figures are provider/model/date-specific and must be verified. A cache write can cost more than ordinary input, so a low hit rate can make caching neutral or more expensive.

Before relying on the saving, measure:

1. static-prefix token count and cache eligibility;
2. cache hit rate under realistic traffic and request routing;
3. write/refresh frequency for each TTL;
4. cost of cache writes versus reads and ordinary input;
5. effect of prompt/schema version changes on cache reuse;
6. uncached, cache-miss, and cache-hit latency/correctness separately.

Only change the prompt, schema, or examples after the experiment is versioned. A changed prefix may invalidate cache reuse and also change model quality, so record both effects.

---

## 7. Leaderboards, normalization, and the 90/10 scoring idea

The notes suggest: eliminate models over the monthly cost ceiling; normalize ratings and speed to 0–1; then combine roughly 90% coding capability/rating with 10% latency. This is a useful sketch for a ranking table, but it needs two changes:

1. Use **custom text-to-SQL correctness** as the main quality measure, not an unrelated general leaderboard rating.
2. Treat cost, safety, and minimum correctness as **hard gates**. A weighted average can hide an unacceptable failure—for example, a very fast but inaccurate or unsafe model should not win because of its latency score.

After the gates, a transparent tie-break score might be:

```text
score = 0.90 × custom_quality_score
      + 0.10 × latency_utility_score
```

For a latency measure where lower is better, normalize in the correct direction:

```text
latency_utility = (slowest_latency - model_latency)
                  / (slowest_latency - fastest_latency)
```

Alternatively, normalize throughput with higher values mapped to higher scores. Do not use `(value - min)/(max - min)` on raw latency because that would reward slower models. Handle the `max = min` case explicitly. Min-max values also depend on which candidates happen to be in the set; retain raw milliseconds, tokens/second, cost, and accuracy in the report.

An even safer selection order is:

1. reject safety failures;
2. reject models below the predeclared correctness floor;
3. reject models over the forecast budget;
4. reject models over the p95 latency/service requirements;
5. compare feasible candidates on quality, cost per correct answer, reliability, and deployment fit;
6. use the 90/10 score only as a disclosed tie-breaker, not as the whole decision.

A “blended number” from different leaderboard columns is not meaningful unless the source defines the metric and you normalize direction/scale correctly. General leaderboard rank is a screening signal; the fixed custom benchmark is the decision evidence.

---

## 8. Security and production execution guardrails

A system prompt that says “return only SQL” is not a security boundary. Before any generated SQL is executed, use deterministic controls outside the model:

- run through a read-only SQLite connection/snapshot;
- permit exactly one statement and allow only `SELECT` or an approved `WITH … SELECT` form;
- reject `INSERT`, `UPDATE`, `DELETE`, DDL, `ATTACH`, extension loading, unsafe pragmas, and unauthorized tables/columns;
- use a parser/authorizer or equivalent allowlist, not only keyword substring checks;
- impose query time, row-count, and resource limits; return a safe error on timeout;
- limit access to approved fields and enforce user permissions in trusted application code;
- keep provider API keys server-side in a secret manager; never send them to a browser;
- review provider data retention/training terms, data residency, logging and privacy requirements before sending user prompts or data;
- redact or minimize sensitive data in logs; define retention and access controls for evaluation artifacts.

A safe model response can still contain a costly query. Consider query-plan/cost limits and a capped result set, especially if the production database is large. Evaluation should use a dedicated sandboxed copy and never connect a candidate directly to the production database.

---

## 9. Reporting and final decision template

For each candidate, report at least:

| Field | Value to record |
|---|---|
| Exact provider/model/version | Exact API identifier, not a shorthand label |
| Prompt/schema/database versions | Hash or immutable version for each |
| Benchmark size and split | Include denominator and difficulty distribution |
| Result accuracy | Overall and per difficulty/intent, plus interval if possible |
| Execution success | Parse/run success; classify errors/timeouts |
| Safety | Violations and policy-test results |
| Latency | p50/p95 generation and end-to-end, under stated load |
| Cost | Mean/median/p95 per request, monthly scenarios, cost per correct answer |
| Token/cache usage | Input, output, cache writes, cache reads, hit rate |
| Reliability | Provider error rate, retries, incomplete outputs, variance |
| Decision | Pass/fail for each hard gate, then reason for selection/rejection |

A final recommendation should be framed as: **“Model X is the best feasible candidate for this frozen benchmark and these measured assumptions,”** not “Model X is universally the best model.” Re-run the evaluation when the schema, prompt, provider/model version, price sheet, or production question mix changes.

---

## 10. Recommended next actions

1. Confirm the true target dataset/schema and whether IPL is only a prototype for Asian Games cricket.
2. Confirm the 30K/50K traffic estimate, ₹3L currency/coverage, and the 2–3 second latency definition.
3. Review the economy-rate definition and the extras mapping with a cricket data expert; fix the few-shot example before using it.
4. Build and hash the SQLite snapshot; extract and review schema/relationship/domain documentation.
5. Create a 20-question smoke set, then freeze either a 50-case or 60-case reviewed benchmark with a held-out portion.
6. Verify exact candidate model IDs and current official token/cache pricing; calculate realistic low/base/peak monthly cost before full model runs.
7. Implement the same safe SQL-validation/execution/evaluation path for every candidate, then run the shortlisted models and publish the scorecard.
8. Select a model only after safety, correctness, budget, latency, and deployment gates; validate user retention separately through a controlled product test.
