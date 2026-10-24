# Reflection Brief — Evaluation and Observability Capstone

**Name:** Gomathi Namashamgari
**Date:** 2026-10-24

> Answers are grounded in the saved test results and run artifacts. Live API and routing results that could not be generated are explicitly identified.

---

## 0. Environment

| Field | Value |
|---|---|
| OS & version | Linux x86_64, kernel 6.6.97+ |
| Python version | Python 3.13.0 |
| Date run | 2026-10-24 |
| Ran any system live? (which) | No successful live API run. System 1 was attempted but failed because Anthropic authentication was not configured. Systems 2 and 3 were evaluated using cached/replay or offline evidence. |

---

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 45 passed, 3 skipped (`01-policy-pipeline/tests.txt`) |
| Routing output file | `01-policy-pipeline/routing_decisions.json` contains two offline test-fixture decisions generated using the project's routing function. It is not live pipeline output; the live run failed because Anthropic authentication was unavailable. |
| auto_approve / human_review / spot_check counts | Offline fixture only: 1 `auto_approve`, 1 `human_review`, 0 `spot_check`. These counts describe the two test fixtures, not live pipeline results. |

**1a. Retry boundary.** From your perturbation run (a required field removed), paste the escalation record. How many API calls did the system make, and why is retrying a futile case worse than escalating it?

> The original policy produced a `PolicyExtraction` with one recorded client call. After the `ENDORSEMENTS` section was removed from the modified copy, the recorded response returned `endorsements=None`. The resulting escalation record was `RetryFutileEscalation(policy_id='POL-2025-001', field='endorsements', category='missing_source', detected_pattern='endorsements_absent', reason="Field 'endorsements' returned null — the source document does not contain this information. Retry is futile; escalate to human review.")`. The perturbed run made **one recorded client call**, despite `max_retries=3`, because `extract_with_retry()` immediately returns when validation identifies `missing_source`. These were deterministic recorded-response experiments, not live API calls. Retrying is futile when the required source information is absent: it wastes calls and risks encouraging fabricated values. Escalation preserves the missing-data signal and allows human review.

**1b. Reading the router.** Pick one `human_review` record from your routing output. Which of the three signals (confidence, reviewer, integration) sent it to a human? If you had trusted the model's confidence alone, what would have happened?

> In the offline fixture `POL-B` in `01-policy-pipeline/routing_decisions.json`, `premium_amount` confidence is 0.5, below the 0.90 threshold. The record is routed to `human_review`, and `fields_below_threshold` identifies `premium_amount`. The fixture has no reviewer disagreements or integration failures, so low confidence is the recorded routing signal. This is a test-fixture example, not a live pipeline result.

**1c. Where the aggregate lies.** Run the calibration snippet. Quote the one cell whose accuracy lags its confidence, plus the overall figure. What does slicing by `policy_type × field` catch that a single number hides?

> In `01-policy-pipeline/calibration-report.txt`, the `umbrella / exclusions` cell has `n=2`, `conf=0.93`, `acc=0.00`, and `brier=0.865`. The overall figure is `OVERALL brier=0.291`. This slice shows that a high-confidence prediction can be wrong for a particular policy-type/field combination even when an aggregate score hides that weakness. The sample contains only two cases, so it is a warning signal rather than proof of general performance.

---

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 passed (`02-mortgage-extraction/tests.txt`) |
| Document run | `02-mortgage-extraction/extract-run.txt` |
| Classified type | Appraisal document for the square-footage example; income-verification document for the missing-bonus and income-mismatch examples. |

**2a. Two guarantees.** Paste your discrepancy-run output. Tool use already forces valid JSON, yet the validator still catches a bad sum. Why are these two different guarantees? Name one error each cannot catch.

> From `02-mortgage-extraction/discrepancy-run.txt`, the extraction reports `stated_monthly_total: 10892.17`. Validation reports `consistent: false`, with `field: total_monthly_income`, `calculated: 9642.17`, `stated: 10892.17`, and `delta: -1250.0`. Structured tool output enforces the expected data shape and types, but it does not guarantee that the numbers agree mathematically. The deterministic validator catches the arithmetic inconsistency. Schema validation alone might not catch a validly typed but incorrect total; arithmetic validation alone might not catch a wrong borrower name or a value extracted from the wrong document.

**2b. Refusing to fabricate.** Run on a document missing a field. Paste that field's output. Why null instead of an invented value? Point to the schema choice that allows it.

> In `02-mortgage-extraction/extract-run.txt`, the missing-bonus example has `"bonus_monthly": null` and `"bonus_ytd": null`. Null preserves the fact that the document does not provide those values instead of inventing a number or assuming the bonus is zero. The extraction schema permits these fields to be nullable. The validation output is `consistent: true` with no discrepancies, which does not mean the missing bonus was known to be zero.

