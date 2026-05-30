# Complaint Volume Forecasting
[👉 Click Here to Open this Notebook in Google Colab](https://colab.research.google.com/github/Lua-Matlab-Python-R-J2EE/ombudsman-complaints-forecast/blob/main/notebooks/pds_task.ipynb)

## Overview
- Forecasts daily complaint volumes for the next 90 days using 3 years of historical operational data (2023–2025).
- Supports capacity planning (deciding how much work the team can physically handle), triage resourcing (deciding who fixes urgent/unexpected issues right now), and team prioritisation (deciding which planned tasks are the most important).

---

## Quick Start & Installation Instructions for Reproducibility

Follow these steps to clone the repository, set up an isolated virtual environment, install dependencies, and run the pipeline from scratch:

### Windows (Command Prompt)

```cmd
:: ==============================================================================
:: STEP 1: NAVIGATE AND CLONE THE REPOSITORY
:: ==============================================================================
:: 1. Change directory to your preferred workspace
cd C:\path\to\your\TEST

:: 2. Clone the project repository from GitHub
git clone https://github.com/Lua-Matlab-Python-R-J2EE/ombudsman-complaints-forecast

:: 3. Enter the project directory
cd ombudsman-complaints-forecast

:: ==============================================================================
:: STEP 2: CREATE AND ACTIVATE VIRTUAL ENVIRONMENT
:: ==============================================================================
:: 4. Create an isolated virtual environment to prevent package conflicts
python -m venv test_env

:: 5. Activate the virtual environment
test_env\Scripts\activate.bat

:: ==============================================================================
:: STEP 3: INSTALL DEPENDENCIES AND LAUNCH
:: ==============================================================================
:: 6. Upgrade the package installer, install dependencies, and ensure Jupyter is available
pip install --upgrade pip
pip install -r requirements.txt
pip install notebook

:: 7. Launch the notebook environment
jupyter notebook notebooks/pds_task.ipynb
```

### macOS (Terminal)

```bash
# ==============================================================================
# STEP 1: NAVIGATE AND CLONE THE REPOSITORY
# ==============================================================================
# 1. Change directory to your preferred workspace
cd /path/to/your/TEST

# 2. Clone the project repository from GitHub
git clone https://github.com/Lua-Matlab-Python-R-J2EE/ombudsman-complaints-forecast

# 3. Enter the project directory
cd ombudsman-complaints-forecast

# ==============================================================================
# STEP 2: CREATE AND ACTIVATE VIRTUAL ENVIRONMENT
# ==============================================================================
# 4. Create an isolated virtual environment to prevent package conflicts
python3 -m venv test_env

# 5. Activate the virtual environment
source test_env/bin/activate

# ==============================================================================
# STEP 3: INSTALL DEPENDENCIES AND LAUNCH
# ==============================================================================
# 6. Upgrade the package installer, install dependencies, and ensure Jupyter is available
pip install --upgrade pip
pip install -r requirements.txt
pip install notebook

# 7. Launch the notebook environment
jupyter notebook notebooks/pds_task.ipynb
```

### Linux (Terminal)

```bash
# ==============================================================================
# STEP 1: NAVIGATE AND CLONE THE REPOSITORY
# ==============================================================================
# 1. Change directory to your preferred workspace
cd /path/to/your/TEST

# 2. Clone the project repository from GitHub
git clone https://github.com/Lua-Matlab-Python-R-J2EE/ombudsman-complaints-forecast

# 3. Enter the project directory
cd ombudsman-complaints-forecast

# ==============================================================================
# STEP 2: CREATE AND ACTIVATE VIRTUAL ENVIRONMENT
# ==============================================================================
# 4. Create an isolated virtual environment to prevent package conflicts
python3 -m venv test_env

# 5. Activate the virtual environment
source test_env/bin/activate

# ==============================================================================
# STEP 3: INSTALL DEPENDENCIES AND LAUNCH
# ==============================================================================
# 6. Upgrade the package installer, install dependencies, and ensure Jupyter is available
pip install --upgrade pip
pip install -r requirements.txt
pip install notebook

# 7. Launch the notebook environment
jupyter notebook notebooks/pds_task.ipynb
```

---


### Reproducibility Verification
To ensure identical outputs, perform the following validation steps inside the Jupyter interface:
1. Click **Kernel** from the top menu bar.
2. Select **Restart & Run All** to execute the pipeline sequentially from a fresh state.
3. Once execution completes, the pipeline automatically saves the final clean predictions to `outputs/forecast_90_days.csv`.

---

## Project Structure
```text
ombudsman-complaints-forecast/
├── config.py
├── data/                          # Raw input dataset
│   └── data.xlsx                  # Dataset used in this notebook
├── notebooks/                     # Development notebooks
│   └── pds_task.ipynb             # Main notebook (end-to-end)
├── outputs/                       # Forecast CSV output
│   ├── forecast_90_days.csv       # 90-day forecast (XGBoost + Prophet)
│   └── forecast_chart.png         # XGBoost + Prophet results display
├── requirements.txt               # Python dependencies
├── .gitignore               
└── README.md
```

--- 

## Approach & Process

### 1. Data Cleaning
- Dropped 10 rows where target variable (`complaints`) was missing (~1% of data).
- Forward-filled operational features (`staffing_level_fte`, `backlog_days`, `channel_mix_index`) -> assumes short-term stability.
- Dropped the first 7 rows of the dataset to eliminate NaN values caused by lag and rolling mean calculations.
- Dropped `centered_7d_mean` -> uses future data, causes leakage.
- Converted `complaints` to integer after null removal.


### 2. Exploratory Data Analysis
- Trend: Clear upward trajectory: complaints grew from ~70/day (2023) to ~130/day (2025).
- Weak bimodal seasonality: spring (Mar-May) and autumn (Oct-Nov) peaks.
- High day-to-day variance throughout: difficult to predict individual days.

### 3. Feature Engineering

| Feature | Description |
|---|---|
| `day_of_week` | Day of the week (0=Monday, 6=Sunday) |
| `month` | Month of the year (1–12) to catch seasonal trends |
| `year` | Calendar year to track long-term growth |
| `lag_1` | Yesterday's complaint count (recent momentum) |
| `lag_7` | Complaint count from the same day last week |
| `rolling_mean_7` | Average complaints over the last 7 days (smooths spikes) |


### 4. Train / Validation / Test Split (By Timeline)

The data is split by timeline to prevent looking ahead into the future. Each split matches our 90-day forecasting window:

-   **Train Set:** 2023-01-08 -to- 2025-07-04 (856 rows)
-   **Validation Set:** 2025-07-05 -to- 2025-10-02 (90 calendar days): Used to tune the model via Optuna.
-   **Test Set:** 2025-10-03 -to- 2025-12-31 (90 calendar days): Used for the final score evaluation.

> Note: All datasets include weekends. The model uses the 'is_weekend' flag to learn that complaint volumes drop on Saturdays/Sundays.

---

## Models

### XGBoost
- Extreme Gradient Boosting -> A tree-based machine learning model.
- Tuned using Optuna to minimize prediction errors.
- Forecasts 90 days ahead by feeding each prediction back into the model.
- Drawback: Over time, predictions stop fluctuating and just stick to the training data average.

**Input features/regressors:**

| Feature | Description | Source |
|---|---|---|
| `is_weekend` | 1 if Saturday/Sunday, else 0 | Raw data |
| `bank_holiday_flag` | 1 if public holiday, else 0 | Raw data |
| `staffing_level_fte` | Staffing level in Full Time Equivalents (FTE) | Raw data |
| `backlog_days` | Number of days behind on resolving complaints | Raw data |
| `media_mentions` | Count of media/social mentions that day | Raw data |
| `channel_mix_index` | 0–100 index representing complaint channel distribution | Raw data |
| `day_of_week` | Day of the week (0=Monday, 6=Sunday) | Engineered |
| `month` | Month of the year (1–12) to catch seasonal trends | Engineered |
| `year` | Calendar year to track long-term growth | Engineered |
| `lag_1` | Yesterday's complaint count (recent momentum) | Engineered |
| `lag_7` | Complaint count from the same day last week | Engineered |
| `rolling_mean_7` | Average complaints over the last 7 days (smooths spikes) | Engineered |
| `complaints` | daily count of complaints received | **Target variable** |

### Prophet (Facebook)
- Handles trends and seasons automatically: No manual setup needed to track yearly or weekly patterns.
- Fast forecasting: Predicts all 90 days at once without needing step-by-step loops.
- Shows best and worst cases: Provides high and low prediction ranges, which helps with staffing plans.
- Uses official UK bank holiday data: We manually feed in official holiday dates so the model knows exactly when offices are closed.

**Input features/regressors:**

| Feature  | Description | Source | 
|---|---|---|
| `ds` | The specific date of the day being tracked | Calendar Date | 
| `holidays` |  Our custom list of bank holidays used to adjust for office closures | UK Holiday List |
| `y` or `complaints` | daily count of complaints received | **Target variable** |

---

## Evaluation & Metrics

We evaluate performance using metrics that directly map to real-world staffing and capacity risks:

-   **MAE (Mean Absolute Error => Average Daily Miss => Main Metric):** Shows the average number of daily complaints the model misses. Since it maps 1:1 to actual casework volume, our planning teams can use this number directly to calculate needed staff numbers. It works reliably on low-volume weekends.
-   **RMSE (Root Mean Squared Error => Large Error Tracker => Variance Tracker):** Focuses heavily on big mistakes. Comparing this to our main metric tells us if a model risks severely underestimating staffing needs during sudden, massive demand surges.
-   **MAPE (Mean Absolute Percentage Error => Percentage Error => Rejected Metric):** Included only for reference but ignored for final decisions. Because weekend volumes drop close to zero, even minor errors look like massive percentage spikes, which artificially distorts the true accuracy score.

### Performance Summary Scorecard

| Metric | XGBoost (Tuned) | Meta Prophet | Winner | Business Benefit |
| :--- | :---: | :---: | :---: | :--- |
| **Average Daily Miss (MAE)** | 29.63 | **23.47** | **Meta Prophet** | ~21% better accuracy for staff planning |
| **Large Error Risk (RMSE)** | 36.32 | **29.40** | **Meta Prophet** | ~19% fewer surprises during high-volume spikes |
| **Percentage Error (MAPE)** | 34.69% | **32.85%** | **Meta Prophet** | N/A – rejected due to low weekend volumes |

![Prophet vs XGBoost 90-Day Forecast](outputs/forecast_chart.png)

### Final Model Selection Decision
- Meta Prophet is the stronger candidate of the two tested models. While both models still carry a high margin of error due to the noisy daily data, Meta Prophet provides a more useful baseline because it captures real fluctuations and provides clear high-and-low prediction ranges.
- For staff scheduling, relying on a model that smooths out predictions creates tracking vulnerabilities (missing sudden volume spikes leaves teams dangerously understaffed). Meta Prophet gives better baseline accuracy because it models weekly and yearly patterns all at once instead of guessing day-by-day, captures real fluctuations, and provides explicit high-and-low prediction ranges. These best and worst-case scenarios help managers with risk-managed staff planning.

---

## Limitations

### XGBoost Constraints
1. **Loss of Detail Over Time:** The model's prediction variety drops significantly after 14 days. As it relies on its own past guesses instead of real data, the numbers quickly flatten out.
2. **Snowballing Errors:** Because each prediction relies on the previous one, mistakes accumulate, making the final days of the 90-day forecast the least reliable.
3. **Cannot Predict New Trends:** Tree-based models cannot project growth higher than what they have already seen. When it reads the year 2026, it caps the trend at 2025 levels, artificially flattening long-term growth. 
   * **Alternative Solution:** Could use a Hybrid Model, which combines a trend-tracking model (like Prophet or Linear Regression) with XGBoost so growth and daily spikes are predicted together.

### Data Gaps & Assumptions
4. **Unexplained Drops (Both Models):** The models cannot predict random events like IT system outages or strikes that cause complaint numbers to drop to zero on normal workdays.
5. **Missing Data History (Both Models):** The dataset only spans about 3 years, leaving fewer than 900 days for training. This short window makes it harder for both models to learn reliable, long-term yearly patterns.
6. **Static Staffing and Backlogs:** XGBoost assumes the staff levels and backlogs will stay exactly the same for the next 90 days. **Meta Prophet Advantage:** Completely unaffected because it bypasses these operational features entirely and relies purely on time patterns.
7. **No Future News Tracking:** XGBoost assumes there will be zero social media or news mentions in the future, meaning it will under-forecast if a major public PR event occurs. **Meta Prophet Advantage:** Completely unaffected because it does not use media features to build its predictions.

### Holiday Flaws
8. **Incorrect Bank Holiday Flags:** The source data contains multiple errors regarding bank holiday flagging across 2023 and 2025. Specifically, three rows in 2023 have inverted flags (e.g., mislabeling New Year's Day and King Charles III's Coronation holiday), and a May 2025 bank holiday was completely missed. These inaccuracies confuse the model regarding how holidays impact volume drops.
9. **Unverified Historical Calendars:** Holiday flags for 2023–2025 were taken as-is from the data without double-checking. Only the upcoming 2026 forecast holidays were verified against official government calendars.

### Metric Flaws
10. **Misleading Percentage Scores:** Percentage error breaks down when actual numbers are very small. **Example:** If the office receives only 2 complaints on a Sunday and the model guesses 6, it is only off by 4 cases. However, mathematically, this registers as a massive **200% error spike**. Averaging these artificial spikes makes the model look far worse than it actually is.

---

## Abbreviations

| Abbreviation | Full Form |
|---|---|
| MAE | Mean Absolute Error |
| RMSE | Root Mean Squared Error |
| MAPE | Mean Absolute Percentage Error |
| FTE | Full Time Equivalent |
| GB | Gradient Boosting |
| XGB | XGBoost (Extreme Gradient Boosting) |
| CSV | Comma Separated Values |
| UK | United Kingdom |
| std | Standard Deviation |