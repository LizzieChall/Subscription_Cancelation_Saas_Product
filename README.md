# 🚪 Subscription Cancellation Analysis for SaaS · Product Analytics · Retention

**Tools:** SQL · Snowflake · Hex (Data Science Notebook) · Data Visualization  
**Skills:** CTEs · CASE Statements · UNION · Window Functions · Aggregate Functions · View Creation · Data Wrangling · EDA  
**Domain:** SaaS · Product Analytics · Retention · Customer Success  

🔗 [View the full Hex Notebook](https://app.hex.tech/big-sql-energy/hex/Subscription-Cancelation-Analysis-0334bQyCM2OSUyjfIV8IiS/draft/logic)

---

## 📋 Executive Summary

Facing a significant retention crisis, company leadership identified that churn had reached unsustainable levels, threatening both revenue and the upcoming board review. This analysis examines user-reported cancellation data from 22 canceled subscriptions to identify the most common reasons customers leave, evaluate the reliability of the data collection workflow, and deliver actionable recommendations to reduce churn. Key findings reveal that **Not Useful** (36.4%) and **Expensive** (31.8%) are the top primary cancellation drivers, **Went to a Competitor** surges as the #1 secondary reason, and **Bad Customer Service** dramatically spikes as the dominant third reason (62.5%) — signaling that a meaningful portion of customers who dig deeper into the cancellation flow are citing a support experience failure on top of their initial exit reason.

---

## 🧩 Business Problem

Company leadership escalated a company-wide retention concern after observing churn rates significantly higher than expected for the year. Customers were canceling their subscriptions rather than renewing, creating a direct negative impact on revenue and raising red flags ahead of an upcoming board meeting.

To investigate, I aligned with the product manager to understand exactly how the cancellation workflow is structured. When a user decides to cancel, they navigate to their subscription settings, click cancel, and enter the cancellation flow. The flow requires them to select **one primary reason** for canceling, followed by **two optional additional reason selections**. All three selections draw from the same set of preset categorical dropdown values — making this data significantly more structured and analysis-ready than free-text inputs would be.

Before diving into trends, a key data quality question had to be answered first: **are users actually engaging thoughtfully with the cancellation workflow, or are they clicking through as quickly as possible?** If users are rushing through and selecting random options, the data's reliability as a signal is compromised. This shaped the first phase of the analysis.

**Objectives:**
- Determine the completeness of the cancellation data by exploring how many users select a first, second, and third reason for canceling
- Identify trends in cancellation reasons across all three reason selections, and evaluate whether patterns shift between them
- Translate findings into product and retention recommendations for the CEO and leadership team

---

## 🗂️ Data Model

The analysis was built on the SaaS subscription data model with a focus on the `Cancelations` table:

| Table | Type | Description |
|---|---|---|
| `Subscriptions` | Fact | Core subscription records including revenue, order date, and subscription status |
| `Cancelations` | Fact | Cancellation records with `cancelation_reason1` (required), `cancelation_reason2`, and `cancelation_reason3` (optional), plus `cancel_date`, `contract_id`, `customer_id`, `product_id`, `purchased_users`, and `subscription_id` |
| `Customers` | Dimension | Customer profile information |
| `Products` | Dimension | Product catalog data |
| `Payment_Status_Log` | Fact | Event-level log of every payment status movement per subscription |
| `Payment_Status_Definitions` | Dimension | Maps numeric status IDs to descriptions |

> `cancelation_reason1` is required; `cancelation_reason2` and `cancelation_reason3` are optional. All three pull from the same preset dropdown values: **Expensive**, **Not useful**, **Bad customer service**, and **Went to a competitor**.

---

## 🔀 Cancellation Workflow Architecture

```
User navigates to Subscription Settings
            │
            └─► Clicks "Cancel Subscription"
                        │
                        └─► Cancellation Flow Begins
                                    │
                                    ├─► ✅ Reason 1 Selected (Required)
                                    │     [Expensive | Not useful | Bad customer service | Went to a competitor]
                                    │
                                    ├─► Reason 2 Selected (Optional)
                                    │     [Same preset dropdown values]
                                    │
                                    └─► Reason 3 Selected (Optional)
                                              [Same preset dropdown values]
```

*Because all three fields use the same categorical values, the data is highly structured and aggregatable across positions — but the optional nature of fields 2 and 3 creates a completeness question that must be addressed before drawing conclusions.*

---

## 🔧 Methodology

### 1. Exploratory Data Analysis (EDA)

Profiled the full `Cancelations` table (22 rows) to understand its structure, the range of values present in each reason column, and confirm that all three reason fields draw from the same preset value set.

```sql
SELECT *
FROM public.cancelations
```

### 2. Reason Distribution by Field

Analyzed the frequency of each cancellation reason independently across all three fields to understand what users are reporting at each stage of the workflow.

```sql
-- Reason 1 distribution
SELECT
    cancelation_reason1,
    COUNT(*) AS num_instances,
    COUNT(DISTINCT subscription_id) AS num_subs
FROM public.cancelations
GROUP BY 1

-- Repeated identically for cancelation_reason2 and cancelation_reason3
```

### 3. Data Completeness Analysis — Binary Flags

Before drawing trend conclusions, I measured user engagement with the workflow by flagging whether each subscription selected each reason, then summing to get a total reasons count per user.

```sql
SELECT
    subscription_id,
    CASE WHEN cancelation_reason1 IS NOT NULL THEN 1 ELSE 0 END AS has_reason1,
    CASE WHEN cancelation_reason2 IS NOT NULL THEN 1 ELSE 0 END AS has_reason2,
    CASE WHEN cancelation_reason3 IS NOT NULL THEN 1 ELSE 0 END AS has_reason3,
    has_reason1 + has_reason2 + has_reason3 AS total_reasons,
    has_reason2 + has_reason3 AS additional_reasons
FROM public.cancelations
```

### 4. Average Completeness per Subscription

Wrapped the binary flag logic in a CTE to calculate the average number of total reasons and average number of additional (optional) reasons selected per canceling subscription.

```sql
WITH cancels AS (
    SELECT
        subscription_id,
        CASE WHEN cancelation_reason1 IS NOT NULL THEN 1 ELSE 0 END AS has_reason1,
        CASE WHEN cancelation_reason2 IS NOT NULL THEN 1 ELSE 0 END AS has_reason2,
        CASE WHEN cancelation_reason3 IS NOT NULL THEN 1 ELSE 0 END AS has_reason3,
        has_reason1 + has_reason2 + has_reason3 AS total_reasons,
        has_reason2 + has_reason3 AS additional_reasons
    FROM public.cancelations
)
SELECT
    AVG(total_reasons) AS avg_total_per_sub,
    AVG(additional_reasons) AS avg_additional_per_sub
FROM cancels
```

### 5. Unified Reason View Using UNION

To analyze cancellation reason trends across all three fields simultaneously, I stacked all three columns into a single `cancelation_reason` column using `UNION`. This deduplicates at the subscription level — if a user selected the same reason in multiple fields, they are only counted once per reason.

```sql
SELECT subscription_id, cancelation_reason1 AS cancelation_reason
FROM public.cancelations

UNION

SELECT subscription_id, cancelation_reason2
FROM public.cancelations

UNION

SELECT subscription_id, cancelation_reason3
FROM public.cancelations
```

> **UNION vs. UNION ALL:** Using `UNION` deduplicates at the subscription level — if the same subscription selected "Expensive" in both Reason 1 and Reason 2, they are counted once. This is the right approach for "how many distinct users reported each reason" rather than inflating counts.

### 6. View Creation

Created a saved view in Snowflake to make the unified reason dataset reusable without re-running the UNION query each time. The view adds `cancel_date` and a `reason_number` column (1, 2, or 3) to track which position each reason came from.

```sql
CREATE OR REPLACE VIEW junk.all_cancelation_reasons_liz AS

SELECT subscription_id, cancel_date, cancelation_reason1 AS cancelation_reason, 1 AS reason_number
FROM public.cancelations

UNION

SELECT subscription_id, cancel_date, cancelation_reason2, 2
FROM public.cancelations

UNION

SELECT subscription_id, cancel_date, cancelation_reason3, 3
FROM public.cancelations
```

### 7. Cross-Position Trend Analysis

Queried the view to compare cancellation reason frequency across all three positions, enabling direct comparison of whether users report the same or different reasons depending on how far into the workflow they go.

```sql
SELECT
    cancelation_reason,
    reason_number,
    COUNT(subscription_id) AS num_subs
FROM junk.all_cancelation_reasons_liz
GROUP BY 1, 2
```

### 8. Annual Trend Analysis with Window Functions

Calculated the percentage share of each cancellation reason per year using a window function, enabling year-over-year trend analysis to see whether the composition of cancellation reasons is shifting over time.

```sql
WITH yearly AS (
    SELECT
        DATE_TRUNC('year', cancel_date::date) AS cancel_year,
        cancelation_reason,
        COUNT(*) AS num_reason
    FROM junk.all_cancelation_reasons_liz
    GROUP BY 1, 2
    ORDER BY cancel_year
)
SELECT
    cancel_year,
    cancelation_reason,
    num_reason,
    SUM(num_reason) OVER (PARTITION BY cancel_year) AS year_total,
    ROUND(num_reason * 100 / year_total, 2) AS percent_reason_annual
FROM yearly
```

### 9. Data Visualization

Built the following charts in Hex to communicate findings clearly:

- **Bar chart** — Cancellation reason frequency for Reason 1 (primary, required)
- **Bar chart** — Cancellation reason frequency for Reason 2 (first optional)
- **Bar chart** — Cancellation reason frequency for Reason 3 (second optional)
- **Faceted bar chart** — Number of subscriptions per cancellation reason, broken out by reason position (1, 2, 3) side by side
- **Line chart** — Annual percentage share of each cancellation reason over time, color-coded by reason

---

## 📊 Results

### Cancellation Reason 1 — Primary (Required)

*(Add Reason 1 bar chart here)*

### Cancellation Reason 2 — First Optional

*(Add Reason 2 bar chart here)*

### Cancellation Reason 3 — Second Optional

*(Add Reason 3 bar chart here)*

### Number of Subscriptions per Cancellation Reason, by Position

*(Add faceted bar chart here)*

### Annual Cancellation Reason Trend

*(Add line chart here)*

---

### Data Completeness

| Selection | # of Subscriptions | % of Total Cancellations |
|---|---|---|
| Reason 1 — Required | 22 | 100% |
| Reason 2 — Optional | 18 | 81.8% |
| Reason 3 — Optional | 8 | 36.4% |
| **Avg. total reasons per canceling user** | **2.18** | — |
| **Avg. additional (optional) reasons per user** | **1.18** | — |

> The majority of users (81.8%) did engage with at least one optional field, which is a positive signal — this data is more reliable than if users had dropped off immediately. However, only 36.4% selected a third reason, and the average user selected just over 2 reasons total, suggesting there is still meaningful completeness drop-off and the optional fields should be interpreted with slightly more caution than the required first reason.

### Cancellation Reason Distribution — By Position

| Cancellation Reason | Reason 1 (of 22) | Reason 2 (of 18 who selected) | Reason 3 (of 8 who selected) |
|---|---|---|---|
| Not useful | 8 — 36.4% | 3 — 16.7% | 2 — 25.0% |
| Expensive | 7 — 31.8% | 6 — 33.3% | 1 — 12.5% |
| Went to a competitor | 6 — 27.3% | 7 — 38.9% | 0 — 0% |
| Bad customer service | 1 — 4.5% | 2 — 11.1% | 5 — 62.5% |

### Overall Cancellation Reason Frequency (Across All Positions, Distinct Subscriptions)

| Cancellation Reason | # of Subscriptions |
|---|---|
| Expensive | 13 |
| Went to a competitor | 13 |
| Not useful | 10 |
| Bad customer service | 8 |

### Annual Cancellation Reason Trend (% of All Reason Selections That Year)

| Year | Expensive | Not useful | Went to a competitor | Bad customer service |
|---|---|---|---|---|
| 2022 | — | 33.33% | 33.33% | — |
| 2023 | 16.67% | 16.67% | 16.67% | 8.33% |
| 2024 | 23.53% | 19.61% | 19.61% | 13.73% |

> *Note: Annual percentages include null (no selection) entries in the denominator from optional reason fields. 2022 data reflects a very small sample (1 cancellation) and should not be used to draw trend conclusions.*

### Key Findings

- **81.8% of canceling users selected at least one optional reason beyond the required first**, and the average user selected 2.18 total reasons. This is a positive signal for data reliability — most users did engage with the workflow rather than rushing through it. However, completion drops significantly for Reason 3 (only 36.4%), so optional field data should be treated as directional.
- **"Not useful" was the most common primary cancellation reason (36.4%), followed closely by "Expensive" (31.8%).** These two reasons together account for 68.2% of all first-reason selections, making them the clearest and most reliable signal in the dataset.
- **"Went to a competitor" shifts dramatically across positions** — it's the #3 primary reason (27.3%) but becomes the #1 secondary reason (38.9%), then drops to zero for the third reason. This suggests customers cite competitive alternatives as a contributing factor but rarely as their singular, top-of-mind reason for leaving.
- **"Bad customer service" spikes dramatically as a third reason (62.5%)**, despite being the least common primary reason (4.5%). Users who went deeper into the cancellation flow and selected a third reason were overwhelmingly citing a service experience failure alongside their initial exit reason — a signal that cannot be found by looking at first-reason data alone.
- **Expensive and Went to a Competitor are the most commonly cited reasons overall** across all three positions combined, each appearing across 13 distinct subscriptions, followed by Not useful (10) and Bad customer service (8).
- **Cancellation volume is heavily concentrated in 2024**, with 51 total reason entries that year compared to 12 in 2023 and 3 in 2022 — reinforcing the urgency of the churn problem leadership has flagged.

---

## 💡 Business Recommendations

**1. Invest in onboarding to address "Not Useful" — highest priority**
The most common primary cancellation reason is that users don't find the product useful, which typically signals they never experienced its core value before deciding to leave. Investing in stronger onboarding — in-app guides, proactive check-ins in the first 30–60 days, and feature education at the right moments — can help users reach their "aha moment" before the decision to cancel is made. If users find the product more useful and valuable, they may also become less cost-sensitive over time, addressing the "Expensive" reason indirectly.

**2. Introduce a rescue offer for cost-sensitive users at the start of the cancellation flow**
With "Expensive" as the #2 primary reason and tied for #1 overall, a targeted offer presented at the beginning of the cancellation flow could retain a meaningful share of cost-conscious users before they complete the exit. Options could include a discount, a downgrade to a lower-cost tier, or a pause option for users who aren't ready to commit but also aren't fully ready to leave.

**3. Research and respond to competitive threats**
"Went to a competitor" is the #1 secondary reason and tied for the most commonly cited reason overall across all three positions. A competitive audit identifying where the product falls short on features, pricing, or experience would give the product team actionable input for roadmap prioritization and help stem exits driven by competitive pull.

**4. Investigate and address the "Bad customer service" spike**
Although only 1 user cited bad customer service as their primary reason, it accounts for 62.5% of all third-reason selections. This is not a small footnote — it suggests that a subset of churned users had a poor support experience that compounded their other reasons for leaving. Reviewing support ticket data and customer service response quality for churned subscriptions should be a near-term priority.

**5. Redesign the cancellation flow to capture richer data**
Only 36.4% of users selected a third reason. Redesigning the flow to feel less like a form — surfacing one question at a time, adding a progress indicator, or including an optional free-text comment field — could meaningfully improve data completeness and surface insights that the current structure misses.

---

## 🔭 Next Steps

- **Engagement analysis of churned subscriptions:** Explore how canceled subscribers used the product before leaving. What features did they use or avoid? How frequently were they active? Comparing engagement patterns between churned and retained users could surface early warning signals for at-risk identification.
- **Cohort analysis by subscription age:** Do users who cancel within the first 30 days report different reasons than users who cancel after 6+ months? Understanding the "when" alongside the "why" will sharpen where intervention efforts should be focused.
- **Dig into the "Bad customer service" spike:** Cross-reference the 8 subscriptions that cited bad customer service with support ticket records to understand what type of service failures drove these exits and whether there are fixable systemic issues.
- **Work with the product manager to redesign the cancellation flow:** Incorporate rescue tactics (discount offers, pause options) and explore adding a free-text optional comment field to capture nuance beyond the preset dropdown values.
- **Build a churn prediction model:** With a larger dataset, use engagement signals (login frequency, feature usage, payment history) to predict which active subscribers are most at risk of canceling, enabling proactive retention outreach before the decision is made.

---

## 🛠️ Skills Demonstrated

`SQL` · `CTEs` · `CASE Statements` · `UNION` · `Window Functions (SUM OVER PARTITION BY)` · `DATE_TRUNC` · `Aggregate Functions` · `COUNT DISTINCT` · `Null Handling` · `Binary Flag Logic` · `View Creation` · `Data Completeness Analysis` · `Data Wrangling` · `EDA` · `Funnel Analysis` · `Data Visualization` · `Snowflake` · `Hex Notebook` · `Stakeholder Communication` · `Business Recommendations`
