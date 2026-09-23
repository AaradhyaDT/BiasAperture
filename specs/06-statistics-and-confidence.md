# 06 - Statistics and Confidence

**Status:** Implemented in the fairness/statistics layer; end-to-end report inspection remains verification work

Every reported disparity must carry sample size, a p-value or an explicit insufficient-sample flag, and a 95% confidence interval when the metric is defined.

## Significance & Metric-Specific Hypothesis Testing

Hypothesis tests use `ALPHA = 0.05` and are tailored specifically to the mathematical definition of each metric:
- **Demographic Parity Difference (DPD) & Disparate Impact Ratio (DIR)**: Pearson's $\chi^2$ test of independence on overall positive prediction rates across demographic groups ($\text{family} = \text{"selection\_rate"}$).
- **Equal Opportunity Difference (EOP)**: Conditional $\chi^2$ test of independence restricted strictly to the positive ground-truth stratum $Y_{\text{true}} = 1$ to test $H_0: \text{TPR}_a = \text{TPR}_b$ ($\text{family} = \text{"tpr\_conditional"}$).
- **Equalized Odds Difference (EOD)**: Joint union-intersection test combining conditional $\chi^2$ on $Y_{\text{true}} = 1$ (TPR equality) and $Y_{\text{true}} = 0$ (FPR equality) via Bonferroni significance bounding ($\text{family} = \text{"conditional\_odds"}$).

Multiple-testing control is enforced via the Holm–Bonferroni step-down procedure within hypothesis families, recording both `raw_p_value` and `adjusted_p_value` in the `MetricResult` records. Exact p-values are preserved to 4 decimal places rather than reduced to binary pass/fail indicators.

## Bootstrap Confidence Intervals

Confidence intervals use at least `MIN_BOOTSTRAP_RESAMPLES = 1000` resamples. Resampling is stratified within eligible demographic groups to preserve observed subgroup allocations:
- **Global Metrics**: BCa bootstrap confidence intervals. For cohorts with $n > 300$, a delete-$d$ subsampled block jackknife approximation ($d = \max(1, n // 100)$) maintains computational tractability while preserving asymptotic consistency. If acceleration or resampling degenerates, the engine falls back to empirical percentiles or the clamped point estimate.
- **Subgroup Metrics**: Computed via stratified bootstrap percentile intervals evaluating each subgroup's disparity relative to the cohort baseline, eliminating arbitrary heuristic windows.

## Sample guard

Any subgroup with `n < 30` is not reportable as a computed metric. Its `MetricResult` must set `insufficient_sample=True` and leave computed values unset. This is an integrity rule, not a presentation preference.

See [schema-lock-m1.md](../docs/schema-lock-m1.md), [LOW_LEVEL_SPECIFICATION.md](../docs/research/LOW_LEVEL_SPECIFICATION.md), and [verification](09-verification.md).
