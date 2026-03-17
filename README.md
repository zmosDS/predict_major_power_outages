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

This table groups outages by both cause category and climate condition to show how average duration varies across combinations of the two. Fuel supply emergencies stand out dramatically, particularly during warm and cold climate periods, suggesting that extreme climate conditions compound the difficulty of resolving certain types of outages. Severe weather outages show a more modest but consistent increase in duration across all climate categories.

<iframe
  src="figures/aggregation/duration_by_cause_climate.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

---

## Assessment of Missingness

### NMAR Analysis

`CUSTOMERS.AFFECTED` is NMAR. Utilities may not report the number of affected customers when an outage is small or localized, meaning the missingness is related to the value itself. Smaller outages are less likely to have customer counts recorded. To make this MAR, we would need additional data on reporting thresholds used by each utility, which would explain why some outages have missing customer counts regardless of outage size.

### Missingness Dependency

We analyzed whether the missingness of `CUSTOMERS.AFFECTED` depends on other columns using permutation tests with a test statistic of max minus min missingness rate across groups.

**Depends on: `CAUSE.CATEGORY` (p = 0.0)**
The observed difference in missingness rates across cause categories was 0.806, far outside the range of what would occur by chance. Intentional attacks have very high rates of missing customer counts, while fuel supply emergencies almost always have them recorded. This suggests utilities report customer impact differently depending on the type of event.

**Does not depend on: `ANOMALY.LEVEL` (p = 0.514)**
The observed difference in missingness rates across anomaly levels was 0.538, well within the range expected by chance. Climate conditions do not appear to influence whether customer counts get recorded.

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

**Test Statistic:** Difference in mean outage duration (in hours) between abnormal and normal climate conditions. This is a good choice because we have a clear directional hypothesis and the difference in means directly measures what we care about.

**Significance Level:** 0.05

**Results:** The observed difference was 3.49 hours. After running 5,000 permutations, the p-value was 0.267.

**Conclusion:** We fail to reject the null hypothesis. Although outages during abnormal climate conditions lasted approximately 3.49 hours longer on average, this difference is not statistically significant and could reasonably have occurred due to random variation. We do not find sufficient evidence to conclude that abnormal climate conditions lead to longer outage durations.

<iframe
  src="figures/hypothesis/climate_permutation_dist.html"
  width="800"
  height="500"
  frameborder="0"
></iframe>

---

## The Prediction Problem

**Prediction Problem:** Can we predict, using only information available at the time an outage begins, whether the outage will become severe (defined as lasting 12 or more hours)?

**Type:** Binary classification. The two classes are Severe (outage lasting 12 or more hours) and Not Severe.

**Response Variable:** `SEVERE_OUTAGE`, derived from `OUTAGE.DURATION`. We chose this over predicting exact duration because utilities care most about identifying outages that will become major events, not the precise number of minutes. A binary flag is more actionable for resource allocation decisions.

**Time of Prediction:** We only use features available at the moment an outage is reported, including cause category, climate conditions, geographic information, time of onset, and state-level economic and population data. We explicitly exclude restoration time, total duration, customers affected, and demand loss since these are only known after the outage has already progressed.

**Evaluation Metric:** We use F1-score and PR AUC (Average Precision). Accuracy is a poor choice here because even a naive model that always predicts Not Severe could appear reasonable. F1-score balances precision and recall, and PR AUC captures model performance across all classification thresholds, both of which are more meaningful when the cost of missing a severe outage is high.

---

