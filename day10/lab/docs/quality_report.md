# Quality Report - Lab Day 10

**run_id (inject):** `completion-inject`  
**run_id (after fix):** `completion-good`  
**Date:** 2026-04-15

---

## 1. Summary metrics

| Metric | Injected bad run | After fix run | Notes |
|--------|------------------|---------------|-------|
| `raw_records` | 10 | 10 | Same raw export source |
| `cleaned_records` | 6 | 6 | Stable publish volume |
| `quarantine_records` | 4 | 4 | Same quarantine volume |
| `Expectation halt?` | Yes (`refund_no_stale_14d_window` fail) | No | Inject run used `--skip-validate` intentionally |

Evidence:
- `artifacts/logs/run_completion-inject.log`
- `artifacts/logs/run_completion-good.log`

---

## 2. Before/after retrieval evidence

Files:
- `artifacts/eval/after_inject_bad.csv`
- `artifacts/eval/after_fix.csv`

Key question: `q_refund_window`

- Injected run (`after_inject_bad.csv`): `contains_expected=yes`, `hits_forbidden=yes`
- After fix (`after_fix.csv`): `contains_expected=yes`, `hits_forbidden=no`

Interpretation:
- The bad run still retrieved a correct phrase, but top-k also contained stale content (forbidden 14-day window), which is exactly the observability risk this lab targets.
- The fixed run removed stale retrieval signal (`hits_forbidden=no`) while preserving expected answerability.

Merit question: `q_leave_version`

- Injected run: `contains_expected=yes`, `hits_forbidden=no`, `top1_doc_expected=yes`
- After fix run: `contains_expected=yes`, `hits_forbidden=no`, `top1_doc_expected=yes`

Interpretation:
- HR version guard remained stable in both runs; refund corruption injection did not regress HR retrieval.

---

## 3. Freshness and monitoring

Manifest checks:
- `manifest_completion-inject.json`: `freshness_check=FAIL`
- `manifest_completion-good.json`: `freshness_check=FAIL`

Reason:
- `latest_exported_at=2026-04-10T08:00:00` exceeded `FRESHNESS_SLA_HOURS=24` at execution time.

Operational meaning:
- Pipeline quality can pass while freshness fails; this is expected because freshness measures data age, not rule correctness.

---

## 4. Corruption inject scenario

Inject command:

```bash
python etl_pipeline.py run --run-id completion-inject --no-refund-fix --skip-validate
```

Detection signals:
- Expectation fail: `refund_no_stale_14d_window` (`violations=1`)
- Retrieval contamination: `q_refund_window` has `hits_forbidden=yes`

Recovery command:

```bash
python etl_pipeline.py run --run-id completion-good
```

---

## 5. Limitations and next actions

- Freshness currently depends on manifest timestamp only; no upstream watermark integration yet.
- Model loading prints Hugging Face cache permission warnings in this environment; pipeline still succeeds.
- Next action: add CI job to run freshness and grading checks on schedule and publish alerts to `#data-observability`.
