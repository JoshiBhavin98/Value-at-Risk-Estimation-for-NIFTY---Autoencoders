# Value at Risk Estimation for the Nifty 50

1-day Value at Risk (VaR) for a long Nifty 50 position. The study compares historical simulation, a normal single-asset VaR, delta-normal VaR, rolling GARCH(1,1), Monte Carlo simulation, and Encoded VaR (a variational autoencoder).

This is the Group 5 capstone for MSDSM01, supervised by Dr. Puneet Pasricha (IIT Ropar). The method notes in this repository follow that report. The PDF itself stays on disk under `report/` and is not part of the published tree.

## Results

Exception rates below are the figures published in the report. Confidence levels differ by method. The comparable column is the AE ratio: actual VaR violations divided by the number of violations the confidence level implies. A ratio near 1 is a calibrated model. A ratio above 1 means the threshold was breached more often than the model claimed.

| Method | Confidence | Backtest evidence | AE ratio |
| --- | ---: | --- | ---: |
| Historical simulation | 99% | More exceptions than a 1% tail implies | ~3.4 |
| Single-asset normal VaR | 95% | Mean exception ratio across rolling windows | ~2.33 |
| Delta-normal | 95% | 45 violations in a 250-day window | 3.75 |
| GARCH(1,1), rolling, Student-t | 95% | 13 violations in that window | ~1.0 |
| Monte Carlo | 99% | 2.4% of days breached a 1% VaR | 2.4 |
| Encoded VaR (VAE) | 95% | Exception rate in line with a 5% tail | 1.04 |

On this sample, rolling GARCH and Encoded VaR are the two specifications whose reported AE ratios sit next to 1. Nifty log returns in the study window show volatility clustering and heavier tails than a normal distribution, which is the empirical reason a constant-volatility normal VaR under-covers the losses.

## Documentation

| Document | Contents |
| --- | --- |
| [Problem Statement.md](Problem%20Statement.md) | Decision problem, scope, and acceptance criteria |
| [Historical.md](Historical.md) | Historical simulation: design, backtest, limitations |
| [GARCH.md](GARCH.md) | Delta-normal benchmark and rolling GARCH(1,1) |
| [Encoded VaR.md](Encoded%20VaR.md) | VAE architecture, sample split, and scenario VaR |

## Repository layout

```text
.
├── README.md
├── Problem Statement.md
├── Historical.md
├── GARCH.md
├── Encoded VaR.md
├── requirements.txt
├── data/
│   ├── Nifty50_data.xlsx          # report sample, 4 Jan 2005 – 25 Sep 2023
│   └── Nifty50_2yr.csv            # two-year extract used by one VAE draft
├── Historical/                    # historical-simulation notebooks
├── GARCH/                         # delta-normal and GARCH notebooks
├── EncodedVaR/                    # variational autoencoder notebooks
├── MonteCarlo/                    # Monte Carlo notebooks
└── Single Asset Case/             # normal single-asset VaR notebook
```

| Folder | Primary notebook | What it estimates |
| --- | --- | --- |
| `Historical/` | `VaR_Estimation_of_nifty50_using_historical_method.ipynb` | 99% VaR from a 501-day empirical loss distribution |
| `GARCH/` | `VaR_Forecasting_GARCH.ipynb` | 95% delta-normal VaR and rolling GARCH-t VaR |
| `EncodedVaR/` | `Encoded_VaR_FCP.ipynb` | 95% VaR from VAE scenarios; published AE ratio 1.04 |
| `MonteCarlo/` | `monte carlo model for VAR_N50 - F.ipynb` | 99% Monte Carlo VaR, 90-day lookback, 10,000 draws |
| `Single Asset Case/` | `VaR_Single_Asset.ipynb` | 95% normal VaR on 1,000 index units |

`VaR_historical_nifty.ipynb`, `VaR_GARCH.ipynb`, `VaR_EST.ipynb`, `Encoded_VaR_try2.ipynb`, and `EncodedVaR_2yr.ipynb` are earlier working copies. They are kept so the path from draft to the reported specification stays visible. Use the primary notebook in each folder when reproducing a reported number.

## Requirements

Python 3.10 or later. Packages are listed in [`requirements.txt`](requirements.txt). Versions are unpinned: the notebooks depend on the public APIs of pandas, SciPy, `statsmodels`, the `arch` package, and TensorFlow Keras.

