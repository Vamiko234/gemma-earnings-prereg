# Deviations from the Pre-Registration

`docs/prereg.md` was frozen at commit `9073d54eebb92eff59d0911e1e41d316a12f369b`
(2026-09-21). Every departure from it after that point is recorded here, before or at the
time it takes effect — never retroactively.

Each entry states: the date, the prereg section affected, what changed, **why**, and what was
re-run as a result. A deviation that would alter a reported number requires the affected arm
to be recomputed, and the entry says so explicitly.

---

## D-001 · 2026-09-21 · Trimming budget reduced 28,000 → 24,000 tokens

**Prereg section:** §4.5 (release length and the trimming rule)

**Change:** the pre-registered trim budget of **28,000 tokens** is reduced to **24,000
tokens**, and the token→character conversion now uses the **minimum** observed density
(3.47 chars/token) rather than the mean (3.70).

**Why:** the original rule had no margin for token-estimation error, and it crashed on the
first event of the Phase 3 pilot.

Ollama raises `CUDA error: an illegal memory access was encountered` whenever
`prompt + num_predict > num_ctx` (established in Phase 1). The trim was specified in
*estimated* tokens using the mean density, but the true density varies 3.47–3.89 chars/token:

| True density | 28,000 est-tokens → actual | + `num_predict` 4,000 | vs `num_ctx` 32,768 |
|---|---|---|---|
| 3.47 | 29,856 | 33,856 | **OVERFLOW** |
| 3.70 | 28,000 | 32,000 | ok |
| 3.89 | 26,632 | 30,632 | ok |

A release at the dense end of the range therefore overflowed the context and crashed the run.
The error was mine: I set the threshold from the average case when it had to survive the
worst case.

Revised arithmetic, safe across the whole observed range:

| True density | 24,000-token budget → actual | + 4,000 | vs 32,768 |
|---|---|---|---|
| 3.47 | 24,000 | 28,000 | OK |
| 3.70 | 22,508 | 26,508 | OK |
| 3.89 | 21,409 | 25,409 | OK |

Headroom is ~4,700 tokens at worst case.

**Also added (defensive, not a change of rule):**
- `score()` retries with a 25%-harder trim if Ollama still reports a context/CUDA error, up
  to 3 attempts, and records `retry_attempts` and `trim_budget_tokens` per event. After 3
  failures it raises rather than returning a result.
- A hard post-hoc guard discards any result where `prompt_eval_count + eval_count` exceeded
  `num_ctx`, so a fitted-looking result from an overflowed prompt can never be recorded.

**Effect on the sample:** the trim threshold moves from roughly the 99th percentile of
release length to between the 97th and 99th (measured: p97 = 20,483 tokens, p99 = 28,897).
Approximately **2–3% of releases are trimmed instead of ~1%**. Every trimmed event is logged,
and the pre-registered untrimmed-only sensitivity check (§4.5) is unchanged and covers this.

**Re-run required:** none. The deviation was found during the Phase 3 pilot, **before any
confirmatory event was scored**. No development, holdout, pending, or forward-test event
existed at the time of the change.

**Detected by:** the pilot gate, which is what it is for.

---

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-21 | §4.5 | Trim 28,000 → 24,000 tokens; use min density 3.47 | Context overflow crashed the pilot; original margin assumed mean token density | None — pre-confirmatory |

---

**Editorial note (added 2026-09-22):** D-002 and D-003 below were authored and first written
into this document on 2026-09-21, in the same session as D-001, but were left uncommitted in
the working tree and were not pushed to the remote until 2026-09-22, alongside the Task
Scheduler automation fixes (see D-004). This gap is disclosed rather than hidden. It does not
affect the validity of D-002 or D-003: both were logged before any confirmatory event existed,
and the only events scored in the interim were the Phase 3 pilot (excluded per §4.9) and one
pipeline-validation test event (excluded per D-004). Going forward, deviations are committed
and pushed the same day they are written.

---

## D-002 · 2026-09-21 · Four implementation defects found by pilot run 1

All four are **bug fixes or faithful implementations of existing pre-registered rules**, not
changes to the rules themselves. All were found before any confirmatory event was scored.

### (a) SUE returned nothing for 20/20 events — unpadded CIK

The XBRL `companyconcept` endpoint requires a **zero-padded 10-digit CIK**. `events_pit.csv`
stores CIK as an integer, so the request was built as `CIK1090872` and returned **404**;
`CIK0001090872` returns 237 facts. Every SUE value was therefore `NaN`.

Worse, the fallback message read `[MISSING: no usable XBRL EPS facts]` — which was false. The
facts existed; the URL was wrong. The message now names the actual failure, including the HTTP
status. **A "fail loudly" rule is worthless if the message misattributes the cause.**

*Prereg §5.3 unchanged.*

### (b) Anonymisation leaked the company name in 5/20 events

`MARRIOTT`, `BERKLEY`, `VERISIGN`, `DECKERS` and `Johnson` all survived, because only the
full name and the suffix-stripped core were replaced — never the individual significant
tokens. A release saying "MARRIOTT REWARDS" kept the name in plain sight, which defeats the
ISOLATION requirement outright.

Now replaces the full name, the suffix-stripped core, **and each significant token** (≥4
chars, excluding a fixed generic list such as "international", "technologies", "brands").
Matching is right-bounded: a left `\b` fails on "W. R. BERKLEY", and unbounded matching ate
into the following word ("Deckers Outdoor Corp" inside "DECKERS OUTDOOR CORPORATION" left a
stray "ORATION"). Regression-tested on all five leaked cases plus YUM and LIN: **7/7 clean**.

*Prereg §7.1 unchanged — this makes the code do what §7.1 already required.*

### (c) Input-truncation detector produced 3 false positives

The detector compared `prompt_eval_count` against a character-derived estimate
(`ptok < expected * 0.85`), so it fired on any text **sparser** than 4.35 chars/token rather
than on truncation. The three flagged events measured 6.93 (PLD), 4.84 (FE) and 4.31 (ED)
chars/token; maximum observed `prompt_eval_count` was 18,527 against a real truncation
boundary of 28,768. **Zero events were actually truncated.**

Now flags only `ptok >= num_ctx - num_predict`, which is the condition under which truncation
can occur, and records `chars_per_token` per event so density is visible rather than inferred.

*Prereg §7.4 unchanged.*

### (d) Trimming used a corpus-average density and over-trimmed sparse releases

PLD's release (113,572 chars) was trimmed although at its true density of 6.93 chars/token it
was only ~16,400 tokens and would have fitted whole. Token density varies far more per
document (4.31–6.93 observed in the pilot) than the corpus-wide 3.47–3.89 range suggested.

`score()` now measures **that document's** density with one cheap `num_predict=1` probe on the
first 20,000 characters, applies a 5% safety factor, and converts the 24,000-token budget at
the measured rate. The retry-on-overflow guard and the hard `prompt+output > num_ctx` check
both remain.

*Prereg §4.5 rule unchanged ("first 24,000 tokens"); this measures tokens instead of
estimating them, which is what the rule always meant.*

**Re-run required:** pilot only. No development, holdout, pending, or forward-test event had
been scored.

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-21 | §5.3 (impl) | Zero-pad CIK for XBRL; honest error message | 404 made SUE fail 20/20 while reporting the wrong cause | Pilot only |
| 2026-09-21 | §7.1 (impl) | Replace each significant name token, right-bounded | 5/20 events leaked the company name | Pilot only |
| 2026-09-21 | §7.4 (impl) | Truncation flag uses the real context boundary | 3 false positives from a density heuristic | Pilot only |
| 2026-09-21 | §4.5 (impl) | Trim on measured per-document token density | Sparse releases were trimmed unnecessarily | Pilot only |

---

## D-003 · 2026-09-21 · Primary signal becomes signed continuous; unparseable handling pre-registered

**Logged BEFORE any confirmatory event is scored.** Pilot events only had been scored at the
time of this entry, and they are excluded from all confirmatory analyses (§4.9).

### (a) Primary Gemma signal: `direction x confidence`

**Prereg section:** §5.2

**Was:** `SIGNAL` mapped to +1 / 0 / -1. Confidence was recorded but unused.

**Now:** the primary signal is **`signal_num x confidence`**, i.e.
`+confidence` for BULLISH, `-confidence` for BEARISH, `0` for NEUTRAL. The
direction-only +1/0/-1 signal remains **pre-specified as a secondary statistic**, and
**both are reported in every table, always, with the same sample.** Neither may be
presented alone.

**Why, stated before any result is known:** the pilot produced **17 BULLISH of 20**, so the
direction-only signal ties roughly 85% of events at a single value. Spearman rank IC is
attenuated by ties of that size, which would understate *and* destabilise the primary
statistic for reasons that have nothing to do with the research question. The model already
emits a continuous confidence spanning 0.60-0.95 across 6 distinct values, with genuine
within-direction variation (the 17 bullish events spanned 0.70-0.95), so the continuous form
breaks ties on information the model actually produced rather than on noise we invent.

**This is not a response to a result.** No IC, spread, or return has been computed for any
event. The change is driven by the *distribution of the signal itself*, which is a property
of the predictor, not of its relationship to outcomes.

**Guard against multiple testing:** because both forms are pre-specified and both are always
reported, there is no opportunity to select whichever performs better. §11.1's result
hierarchy is unchanged, and §12's falsification criterion now requires the **primary
(continuous)** signal to meet every listed condition.

**Also unchanged on purpose:** the prompt wording. The bullish lean is **reported as a
finding**, not engineered away. Changing the prompt in response to the lean would make the
signal a product of iteration against observed behaviour.

### (b) Unparseable outputs

**Prereg section:** §7.4 (addition)

An output containing no parseable `SIGNAL` line is recorded as `UNPARSEABLE` and:

- **counted as missing** for that event -- it is NOT coerced to NEUTRAL, because NEUTRAL is a
  substantive prediction the model did not make;
- **excluded from IC computation** for that event, with the sample size reduced accordingly
  and stated;
- **reported** as a count and share in the coverage table of every arm;
- **never retried** -- not with a different seed, not with a different temperature, not with a
  reworded prompt. A retry would make the recorded output a function of how many attempts it
  took to get a parseable one.

Observed rate in the pilot: **1 of 20** (PLD, a 113,572-character release). If the rate
exceeds 5% in any arm, that fact is reported prominently in the limitations section rather
than mitigated after the fact.

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-21 | §5.2 | Primary signal = direction x confidence; direction-only kept as pre-specified secondary, both always reported | 85% of events tied at +1 attenuates rank IC; continuous field already emitted and varies within direction | None -- pre-confirmatory |
| 2026-09-21 | §7.4 | Unparseable = missing, reported, never retried | Retrying would make output a function of attempt count | None -- pre-confirmatory |

---

## D-004 · 2026-09-22 · One pipeline-validation test event scored outside the season window

**Prereg section:** §4.8, §4.9 (analogy to pilot-event exclusion)

**What happened:** the Windows Task Scheduler automation (`scripts/run_forward_test.bat` →
`src/forward_test_daily.py`) was being verified end-to-end for the first time. Two defects
were found and fixed before this entry (see below), and then one real, already-public 8-K
Item 2.02 filing was deliberately run through the **full** pipeline — poll, extract, anonymize,
score on the frozen §2 configuration, hash, commit, and push — to confirm the automation
actually works, not just that it imports cleanly.

**Event:** `ADBE`, accession `0000796343-26-000147`, filed 2026-09-10 (i.e. **before** the
frozen forward-test season window of 2026-10-01 → 2026-11-30, §4.8). Scored 2026-09-22 16:42:20
local, signal BULLISH, confidence 0.95. Batch `batch_20260922T164219.jsonl`, sha256
`c0256f32ef63c7dd...`, committed `0c5a19883d95e3a25d59f4a05f62becbc238de0b`, pushed to
`https://github.com/Vamiko234/gemma-earnings-study/commit/0c5a19883d95e3a25d59f4a05f62becbc238de0b`.

**Why this does not contaminate the forward test:**
1. Its filing date (2026-09-10) is chronologically before the season window start
   (2026-10-01), so it is excluded by the season-window date filter used by every real run —
   this is not a special case added after the fact, it is a consequence of when the test was
   run.
2. It is additionally listed by accession number in `results/forward_test/test_event_ids.csv`
   (mirroring `results/pilot/pilot_event_ids.csv`, §4.9) so any analysis script can assert its
   exclusion explicitly rather than relying on the date filter alone.
3. It is retained on disk with its hash, never deleted, per the same standard applied to void
   and pilot events.

**Two bugs found and fixed by this same verification run, before any season-window event
existed:**
- `poll_edgar()` returned `pd.DataFrame(out)` with no declared columns. When zero candidates
  were found (the normal case outside earnings season), the DataFrame had no `.acc` column,
  and `main()` crashed with `AttributeError: 'DataFrame' object has no attribute 'acc'`
  (`src/forward_test_daily.py`, `poll_edgar`). This crashed all three manual runs on
  2026-09-22 before the fix (rc=1 each). Fixed by declaring
  `columns=["ticker","cik","acc","filing_date"]` explicitly. Verified after the fix: a
  zero-candidate run now completes with `"nothing to score"` and exit code 0.
- (Operational, not code) The Task Scheduler task "Gemma Forward Test" had never fired since
  creation (`LastTaskResult` 267011 = has-not-yet-run) and the Task Scheduler operational
  event log was disabled, so no scheduler-side history existed to check. Not a pipeline defect;
  noted here because it explains why no prior scheduler-triggered run could be inspected.

**Re-run required:** none. No development, holdout, pending, or season-window forward-test
event existed at the time of this test.

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-22 | §4.8/§4.9 | One test event (ADBE, `0000796343-26-000147`) scored pre-season to validate the Task Scheduler automation end-to-end; excluded via `test_event_ids.csv` and by season-window date | Automation had never been verified to actually run — need to distinguish "silently working" from "silently broken" | None — pre-season, test event excluded |

---

## D-005 · 2026-09-22 · Two-tier scrubbing, independent leak detector, H3 identity probe, SUE fallback

**Logged before any confirmatory event was scored.** Approved by the author 2026-09-22.
Every item below is a faithful implementation of an already-frozen rule, plus three new
pre-registrations in (5).

### (1) The leak detector is now INDEPENDENT of the scrubber

