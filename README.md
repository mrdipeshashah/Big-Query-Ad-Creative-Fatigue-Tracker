# OVERVIEW

This repository contains Big Query code designed to identify and remediate ad creative fatigue across Ad Platform (i.e. Meta Ads and Google Ads) as the first step of a diagnostic framework to spot and remediate ad creative fatigue. 

# DASHBOARD

The data studio dashboard (https://datastudio.google.com/reporting/0783b9bf-5507-470e-a847-aeabfbf4b560) brings many of the insights to life.

## DIAGNOSTIC MODEL EVOLUTION

This repository provides two alternative SQL view architectures to suit different media buying workflows and governance standards:

1. **Format-Tiered Model (`1.0_creative-performance-format-tiered-model.sql`):** The primary production engine. Dynamically classifies creative formats from naming conventions and applies format-aware sensitivity thresholds to eliminate false-positive fatigue alerts across video and search formats.
2. **Generic Model (`2.0_creative-performance-generic-model.sql`):** The baseline architecture. Operates on a single cross-format relative threshold ($\pm 10\%$), ideal for accounts without structured creative naming conventions.

### Architectural Comparison Matrix

| Evaluation Dimension | Generic Model (`2.0`) | Format-Tiered Model (`1.0`) |
| :--- | :--- | :--- |
| **Format Tagging** | Raw string parsing | Dynamic `creative_format` classification (`VIDEO`, `STATIC`, `CAROUSEL`, `TEXT`, `MULTI_ASSET`) |
| **Video Sensitivity** | $\le -10\%$ CTR Drop / $\ge +10\%$ CPC | $\le -15\%$ CTR Drop / $\ge +10\%$ CPC *(Noise Tolerant)*[cite: 1] |
| **Static / Carousel** | $\le -10\%$ CTR Drop / $\ge +10\%$ CPC | $\le -10\%$ CTR Drop / $\ge +10\%$ CPC *(Direct Response)*[cite: 1] |
| **Text / Multi-Asset** | $\le -10\%$ CTR Drop / $\ge +10\%$ CPC | $\le -5\%$ CTR Drop / $\ge +15\%$ CPC *(Auction Pressure Focus)* |
| **Primary Use Case** | Unstructured creative naming | Standardized naming taxonomies (`meta_vid_...`, `goog_txt_...`) |

## SCHEMA 

The required schema to understand ad creative fatigue 

| Field Name | Data Type | Mode | Description | Example Value |
| :--- | :--- | :--- | :--- | :--- |
| **date** | `DATE` | `NULLABLE` | Performance record date (`YYYY-MM-DD`). | `2026-10-12` |
| **platform** | `STRING` | `NULLABLE` | Ad platform identifier (`meta`, `google`). | `meta` |
| **campaign_name** | `STRING` | `NULLABLE` | Campaign name string. | `Prospecting_Conversions_UK` |
| **creative_name** | `STRING` | `NULLABLE` | Human-readable creative name or concept code. | `Static_Hook01_AngleA` |
| **impressions** | `INTEGER` | `NULLABLE` | Total daily ad impressions delivered. | `14500` |
| **clicks** | `INTEGER` | `NULLABLE` | Total daily link clicks generated. | `185` |
| **cost** | `FLOAT` | `NULLABLE` | Total daily ad spend (account currency). | `124.50` |

## THE PROBLEM & BUSINESS VALUE

### The Mechanics of Creative Fatigue
Digital marketing campaigns on performance platforms like Meta Ads and Google Ads inevitably suffer from ad creative fatigue. As target audiences are repeatedly exposed to the same visual hook or message, a predictable decay chain occurs:
1. **Audience Saturation & CTR Decay:** Viewer responsiveness drops, causing the Click-Through Rate (CTR) to decline over time.
2. **Algorithmic Penalties & Auction Pressure:** Modern auction engines rely heavily on ad relevance scores. As engagement (CTR) drops, algorithms penalize the creative by increasing Cost Per Mille (CPM) to protect platform yield.
3. **Compounded Cost Spikes (CPC):** The combination of falling CTR and rising CPM creates an exponential spike in Cost Per Click ($\text{CPC} \approx \frac{\text{CPM}}{\text{CTR} \times 10}$), resulting in wasted ad spend and rapidly diminishing ROAS.

### The Automated Diagnostic Solution
This pipeline acts as the **first step of an automated diagnostic framework** by evaluating **relative rate of decay**. By pairing rolling 3-day moving averages with 7-day period-over-period (L7D vs. P7D) trends, the system isolates true fatigue from market noise:
* **Stable Low Performers:** An ad with a flat 0.50% CTR is classified as `Healthy / Keep Running` (stable efficiency, no active fatigue)
* **Active Fatigue Drops:** An ad dropping from 2.5% to 1.2% CTR with a rising CPC triggers an immediate `Fatigue: Refresh Creative` alert—giving media buyers an actionable task list to swap assets *before* overall campaign ROAS collapses

### Solving the Baseline v Format & Decay Dilemma
A primary challenge in performance marketing is distinguishing between format-level baseline differences (e.g., a video ad launching with a normal 0.20% link CTR vs. a static image at 1.50%) and **true creative fatigue** (an ad whose performance actively decays relative to its prior baseline).

Evaluating raw CTR alone leads to premature pausing of high-converting video assets:
* **CTR is the Cause (Audience Perception):** Serves as the first-line diagnostic indicator of whether a visual hook or angle is losing resonance relative to its own baseline trajectory.
* **CPC is the Effect (Financial Efficiency):** Quantifies the real financial impact of auction pressure and declining engagement.

To address this, the pipeline incorporates **automated format classification directly in SQL** (`VIDEO`, `STATIC`, `CAROUSEL`, `TEXT`, `MULTI_ASSET`), combined with **relative rate of decay** tracking:
* **Automated Format Parsing:** Standardized naming conventions (e.g., `meta_vid_`, `goog_txt_`, `meta_car_`) are dynamically parsed in BigQuery using positional indexing and fallback pattern matching to prevent false-positive alerts across format types.
* **Format Baseline Stability:** A video ad holding a flat 0.20% CTR is classified as `Healthy / Keep Running` (stable efficiency, no active decay)[cite: 1].
* **Active Fatigue Drops:** An ad whose CTR drops from 0.20% to 0.14% ($\le -10\%$) while CPC rises ($\ge +10\%$) triggers an immediate `Fatigue: Refresh Creative` alert[cite: 1].

## SMOOTHING v TREND WINDOWS: 3-DAY MOVING AVERAGE v 7-DAY PERRIOD-OVER-PERIOD 

A core engineering decision in this pipeline is the combination of **3-day moving averages** and **7-day period-over-period (L7D vs. P7D)** comparisons. Each window serves a distinct analytical purpose, and using them together eliminates false positives caused by day-of-week seasonality or single-day delivery glitches.

**Data Processing Flow:**
`Raw Daily Data (High Volatility)` ➔ `3-Day Moving Average (Noise Filtering)` ➔ `7-Day Comparison (Directional Trajectory)`

## FORMAT DIANOSTIC MATRIX FRAMEWORK

The diagnostic engine evaluates campaign metrics by tracking daily trends smoothed with a 3-day moving average and comparing performance over the **Last 7 Days (L7D)** relative to the **Prior 7 Days (P7D)**. To prevent false positives across varied ad types, the engine applies format-aware sensitivity thresholds across each format category (`VIDEO`, `STATIC`, `CAROUSEL`, `TEXT`, `MULTI_ASSET`):

| Creative Format | CTR Decay Rule (L7D vs P7D) | CPC Spike Rule (L7D vs P7D) | Automated Status Trigger | Action Required |
| :--- | :---: | :---: | :--- | :--- |
| **`VIDEO`** | **$\le -15\%$** *(higher noise tolerance)* | **$\ge +10\%$** | `Fatigue: Refresh Creative` | Swap video hook / first 3s visual. |
| **`STATIC` / `CAROUSEL`** | **$\le -10\%$** *(standard direct response)* | **$\ge +10\%$** | `Fatigue: Refresh Creative` | Refresh static visual / banner copy. |
| **`TEXT` / `MULTI_ASSET`** | **$\le -5\%$** | **$\ge +15\%$** *(auction cost focus)* | `Fatigue: Refresh Creative` | Refresh headline angles & asset groups. |
| **All Formats** | Flat ($\pm 10\%$) | **$\ge +10\%$** | `Crowded Auction: Check CPM/Insights` | Auction pressure. Check CPMs or Auction Insights. |
| **All Formats** | **$\le -10\%$** | **$\le -10\%$** | `Low Demand: Scale Spend Down` | Market contraction. Scale back budget. |
| **All Formats** | **$\ge +10\%$** | **$\ge +10\%$** | `High Demand: Leave Active` | Winning creative in active auction. Keep running. |
| **All Formats** | Within $\pm 10\%$ | Within $\pm 10\%$ | `Healthy / Keep Running` | Baseline performance. No action required. |

## CALCULATED FIELDS 

To ensure mathematical accuracy across aggregated selections and prevent "average of averages" skewing, the dashboard uses the following custom calculated fields:

| Field Name | Formula | Type | Purpose |
| :--- | :--- | :--- | :--- |
| **Weighted CPM** | `(SUM(cost) / SUM(impressions)) * 1000` | Currency (`£`) | Calculates true weighted CPM across any aggregated selection. |
| **Aggregate CTR** | `SUM(clicks) / SUM(impressions)` | Percent (`%`) | Ensures mathematically accurate CTR totals without average skewing. |
| **Aggregate CPC** | `SUM(cost) / SUM(clicks)` | Currency (`£`) | Calculates true blended cost-per-click for aggregated views. |
| **Low Volume Flag** | `CASE WHEN SUM(impressions) < 1000 THEN "Low Volume (<1k Imps)" ELSE "Sufficient Volume" END` | Text | Identifies low-volume statistical noise versus true fatigue. |


