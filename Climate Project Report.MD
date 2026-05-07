# 🌍 Climate Change Public Discourse Analysis (2006–2019)
**End-to-End SQL & Power BI Project | PostgreSQL · Power BI · Data Analytics**

![My Image Description](https://github.com/R887645/-Climate-Change-Public-Discourse-Analysis-2006-2019-/blob/main/Climate%20Cange%20image.png?raw=true)
---

## 📌 Project Overview

This project presents a comprehensive end-to-end data analysis of global public discourse on climate change, built entirely without Python or external programming languages. Using **PostgreSQL 17** for all data engineering and **Microsoft Power BI** for interactive visualization, I analysed over **15.7 million tweets** spanning 13 years alongside a **4,913-record Global Disasters dataset** to answer two critical questions posed by a climate research organisation ahead of a policy briefing:

![My Image Description](https://github.com/R887645/-Climate-Change-Public-Discourse-Analysis-2006-2019-/blob/main/Climate%20Change%20Analysis.png?raw=true)

1. How have public **sentiment and stance** on climate change shifted over 13 years?
2. Which **topics and regions** are driving the most divisive and aggressive discourse?

The project demonstrates the full data analytics pipeline — from raw ingestion of a 1GB CSV file into PostgreSQL, through structured cleaning, transformation and advanced SQL analytics, to a published interactive Power BI dashboard connected live to PostgreSQL views.

---

## 🛠️ Tech Stack

| Tool | Purpose |
|---|---|
| PostgreSQL 17 | Data ingestion, cleaning, transformation, analytics |
| Power BI Desktop | Dashboard development, DAX measures |
| Power BI Service | Dashboard publishing and hosting |
| pgAdmin 4 | Database management |
| PowerShell | Large file import (1GB+ CSV) |

---

## 📂 Repository Structure

```
climate-change-discourse-analysis/
│
├── sql/
│   └── climate_analysis.sql          # Full SQL script — all phases
│
├── dashboard/
│   └── climate_dashboard.pbix        # Power BI file
│
├── report/
│   └── Climate_Change_Analysis.pdf   # Structured analysis report
│
└── README.md
```

---

## 🗄️ Datasets

| Dataset | Records | Source |
|---|---|---|
| Climate Change Twitter Dataset | 15,789,411 tweets | [Mendeley Data](https://data.mendeley.com/datasets/mw8yd7z9wc-2) |
| Global Disasters Dataset | 4,913 events | EM-DAT / disasters.csv |

**Twitter Dataset Columns:**
`created_at` · `id` · `lat` · `lng` · `topic` · `sentiment` · `stance` · `gender` · `temperature_avg` · `aggressiveness`

**Disasters Dataset Columns:**
`disaster_type` · `country` · `start_date` · `total_deaths` · `total_damages` · `no_affected`

---

## ⚙️ Project Pipeline

```
Phase 1 → Data Ingestion & Understanding
Phase 2 → Data Quality Assessment
Phase 3 → Data Cleaning & Validation
Phase 4 → Data Wrangling & Transformation (Views)
Phase 5 → Descriptive Analytics (SQL)
Phase 6 → Diagnostic Analytics (SQL)
Phase 7 → Power BI Dashboard
Phase 8 → Recommendations & Report
```

---

## 🧹 Phase 1–3: Data Ingestion, Quality & Cleaning

### Ingestion
The 1GB Twitter CSV was imported directly into PostgreSQL via PowerShell using the `\copy` command — bypassing pgAdmin's upload limitations entirely.

```sql
-- PowerShell import command
\copy climate_change_tweets_raw 
FROM 'path/to/Climate_Change_Twitter_Dataset.csv' 
WITH (FORMAT csv, HEADER true, DELIMITER ',')
```

### Quality Issues Found

| Column | Issue | Resolution |
|---|---|---|
| `lat` / `lng` | 10,481,873 missing (66.4%) | Flagged as `has_location` |
| `gender` | 586,875 undefined values | Standardised to `Unknown` |
| `temperature_avg` | 10,481,873 missing | Flagged as `has_temp_data` |
| `id` | Checked for duplicates | Zero duplicates confirmed |
| `sentiment` | Range check (-1 to +1) | All values within range |

### Cleaning Approach
Rather than deleting incomplete records, all 15,789,411 rows were retained. Missing values were handled through **flagging and standardisation** to preserve analytical completeness.

```sql
-- Sentiment categorisation
CASE
    WHEN sentiment > 0.05  THEN 'Positive'
    WHEN sentiment < -0.05 THEN 'Negative'
    ELSE 'Neutral'
END AS sentiment_category,

-- Gender standardisation
CASE
    WHEN LOWER(gender) = 'undefined' THEN 'Unknown'
    WHEN gender IS NULL THEN 'Unknown'
    ELSE gender
END AS gender
```

**Performance indexes created** on `tweet_year`, `stance`, `topic`, and `aggressiveness` to support fast Power BI queries against 15.7M rows.

![My Image Description](https://github.com/R887645/-Climate-Change-Public-Discourse-Analysis-2006-2019-/blob/main/Screenshot%202026-05-07%20102904.png?raw=true)

![My Image Description](https://github.com/R887645/-Climate-Change-Public-Discourse-Analysis-2006-2019-/blob/main/Screenshot%202026-05-07%20102913.png?raw=true)

![My Image Description](https://github.com/R887645/-Climate-Change-Public-Discourse-Analysis-2006-2019-/blob/main/Screenshot%202026-05-07%20103013.png?raw=true)

---

## 🔄 Phase 4: Data Wrangling & Transformation

Seven analytical views were created in PostgreSQL — each pre-aggregating the data for a specific business question and serving as a direct data source for Power BI.

| View | Purpose | Power BI Usage |
|---|---|---|
| `vw_sentiment_trend` | Annual sentiment shift | Line chart — Page 1 |
| `vw_stance_trend` | Believer/denier/neutral by year | Stacked bar — Page 1 |
| `vw_topic_aggressiveness` | Topic divisiveness scoring | Bar + table — Page 2 |
| `vw_regional_analysis` | Geotagged tweet patterns | Map visual — Page 2 |
| `vw_regional_analysis_agg` | Aggregated location clusters | Map bubbles — Page 2 |
| `vw_gender_analysis` | Gender sentiment breakdown | Donut + bar — Page 3 |
| `vw_disaster_sentiment` | Tweets × disasters JOIN by year | Dual axis — Page 3 |
| `vw_monthly_trends` | Monthly tweet volume | Line chart — Page 3 |

### Key JOIN — Tweets × Disasters

```sql
CREATE OR REPLACE VIEW vw_disaster_sentiment AS
SELECT
    t.tweet_year,
    COUNT(*) AS total_tweets,
    ROUND(AVG(t.sentiment)::NUMERIC, 4) AS avg_sentiment,
    COUNT(*) FILTER (WHERE t.aggressiveness = 'aggressive') AS aggressive_tweets,
    ROUND(COUNT(*) FILTER (WHERE t.aggressiveness = 'aggressive') 
        * 100.0 / COUNT(*), 2) AS aggressive_pct,
    COUNT(DISTINCT d.event_name) AS total_disasters,
    COALESCE(SUM(d.total_deaths), 0) AS total_deaths,
    COALESCE(SUM(d.total_damages), 0) AS total_damages
FROM climate_change_tweets_clean t
LEFT JOIN disasters_clean d ON t.tweet_year = d.disaster_year
GROUP BY t.tweet_year
ORDER BY t.tweet_year;
```

### Divisiveness Score

A custom composite metric was calculated for each topic combining aggressive percentage and denier concentration:

```sql
ROUND(
    (aggressive_pct + 
    (denier_count * 100.0 / NULLIF(total_tweets, 0)))::NUMERIC / 2, 2
) AS divisiveness_score
```

---

## 📊 Phase 5: Descriptive Analytics

### Overall Dataset Summary

| Metric | Value |
|---|---|
| Total Tweets | 15,789,411 |
| Years Covered | 2006 – 2019 (13 years) |
| Avg Sentiment Score | 0.003 (near neutral) |
| Aggressive Tweets | 4,527,267 (28.7%) |
| Geotagged Tweets | 5,307,538 (33.6%) |
| Unique Topics | 10 |

### Stance Distribution

| Stance | Count | % |
|---|---|---|
| Believer | 11,292,424 | 71.5% |
| Neutral | 3,305,601 | 20.9% |
| Denier | 1,191,386 | 7.5% |

The **believer-to-denier ratio of 9.48:1** has remained stable across the entire 13-year period — a strong and consistent signal of public acceptance of climate change.

### Sentiment Categories

| Category | % of All Tweets |
|---|---|
| Negative | 46.77% |
| Positive | 43.21% |
| Neutral | 10.02% |

### Topic Volume

| Topic | Tweets |
|---|---|
| Global Stance | 4,135,619 |
| Importance of Human Intervention | 2,594,941 |
| Weather Extremes | 2,464,814 |
| Politics | 1,809,583 |
| Donald Trump versus Science | 996,244 |


![My Image description](https://github.com/R887645/-Climate-Change-Public-Discourse-Analysis-2006-2019-/blob/main/Screenshot%202026-05-07%20103115.png?raw=true)

![My Image description](https://github.com/R887645/-Climate-Change-Public-Discourse-Analysis-2006-2019-/blob/main/Screenshot%202026-05-07%20103134.png?raw=true)

![My Image description](https://github.com/R887645/-Climate-Change-Public-Discourse-Analysis-2006-2019-/blob/main/Screenshot%202026-05-07%20103240.png?raw=true)

---

## 🔍 Phase 6: Diagnostic Analytics

### Correlation Analysis — SQL

```sql
SELECT
    tweet_year,
    avg_sentiment,
    aggressive_pct,
    total_disasters,
    ROUND(CORR(aggressive_pct, total_disasters::FLOAT) 
        OVER ()::NUMERIC, 4) AS aggression_disaster_correlation,
    ROUND(CORR(avg_sentiment, total_disasters::FLOAT) 
        OVER ()::NUMERIC, 4) AS sentiment_disaster_correlation
FROM vw_disaster_sentiment
ORDER BY tweet_year;
```

![My Image description](https://github.com/R887645/-Climate-Change-Public-Discourse-Analysis-2006-2019-/blob/main/Screenshot%202026-05-07%20103318.png?raw=true)


### Key Diagnostic Findings

**Finding 1 — Disaster events correlate with aggressive discourse**
A correlation of **0.72** was found between annual disaster event counts and tweet aggressiveness. The year 2018 recorded the highest disaster count at 54 events, coinciding with elevated aggressive discourse.

**Finding 2 — Politics drives the most divisive discourse**
Politics and Donald Trump versus Science score highest on the divisiveness index at **75.49** and **81.59** respectively. Both generate aggressive rates above 43%.

**Finding 3 — Gender does not drive aggressiveness**
Aggressiveness rates across all gender groups are within 0.6 percentage points:
- Male: 35.7% · Unknown: 35.4% · Female: 35.1%

This confirms aggression is topic-driven, not demographic.

**Finding 4 — Yearly aggressiveness ranking**
```sql
SELECT
    tweet_year,
    aggressive_pct,
    RANK() OVER (ORDER BY aggressive_pct DESC) AS aggression_rank
FROM vw_disaster_sentiment
ORDER BY aggressive_pct DESC;
-- Result: 2006 ranks #1 at 83% (low volume outlier)
-- Stable decline from 2008–2019
```

---

## 📈 Phase 7: Power BI Dashboard

![My Image Description](https://github.com/R887645/-Climate-Change-Public-Discourse-Analysis-2006-2019-/blob/main/climate%20data%20prep.png?raw=true)

![My Image Description](https://github.com/R887645/-Climate-Change-Public-Discourse-Analysis-2006-2019-/blob/main/climate_data%20modelling.png?raw=true)

The dashboard is structured across **3 interactive pages**, each answering a specific analytical question.

### Page 1 — Climate Change Public Sentiment & Stance Analysis
> *How have sentiment and stance shifted over 13 years?*

- **KPI Cards:** 16M total tweets · -0.47% avg sentiment · 28.67% aggressive · 9.48 believer/denier ratio
- **Line Chart:** Sentiment score trend 2006–2019
- **Stacked Bar:** Stance distribution by year (believers / deniers / neutrals)
- **Donut Chart:** Sentiment category breakdown
- **Slicer:** Year range filter

![My Image Description](https://github.com/R887645/-Climate-Change-Public-Discourse-Analysis-2006-2019-/blob/main/climage%20analysis%201.png?raw=true)


### Page 2 — Divisive Topics & Regional Aggression Hotspots
> *Which topics and regions drive the most aggressive discourse?*

- **KPI Cards:** 5M geotagged tweets · 36.22 divisiveness score · 71.52% believers · 7.55% deniers
- **Horizontal Bar:** Most aggressive climate topics ranked
- **Map Visual:** Regional aggression hotspots (lat/lng bubbles)
- **Table:** Topic divisiveness scorecard
- **Clustered Bar:** Stance distribution by topic

![My Image Description](https://github.com/R887645/-Climate-Change-Public-Discourse-Analysis-2006-2019-/blob/main/climate%20analysis%202.png?raw=true)


### Page 3 — Diagnostic Analysis: Why Does Aggressive Discourse Occur?
> *What factors explain aggressive discourse patterns?*

- **KPI Cards:** 496 total disasters · 716K total deaths · -12.06% YoY aggressive change · -66.26% correlation
- **Dual Axis Chart:** Disaster events vs public sentiment 2006–2019
- **Bar Chart:** Yearly aggressiveness ranking
- **Donut + Bar:** Tweet volume and aggressiveness by gender
- **Line Chart:** Monthly tweet volume trends

![My Image Description](https://github.com/R887645/-Climate-Change-Public-Discourse-Analysis-2006-2019-/blob/main/Screenshot%202026-05-07%20101600.png?raw=true)

### DAX Measures (Sample)

```dax
-- YoY Tweet Growth with directional arrow
YoY Tweet Growth = 
VAR Positive_Arrow = UNICHAR(9650)
VAR Negative_Arrow = UNICHAR(9660)
VAR Growth = [YoY Tweet Growth %]
RETURN
    IF(Growth > 0, Positive_Arrow, Negative_Arrow) & "  "
    & ROUND(Growth * 100, 2) & "%"

-- Believer to Denier Ratio
Believer to Denier Ratio = 
DIVIDE(
    SUM('public vw_stance_trend'[Believers]),
    SUM('public vw_stance_trend'[Deniers]),
    0
)

-- Divisiveness Score
Divisiveness Score = 
DIVIDE(
    SUM('public vw_topic_aggressiveness'[aggressive_count]) + 
    SUM('public vw_topic_aggressiveness'[Denier Count]),
    SUM('public vw_topic_aggressiveness'[total_tweets]),
    0
) * 100
```

---

## 💡 Key Findings & Recommendations

### Findings

| # | Finding |
|---|---|
| 1 | Believers dominate at 71.5% with a stable 9.48:1 ratio — public acceptance of climate change is strong |
| 2 | 28.7% of all tweets are aggressive — concentrated in political topics |
| 3 | Politics and Donald Trump vs Science score highest on divisiveness |
| 4 | Disaster events correlate with aggressive discourse (r = 0.72) |
| 5 | Gender has no meaningful effect on aggressiveness — it is topic-driven |
| 6 | Overall sentiment has remained near-neutral for 13 years |

### Recommendations

1. **De-politicise climate communications** — political framing generates the highest aggression and should be avoided in public-facing messaging
2. **Deploy proactive disaster communications** — prepare templated messaging before known high-risk disaster seasons
3. **Amplify the believer majority** — 11.3M believer voices represent an enormous untapped advocacy network
4. **Focus regional engagement** on North America, Western Europe and Australia where discourse is highest volume
5. **Acknowledge public mood honestly** — near-neutral sentiment for 13 years signals that optimistic framing is not resonating

---

---

## 📋 SQL Script Structure

```
SECTION 1:  Data Understanding & Preparation
SECTION 1B: Data Quality Assessment — Tweets
SECTION 1C: Data Cleaning — Tweets Table + Indexes
SECTION 1D: Data Quality Assessment — Disasters
SECTION 1E: Data Cleaning — Disasters Table + Indexes
SECTION 2:  Data Wrangling & Transformation
SECTION 2A: Table JOIN — Tweets × Disasters
SECTION 2B: Analytical Views (7 views)
SECTION 3:  Descriptive Analytics (7 queries)
SECTION 4:  Diagnostic Analytics (4 queries)
```

---

## 📄 Deliverables

| Deliverable | Description |
|---|---|
| `climate_analysis.sql` | Full commented SQL script — all phases |
| `climate_dashboard.pbix` | Interactive Power BI dashboard |
| `Climate_Change_Analysis.pdf` | Structured analysis report |
| `README.md` | This documentation |

---

## 🔗 Dashboard Link

> 🌐 **[View Live Dashboard on Power BI Service](https://app.powerbi.com/view?r=eyJrIjoiYmY1NWMwNTgtNmRiNC00ZGVjLTllMjUtMWE4YTZmNzEyZDFmIiwidCI6ImNmYzA4YzM2LWYxMGEtNDU1NS1iZjZiLWMwMjI4YzVlNjEzYSIsImMiOjJ9)*

---

## 👩‍💻 Author

**Abdul-Azeez Rofiah Kehinde**
Data Analyst · HNG Internship Stage 4
May 2026

[Let's connect on LinkedIn](https://www.linkedin.com/in/rofiah-abdul-azeez/)
[My Portfolio](https://img.shields.io/badge/GitHub-Follow-black)](https://rofiahportfolio.mystrikingly.com/)

---

## 📚 References

1. Effrosynidis, D., Sylaios, G., & Arampatzis, A. (2022). *The Climate Change Twitter Dataset*. Mendeley Data, V2. https://doi.org/10.17632/mw8yd7z9wc.2
2. Centre for Research on the Epidemiology of Disasters (CRED). *EM-DAT: The International Disaster Database*. https://www.emdat.be/
3. PostgreSQL Global Development Group (2024). *PostgreSQL 17 Documentation*. https://www.postgresql.org/docs/17/
4. Microsoft Corporation (2024). *Power BI Documentation*. https://learn.microsoft.com/en-us/power-bi/

---

*This project was completed as part of the HNG Internship Stage 4 Data Analytics track.*
