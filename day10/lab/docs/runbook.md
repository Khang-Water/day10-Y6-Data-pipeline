# Runbook - Lab Day 10 Incident Response

---

## Symptom

- Agent answers refund window as `14 days` instead of `7 working days`.
- Retrieval output includes stale HR text (`10 annual leave days`) for 2026 questions.
- Freshness command returns `FAIL` for latest manifest.

---

## Detection

- ETL logs show expectation failure, especially `refund_no_stale_14d_window` or `hr_leave_no_stale_10d_annual`.
- Retrieval eval reports `hits_forbidden=yes` in `artifacts/eval/*.csv` or `grading_run.jsonl`.
- `python etl_pipeline.py freshness --manifest ...` returns `FAIL` with `freshness_sla_exceeded`.

---

## Diagnosis

| Step | Action | Expected result |
|------|--------|-----------------|
| 1 | Check latest `artifacts/manifests/manifest_<run_id>.json` | `run_id`, record counts, and `latest_exported_at` are present |
| 2 | Inspect `artifacts/quarantine/quarantine_<run_id>.csv` | bad rows are quarantined with explicit `reason` |
| 3 | Inspect `artifacts/logs/run_<run_id>.log` | expectation status and embed summary are visible |
| 4 | Run `python eval_retrieval.py --out artifacts/eval/before_after_eval.csv` | `contains_expected=yes` and `hits_forbidden=no` for key questions |
| 5 | Run `python grading_run.py --out artifacts/eval/grading_run.jsonl` | all `gq_d10_01..03` present and pass merit checks |

---

## Mitigation

- Standard recovery: run `python etl_pipeline.py run` (without inject flags) to rebuild cleaned snapshot and republish embedding.
- If bad data was intentionally injected, rerun without `--no-refund-fix` and without `--skip-validate`.
- If freshness fails due to old export, trigger upstream export and rerun ETL.
- If required, isolate serving by switching `CHROMA_COLLECTION` to a safe baseline until quality is restored.

---

## Prevention

- Keep expectations in `halt` for stale refund and stale HR version rules.
- Monitor freshness in CI or scheduled job and alert `#data-observability` on `FAIL`.
- Keep contract (`contracts/data_contract.yaml`) aligned when adding new `doc_id` or source files.
- Retain per-run logs/manifests/eval artifacts to support incident timeline and root-cause analysis.
