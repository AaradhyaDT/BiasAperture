# Schema Lock — Milestone M1 (WP1)

Locked per WBS 1.1/1.2 (`report/src/chapters/systemArchitectureAndMethodology.tex`
§Work Breakdown Structure). This is the reference record; the enforced
version lives in `src/bias_aperture/schema.py`.

## Classifier baseline

**dchen236/FairFace** (and `joojs/fairface`) — inference fork of the FairFace
paper's official pretrained ResNet-34 (`fairface_alldata_20191111.pt`, with
`res34_fair_align_multi_7_20190809.pt` as alternative checkpoint), **race_7**
variant (not race_4).

Rationale: matches FairFace's own 7 race groups (White, Black, Indian,
East Asian, Southeast Asian, Middle Eastern, Latino), the dataset
`requirements.tex` FR-001 names as primary (97,698 released images on disk:
86,744 train + 10,954 val), and gives finer-grained subgroup resolution than
race_4 for the fairness metrics in FR-003. This is also the "public pretrained
checkpoint and inference script" the report's descoping table (cutlist #4)
already assumes as the fallback if in-process inference is cut — so locking
on it now means WP2's predictions-file path and the descope fallback are the
same artifact, not two.

## Internal schema (FR-001)

| Field              | Type                          | Source                                                  |
|---------------------|--------------------------------|----------------------------------------------------------|
| `image_id`          | `str`                          | FairFace `face_name_align` column                        |
| `race`               | one of 7 labels below          | FairFace `race` column, race_7 model                      |
| `gender`             | one of 2 labels below          | FairFace `gender` column                                  |
| `age`                | one of 9 labels below          | FairFace `age` column                                     |
| `true_label`         | `str`, audit-task-specific     | caller-specified column at ingestion                      |
| `predicted_label`    | `str`, audit-task-specific     | caller-specified column at ingestion                      |

`true_label`/`predicted_label` are audit-specific (e.g. gender
classification as the audited task, race/age as protected axes) — not
fixed vocabulary, unlike the three demographic fields.

**Race labels (7):** White, Black, Latino_Hispanic, East Asian,
Southeast Asian, Indian, Middle Eastern

**Gender labels (2):** Male, Female

**Age labels (9):** 0-2, 3-9, 10-19, 20-29, 30-39, 40-49, 50-59, 60-69, 70+

## Detection engine output schema (FR-003/FR-004)

Per row: `metric_name`, `subgroup`, `subgroup_sample_size`,
`metric_value`, `ci_lower`, `ci_upper`, `p_value`, `insufficient_sample`.

**Statistical Extension Fields (Non-breaking, Phase 2):**
- `raw_p_value`: Unadjusted asymptotic p-value from metric-specific test.
- `adjusted_p_value`: FWER-adjusted p-value via Holm–Bonferroni step-down.
- `hypothesis_family`: Test grouping (`"selection_rate"`, `"tpr_conditional"`, `"conditional_odds"`).
- `adjustment_method`: Adjustment label (e.g. `"holm_bonferroni"`).

`metric_name` is one of: `demographic_parity_difference`,
`equalized_odds_difference`, `equal_opportunity_difference`,
`disparate_impact_ratio` (the Core Four, FR-003).

**NFR-003 guard (enforced at construction, not just by convention):** any
row with `subgroup_sample_size < 30` must have `insufficient_sample=True`
and `metric_value=None` — a flagged row is never allowed to carry a
computed value. This is enforced in `MetricResult.__post_init__`, not
left to the caller to remember.

`subgroup`'s composite-key format for intersectional rows (e.g.
`race=Black&gender=Female`) is **not** locked at M1 — only the field's
presence is. Format finalized in WP4.

## Constants locked with the schema

- `MIN_SUBGROUP_SAMPLE_SIZE = 30` (NFR-003)
- `ALPHA = 0.05` (NFR-001)
- `MIN_BOOTSTRAP_RESAMPLES = 1000` (NFR-002)

## What this unblocks

WP2 (Stream A) and WP3 (Stream B) can now start in parallel — both build
against the field set and label vocabularies above. Stream B's mock
metrics dictionary must validate against the `MetricResult` shape, not
an approximation of it, so the WP5 mock-to-real swap stays mechanical.

## Change policy

Any change to field names, dtypes, or label vocabularies after this
lock is a breaking change to both streams and must be re-synced with
whoever owns Stream A and Stream B before merging.
