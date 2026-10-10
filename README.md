# 🎯 Ad Creative Fatigue Tracker

An end-to-end data pipeline and automated diagnostic engine designed to identify ad creative fatigue across Meta Ads and Google Ads before campaign performance degrades.

---

## 📌 Problem & Business Value

Digital marketing campaigns naturally experience creative fatigue when audiences see the same ad creatives too many times. Continuing to run fatigued creatives results in wasted ad spend, declining Click-Through Rates (CTR), and escalating Costs Per Click (CPC).

This repository automates the identification of creative fatigue by processing raw daily ad performance metrics into a BigQuery analytics view and Looker Studio dashboard, serving a daily operational action list for media buyers.

---

## 🧠 Diagnostic Matrix & Framework

The diagnostic engine evaluates campaign metrics by tracking daily trends smoothed with a 3-day moving average and comparing performance over the **Last 7 Days (L7D)** relative to the **Prior 7 Days (P7D)**:

| CTR Trend (L7D vs P7D) | CPC Trend (L7D vs P7D) | Diagnosis | Automated Status | Action Required |
| :---: | :---: | :--- | :--- | :--- |
| **Down** ($\le -10\%$)[cite: 1] | **Up** ($\ge +10\%$)[cite: 1] | **Creative Fatigue**[cite: 1] | `Fatigue: Refresh Creative`[cite: 1] | Swap hook/visual for same audience[cite: 1]. Do **NOT** rebuild campaign[cite: 1]. |
| **Flat** ($-10\%$ to $+10\%$)[cite: 1] | **Up** ($\ge +10\%$)[cite: 1] | **Crowded Auction**[cite: 1] | `Crowded Auction: Check CPM/Insights`[cite: 1] | Auction pressure[cite: 1]. Meta: check CPM[cite: 1]. Google: check Auction Insights[cite: 1]. |
| **Down** ($\le -10\%$)[cite: 1] | **Down** ($\le -10\%$)[cite: 1] | **Low Demand**[cite: 1] | `Low Demand: Scale Spend Down`[cite: 1] | Broad market decline[cite: 1]. Scale spend back until demand returns[cite: 1]. |
| **Up** ($\ge +10\%$)[cite: 1] | **Up** ($\ge +10\%$)[cite: 1] | **High Demand / Winning Ad**[cite: 1] | `High Demand: Leave Active`[cite: 1] | Ad is winning in competitive auction[cite: 1]. Keep active[cite: 1]. |
| Variance within $\pm 10\%$[cite: 1] | Variance within $\pm 10\%$[cite: 1] | **Baseline / Normal**[cite: 1] | `Healthy / Keep Running`[cite: 1] | Standard fluctuations[cite: 1]. No action required[cite: 1]. |

---

## 🏗️ Data Architecture

```text
┌──────────────────────────────┐
│  Ad Platforms (Meta/Google)  │
└──────────────┬───────────────┘
               │ Daily Ingestion (API / Fivetran)
               ▼
┌──────────────────────────────┐
│  BigQuery: Raw Table         │  raw_ad_metrics
│  (date, impressions, clicks) │  (Additive base metrics)
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
│  Looker Studio Dashboard     │  3-Chart Decision Center
└──────────────────────────────┘
