# GARCH and Delta-Normal VaR

**Notebook:** [`GARCH/VaR_Forecasting_GARCH.ipynb`](GARCH/VaR_Forecasting_GARCH.ipynb)  
**Report section:** 5. Delta-normal and GARCH  
**Confidence:** 95%

`GARCH/VaR_GARCH.ipynb` is an earlier pass over the same data. Use the forecasting notebook above when reproducing the 45-versus-13 violation comparison.

## Why this section exists

The Nifty log-return series used here is stationary (the notebook's ADF test rejects a unit root), and it is not Gaussian. The density is leptokurtic: too peaked in the centre and too heavy in the tails for a normal curve. The time plot also shows volatility clustering, with quiet stretches followed by stretches of large moves. A single standard deviation cannot describe both regimes. GARCH can, because tomorrow's variance is a function of today's shock and today's variance.

The report compares two forecasts on the same book:

- **Delta-normal**, a constant-volatility benchmark.
- **GARCH(1,1)** with a Student-t innovation, refit on a rolling window.

## Portfolio and sample split

The book is 1,000 long units of the Nifty 50. On 4 January 2005 that position is worth ₹21,03,750. Portfolio value on later days is `1,000 × index level`.

The illustrated exercise uses the first 1,002 rows of the workbook. Log returns are

```text
r_t = ln(P_t / P_{t-1})
```

and are then scaled by 100, so the VaR numbers in this notebook are in percent, not in rupees. The confidence level for the comparison is 95% (`alpha = 0.05`).

The rolling design in the report and the notebook:

| Piece | Length |
| --- | --- |
| Rows taken from the start of the file | 1,002 |
| Observations used to fit each GARCH model (`n_start`) | 753 |
| Following days on which VaR is scored | about 250 |

Three later blocks of 1,002 rows each are scored at the end of the notebook so the same rolling idea can be seen deeper in the 2005–2023 sample.

## Delta-normal

The variance-covariance VaR for a confidence level with lower-tail probability α is

```text
VaR_α = μ + z_α × σ
```

where μ is the mean return, σ is the standard deviation of returns, and z_α is the standard-normal quantile. For a 95% VaR, z_α is `norm.ppf(0.05)`, about −1.645. The formula is a left-tail number, so VaR itself is negative when it is expressed as a return. A violation is an actual return more negative than this number.

In the notebook the delta-normal figure is computed once, from the first 753 scaled log returns, and then held fixed across the test days:

```text
VaR = mean(r[:753]) + norm.ppf(0.05) * std(r[:753])
```

**Result reported:** 45 test-day violations. At 5% of 250 days the expected count is 12.5, so

```text
AE ratio = 45 / 12.5 = 3.75
```

The threshold does not widen when the test window becomes turbulent. The report treats this as the baseline failure of constant volatility.

## GARCH(1,1)

The model for conditional variance is

```text
σ²_t = ω + α₁ ε²_{t-1} + β₁ σ²_{t-1}
```

ω is the long-run weight, α₁ is the weight on yesterday's shock, and β₁ is the weight on yesterday's variance. A large return today raises σ tomorrow. A stretch of small returns lets σ decay toward ω / (1 − α₁ − β₁), provided α₁ + β₁ < 1.

VaR on each test day uses the latest conditional volatility and a Student-t quantile rather than a normal quantile, because of the heavy tails:

```text
VaR_t = μ_t + σ_t × t.ppf(0.05, df)
```

The notebook fixes the degrees of freedom at `3.7545967917` for every window (a heavy-tailed t, well below the normal limit). Each day the model is re-estimated on the previous 753 scaled log returns with `arch_model(..., vol="Garch", p=1, q=1, dist="t")`. Only the last conditional volatility of that fit is kept.

**Result reported:** 13 violations in the test sample, and an AE ratio near 1. The expected count at 5% is about 12 to 13, so 13 exceptions is the calibration the report describes as robust, including through high-volatility days. The notebook plots those test returns against the moving VaR and marks breaches in red.

The same notebook then computes Kupiec's proportion-of-failures statistic, Christoffersen's independence statistic, and the joint conditional coverage statistic on that exception series.

## Side-by-side

| | Delta-normal | GARCH(1,1) |
| --- | --- | --- |
| Return distribution | Normal | Student-t |
| Volatility | One σ for the whole test | σ updated every day |
| Adaptation when volatility clusters | None | High |
| Violations in the illustrated test | 45 | 13 |
| AE ratio | 3.75 | about 1 |

The report's conclusion is that the gain comes from modelling clustering, not from a more complicated mean equation. An AR(2) is fit earlier in the notebook only as a diagnostic; its residuals are passed to a first GARCH fit so the variance dynamics can be seen before the rolling VaR loop.

## Implementation notes for anyone re-running the notebook

- Scaling returns by 100 stabilises the `arch` optimiser. Compare VaR with the scaled returns, not with raw log returns.
- The first rolling loop uses GARCH(1,1) (`p=1, q=1`). The later three blocks call `arch_model` with `q=0`, which is an ARCH(1) specification. The printed label still says GARCH. Read the `q` argument before comparing those exception counts with the 13-violation result.
- `shape_param` is not re-estimated inside the loop. Changing it changes the quantile and therefore the AE ratio.
- Refitting every day is the expensive step. The first window is the one the report discusses in detail.

## How the notebook is organised

| Section | Content |
| --- | --- |
| Prices and log returns | Level plot and return plot for the full file, then the first 1,002 days |
| Diagnostics | ADF stationarity, AR(2), residual histogram, Ljung-Box |
| First GARCH fit | One GARCH(1,1) on the AR residuals, then a normal left-tail number from ω |
| Quantiles | Normal versus Student-t critical values at 5% and 1% |
| Rolling VaR | Daily refit, 95% t-VaR, exception count, breach plot |
| Coverage tests | Kupiec, independence, conditional coverage |
| Delta-normal score | 45 versus the GARCH exception count |
| Later windows | Three further 1,002-day blocks |