Pilot run 3 reported "0/20 leaks" while `www.firstenergycorp.com`, `hp.com` (136 times),
`@MarriottIntl` and `341 White Pond Drive Akron, Ohio` were all still in the text. The
detector used `\b` word boundaries, which never match inside `firstenergycorp`, and it drew
its probes from the same logic as the scrubber — so it could only ever confirm the scrubber's
own assumptions.

The detector now builds its probe list from **sources the scrubber never consults**: the
EDGAR entity name and **former names**, the tickers EDGAR lists, the HQ **street and city**
from EDGAR's address block, and any **website domain discovered in the filing itself**. It
matches by **substring**, except for probes of 3 characters or fewer which use
case-sensitive whole-token matching.

*Measured on the 20 pilot events: 2,818 raw identifier occurrences → **0 residual**.*

### (2) Two scrub tiers

**Tier 1 — removed** (pure identifiers, no financial meaning): URLs, bare domains, email
addresses, document filenames, social handles, postal addresses, ZIP codes, EDGAR-sourced
street and city, tickers, every company-name variant including former names, and executive
names adjacent to a title cue.

**Tier 2 — neutralised** (identifying but financially meaningful): brand and product names
become `BRAND_1`, `BRAND_2`, … consistently within a document, so
"BRAND_3 system sales +14%" keeps its financial structure while losing its identity.

**Names of 3 characters or fewer** use case-sensitive whole-token matching, so `HP` does not
consume `hp` inside other words and `ALL`/`CAT`/`LIN` do not eat ordinary prose. Tickers of
4+ characters are matched with a **left boundary only**, because extraction artefacts glue
the ticker to the following glyph (`NDAQF` = NDAQ + a footnote marker) and a trailing
boundary let that through.

### (3) Direct identity probe for H3

Re-identification was previously inferred by scanning the reasoning trace, which only catches
what the model happens to verbalise while doing another task. H3 is now measured **directly**:
after scoring completes, a **separate** model call on the **same scrubbed text** asks which
company issued the release, returning a guess, a confidence, and its strongest clue.

The probe **never feeds the signal**: it runs after scoring, its output is never shown to the
scoring call, and scoring is temperature 0 with a fixed seed, so it cannot influence the
prediction.

**H3 is reported split pre- versus post-cutoff**, because re-identification only creates
memorization risk for events inside the training window. A high post-cutoff rate means the
model can infer identity from business facts; a high **pre-cutoff** rate is what would
implicate recall. All 20 pilot events are pre-cutoff, so the pilot measures only one side.

### (4) SUE companyfacts fallback

`companyconcept` returns an **empty units dict** for some filers — JCI returned 537 bytes and
zero facts while `companyfacts` held 249 diluted-EPS facts for the same company. Sampling the
development segment's 51 missing-SUE tickers, **13 of 18 were recoverable** this way.

Both endpoints are now tried, the **identical** point-in-time rules are applied to whichever
returns data (80–100 day periods, earliest-filed per period, lagged sigma, nothing filed on or
after the event), and **the endpoint used is recorded per event** so coverage can be audited
by source. Pilot SUE coverage: **19/20 → 20/20**.

### (5) Newly pre-registered

- **SUE winsorisation: ±5**, applied **only** to magnitude-sensitive statistics (means,
  spreads, regressions). **Rank IC is always computed on unwinsorised values.** Measured tail
  in the development segment: |SUE|>3 is 7.04%, **|SUE|>5 is 1.32%**, |SUE|>10 is 0.18%, max
  13.4. The count of winsorised events is reported.
- **Missing SUE:** Gemma results are reported on the **full sample**; the Gemma-vs-SUE
  comparison is computed on the **SUE-available subset**, with **both sample sizes stated in
  every table**. Dropping ~10% of events from the headline to accommodate a benchmark
  limitation would distort the primary result, while comparing across different samples is
  precisely the error that produced the prior version's false headline.
- **Known limitation — bearish confidence clustering:** in the pilot all three BEARISH events
  carried confidence exactly 0.70, so the short side of the signed continuous signal has no
  within-direction variation. The continuous signal is therefore effectively "confidence among
  bullish events". This is disclosed in limitations and re-measured on the full sample.

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-22 | §7.1 (impl) | Independent substring leak detector from EDGAR + filing domain | Word-bounded, scrubber-derived detector reported 0 leaks with 2,818 present | Pilot only |
| 2026-09-22 | §7.1 (impl) | Two-tier scrub; short names case-sensitive; tickers left-bounded | Short names and glued tokens bypassed the old rules | Pilot only |
| 2026-09-22 | §7.3 | Direct identity probe; H3 split pre/post cutoff | Trace scanning under-measures; risk is pre-cutoff only | Pilot only |
| 2026-09-22 | §5.3 (impl) | companyfacts fallback; endpoint logged per event | companyconcept returns empty for some filers | Pilot only |
| 2026-09-22 | §9 | SUE winsorised ±5, magnitude statistics only | 1.32% of events, max \|SUE\| 13.4 | None — pre-confirmatory |
| 2026-09-22 | §5.3 | Missing SUE: full sample for Gemma, subset for comparison, both n | Avoids both sample distortion and cross-sample comparison | None — pre-confirmatory |
| 2026-09-22 | §11 | Bearish confidence clustering disclosed as a limitation | All bearish events at 0.70 in the pilot | None — pre-confirmatory |

---

## D-006 · 2026-09-22 · Pre-registered 2×2 scrubbed/unscrubbed × pre/post-cutoff ablation

**Logged before any confirmatory event was scored.**

### The design

Every event is scored **twice** on the frozen configuration — once on **scrubbed** text and
once on **unscrubbed** text — in both cutoff eras:

| | scrubbed | unscrubbed |
|---|---|---|
| **post-cutoff** | **PRIMARY (H1)** | ablation |
| pre-cutoff | ablation | ablation |

**What each gap measures:**

- **Post-cutoff (unscrubbed − scrubbed)** = information destroyed by scrubbing. The outcome
  did not exist when the model was trained, so identity cannot help by recall; any advantage
  from unscrubbed text is the value of the identifiers themselves (brands, geography, segment
  names) as financial context.
- **Pre-cutoff (unscrubbed − scrubbed)** = that same information loss **plus** whatever
  identity-driven memorization the model can exploit once it knows which company it is.
- **Difference of the two gaps** = the **headline memorization estimate**, with the
  information-loss component differenced out.

This is a cleaner estimator than comparing pre- and post-cutoff performance directly, because
it removes any level difference between eras (market regime, sample composition, base rates)
that would otherwise be misread as memorization.

**Primary H1 is unchanged**: scrubbed, post-cutoff. The three other cells are ablations and
are labelled as such. The result hierarchy of §11.1 is unchanged.

### Why this is not a fishing expedition

All four cells are specified now, before any confirmatory event exists, and **all four are
reported whatever they show** — including the case where scrubbing costs nothing (gaps ≈ 0)
or where the memorization estimate is negative. The estimator is a fixed arithmetic
combination named in advance, not a choice made after seeing which contrast is largest.

### Runtime

Scoring doubles. Measured cost per event on the frozen configuration in pilot 4:
scoring ≈ 20 s, plus the H3 identity probe ≈ 6 s, plus a density probe ≈ 2 s.

| Arm | Events | Scorings | Estimated wall time |
|---|---|---|---|
| Post-cutoff, scrubbed (PRIMARY) | 3,401 | 3,401 | ~26 h |
| Post-cutoff, unscrubbed | 3,401 | 3,401 | ~26 h |
| Pre-cutoff, scrubbed | 9,209 | 9,209 | ~72 h |
| Pre-cutoff, unscrubbed | 9,209 | 9,209 | ~72 h |
| **Total** | 12,610 | **25,220** | **~8.2 days** |

Up from ~3.5 days for the single-arm design. The post-cutoff pair alone — which carries the
primary result and one of the two gaps — completes in ~2.2 days and is run **first**, so the
headline and the information-loss measurement exist well before the pre-cutoff arms finish.

The H3 identity probe runs on the **scrubbed** cells only; asking which company issued an
unscrubbed release is not a measurement of anything.

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-22 | §4.8, §9 | 2×2 scrubbed/unscrubbed × pre/post ablation; memorization = difference of gaps | Separates information lost by scrubbing from identity-driven memorization | None — pre-confirmatory |

---

## D-005 (continued) · 2026-09-22 · Scrubber generalization audit and resulting fixes

Three audits of 50 randomly drawn events each, stratified across pre-cutoff / development /
holdout / pending, **excluding pilot events**, scrubbing only. Each audit used a fresh seed
and a fresh draw, so no fix was tuned on the sample that measured it.

| | audit1 (seed 101) | audit2 (seed 202) | audit3 (seed 303) |
|---|---|---|---|
| events scrubbed | 47 | 44 | 46 |
| raw identifier hits | 5,762 | 5,511 | 5,910 |
| **residual IDENTITY leaks** | **56** (13 events) | **32** (8 events) | **3 → 0** (0 events) |
| **over-scrub (financial terms)** | not yet measured | **29 / 44 events** | **0 / 46** |

**The pilot's "0 residual across 20 events" did not generalise.** Those twenty companies
happened to have distinctive names. A wider draw broke the rules in five separate ways.

### Defects found and fixed

1. **`FINANCIAL_STOP` was serving two incompatible jobs.** It filtered brand candidates *and*
   company-name tokens. Adding `"regions"` to stop it becoming `BRAND_n` silently deleted
   `REGIONS` from Regions Financial's name variants, and **"Regions Reports Solid Results"
   went unscrubbed 20 times in Regions' own release**. Same mechanism hid `HEALTH`/`CARE`
   (Welltower), `GENERAL` (General Dynamics), `DIGITAL` (Digital Realty). Name variants now
   use `NAME_SUFFIX_STOP` — corporate suffixes only.

2. **`EPS` was being destroyed.** The brand detector mapped `EPS` → `BRAND_45` across 28
   occurrences, and over-scrub affected **29 of 44 events**. The model would have read
   "BRAND_45 increased to $1.94". This leaks nothing, so the leak detector never objected —
   it was caught only by the audit's `MUST_SURVIVE` check. ~80 financial acronyms and
   headline terms added (`EPS`, `EBITDA`, `FFO`, `ROIC`, `CAPEX`, `backlog`, `bookings`,
   `ARPU`, …). Over-scrub is now **0/46**.

3. **URL path remnants.** Extraction splits `abc.com/NewsFromGoogle`, so the domain rule
   consumed the domain and left the brand. Path remnants are now removed.

4. **Glued tokens defeated both boundaries.** `ConsolidatedUnitedHealthcare`, `IQVIAFIN`.
   Distinctive single-token names (≥5 chars, used almost always capitalised in the original)
   are now replaced as bare substrings. Distinctiveness is measured from casing, so `ALIGN`
   is excluded and the ordinary noun "aligners" survives.

5. **Trademarked product names.** `iTero™`, `exocad™` were missed by frequency-based brand
   detection. A token immediately followed by ™ or ® is now treated as a product name
   regardless of frequency.

### Detector corrections (separate from the scrubber)

The detector was over-reporting as well as under-reporting. Two fixes, both principled
rather than cosmetic:

- **Specificity test.** A single name token is classified by how the *original* text cases
  it: a name used as a name is capitalised (`Marriott`, `Verisign`); a word that merely
  appears in a company's name shows up largely lower case (`health care` in a healthcare
  REIT, `price` in a Costco release). Generic matches are reported separately and not counted
  as identity leaks — a generic word reveals nothing about the issuer. This test is scoped to
  name-token probes only: domains, tickers and addresses identify the issuer regardless of
  casing, which is why `seabourn.com` must stay "specific".
- **Corporate-form words** (`REIT`, `HOLDING`, `GROUP`, `NATIONAL`, `INTERNATIONAL`) are
  classified generic outright. They reach the probe list through entity names and would
  otherwise count as leaks on a single occurrence, where the usage test has too little
  evidence to judge.

### Corrections to earlier claims in this file

An earlier entry justified aggressive name scrubbing as "over-scrubbing is safe; under-
scrubbing is not". **That is wrong**, and audit 2 proved it: scrubbing `PRICE` from a Costco
release or `EPS` from any release destroys the financial content the model must read. Both
failure directions are costly, and both are now measured every audit.

### Known, disclosed limitations

- **Ambiguous single-token names.** `ALIGN`, `EXPRESS`, `GENERAL`, `Desk` are simultaneously
  company names and ordinary words. Replacing them damages prose; leaving them is a partial
  identity hint. Not solvable by rule; reported rather than forced to zero.
- **Brand residue: 8 of 46 events, median 2 per affected event.** Mostly geography
  (`Uganda`, `Singapore`, `Canada`, `Nevada`) and low-frequency product names (`Megapacks`,
  `Cybercab`, `BREIT`). Geography is both identifying and financially meaningful, so it is
  deliberately left where Tier 2's frequency threshold does not reach it.
- These residues inflate H3 (re-identification) and are therefore reported as making H3 an
  **upper bound**.

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-22 | §7.1 (impl) | Name variants use NAME_SUFFIX_STOP, not FINANCIAL_STOP | Company names were silently deleted from the variant list | Pilot only |
| 2026-09-22 | §7.1 (impl) | ~80 financial acronyms added to brand stoplist | `EPS` was mapped to BRAND_n in 29/44 events | Pilot only |
| 2026-09-22 | §7.1 (impl) | URL path remnants; distinctive-substring names; trademark rule | Glued tokens and ™-marked products defeated boundary matching | Pilot only |
| 2026-09-22 | §7.1 (impl) | Detector specificity test; corporate-form words generic | Generic vocabulary was miscounted as identity leakage | Pilot only |

---

## D-007 · 2026-09-22 · Temporal, contact and state leaks; H3 time probe. Scrubber unfrozen and refrozen

**Logged before any confirmatory event was scored.** The scrubber frozen in D-005 is
**unfrozen** for these fixes and **refrozen** afterwards with new hashes in prereg §13.1.

### How these were found

The author read the FE sample in Obsidian. Every class below was present in text the three
generalization audits had passed, because **the audits only ever tested for identity**. The
prereg requires all dates removed (§7.1) and the detector never checked.

### (1) Temporal leaks

