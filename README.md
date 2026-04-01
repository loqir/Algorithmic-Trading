# Pairs Trading on S&P 500 Stocks

This project  builds and evaluates a statistical arbitrage strategy on S&P 500 equities using PCA-based clustering, cointegration screening, rolling z-score signals, validation-based parameter tuning, and out-of-sample backtesting.

The main workflow lives in `pairs_trading.ipynb` and uses a saved price snapshot in `sp500_pricesto2019_snapshot.csv`.

## Project Objective

The notebook aims to:

- reduce a large stock universe into smaller, more comparable groups using PCA loadings and DBSCAN clustering,
- search those clusters for cointegrated pairs,
- build mean-reversion signals from rolling spreads and z-scores,
- tune trading parameters on a validation set using Sharpe ratio,
- evaluate the selected configuration on a held-out test set.

## Repository Contents

- `pairs_trading.ipynb`: end-to-end notebook containing data loading, clustering, pair selection, signal construction, validation tuning, backtest, and visualisations.
- `sp500_pricesto2019_snapshot.csv`: local price snapshot used by the notebook.

## Strategy Overview

### 1. Data Preparation

The notebook loads a saved S&P 500 price snapshot and sorts it by date and ticker.

It then splits the full dataset into:

- training set: 60%
- validation set: 20%
- test set: 20%

This separation is used to reduce look-ahead bias when selecting parameters.

### 2. PCA on Returns

The training set is converted to daily percentage returns, standardized, and passed through PCA.

PCA is used here as a dimensionality reduction step so that stocks can be compared based on their loading patterns rather than raw prices.

Key configuration used in the notebook:

- `N_PRIN_COMPONENTS = 50`
- `MAX_COMPONENTS = 30`

### 3. DBSCAN Clustering

The notebook clusters stocks using their PCA loadings.

It performs a grid search over:

- `MIN_SAMPLES_GRID = [3, 4, 5, 6, 7]`
- `EPS_QUANTILE_GRID = [0.6, 0.65, 0.7, 0.75, 0.8, 0.85]`

For each parameter combination, it evaluates candidate clusterings using silhouette score and a noise-ratio tiebreaker. Noise points are removed, and downstream pair search only considers clusters with:

- more than 1 member
- at most `CLUSTER_SIZE_LIMIT = 50`

This step narrows the universe before cointegration testing.

### 4. Cointegrated Pair Selection

Within each valid cluster, the notebook searches for cointegrated pairs using the Engle-Granger cointegration test.

It then evaluates both spread directions and keeps a direction only if the resulting spread passes a KPSS stationarity screen.

This means the notebook does not simply test whether two price series are cointegrated; it also tries to ensure the chosen trading spread is stationary in the direction actually traded.

Spread evaluated is not the raw spread price but rather spread from a rolling regression described below.

### 5. Spread, Z-Score, and Signals

For each pair, the notebook:

- estimates a rolling regression of stock y on stock x using RollingOLS
- uses the rolling intercept and hedge ratio to build a synthetic spread
- defines the synthetic spread as the rolling regression residual, not the raw price difference
- computes a rolling z-score of that residual spread
- generates long and short spread signals from z-score entry and exit thresholds
- optionally applies a z-score stop loss
- converts spread positions into executable asset-leg units
More specifically, the notebook models the pair as:

y_t = alpha_t + beta_t * x_t + residual_t

and defines the synthetic spread as:

residual_t = y_t - (alpha_t + beta_t * x_t)

rather than the simple raw spread:

y_t - x_t

A raw price difference assumes a fixed 1-to-1 relationship, which is often unrealistic when the two stocks have different price levels or different sensitivities.
Using the rolling regression residual therefore gives a more meaningful measure of temporary deviation from the pair’s estimated equilibrium relationship, which is more likely to be stationary than the raw spread.

### 6. Validation Grid Search

The validation set is used to tune the core trading parameters.

Grid searched parameters:

- lookback window: `50` to `120` in steps of `10`
- z-score window: `20` to `50` in steps of `5`
- entry threshold: `[1.5, 2.0, 2.5]`
- exit threshold: `[0.5, 1.0]`
- stop loss z-score: `[None, 3.0, 3.5, 4.0]`
- trade fraction: `[0.05, 0.07, 0.1]`

Each configuration is scored using annualized Sharpe ratio on the validation portfolio return series.

The notebook stores the top results and selects `best_params` from the validation run.

### 7. Test Set Backtest

The chosen parameter set is then applied to the test data only.

For each selected pair, the notebook:

- rebuilds the rolling spread and z-score,
- generates pair signals,
- computes pair-level PnL,
- aggregates cumulative PnL across all traded pairs,
- converts cumulative PnL into a portfolio value series.

The test section is meant to represent out-of-sample evaluation after model and parameter selection has already been fixed.

## Visual Outputs

The notebook includes several diagnostic and result plots:

- PCA explained variance and cumulative explained variance
- DBSCAN k-distance curve for the selected clustering parameters
- cluster member count chart
- sample raw-price plots for a few clusters
- test-set equity curve
- test-set drawdown chart
- distribution of final pair PnL
- winner/loser pair summary charts

## Performance Metrics Used

The notebook reports, especially on the test set:

- annualized return
- annualized volatility
- Sharpe ratio
- maximum drawdown
- pair-level win/loss distribution

## Assumptions and Simplifications

 Several simplifying assumptions are built into the workflow:

- ignores transaction costs, commissions, borrow costs, slippage, and market impact
- assumes trades can be executed directly from the model-generated target units
- uses fixed-notional style scaling per trade rather than a fully realistic execution and portfolio accounting framework



