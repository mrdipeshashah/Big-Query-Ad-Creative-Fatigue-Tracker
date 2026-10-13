# 🎯 Ad Creative Fatigue Tracker

An end-to-end data pipeline, BigQuery diagnostic SQL view, and multi-page Looker Studio decision system designed to identify and remediate ad creative fatigue across Meta Ads and Google Ads before campaign performance degrades.

## 🗄️ Raw Input Data Schema

To feed the BigQuery diagnostic view (`adplatform_creative_performance`), the raw underlying table (`raw_ad_metrics`) uses the following top-of-funnel schema:

| Field Name | Data Type | Mode | Description | Example Value |
| :--- | :--- | :--- | :--- | :--- |
| **date** | `DATE` | `NULLABLE` | Performance record date (`YYYY-MM-DD`). | `2026-10-12` |
| **platform** | `STRING` | `NULLABLE` | Ad platform identifier (`meta`, `google`). | `meta` |
| **campaign_name** | `STRING` | `NULLABLE` | Campaign name string. | `Prospecting_Conversions_UK` |
| **creative_name** | `STRING` | `NULLABLE` | Human-readable creative name or concept code. | `Static_Hook01_AngleA` |
| **impressions** | `INTEGER` | `NULLABLE` | Total daily ad impressions delivered. | `14500` |
| **clicks** | `INTEGER` | `NULLABLE` | Total daily link clicks generated. | `185` |
| **cost** | `FLOAT` | `NULLABLE` | Total daily ad spend (account currency). | `124.50` |

## 📌 Problem & Business Value

### The Mechanics of Creative Fatigue
Digital marketing campaigns on performance platforms like Meta Ads and Google Ads inevitably suffer from ad creative fatigue. As an audience is exposed to the same visual hook or message repeatedly:
1. **Audience Saturation & CTR Decay:** Viewer responsiveness drops, causing the Click-Through Rate (CTR) to decline over time.
2. **Algorithmic Penalties & Auction Pressure:** Modern ad platform auction engines rely on ad relevance scores. As engagement (CTR) drops, algorithms penalize the creative by increasing Cost Per Mille (CPM) to maintain platform yield.
3. **Compounded Cost Spikes (CPC):** The combination of falling CTR and rising CPM creates an exponential spike in Cost Per Click (CPC) (CPC ≈ CPM / (CTR * 10)), resulting in wasted ad spend and diminished ROAS.

### Solving the Baseline vs. Decay Dilemma
A common challenge in performance marketing is distinguishing between a **weak initial ad** (e.g., an ad launching with a low baseline CTR of 0.50% that remains flat) and **true creative fatigue** (an ad whose CTR actively decays relative to its prior performance). 

Evaluating raw CTR alone is misleading:
* **CTR is the Cause (Audience Perception):** Indicates whether the visual/hook is losing resonance over time.
* **CPC is the Effect (Financial Efficiency):** Measures the financial impact of changing auction costs and engagement rates.

By pairing rolling 3-day moving averages with period-over-period (Last 7 Days vs. Prior 7 Days) relative percentage changes, this pipeline evaluates **rate of decay** rather than static global benchmarks. An ad with a flat 0.50% CTR is classified as `Healthy / Keep Running` (since performance is stable), while an ad dropping from 2.5% to 1.2% with a rising CPC is flagged as `Fatigue: Refresh Creative`.

---

## ⏱️ Smoothing vs. Trend Windows: 3-Day Moving Averages vs. 7-Day Period-over-Period

A core engineering decision in this pipeline is the combination of **3-day moving averages** and **7-day period-over-period (L7D vs. P7D)** comparisons. Each window serves a distinct analytical purpose, and using them together eliminates false positives caused by day-of-week seasonality or single-day delivery glitches.

**Data Processing Flow:**
`Raw Daily Data (High Volatility)` ➔ `3-Day Moving Average (Noise Filtering)` ➔ `7-Day Comparison (Directional Trajectory)`

---

## 🧠 Diagnostic Matrix & Framework

The diagnostic engine evaluates campaign metrics by tracking daily trends smoothed with a 3-day moving average and comparing performance over the **Last 7 Days (L7D)** relative to the **Prior 7 Days (P7D)**:

| CTR Trend (L7D vs P7D) | CPC Trend (L7D vs P7D) | Diagnosis | Automated Status | Action Required |
| :---: | :---: | :--- | :--- | :--- |
| **Down** ($\le -10\%$) | **Up** ($\ge +10\%$) | **Creative Fatigue** | `Fatigue: Refresh Creative` | Swap hook/visual for same audience. Do **NOT** rebuild campaign. |
| **Flat** ($-10\%$ to $+10\%$) | **Up** ($\ge +10\%$) | **Crowded Auction** | `Crowded Auction: Check CPM/Insights` | Auction pressure. Meta: check CPM. Google: check Auction Insights. |
| **Down** ($\le -10\%$) | **Down** ($\le -10\%$) | **Low Demand** | `Low Demand: Scale Spend Down` | Broad market decline. Scale spend back until demand returns. |
| **Up** ($\ge +10\%$) | **Up** ($\ge +10\%$) | **High Demand / Winning Ad** | `High Demand: Leave Active` | Ad is winning in competitive auction. Keep active. |
| Variance within $\pm 10\%$ | Variance within $\pm 10\%$ | **Baseline / Normal** | `Healthy / Keep Running` | Standard fluctuations. No action required. |

---

## 📊 Dashboard Architecture & Performance Measurement

The Looker Studio dashboard is structured into two complementary operational pages:

