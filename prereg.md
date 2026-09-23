# Pre-Registration

**Study:** Does an open-weight LLM, reading only the text of an earnings release, predict
post-earnings abnormal returns better than a systematic earnings-surprise factor?

**Author:** Mohammad Vamik Hussain
**Written:** 2026-09-21
**Status:** DRAFT — pending approval at STOP GATE 2. Once approved this document is FROZEN.
Any later change must be logged in `docs/deviations.md` with a reason and a date.

**Execution order (locked):** prereg frozen → pilot gate → scoring. **No event is scored
before this document is frozen.** No price data is pulled before the data-source terms and
quota are confirmed.

---

## 1. Research question and hypotheses

**H1 (primary).** On earnings events occurring **after** the model's training-data cutoff,
Gemma 4's directional signal derived solely from anonymized release text has a rank
information coefficient (IC) against subsequent abnormal returns that is **not** greater
than that of SUE.
*Direction of interest:* we test whether the LLM adds information over a mechanical factor.
H1 is stated as a null; a null result is a valid and publishable outcome.

**H2 (memorization gap).** Gemma 4's IC is **higher** on pre-cutoff events than on
post-cutoff events, on the matched-company sample (§4.4). A positive gap is evidence of
memorization or leakage rather than reading comprehension.

**H3 (re-identification).** The rate at which the model names the company in its own
reasoning trace, despite anonymization, is **higher** pre-cutoff than post-cutoff.

No hypothesis is contingent on any result. Nulls are reported as they come out.

---

## 2. Model specification (exact; recorded from the installed artifact)

| Property | Value |
|---|---|
| Ollama tag | `gemma4:26b-a4b-it-qat` |
| Ollama model ID | `2dd70431afed` |
| Weight blob | `sha256-4c856523d61d77922dbc0b26753a6bf6208e5d69d80db0c04dcd776832d054c5` |
| Projector blob | `sha256-d8e2de16e17515d9061b23c9a002715f996f9e0c87b93a9354264611bfab9239` |
| Architecture | `gemma4` (Mixture of Experts) |
| Parameters | **25.2B total** (model card: 3.8B active per token) |
| Quantization | **Q4_0** (official quantization-aware-training build) |
| Native context length | 262,144 |
| Embedding length | 2,816 |
| Vision projector | clip, 572.79M parameters |
| Capabilities | completion, vision, tools, **thinking** |
| License | Apache 2.0 |
| Ollama runtime | 0.34.2 (model requires ≥ 0.30.5) |
| GPU | NVIDIA GeForce RTX 4080 SUPER, 16,376 MiB |
| Driver | 591.86 |

**Training-data cutoff.** Quoted verbatim from the official Gemma 4 model card
(`https://ai.google.dev/gemma/docs/core/model_card_4`):

> "Our pre-training dataset is a large-scale, diverse collection of data encompassing a wide
> range of domains and modalities, which includes web documents, code, images, audio, with a
> cutoff date of January 2025."

The model card states one cutoff for all variants; no per-variant cutoff is published.
**Cutoff date used throughout: 2025-01-31.**

**Inference settings (frozen).** Identical for every event in every arm:

| Setting | Value | Reason |
|---|---|---|
| `think` | `true` | Thinking traces are required for the leakage scan (§7.3) |
| `temperature` | `0` | Determinism |
| `seed` | `42` | Determinism |
| `num_ctx` | `32768` | Holds ~99% of releases whole (§4.5) |
| `num_gpu` | `28` | Measured: prevents the context-overflow crash; also fastest for this model |
| `num_predict` | `4000` | Must exceed thinking + answer; measured traces reached 4,506 chars |

**Thinking mode is never mixed between arms.** If any inference setting changes, that arm
restarts from zero and the change is logged in `deviations.md`.

### 2.1 Version lock (binding for the forward test)

A local model's behaviour depends on the runtime and the driver, not only on the weights. A
silent background update would therefore break the prospective claim of §4.8 without changing
a single line of code. To prevent that:

**Recorded on every day the forward test runs**, before any scoring, into
`results/forward_test/environment_log.csv`:

| Field | Source |
|---|---|
| `ollama_version` | `ollama --version` |
| `model_digest` | `ollama list` ID for `gemma4:26b-a4b-it-qat` |
| `weight_blob_sha256` | `ollama show --modelfile` |
| `gpu_name`, `driver_version` | `nvidia-smi` |
| `python_version`, key package versions | interpreter |
| `run_timestamp` | wall clock, with timezone |

**Void rule.** If any of `ollama_version`, `model_digest`, `weight_blob_sha256`, or
`driver_version` differs from the frozen values in §2, then **every forward-test event scored
on or after that change is void as a prospective result**. Void events are:

- retained on disk with their hashes (never deleted),
- labelled `VOID_ENVIRONMENT_CHANGE` with the field that changed and its old and new values,
- excluded from the forward-test statistics,
- reported in the paper as a disclosed count, not omitted.

The change is logged in `deviations.md`. Scoring does not resume as "prospective" until either
the environment is restored to the frozen values or a new, separately labelled forward-test
segment is opened with its own frozen environment.

The daily job **halts and alerts rather than scoring** when it detects a mismatch, so void
events are the exception rather than the accumulating norm.

---

## 3. Data sources (locked)

| Input | Source | Notes |
|---|---|---|
| Index membership | `fja05680/sp500` (MIT), 2,720 change-dates, 1996 → 2026-08-18 | **Point-in-time** |
| Earnings events | SEC EDGAR 8-K Item 2.02 | via `data.sec.gov/submissions` |
| Release text | SEC EDGAR `EX-99*` exhibits | preference order in §5.1 |
| Event timing | SEC EDGAR **acceptance timestamp** | not the filing date |
| Actual EPS | SEC EDGAR XBRL `companyconcept` | carries `filed` dates → point-in-time |
| **Prices (both arms)** | **Tiingo, dividend-adjusted close (`adjClose`)** | **single source** |
| Market benchmark | **SPY from Tiingo, `adjClose`** | total return, same source |
| Consensus EPS | — | **`[MISSING]`** — see §5.4 |
| Cross-check only | market-data API (Polygon) | never mixed into results (§3.1) |

