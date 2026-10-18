# Encoded VaR

**Notebook:** [`EncodedVaR/Encoded_VaR_FCP.ipynb`](EncodedVaR/Encoded_VaR_FCP.ipynb)  
**Report section:** 7. Encoded VaR  
**Confidence:** 95%. Reported AE ratio: 1.04

Earlier experiments, pointed at the shared datasets: `Encoded_VaR_try2.ipynb` and `EncodedVaR_2yr.ipynb`. The published AE ratio comes from the primary notebook, not from those drafts.

## Why an autoencoder

The traditional models in this project either replay the past (historical simulation) or impose a small parametric variance equation (delta-normal, GARCH, Monte Carlo). Both can fit noise. A network that reconstructs every wiggle of a return window will look accurate in-sample and then fail when the next high-volatility stretch does not resemble that noise. Encoded VaR, as used in this project and in Arian, Moghimi, Tabatabaei, and Zamani (2022), tries to learn the distribution of returns rather than a single path.

The tool is a variational autoencoder (VAE).

- The **encoder** compresses a window of returns into two vectors: a mean μ and a log-variance of a latent Gaussian.
- A **reparameterization** draw, `z = μ + exp(0.5 log σ²) × ε` with ε standard normal, keeps the sampling step differentiable.
- The **decoder** maps z back to a window of returns.
- The **loss** is a reconstruction term (mean squared error between input window and decoded window) plus a Kullback-Leibler term that pulls the latent distribution toward a standard normal. The KL weight is `beta` (1.0 in this notebook).

Once the decoder has learned that map, new scenarios are produced by sampling latent points and decoding them. VaR is a quantile of the resulting returns, not a quantile of the raw history.

## Architecture used in the notebook

Input length is one window of returns. The default window is 90 trading days, and the latent width is 32.

| Block | Layers |
| --- | --- |
| Encoder | Dense 128, ReLU → Dense 64, ReLU → Dense 32 (`z_mean`) and Dense 32 (`z_log_var`) |
| Latent draw | `z = z_mean + exp(0.5 z_log_var) * ε` |
| Decoder | Dense 64, ReLU → Dense 128, ReLU → Dense 90 (linear) |
| Optimiser | Adam, learning rate 0.001 |

The first implementation is a `tf.keras.Model` whose `call` adds reconstruction loss and `beta × KL` through `add_loss`. The second implementation, later in the same file, uses a custom `train_step` and reports `loss`, `reconstruction_loss`, and `kl_loss` separately. Both are complete training loops. They share class names, so restart the kernel before running the second one.

## Data handling in the reported experiment

The workbook is `data/Nifty50_data.xlsx`.

| Split | Dates in the notebook |
| --- | --- |
| Initial training | 5 January 2005 through 31 December 2017 |
| Fine-tuning and backtest | After 31 December 2017, through the end of the file (25 September 2023) |

Preprocessing in the first implementation:

1. Convert the index level to a simple return with `pct_change`.
2. Standardise with a trailing 90-day mean and standard deviation, so the network sees a comparable scale in calm and turbulent years.
3. Cut the standardised series into sliding windows of length 90. Those windows are the VAE training rows.

The initial fit runs for 50 epochs. Weights are written to `vae_model.weights.h5`.

Fine-tuning walks the post-2017 sample in blocks. On each block the saved weights are updated for a further 50 epochs (batch size 16), then VaR is scored on the following 10 return observations. This is the notebook's way of letting the latent model move with the new regime without training only on the distant past.

## From latent samples to a VaR number

For a 95% VaR the notebook does the following.

1. Standardise the latest 90-day price window.
2. Draw `n_scenarios` latent vectors (5,000 in the first implementation, 1,000 in the second).
3. Decode them. The last position of each decoded window is kept as a standardised scenario return.
4. In the first implementation those scenarios are mapped back to the original return scale with the current 90-day mean and standard deviation:

```text
return = scenario × current_std + current_mean
```

5. VaR is the empirical 5th percentile of those returns: `(1 − 0.95) × 100 = 5`.
6. A violation is a realised return below that percentile.

The second implementation samples around the encoder's own `z_mean` and `z_log_var` for the current window, and it counts a violation when the standardised test point is below the negative of the predicted percentile. Read that sign convention in `fine_tune_and_calculate_var` before comparing its AE ratio with the first implementation.

## Result reported

At the 95% level the report's Encoded VaR backtest has an **AE ratio of 1.04**. Expected violations are 5% of the scored days. An AE ratio of 1.04 means the breach count is almost exactly that 5% benchmark.

The report also states that the p-values of the unconditional coverage test and the conditional coverage test sit above the significance level, which the authors read as evidence that breaches are not clustering and that the model is calibrated. Relative to the other methods in the same report, this is the AE ratio closest to 1, together with rolling GARCH (AE about 1.0 at the same 95% level).

## What the two notebooks besides the primary file are

| Notebook | Role |
| --- | --- |
| `Encoded_VaR_FCP.ipynb` | Full-sample study described above. Two implementations, one file. |
| `Encoded_VaR_try2.ipynb` | Earlier split into an initial-weight sample and a fine-tune sample. It now reads `data/Nifty50_data.xlsx` and splits at 31 December 2018, matching the dates written in the original Colab comments (2005–2018, then 2019 onward). |
| `EncodedVaR_2yr.ipynb` | Same VAE idea on the short file `data/Nifty50_2yr.csv` (from 27 September 2021, 525 rows), with a 90-day window, a 21-day step, and a 10-day test block. |

The original Colab paths (`/content/drive/...`, `/content/2y_Nifty50_data.xlsx`, and the two split workbooks) are preserved as comments. Those extra workbooks are not in this repository; the shared `data/` files are the inputs.

## How to read `Encoded_VaR_FCP.ipynb`

| Cells | Content |
| --- | --- |
| Setup | Imports and a path resolver for `data/Nifty50_data.xlsx` |
| First model | `VariationalAutoencoder` and `EncodedVaR`: rolling standardisation, 2005–2017 pretraining, post-2017 fine-tune, 5th-percentile VaR, AE ratio |
| Second model | Custom `train_step`, 21-day step, latent sampling around the encoded posterior, a second AE ratio |

Training the first model for 50 epochs on every 90-day window, and then fine-tuning again on every test block, is a long run. The report's published AE ratio of 1.04 is the outcome of that procedure, not a number hardcoded in the notebook.

## Where this sits against GARCH

GARCH adapts through a variance recursion with a handful of parameters. Encoded VaR adapts by updating a neural decoder and by regenerating scenarios from the latent space. The report's claim is that the VAE can represent nonlinear structure in the return distribution that a GARCH(1,1) recursion does not attempt to represent. On the published AE ratios the two methods are close (about 1.0 and 1.04). The suggested next step in the report is a hybrid: a statistical volatility model together with a learned scenario generator.