| Class | Example found | Fix |
|---|---|---|
| Bare month names | `closed in March YEAR_X` ×2 | Full dates matched **before** bare months, so "April 23, 2025" no longer degrades to "MONTH_X 23, YEAR_X" and leak the day |
| 6-digit release codes | `(042325)` = 23 April 2025 | `DATECODE_X` |
| Numeric dates | `4/23/25` | `DATE_X` |
| Fiscal-year markers | `FY25` | `FISCAL_X` |

**The detector now tests time as well as identity** — `detect_temporal()` probes month names,
month abbreviations, 4-digit years, 6-digit date codes, numeric dates, day-month patterns,
quarter-year and fiscal-year markers, and phone numbers. These probes are independent of the
scrubber's own rules, as with the identity probes.

### (2) Contact blocks

`Tricia Ingraham` and `Karen Sagot` survived: the executive rule required a title cue
(CEO/CFO/President) and contact-block names carry none. **All** person names following a
contact label are now `PERSON_X`.

**Phone numbers survived entirely** — `(330) 384-5247`. The old pattern led with an optional
country code whose `\d{1,2}` swallowed the opening digits of the area code, so it never
matched. Area codes identify the HQ as precisely as the street address: 330 is Akron.

### (3) US states — one rule, applied consistently

The FE sample read **"customers in BRAND_12, BRAND_2, BRAND_5, West BRAND_8, Maryland and
BRAND_14 York"**. States were being swept up incidentally by brand detection: some
neutralised, Maryland untouched, "West Virginia" and "New York" split across a placeholder.

All 50 states plus DC are now a single Tier-1 rule producing `STATE_X`, applied **before**
brand detection, longest name first so compound states are never split.

### (4) Common words no longer become brands

Confirmed and fixed: `Transmission`, `Information`, `Highlights`, `Strategic`, `Special`,
`Changes`, `Less` were all mapped to `BRAND_n`. ~60 further report-structure terms added to
`FINANCIAL_STOP` (now 395 terms). Verified **none** of the watched words map to a brand
token, while `EPS` survives (15 / 25 / 20 occurrences in FE / HPQ / YUM).

### (5) Unique-event fingerprints — disclosed limitation, not fixable by rule

Phrases such as **"House Bill 6"** or **"Deferred Prosecution Agreement"** identify an issuer
to anyone who follows the sector, and no pattern can catch them without deleting
company-specific events that are exactly the financial substance under study.

They are **not** scrubbed. Their effect is instead **measured** by the identity probe (§7.3):
if such fingerprints drive re-identification, the probe's hit rate reflects it. H3 is
therefore reported as an **upper bound** on what scrubbing can achieve, and this limitation is
named in the paper.

### (6) H3 time probe — the memorization channel

Identity alone is weak evidence of memorization: a model may infer the issuer from business
facts it has never seen. **Identity AND date together** is what would let it recall the actual
outcome.

A second probe now asks, on the same scrubbed text and after scoring, which fiscal quarter and
calendar year the release reports. Three figures are reported, each **split pre- versus
post-cutoff**:

1. **company identification** rate
2. **date identification** rate (calendar year; quarter reported separately because fiscal
   calendars vary)
3. **both correct** — the memorization channel

Neither probe ever feeds the signal: both run after scoring, and scoring is temperature 0 with
a fixed seed.

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-22 | §7.1 (impl) | Temporal rules: full dates before bare months, date codes, numeric dates, fiscal markers | Prereg requires all dates removed; audits never tested time | Pilot only |
| 2026-09-22 | §7.1 (impl) | All contact-block person names and all phone numbers | Area codes identify the HQ; contact names carry no title cue | Pilot only |
| 2026-09-22 | §7.1 (impl) | All 50 states + DC in one rule, before brand detection | States were handled inconsistently by brand detection | Pilot only |
| 2026-09-22 | §7.1 (impl) | ~60 report-structure terms added to FINANCIAL_STOP | Common words were becoming BRAND_n | Pilot only |
| 2026-09-22 | §7.3 | H3 time probe; report identity / date / both, split pre vs post | Identity+date together is the memorization channel | None — pre-confirmatory |
| 2026-09-22 | §7.1 | Unique-event fingerprints disclosed as an unfixable limitation | Cannot be rule-scrubbed without deleting the financial substance | None — pre-confirmatory |

---

## D-008 · 2026-09-22 · Scrubber stopping rule, and the disclosed residual classes

### Stopping rule (pre-registered, author-approved 2026-09-22)

The scrubber is **refrozen** when a single fresh audit of 50 events finds:

1. **zero NEW leak classes**, and
2. **zero over-scrubbed financial terms**.

A *new class* is a mechanism not already listed under "disclosed residual classes" below.
Recurrences of a disclosed class do **not** block refreezing — otherwise the audit loop never
terminates, because each fresh draw surfaces new company-name shapes from a long tail.

If an audit finds a new class: fix it, log it here, and run **one more** fresh audit under the
same rule. Audits use a fresh seed and a fresh draw, so no fix is ever tuned on the sample
that measured it.

**Assessment of the audits run so far:**

| Audit | New classes found | Over-scrub | Meets rule? |
|---|---|---|---|
| audit4 (404) | compact quarter-year (`3Q25`, `4Q'25`) | 0 | no |
| audit5 (505) | `FY'21` apostrophe form; decade years (`2050s`) | 0 (the 1 flagged was a false positive) | no |
| audit6 (606) | issuer's own web domain as an identifier (`ti` → TI) | 0 | no |
| audit7 (707) | *pending* | — | *pending* |

### Disclosed residual classes (limitations section, with examples)

These are **not** fixable by rule without deleting the financial substance under study. Each
is named in the paper's limitations, and together they are why **H3 is reported as an upper
bound** on what scrubbing can achieve.

| Class | Example | Why it cannot be scrubbed |
|---|---|---|
| **Third-party IR agencies** | `PondelWilkinson Inc.` (Monster Beverage) | A separate firm named in the release; not derivable from the issuer's EDGAR record |
| **Name-words that are ordinary vocabulary** | `TAKE` (Take-Two), `Line` (Norwegian Cruise Line), `NEWS` (News Corp), `EXPRESS` (American Express), `GENERAL` (General Dynamics), `Desk` (The Trade Desk), `ALIGN` (Align Technology, inside "aligners"), `Booking` (Booking Holdings, inside "gross bookings"), `CAT` (Caterpillar), `IT` (Gartner), **`53`** (Fifth Third, whose domain is `53.com`, against ordinary numerals) | Replacing them destroys ordinary prose; leaving them is a partial identity hint |
| **Unique-event fingerprints** | `House Bill 6`, `Deferred Prosecution Agreement`, `VERITAC-2` (Pfizer trial) | Identifies the issuer to a sector reader, but is exactly the company-specific financial event under study |
| **Geography below the brand threshold** | `Sydney`, `Midwest`, `Uganda`, `Japan`, `Haiti` | Simultaneously identifying and financially meaningful (revenue by region); Tier 2's frequency threshold deliberately does not reach single mentions |
| **Low-frequency product names** | `Megapacks`, `Cybercab` (Tesla), `Vyndamax` (Pfizer), `BREIT` (Blackstone) | Appear once or twice, below the ≥2 frequency threshold that separates brands from prose |

### Detector corrections made in the same pass

- **`entity_token` probes now match whole tokens.** Substring matching found `ROPER` inside
  **"Property, plant and equipment"** and reported it as a Roper Technologies leak (audit 6).
- **Issuer web domains are now scrubbable name variants.** Texas Instruments' release opens
  "TI reports ...", and `TI` is derivable only from `ti.com`, never from the EDGAR entity
  name "TEXAS INSTRUMENTS INC" (audit 6).
- **Third-party platform domains excluded from probes.** `youtube`, `facebook`, `linkedin`,
  `twitter` were counted as identity leaks for Citigroup and CSX; a company linking to a
  platform does not make that platform its identifier (audit 4).
- **Over-scrub check is word-bounded.** An unbounded case-insensitive count matched `eps`
  inside `steps`, reporting PEP as `EPS: 72->17` when the true figure was `17->17` (audit 5).

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-22 | §7.1 | Stopping rule for scrubber audits | The long tail of company-name shapes makes "zero residue" unreachable; a termination criterion is needed | None — pre-confirmatory |
| 2026-09-22 | §11 | Five disclosed residual classes named with examples | Honest bound on what scrubbing achieves; H3 is an upper bound | None — pre-confirmatory |
| 2026-09-22 | §7.1 (impl) | Issuer domain as name variant; entity_token whole-token probes | `TI` leaked; `ROPER` false-positived inside "Property" | Pilot only |

### D-008 continued · audit 7 assessment and fixes

Audit 7 (seed 707, 48 events) did **not** meet the stopping rule: it found one real new
sub-class. Residues classified:

| Residue | Verdict |
|---|---|
| `TRMB` x1 | **NEW sub-class — real leak.** `FTRMB` glues a footnote marker to the LEFT of the ticker; the NDAQF fix removed only the right-hand boundary. Tickers of 4+ characters now match with no boundary on either side. |
| `day_month` x2 (SPG) | **Detector false positive.** "Augustine" (St. Augustine) matched the month "August". The temporal probe's month names now require a right boundary. |
| `Booking` x14 (BKNG) | **Disclosed class**, not new. The release writes both "Gross bookings grew 13%" and "Gross Bookings $46.7B", so the casing test cannot separate the company from the ordinary noun. Added to the name-words row above. |

Per the stopping rule, the two fixes were applied and one further fresh audit follows.

**Tooling note.** The patch that applied these fixes mis-matched line numbers and silently
deleted `TEMPORAL_PATTERNS` and `detect_temporal` from `scrub.py`. This was caught by an
import check immediately afterwards and the block was restored verbatim. Recorded because a
silent deletion of the temporal detector would have made every subsequent audit report zero
temporal leaks for the wrong reason.

---

## D-009 · 2026-09-22 · Systemic over-scrub of ordinary words. Scrubber unfrozen and refrozen

**Found by the author reading the refrozen FE sample.** Audit 8 passed it because the
over-scrub check only looked for a fixed list of financial terms.

### (1) The modal verb "may" was being deleted

`rep()` defaults to `re.I`, so every month rule matched case-insensitively and the lowercase
modal **"may" became `MONTH_X`**: 13 times in FE, 9 in HPQ, 8 in YUM. Every
forward-looking-statements section is full of it, so this affected essentially every event in
the corpus. The clearest illustration is HPQ's list of modals with one member replaced:

> "will," "would," "could," "can," " MONTH_X ," and similar terms

