# Determinants of U.S. Consumer Price Inflation (1969-2026)

A comparison of ARIMAX and Random Forest as predictive models of U.S. CPI inflation; built on 686 monthly observations from the Federal Reserve Economic Data (FRED) database.

## What this is

Inflation is rarely the product of a single cause; it emerges from the interaction of monetary policy, labour costs, producer prices, and real-sector activity. This project tests how well two fundamentally different modelling approaches, one explicitly time-aware and one not, capture that interaction.

We model the year-over-year change in CPI for All Urban Consumers using eight macroeconomic regressors pulled from FRED, spanning monetary instruments, yield curve dynamics, labour market conditions, and producer-level prices. Two models are estimated and compared head to head: an ARIMAX(1,0,1) and a Random Forest Regressor with 200 trees.

## Data

All series are sourced from FRED. The raw pull covers January 1968 through March 2026 (699 monthly observations). After YoY transformation, which consumes the first twelve months, 687 records remain; after cleaning, 686.

Eleven variables were considered initially:

| Variable          | FRED Code  | Definition                                          |
|-------------------|------------|-----------------------------------------------------|
| CPI_YoY (DV)      | CPIAUCSL   | CPI for All Urban Consumers, YoY %                  |
| FedFunds_Rate     | FEDFUNDS   | Federal Funds Effective Rate (%)                    |
| Treasury_10Y      | GS10       | 10-Year Treasury Yield (%)                          |
| IndProd_YoY       | INDPRO     | Industrial Production Index, YoY %                  |
| WageGrowth_YoY    | AHETPI     | Avg. Hourly Earnings, Prod. & Nonsup. Emp., YoY %   |
| M2_YoY            | M2SL       | M2 Money Supply, YoY %                              |
| Yield_Spread      | T10YFFM    | 10-Year Treasury Minus Fed Funds Rate (%)           |
| PPI_YoY           | PPIACO     | Producer Price Index by Commodity, YoY %            |
| Payroll_YoY       | PAYEMS     | All Employees, Total Nonfarm, YoY %                 |
| Capacity_Util     | TCU        | Capacity Utilisation, Total Index (%)               |
| MonBase_YoY       | BOGMBASE   | Monetary Base: Total, YoY %                         |

Six of these arrive as index levels rather than rates, so they were converted to YoY percentage change before anything else; this keeps every variable stationary-ish and on a comparable scale. Yield_Spread and Capacity_Util are kept in their original levels since they're already rates. Two missing values (one in CPI_YoY, one in Payroll_YoY) were imputed with the column median; nothing was dropped.

Multicollinearity check removed two variables before modelling:
- **FedFunds_Rate**, correlation of 0.903 with Treasury_10Y
- **Payroll_YoY**, correlation of 0.760 with IndProd_YoY

Eight regressors remain for estimation: Treasury_10Y, IndProd_YoY, WageGrowth_YoY, M2_YoY, Yield_Spread, PPI_YoY, Capacity_Util, MonBase_YoY.

The dataset is split chronologically, not shuffled, to avoid leakage: 80% train (548 obs, Jan 1969-Aug 2014), 20% test (138 obs, Sep 2014-Mar 2026).

## Models

**ARIMAX(1,0,1).** Estimated via maximum likelihood using statsmodels' SARIMAX implementation. The YoY transform already handles stationarity, so no further differencing was needed. Six of the eight exogenous regressors come out significant at the 5% level (Treasury_10Y, IndProd_YoY, WageGrowth_YoY, PPI_YoY, Capacity_Util, MonBase_YoY); M2_YoY and Yield_Spread don't clear significance, suggesting their effects either get absorbed by the dominant regressors or operate on a longer horizon than monthly data picks up.

**Random Forest Regressor.** 200 trees, random_state=42, fit via scikit-learn. No lagged dependent variable; it runs purely on the eight contemporaneous features so the comparison against ARIMAX stays clean. WageGrowth_YoY and PPI_YoY dominate the feature importance ranking (roughly 0.52 and 0.28 respectively), accounting for about 80% of the model's splitting decisions on their own.

## Results

| Model           |    MSE   |   RMSE   |   MAE    |   R²   |
|-----------------|----------|----------|----------|--------|
| ARIMAX (1,0,1)  | 0.8386   | 0.9157   | 0.7468   | 0.8168 |
| Random Forest   | 1.5959   | 1.2633   | 0.8623   | 0.6514 |

ARIMAX wins on every metric, and not by a small margin. Its RMSE of 0.92 points means it's typically within a percentage point of actual CPI YoY; Random Forest sits at 1.26. ARIMAX explains over 80% of test-set variance against Random Forest's 65%.

The gap comes down to one structural fact: inflation is serially correlated, and ARIMAX's autoregressive term gives it direct access to that momentum, while Random Forest treats each month as an independent draw. You can see this clearly in the prediction plots; ARIMAX tracks the COVID-era dip and the 2021-2022 spike with real fidelity, while Random Forest consistently undershoots peaks and overshoots troughs during that same volatile stretch. Residuals confirm it: ARIMAX's are tight and roughly symmetric around zero (σ = 0.92), Random Forest's carry heavier tails and a slight negative skew.

That said, Random Forest isn't wasted effort. Its feature importance output is the most useful artifact of the whole comparison; it independently confirms that wage growth and producer prices are the dominant transmission channels into CPI, which lines up with the wage-price spiral and cost-push literature this project leans on.

## Takeaway

ARIMAX is the stronger predictive model here, full stop. But the two approaches aren't really competing, they're answering different questions. ARIMAX tells you where inflation is going; Random Forest tells you why. A natural next step would be to use Random Forest's feature ranking to inform variable selection for the ARIMAX exogenous set, getting the predictive strength of one and the interpretability of the other in the same pipeline.

## Repository contents

- `*.ipynb` — Jupyter notebook with the full implementation: data loading, cleaning, YoY transformation, multicollinearity check, train/test split, ARIMAX and Random Forest estimation, and evaluation
- Dataset (FRED pull, cleaned and transformed; 686 monthly observations, 1969-2026)
- `ARIMAX and RANDOM FOREST PRED vs ACTUAL.png` — predicted vs. actual CPI_YoY on the test set, month by month
- `error metrics.png` — side-by-side comparison of error metrics and R² for both models
- `fig_residuals.png` — residual distribution overlay for ARIMAX vs. Random Forest
- `Feature Importance.png` — Random Forest feature importance chart

## Tools used

Python (statsmodels for SARIMAX, scikit-learn for Random Forest, pandas for data wrangling).
