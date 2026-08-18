# Swiss Market Volatility Forecasting — UBS/SMI Risk Framework

Data Science project forecasting volatility and Value-at-Risk for SIX-listed
equities (UBSG, NESN, NOVN, ROP/Roche) and the SMI, built as a 3-milestone
progression: data audit → GJR-GARCH modeling & backtesting → skew-t refinement
& ML challenger evaluation.

**Author:** Camilo Seydoux

---

## 1. The problem

UBS Group AG's legal merger with Credit Suisse Group AG (closed 12 June 2023)
materially changed UBSG's balance sheet composition — international asset/
liability mix, foreign-revenue exposure, and observed volatility persistence
all shifted around that date. A volatility/VaR framework built on data
spanning the merger needs to actually test whether UBSG behaves differently
from the other Swiss names post-merger, not assume it — and needs to be
honest about the tail risk that generic VaR models systematically understate.

## 2. The solution

**GJR-GARCH(1,1)** with asymmetric (skewed) Student-t innovations for UBSG,
fit **separately** from a pooled symmetric-t specification for NESN/NOVN/ROP —
the separate-vs-pooled decision made via a formal likelihood-ratio test, not
assumed. Walk-forward out-of-sample backtest (2021-01-04 → present, ~1,397
trading days, quarterly re-estimation), evaluated with Kupiec and
Christoffersen coverage tests.

**Key result:** switching UBSG from symmetric-t to skew-t innovations moves
the 99% VaR Kupiec test from borderline-failing (p=0.0144) to comfortably
passing (p=0.0785) — verified via a paired day-by-day comparison (not just
two separate p-values) and decomposed to confirm the improvement is *mostly*,
not purely, attributable to skewness (one of three reclassified days showed a
joint effect with the degrees-of-freedom parameter, disclosed rather than
rounded off).

## 3. Champion vs. Challenger

A LightGBM model (gradient boosting on engineered realized-volatility
features) was evaluated against GJR-GARCH skew-t using a criterion **fixed
before any fitting**: beat the champion on QLIKE loss *and* not fail
calibration where the champion passes.

| Model | QLIKE (lower=better) | 99% VaR Kupiec p-value |
|---|---|---|
| GJR-GARCH skew-t (champion) | −7.017 | 0.079 (pass) |
| LightGBM (challenger) | **−7.078 (wins)** | **0.026 (fails)** |

The challenger wins on average loss but fails calibration exactly where it
matters most for tail risk — so per the pre-registered rule, **GJR-GARCH
skew-t remains champion**, with LightGBM retained as a monitored alternative
rather than discarded or adopted.

## 4. Open threads — read this before assuming anything is fully closed

`standing_open_threads.pdf` lists six items deliberately **not** presented as
resolved: UBSG's post-merger sample size (~2 years, directionally consistent
but never independently large-sample-confirmed), a deferred event-clustering
methodology, an undecomposed joint effect in one backtest refit, the ML
challenger's monitored (not settled) 99% calibration gap, a BIC near-tie on
the skew-t decision, and a process note on a recurring environment bug. This
file exists specifically so nobody mistakes "the memo didn't mention it" for
"it's resolved."

## 5. What's in this repo

| File | What it is |
|---|---|
| `01_eda_regime_correlation.ipynb` / `.html` | Milestone 1: data audit, corporate-action handling (Sandoz spin-off, Roche ticker conversion), regime-conditioned correlation, FX-beta decomposition |
| `02_garch_var_backtest.ipynb` / `.html` | Milestone 2: GJR-GARCH fitting, separate-vs-pooled LR test, VaR backtesting, two real fitting bugs found and fixed (documented, not hidden) |
| `03_skewt_ml_challenger.ipynb` / `.html` | Milestone 3: skew-t refinement, paired-comparison verification, LightGBM challenger evaluation |
| `milestone1_memo.pdf`, `milestone2_memo.pdf`, `milestone3_memo.pdf` | One-page, recommendation-first memos for each milestone |
| `standing_open_threads.pdf` | What's still provisional — see Section 4 above |
| `swiss_market_data.zip`, `_part2.zip`, `_part3.zip` | Raw daily price data pulled via `yfinance` (SSMI, UBSG, NESN, NOVN, ROP, RO, ZURN, VIX, USDCHF, EURCHF), unmodified from source |

`.html` files are self-contained (charts embedded, not linked) — open with a
double-click, no Python required. `.ipynb` files render on GitHub directly,
or open in Jupyter for the live code.

## 6. Reproducing this — stated honestly, not aspirationally

**What works right now, with zero setup:** every notebook already contains
its full computed output — tables, charts, narrative — saved inside the file
itself. Opening the `.ipynb` (on GitHub or in Jupyter) or the `.html` shows
the complete, correct analysis. This is true today, unconditionally.

**What does *not* yet work out of the box:** re-executing notebooks 02 and 03
from a clean clone (e.g. "Restart & Run All"). They load several intermediate
result files (backtest summaries, per-refit diagnostics) that were generated
during the original interactive analysis sessions and are not yet included in
this repo. This does not affect the validity of any finding — it only affects
literal from-scratch re-computation. Regenerating them is a scoped, known task
(not a mystery gap) and is tracked as follow-up work.

To work with the raw data directly: unzip the three `swiss_market_data*.zip`
files into `data/raw/` and see `01_eda_regime_correlation.ipynb` for the
loading/cleaning pipeline.

## 7. Methodology notes worth knowing before reading the notebooks

- **1-day VaR**: direct GJR-GARCH conditional-variance forecast.
- **10-day stress VaR**: Monte Carlo forward simulation as primary, Filtered
  Historical Simulation as cross-check — not blended, reported side by side
  because they disagree in an informative way (traced to the innovation
  distribution being measurably too thin-tailed, not treated as noise).
- **Backtest window**: 2021-01-04 → present, chosen specifically because
  shorter (e.g. 250-day) windows give standard coverage tests almost no
  statistical power at the 99% level — disclosed and reasoned about, not a
  default left unquestioned.
- Every finding in this repo that could be traced to a specific verification
  step was traced to one — including two hypotheses (a persistence-based
  explanation and an event-clustering narrative around the 2023 Credit
  Suisse crisis) that were built, tested, and rejected by their own evidence
  rather than kept because they sounded plausible.