SEC access declares a User-Agent with name and email, paces ≤ 10 requests/second, and caches
every download locally.

### 3.1 One price source, no seam

Tiingo supplies **every** price in both arms, and SPY as the benchmark, all as
dividend-adjusted closes, so all returns are **total returns** everywhere. The market-data
API is used **only** as an independent cross-check and never contributes a number to any
result. Measured agreement on the overlap window 2024-09-20 → 2025-01-31 (8 tickers):

- minimum daily-return correlation **0.996564**
- worst maximum absolute daily-return difference **0.008150**

The residual concentrates in high-dividend names (KO 0.0075, XOM 0.0082) and is consistent
with Tiingo's dividend adjustment versus the cross-check's split-only adjustment. This is
precisely why a single source is mandated rather than a blend.

### 3.2 Trading calendar (committed, cross-checked)

The trading calendar is a **committed data file**, `data/trading_calendar.csv`, not something
reconstructed at analysis time. Every window computation reads it.

| | |
|---|---|
| Source of record | `exchange_calendars` **4.13.2**, exchange `XNYS` |
| Coverage | 1,907 sessions, 2019-06-03 → 2026-12-31 |
| Independent cross-check | Alpaca market-calendar feed, span 2025-01-02 → 2026-09-21 |
| **Agreement** | **430 / 430 sessions — exact, zero disagreement** |

The file records, per session, its source, whether it lies inside the cross-checked span,
whether the independent feed agreed, and whether it was a scheduled early close (3 flagged:
2025-07-03, 2025-11-28, 2025-12-24). Built by `src/build_calendar.py`, which prints any
disagreement rather than resolving it silently.

Outside the cross-checked span XNYS is authoritative and unverified against a second source;
this is stated rather than implied. If the calendar is ever rebuilt, any change in session
count is a deviation and is logged.

---

## 4. Sample

### 4.1 Universe — point-in-time

A company contributes an event only if it was in the S&P 500 **on the acceptance date of
that filing**. Companies later removed, acquired, renamed, or delisted are included for the
period they were members. Today's membership list is never used to select history.

### 4.2 Two survivorship numbers, reported separately

These are different quantities and must never be conflated (a conflation corrected during
Phase 1):

1. **Left the index** — in the index at some point since 2019-06 but not on 2026-08-18:
   **164 of 667 tickers (24.6%)**. This justifies point-in-time membership.
2. **Stopped existing as an SEC filer** — CIK no longer appears in SEC's current ticker map:
   **16 confirmed (2.4%)**, rising to at most **86 (12.9%)** if every unresolved ticker
   (§4.3) proves to be a true exit. This sizes the *pricing* problem.

Both figures appear in the paper with these exact labels.

### 4.3 Resolution rule for unresolved tickers (must precede any IC)

70 tickers are currently `NEEDS_REVIEW` — mostly short symbols (`FB`, `RE`, `FI`, `K`,
`DAY`) that full-text search cannot disambiguate. Each is resolved, **before** the matched
sample is built and **before any IC is computed**, by this rule applied in order:

1. **CIK continuity.** Obtain the CIK from the company's own 8-K filings during its
   membership period. If that CIK appears in SEC's current ticker map under a different
   symbol → **RENAMED**, and the new symbol is recorded.
2. **Filer status.** If the CIK exists but no longer appears in the current map → **REMOVED**
   (stopped filing).
3. **Recycled-symbol guard.** A candidate is rejected if its filing history does not overlap
   the ticker's index-membership window. *Verified necessity:* `LLL` (L3 Technologies →
   L3Harris/`LHX`) resolves by ticker lookup to "JX Luxventure", an unrelated company that
   reused the symbol, and in Tiingo to a GraniteShares ETF.
4. **Manual verification.** Anything still ambiguous is resolved by hand against EDGAR and
   recorded with the evidence used.

Every row of `data/universe_classification.csv` carries a free-text `evidence` field. No
ticker is classified without one. Classification is never inferred from the author's or the
model's recollection.

### 4.4 Eras and the matched sample

| Arm | Window | Events | Unique tickers |
|---|---|---|---|
| **Post-cutoff (headline)** | 2025-02-01 → latest available | **3,401** | 519 |
| Pre-cutoff (contaminated comparison) | 2020-01-01 → 2025-01-31 | **9,209** | 519 |
| Total | | **12,610** | 562 |

**Matched-company sample: 476 tickers** present in the index in *both* eras. **H2 and H3
(the memorization gap) are measured only on this matched set**, so the gap cannot be an
artifact of changing index composition. The full-universe post-cutoff result remains the
headline.

### 4.5 Release length and the trimming rule

Measured over 292 post-cutoff releases (estimated at 3.70 chars/token, calibrated against
six measured prompts spanning 3.47–3.89):

| Percentile | chars | est. tokens |
|---|---|---|
| 50 | 32,282 | 8,724 |
| 90 | 54,307 | 14,677 |
| 95 | 65,945 | 17,822 |
| 99 | 106,919 | 28,897 |
| max | 219,277 | 59,264 |

**Trimming rule (fixed, applied by us, never by the runtime):** the anonymized release is
truncated to the **first 24,000 tokens**, where the token budget is converted to characters
using the **minimum** observed density (3.47 chars/token) so the budget is always
under-filled rather than over-filled. With `num_ctx=32768` and `num_predict=4000` this leaves
~4,700 tokens of headroom at worst-case density, and passes ~97% of releases whole.

*Revised from 28,000 tokens after the Phase 3 pilot — see `docs/deviations.md` D-001. The
original value assumed the mean token density and overflowed `num_ctx` on dense releases,
which crashes Ollama rather than degrading gracefully.*

Every trimmed event is logged with its original and trimmed token counts.
**Sensitivity check (pre-registered):** all primary results are recomputed on the
untrimmed-only subset and reported alongside the full-sample results.

