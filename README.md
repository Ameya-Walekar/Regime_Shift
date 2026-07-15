# Regime Shift

## Introduction:

This is Regime-Shift, a dynamic asset allocation engine which uses Hidden Markov Model(HMM) to detect market regime and accordingly rebalance the portfolio.

## Architecture Decisions:

The main components of the engine:

1. Data Ingestion
   - Sources include Yahoo Finance(yfinance) and FRED API.
   - From yfinance we obtain the stock market prices (8-asset universe SPY, TLT, GLD, EEM, VNQ, LQD, BIL, QQQ), CBOE VIX data, SKEW data.
   - From FRED API we get macroeconomic indicators ( the 3-month Treasury yield (DGS3MO) and the 10-year Treasury yield (DGS10)).
2. Feature Engineering
   - In this step we extract the exact features which we require from the sources and which will be fed to the HMM model.
   - The features are:
     - ema14 — 14 day EMA of SPY Close
     - daily_range - intraday value of (High-Low)/Close
     - ewma20_vol — sqrt of a 20-day EWMA of squared SPY returns
     - macd_main — normalized MACD
     - vix — CBOE VIX level (volatility)
     - skew_idx / skew_idx_lag — CBOE SKEW index and its 1-day lag
     - t3mo — 3-month Treasury yield
     - yield_ratio — DGS3MO / DGS10
   - These features are standardized by applying z-score on each before going into the HMM model.
3. The Hidden Markov Model:
   - Each fold follows the same step flow: (explanation of folds is given later)
     - fit the HMM on that fold's training window.
     - decode the test window's hidden states via Viterbi, then label those raw states as Bull/Bear.
     - test the model on each fold's testing period.
   - The model is a Gaussian HMM with 2 states Bear and Bull.
   - The Model is trained in an unsupervised way on the feature matrix (via Baum Welch EM).
   - Since HMMs are sensitive to initialization, 8 random restarts are run per fold and the one with the highest training log likelihood is used in the fold.
   - The default prediction method of Gaussian HMM uses Viterbi Algorithm.
   - The raw Viterbi states (0/1) cannot be labelled as Bull/Bear directly, thus we use Oracle labels. An oracle label is built from forward SPY returns over a 21 day horizon. The Hungarian algorithm is used to find the optimal state to regime assignment. This mapping is frozen for that particular fold and the same is used for testing.
   - The Walk forward validation (i.e. testing) of the engine is done by implementing a multi fold strategy with expanding window of training data. Each fold's test window is a fixed period of 2 years. The fold strategy is used to test whether the system adapts to various time periods or does it fail.
4. Convex Portfolio Optimization
   - Based on the verdict by the HMM the engine decides how much to allocate in available asset according to a few optimization rules:
     - Higher allocation to assets with strong trailing returns.
     - Don't stray too far from the default allocation for the current regime.
     - Avoid high risk - high volatility is one of the important factors to account for allocation. Co-variance between assets is also taken into consideration.
     - Avoids excessive rebalancing.
   - Constraints: No short positions, and no single asset can exceed a regime-specific cap.
   - Features of allocation based on the regime:
     - Bull Regime : Low risk aversion, loose anchor pull i.e. optimizer is freer to chase returns and concentrate.
     - Bear Regime : High risk aversion, tight anchor pull i.e. optimizer is pushed toward safety
   - The exact formula used is explained in the code.
   - If there is an error in the optimizer then the allocation falls back to the the default allocation for the regime.

5. Performance Metrics and Charts
   - The strategy is benchmarked against a static 60/40 (SPY/GLD) portfolio and an equal-weight portfolio.
   - Performance metrics include: annualized return, annualized volatility, Sharpe, Sortino, max drawdown, Calmar, total return, and max drawdown duration.
   - Charts produced: cumulative return curves (with regime shading), drawdown curves, rolling sharpe visualization, weight allocation chart, regime frequency, monthly heatmap, and hmm transition matrices of all folds.
   - CSV files produced: Result of walk forward validation, tearsheet of all folds and a tearsheet summary.
   - Text Files produced - Readable files for walk forward validation and performance metrics.
   - All results are stored in a folder "output" which is created in the same folder as the code.

## Reproducibility Guide:

1. Download the 'regime_shift_final.ipynb file'
2. Install python(3.11 or above) and all the python libraries required to run the code given in requirements.txt.
3. Generate a FRED API key from their official website (you have to sign in or create a new account).
4. Create a '.env' file in the same folder as the code and write one line in it:
   FRED_API_KEY = "your_newly_created_FRED_API_key"
5. Running the code (Open in Jupyter and run all cells of the .ipynb file) gives all the results in output folder which includes:
   - 7 charts
   - 3 csv files
   - 2 text files

## Testing Instructions:

A test is considered successful if all of the following checks pass:

- All sources (yahoo finance and FRED) are loaded successfully without any failure messages, (such as FRED unavailable or Could not fetch price data).
- Feature matrix is well formed. 9 columns, no NaNs in the correlation matrix.
- If the data ingestion and feature engineering step is successful the HMM model fitting will occur without any errors.
- Walk forward validation is successful - the walk_forward_backtest.txt (in the output folder) shows details about the hmm prediction and the subsequent portfolio reallocation.
- The performance_tear_sheet.txt displays the performance metrics.
- All expected files produced — 7 charts + 3 CSVs + 2 text files in output folder.
