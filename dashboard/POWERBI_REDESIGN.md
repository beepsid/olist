# Power BI redesign: Olist dashboard

Goal: make `dashboard.pbix` match the interactive web version (`dashboard.html`). Expect about 45 minutes.

## 1. Apply the theme (2 min)
View → Themes → Browse for themes → `dashboard/powerbi-theme.json`.
This gives a warm off-white page, white rounded cards, one blue for data, red only for "bad", and quiet gridlines.

## 2. New measures
```DAX
Late Orders % =
DIVIDE(
    CALCULATE(COUNTROWS(fact_orders), fact_orders[order_status] = "delivered",
              fact_orders[order_delivered_customer_date] > fact_orders[order_estimated_delivery_date]),
    CALCULATE(COUNTROWS(fact_orders), fact_orders[order_status] = "delivered"))

Delivery vs Estimate (days) =
DATEDIFF(fact_orders[order_estimated_delivery_date], fact_orders[order_delivered_customer_date], DAY)   -- calculated column

Delivery Timing Bucket =                                                                          -- calculated column
SWITCH(TRUE(),
    [Delivery vs Estimate (days)] <= -15, "1. 15+ days early",
    [Delivery vs Estimate (days)] <= -8,  "2. 8–14 days early",
    [Delivery vs Estimate (days)] <= -1,  "3. 1–7 days early",
    [Delivery vs Estimate (days)] = 0,    "4. On the estimate day",
    [Delivery vs Estimate (days)] <= 7,   "5. 1–7 days late",
    "6. 8+ days late")

One-Star Share = DIVIDE(CALCULATE(COUNTROWS(dim_reviews), dim_reviews[review_score] = 1), COUNTROWS(dim_reviews))
Seller Revenue Share = DIVIDE([Total Delivered Revenue], CALCULATE([Total Delivered Revenue], ALL(dim_seller_geo)))
```
(Rename the tables to match your model if they differ.)

## 3. Overview page layout (16:9, 1280×720)
| Row | Visuals |
|---|---|
| Top (y 16) | Text box title "Olist marketplace: what drives revenue and reviews" plus a one-line subtitle |
| KPI row (y 80, height 90) | 6 cards: Total orders · Delivered revenue · Avg order value · Avg delivery days · Late % · Avg review |
| Middle left (60% width) | Area/line: Delivered revenue by YearMonth, Jan 2017–Aug 2018. Analytics pane → add a constant line at Nov 2017 labelled "Black Friday" |
| Middle right (40%) | **Replace the order-status donut** (97% one slice says nothing) with a column chart: Avg review by Delivery Timing Bucket. Conditional formatting: red for the two "late" buckets, blue otherwise. Constant line at 4.09 |
| Bottom | Left: top 12 categories bar (revenue). Right: table of states with Revenue, demand share, supply share, avg delivery, and Late % with red/green background rules |

## 4. Titles that state the finding
Use sentence titles, not field names:
- "Late deliveries wreck reviews: 1.7★ when 8+ days late vs 4.3★ when early"
- "Revenue peaked at Black Friday 2017"
- "Sellers sit in São Paulo; buyers are spread across Brazil"

## 5. Polish checklist
- Turn off visual borders and shadows that the theme doesn't already handle; keep 16 px gaps between cards
- Number formats: R$ with 2 decimals for millions (R$13.22M), whole numbers for counts
- Tooltips: add Orders and One-Star Share to the review chart tooltip
- Remove the unused pages' default "Page 1" names; name them Overview · Delivery · Products · Geography · Payments
- Export a new screenshot to `images/dashboard.png` for the README and your Upwork portfolio