### 4.6 Holdout

The holdout is the **single most recent calendar quarter whose return windows are fully
elapsed at both reported horizons**. It is not examined, plotted, or summarized until Phase 5,
and Phase 5 runs the frozen pipeline **once** with no retuning.

**Frozen boundary (computed from the trading calendar, data date 2026-09-21):**

| | |
|---|---|
| Holdout quarter | **2026Q1** |
| Boundary rule | acceptance date in **2026-01-01 … 2026-03-31** |
| Holdout events | **531** (15.6% of post-cutoff) |
| Window completeness | **100% at +21 and 100% at +61** |
| Holdout span | 2026-01-02 → 2026-03-31 |

Completeness by quarter at the data date (the basis for this choice):

| Quarter | events | complete +21 | complete +61 |
|---|---|---|---|
| 2025Q1 | 335 | 100.0% | 100.0% |
| 2025Q2 | 493 | 100.0% | 100.0% |
| 2025Q3 | 501 | 100.0% | 100.0% |
| 2025Q4 | 516 | 100.0% | 100.0% |
| **2026Q1** | **531** | **100.0%** | **100.0%** |
| 2026Q2 | 518 | 100.0% | 98.3% |
| 2026Q3 | 507 | 92.9% | 0.0% |

### 4.6.1 Three segments (a consequence of the one-quarter holdout)

Because 2026Q1 is the most recent *complete* quarter, 1,025 events in 2026Q2–Q3 fall
chronologically **after** the holdout. They are therefore neither development data (using
them to develop would mean tuning on data later than the holdout) nor part of the one-shot
holdout test. They form an explicit third segment:

| Segment | Events | Role |
|---|---|---|
| 2025Q1 – 2025Q4 | **1,845** | Development / non-holdout (Phase 4) |
| **2026Q1** | **531** | **Holdout — one shot, no retuning (Phase 5)** |
| 2026Q2 – 2026Q3 | **1,025** | **Pending**: enters each horizon only as its windows elapse (§6.1) |
| Oct–Nov 2026 season | TBD | **Prospective forward test (§4.8)** |
| **Total post-cutoff** | **3,401** | |

The pending segment is reported separately and is never merged into the development set. As
its windows elapse it provides additional out-of-sample evidence at each horizon, with the
data date of each computation recorded.

### 4.7 Event inclusion criteria

An event is included only if all hold:

1. Form 8-K containing Item 2.02, accepted while the company was an index member.
2. An `EX-99*` exhibit exists (§5.1). **No 8-K-body fallback** — verified to yield SEC
   cover-page boilerplate (EOG `0000821189-25-000036`).
3. The exhibit passes the **earnings-content gate**: mentions a per-share result, a stated
   period (quarter / fiscal year / full year), and a revenue or income line.
   *Verified necessity:* Item 2.02 over-captures. EOG's was a commodity-hedging disclosure;
   ABBV `0001551152-26-000021` a guidance reconciliation table; TTD `0001193125-26-021804`
   a CFO-appointment notice. All three are correctly excluded, and manual inspection
   confirmed no false exclusions among tested cases.
4. Prices are available to compute at least the shortest return window (§6), or the event is
   handled under §8.
5. One event per company per calendar day; duplicates are dropped keeping the earliest
   acceptance timestamp.

Measured exhibit-extraction failure rate after fixes: **2.7%** (8 of 300). Every excluded
event is written to `results/excluded_events.csv` with its reason. Nothing is dropped
silently.

### 4.8 Prospective forward test (strongest out-of-sample evidence)

Every S&P 500 earnings release in the **October–November 2026 reporting season** (fiscal Q3
2026) is scored on the **frozen configuration of §2**, as releases appear, **before any
return window has completed**. This is the strongest test available because at scoring time
the outcome does not yet exist — not for the model, not for the author, and not in any
dataset.

Protocol, mandatory:

1. **Scored prospectively.** Each release is scored within 72 hours of its EDGAR acceptance
   timestamp, and always before its +21 window has elapsed.
2. **Timestamped at scoring.** Each record stores the scoring wall-clock time alongside the
   release's acceptance timestamp, so the ordering is auditable.
3. **Hashed, committed, and pushed off-machine the same day.** After each scoring batch the
   raw outputs (full thinking trace and answer) are written to disk, a **SHA-256 hash of the
   batch file** is recorded in `results/forward_test/hashes.csv` with its timestamp, and the
   hash file is committed **and pushed to a private remote repository the same day**. The
   remote commit URL is recorded in `results/forward_test/remote_commits.csv`.

   *Why the remote matters:* a hash held only on the author's machine is timestamped by a
   clock the author controls. A same-day push places an independent record of when each
   prediction existed outside the author's control. `docs/prereg.md` is pushed the same way at
   freeze, and its freeze commit hash and URL are recorded in §13. Predictions cannot be
   revised after the fact without the hash mismatching, and the mismatch is verifiable by a
   third party against the remote history.

4. **Scored-before-open condition, recorded per event.** For each event the job records
   `scored_before_next_open` (boolean), comparing the scoring timestamp against the next
   regular-session open from `data/trading_calendar.csv`. **A forward-test event contributes
   to the +1-horizon result only if `scored_before_next_open` is true.** Events failing the
   condition remain in the +21 and +61 forward results and are reported with their count. The
   job targets after-close releases before the following open precisely to satisfy this.
5. **Analyzed only after elapse.** No IC, spread, or t-statistic is computed for a horizon
   until that horizon's window has fully elapsed for the events being analyzed (§6.1). The
   +1 and +21 results will therefore be available before the +61 results, and each is
   reported with its own data date.
6. **Frozen configuration.** The forward test uses the §2 settings unchanged. Any change
   whatsoever voids the forward test as a prospective result, and it would be re-labelled and
   disclosed rather than quietly reported.
7. **Reported whatever it says.** A null or negative forward-test result is reported as the
   headline out-of-sample finding.