**2c. Normalization.** Quote one field where the source text and extracted value differ in format ("about 2,400 sq ft" → `2400`). Why normalize at extraction time rather than downstream?

> In `02-mortgage-extraction/extract-run.txt`, the informal appraisal's gross living area is extracted as `"gross_living_area_sqft": 2400`. The source describes the area informally as approximately 2,400 square feet, while the extracted field is a numeric value. Normalizing at extraction time gives downstream validation and reporting a consistent type and unit, avoiding repeated parsing and format errors.

---

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 passed (`03-supply-chain/tests.txt`) |
| Briefing file | `03-supply-chain/briefing.md` contains the normal offline briefing; `03-supply-chain/investigation-run.txt` contains the captured investigation run, and `03-supply-chain/timeout-run.txt` records the simulated-timeout run. |
| Section the conflict landed in | `Contested`, for `on_time_delivery_rate`. |

**3a. Annotate, don't arbitrate.** Quote one conflicting-metric pair from your briefing — both values, sources, dates. Give one way a reader is better served by the preserved conflict than by a single reconciled number.

> In `03-supply-chain/investigation-run.txt`, `on_time_delivery_rate` is reported as 95.0 percent by `supplier_audit` as of 2026-04-10 and 78.0 percent by `logistics` as of 2026-04-05. The briefing places the metric in `Contested` and escalates it as a high-impact conflict. Preserving both values and their provenance lets a reader investigate the difference rather than trusting an unexplained average or silently selecting one source.

**3b. Source goes dark.** Run with `--simulate-timeout`. Paste the part of the briefing showing the failed source. How is "unreachable" handled differently from "nothing to report," and why does the run still finish?

> In `03-supply-chain/timeout-run.txt`, the briefing states `Sources unavailable: logistics unavailable (timeout)`. It marks `late_shipment_count` as `[missing source: timeout reading logistics]`. By contrast, `production_capacity_utilization` is marked `[missing source: no source reported this metric]`. The first means a source was unavailable; the second means no source reported the metric. These are different explanations for missing evidence. The coordinator reports the failure and continues processing available sources, so the investigation still produces a briefing.

**3c. Dates as a guardrail.** Quote two claims about the same supplier with different dates. How does requiring a date stop a time difference from reading as a contradiction?

> The briefing reports `on_time_delivery_rate` as 95.0 percent from `supplier_audit` as of 2026-04-10 and 78.0 percent from `logistics` as of 2026-04-05. These are the same metric for different reporting dates and sources. Requiring dates helps the reader check whether the values refer to different periods or snapshots before concluding that one source is wrong. The dates provide context but do not, by themselves, prove why the values differ.

---

## 4. Synthesis

**4a. One principle.** Name the single moment in your runs (system + artifact) where *evaluate the output, don't trust the model's word* most clearly caught something a trusting design would have shipped.

> System 2, `02-mortgage-extraction/discrepancy-run.txt`: the extraction returned a stated monthly total of 10,892.17, but deterministic validation calculated 9,642.17 and flagged a -1,250.00 discrepancy. A system that trusted the extracted total without checking the arithmetic could have passed an inconsistent result downstream.

**4b. Confidence ≠ correctness.** Pick the system where this mattered most, and explain why using something you observed.

> System 1 illustrates this through `01-policy-pipeline/calibration-report.txt`: the umbrella/exclusions slice had confidence 0.93 but accuracy 0.00 across two samples. This shows that confidence alone is not a guarantee of correctness and that evaluation should be sliced by policy type and field. The sample is small, and the live pipeline was not available to verify current routing outcomes.

**4c. Apply it.** Describe a real workflow where an LLM pulls structured results from messy input. Which pattern — validated retry with escalation, independent review with deterministic routing, or provenance-preserving conflict annotation — would you reach for first, and what would you instrument to know when it broke?

> For extracting claim details from insurance forms, I would start with validated retry and escalation. Schema and business-rule validation can detect malformed or inconsistent fields; a retry can address correctable formatting or consistency errors, while genuinely missing source information should escalate instead of triggering repeated calls. I would instrument validation-failure patterns, retry counts, escalation rates, field-level accuracy, confidence calibration, API errors, and the percentage of records needing human correction. A sudden increase in missing-source escalations or a drop in accuracy for a particular field would indicate that the workflow needs investigation.

---

## Evidence limitations

- System 1's live pipeline run and live routing output could not be generated because Anthropic authentication was unavailable. The saved `routing_decisions.json` is an offline test-fixture artifact, as documented in `01-policy-pipeline/offline-routing-test.txt`.
- The System 2 corrected fixture could not be fully replayed because no cached response existed for the changed input. The corrected and original totals were compared using the project's actual offline validator.
- System 3's normal and simulated-timeout offline runs completed and were captured in their respective text artifacts.
