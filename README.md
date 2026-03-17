# When the Lights Go Out: Predicting Major US Power Outage Severity

**Name:** Zack Mosley

---

## Introduction

Power outages affect millions of Americans every year, disrupting daily life, businesses, and critical infrastructure. Understanding what drives outage duration could help utilities better prepare for and respond to future events. This project uses a dataset of major power outages in the United States from 2000 to 2016, containing 1,534 rows and covering outage causes, climate conditions, geographic information, and economic data.

The central question driving this analysis is: **what factors best predict how long a power outage will last?**

The columns most relevant to this question are:

| Column | Description |
|---|---|
| `OUTAGE.DURATION` | Duration of the outage in minutes (prediction target) |
| `CAUSE.CATEGORY` | High level category of the outage cause |
| `CAUSE.CATEGORY.DETAIL` | Detailed description of the outage cause |
| `CLIMATE.CATEGORY` | Climate episode during the outage (warm, cold, or normal) |
| `CLIMATE.REGION` | U.S. climate region where the outage occurred |
| `U.S._STATE` | State where the outage occurred |
| `NERC.REGION` | North American Electric Reliability Corporation region |
| `ANOMALY.LEVEL` | Oceanic El Nino/La Nina index indicating climate anomaly severity |
| `CUSTOMERS.AFFECTED` | Number of customers affected by the outage |
| `DEMAND.LOSS.MW` | Amount of peak demand lost in megawatts |
| `TOTAL.CUSTOMERS` | Total customers served in the state |
| `POPDEN_URBAN` | Population density of urban areas in the state |
| `PC.REALGSP.STATE` | Per capita real gross state product |
| `UTIL.CONTRI` | Utility industry contribution to total state GDP |
| `OUTAGE.START` | Combined timestamp of outage start date and time |
| `OUTAGE.RESTORATION` | Combined timestamp of power restoration date and time |

---

## Data Cleaning and Exploratory Data Analysis

### Data Cleaning

The raw dataset required several cleaning steps before analysis could begin.

**Dropping rows with missing start times:** 9 rows were missing `OUTAGE.START.DATE` or `OUTAGE.START.TIME`. Since these timestamps are fundamental to understanding when outages occurred and constructing time-based features, these rows were dropped entirely.

**Dropping invalid durations:** 58 rows had a missing `OUTAGE.DURATION` and 78 rows had a duration of exactly 0 minutes. Rows with missing duration have no restoration time recorded either, making recovery impossible. Zero-duration outages are almost certainly recording errors rather than real events. Since duration is our prediction target, both were dropped.

**Dropping non-continental states:** 6 rows corresponded to outages in Hawaii and Alaska. These states are not part of the continental U.S. climate region system and would never have valid `CLIMATE.REGION` values, making them incompatible with the rest of the dataset.

**Combining timestamps:** `OUTAGE.START.DATE` and `OUTAGE.START.TIME` were combined into a single `OUTAGE.START` timestamp, and similarly for restoration. This makes time-based feature engineering cleaner and avoids redundant columns.

After cleaning, the dataset contains **1,393 rows** and **23 columns**.

Here are the first few rows of the cleaned dataset:

|   YEAR |   MONTH | U.S._STATE   | CAUSE.CATEGORY     | CLIMATE.CATEGORY   |   OUTAGE.DURATION | CUSTOMERS.AFFECTED   | OUTAGE.START              |
|-------:|--------:|:-------------|:-------------------|:-------------------|------------------:|:---------------------|:--------------------------|
|   2011 |       7 | Minnesota    | severe weather     | normal             |              3060 | 70000.0              | 2011-07-01 17:00:00-01:00 |
|   2014 |       5 | Minnesota    | intentional attack | normal             |                 1 | —                    | 2014-05-11 18:38:00-01:00 |
|   2010 |      10 | Minnesota    | severe weather     | cold               |              3000 | 70000.0              | 2010-10-26 20:00:00-01:00 |
|   2012 |       6 | Minnesota    | severe weather     | normal             |              2550 | 68200.0              | 2012-06-19 04:30:00-01:00 |
|   2015 |       7 | Minnesota    | severe weather     | warm               |              1740 | 250000.0             | 2015-07-18 02:00:00-01:00 |

### Exploratory Data Analysis

### Univariate Analysis

The distribution of outage duration is heavily right-skewed, with the vast majority of outages resolving within 5,000 minutes and a small number of extreme events stretching past 15,000 minutes. This skew motivates framing the prediction problem as binary classification (severe vs. not severe) rather than predicting exact duration.

<iframe
  src="figures/univariate/outage_duration_dist.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

### Bivariate Analysis

Fuel supply emergencies and severe weather stand out as the causes most associated with longer outage durations, while intentional attacks tend to resolve much more quickly. This pattern suggests that cause category carries meaningful signal for predicting whether an outage will become severe.

