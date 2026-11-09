# DMAPRIME-QR COVID-19 Hospitalization Forecast

DMAPRIME-QR forecasts weekly COVID-19 hospital admissions for the United States or an individual state. It combines quantile regression and historical analogs to produce 23 forecast quantiles for horizons 0–3 in the COVID-19 Forecast Hub format.

## Data

The script downloads public data from the [CDC COVID-19 Forecast Hub](https://github.com/CDCgov/covid19-forecast-hub): weekly hospitalization observations, historical data releases, location information, and the COVID-19 emergency department (ED) visit share. It uses the most recent release for the forecast and uses historical releases to estimate reporting revisions.

## Method

1. **Prepare recent observations.** In live mode, the model adjusts the latest three observed hospitalization weeks for reporting revisions. For each lag, it estimates a multiplier from the median ratio between older releases and a later release, when sufficient historical data are available.
2. **Build predictors.** Admissions are transformed as `log(1 + admissions)`. Features describe recent levels, lags, growth and acceleration, rolling means and variability, distance from recent peaks, seasonality, and the share of states with rising admissions. When sufficient ED data are available, changes in the COVID-19 ED visit share are included as a leading predictor.
3. **Identify the current phase.** The model uses recent growth, decline, and proximity to a recent peak to distinguish phases of the epidemic curve. A seasonal gate limits peak and decline flags when activity is low relative to typical past seasonal peaks. The near-peak rule accounts for estimated count noise.
4. **Predict with regression and analogs.** A median quantile regression predicts the next week's log admissions. The model also finds past weeks with similar standardized features, giving extra weight to trend, acceleration, and peak or decline features. It weights analogs by similarity and phase, then combines their subsequent changes with the regression forecast. This step is repeated recursively to produce the forecast path.
5. **Adjust the forecast path.** The default configuration also fits models pooled across states after rescaling their counts to the target population. Their predictions may be blended with the recursive path using weights chosen on validation data. A tuned growth cap can damp rapid increases. Other optional adjustments are controlled by switches in the script.
6. **Construct forecast quantiles.** By default, the script uses horizon-specific errors from historical validation forecast paths to calibrate uncertainty around the final median, when at least 20 paths are available per horizon. Otherwise it retains the model's error-based intervals. Forecasts are transformed back to admission counts, constrained to be nonnegative, and ordered so quantiles do not cross.

The tuning procedure uses historical forecast origins and evaluates candidate analog and blending settings on prior observations. Training labels are restricted to those available at each forecast origin. The optional backtest includes a future-data perturbation check for the tested origins. These safeguards do not guarantee performance on future data.

## Requirements

R and the following packages are required: `dplyr`, `lubridate`, `ggplot2`, `purrr`, `readr`, `quantreg`, `tidyr`, `slider`, `scales`, and `arrow`.

```r
install.packages(c("dplyr", "lubridate", "ggplot2", "purrr", "readr",
                   "quantreg", "tidyr", "slider", "scales", "arrow"))
```

An internet connection is required to download the hub data.

## Run

In `DMAPRIME-QR_COVIDHub.R`, set `TARGET` to `"US"` or a two-letter state code such as `"SC"`. Run the script from the project directory:

```r
source("DMAPRIME-QR_COVIDHub.R")
```

The default settings run a live forecast and save submission files under `output/CDC_submission/` relative to the R working directory. Each location is saved separately in `by_location/`; the combined CSV for a reference date is rebuilt from the saved location files for that date. Run the script once per desired location.

Set `RUN_MODE` and `RUN_STAGE` to `"backtest"` to run retrospective evaluation. Set `REUSE_TUNING` to `TRUE` to save and reuse tuning results in submission mode.

## Output

The combined CSV contains one row per location, horizon, and quantile, using the hub's `wk inc covid hosp` target. The script also produces forecast plots and checks the submission format.
