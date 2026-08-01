# Marketing Analytics & Customer Intelligence Pipeline

## Overview

This project is an end-to-end data engineering pipeline that ingests, cleans,
and transforms multi-source e-commerce data into analytics-ready tables for
marketing and sales reporting — with a customer segmentation model as the
final step.

**Motivation:** During a summer internship at NoeCee Gloal, I manually
consolidated and analyzed marketing and sales data across multiple stores
using Excel, then prepared the resulting summaries for leadership review.
That process was manageable at the internship's scale, but it doesn't hold up
as data volume, source count, or reporting frequency grows — every new store,
channel, or reporting cycle meant repeating the same manual work from
scratch.

This project takes that same underlying problem — multi-source data
consolidation feeding leadership-ready reporting — and rebuilds it as an
automated, scalable pipeline. I used a public e-commerce dataset (Olist,
~100K orders across multiple independent sellers and regions) rather than the
internship's own data, both because the internship data was small and
proprietary, and because building this at realistic production volume forced
me to solve real engineering problems — incremental processing, data quality
enforcement, orchestration — that a small manual dataset never would have
surfaced.

## What this pipeline does

- **Ingests** multi-source order, customer, payment, and marketing-lead data,
  plus external signals (public holidays, search interest trends), simulating
  incremental daily arrival rather than a single batch load.
- **Cleans and validates** the data through a Bronze → Silver → Gold
  (medallion) architecture, with explicit data quality checks and a
  quarantine path for bad records.
- **Aggregates** the data into business-facing Gold tables that mirror the
  kind of leadership reporting I built by hand during the internship: sales
  and conversion performance by store/seller, channel, region, and season.
- **Feeds a downstream model**: RFM-based customer segmentation, replacing
  what would otherwise be a recurring manual analysis step.

## Architecture

```
Sources (Olist orders/customers/payments/leads, Holiday API, Google Trends)
        │
        ▼
   Bronze (raw, as-landed, with ingestion metadata)
        │
        ▼
   Silver (cleaned, validated, conformed schema)
        │
        ▼
   Gold (business aggregates: sales, conversion, seasonality)
        │
        ▼
   ML: Customer segmentation (RFM + clustering), logged with MLflow
```

Orchestrated end-to-end with Databricks Workflows; incremental loads handled
via Delta Lake MERGE rather than full reprocessing.

## Why this design

*(This section gets filled in as each week's decisions get made — e.g. why
Auto Loader over a scheduled batch read, why partitioning by date, why
quarantine instead of drop-on-failure. Documenting real trade-offs here, not
just describing what the code does.)*

**Silver layer validation strategy:** I used an inner join between orders and
customers when building the Silver layer, rather than a left join, to
guarantee referential integrity in the cleaned data — every order in Silver
is guaranteed to have a valid, matching customer record. Orders that fail
this check (along with duplicate `order_id`s and rows with a null
`order_status`) are not silently dropped; they're written to separate
`silver_quarantine` Delta tables, so any excluded record can be audited later
without re-running the full pipeline. This mirrors a real data quality
practice: bad data should be visible and traceable, not invisible.

## Gold Layer

The Gold layer turns validated Silver data into business-facing, aggregated
tables — each one built to answer a specific question a marketing or sales
stakeholder would actually ask, mirroring the kind of leadership reporting
I built manually in Excel during my internship.

### Tables

**`gold/sales_by_region`** — Which states drive the most revenue, and how
does order volume compare across regions? Aggregates total orders, total
revenue, and average order value by customer state.

**`gold/daily_trends`** — Is order volume and average order value growing,
shrinking, or seasonal over time? Aggregates daily order count and average
order value across the full date range of the dataset.

**`gold/holiday_comparison`** — Do public holidays measurably affect order
volume or order value? Joins daily order trends against the Brazilian
public holiday data pulled from the Nager.Date API in Week 1, comparing
average daily orders and order value between holiday and non-holiday days.
This table closes the loop on the project's first external API
integration.

**`gold/payment_method_breakdown`** — How do customers pay, and does
payment method correlate with order value? Aggregates total orders,
revenue, and average order value by payment type.

### A schema lesson worth documenting

While building Gold tables, joining `orders_silver` against
`customers_silver` produced an `AMBIGUOUS_REFERENCE` error — both tables
turned out to carry a `customer_state` column, since `orders_silver` had
already picked one up during its own Day 3 validation join. The fix was to
check each table's actual schema before joining, rather than assuming a
table only contains what its name implies — a useful reminder that
real-world debugging is part of the DE workflow, not a sign something went
wrong.

## Orchestration

Bronze, Silver, and Gold notebooks are connected into a Databricks
Workflow with explicit task dependencies (Bronze → Silver → Gold), so the
full pipeline can run end-to-end with a single trigger rather than being
run manually, notebook by notebook. Task dependencies enforce that Silver
cannot start until Bronze succeeds, and Gold cannot start until Silver
succeeds.
---
*Status: Week 1 — data ingestion (Bronze layer) in progress.*

*Status: Bronze, Silver, and Gold layers complete. Orchestration
(Databricks Workflows) in progress.*