<iframe
  src="figures/bivariate/duration_by_cause.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

### Interesting Aggregates

The pivot table below shows mean outage duration (in hours) grouped by cause category and climate condition. Fuel supply emergencies stand out dramatically — lasting 290 hours during cold periods and 380 hours during warm periods — suggesting extreme climate conditions severely compound the difficulty of resolving fuel-related outages. Intentional attacks resolve quickly regardless of climate, while severe weather shows a steady increase in duration as conditions warm.

<iframe
  src="figures/aggregation/cause_climate_pivot.html"
  width="800"
  height="320"
  frameborder="0"
></iframe>

---

## Assessment of Missingness

### NMAR Analysis

`CUSTOMERS.AFFECTED` is NMAR. Utilities may not report the number of affected customers when an outage is small or localized, meaning the missingness is related to the value itself. Smaller outages are less likely to have customer counts recorded. To make this MAR, I would need additional data on reporting thresholds used by each utility, which would explain why some outages have missing customer counts regardless of outage size.

### Missingness Dependency

I analyzed whether the missingness of `CUSTOMERS.AFFECTED` depends on other columns using permutation tests with a test statistic of max minus min missingness rate across groups.

**Depends on: `CAUSE.CATEGORY` (p = 0.0)**
The observed difference in missingness rates across cause categories was 0.806, far outside the range of what would occur by chance. Intentional attacks have very high rates of missing customer counts, while fuel supply emergencies almost always have them recorded. This suggests utilities report customer impact differently depending on the type of event.

**Does not depend on: `ANOMALY.LEVEL` (p = 0.502)**
The observed difference in missingness rates across anomaly levels was 0.502, well within the range expected by chance. Climate conditions do not appear to influence whether customer counts get recorded.

<iframe
  src="figures/missingness/cause_missingness_dist.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

---

## Hypothesis Testing

**Null Hypothesis:** The mean outage duration during abnormal climate conditions (warm or cold) is equal to the mean outage duration during normal conditions.

**Alternative Hypothesis:** The mean outage duration during abnormal climate conditions is greater than the mean outage duration during normal conditions.

**Test Statistic:** Difference in mean outage duration (in hours) between abnormal and normal climate conditions. This is a good choice because there is a clear directional hypothesis and the difference in means directly measures what we care about.

**Significance Level:** 0.05

**Results:** The observed difference was 3.49 hours. After running 5,000 permutations, the p-value was 0.267.

**Conclusion:** I fail to reject the null hypothesis. Although outages during abnormal climate conditions lasted approximately 3.49 hours longer on average, this difference is not statistically significant and could reasonably have occurred due to random variation. There is not sufficient evidence to conclude that abnormal climate conditions lead to longer outage durations.

<iframe
  src="figures/hypothesis/climate_permutation_dist.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

---

## Framing a Prediction Problem

**Prediction Problem:** Can we predict, using only information available at the time an outage begins, whether the outage will become severe (defined as lasting 12 or more hours)?

**Type:** Binary classification. The two classes are Severe (outage lasting 12 or more hours) and Not Severe.

**Response Variable:** `SEVERE_OUTAGE`, derived from `OUTAGE.DURATION`. This was chosen over predicting exact duration because utilities care most about identifying outages that will become major events, not the precise number of minutes. A binary flag is more actionable for resource allocation decisions.

**Time of Prediction:** Only features available at the moment an outage is reported are used, including cause category, climate conditions, geographic information, time of onset, and state-level economic and population data. Restoration time, total duration, customers affected, and demand loss are explicitly excluded since these are only known after the outage has already progressed.

**Evaluation Metric:** I use F1-score and PR AUC (Average Precision). Accuracy is a poor choice here because even a naive model that always predicts Not Severe could appear reasonable. F1-score balances precision and recall, and PR AUC captures model performance across all classification thresholds, both of which are more meaningful when the cost of missing a severe outage is high.

---

## Baseline Model

The baseline model is a logistic regression classifier trained on 17 features available at outage onset. Of these, 10 are quantitative (year, month, anomaly level, and state-level economic and population indicators), 0 are ordinal, and 7 are nominal (cause category, climate category, state, and grid region). Nominal features were one-hot encoded, missing numeric values were median-imputed, and missing categorical values were filled with the most frequent value, all within a single sklearn Pipeline.

**Performance:** Trained on outages before 2015 and tested on 2015 onward, the model achieved 70.4% accuracy, an F1-score of 0.536 on the severe class, and a PR AUC of 0.60.

**Assessment:** While 70% accuracy sounds reasonable, the F1 of 0.54 on the severe class shows the model struggles to reliably identify outages that will become major events. This is not yet a good model for real utility decision-making, but it establishes the benchmark for the final model to improve upon.

