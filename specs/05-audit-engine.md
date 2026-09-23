# 05 - Audit Engine

**Status:** Core metrics implemented and tested

## Core Four

For binary or one-vs-rest labels across protected groups:

- **Demographic parity difference:** spread of positive prediction rates.
- **Equalized odds difference:** worst-case spread across TPR and FPR.
- **Equal opportunity difference:** spread of TPR values.
- **Disparate impact ratio:** minimum positive prediction rate divided by the maximum.

Fair values are zero for difference metrics and one for the ratio metric.

## Backend harmonization and failure isolation

Fairlearn and AIF360 are treated as independent strategies. Their known definitional differences are normalized before comparison: equalized odds uses the worst-case TPR/FPR gap, and equal opportunity is stored as an unsigned difference. A cross-validation layer flags mathematical divergences beyond configured thresholds ($|\Delta| > 0.05$ for difference metrics, $|\Delta| > 0.10$ for ratios).

Backend failures or missing dependencies are strictly isolated: if a backend raises an error or is unavailable, the system produces `insufficient_sample=True` records with error reasons and emits `DivergenceAlert` with `difference=nan` rather than silently falling back to an alternate backend.

## Edge cases

If every group has zero positive predictions, DIR is one while the report should warn that absolute selection is absent. If at least one group has positive predictions and another has none, DIR is zero. Undefined rate calculations must be represented as insufficient or invalid evidence, not as fabricated zeros.

See the [low-level mathematical specification](../docs/research/LOW_LEVEL_SPECIFICATION.md) for formulas and the source under [`src/bias_aperture/fairness/`](../src/bias_aperture/fairness/) for executable behavior.