Expected size is approximately 500 events, based on the observed per-quarter counts
(2025Q4 516, 2026Q1 531, 2026Q2 518, 2026Q3 507); the realised count is reported exactly.

### 4.9 Pilot events are excluded from all confirmatory analyses

The 20 events scored in the Phase 3 pilot are recorded in `results/pilot/pilot_event_ids.csv`
and **excluded from every confirmatory analysis** — development, holdout, pending, and
forward test alike. They were inspected by hand, so they are no longer out-of-sample for any
purpose.

Any change to the prompt, the anonymization rules, or any inference setting **after** the
pilot is logged in `docs/deviations.md` with its date and reason, and the affected arm is
re-scored from zero.

---

## 5. Signal definitions

### 5.1 Exhibit selection (fixed preference order)

`EX-99.1` → `EX-99` → lowest-numbered `EX-99.x`. The label actually used is recorded per
event. `EX-99` (unnumbered) is required because some filers use it exclusively (verified:
Dominion `D`, Rockwell `ROK`).

### 5.2 Gemma signal

The model returns `SIGNAL ∈ {BULLISH, NEUTRAL, BEARISH}` plus a confidence in [0,1] and a
one-sentence reason, mapped to **+1 / 0 / −1**.

**ISOLATION.** The model sees only the anonymized release text. It never sees SUE,
consensus, prices, returns, the ticker, the company name, or the date.

**Primary signal (revised — see `docs/deviations.md` D-003, logged before any confirmatory
event was scored):** the primary signal is **`direction x confidence`** — `+confidence` for
BULLISH, `-confidence` for BEARISH, `0` for NEUTRAL. The direction-only `+1/0/-1` signal is
retained as a **pre-specified secondary** statistic. **Both are reported in every table,
always, on the same sample**; neither is ever presented alone.

*Reason:* the pilot produced 17 BULLISH of 20, tying ~85% of events at one value, which
attenuates Spearman rank IC for reasons unrelated to the research question. Confidence spans
0.60-0.95 over 6 distinct values with genuine within-direction variation, so the continuous
form breaks ties using information the model already emits. The change follows from the
*distribution of the predictor*, not from any observed relationship to returns — none had been
computed.

**Primary IC uses the full sample with NEUTRAL scored as 0.** A selective IC over
non-neutral events only is reported as a secondary statistic, with both sample sizes stated.
*Reason:* the prior version of this study compared the LLM's full-sample IC against SUE's
active-subset IC, which alone produced its headline result.

**The prompt wording is NOT changed in response to the bullish lean.** The lean is reported as
a finding (§11). Tuning the prompt against observed model behaviour would make the signal a
product of iteration rather than a fixed predictor.

### 5.3 SUE (seasonal random walk)

SUE = (EPS_q − EPS_{q−4}) / σ(past 8 available surprises), all from EDGAR XBRL, requiring no
analyst data.

Three rules, each verified necessary in Phase 1:

1. **Quarterly-period filter.** Only facts with `(end − start)` of **80–100 days** are
   quarterly. *Necessity:* 336 of 338 Apple EPS facts share duplicate `end` dates because
   year-to-date and quarterly facts collide; keying on `end` reads a 9-month figure (6.88) as
   a quarter (2.02).
2. **Earliest-`filed` per period.** Use the value as **originally reported**. *Necessity:*
   prior-year comparatives embedded in later 10-Qs carry `filed` dates up to a year after the
   period, which would inject look-ahead.
3. **Fiscal Q4 derivation.** No 10-Q is filed for fiscal Q4, so Q4 EPS = full-year minus
   9-month year-to-date. *Necessity:* roughly 25% of events otherwise have no quarterly EPS.

Every rolling statistic is lagged (`.shift(1)`). σ uses only surprises whose `filed` date
precedes the event's acceptance timestamp.

### 5.4 Consensus surprise — `[MISSING]`

No point-in-time consensus source is available. Benzinga returns `403 not entitled`; Finnhub's
free tier returns only four quarters (2025-09-30 → 2026-06-30) with no field establishing
whether the estimate is as-of-announcement or restated. Consensus surprise is therefore
**`[MISSING: no verifiable point-in-time free source`** and is named as a limitation. The
study compares the LLM against SUE only. This does not weaken H1, which concerns whether the
LLM beats a mechanical factor.

---

## 6. Returns, entry timing, and abnormal returns

**Entry timing rule.** Let `T_accept` be the EDGAR acceptance timestamp (ET). Entry is at
**the first regular-session open strictly after `T_accept`**, and returns begin from that
open.

Stated this way, a 07:00 filing enters at that same morning's open (the information is public
before the bell) and a 16:30 filing enters the next morning — without a special case, and
without accidentally imposing a one-day delay on half the sample. Measured distribution over
292 events: **54.8%** before 09:30, **44.2%** at/after 16:00, **1.0%** intraday (3 events).
Intraday filings enter at the **next** session's open, and their count is reported.

**Return windows.** Entry open → close of trading day **+21**, and entry open → close of
trading day **+61**, measured in trading days. A short **+1** window is reported as a
secondary descriptive statistic only.

**Abnormal return.** `AR = r_stock − r_SPY` over the identical window, both from Tiingo
dividend-adjusted closes, so both legs are total returns. A Fama-French 3-factor adjusted
version is reported as robustness, using factor data fetched from the Ken French data
library at analysis time.

**No look-ahead.** Every signal must be computable strictly from information public before
entry. This is verified mechanically in the audit (§9).

### 6.1 Window-completeness rule (no partial windows, ever)

An event enters an analysis **at a given horizon** only if the **entire** return window for
that horizon has elapsed on or before the **data date** of that computation. Partial windows
are never truncated, extrapolated, annualized, or filled.

Consequences, stated so they cannot be mistaken for data loss:

- The same event is eligible at +1 and +21 but not yet at +61. This is expected.
- Each reported result carries its **own data date and its own `n`**, and `n` differs across
  horizons within the same table. Tables state `n` per horizon explicitly.