**Fix.** Month names are matched **case-sensitively** and only in **date context**: adjacent
to a day number, adjacent to a year, or following a date preposition ("in May", "since
March", "as of June"). A lowercase "may", "march" or "can" is never a month. Verified:
"results that may not be consistent" is untouched, while "April 23, 2025", "in May" and
"since March 2024" still scrub.

### (2) A general lexical-loss check, because the specific one was not enough

The financial-term check tested a fixed list and therefore could not see this. The audit now
computes **lexical loss**: any ordinary lower-case word appearing ≥3 times whose count drops
by more than half is reported, whatever it is, with the **top 20 removed tokens per audit**
and a placeholder census per event.

This check is general. It would have caught both the EPS destruction and the "may" bug.

### (3) Checking all three samples found two more instances of the same class

Applying the new check immediately surfaced worse damage than "may":

| Word | Before | Cause |
|---|---|---|
| `our` (YUM) | **56 → 0** | `Our` selected as a brand; case-insensitive replacement ate the pronoun |
| `property` (YUM) | 27 → 0 | `Property` → BRAND_34 |
| `foreign` (YUM) | 9 → 0 | `Foreign` → BRAND_43 |
| `restaurants` (YUM) | 25 → 0 | Former name **"TRICON GLOBAL RESTAURANTS INC"** put `RESTAURANTS` in the name-variant list; 22 of 25 occurrences were ordinary prose |

**Fixes, both data-driven rather than stoplist-driven:**

- **Single-word brand candidates** must be *predominantly capitalised* in the original to be
  treated as brands, and single-word brands are replaced **case-sensitively**. Multi-word
  brands keep case-insensitive replacement, which is what catches ALL-CAPS table headers
  such as "TACO BELL".
- **Single-token name variants** that are ordinary lower-case vocabulary are matched
  **case-sensitively**. Case-insensitive matching remains for names genuinely used as names —
  it is what catches "Yum!".

After the fixes, every remaining lexical loss is **by design**: `quarter`/`first` (consumed by
`QUARTER_X`), `www`/`com` (URL_X), `fiscal` (FISCAL_X), and the company's own name.

### (4) Era markers added to the disclosed residual classes

| Class | Example | Why it cannot be scrubbed |
|---|---|---|
| **Era markers** | "a new U.S. presidential administration", "the recent tariff announcements", "post-pandemic demand" | Dates the release to within months without naming a date, and is often the substance of the outlook being discussed. **Measured by the H3 time probe**, which is precisely what that probe exists for. |

### Why this keeps happening

Every failure in this family has the same shape: a rule written to catch identifiers also
matches ordinary language, and the detector that should have caught it was testing something
narrower. The leak detector cannot see over-scrub, because deleting "may" leaks nothing. The
financial-term check could not see "may", because "may" is not a financial term. The general
lexical-loss check is the first instrument that tests the right thing.

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-22 | §7.1 (impl) | Months case-sensitive and date-context only | Lowercase modal "may" became MONTH_X in ~every event | Pilot only |
| 2026-09-22 | §7.1 (impl) | Single-word brands: capitalisation test + case-sensitive replacement | "Our" 56→0, "Property" 27→0 | Pilot only |
| 2026-09-22 | §7.1 (impl) | Single-token name variants: case-sensitive when ordinary vocabulary | "restaurants" 25→0 via a former name | Pilot only |
| 2026-09-22 | §7.1 | Lexical-loss check added to the audit; top 20 reported | Financial-term list could not see non-financial over-scrub | None — pre-confirmatory |
| 2026-09-22 | §11 | Era markers added to disclosed residual classes | Dates a release without naming a date; measured by the time probe | None — pre-confirmatory |

### D-009 continued · audits 9-11

| Audit | n | identity | temporal | financial over-scrub | verdict |
|---|---|---|---|---|---|
| audit9 (909) | 44 | 23 | 97 | 0 | **fail** - my own regressions |
| audit10 (1010) | 43 | 3 | 26 | 0 | **fail** - incomplete month and brand fixes |
| **audit11 (1111)** | **45** | **11** | **1** | **0** | **PASS** - all residues in disclosed classes |

**Audit 9 failed on two regressions I introduced while fixing "may":**

- "Case-sensitive" name matching compared against the literal EDGAR string, so the variant
  `NETFLIX` never matched `Netflix` in prose: 12 residues for NFLX, 6 for PNW, 4 for CAT.
  Fixed by matching the **capitalised forms** (upper and title case), not the literal string.
- The month-block rewrite **deleted the compact quarter-year rules**, which showed up
  immediately as 29 temporal residues for JPM and 10 for MS. Restored.

That was the third time in this work that a line-range patch silently removed working rules
(after `TEMPORAL_PATTERNS` and `detect_temporal`). The audits are functioning as the test
suite the project does not otherwise have.

**Audit 10 failed on two incomplete fixes:**

- The bare-month rule only fired after a date preposition, missing "ended December" and
  "the December quarter" (26 residues, 15 of 43 events). A **capitalised** month is now
  always a month; lower-case "may"/"march" stay safe because the match is case-sensitive.
- Multi-word brands were still replaced case-insensitively, so "Natural Gas" and "Fair
  Value" destroyed the lower-case prose: `gas` -82, `natural` -62, `rental` -59, `value`
  -37, `fair` -33. Multi-word brands now match **Title Case and ALL CAPS only**.

**One root cause, five rules.** Every failure in this family was the same defect applied to a
different rule: a pattern meant to catch identifiers matched case-insensitively and therefore
also destroyed ordinary language. Company names, single-word brands, name tokens, months, and
multi-word brands each had to be corrected separately. The leak detector could never see any
of it, because deleting "may" or "our" leaks nothing; only the general lexical-loss check
tests the right thing.

Verified after the fixes that scrubbing did not weaken: Taco Bell 35->0, KFC 38->0, Pizza Hut
30->0, Personal Systems 5->0, Energize365 3->0, while `natural gas`, `fair value`, `our`,
`restaurants` and `may` all survive.

---

## D-010 · 2026-09-22 · Correction: the pilot's date result was misread, and the H3 metric is respecified

### The error

Pilot 5 was reported with the line "all pilot events are PRE-cutoff" and the conclusion that
"the memorization channel is closed". **Both are wrong.**

Pilot events are drawn from the development segment, which is 2025Q1–Q4 — every one of the
20 is dated **2025-02-06 to 2025-11-05**, i.e. entirely **POST-cutoff**. `events_pit.csv`
labels all 20 `post`.

The model's training cutoff is January 2025, so it **cannot name a year it never saw**. A
near-zero date-identification rate on post-cutoff events is expected **by construction** and
carries no information about memorization. The observed 1/20 exact-year rate is what the
design predicts, not a finding.

The word "closed" was also unsupportable on its own terms: **0 of 20 has a 95% Wilson
interval of [0%, 16%]**. An upper bound of one in six is not a closed channel.

Both statements are corrected in `src/pilot.py`, which now prints the structural caveat, and
in every report.

### (1) Pre-cutoff probe run

20 PRE-cutoff events are drawn at random (seed 4242), scrubbed with the frozen scrubber, and
passed through the identity and time probes **only** — no signal scoring, no SUE, no returns.
Results in `results/probe_precutoff/`. Like the pilot events, these 20 are **excluded from
every confirmatory analysis** and their ids are recorded in
`results/probe_precutoff/precutoff_event_ids.csv`.

### (2) Identity matching by alias, not by character count

The old rule took the first token of the EDGAR name and required ≥3 characters, which marked
**"HP Inc." wrong for HP Inc.** and under-reported H3 in every pilot run.

`src/identity_match.py` now matches against an alias set built from the EDGAR entity name,
EDGAR former names, and a small explicit table of short forms the company uses for itself
(`HP`, `IBM`, `3M`, `TI`, `AmEx`, `J&J`, …). Aliases of ≤3 characters require a whole-token
match so "GE" cannot hit "General".

### (3) Date metric respecified

The date metric is **absolute year error** plus **exact-match rate**, reported with its full
distribution rather than a single hit rate. Recorded per event: the probe's year, |error|,
and whether the probe returned a year at all.

**Post-cutoff date identification is structurally near zero.** It is reported for
completeness but is never interpreted as evidence about memorization. **The memorization
channel — identity AND date together — is defined on PRE-cutoff events only.**

### (4) Unparseable outputs are non-random

Measured on the pilot's 20 events:

| | |
|---|---|
| corr(unparseable, release chars) | **0.620, p = 0.004** |
| median chars, unparseable | **79,708** |
| median chars, parsed | **33,662** |

Two distinct mechanisms, one each:
- **ZTS** hit the 4,000-token `num_predict` cap (`done_reason = length`); the answer was cut
  off before the `SIGNAL` line.
- **PLD** is the longest release in the sample (113,572 chars), had its input trimmed, and
  returned `done_reason = stop` with no parseable signal.

This is **non-random missingness correlated with release length** and is reported as such,
not treated as random dropout. The caveat is that it rests on **2 events out of 20**, so the
correlation is suggestive rather than established; the full run will measure it properly.

### (5) Reporting language

All H3 and coverage figures are reported as **pilot estimates with 95% Wilson intervals**,
never as conclusions. Wilson is used rather than the normal approximation because it remains
valid at 0 and at n.

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-22 | §7.3 | Memorization channel defined on PRE-cutoff events only | Post-cutoff date ID is structurally near zero | None — pre-confirmatory |
| 2026-09-22 | §7.3 | Date metric = absolute year error + exact match, with distribution | A single hit rate hides how far off the guesses are | None — pre-confirmatory |
| 2026-09-22 | §7.3 (impl) | Alias-based identity matching | Character-length rule marked "HP Inc." wrong for HP Inc. | Pilot only |
| 2026-09-22 | §7.4 | Unparseable reported as non-random missingness vs length | corr 0.620, p=0.004 on pilot events | None — pre-confirmatory |
| 2026-09-22 | §11 | All probe figures reported with 95% Wilson intervals | 0/20 has an upper bound of 16%, not 0% | None — pre-confirmatory |

---

## D-011 · 2026-09-22 · Alias rule extended; config B adopted; GPU priority lock

### (1) Aliases include the issuer's operating subsidiaries and brands

Scoring **"KeyBank" as wrong for KeyCorp** was a measurement error, not a model error:
KeyBank is KeyCorp's operating bank and is named throughout KeyCorp's own release.

A guess is now accepted when any of the following holds, and the basis is recorded per event
so every acceptance is auditable:

1. **`edgar_alias`** — matches the EDGAR entity name, an EDGAR former name, or an explicit
   short form the company uses for itself (`HP`, `IBM`, `3M`, `TI`, `AmEx`).
2. **`shared_root`** — shares the parent's distinctive root, which is how operating
   subsidiaries are named (`KEYCORP` → root `key` → accepts `KeyBank`).
3. **`filing_alias`** — appears as a repeated, **multi-word**, non-financial capitalised name
   in the issuer's own filing.

The rule is applied **identically to both eras** and cannot be tuned per company.

**A first version was too loose and was caught during re-scoring.** It accepted
`CNC → "Elevance Health"` via the single generic word `Health`. Centene is not Elevance.
Filing aliases must now be multi-word and at least 8 characters.

**Both versions reported, as required:**

| | v1 EDGAR alias only | v2 + subsidiaries/brands |
|---|---|---|
| Pre-cutoff identity (n=20) | 12/20 = 60% [39%, 78%] | **13/20 = 65% [43%, 82%]** |
| Post-cutoff identity (n=19) | 11/19 = 58% [36%, 77%] | 11/19 = 58% [36%, 77%] |

Exactly one answer changes (`KEY → KeyBank`). Post-cutoff is unchanged, so the rule does not
inflate one era relative to the other.

### (2) Configuration B adopted

Tested on the 7 longest pilot releases, which include both unparseable events:

| Config | crashes | mean (ex-PLD) | peak VRAM | PLD prompt tokens |
|---|---|---|---|---|
| A `32k / 4000` (frozen) | 0/7 | 27.4 s | 15,189 MiB | 16,387 |
| **B `40,960 / 6000`** | **0/7** | **32.8 s** | **15,351 MiB** | **39,357** |
| C `32k / 6000` | 0/7 | 27.4 s | 15,195 MiB | 31,074 |

All three are stable with ~1 GB of VRAM headroom. **B is adopted**: it is stable, and it
passes **2.4× more of the longest documents** to the model (39,357 vs 16,387 prompt tokens on
PLD), which directly reduces trimming. The cost is ~20% slower per normal event.

`TRIM_TOKENS` rises 24,000 → 30,000 accordingly.

**One event remains unparseable under every configuration.** PLD's release is 114,038
scrubbed characters; under B it fills the entire context (39,357 prompt + 1,603 generated =
40,960 = `num_ctx`) and is cut off before emitting a `SIGNAL` line. Raising `num_predict`
does not help, because the binding constraint is input length, not output length. This is
reported as **non-random missingness correlated with release length** (D-010 item 4), not
treated as random dropout.

### (3) GPU priority lock

The main 2×2 run will still be going when the forward test starts on 1 October. Both need the
same GPU and the same 15 GB model.

- The **forward test holds priority** and never waits: it is time-critical, since an
  after-close release must be scored before the next open.
- The **main runner checks the lock between events** and pauses while it is held. It never
  interrupts an event in flight, so no partial result is ever written.
- Locks older than **3 hours are stale** and are ignored and cleared, so a crashed forward
  run cannot block the main run indefinitely.

Five end-to-end tests pass: free initially; acquire and observe; main pauses and resumes on
release; stale lock ignored and cleared; lock released after an exception.

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-22 | §7.3 | Aliases include subsidiaries and filing brands, both eras | KeyBank for KeyCorp was a measurement error | None — pre-confirmatory |
| 2026-09-22 | §2 | `num_ctx` 32,768 → 40,960; `num_predict` 4,000 → 6,000; trim 24,000 → 30,000 | Stable, and passes 2.4× more of the longest documents | None — pre-confirmatory |
| 2026-09-22 | §7.6 | GPU priority lock between the forward test and the main run | The two overlap from 1 October | None — pre-confirmatory |

---

## D-012 · 2026-09-23 · The trim rule never enforced. Ollama truncates silently, and the truncation detector could not see it

**Bug fix to an existing pre-registered rule, not a change to the rule.** §4.5 says at most
`TRIM_TOKENS` tokens of the release are kept. That is still the rule. What changes is that
the code now *enforces* it, by measuring, instead of estimating it from a character ratio.

### The symptom

D-011 recorded PLD at **39,357 prompt tokens against a 30,000-token budget** and read that
as an input-length limit. It is not a limit, it is a defect: the budget was exceeded by
**9,357 tokens (31%)**, which left 1,603 tokens of the 40,960 context for output where
`num_predict` asks for 6,000.

### Defect 1 — the safety factor pointed the wrong way

```python
density = max(CHARS_PER_TOK_LO, density * 0.95)      # CHARS_PER_TOK_LO = 3.47
body    = text[: int(TRIM_TOKENS * density)]
```

Characters per token is a *sparseness* measure, so converting a token budget to characters
conservatively means taking the **smallest** plausible ratio. `max()` takes the largest. For
any document denser than 3.47 c/t the measured value was discarded in favour of a number
that over-states sparseness, and the character budget it implied overflowed the token budget.

PLD measures **2.51 c/t** over its opening 20,000 characters. `max(3.47, 2.51 × 0.95)` =
3.47, so the cut was 30,000 × 3.47 = 104,100 characters, which is 39,357 tokens. The comment
above the line ("never assume the tail is sparser than the probed sample") describes the
opposite of what the line does.

### Defect 2 — one ratio cannot describe one document

Measured on PLD 2025-04-16 (`prompt_eval_count`, generation suppressed):

| prefix chars | tokens | cumulative c/t | marginal c/t |
|---|---|---|---|
| 5,000 | 1,409 | 3.55 | 3.55 |
| 10,000 | 3,252 | 3.08 | 2.71 |
| 20,000 | 7,955 | 2.51 | 2.13 |
| 40,000 | 20,073 | 1.99 | **1.65** |
| 60,000 | 28,984 | 2.07 | 2.24 |
| 83,280 | 35,053 | 2.38 | 3.84 |
| 104,100 | 39,282 | 2.65 | 4.92 |

Density inside a single release moves by a factor of **three** — narrative prose at one end,
financial tables at the other. Any ratio taken from the opening therefore mis-states the
rest, in a direction that depends on the document. No fixed or sampled constant is safe, so
the fix does not look for a better constant.

### Defect 3 — the engine truncates silently, and the detector was blind to it

Ollama 0.34.2 does not reject an over-long prompt. It discards all but roughly half the
context window, **and reports the surviving count** as `prompt_eval_count`. Measured by
sending prompts of increasing length:

| `num_ctx` | tokens sent | `prompt_eval_count` returned |
|---|---|---|
| 4,096 | 6,400 | 2,051 |
| 4,096 | 32,000 | 2,051 |
| 32,768 | 35,053 | 16,387 |
| 40,960 | ~41,300 | 20,483 |

The returned value is `num_ctx // 2 + 3` whatever was sent. The old detector asked
`ptok >= num_ctx - num_predict`; a truncated event reports a *small* number and so can never
trip it. The check was reading the output of the very step it was trying to detect
(rule 3), and it reported clean for exactly the events that were broken.

**It is the tail that survives, so the instruction is what gets discarded.** Verified
directly: the instruction "Ignore the text that follows. Reply with exactly one word: HEAD"
followed by 5,000 characters returns `HEAD`; followed by 540,000 characters it returns "It
appears you have provided a long repetition of the famous...". That is the same shape as
PLD's recorded pilot answer, "It appears you have provided a large, unstructured data
dump...". PLD was not refusing the task; it was never given the task.

### What was actually affected

`prompt_tokens == num_ctx // 2 + 3` is a signature, and every scored row was checked for it.

| Run | events | truncated |
|---|---|---|
| Pilot runs 1, 3, 4, 5 (`num_ctx` 32,768) | 20 each | **PLD only** (16,387 = 32,768/2+3, every run) |
| Pre-cutoff H3 probe (`num_ctx` 32,768) | 20 | none — the 11 longest were re-measured and the largest prompt was PFE at 26,921 tokens |
| Forward test to date | 1 | none (5,476 tokens) |

So the exposure is **one event, PLD, in the post-cutoff pilot**, in its score call and in
both of its H3 probes. No confirmatory result exists yet, so nothing confirmatory is
affected. **The Gate 3 pre-cutoff identity figure of 13/20 is not contaminated by this bug.**

### The fix

`fit_to_budget()` replaces the ratio with measurement: propose a character cut, count the
resulting prompt's tokens with a `num_predict=1` call, shrink, repeat, and **raise** rather
than return if the release still exceeds the budget. The measuring call uses options
identical to the scoring call, so the scoring call that follows reuses its KV cache — the
measurement is a prefetch, not duplicated work.

Truncation is guarded two ways, neither of which trusts the number it is checking:

1. **Monotonicity.** A longer prefix cannot tokenise to fewer tokens than a shorter one, so
   any count below the opening sample's count is not a count.
2. **The pinned constant.** Truncation returns the same value whatever is sent, so whenever
   a count lands in the only range where truncation is possible (`>= num_ctx / 2`), a
   10%-shorter prefix is measured as well. An honest count falls; a pinned one does not.

Three further assertions fail loudly rather than record a bad row: the release must be
within budget; the scoring call must report the **same** token count as the measurement
(otherwise the input changed under us); and `TRIM_TOKENS + num_predict < num_ctx` is asserted
at import, so a run that obeys the trim can never enter the truncation regime at all.

`identity_probe()` and `time_probe()` had the same `int(TRIM_TOKENS * density)` cut and now
use the same fitter.

### Verification

`src/test_trim.py`, offline, against a simulated tokenizer written from the *observed*
engine behaviour above rather than from the trimming code's assumptions. Six tests, all
passing, including one that confirms the simulator still reproduces the original overflow
(39,292 tokens for a 30,000 budget) and one that confirms a truncated count is rejected
rather than believed.

Live re-run of the two unparseable pilot events, on the identical stored scrubbed text
(`results/trim_fix/rerun_2026-09-23.log`):

| | before | after |
|---|---|---|
| **PLD** | 16,387 prompt tokens (truncated), `UNPARSEABLE` | 29,752 prompt / **29,662 release** ≤ 30,000, 11,208 tokens of output room, **BULLISH 0.75** |
| **ZTS** | 12,589 prompt tokens, `gen=4,000 done=length`, `UNPARSEABLE` | 12,589 prompt (never trimmed, never truncated), `gen=1,389 done=stop`, **BULLISH 0.70** |

**Both now parse.** The two had different causes and only PLD's is this bug: ZTS was never
over the trim, it ran out of *output* room under the old `num_predict=4,000` and was already
fixed by D-011's config B. PLD's H3 probes, which returned `[UNPARSED]` when truncated, now
return **Prologis** (identity, correct) and **Q2 2024** (date, wrong — the release is Q1
2025).

### Consequence for D-011's conclusion

D-011 stated that "one event remains unparseable under every configuration" and that "the
binding constraint is input length, not output length". Both statements were consequences of
this bug and are **withdrawn**. PLD parses once the trim is enforced. The pilot's PLD row in
all four archived runs is a corrupted row: it records `input_trimmed=True`,
`input_truncated=False`, `status=OK` for a prompt that had lost its instruction.

### Cost

One extra `num_predict=1` call per event, whose KV cache the scoring call reuses. Documents
that fit whole cost one measurement and are not otherwise touched. PLD needed four fitting
rounds and 196 s against ~33 s for a normal event; it is the longest release in the pilot.

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-23 | §4.5 | Trim enforced by measuring prompt tokens, not by a chars-per-token ratio | The ratio over-shot the budget by 31% on PLD; the rule itself is unchanged | None — pre-confirmatory. Pilot PLD rows are marked corrupted. |
| 2026-09-23 | §4.5 | Independent truncation guards (monotonicity, pinned constant) replace `ptok >= num_ctx - num_predict` | The old check read the output of the step it was detecting and could never fire | None — pre-confirmatory |

---

## D-013 · 2026-09-23 · Independent timestamping moves from OSF to a public mirror plus two archives

**Why.** The protocol assumed the pre-registration would be registered on the Open Science
Framework. **OSF requires account holders to be 18. The author is 15.** Registering there
personally is not available, and no amount of protocol design changes that.

This was a real gap, not a formality. Everything this study claims about "pre" rests on a
date that somebody other than the author can vouch for. The private remote is off-machine
but not independent: it is controlled by the author and visible to nobody else.

### What replaces it

A **public mirror**, `github.com/Vamiko234/gemma-earnings-prereg`, archived by two services
that are independent of both the author and GitHub:

- **Internet Archive (Wayback Machine)** — captures the rendered pages
- **Software Heritage** — archives the repository with its full commit history

Recorded in prereg §13.3, with a table of archive links and dates that stays `[PENDING]`
until the submissions are confirmed. **Confirmatory scoring does not begin until those links
are recorded.** OSF is removed as a launch requirement; a parent-hosted OSF registration may
be added later as an *additional* timestamp, never as a replacement.

Two honest limits, stated rather than glossed:

1. Neither archive attests that the *content is true*, only that it existed on a date. That
   is all a pre-registration timestamp ever attests, including OSF's.
2. The author can still delete or rewrite the public repo. What the author cannot do is
   un-archive it. The archives, not the repo, are the evidence.

### What is published, and what is not

| Published | Withheld |
|---|---|
| `prereg.md`, `deviations.md` | Earnings-release text (copyright, and it is the input under study) |
| `scrubber_freeze.json` (stop-lists, audit history, source hashes) | The model prompts |
| `excluded_event_ids.csv` (the 41 excluded events) | Raw model outputs, prices, credentials |
| `FREEZE_HASHES.txt`, `README.md` | All source code — **hashes only** |

Publishing hashes without code fixes the content of the analysis code now, while no
confirmatory result exists, and defers disclosure until publication. A reader today must
take on trust that the hashes correspond to working code; that becomes checkable when the
private repository is opened.

`scripts/sync_public_prereg.py` performs the mirror. It publishes only an explicit
allowlist, refuses to run if any other file has appeared in the public repo, and scans every
file for credential patterns, local paths and the scoring prompt before writing anything.

### Mirror timing is part of the protocol

The mirror is updated **in the same session** as any change to `prereg.md`,
`deviations.md` or a freeze file (runbook). A deviation that sits unmirrored has an archived
date later than its real date, which defeats the point of writing deviations as they happen.

### A freeze-record defect found while doing this

`data/scrubber_freeze.json` recorded source hashes taken from the **Windows working tree**,
where `core.autocrlf=true` stores LF and checks out CRLF. Seven of its nine hashes could not
be reproduced by anyone who cloned the repository, and two (`pilot.py`,
`forward_test_daily.py`) had additionally gone stale under D-010, D-011 and D-012. This is
the same defect corrected in §13.1 on the same day, from the same cause.

All nine are now recorded in the committed-LF convention, with `hash_convention` and
`hashes_at_commit` fields stating it in the file itself so the next reader cannot repeat the
mistake. `src/scrub.py` is byte-identical to its D-009 refreeze: **the scrubber freeze
itself never moved**, only the record of it was unverifiable.

`src/identity_match.py` and `src/test_trim.py` are included in the manifest; §13.1 had
listed neither.

### Also noted

The leveraged-ETF paper's plan to post to **OSF Preprints is blocked for the same reason**,
recorded in the runbook so it is not rediscovered later.

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-23 | §13.3 (new) | Public mirror + Wayback + Software Heritage replace OSF as the independent timestamp | OSF requires account holders to be 18; the author is 15 | None — pre-confirmatory |
| 2026-09-23 | §13.3 | Confirmatory scoring gated on the archive links being recorded | An unproven independence claim is worse than a disclosed one | None — pre-confirmatory |
| 2026-09-23 | §13.1 | `scrubber_freeze.json` hashes corrected to the committed-LF convention | Seven of nine were irreproducible off-machine; two were stale | None — the scrubber is byte-identical |

---

## D-014 · 2026-09-23 · The confirmatory 2×2 runner: design, and four decisions taken before it ran

**Context.** The 2×2 ablation was pre-registered in D-006 and the configuration frozen in
D-011/D-012, but **no confirmatory runner existed**. `pilot.py` scores 20 fixed events on a
single arm; it has no arm handling, no resumption, no exclusion logic and no GPU handshake.
D-011's statement that "the main runner checks the lock between events and pauses while it
is held" described code that had never been written. This entry records the runner that now
exists (`src/run_2x2.py`), and the decisions taken **before** it scored anything.

### (1) The headline gap is measured on the never-trimmed subsample

Scrubbing changes a release's length, so the scrubbed and unscrubbed versions of the same
filing tokenise differently and the 30,000-token trim **cuts them at different points in the
document**. For any trimmed event, the scrubbed/unscrubbed gap would then mix "what scrubbing
removed" with "how much of the document each arm saw" — two different things, one number.

**The headline gap is therefore computed on filings that were trimmed in NEITHER arm**, with
the full sample reported alongside as a robustness check and **trimmed counts reported per
arm**. Pre-registered here, before any confirmatory event exists, so the subsample cannot be
chosen to suit a result.

Rejected alternatives, and why: trimming the raw text first and scrubbing afterwards would
make the main run inconsistent with the forward test; trimming both arms to the more
restrictive arm's span would discard content from the arm that did not need trimming and
change the pre-registered rule for every event to fix a minority of them.

### (2) A manipulation check on the unscrubbed arms (deviation from D-006)

D-006 ruled that "the H3 identity probe runs on the scrubbed cells only; asking which company
issued an unscrubbed release is not a measurement of anything." That is correct as
*measurement* and wrong as *instrument validation*. If the identity probe does not score near
100% when the company name is in plain sight, the probe is broken — and every H3 number it
has ever produced is suspect.

So the identity probe also runs on a **random 50-event subsample of each unscrubbed arm**,
fixed seed 20260923, chosen before any result is seen and identical on a resume. These rows
are labelled `probe_kind=manipulation_check` and are **never pooled into H3**. This is an
independent check on the measuring instrument, which is the only kind of check worth having
(rule 3).

### (3) Version lock: RESTORE, not void

prereg §2.1 halts a run whose environment leaves the frozen values. What happens *next* was
never specified, and the sequential arm order makes it urgent: arms 1 and 4 are about ten
days apart, so an Ollama or driver update on day five would leave the pre/post comparison
confounded with the environment rather than the cutoff.

**The pre-registered response is to RESTORE.** The run halts without scoring; the machine is
rolled back to the exact frozen Ollama and driver versions; the fingerprint is verified; the
run resumes where it stopped. Events already scored stay valid, because they ran on the
frozen environment — that is what the halt guarantees. **Only if rollback is impossible are
the affected arms voided and restarted.**

For this to be possible rather than aspirational, the exact installers are archived locally
with their SHA-256 hashes **before launch**, and beginner-level rollback steps are written
into `docs/runbook.md` ("Rolling back Ollama and the NVIDIA driver"). A rollback plan that
depends on a vendor still hosting an old installer is not a plan.

### (4) No analysis of any arm until all four finish

The arm order — post scrubbed, post unscrubbed, pre scrubbed, pre unscrubbed — means the
primary result exists days before the ablations. That is deliberate: an abort still yields
the headline. It also creates an opportunity to look at arm 1 and let it influence arms 2–4.

**Pre-registered: no analysis of any arm is run until all four arms complete.** Progress
files report counts, timings and ETAs, never signals or outcomes.

### (5) One filing, one scoring — a duplicate found while building

**63 accession numbers appear twice in `events_pit.csv`**: dual-class shares (FOX/FOXA,
GOOG/GOOGL, UA/UAA, NWS/NWSA) put one CIK's single 8-K into the index under two tickers.

Scoring both is not merely wasteful. The text, CIK and identity are identical, and `scrub()`
already removes **every** ticker registered to the CIK, so both rows produce a byte-identical
prompt and — at temperature 0 with a fixed seed — a byte-identical answer. The filing would
enter the analysis twice as though it were two independent observations, inflating the
effective sample and breaking any standard error that assumes independence.

Each filing is therefore scored **once**, with every share class recorded in a `tickers`
column. The share classes have different prices, so the single signal is joined back to each
ticker's returns at analysis time, and **standard errors cluster on the filing, not the
ticker**.

Event counts after exclusions and this collapse:

| Arm | Filings |
|---|---|
| post_scrubbed (PRIMARY) | 3,358 |
| post_unscrubbed | 3,358 |
| pre_scrubbed | 9,148 |
| pre_unscrubbed | 9,148 |
| **Total scorings** | **25,012** |

D-006's runtime table assumed 25,220, i.e. 3,401 and 9,209 per arm, which counted the
exclusions and the dual-class duplicates. Those figures are superseded.

### The runner

`src/run_2x2.py`. It owns selection, persistence, resumption, progress and the GPU
handshake, and **never the scoring rule** — scoring goes through `scoring.py` and nothing
else (D-015).

**Resumption.** Per event: the raw output is written and fsynced, then the result row is
written and fsynced, then the resume key `(acc, arm)` is written and fsynced. The order
matters. A crash between the row and the key causes a **re-score**, never a **skip**: a
duplicate row is recoverable and a missing event is not. And because scoring is deterministic,
a duplicate pair is a free determinism check.

**Exclusions**, applied at selection, asserted three ways. Not "exactly 41 removed per arm" —
the 41 excluded events do not all live in every pool (20 pilot events are post-cutoff, 20
probe events are pre-cutoff), so that assertion fails on the first arm. What is asserted is
that per arm the removal equals the pool overlap; that no excluded id survives selection; and
that **across the whole run every one of the 41 matched some pool**. An id that never matches
is a typo or a stale accession number, which is the failure worth catching. Measured: 21, 21,
20, 20; union 41; none unmatched.

**Hashing** every 100 events and at each arm boundary, pushed daily. Recorded as
`kind=tamper_evidence_not_foreknowledge`, because these arms are historical: the outcomes
already exist, so a hash proves only that the file has not changed since it was pushed.
**prereg §4.8's "hashed before return windows elapse" language does not apply here** and must
not be reused for these arms. It applies to the forward test, where it means what it says.

**GPU lock** checked between every event via `gpulock.wait_if_held`. The runner never
acquires the lock; the forward test holds priority.

**Progress** per arm — done, left, percent, median seconds, mean seconds, ETA hours, expected
finish, elapsed. The ETA uses a **trimmed median**, not a mean: PLD took 196 s against ~33 s
typical, and a handful of long releases would make a mean-based ETA useless.

**Version lock** before each arm, every 100 events, and stamped into **every row** as an
environment fingerprint. Re-reading the environment costs about a second, too much per event;
stamping a cached value costs nothing and is what makes "which events ran on which
environment" answerable after the fact instead of inferred from timestamps.

### Tests

`src/test_runner.py` drives the **real** `run_arm()` with scoring faked out — the distinction
that D-011 got wrong. Its five gpulock tests passed while nothing called the lock, because
they tested `gpulock.py` in isolation. A component tested alone cannot tell you the system
uses it.

| Test | What it establishes |
|---|---|
| GPU lock integration | No event starts while the lock is held; no event is interrupted mid-flight; the runner visibly pauses; it never acquires the lock itself |
| Resume after a crash | Stop after 3 of 6, restart: arm completes, no duplicates, nothing lost, keys match rows |
| Write ordering | The row is durable before the key; three fsyncs |
| Progress ETA | Median 33 s survives a 196 s outlier that drags the mean to 41 s |
| Failed rows | An EDGAR failure still produces a full-width row with the fingerprint; the CSV never goes ragged |
| Probe subsample | 50 events, deterministic, scrubbed arms probe everything |

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-23 | §9 | Headline scrubbed/unscrubbed gap on the never-trimmed-in-either-arm subsample; full sample as robustness | Per-arm trimming cuts the two arms at different points, confounding the gap | None — pre-confirmatory |
| 2026-09-23 | §7.3 | Identity probe on a 50-event unscrubbed subsample as a manipulation check | An unvalidated instrument makes every H3 number suspect | None — pre-confirmatory |
| 2026-09-23 | §2.1 | Environment mismatch → RESTORE and resume; void only if rollback is impossible | Voiding ten days of scoring for a recoverable change is the wrong default | None — pre-confirmatory |
| 2026-09-23 | §11 | No analysis of any arm until all four complete | Sequential arms otherwise allow arm 1 to influence arms 2–4 | None — pre-confirmatory |
| 2026-09-23 | §4.9 | One scoring per filing; dual-class share classes collapsed, tickers retained | 63 filings appeared twice and would have entered the analysis as independent observations | None — pre-confirmatory |

---

## D-015 · 2026-09-23 · There were two anonymisers. The prospective arm was using the weaker one

**Found while specifying what "scrubber off" means for the 2×2's unscrubbed arm** (D-014).
The question "which function do we turn off?" had two answers, which is how the defect
surfaced — not through any test.

### The defect

`pilot.py` anonymised with the frozen `scrub()` — the one carried through eleven audits,
unfrozen and refrozen by D-005, D-007, D-008 and D-009, hashed in prereg §13.1.
`forward_test_daily.py` anonymised with its own `anonymise()`, a shorter, older function
that had never been through any audit at all.

The **forward test is the prospective arm**: the most credible part of the study, the one
whose predictions are hashed and pushed before the outcome exists. It was running the weaker
anonymiser, and was eight days from scoring live events.

Measured on identical input:

| Input | `anonymise()` (forward test) | `scrub()` (frozen) |
|---|---|---|
| `alphacorp.com` | **survives** | `URL_X` |
| `ir@alphacorp.com` | **survives** | `EMAIL_X` |
| `Revenue rose in May 2024` | `May YEAR_X` — **month survives** | `DATE_X` |

All three are classes the scrubber was specifically unfrozen to fix: contact blocks and
temporal leaks in D-007, issuer web domains in D-008 — the entry that records `TI` being
derivable only from `ti.com`. The forward test received none of those fixes, because it was
never on that code path.

`anonymise()` also has no tier-2 brand neutralisation, no stop-lists, no leak probes and no
temporal-residue detection. On YUM's release the frozen scrubber fires 16 tier-1 rules and
neutralises 49 brands; `anonymise()` has no equivalent step.

**Checked and cleared:** `anonymise()` does *not* carry the D-009 "may" defect. Its month
pattern requires a digit after the month name, so the lowercase modal verb survives. The
suspicion was tested rather than asserted.

### Why no test caught it

Every test in the project exercised one path or the other. Nothing tested that they were
**the same path**. A divergence between two implementations is invisible to any test of
either implementation, and this one survived eleven scrubber audits for exactly that reason.

### The fix

`src/scoring.py` is now the single scoring path, and holds the frozen configuration
(prereg §13.2 updated accordingly):

    prepare_text  ->  fit_to_budget  ->  score  ->  identity_probe / time_probe

`prepare_text(raw, ticker, cik, arm)` is the **only** place the scrubbed/unscrubbed
difference lives, which is what makes the 2×2's gap attributable to scrubbing rather than to
two code paths that differ in unrecorded ways. `anonymise()` is deleted.
`forward_test_daily.py` drops from 690 to 345 lines and keeps only what belongs to the daily
job: EDGAR polling, the trading calendar, the health line, hashing, git.

### The test that keeps it fixed

`src/test_shared_path.py` checks **identity, not behaviour**: `forward_test_daily.score`
must *be* `scoring.score`, the same object. A behavioural test would pass on the day a
copy-paste duplicate was made and then drift silently, which is precisely what happened here.
It also fails if any runner defines its own `score`, `fit_to_budget`, `prepare_text` or
anonymiser, or redefines the frozen config, and it scans every module in `src/` for a second
anonymiser under any name.

### Consequence for the forward test

The single event scored so far is the excluded ADBE pipeline test (D-004), so **no forward-
test result is affected**. From now on the forward test anonymises with the frozen, audited
scrubber, identically to the confirmatory arms.

Verified before the change was committed: the forward-test dry run completes cleanly, and
`prepare_text` was exercised on a real filing through the forward test's own call shape
(YUM 0001041061-25-000008: 50,827 → 50,877 characters, 16 tier-1 rules, 49 brands, ticker
not leaked, `URL_X` present).

One side effect worth recording: `test_trim.py` began hanging after the split. It patched
the token counter in the re-exporting module rather than in `scoring`, so the fake never took
effect and the "offline" test was making real GPU calls against 300,000-character synthetic
documents. Repointed at `scoring`; `calls=0` in the stale run is the evidence that the fake
had been bypassed.

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-23 | §7.1 | The forward test anonymises with the frozen `scrub()`; `anonymise()` deleted | The prospective arm was leaking issuer domains, IR emails and month names that the frozen scrubber removes | None — only the excluded ADBE test event had been scored |
| 2026-09-23 | §13.2 | The frozen config and the scoring path move to `src/scoring.py` | One definition, imported by every runner, so the two cannot drift apart again | None — pre-confirmatory |

---

## D-016 · 2026-09-23 · The 5-event rehearsal: a 15% transient GPU failure rate, silently discarded

**The rehearsal.** Before launch, the new runner was run end to end on **5 excluded events
per arm, through all four arms** — 20 scorings. Excluded events are the right rehearsal set
because they are already barred from every confirmatory result, and the runner writes test
output to `results/runner_test/`, never to `results/confirmatory/` (it asserts this).

It found two defects, both introduced by earlier entries in this file.

### Defect 1 — transient engine faults killed events outright

**3 of 20 scorings (15%) failed** with `CUDA error: an illegal memory access was
encountered`:

| Event | Arm | Outcome |
|---|---|---|
| HPQ | post_scrubbed | failed — **then scored fine on post_unscrubbed four minutes later** |
| SHW | pre_scrubbed | failed — **then scored fine on pre_unscrubbed** |
| ALB | pre_scrubbed | failed — **then scored fine on pre_unscrubbed** |

Every one of the three succeeded on a later call. The fault is the engine, not the release.

The pre-D-012 `score()` retried CUDA errors. **The D-012 rewrite dropped that retry**, so a
transient fault now raised and the event was recorded `SCORE_FAILED`. Projected over 25,012
scorings at the observed rate: **~3,751 events lost**.

### Defect 2 — a failed event was never retried, ever

Worse than losing them once: `append_row()` wrote the resume key for failed rows too. The
resume logic skips any event with a key, so a transient fault removed an event
**permanently** — not just from that run, but from every future resume. No amount of
re-running would recover it.

The two defects compound into the failure mode this study is least able to tolerate:
**missingness correlated with GPU state rather than with anything about the release**,
invisible in the output, on ~15% of events. All three failures happened on *scrubbed* arms,
which make three model calls per event (score, identity probe, time probe) against one for
most unscrubbed events — so the loss would have been biased **toward the primary arm**.

### The likely cause, stated as a suspicion

VRAM sits at **15,252 of 16,376 MiB (93%, ~1.1 GB headroom)** during scoring, at 74 °C.
D-011 measured peak 15,351 MiB across 7 events and called config B "stable with ~1 GB of
VRAM headroom" — on a test that ran no H3 probes and saw no crashes. That test was too small
and too short to see a 15% rate.

This is **not** established as the cause. What is established is that the faults are
transient and recoverable. The fix therefore treats them as transient rather than changing
`num_gpu`, which would mean re-freezing the configuration (D-011, prereg §13.2) days before
launch, on a suspicion.

### The fix

`score()` re-sends the **identical** prompt up to `ENGINE_RETRIES=3` times with a 20-second
backoff, for faults matching a known-transient list. It does **not** shrink the budget the
way the pre-D-012 loop did: after D-012 the prompt is *measured* to fit, so shrinking would
silently change what the model read for a reason having nothing to do with the release.
Retries are recorded per row (`engine_retries`, `engine_retry_detail`).

`run_2x2` keeps `failed_<arm>.csv` with attempt counts and **withholds the resume key until
an event is terminal**, so a failed event is retried on the next run. After
`MAX_EVENT_ATTEMPTS=3` it is given up on *deliberately*, logged as permanently failed rather
than dropped silently.

Covered by `test_a_transient_failure_is_retried_on_the_next_run`: a fault on the first pass
produces a failure row and **no** resume key; the next run retries the event, succeeds, and
writes the key; no event is lost.

### Defect 3 — the row schema had drifted, caught by the rehearsal's own test

Adding an `attempts` field gave failure rows 33 columns against the good rows' 32. The CSV
header is written once from the first row, so the file went ragged and unreadable. `SCHEMA`
is now the single definition, every row passes through `_pad()`, and `_pad()` asserts on
fields outside the schema. The project's own ragged-CSV test caught this within minutes.

### What else the rehearsal established

**The manipulation check works: 10 of 10.** Every unscrubbed event was identified correctly
— Yum! Brands, Marriott International, HP Inc., Prologis, W. R. Berkley, Eversource Energy,
The Home Depot, KeyCorp, The Sherwin-Williams Company, Albemarle. The probe is therefore a
working instrument, which is what makes a *low* score on scrubbed text evidence about
scrubbing rather than evidence about the probe (D-014 item 2).

**Scrubbing visibly changes the answer.** ES scrubbed → "Avangrid" (wrong); ES unscrubbed →
"Eversource Energy" (right). KEY scrubbed → "KeyBank"; unscrubbed → "KeyCorp". HD was
identified correctly in **both** arms — a genuine residual leak of the kind D-008 discloses.

**Measured runtime, replacing every earlier estimate:**

| Arm | n (OK) | median | filings | projected |
|---|---|---|---|---|
| post_scrubbed | 4 | 50.1 s | 3,358 | 46.7 h |
| post_unscrubbed | 5 | 46.8 s | 3,358 | 43.7 h |
| pre_scrubbed | 3 | 63.9 s | 9,148 | 162.4 h |
| pre_unscrubbed | 5 | 38.0 s | 9,148 | 96.6 h |
| **Total** | | | **25,012** | **349 h ≈ 14.6 days** |

Against D-006's 8.2 days and the 11.9 days carried in the handoff. The increase is config B
(~20% slower per event, D-011), the D-012 fit probe, and the H3 probes being counted. These
medians rest on 3–5 events per arm and the pre_scrubbed figure on only 3, so they are
indicative, not precise — the unscrubbed arms in particular will run **faster** than
projected here, because the rehearsal probed all 5 events while the real unscrubbed arms
probe only 50 in total.

**Trimming, for the never-trimmed headline subsample (D-014 item 1):** PLD was trimmed in
both post-cutoff arms and is therefore excluded from the headline gap; 3 of 4 post-cutoff
pairs and 3 of 3 pre-cutoff pairs were usable. Scrubbing *increases* token count — YUM 14,872
unscrubbed → 15,126 scrubbed — because `COMPANY_A` and `BRAND_33` tokenise longer than the
names they replace, which is precisely why the two arms cut at different points.

**One unparseable outcome** that is not a bug: ALB on the unscrubbed arm, 12,345 release
tokens, neither trimmed nor truncated. The model simply did not emit a `SIGNAL` line.
Unparseable outputs remain non-random and are reported as such (D-003, D-010 item 4).

### Open, and deliberately not resolved here

PLD scored **BULLISH 0.75** in a standalone run this afternoon and **BULLISH 0.80** in the
rehearsal, on a prompt of identical token length (29,662 release tokens both times), at
`temperature=0, seed=42`. If scoring is not reproducible, then a duplicate row is not a
determinism check, the forward test's frozen config does not fix its predictions, and D-003's
signed continuous signal carries noise in its confidence term. **This is being measured
before launch, not assumed either way.**

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-23 | §4.5 | Transient engine faults retried up to 3× on the identical prompt | 15% of rehearsal scorings failed on a recoverable fault and were discarded | None — pre-confirmatory |
| 2026-09-23 | §4.9 | A failed event keeps no resume key; retried next run, given up on after 3 attempts | A transient fault permanently removed an event from every future run | None — pre-confirmatory |
| 2026-09-23 | §10 | Runtime estimate revised to ~14.6 days from measurement | D-006's 8.2 days predates config B, the D-012 fit probe and the H3 probes | None — pre-confirmatory |

### D-016 continued · the clean rehearsal, on the fixed code

The rehearsal was re-run end to end after D-016 (retry, resume keys, schema) and D-017
(scrubber determinism). Old output preserved at `results/runner_test_prefix_d016/`.

**20 of 20 scored. Zero failures.**

Which is *not* evidence that the CUDA problem is solved, and must not be reported as such.
Nothing was changed that would prevent a fault — retries were added, and **the retry path
never fired**. The honest reading is that the fault rate is variable: 15% in the first
rehearsal, 0% in the second. One plausible difference is machine load — the first ran while
2.4 GB of installers were downloading and being SHA-256 hashed on the same machine; the
second ran on an idle system. That is a hypothesis, not a finding. The retry exists because
the fault is real and recoverable, whatever its rate.

**Measured runtime, second rehearsal:**

| Arm | median | filings | projected |
|---|---|---|---|
| post_scrubbed | 46.4 s | 3,358 | 43.3 h |
| post_unscrubbed | 37.5 s | 3,358 | 35.0 h |
| pre_scrubbed | 27.1 s | 9,148 | 68.9 h |
| pre_unscrubbed | 25.8 s | 9,148 | 65.6 h |
| **Total** | | **25,012** | **213 h ≈ 8.9 days** |

Against 14.6 days from the first rehearsal. **The gap between the two is the honest measure
of how little five events per arm can tell you**: the pre-cutoff scrubbed median moved from
63.9 s to 27.1 s on the same five releases. Per-event times within a single arm span
25–185 s. Treat 9–15 days as the range and the progress file's running median as the real
estimate once the run is under way.

### Unparseable outputs now have exactly one cause, and it is the output budget

2 of 20 scorings were `UNPARSEABLE`. Both had `gen_tokens = 6000 = num_predict`,
`done_reason=length`, and an empty answer after 18,759 and ~19,000 characters of thinking:

| Event | Release tokens | Trimmed | Cause |
|---|---|---|---|
| PLD | 29,648 | yes | thinking exhausted `num_predict` |
| ALB | 12,345 | **no** | thinking exhausted `num_predict` |

ALB matters more than PLD here: at 12,345 release tokens it is an ordinary-sized release,
nowhere near the trim. **This is not a length problem.** Some releases simply make the model
think past 6,000 tokens. With the trim enforced at 30,000, a worst-case prompt plus
`num_predict` uses 35,738 of 40,960 context tokens, and ALB's uses about 18,400 — so there
are 5,000 to 22,000 context tokens sitting unused while events are lost to the output cap.

**A change to `num_predict` was considered and is NOT being made unilaterally.** The
argument for it — greedy decoding at temperature 0 emits the same tokens regardless of the
cap, so raising it can only rescue events that currently hit it — was tested rather than
asserted, and the test did not fully support it:

| Event | `num_predict` 6,000 | `num_predict` 14,000 | |
|---|---|---|---|
| PLD | gen 2,314, `stop` | gen 2,314, `stop` | identical |
| WRB | gen 1,530, think 5,358 ch | gen 1,538, think 5,210 ch | **differs slightly** |

PLD is byte-identical; WRB is not. Either `num_predict` has a small effect on generation, or
there is occasional run-to-run variation at the margins that four identical WRB runs earlier
did not reveal. Either way the neutrality claim is **unproven**, so raising `num_predict` is
a genuine change to the frozen configuration (D-011, prereg §13.2) and belongs to the author,
before launch, not to whoever is writing code that evening.

### Unparseable outcomes are unstable, not just non-random

PLD on the **stored** pre-D-017 scrubbed text thinks 7,673 characters and answers BULLISH
0.75. PLD on the **new deterministic** scrubbed text — the same release, differing by 14
tokens — thinks 18,759 characters and never answers.

A 14-token change in the input flipped the outcome between "answers comfortably" and
"exhausts the entire output budget". D-010 item 4 records that unparseable outputs are
non-random and correlated with release length; this adds that for borderline events they are
also **unstable under trivial input perturbation**. Reported in the limitations, and a
further reason the unparseable rate is not treated as random dropout.

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-23 | §10 | Runtime stated as a 9–15 day range, not a point estimate | Two rehearsals of the same 20 events gave 14.6 and 8.9 days | None |
| 2026-09-23 | §11 | Limitations: unparseable outcomes are unstable under trivial input change, not merely non-random | PLD flipped on a 14-token input difference | None |

---

## D-017 · 2026-09-23 · The frozen scrubber was not a function. Same release, four runs, four documents

**This is the most serious defect found in the project so far.** `scrub()` — the frozen,
eleven-times-audited instrument that defines what the model is allowed to see — **returned a
different document on every run.**

### How it surfaced

Not from a test. PLD scored `BULLISH 0.75` in one run and `BULLISH 0.80` in another, on a
prompt of identical token length (29,662 release tokens both times). The obvious suspect was
the model, so that was measured first: WRB scored four consecutive times produced
**byte-identical** answers, identical `gen_tokens`, identical thinking length. The model is
deterministic. The input was not.

Scrubbing the same PLD release in four separate processes:

| Process | Characters | SHA-256 (first 16) |
|---|---|---|
| 1 | 114,043 | `c2efd731af0cb84f` |
| 2 | 114,054 | `b3a2c7602da392be` |
| 3 | 114,068 | `d99051f1685dccaf` |
| 4 | 114,067 | `2570a0bd35fe99d6` |

Four runs, four documents. **Within** a single process it was perfectly stable — scrubbing
the same text twice in one process gave identical output every time.

### The cause

```python
brands = [p for p, c in sorted(candidates.items(), key=lambda kv: -kv[1]) ...]
brands = sorted(set(brands), key=len, reverse=True)
```

Both sort keys are **non-total**. Brands with equal counts, and brands of equal length, tie —
and the tie is then broken by the iteration order of a `dict` or a `set` of strings. CPython
randomises string hashing per process unless `PYTHONHASHSEED` is fixed, so **that order
changes every time the interpreter starts.**

Brands are replaced longest-first and numbered in order, so a different tie order produces
different `BRAND_n` numbering *and* a different replacement sequence — which is why the
output length moved too, not just the labels. The first difference in PLD appears at
character 7: `BRAND_74` in one run, `BRAND_73` in the next.

### Why eleven audits, five pilot runs and a probe run all missed it

Every one of them scrubbed each document **once, in one process**, and had nothing to compare
against. Within a process the function is stable, so every check ever written agreed with
itself. The property that failed — reproducibility across runs — was never the thing being
tested, by anything.

This is the same shape as D-015 (two anonymisers, neither test ever comparing them) and D-012
(a truncation detector reading the output of the truncation). **A check that shares the
code's blind spot reports clean**, and this project has now produced three of them.

### The fix

Every sort key in `scrub.py` that could tie now ends in the string itself, making the order
total and independent of the hash seed:

```python
brands = [p for p, c in sorted(candidates.items(), key=lambda kv: (-kv[1], kv[0])) ...]
brands = sorted(set(brands), key=lambda b: (-len(b), b))
```

The same latent defect in two `sorted({...}, key=len)` alternation lists was fixed in the
same pass. After the fix, four separate processes produce **byte-identical** output
(114,007 characters, one hash), and so do three processes on the full 113,572-character
release.

### The test

`src/test_determinism.py`, which spawns **real subprocesses** with the default randomised
hash seed. An in-process test of this property is worthless — it passes while the property
is false, which is exactly what happened for eleven audits. It also scans `scrub.py` for the
defect *class*: any length-only or count-only sort over a set or dict.

### What this means for results already recorded

**Nothing confirmatory has been scored, so no confirmatory result is affected.**

For the pre-confirmatory work, the honest statement is that the exact anonymised text behind
each recorded row **cannot be regenerated** from the frozen code — the stored
`scrubbed_text` in `pilot_raw.jsonl` is the only record of what the model actually read.
Rows remain valid as records of *what was scored*; they are not reproducible *inputs*.

The leak properties are very likely unaffected: the brand **set** is determined by counts and
capitalisation, not by ordering, so the same names were neutralised in every run — only
their labels and replacement order moved. "Very likely" is doing real work in that sentence,
and it is why audit 12 is being run on the fixed scrubber rather than assumed to pass. The
pilot and probe results already carry the caveat that they are pre-confirmatory.

**The H3 numbers presented at Gate 3 were produced under the unfixed scrubber** and should be
read with that in mind. They are pilot figures on n=20 with Wilson intervals, already
reported as indicative.

### Refreeze

The scrubber is unfrozen and refrozen under the D-008 stopping rule, with audit 12 (seed
1212) as the gate. `src/scrub.py` changes, so prereg §13.1 and `data/scrubber_freeze.json`
take new hashes.

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-23 | §7.1 | All tie-capable sorts in `scrub.py` made total (hash-seed independent) | `scrub()` returned a different document on every run; same release, four processes, four outputs | None confirmatory — nothing confirmatory has been scored |
| 2026-09-23 | §13.1 | Scrubber refrozen with new hashes, gated on audit 12 | `src/scrub.py` changed | None — pre-confirmatory |
| 2026-09-23 | §11 | Limitations: pre-confirmatory anonymised text is not regenerable from the frozen code | Stated rather than discovered later by a replicator | None |

---

## D-018 · 2026-09-24 · A disk floor, because running out of space mid-run does not fail cleanly

**Why now.** The confirmatory run writes roughly 1.2 GB of raw outputs over about two weeks
onto a volume that is **98% full — 46 GB free of 1.9 TB**. Ollama's model store alone accounts
for 89 GB of that.

Running out of space is not a clean failure. A half-written JSONL line, a truncated CSV row or
a resume key that never reached the disk would be discovered days later, and would look
exactly like the silent corruption this project has now found four times. The run should stop
while there is still room to stop safely.

### The rule

`assert_disk_space()` in `scoring.py` — the shared path, so both runners get the same check
through the same function and cannot drift apart on what "too full" means. **Floor: 20 GB.**

- The **2×2 runner** checks inside `verify_environment()`, which already runs before each arm
  and every 100 events, and logs free space alongside the environment fingerprint. A breach
  raises and `main()` exits **3**.
- The **forward test** checks before scoring and returns **3**, logging the halt and writing
  `DISK_FULL` to the health line, so a full disk is visible from off-machine in the pushed
  health log rather than only in a local traceback.
- `scripts/run_forward_test.bat` documents rc=3 alongside rc=2.

The disk check runs **before** the version lock, because it is the cheaper of the two and
because it is the one that makes every other artefact — logs, resume keys, the environment log
itself — unwritable when it fails.

Both halts are resumable by construction: nothing is scored, and the per-event resume keys
(D-014 item b) mean re-running after freeing space continues exactly where it stopped.

### Models installed, for the record

Only `gemma4:26b-a4b-it-qat` (15 GB, digest `2dd70431afed`) is used by this study. The other
fourteen models on the machine belong to other projects or to earlier work; the listed sizes
sum to ~111 GB against an 89 GB store, so blobs are shared and deleting a model reclaims less
than its listed size. Which to remove is the author's decision, not this study's — several are
JARVIS's brain.

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-24 | §2.1 | Both runners halt cleanly below 20 GB free (exit 3) | The volume is 98% full and the run writes ~1.2 GB; a mid-write exhaustion corrupts silently | None — pre-confirmatory |

---

## D-019 · 2026-09-24 · A dynamic output budget was tested and REJECTED. `num_predict` stays at 6,000

**Recorded because it was tested, not because it changed anything.** A rejected change belongs
in the log as firmly as an adopted one, or it gets proposed again by whoever next notices the
unused context.

### The proposal

Two of twenty rehearsal scorings (D-016) went `UNPARSEABLE` by exhausting `num_predict=6000`
while thinking. With the D-012 trim enforced at 30,000 tokens, a worst-case prompt plus 6,000
uses 35,738 of the 40,960 context and an ordinary one uses far less, so thousands of context
tokens sit unused while events are lost to the cap. The candidate rule:

    num_predict = max(6000, min(12000, num_ctx - prompt_tokens - 512))

### The test

All **20 pilot events**, each scored **twice on byte-identical text** (cached, so the only
thing differing between the two runs is the cap). Pilot events are excluded from every
confirmatory analysis. `src/test_numpredict.py`,
`results/numpredict_test/numpredict_comparison.csv`.

| | unparseable | `done_reason=length` | total runtime | median |
|---|---|---|---|---|
| fixed 6,000 | **2/20** | 2 | **10.1 min** | 19.1 s |
| dynamic (10,710–12,000) | **2/20** | 2 | **17.3 min** | 19.1 s |

### The result: no gain, +71% runtime

**Not one event was rescued.** Both cap-hitting events consumed every token they were given
and still emitted no `SIGNAL` line:

| Event | Release tokens | fixed 6,000 | dynamic | cost |
|---|---|---|---|---|
| ALB | 8,753 | UNPARSEABLE, gen 6,000, 84.6 s | UNPARSEABLE, gen **12,000**, 166.1 s | 2.0× |
| PLD | 29,648 | UNPARSEABLE, gen 6,000, 149.8 s | UNPARSEABLE, gen **10,710**, 419.2 s | 2.8× |

These are not events that needed *slightly* more room. On these releases the model's thinking
does not terminate within any budget the context allows, so **every finite cap fails and a
larger one only costs more** — and it costs it on precisely the slowest events, which is the
worst place to spend it. The median is unchanged at 19.1 s because the entire cost lands on
the two failures.

Scaled to the confirmatory run, +71% would turn the measured ~213 hours into ~364 hours: from
about 9 days to about 15, to recover nothing.

### Three further reasons not to adopt

1. **The cap is not a neutral parameter.** Generation changed on events that completed
   comfortably: SRE 1,174 → 1,307 tokens, WRB 2,027 → 1,530. Adopting the rule would perturb
   what the model produces on *every* event in order to fix none.
2. **It runs the context to the edge.** Peak context used rose from 35,738 to **40,448 of
   40,960** — 512 tokens of headroom, exactly the safety margin and nothing more. Given that
   Ollama's response to exceeding `num_ctx` is to silently discard half the prompt and report
   the survivors as if nothing happened (D-012), operating 512 tokens from that boundary
   trades a visible, honest failure for the invisible kind.
3. **The floor was never needed.** The computed cap ranged 10,710–12,000 and the 6,000 floor
   engaged on 0 of 20 events, confirming the trim already guarantees it — as expected, and
   now measured.

### Decision

**`num_predict` stays at 6,000.** The configuration of D-011 and prereg §13.2 is unchanged and
needs no refreeze on this account.

The ~10% unparseable rate is therefore accepted and reported as **non-random missingness**,
consistent with D-003, D-010 item 4 and the D-016 continuation: correlated with thinking
length rather than release length (ALB is an ordinary 8,753-token release), and unstable under
trivial input perturbation. It is not treated as random dropout, and the affected events are
reported with their `done_reason` so a reader can see exactly what happened rather than
finding a blank cell.

### Incidental

One engine error occurred during the test (NDAQ, fixed-cap run, the fourth transient CUDA
fault observed) and is counted separately from unparseable answers — conflating an engine
fault with a model outcome would have credited the dynamic cap with a rescue it did not
perform. The test script calls Ollama directly and therefore does **not** benefit from
D-016's retry, which is why the fault surfaced as a bare error here.

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-24 | §2 | **No change.** Dynamic `num_predict` tested on 20 events and rejected | Rescued 0 of 2 unparseable events at +71% runtime; perturbs completing events; leaves 512 tokens of context headroom | None |
| 2026-09-24 | §11 | Unparseable rate (~10%) accepted and reported as non-random missingness | No configuration within the context window removes it | None |

---

## D-020 · 2026-09-24 · The launch gate was prose. Now it is code. And engine faults become their own metric

### (1) The forward test would have scored on 1 October regardless

prereg §13.3 says confirmatory scoring does not begin until the pre-registration has been
archived by parties independent of the author. **Nothing read that sentence.** The forward
test gated on disk (D-018, rc=3) and the version lock (§2.1, rc=2) and nothing else.

So on 1 October, with the archive links still `[PENDING]`, the job would have polled EDGAR,
found candidates, scored them, hashed them, pushed them and reported success — producing
confirmatory events whose only timestamp was GitHub's own commit date. The forward test is the
single arm whose entire value rests on a date somebody else can vouch for, and it was the one
arm with no check.

This is the same failure shape as D-014 item e, where D-011 claimed the runner checked the GPU
lock and no code did. **A rule written only in prose is not a rule**, and this project has now
produced that mistake twice.

**The gate.** `data/archive_record.json` holds the submissions; `assert_archives_recorded()`
in `scoring.py` — the shared path, so both runners gate identically — requires:

- at least one Wayback capture, identified by a `web.archive.org/web/…` link
- at least one Software Heritage snapshot, identified by an `swh:1:snp:…` id
- a `date_submitted` on every entry
- **and** that prereg §13.3's table contains no `[PENDING]` rows

The last one is a cross-check between two representations that must both be complete, which is
harder to fool than either alone. The file ships empty on purpose: the gate is **closed right
now**, and `src/test_runner.py` asserts that the runner refuses to score and writes nothing
when it fires.

**What happens on 1 October, by exit code:**

| Condition | Forward test |
|---|---|
| Archives recorded | scores normally, logs `independent timestamp OK (n Wayback, m Software Heritage)` |
| Archives pending | **halts without scoring, rc=4**, health line `ARCHIVE_PENDING`, pushed so the gap is visible off-machine |
| Archives pending, `--allow-missing-archives` | scores, and marks **every** event `VOID_NO_INDEPENDENT_TIMESTAMP` |

**Why an override exists at all.** Halting is not free. A release scored after the next open
is worthless to the forward test, so a day spent halted is a day of events permanently lost —
they cannot be scored retrospectively and claimed as prospective. The override therefore
exists, but it labels what it produces instead of pretending, exactly as §2.1's
`VOID_ENVIRONMENT_CHANGE` does. Voided events are retained as evidence and excluded from the
confirmatory forward-test analysis.

**The default is to halt**, because the season runs 1 October to 30 November and losing a few
early days costs far less than weakening the arm that the whole prospective claim rests on.

### (2) Transient engine faults are now their own reported metric

The transient CUDA faults of D-016 were being counted nowhere. Four have now been observed
across rehearsals and the D-019 experiment, at a volatile rate — 15% of 20 scorings in one
rehearsal, 0% in the next. Folding them into `unparseable` would let a bad GPU night look like
a property of the earnings releases.

Five separate numbers, written to the per-arm progress file continuously and reported at
**Gate 4** (prereg §9.2):

| Metric | Meaning |
|---|---|
| `unparseable` | The model answered but emitted no `SIGNAL` line — in practice exhausted `num_predict` while thinking (D-019). **The model.** |
| `transient_faults_seen` | Events that hit at least one transient engine fault. **The machine.** |
| `recovered_on_retry` | Of those, how many then scored — the D-016 retry earning its keep, and the number that shows how much of the fault rate was invisible before the retry existed |
| `gave_up_transient` | Abandoned after three attempts; reported as lost rather than left as a blank cell |
| `EXCLUDED_FETCH` | The release could not be retrieved from EDGAR |

Rows now carry `engine_retries` and `engine_retry_detail`, and a give-up on a transient fault
is stored as `SCORE_FAILED_TRANSIENT` rather than `SCORE_FAILED`, so the two are separable
after the fact without parsing error strings. `test_fault_counts_separate_machine_from_model`
pins the arithmetic, including that `unparseable` excludes engine faults.

The rate is reported as a **measured count per arm, never as an assumed rate** — two rehearsals
of the same twenty events gave 15% and 0%, so any single figure would be fiction.

### Housekeeping

Disk: the author removed six unused models. The store went 89 GB → 52 GB and free space 46 GB
→ 113 GB. The study model `gemma4:26b-a4b-it-qat` digest `2dd70431afed` is intact, the
environment fingerprint is unchanged, and the JARVIS models were deliberately kept.

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-24 | §13.3 | The archive gate is enforced in code; both runners halt (rc=4) without a recorded independent timestamp | The rule existed only as prose; the forward test would have scored on 1 October regardless | None — nothing confirmatory scored |
| 2026-09-24 | §4.8 | `--allow-missing-archives` scores but marks every event `VOID_NO_INDEPENDENT_TIMESTAMP` | A halted day loses its events permanently; the override labels rather than pretends | None |
| 2026-09-24 | §9.2 (new) | Missingness reported by cause, engine faults separate from unparseable, at Gate 4 | Folding a GPU fault into an unparseable count would attribute a machine failure to the releases | None |

---

## D-021 · 2026-09-26 · The archive gate is satisfied. And the manifest was hashing the wrong bytes

### The gate is open

Seven captures recorded in `data/archive_record.json` and prereg §13.3, and **verified rather
than trusted**:

| Service | Verified how |
|---|---|
| Wayback × 5 raw files | fetched back and compared byte for byte against what the mirror serves — all five identical |
| Wayback × 1 repo home page | HTTP 200; a rendered page, so no byte hash applies |
| Software Heritage | visit 1 `status=full`; snapshot `refs/heads/master` → `5592bdd276…`, **identical to the public mirror HEAD** |

Snapshot: `swh:1:snp:a3d93f0fa398f75017d5bf1e11b847dc84b3c5cc`, save request 421991555, visit
2026-09-26T17:17:28Z. The archived mirror commit `5592bdd276…` is the mirror of private commit
`b4364e4ac92a`, containing deviations through **D-020**, with **zero confirmatory events
scored**.

`assert_archives_recorded()` now returns "6 Wayback, 1 Software Heritage" and both runners
pass their pre-flight. The forward test will score on 1 October.

### Two things the verification exposed

**1. `FREEZE_HASHES.txt` was hashing the wrong bytes.** Its published-file section was computed
with `sha256sum` over the **Windows working tree**, which is CRLF, while the repository is
cloned with `core.autocrlf=true` — so git stores LF and `raw.githubusercontent.com` serves LF.
Every one of the four published-file hashes therefore disagreed with the file anyone would
actually download:

| File | manifest said (CRLF) | actually served (LF) |
|---|---|---|
| `prereg.md` | `861cd4d0ee40dbfb…` | `28996b0c059abe0c…` |
| `deviations.md` | `066547950b24e020…` | `654318f02c3578ba…` |
| `scrubber_freeze.json` | `a4bc36121cda68e2…` | `2f5db4a4150a6c35…` |
| `excluded_event_ids.csv` | `051c4e97242f7009…` | `13edbf15e9508467…` |

This is the **third** appearance of the same CRLF defect: §13.1's hash table (D-013), then
`scrubber_freeze.json` (D-013), now the published manifest. In each case the hash was taken
from the working tree on a Windows machine and could not be reproduced by anybody else — which
defeats the entire purpose of publishing a hash.

`write_manifest()` now normalises CRLF to LF before hashing, and the manifest states the two
commands that reproduce it (`git show HEAD:<file> | sha256sum` in a clone, or
`curl -sL --compressed <raw URL> | sha256sum`) together with an explicit warning that
`sha256sum <file>` on a Windows checkout will not match. To make the mistake impossible to
reintroduce through an escaped-string slip, the normalisation is written as
`replace(bytes([13, 10]), bytes([10]))` — byte values, no escape sequences.

**2. Wayback serves the original compressed payload.** A `web.archive.org/web/<ts>id_/…` fetch
returns the resource exactly as captured, **including its gzip Content-Encoding**. A naive
`curl -sL -o file` therefore saves gzip bytes and every hash comparison fails for a reason that
has nothing to do with the archive. The first verification pass produced five mismatches from
precisely this, and the tell was the file starting `1f 8b 08`. The fetch must pass
`--compressed` (or pipe through `gunzip`). Recorded in `archive_record.json` so the next person
to verify these links does not lose an hour to it.

### The circularity, stated plainly

Recording the archive links changes the very files that were archived, so **no capture can
ever contain its own provenance.** The 2026-09-26 captures are therefore the *evidential*
ones: they predate this record and predate every result. Captures taken after the links are
written in are housekeeping — they let a reader who lands on the archived document see where it
was archived — and are not what the "pre" in pre-registration rests on.

Files changed by recording, and therefore needing a fresh capture for that housekeeping
purpose: `prereg.md`, `deviations.md`, `scrubber_freeze.json`, `FREEZE_HASHES.txt`, and the
repository home page. `excluded_event_ids.csv` is unchanged and its capture stands.

### One test rewritten

`test_the_archive_gate_stops_the_runner` originally asserted that the real archive record was
incomplete. That passed only while the archives were pending and failed the moment they were
recorded. It now points the gate at an empty temporary record and checks both directions —
empty closes it, complete opens it — so it keeps protecting the mechanism after launch. **A
test that expires is a test that stops protecting anything.**

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-26 | §13.3 | Seven archive captures recorded and verified; the gate is satisfied | Independent timestamps now exist and were checked, not assumed | None |
| 2026-09-26 | §13.1 | `FREEZE_HASHES.txt` published-file hashes normalised to LF, with reproduction commands | The CRLF hashes matched nothing anyone could download — third instance of this defect | None |

---

## D-022 · 2026-09-26 · Tiingo: an age problem and a licence problem. Prices embargoed; the 2×2 launches without them

### Two separate problems, found by reading the terms

**1. Age.** Tiingo's Terms of Use require the legal capacity to contract. The author is 15.
The account therefore moves to a parent's account, exactly as OSF did (D-013). This is the
**second** blocker in this project caused by an age requirement, and the pattern is now worth
stating: check the terms of a service before building a dependency on it, not after.

**2. Licence.** The free plan grants **personal and internal use with no redistribution.**
That is a constraint on what the *paper* may contain, not merely on what the repository may
hold. Whether publishing aggregate statistics derived from the data is permitted is being put
to Tiingo in writing. Until they answer, the conservative reading governs.

### The embargo

**Nothing price-derived appears in anything public** — paper, poster, repository, slide —
until Tiingo confirms in writing. Enforced in `scripts/sync_public_prereg.py`, not left to
vigilance, because a licence breach discovered after publication cannot be taken back:

- **By filename**, on both the source and the published path, so renaming on the way out does
  not launder it: `price`, `tiingo`, `ohlc`, `bars`, `quotes`, `adjclose`, `return`, `abret`,
  `spy`.
- **By content**, two ways. Tiingo's own field names (`adjClose`, `adjOpen`, `divCash`,
  `splitFactor`, …) refuse on **two or more distinct hits**, because one alone can be prose —
  this document discusses `adjClose`. A strict list refuses on a **single hit**, for things
  that cannot appear innocently: `abret_21d`-style output columns, `api.tiingo.com`, a token.
