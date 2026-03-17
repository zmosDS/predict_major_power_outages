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