- As time passes, more events become eligible. Any re-run after a later data date reports the
  new data date and the new `n` alongside the original, rather than silently replacing it.

**Exclusions at the data date 2026-09-21** (post-cutoff arm, 3,401 events):

| Horizon | Eligible | Excluded (window not elapsed) | % excluded | Last eligible entry |
|---|---|---|---|---|
| +1 | 3,401 | 0 | 0.0% | 2026-09-18 |
| **+21** | 3,365 | **36** | 1.1% | 2026-08-20 |
| **+61** | 2,885 | **516** | **15.2%** | 2026-06-24 |

The +61 horizon excludes 15.2% of post-cutoff events at this data date — essentially all of
2026Q3 and part of 2026Q2. This is a property of the calendar, not a data defect, and it is
reported as such. Counts are recomputed and re-reported at every data date, and the trading
calendar used is the one recorded in `src/window_completeness.py`.

---

## 7. Design rules

### 7.1 Anonymization

Before the model sees any text, the following are replaced with neutral tokens: company name
and all variants, ticker, brand and product names, executive and director names, city and
state of headquarters, and all dates, years, quarters, and fiscal-period labels. Every
replacement is logged per event to `results/anonymization_log/`, enabling the exact prompt to
be reconstructed and audited.

Anonymization is applied identically in both arms. It is **never** weakened or strengthened in
response to observed results.

### 7.2 Raw output retention

For every event, the **full thinking trace and final answer** are written to disk verbatim,
with token counts, `done_reason`, and wall time.

### 7.3 Leakage / re-identification scan

After scoring, every thinking trace is scanned for the real company name, ticker, product
names, and executive names. The **share of events where the model re-identified the company**
is reported, split pre- versus post-cutoff, on the matched sample.

**This is a finding, not a bug.** The prompt is never modified to suppress
re-identification, and re-identified events are never dropped. They are reported, and
primary results are additionally reported excluding them as a sensitivity check.

### 7.4 Truncation detection (both directions)

- **Input truncation.** Expected prompt tokens are compared against the returned
  `prompt_eval_count`; any divergence is logged as `INPUT_TRUNCATED`.
  *Necessity:* at `num_ctx=8192`, Ollama silently cut long prompts to **4,099 tokens** and
  still returned `done_reason: stop`. The median release is 8,724 tokens, so over half the
  corpus would have been scored as a fragment with no error raised.
- **Output truncation.** `done_reason == "length"` is logged as `OUTPUT_TRUNCATED`.

Truncated events are **reported, never silently dropped**.

**Unparseable outputs (see `docs/deviations.md` D-003b).** An output with no parseable
`SIGNAL` line is recorded as `UNPARSEABLE` and treated as **missing for that event** — not
coerced to NEUTRAL, since NEUTRAL is a prediction the model did not make. Such events are
excluded from IC with the sample size stated, reported as a count and share in every arm's
coverage table, and **never retried** — not with a different seed, temperature, or prompt.
A retry would make the recorded output a function of how many attempts it took. Observed in
the pilot: 1 of 20. A rate above 5% in any arm is reported prominently in limitations.

### 7.5 Reproducibility

After the main run, **50 randomly selected events are re-scored** with identical settings.
The **exact-match rate** for the final label and for the confidence value is reported. Any
non-determinism is disclosed rather than averaged away.

### 7.6 Run engineering

Results append to CSV after **every** event; the run resumes from the last completed event
after any crash or restart. Post-cutoff events are scored **first** so the headline data
completes first. GPU temperature and VRAM are logged every 10 minutes. A progress file
reports events done, remaining, mean seconds per event, and projected finish.

Measured throughput at the frozen settings: **23.7 s/event** → ~22 hours for the post-cutoff
arm, ~3.5 days for all 12,610 events.

No Windows settings are changed by the pipeline; required changes (sleep, automatic restart)
are reported to the author to apply.

---

## 8. Missing prices and delisting

Missing delisted companies are **not missing at random** — they are disproportionately
failures and acquisitions, exactly the left tail. Therefore:

1. **Ticker-variant resolution.** For any unresolved symbol, try the bankruptcy/OTC variants
   (`Q` suffix, then other documented variants) before declaring it missing.
   *Verified:* `SIVB` returns 404 but **`SIVBQ`** returns SVB Financial Group with 1,006 rows.
2. **Identity verification.** A price series is accepted only if the source's company
   identity matches the expected company. Tiingo's `name` field can be overwritten by a later
   ticker reuse while the `description` field retains the original company and a `DELISTED`
   prefix; identity is therefore checked against `description` as well as `name`, and the
   coverage window must overlap the index-membership period.
   *Verified:* `INFO` (IHS Markit) now returns "HARBOR PANAGORA DYNAMIC LARGE…" with data
   beginning 2024-10-10 — the original history is unavailable and the event must be excluded,
   not silently mis-priced.
3. **Every missing event is reported** in `results/missing_prices.csv` with its reason
   (`not_found`, `identity_mismatch`, `window_gap`, `insufficient_history`), tabulated by era
   and by removal reason.
4. **Delisting-return rule (pre-registered).** Where a company ceases trading inside a return
   window, the remaining window is assigned a delisting return of **−100% for bankruptcy or
   liquidation** and **the last available total return for acquisitions**, with the stub
   period earning the benchmark return. **All primary results are reported both with and
   without this rule applied**, so its influence is visible rather than assumed.

Measured delisted coverage at Tiingo with variant resolution: **8 of 10** tested names
(`FRC` not found under any tested variant; `INFO` excluded by the identity guard).

---

## 9. Statistical tests

- **Rank IC** (Spearman) between signal and abnormal return, per window.
- **t-statistics clustered by event date**, or Newey-West where a time series is used. Naive
  i.i.d. standard errors are not reported as primary.
- **Quintile long-short spreads** where signal granularity permits; for the 3-level LLM
  signal, the BULLISH-minus-BEARISH spread is reported instead and labelled as such.
