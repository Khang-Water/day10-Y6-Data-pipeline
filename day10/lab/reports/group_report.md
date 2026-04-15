# Group Report - Lab Day 10: Data Pipeline and Observability

**Group name:** Day10-Y6-Multi-agent  
**Members:** see repository contributors and commit history  
**Submitted on:** 2026-04-15  
**Repo:** `day10/day10-Y6-Multi-agent`

---

## 1. Pipeline overview

Raw input is `data/raw/policy_export_dirty.csv`, representing an exported snapshot before data is published to retrieval. The ETL entrypoint is `etl_pipeline.py` with the flow ingest -> clean -> validate -> embed -> manifest -> freshness check. Every run emits `run_id`, counts (`raw_records`, `cleaned_records`, `quarantine_records`), cleaned/quarantine CSV artifacts, and a manifest. This gives clear lineage and reproducibility for both normal operations and intentional corruption experiments.

The pipeline integrates with Chroma using deterministic `chunk_id` and idempotent upsert. Before upsert, stale IDs not present in the latest cleaned snapshot are pruned. That behavior is critical because stale vectors can remain in top-k even when top-1 looks correct. Our logs show prune activity (`embed_prune_removed=1`) during reruns, which confirms publish-boundary cleanup works.

One-line command for the normal pipeline run:

```bash
python etl_pipeline.py run --run-id completion-good
```

---

## 2. Cleaning and expectation summary

Baseline cleaning rules handled unknown `doc_id`, missing/invalid effective date, stale HR policy version, missing text, duplicate text, and refund-window correction (14 -> 7 working days). The expectation suite enforced hard stops for empty results, empty `doc_id`, stale refund window, invalid ISO date, and stale HR annual leave statement.

### 2a. Metric impact table

| Rule / expectation | Before (inject) | After fix | Evidence |
|-------------------|-----------------|-----------|----------|
| `refund_no_stale_14d_window` (halt) | FAIL (`violations=1`) | OK (`violations=0`) | `artifacts/logs/run_completion-inject.log`, `artifacts/logs/run_completion-good.log` |
| Retrieval contamination check (`q_refund_window`) | `hits_forbidden=yes` | `hits_forbidden=no` | `artifacts/eval/after_inject_bad.csv`, `artifacts/eval/after_fix.csv` |
| Idempotent publish snapshot (prune stale IDs) | `embed_prune_removed=1` | `embed_prune_removed=1` | same run logs |

Expectation failure example and handling:
- Inject command used `--no-refund-fix --skip-validate`, causing expected halt failure on stale refund text.
- Recovery rerun removed inject flags, restored expectation pass, and republished clean embeddings.

---

## 3. Before/after retrieval impact

Inject scenario:

```bash
python etl_pipeline.py run --run-id completion-inject --no-refund-fix --skip-validate
python eval_retrieval.py --out artifacts/eval/after_inject_bad.csv
```

Recovery scenario:

```bash
python etl_pipeline.py run --run-id completion-good
python eval_retrieval.py --out artifacts/eval/after_fix.csv
```

Quantitative result:
- For `q_refund_window`, both runs still had `contains_expected=yes`, but inject run produced `hits_forbidden=yes` while fixed run produced `hits_forbidden=no`.
- This shows why top-k observability matters: answer can look correct even when stale policy still contaminates retrieved context.
- For HR version question (`q_leave_version`), both runs remained stable (`contains_expected=yes`, `hits_forbidden=no`, `top1_doc_expected=yes`), indicating the corruption was isolated to refund policy logic.

---

## 4. Freshness and monitoring

Freshness check on `manifest_completion-good.json` returned `FAIL` because `latest_exported_at=2026-04-10T08:00:00` exceeded `sla_hours=24`. This is expected with static sample export data and demonstrates that freshness monitoring is independent from transform quality. Operationally, the team should treat this as a data-age incident: request a new source export, rerun ETL, and verify freshness status before promoting to production retrieval.

---

## 5. Connection to Day 09

Day 10 ensures Day 09 retrieval agents consume cleaned, version-correct, and observable corpus snapshots. We intentionally use a dedicated collection (`day10_kb`) to isolate data quality experiments from baseline agent demos. This allows controlled corruption tests without polluting other runs.

---

## 6. Remaining risks and next work

- Freshness is based on manifest timestamps only; upstream watermark checks are not implemented yet.
- Contract/schema validation is code-level; no external validator framework is enforced yet.
- Next step is CI automation for ETL run, eval retrieval, grading JSONL, and freshness alert routing.