```bash
python -m venv .venv
# Windows: .venv\Scripts\activate
# macOS/Linux: source .venv/bin/activate
pip install -r requirements.txt
```

TensorFlow is required only for `EncodedVaR/`. The statistical notebooks run without it.

## Run a notebook

Open a notebook from its method folder, or from the repository root, and run the cells from the top. Each notebook resolves `data/Nifty50_data.xlsx` relative to the working directory (the repository root, or the parent of the method folder). No absolute drive path is required.

Two execution constraints:

- `EncodedVaR/Encoded_VaR_FCP.ipynb` defines two complete models in one file, under the same class names. Restart the kernel before running the second implementation.
- `GARCH/VaR_Forecasting_GARCH.ipynb` refits a GARCH model on every test day. The first 1,002-day block is the exercise reported in the paper. The three later blocks repeat that design and take longer.

Saved VAE weights (`*.weights.h5`) are training artifacts and are gitignored.

## Data

| File | Columns | Use |
| --- | --- | --- |
| `data/Nifty50_data.xlsx` | `Dates`, `NIFTY Index` | All reported backtests. Daily Nifty 50 levels, 4 January 2005 through 25 September 2023. |
| `data/Nifty50_2yr.csv` | `Dates`, `NIFTY Index` | `EncodedVaR/EncodedVaR_2yr.ipynb` only. 525 rows beginning 27 September 2021. |

The portfolio is one index, not a multi-asset book. The report limits the scope to Nifty 50 because of the cost of the autoencoder runs.

Position sizing used in the notebooks:

- Historical simulation: ₹1,00,00,000 notional on the first date, unit count held fixed.
- Single-asset and GARCH: 1,000 index units, worth ₹21,03,750 on 4 January 2005.

## Evaluation protocol

A violation is a day on which the realised loss is worse than that day's VaR. Models are scored with:

1. **AE ratio** — actual violations / expected violations at the stated confidence level.
2. **Kupiec unconditional coverage** — whether the breach frequency matches that confidence level.
3. **Christoffersen independence** — whether breaches cluster.
4. **Conditional coverage** — the joint test (sum of the two likelihood-ratio statistics).

Window length, innovation distribution, and confidence level are method-specific. They are documented in the matching note and set in the primary notebook. Do not rank two AE ratios without checking that the confidence levels match.

## Limitations

- One equity index, one long-only position, a one-day horizon.
- Historical and Monte Carlo exercises are reported at 99%. GARCH, single-asset, and Encoded VaR are reported at 95%. The AE ratio is the cross-method summary, not the raw violation count.
- The historical notebook rolls overlapping 51-day test windows. The study figure to cite is the report's AE ratio of about 3.4.
- Later blocks in the GARCH notebook set `q=0` (ARCH(1)). The 13-violation result is the earlier GARCH(1,1) loop (`p=1, q=1`).
- Encoded VaR training is stochastic (latent sampling, weight initialisation). A fresh run will not reprint 1.04 exactly. That figure is the reported outcome of the procedure, not a constant in the code.
- This repository does not produce a trading signal or a regulatory capital number.

## Authors

Georgy S Pulayath, Joshi Bhavin Rasikbhai, Prabhujeet Sahu, and Shrenik Nandre. MSDSM01. Guide: Dr. Puneet Pasricha, IIT Ropar.

## Citation

```text
Pulayath, G. S., Joshi, B. R., Sahu, P., & Nandre, S. A comparative analysis of
traditional models and deep learning for value at risk estimation. Final capstone
report, Group 5, MSDSM01, IIT Ropar. Supervised by Puneet Pasricha.
```

## References

1. Arian, H., Moghimi, M., Tabatabaei, E., & Zamani, S. (2022). Encoded Value-at-Risk: A machine learning approach for portfolio risk measurement. *Mathematics and Computers in Simulation, 202*, 500–525. https://doi.org/10.1016/j.matcom.2022.07.015
2. Goel, A., Pasricha, P., & Kanniainen, J. (2024). Time-series foundation model for Value-at-Risk. arXiv:2410.11773. https://doi.org/10.48550/arxiv.2410.11773
3. Hull, J. (2015). *Options, Futures, and Other Derivatives*.
