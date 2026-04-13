# Google Merchandise Store — E-commerce Funnel Analysis

An end-to-end analytics project querying Google Analytics 4 (GA4) public data via BigQuery to map a 5-stage e-commerce purchase funnel, segmented by device type, country, and traffic medium. Findings are visualised in an interactive Looker Studio dashboard.

---

## Tools & Technologies

| Tool | Purpose |
|------|---------|
| BigQuery (SQL) | Querying 326K+ GA4 sessions from the public GA4 obfuscated sample dataset |
| Looker Studio | Interactive dashboard with device, country, and traffic medium filters |
| Google Analytics 4 | Data source (obfuscated public ecommerce dataset) |

---

## Dataset

Source: `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`

This is Google's publicly available, obfuscated GA4 sample dataset representing the Google Merchandise Store — a real e-commerce store selling Google-branded products.

---

## Funnel Stages Analysed

| Step | Event Name |
|------|-----------|
| 1 | `session_start` |
| 2 | `view_item` |
| 3 | `add_to_cart` |
| 4 | `begin_checkout` |
| 5 | `purchase` |

---

## Key Findings

### 1. Overall Revenue
Total revenue across all devices: **$362,165**

| Device | Revenue | Share |
|--------|---------|-------|
| Desktop | $208,815 | 57.7% |
| Mobile | $146,768 | 40.5% |
| Tablet | $6,582 | 1.8% |

### 2. Biggest Funnel Leak — Session → View Item (~78% drop-off)
Across all device types, roughly **78% of users who start a session never view a product**. This is the single largest drop-off point and the highest-impact area for optimisation.

### 3. Mobile Outconverts Desktop

| Device | Sessions | Purchases | Conversion Rate |
|--------|----------|-----------|----------------|
| Desktop | 189,411 | 2,693 | 1.42% |
| Mobile | 129,975 | 1,961 | **1.51%** |
| Tablet | 7,352 | 102 | 1.39% |

Despite desktop driving higher session volume, **mobile users convert at a higher rate (1.51% vs 1.42%)**, challenging the common assumption that desktop users are more likely to purchase.

### 4. Tablet is a Negligible Segment
Tablet contributes only **$6,582 in revenue** and less than 5% of sessions.

---

## Dashboard

Built in Looker Studio with interactive filters for Device Type, Country, and Traffic Medium.

🔗 [View Live Dashboard](https://lookerstudio.google.com/u/0/reporting/050cad8b-490a-47f0-9186-7b55dcd47473/page/h5osF)

---

## SQL Query

```sql
WITH base_funnel AS (
  SELECT
    device.category AS device_type,
    geo.country AS country,
    traffic_source.medium AS traffic_medium,
    event_name AS funnel_stage_raw,
    COUNT(DISTINCT user_pseudo_id) AS unique_users,
    SUM(ecommerce.purchase_revenue) AS total_revenue
  FROM
    `bigquery-public-data.ga4_obfuscated_sample_ecommerce.events_*`
  WHERE
    event_name IN ('session_start', 'view_item', 'add_to_cart', 'begin_checkout', 'purchase')
  GROUP BY
    device_type, country, traffic_medium, funnel_stage_raw
),
ordered_funnel AS (
  SELECT
    device_type,
    country,
    traffic_medium,
    CASE
      WHEN funnel_stage_raw = 'session_start'   THEN '1 - session_start'
      WHEN funnel_stage_raw = 'view_item'        THEN '2 - view_item'
      WHEN funnel_stage_raw = 'add_to_cart'      THEN '3 - add_to_cart'
      WHEN funnel_stage_raw = 'begin_checkout'   THEN '4 - begin_checkout'
      WHEN funnel_stage_raw = 'purchase'         THEN '5 - purchase'
    END AS funnel_stage,
    CASE
      WHEN funnel_stage_raw = 'session_start'   THEN 1
      WHEN funnel_stage_raw = 'view_item'        THEN 2
      WHEN funnel_stage_raw = 'add_to_cart'      THEN 3
      WHEN funnel_stage_raw = 'begin_checkout'   THEN 4
      WHEN funnel_stage_raw = 'purchase'         THEN 5
    END AS step_number,
    unique_users,
    total_revenue
  FROM base_funnel
)
SELECT
  device_type,
  country,
  traffic_medium,
  step_number,
  funnel_stage,
  unique_users,
  total_revenue,
  ROUND(SAFE_DIVIDE(unique_users, LAG(unique_users) OVER(
    PARTITION BY device_type, country, traffic_medium ORDER BY step_number
  )) * 100, 2) AS step_to_step_conv_rate,
  ROUND(SAFE_DIVIDE(unique_users, FIRST_VALUE(unique_users) OVER(
    PARTITION BY device_type, country, traffic_medium ORDER BY step_number
  )) * 100, 2) AS overall_conv_rate
FROM ordered_funnel
WHERE device_type IS NOT NULL
ORDER BY device_type, country, traffic_medium, step_number;
```

---

## Recommendations

1. Fix the session → view item drop — improve product discoverability and landing page relevance
2. Invest in mobile — already converts better; further UX improvements could widen the lead
3. Optimise desktop checkout — highest absolute purchase volume, so reducing friction here has the largest revenue impact
4. Deprioritise tablet — only 1.8% of revenue, low ROI compared to other segments