- **Credentials**: a `TIINGO_API_KEY`-shaped assignment is refused outright.

`src/test_price_embargo.py` actively tries to publish price data four ways — a real Tiingo
JSON payload, a file named like prices, a derived per-event return column, and a price file
renamed on the way out — and requires each to be refused. **A guard nobody has tried to get
past is not a guard.**

It also tests the converse, which matters as much: prose mentioning Tiingo and `adjClose` once
must still publish. If the embargo blocked the pre-registration from describing its own data
source, the first person to hit that would switch the check off, and then it would protect
nothing.

The first version of the guard **failed one of its own tests**: `event_returns.csv` slipped
through, because `\breturns?\b` never matches inside `event_returns` — an underscore is a word
character, so there is no boundary before "returns". Caught before anything was published.

### The launch gate loses a condition it never needed

GPU scoring reads EDGAR text and writes signals. **It does not touch prices.** Realized
abnormal returns are joined at analysis time, which is after all four arms complete (D-014
item 4). Holding the 2×2 hostage to a Tiingo question would cost days of GPU time to protect
nothing.

So the gate splits:

| Gate | Blocks | Condition |
|---|---|---|
| **Scoring gate** | the 2×2 and the forward test | archive links recorded (D-020) and disk above 20 GB (D-018) — **both satisfied** |
| **Price gate** | the price pull, and any public artefact containing price-derived data | a parent-held Tiingo account, and Tiingo's written answer on aggregate statistics — **neither satisfied** |

This is a narrowing of scope, not a relaxation of standards: nothing that was checked before
is unchecked now. Prices were never an input to a signal, and §4.9's rule that no analysis
runs until all four arms finish is unchanged.

**Consequence if Tiingo answers no:** the study still has its signals, its H3 probes and its
scrubbed/unscrubbed gaps. What it would lose is the ability to publish realized-return
results from Tiingo data, which would mean sourcing prices elsewhere before publication. That
is a publication problem, not a scoring problem, and it is better discovered now than after
fourteen days of GPU time.

| Date | Prereg § | Change | Reason | Re-run required |
|---|---|---|---|---|
| 2026-09-26 | §3.1 | Tiingo account moves to a parent; free-plan licence treated as no-redistribution until Tiingo answers | Terms require legal capacity to contract; author is 15 | None |
| 2026-09-26 | §13.3 | The mirror sync refuses price data by filename, by content, and by credential shape | A licence breach cannot be withdrawn once published | None |
| 2026-09-26 | §4.8 | Tiingo removed from the scoring gate; retained as a gate on the price pull and on any public price-derived artefact | Scoring never reads prices; returns are joined after all arms finish | None |
