# E-Commerce Data Mart

A production-style analytics engineering project built with dbt and BigQuery, 
modeling two years of synthetic e-commerce data into a queryable data mart 
for revenue, customer, and retention analysis.

## Architecture

raw (BigQuery) → staging (views) → intermediate (views) → marts (tables)

![alt text](docs/lineage_graph.png)

## Models

| Layer | Model | Description |
|---|---|---|
| Staging | stg_customers | Cleaned customer records |
| Staging | stg_orders | Orders with amounts in dollars |
| Staging | stg_order_items | Line items per order |
| Staging | stg_products | Product catalog with dollar prices |
| Staging | stg_stores | Store locations and open dates |
| Intermediate | int_orders_with_customers | Orders joined with customer info |
| Intermediate | int_order_items_with_products | Line items joined with product details |
| Mart | fct_orders | Order-level fact table with revenue and status |
| Mart | fct_order_items_enriched | Product-level fact table for SKU analysis |
| Mart | dim_customers | Customer LTV, order counts, and segment |
| Mart | fct_revenue_daily | Daily revenue by store with surrogate key |
| Mart | fct_customer_cohorts | Monthly cohort retention rates |

## Architecture Decisions

**Why `{{ source() }}` instead of hardcoded table paths?**  
Using the `source()` macro connects raw tables to the dbt DAG, enabling lineage tracking from raw → staging → marts. It also unlocks `dbt source freshness` checks so pipelines alert when data stops arriving. Hardcoding paths breaks lineage and is considered an anti-pattern in production.

**Why views for staging/intermediate and tables for marts?**  
Views have no storage cost and are always fresh — ideal for the cleaning and joining layers where we want changes to propagate immediately. Mart tables are materialized because BI tools query them repeatedly; a table is far faster than re-executing a multi-join view at query time.

**Why a surrogate key on `fct_revenue_daily`?**  
Aggregated models have no natural single-column primary key. Using `dbt_utils.generate_surrogate_key(['order_date', 'store_id'])` creates a stable, testable key from the composite of columns that uniquely identify a row. This is the production pattern for any aggregated fact table.

**Why does `fct_order_items_enriched` join to `fct_orders` rather than the intermediate layer?**  
Marts can reference other marts via `ref()` when the upstream mart is stable and tested. Joining to `fct_orders` means the product-level fact inherits the same business logic (is_completed flag, order_month calculation) without duplicating it.

## Data Quality

- 25+ tests across all layers (unique, not_null, relationships, accepted_values)
- Surrogate key uniqueness enforced on aggregated mart
- Source freshness configured on raw tables
- All tests passing on 2 years of synthetic data (~100k+ rows)

## Dashboard

Built in Looker Studio connected directly to BigQuery mart tables.  
4 pages: Revenue Overview · Customer Analysis · Product Performance · Cohort Retention

https://datastudio.google.com/reporting/6d8eb6e3-dd09-42b1-a00c-5b2cba397b22

## Business Questions Answered

1. What is daily and monthly revenue by store?
2. Which customer segments drive the most lifetime value?
3. Which products and categories generate the most revenue?
4. What is the retention rate for each monthly signup cohort?

## Stack

dbt · dbt_utils · BigQuery · Looker Studio · Python (jafgen) · GitHub