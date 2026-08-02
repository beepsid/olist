# Olist E-Commerce Analytics Dashboard

End-to-end data analytics project built on the public [Olist dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) — 9 relational CSVs, ~100,000 real anonymized Brazilian e-commerce orders from 2016–2018. The emphasis is the full pipeline: data auditing → cleaning decisions → relational modeling → DAX → dashboard design.

**Tech stack:** Power Query (M) · Power BI · DAX · Star Schema · Python / pandas

---

## Dataset

| File | Description |
|------|-------------|
| `olist_customers_dataset.csv` | Customer IDs and location |
| `olist_geolocation_dataset.csv` | Zip-code-level lat/lng |
| `olist_orders_dataset.csv` | Order lifecycle timestamps |
| `olist_order_items_dataset.csv` | Items, prices, freight per order |
| `olist_order_payments_dataset.csv` | Payment method and value |
| `olist_order_reviews_dataset.csv` | Review scores and comments |
| `olist_products_dataset.csv` | Product attributes (weight, dimensions, category) |
| `olist_sellers_dataset.csv` | Seller IDs and location |
| `product_category_name_translation.csv` | Portuguese → English category names |

---

## Data Cleaning (Power Query)

The dataset is not plug-and-play. Below is every issue found, the decision made, and the reason — no real revenue rows were dropped.

| # | Issue Found | Decision Made | Why |
|---|-------------|---------------|-----|
| 1 | 610 products missing category name | Bucketed as "Uncategorized" | Rows still carried real revenue data; dropping would understate sales |
| 2 | 2 products missing all physical dimensions | Dropped | Statistically immaterial (2 of 33k rows) |
| 3 | 4 products with `weight_g = 0` | Imputed with category-level median weight | Category-specific imputation avoids nonsensical cross-category fallback |
| 4 | Orders missing approval/delivery dates | Left as null | Canceled orders legitimately never got delivered — filling would fabricate data |
| 5 | 775 orders with no matching order items | Kept in orders table, excluded from revenue measures | Preserves accurate cancellation-rate reporting |
| 6 | 814 duplicate review IDs / 551 orders with 2+ reviews | Deduplicated, kept latest review per order | Latest review reflects the customer's final, settled sentiment |
| 7 | 9 payments with value ≤ 0 | Kept, flagged | Likely voucher-only orders — verified against `payment_type` before treating as anomaly |
| 8 | 2 credit card payments with 0 installments | Left as-is | Imputing a value is an assumption; only 2 rows |
| 9 | 3 payments with type = "not_defined" | Recoded to "Other" | Clean categorical grouping |
| 10 | Zip codes truncated (leading zero loss) across customers, sellers, geolocation | Re-cast as text, zero-padded to 5 digits | Silent join failures on any zip starting with 0 |
| 11 | Product category names inconsistent casing/language | Standardized via Power Query, merged 2 missing categories into translation table | Ensures every product resolves to a real English category |
| 12 | 261,831 duplicate rows in geolocation table (1M+ rows total) | Deduplicated, then grouped by zip prefix using centroid (mean lat/lng) | Reduces table to ~19K clean rows; centroid is defensible at city/state map zoom |
| 13 | 31 geolocation rows with lat/lng outside Brazil's bounding box | Filtered out before centroid aggregation | Prevents bad points from skewing the centroid average |
| 14 | Geolocation table had no relationship to customers/sellers | Built as role-playing dimensions via Power Query References | Avoids Power BI's single-active-relationship-path limitation |

---

## Data Model

Star schema with two fact tables and role-playing dimension tables.

**Fact tables**
- `fact_orders` — order-level metrics and timestamps
- `fact_order_items` — line-item revenue and freight

**Dimension tables**
- `dim_products` — product attributes and English categories
- `dim_customers` — customer location
- `dim_sellers` — seller location
- `dim_customer_geo` — role-playing geo dimension (customer side)
- `dim_seller_geo` — role-playing geo dimension (seller side)
- `dim_calendar` — custom DAX calendar table for time intelligence

**Role-playing dimensions**  
Power BI only allows one active relationship from a single dimension table. Both customer and seller location needed to map to the same geo table, so the cleaned geo table was duplicated into `dim_customer_geo` and `dim_seller_geo` via Power Query References — meaning the cleaning logic exists only once and both tables stay in sync.

**Supporting technical details**
- Custom Calendar table built with `CALENDARAUTO()` + `ADDCOLUMNS`, including a numeric `YearMonthSort` column to guarantee correct chronological sorting in visuals (text-based YearMonth sorts alphabetically, which breaks time-series charts)
- Multiple inactive relationships from Calendar to secondary date columns (delivered date, review date) for `USERELATIONSHIP()` time-intelligence measures
- Fixed a cross-filter direction bug: a Single-direction relationship between orders and reviews was silently causing an "Avg Delivery Delay by Review Score" chart to show the same overall average in every bar — fixed by setting the relationship to Both

---

## DAX Measures (20+)

- **Revenue** — total revenue, average order value, freight ratio, revenue by category/seller/state
- **Delivery performance** — on-time / late / early classification, average delivery delay in days, delivery rate by seller and state
- **Time intelligence** — month-over-month growth, rolling 3-month revenue, year-to-date comparisons

---

## Key Findings

- **Delivery timing drives review sentiment** — orders arriving with little or no buffer ahead of the estimated delivery date skewed toward significantly worse review scores, even before counting outright-late deliveries
- **Supply concentrated, demand spread** — seller supply is heavily concentrated in São Paulo, Paraná, and Minas Gerais, while customer demand is distributed much more evenly across all states
- **78% credit card, spike at 10 installments** — consistent with Brazil's common *"10x sem juros"* (interest-free in 10) retail financing pattern
- **Clear seasonal revenue spike** — pronounced peak around November 2017, matching Brazil's Black Friday / holiday season

---

## Dashboard Pages

The `.pbix` file contains 5 interactive pages:

1. **Overview** — high-level KPIs: total orders, revenue, average review score, delivery on-time rate
2. **Delivery Performance** — on-time/late/early breakdown, delay distribution, delivery trends over time
3. **Product & Category Analysis** — top categories by revenue and volume, average product weight vs freight cost
4. **Customer & Seller Geography** — choropleth maps showing order density and revenue by Brazilian state
5. **Payments & Reviews** — payment method split, installment usage, review score distribution and drivers

---

## Tools

- **Power BI Desktop** — data modeling, DAX, and dashboard
- **Power Query (M)** — all data cleaning and transformation
- **Python / pandas** — initial data auditing

---

## Source Data

Olist Brazilian E-Commerce dataset — publicly available on [Kaggle](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce), released under [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/).
