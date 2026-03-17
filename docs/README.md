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
| `CAUSE.CATEGORY` | High level category of the outage cause (e.g. severe weather, intentional attack) |
| `CLIMATE.CATEGORY` | Climate episode category during the outage (warm, cold, or normal) |
| `U.S._STATE` | State where the outage occurred |
| `NERC.REGION` | North American Electric Reliability Corporation region |
| `ANOMALY.LEVEL` | Oceanic El Nino/La Nina index indicating climate anomaly severity |
| `CUSTOMERS.AFFECTED` | Number of customers affected by the outage |
| `DEMAND.LOSS.MW` | Amount of peak demand lost in megawatts |