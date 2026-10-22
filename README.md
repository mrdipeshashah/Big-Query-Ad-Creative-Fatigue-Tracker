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

### DIAGNOSTIC MODEL & WALKTHROUGH EXAMMPLE

Evaluating every creative through a **three-stage process**.

#### Stage 1: "Is this ad mature enough to judge?" (Volume Guardrail)
* **The Logic:** A brand-new ad that ran for two days might show a sudden drop in CTR or a spike in CPC simply because only 50 people saw it. Judging an ad on small sample sizes leads to premature, wasteful creative swaps.
* **The Engine's Decision:** Before checking any performance drops, the engine counts the ad's recent exposure. If it has logged fewer than 1,000 impressions in the last 7 days, the engine immediately labels it **`Low Volume (<1k Imps)`** and stops—protecting learning-phase ads from false-positive fatigue alerts[cite: 1].

---

#### Stage 2: "Is performance degrading relative to its own baseline?" (Normalized % Shifts)
* **The Logic:** Every ad format and audience target has a different baseline. A Search text ad might naturally average a 4.0% CTR, whereas a Meta Video ad might naturally average 0.4% CTR. Using fixed benchmarks (e.g., "alert if CTR drops below 1.0%") fails because it compares apples to oranges.
* **The Engine's Decision:** The model compares the ad's **Last 7 Days (L7D)** directly against its own **Prior 7 Days (P7D)** baseline[cite: 1]. It asks: *"Compared to how this exact ad performed last week, by what percentage has its engagement dropped and its cost inflated?"*[cite: 1]

---

#### Stage 3: "Does this decay match how this specific format behaves?" (Format Rules)
* **The Logic:** Different ad formats suffer from different types of audience friction:
  * **Video Ads:** Users watch video for entertainment; link click-through rates naturally bounce around day-to-day. A small 10% drop in CTR is normal platform noise[cite: 1].
  * **Static Banner Ads:** Static images are purely direct-response. If users stop clicking a static image, it means they've seen it too many times and are actively scrolling past it[cite: 1].
  * **Text / Performance Max Ads:** Text search ads rarely suffer visual fatigue, but they get hit by cost explosions when competitors outbid them in auctions[cite: 1].

* **The Engine's Decision:**
  * **For Video:** It requires a steep drop in CTR ($\ge 15\%$) **and** an increase in CPC ($\ge 10\%$) before flagging fatigue—filtering out normal video engagement noise[cite: 1].
  * **For Static / Carousel:** It applies a tighter tolerance ($\ge 10\%$ drop in CTR **and** $\ge 10\%$ CPC spike) because static decay happens faster and requires quicker visual refreshes[cite: 1].
  * **For Text / PMax:** It primary monitors for sudden auction cost explosions ($\ge 15\%$ CPC spike)[cite: 1].

---

### Real-World Example: Seeing the Logic in Action

Imagine two ads running in the same campaign:

1. **Ad A (New Prospecting Video):**
   * *Last Week:* 0.50% CTR | $2.00 CPC
   * *This Week:* 0.45% CTR (10% drop) | $2.20 CPC (10% spike)
   * **Model Reasoning:** "The cost went up, but a 10% drop in video CTR is normal viewing fluctuation. Don't waste time making a new video yet."
   * **Status:** `Healthy / Keep Running`[cite: 1]

2. **Ad B (Static Discount Banner):**
   * *Last Week:* 2.00% CTR | $1.00 CPC
   * *This Week:* 1.60% CTR (20% drop) | $1.20 CPC (20% spike)
   * **Model Reasoning:** "This static ad lost a fifth of its clickability and got 20% more expensive in just one week. The audience is blind to this image."
   * **Status:** **`Fatigue: Refresh Creative`**[cite: 1]



...................

### CONCRETE CALCULATION & WALKTHROUGH EXAMMPLE

To see how the BigQuery diagnostic engine evaluates an asset step-by-step, consider an ad creative with the following baseline and recent performance profile:

#### Ingested Data Metrics
* **Baseline Window (Prior 7 Days / P7D):**
  * Baseline CTR (`ctr_p7d`): `2.00%` (0.0200)
  * Baseline CPC (`cpc_p7d`): `$1.00`
* **Evaluation Window (Last 7 Days / L7D):**
  * L7D Impression Volume (`impressions_l7d`): `2,500`
  * L7D CTR (`ctr_l7d`): `1.60%` (0.0160)
  * L7D CPC (`cpc_l7d`): `$1.20`

#### Execution Sequence

##### Step 1: Statistical Significance Guardrail
The SQL `CASE` statement first evaluates Rule #0:
$$\text{L7D Impressions} = 2,500 \ge 1,000$$
* **Verdict:** Passes the statistical significance guardrail ($2,500 \ge 1,000$). The ad is not flagged as `Low Volume (<1k Imps)` and proceeds to decay evaluation.

##### Step 2: Period-over-Period Relative Shift Calculations
The engine uses `SAFE_DIVIDE` to calculate normalized percentage shifts between windows:

* **Relative CTR Change (`ctr_7d_change_pct`):**
  $$\frac{\text{CTR}_{L7D} - \text{CTR}_{P7D}}{\text{CTR}_{P7D}} = \frac{1.60\% - 2.00\%}{2.00\%} = \frac{-0.40\%}{2.00\%} = \mathbf{-20.0\%} \quad \text{(Relative CTR Decay)}$$

* **Relative CPC Change (`cpc_7d_change_pct`):**
  $$\frac{\text{CPC}_{L7D} - \text{CPC}_{P7D}}{\text{CPC}_{P7D}} = \frac{\$1.20 - \$1.00}{\$1.00} = \frac{+\$0.20}{\$1.00} = \mathbf{+20.0\%} \quad \text{(Relative Cost Inflation)}$$

##### Step 3: Format-Specific Diagnostic Evaluation

* **Case A: Evaluated as `VIDEO`**
  * **Rule Trigger:** Requires CTR Drop $\le -15\%$ AND CPC Spike $\ge +10\%$.
  * **Evaluation:**
    * $\text{CTR Shift } (-20.0\%) \le -15\%$ $\rightarrow$ **PASS**
    * $\text{CPC Shift } (+20.0\%) \ge +10\%$ $\rightarrow$ **PASS**
  * **Automated Status:** **`Fatigue: Refresh Creative`**

* **Case B: Evaluated as `STATIC` or `CAROUSEL`**
  * **Rule Trigger:** Requires CTR Drop $\le -10\%$ AND CPC Spike $\ge +10\%$
  * **Evaluation:** Both conditions met easily ($-20.0\%$ drop exceeds the $-10\%$ threshold)
  * **Automated Status:** **`Fatigue: Refresh Creative`**

* **Case C: Sensitivity Comparison (What if CTR only dropped to 1.80% / -10% drop?)**
  * **If `STATIC`:** Triggered! ($-10.0\%$ drop meets static threshold)
  * **If `VIDEO`:** **Ignored!** (Video rule suppresses alert to account for link-click volatility; status defaults to `Healthy / Keep Running`)

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
Campaigns on platforms like Meta Ads and Google Ads inevitably suffer from ad creative fatigue. As target audiences are repeatedly exposed to the same visual hook or message, a predictable decay chain occurs:
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


