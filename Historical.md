# Historical Simulation

**Notebook:** [`Historical/VaR_Estimation_of_nifty50_using_historical_method.ipynb`](Historical/VaR_Estimation_of_nifty50_using_historical_method.ipynb)  
**Report section:** 3. Historical simulation  
**Confidence:** 99%

An earlier draft of the same idea is `Historical/VaR_historical_nifty.ipynb`. Reproduce the reported AE ratio from the notebook named above.

## Idea

Historical simulation estimates VaR from the empirical distribution of past portfolio changes. No probability law is imposed on returns. The method sorts what already happened and reads off a quantile. Because the quantile is taken from real market moves, the estimate automatically includes whatever skewness, fat tails, and large one-day moves were present in the window.

That is also the limitation. The future is assumed to look like the chosen history. A short window is noisy. A long window mixes older market regimes into today's threshold. A loss that has never occurred in the window cannot appear in the quantile.

## Portfolio in this study

The illustrated book is a ₹1,00,00,000 position in the Nifty 50, opened at the first index level in the sample (4 January 2005). The number of index units is

```text
units = 1,00,00,000 / Nifty level on day 1
```

and it is held fixed. Each later portfolio value is `units × index level`. The daily profit-and-loss used for VaR is the change in that portfolio value. A negative change is a loss. In the notebook this column is named `Loss`, and VaR is a negative number (a loss threshold). Reported VaR is shown as a positive rupee amount via `abs(VaR)`.

## Estimation steps

The report uses a 501-day block, which produces 500 scenario returns after the first difference.

1. Take 501 consecutive index levels and the matching portfolio values.
2. Compute the day-to-day change in portfolio value. That yields 500 scenarios.
3. Sort those changes from the largest loss (most negative) to the largest gain.
4. At a 99% confidence level the VaR is the 1% empirical quantile. In a list of 500 sorted scenarios the report identifies this with the 5th worst loss.
5. Hold that rupee threshold as the 1-day 99% VaR for the following test days.
6. A violation is a test day whose portfolio change is more negative than this threshold.

The notebook sets `sig = 0.01` and, on the first window, also prints `losses[4]` explicitly as that 5th worst outcome. The same quantile rule is then rolled through the sample: every training block is 501 rows, and each following test block is 51 rows (one row is consumed by the first difference, leaving about 50 loss observations).

## Worked reading of the first window

On the opening 501 days the notebook sorts realised portfolio changes and marks the 99% threshold on a histogram of those changes. The next 51 rows are the first out-of-sample check. The remainder of the notebook repeats that pattern for every later starting date and plots how many violations each test block produced.

## Backtest reported for the full sample

The report gives an AE ratio of about **3.4** for the historical method. Expected violations at 99% are 1% of the test days. An AE ratio of 3.4 means realised breaches were about 3.4 times that benchmark, so the historical threshold was too high (losses were not negative enough) relative to what the next days actually delivered.

The report also records p-values of about **0.998** for the unconditional coverage test, the independence test, and the conditional coverage test. In the standard form of those tests, a p-value this large means the test does not reject its null: the exception rate is treated as consistent with the target probability, and exceptions are treated as independent through time. The report cites those p-values as evidence that the model can still be used, while the AE ratio remains far from 1. Both numbers should be read together. The AE ratio is the direct count. The likelihood-ratio tests are sensitive to how the violations are pooled.

The notebook implementation pools violations by walking the start date forward one day at a time. Neighbouring 51-day test blocks therefore share almost all of their days, and the Kupiec statistic is computed on that stacked series. Overlapping windows make the raw violation total and the test sample size larger than a non-overlapping backtest of the same calendar period. The figure to quote from the study is the report's AE ratio of about 3.4, not a fresh ratio computed from the overlapping sums without adjusting for that overlap.

## What the method is good at, according to the report

- There is no normality assumption and no estimated volatility parameter.
- The calculation is a sort and a quantile.
- Extreme moves that occurred inside the window enter the VaR directly, which is why many risk teams keep historical simulation as a benchmark.

## Where it is weak on this sample

- The threshold cannot invent a shock that is outside the 501-day memory.
- If the window is dominated by a calm regime, the 1% quantile is too mild for the next volatile regime. That is the natural explanation of an AE ratio well above 1.
- The report's own conclusion is that historical simulation is a practical benchmark, and that stress tests and scenario analysis are needed beside it.

## How the notebook is organised

| Step | What the cells do |
| --- | --- |
| Data | Load `data/Nifty50_data.xlsx` and build the ₹1 crore path |
| One window | First 501 days, sort losses, 99% VaR, histogram |
| One test | Next 51 days, count breaches of that VaR |
| Full roll | One training frame and one test frame per start date |
| Tests | Kupiec POF, Christoffersen independence, conditional coverage |
| Plot | Violations per test window against the count expected at 99% |

The coverage-test functions follow the usual likelihood-ratio construction: Kupiec compares the observed breach frequency with `sig`, Christoffersen compares the frequency of breach-after-breach with breach-after-calm, and the conditional coverage statistic adds the two likelihood ratios (chi-square with 2 degrees of freedom).
