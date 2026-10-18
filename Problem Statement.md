# Problem Statement

**Asset:** Nifty 50 index, long-only.  
**Sample:** 4 January 2005 – 25 September 2023.  
**Horizon:** 1 trading day.  
**Question:** which VaR specification keeps the exception rate closest to the confidence level it claims?

## Setting

Financial institutions need a loss number they can defend: how large a loss, over a short horizon, should be treated as exceptional. Value at Risk answers that with a horizon and a probability. For a 1-day 95% VaR, the modelled loss is expected to be worse than the VaR figure on about one trading day in twenty.

Global markets, and the Nifty 50 with them, do not sit in one volatility regime. Returns bunch into calm periods and turbulent periods (volatility clustering), and the distribution of daily log returns has heavier tails than a normal distribution (leptokurtosis). A risk system that assumes a fixed volatility and a normal curve will look stable in a quiet window and then miss a cluster of large losses.

## The question

Can a deep-learning VaR, specifically Encoded VaR built with a variational autoencoder, forecast 1-day Nifty 50 risk with an exception rate closer to the stated confidence level than the traditional methods still used as institutional benchmarks?

The traditional set in this study is:

- Historical simulation, which builds the loss distribution from past portfolio changes and reads VaR off an empirical quantile.
- The single-asset normal (variance) method, which scales a volatility estimate by a normal critical value.
- The delta-normal method, which uses one mean and one standard deviation for the whole estimation window.
- GARCH(1,1), which lets conditional variance move with recent shocks.
- Monte Carlo simulation, which draws a large set of returns from a rolling mean and volatility.

Encoded VaR is the machine-learning arm. A variational autoencoder maps windows of returns into a latent distribution, draws new latent points, and decodes them into synthetic return scenarios. VaR is a quantile of those scenarios. The design aim, following Arian, Moghimi, Tabatabaei, and Zamani (2022), is to learn the shape of the return distribution rather than to memorise noise in one historical path.

## Why the usual shortcuts fail here

The report's log-return chart for 4 January 2005 to 25 September 2023 shows repeated clusters of high volatility. In those stretches, an ARMA or ARIMA mean model does not supply a usable volatility forecast, so it is not used as a VaR engine in this project. The GARCH notebook confirms the same sample properties: log returns are stationary on an ADF test, and their density is peaked with heavy tails.

That combination creates a concrete failure mode.

- A historical window can only repeat losses that already occurred. A new kind of shock inside the test period is invisible until it is inside the window.
- A normal VaR with constant volatility has one width for every day. When volatility rises, that width is too narrow and violations cluster.
- A model that fits noise in-sample can post a flattering backtest and then fail on a later high-volatility stretch. The Encoded VaR section of the report treats that overfitting problem as the reason to regularise the latent space with a Kullback-Leibler term, not only a reconstruction loss.

## What is being forecast

| Item | Choice in this study |
| --- | --- |
| Asset | Nifty 50 index (a one-asset portfolio) |
| Sample | 4 January 2005 to 25 September 2023 |
| Horizon | 1 trading day |
| Confidence | 95% in the GARCH, single-asset, and Encoded VaR exercises; 99% in the historical and Monte Carlo exercises |
| Portfolio scale | ₹1 crore notionally invested on day 1 (historical); 1,000 index units, worth ₹21,03,750 on 4 January 2005 (single-asset and GARCH) |
| Decision metric | AE ratio, plus Kupiec unconditional coverage, Christoffersen independence, and conditional coverage |

The single-index scope is deliberate. The report states that the autoencoder experiments were limited to Nifty 50 because of computing constraints. The same backtesting ideas extend to a multi-asset book; this repository does not estimate one.

## What a successful model looks like

A useful model in this project has three properties at once.

1. **Calibration.** The AE ratio is near 1 at the confidence level the model claims. For a 95% VaR, about 5% of test days should breach the threshold. For a 99% VaR, about 1% should.
2. **Independence.** Breaches should not arrive in clumps. Christoffersen's independence test and the conditional coverage test are the checks used in the notebooks.
3. **Adaptation.** The threshold should widen when recent returns are turbulent and narrow when they are quiet. Rolling GARCH and the fine-tuned autoencoder are built for that. A single delta-normal number is not.

The report's empirical reading of those criteria: rolling GARCH(1,1) brought the illustrated test window to an AE ratio near 1 (13 violations against an expectation of about 12.5). Encoded VaR reported an AE ratio of 1.04 at 95% confidence, with coverage-test p-values above conventional significance levels. Historical simulation (AE about 3.4), delta-normal (AE 3.75), the single-asset normal method (AE about 2.33), and Monte Carlo (AE 2.4 at the 99% level) left more exceptions than the confidence level allows.

## What this project is not claiming

The study is an empirical comparison on one equity index, with each method implemented at the settings in its notebook (window length, confidence level, and innovation distribution). It is a forecasting and backtesting exercise. It does not produce a trading signal, a regulatory capital number, or a multi-asset risk system.