- **Fama-French 3-factor adjusted** abnormal returns as robustness.
- Every comparison between signals is computed **on the identical event sample**, with `n`
  stated for each. Cross-sample comparisons are not reported as findings.

**Multiple testing.** The number of distinct hypotheses tested is fixed by this document.
Because the primary claim is a null (H1), no best-of-N selection is performed. Any additional
exploratory cut is labelled exploratory and is not reported as a result.

The word "significant" appears only where a test recorded in the numbers ledger supports it.

## 9.1 Seven-step audit (run before any writing)

1. Look-ahead check — every signal recomputable from pre-entry information only.
2. Raw-signal regression — signal against realized abnormal return, unconditional.
3. Sub-period split — results by year within each era.
4. Parameter sensitivity — return windows, trimming threshold, NEUTRAL handling.
5. Benchmark comparison — against SUE and against a random-signal null.
6. Truncation and coverage sensitivity — untrimmed-only and fully-parsed-only subsets.
7. Re-identification sensitivity — excluding events where the model named the company.

Every check and its result is written to `audit_report.md`, failures included.

---

## 10. Outputs, ledger, and verification

- `ledger.csv` — every number destined for the paper, with source file, row, column, and the
  script that produced it.
- `verify_paper.py` — extracts every number from the draft and checks it against the ledger.
  **The paper cannot ship until it passes with zero mismatches and zero unledgered numbers.**
- `citations_verified.csv` — every citation with the URL actually fetched. Unverified
  citations are marked `[CITATION NEEDED]` and never guessed.
- Tables and figures are generated by code from results files. No number is typed by hand.
- Daily copy of results, checkpoints, and logs to OneDrive (copy only; the live run never
  writes into a synced folder).

## 11. Reporting commitments

- The model is described exactly as in §2 and nothing more.
- The sample is described exactly as run: event count, ticker count, years, and every
  exclusion.
- **"S&P 500" is never used to describe the sample** unless it is the full index; the
  point-in-time universe is described as such.
- The two survivorship figures of §4.2 are reported separately with their exact labels.
- **Every table states `n` per horizon**, because window completeness (§6.1) makes `n` differ
  across horizons within the same analysis.
- **Every result states its data date.** Results recomputed at a later data date are reported
  alongside the earlier ones, never silently in place of them.
- The forward test (§4.8) is reported with its scoring timestamps and batch hashes, so a
  reader can verify predictions preceded outcomes.
- The 20 pilot events are named as excluded from all confirmatory analyses (§4.9).
- A limitations section covers: `[MISSING]` consensus, the trimming rule, unresolved tickers,
  delisted-price gaps, single price source, quantization, the 15.2% +61 window exclusion at
  the current data date, and the fact that a 4-bit local model is not the frontier.
- **AI-assistance disclosure:** AI wrote the code and drafted prose; the research question,
  design choices, and interpretation are the author's.
- Plain, non-overclaiming language throughout.

## 11.1 Result hierarchy (stated before any result is seen)

Results are reported in this order of evidential strength, and this ordering is fixed now so
it cannot be rearranged to favour whichever arm looks best:

1. **Prospective forward test** (§4.8) — scored before outcomes existed, hashed at scoring.
2. **Holdout** (2026Q1, §4.6) — frozen pipeline, run once, no retuning.
3. **Pending segment** (2026Q2–Q3) — out-of-sample, reported per horizon as windows elapse.
4. **Development sample** (2025Q1–Q4) — where the pipeline was built; weakest evidence.
5. **Pre-cutoff arm** — explicitly contaminated; reported only for the memorization gap.

## 12. Forward-test operations

The forward test runs as an automated daily job, `src/forward_test_daily.py`, invoked by
`scripts/run_forward_test.bat` from Windows Task Scheduler. Its sequence is fixed:

1. **Read and record the environment** (§2.1). **Halt with exit code 2 and score nothing** if
   Ollama version, model digest, weight blob SHA, or GPU driver differs from §2.
2. **Poll EDGAR** for 8-K Item 2.02 filings by point-in-time index members in the season
   window. Events already scored are skipped — the job is idempotent and resumable.
3. **Extract** the release via the §5.1 exhibit order and the §4.7 content gate.
4. **Anonymize** per §7.1, logging every replacement.
5. **Score** on the frozen config, trimming per §4.5 and detecting truncation per §7.4.
6. **Record `scored_before_next_open`** per event, comparing the scoring timestamp against
   the next 09:30 ET open from `data/trading_calendar.csv`.
7. **Hash** the batch (SHA-256) into `results/forward_test/hashes.csv`.
8. **Commit and push** the hash file, environment log, and scored index the same day;
   record the commit SHA and remote URL in `results/forward_test/remote_commits.csv`.
   If no `origin` remote is configured or the push fails, the job logs explicitly that the
   **off-machine timestamp was not established** and the commit is local only.

**Scheduling target.** The job runs twice daily so that after-close releases are scored
before the following open: once at **18:30 ET** (catching the 16:00–18:30 wave) and once at
**05:30 ET** (catching overnight filings and the pre-open wave). Both runs are the same job;
the second simply finds whatever the first did not.

Wall-clock feasibility at the frozen 23.7 s/event: a heavy earnings day in the S&P 500 is
roughly 60–80 releases, i.e. ~25–32 minutes of scoring — comfortably inside the window
between the close and the next open.

**Failure handling.** A failed run does not lose work: events already written are recorded and
skipped next time. A run that halts on an environment mismatch is visible in
`logs/forward_test_task.log` and in `environment_log.csv`, and no events are scored until the
mismatch is resolved.

---

## 13. Freeze record

| | |
|---|---|
| Freeze commit SHA | **`9073d54eebb92eff59d0911e1e41d316a12f369b`** |
| Frozen on | **2026-09-21** (local commit) |
| Files in freeze commit | 41 |
| Remote commit URL | https://github.com/Vamiko234/gemma-earnings-study/commit/9073d54eebb92eff59d0911e1e41d316a12f369b |
| Pushed off-machine | **2026-09-22** |