---

## Final Model

### Feature Engineering

Five new features were engineered on top of the baseline feature set:

**`IS_WEEKEND`**: derived from the outage start timestamp. Utility crews operate on reduced staffing on weekends, meaning dispatch and coordination take longer. An outage beginning Saturday morning faces structurally slower response than one beginning Tuesday morning, making weekend onset a genuine predictor of severity from the data generating process.

**`IS_SUMMER`**: flags outages occurring in June, July, or August. Summer represents peak demand season when transformers and transmission lines are already operating near capacity. When failures occur under high-load conditions, cascading effects are more likely and restoration is more complex, making summer outages more prone to becoming severe.

**`LOG_TOTAL_CUSTOMERS`**: log transformation of the raw customer count. Since `TOTAL.CUSTOMERS` is heavily right-skewed, a few very large states would otherwise dominate the model. The log scale captures the real-world relationship more accurately -- the difference between a state serving 1M vs 2M customers matters more than the difference between 10M vs 11M, and the log transformation reflects this diminishing effect.

**`STATE_SEVERE_RATE`**: the historical rate of severe outages per state, computed only from training data to prevent leakage. This feature encodes structural characteristics of each state's grid -- aging infrastructure, geographic exposure to weather, utility investment levels, and regulatory environment -- that collectively determine how often outages escalate. A state with a history of severe outages is not just unlucky; it has underlying conditions that make severity more likely, and this rate captures that signal directly.

**`CAUSE_CLIMATE`**: an interaction term combining cause category and climate category. The data generating process for outage severity is not additive -- a fuel supply emergency during a warm El Nino period is fundamentally different from the same cause during normal conditions because demand patterns, fuel availability, and grid stress all shift together. Encoding the combination as a single feature lets the model learn these joint effects rather than treating cause and climate as independent signals.

### Modeling Algorithm and Hyperparameter Tuning

The final model is a **LightGBM classifier**, selected after comparing logistic regression, random forest, and LightGBM out of the box. LightGBM achieved the highest PR AUC of 0.6804 without any tuning, making it the strongest candidate to optimize further.

Hyperparameters were selected using `GridSearchCV` with 3-fold cross validation, optimizing for F1-score on the severe class. The best hyperparameters found were:

| Hyperparameter | Best Value |
|---|---|
| `n_estimators` | 50 |
| `learning_rate` | 0.01 |
| `num_leaves` | 5 |
| `min_child_samples` | 10 |
| `subsample` | 0.6 |

### Performance Improvement

| Metric | Baseline (Logistic Regression) | Final (LightGBM) |
|---|---|---|
| Accuracy | 70.4% | 80.9% |
| F1 (Severe) | 0.536 | 0.729 |
| PR AUC | 0.60 | 0.6466 |

The final model meaningfully improved across all metrics. The most significant gain was on the severe class F1 score, improving from 0.536 to 0.729, which is the metric that matters most for real utility decision-making.

<iframe
  src="figures/final_model/confusion_matrix.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

---

## Fairness Analysis

I evaluated whether the final model performs equally well for outages in high-density urban states versus lower-density suburban and rural states. If the model systematically underperforms for less-populated areas, utilities in those regions may receive less reliable early warning about severe outages despite often having fewer resources to respond.

**Group X (Low Density):** States with `POPDEN_URBAN` < 1,600 persons per square mile (suburban and rural). Following the U.S. Census Bureau's finding that the average population density inside U.S. cities is approximately 1,600 people per square mile, this threshold separates clearly urban states from the rest.

**Group Y (High Density):** States with `POPDEN_URBAN` >= 1,600 persons per square mile (urban).

**Evaluation Metric:** F1 score on the Severe class, the class that matters most for utility response planning.

**Null Hypothesis:** The model is fair. Its F1 score for low density and high density states are roughly the same, and any differences are due to random chance.

**Alternative Hypothesis:** The model is unfair. Its F1 score is lower for low density states than for high density states.

**Test Statistic:** Difference in F1 score (high density minus low density).

**Significance Level:** 0.05

| Metric | Value |
|---|---|
| F1 Score (High Density) | 0.7191 |
| F1 Score (Low Density) | 0.8571 |
| Observed Difference | -0.1380 |
| P-value | 0.836 |
| Significance Level | 0.05 |

**Conclusion:** I fail to reject the null hypothesis (p = 0.836). The observed difference of -0.138 in F1 score between high and low density states is not statistically significant, as differences of this magnitude occur frequently by random chance alone. There is not sufficient evidence to conclude that the final model performs unfairly across urban density groups. Interestingly, the model actually performs slightly better for low density states (F1 = 0.857) than high density states (F1 = 0.719), though this difference is not meaningful given the small test set size.

<iframe
  src="figures/fairness/fairness_permutation_dist.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>