```text
┌────────────────────────────────────────────────────────────────────────────────────────┐
│               PAGE 1: EXECUTIVE ALERT & CREATIVE FATIGUE ACTION SYSTEM                  │
│  • High-Level Health Scorecards (Total Spend, Impressions, Clicks, Avg CTR, Fatigued)  │
│  • Operational Action Table (Sorted by CPC Spike Descending w/ Conditional Highlights)  │
│  • 30-Day Trajectory Line Chart (3-Day Moving Average Crossover: CTR vs CPC)           │
│  • Diagnostic Quadrant Scatter Plot (L7D CTR Change % vs L7D CPC Change %)             │
└───────────────────────────────────────────┬────────────────────────────────────────────┘
                                            │ Drill-down via filter / cross-navigation
                                            ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                   PAGE 2: CREATIVE VOLUME & AUCTION PRESSURE DIAGNOSTICS                │
│  • Diagnostic Scorecards (Total Spend, Total Impressions, Average Weighted CPM)        │
│  • Delivery Volume vs. CTR Combo Chart (Daily Impressions Bars + CTR Trajectory Line)  │
│  • Auction Pressure Trend (Dual-Axis CPM vs. CPC Line Chart)                           │
│  • Forensic Daily Breakdown Table (Granular Day-by-Day Heatmap Table)                  │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

### 1. Page 1: Executive Alert & Creative Performance (Core Engine)
Page 1 serves as the primary decision engine for media buyers and marketing leads. Positioned directly below the top scorecards, the **Operational Action Table** functions as a daily prioritized task list:
* **Immediate Priority Ranking:** The action table is sorted by `CPC L7D Change` descending, bringing assets with the largest cost explosions (+70% to +85% CPC spikes) straight to rows 1–4.
* **Conditional Visual Warnings:** Rows categorized as `Fatigue: Refresh Creative` are highlighted in red fill, while `Crowded Auction` scenarios are highlighted in yellow.
* **Cross-Filtering Workflow:** Selecting a flagged asset in the top action table instantly filters the 30-day trajectory chart and diagnostic scatter matrix below, allowing media buyers to pinpoint exact CTR/CPC crossover decay dates.

### 2. Page 2: Creative Volume & Auction Diagnostics (Forensic View)
Page 2 provides forensic diagnostic depth to separate true creative fatigue from statistical noise or external market dynamics:
* **Volume Validation (Delivery vs. CTR Combo Chart):** Overlays daily impression bars against smoothed CTR lines to prove whether CTR drops occurred under high impression volume (True Fatigue) or micro-volume budget cuts (Statistical Noise).
* **Algorithmic Penalty vs. Competition (CPM vs. CPC Trend):** Tracks weighted CPM ($\frac{\text{cost}}{\text{impressions}} \times 1000$) alongside CPC trends to determine if cost spikes stem from platform relevance score penalties (CPM surge + CTR drop) or broader competitive bidding wars.
* **Forensic Breakdown Table:** Displays raw daily figures with heatmaps across spend, impressions, clicks, aggregate CPM, CPC, and CTR.

---

## 🏗️ Data Architecture & Pipeline

```text
┌──────────────────────────────┐
│  Ad Platforms (Meta/Google)  │
└──────────────┬───────────────┘
               │ Daily Ingestion (API / Fivetran)
               ▼
┌──────────────────────────────┐
│  BigQuery: Raw Table         │  raw_ad_metrics
│  (date, impressions, clicks) │  (Additive top-of-funnel base metrics)
└──────────────┬───────────────┘
               │ CTE Window Functions & POP Logic
               ▼
┌──────────────────────────────┐
│  BigQuery: Analytics View    │  adplatform_creative_performance
│  (3-day averages & Status)   │  (Diagnostic engine layer)
└──────────────┬───────────────┘
               │ Direct Query / Live Connection
               ▼
┌──────────────────────────────┐
│  Looker Studio Dashboard     │  Multi-Page Decision System
│  (Page 1: Alert & Action)    │  (Page 1: Action Table, Trajectory, Scatter Plot)
│  (Page 2: Volume & Auction)  │  (Page 2: Combo Chart, CPM/CPC Trend, Heatmap Table)
└──────────────┬───────────────┘
```

---

## 📐 Looker Studio Calculated Fields

To ensure mathematical accuracy across aggregated selections and prevent "average of averages" skewing, the dashboard uses the following custom calculated fields:

| Field Name | Formula | Type | Purpose |
| :--- | :--- | :--- | :--- |
| **Weighted CPM** | `(SUM(cost) / SUM(impressions)) * 1000` | Currency (`£`) | Calculates true weighted CPM across any aggregated selection. |
| **Aggregate CTR** | `SUM(clicks) / SUM(impressions)` | Percent (`%`) | Ensures mathematically accurate CTR totals without average skewing. |
| **Aggregate CPC** | `SUM(cost) / SUM(clicks)` | Currency (`£`) | Calculates true blended cost-per-click for aggregated views. |
| **Low Volume Flag** | `CASE WHEN SUM(impressions) < 1000 THEN "Low Volume (<1k Imps)" ELSE "Sufficient Volume" END` | Text | Identifies low-volume statistical noise versus true fatigue. |

---

## 💻 SQL Source Code

The primary BigQuery SQL view definition is maintained in [`adplatform_creative_performance.sql`](./adplatform_creative_performance.sql). It dynamically calculates rolling 3-day window metrics, L7D vs. P7D period comparisons, and automated status tags directly at the query layer.