**Remote push status: COMPLETE.** The freeze commit and every subsequent commit are pushed to the private remote `github.com/Vamiko234/gemma-earnings-study`.

**Disclosed honestly:** the freeze commit was created locally on 2026-09-21 and pushed on 2026-09-22. The off-machine timestamp therefore attests to 2026-09-22, not 2026-09-21. Anything requiring an independent timestamp — above all the forward test (§4.8) — is dated from the push, not from the local commit. See `docs/deviations.md` for the same disclosure applied to D-002 and D-003.

**This does not block the Phase 3 pilot**, whose 20 events are excluded from every
confirmatory analysis (§4.9). It *does* gate the forward test (§4.8), which must not begin
until the freeze is off-machine.

The freeze commit contains this document, the committed trading calendar, the universe
classification with its evidence fields, and every Phase 1 result file. Pushing it to the
private remote places the pre-registration off the author's machine **before** any event is
scored, which is what makes the "pre" in pre-registration verifiable rather than asserted.

---

### 13.1 Scrubber freeze (deviations D-005, D-007, D-008, D-009)

**Refrozen 2026-09-22** after the over-scrub fixes of D-009. Audit 11 meets the stopping rule.

**Hash convention (corrected 2026-09-23).** Every hash below is the SHA-256 of the file
**as committed**, i.e. with LF line endings, reproducible on any machine with:

```
git show <commit>:<path> | sha256sum
```

The hashes recorded here before 2026-09-23 were taken from the Windows working tree, where
`core.autocrlf=true` stores LF and checks out CRLF. They therefore could not be reproduced
by anyone who cloned the repository, and two of them (`src/pilot.py`,
`src/forward_test_daily.py`) had in addition gone stale when D-010, D-011 and D-012 changed
those files. Both problems are corrected here; the scrubber itself never changed.

**Source freeze at commit `496774ab2867fa5ddb26f2fbc68f2c63cf769d74`:**

| Artefact | SHA-256 (as committed) |
|---|---|
| `src/scrub.py` | `103db06eb37d535445138777209505130af57239dec5f504c916559aeee49e4c` |
| `src/sue.py` | `c009eb40bc41e1ea8619669c4c4d8c896364fedcde1df2e6ed820a092a349944` |
| `src/common.py` | `93966a6a410a21504766f797156a4c3da31467f162b69971e66b3bc76b7ad6fa` |
| `src/forward_test_daily.py` | `062731498075e2a7d852cfe6abdbb2165bdf9717e4c51446aa6fac9adbda49c6` |
| `src/scrub_audit.py` | `cbb4ec58f042d9e8c358d59ce0db607ac813db1a17d5d03d18932017ebb51a42` |
| `src/pilot.py` | `7dc0885e5f04cbf148d5186325b180b8258ba067bfba09cb9acccdb936dd6790` |
| `src/gpulock.py` | `bea5977f78343b435ed35c0fa5e451e514153b2bac4e6d5713d53a0efeb8911d` |
| `src/test_trim.py` | `ea00707c09dbd1f1c99b14dbbc5fb1268df3a8965c2bcc60805c83d336dd63ab` |
| `FINANCIAL_STOP` (427 terms) | `6f85fa4c4870ba1db6e983fae74716312ab767793900f4691247ba100f044f2b` |
| `NAME_SUFFIX_STOP` (28 terms) | `1a7e5e4fbf3174d6ab3fd9a471a13eec2eedf9ddb5896308ad5ca32c5c3da55f` |

Stoplists verbatim in `data/scrubber_freeze.json`. The two stoplist hashes are over the term
lists themselves and are unaffected by line endings; `src/scrub.py` is byte-identical to its
D-009 refreeze, so the scrubber freeze of 2026-09-22 stands.

**Stopping rule (D-008):** refreeze when one fresh audit of 50 finds zero NEW leak classes
and zero over-scrubbed financial terms. Recurrences of a disclosed residual class do not
block refreezing.

| Audit | n | identity | temporal | over-scrub | verdict |
|---|---|---|---|---|---|
| a4 (404) | 46 | 18 | 195 | 0 | fail - compact quarter-year |
| a5 (505) | 46 | 1 | 3 | false + | fail - `FY'21`, `2050s` |
| a6 (606) | 45 | 15 | 1 | 0 | fail - issuer domain |
| a7 (707) | 48 | 15 | 2 | 0 | fail - left-glued ticker |
| a8 (808) | 46 | 1 | 0 | 0 | pass (identity only) |
| a9 (909) | 44 | 23 | 97 | 0 | fail - my own regressions |
| a10 (1010) | 43 | 3 | 26 | 0 | fail - incomplete month/brand fixes |
| **a11 (1111)** | **45** | **11** | **1** | **0** | **PASS** |

Audit 11's residues are all disclosed classes: `CAT` (Caterpillar), `IT` (Gartner), `53`
(Fifth Third). Its lexical-loss top-20 is entirely by design - `quarter`/`first`/`fourth`
consumed by `QUARTER_X`, `fiscal` by `FISCAL_X`, `com`/`www` by `URL_X`, and company names.

**Over-scrub is tested two ways** (D-009): a fixed financial-term list, and a general
**lexical-loss** check reporting any ordinary lower-case word that largely disappears, with
the top 20 per audit. The general check exists because the fixed list could not see the
modal verb "may" being deleted in essentially every event.

Six disclosed residual classes are named with examples in D-008/D-009 and in the paper's
limitations; together they are why **H3 is reported as an upper bound**.


### 13.2 Run configuration freeze (deviations D-011, D-012)

**Frozen 2026-09-23 at commit `496774ab2867fa5ddb26f2fbc68f2c63cf769d74`.** The confirmatory run uses exactly these values:

| Setting | Value | Set in |
|---|---|---|
| model | `gemma4:26b-a4b-it-qat` | `MODEL` |
| model digest | `2dd70431afed` | `FROZEN["model_digest"]` |
| weight blob | `sha256-4c856523d61d77922dbc0b26753a6bf6208e5d69d80db0c04dcd776832d054c5` | `FROZEN["weight_blob_sha256"]` |
| ollama | `0.34.2` | `FROZEN["ollama_version"]` |
| GPU driver | `591.86` | `FROZEN["driver_version"]` |
| `num_ctx` | 40,960 | `OPTS` |
| `num_predict` | 6,000 | `OPTS` |
| `num_gpu` | 28 | `OPTS` |
| `temperature` | 0 | `OPTS` |
| `seed` | 42 | `OPTS` |
| thinking | on | `"think": True` in `score()` |
| release trim | 30,000 tokens, enforced by measurement (D-012) | `TRIM_TOKENS` |

**One file, one definition.** Every one of these lives in **`src/scoring.py`** and nowhere
else (moved there from `src/forward_test_daily.py` by D-015, which also made that module the
single scoring path). `src/forward_test_daily.py`, `src/run_2x2.py`, `src/pilot.py` and
`src/probe_precutoff.py` all import the configuration and the scoring path from it rather
than restating any value. This is enforced by `src/test_shared_path.py`, which fails if any
runner defines its own scoring function or redefines any frozen setting - a check that exists
because two anonymisers coexisted undetected through eleven audits (D-015). The other files in `src/` that contain `num_ctx` or `num_predict`
literals - `bench2.py`, `bench3.py`, `bench8k.py`, `bench_model.py`, `bench_real.py`,
`crash_test.py`, `ctx_capacity.py`, `ptok_test.py`, `test_config.py`, `test_trim.py` - are
benchmarks, capacity probes and tests. None is on the scoring path and none is imported by
it. `test_config.py` deliberately holds the A/B/C candidate configurations of D-011 and must
not be mistaken for the frozen one.

A run whose environment does not match the `FROZEN` block halts without scoring (§2.1).
`TRIM_TOKENS + num_predict < num_ctx` is asserted at import, so the configuration cannot be
edited into a state where the engine silently truncates (D-012).

---

### 13.3 Independent timestamps (public mirror and archives)

The private remote places this document off the author's machine, but a private repository is
not *independently* verifiable: only the author can show it, and the author controls it.
Anything that turns on "this was written before that was known" needs a record held by
somebody else.

**The Open Science Framework is not used.** OSF requires account holders to be 18 and the
author is 15, so registering there personally is not available. A **parent-hosted OSF
registration may be added later**, with an adult as the named account holder. If that
happens it will be recorded here as an *additional* timestamp; it is **not** a precondition
for launching confirmatory scoring, and nothing in this protocol waits on it.

Independence is established instead by a public mirror plus two archives that are
independent of both the author and GitHub:

| Layer | Location | What it attests |
|---|---|---|
| Public mirror | `github.com/Vamiko234/gemma-earnings-prereg` | The pre-registration, deviations, freeze hashes and exclusion list, public and readable by anyone |
| Internet Archive | Wayback Machine capture of the mirror | The rendered pages existed, with this content, on the capture date |
| Software Heritage | Archive of the mirror repository | The repository and its full commit history existed, with these hashes, on the ingest date |

The mirror contains no release text, no prompts, no model outputs and no credentials. The
frozen source files are **not** published; only their hashes are, which fixes the content of
the analysis code now and defers disclosure of the code until publication. The hash
convention is §13.1's: SHA-256 of the file as committed, reproducible with
`git show <commit>:<path> | sha256sum`.

**The mirror is updated in the same session as any change** to this document, to
`deviations.md`, or to a freeze file (`scripts/sync_public_prereg.py`, and see the runbook).
A deviation whose mirror lags is a deviation whose archived date is wrong, which defeats the
purpose of writing deviations as they happen.

**Archive record.** Re-submitted after each freeze, gate and launch; every capture kept, not
replaced, since the sequence of captures is itself the evidence.

The `raw.githubusercontent.com` URLs are archived rather than the rendered `blob` pages,
because the raw URL returns the exact bytes the SHA-256 hashes are taken over. The repository
home page is archived as well, for the rendered README.

| # | Date submitted | Service | URL archived | Archive link |
|---|---|---|---|---|
| 1 | [PENDING] | Wayback Machine | `https://github.com/Vamiko234/gemma-earnings-prereg` | [PENDING] |
| 2 | [PENDING] | Wayback Machine | `https://raw.githubusercontent.com/Vamiko234/gemma-earnings-prereg/master/prereg.md` | [PENDING] |
| 3 | [PENDING] | Wayback Machine | `https://raw.githubusercontent.com/Vamiko234/gemma-earnings-prereg/master/deviations.md` | [PENDING] |
| 4 | [PENDING] | Wayback Machine | `https://raw.githubusercontent.com/Vamiko234/gemma-earnings-prereg/master/scrubber_freeze.json` | [PENDING] |
| 5 | [PENDING] | Wayback Machine | `https://raw.githubusercontent.com/Vamiko234/gemma-earnings-prereg/master/excluded_event_ids.csv` | [PENDING] |
| 6 | [PENDING] | Wayback Machine | `https://raw.githubusercontent.com/Vamiko234/gemma-earnings-prereg/master/FREEZE_HASHES.txt` | [PENDING] |
| 7 | [PENDING] | Software Heritage | `https://github.com/Vamiko234/gemma-earnings-prereg` | [PENDING] |

Software Heritage ingests the repository and its whole commit history, so it covers every
file at once and does not need a row per file.

**Confirmatory scoring does not begin until the two archive submissions above are confirmed
done and their links recorded here.** Until then every `[PENDING]` is exactly that, and the
independence claim is unproven rather than assumed.

---

## 14. What would falsify the headline claim

H1 is a null. It is **rejected** — i.e. the LLM does add information — only if, on
post-cutoff events, Gemma's full-sample rank IC exceeds SUE's on the identical sample, with a
date-clustered t-statistic supporting it, and the result survives the FF3 adjustment, the
sub-period split, the trimming sensitivity check, and the re-identification sensitivity
check. Failing any of these, the reported conclusion is that the LLM did not beat the
mechanical factor.
