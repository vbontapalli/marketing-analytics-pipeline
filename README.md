# Marketing Analytics & Customer Intelligence Pipeline

## Overview

This project is an end-to-end data engineering pipeline that ingests, cleans,
and transforms multi-source e-commerce data into analytics-ready tables for
marketing and sales reporting — with a customer segmentation model as the
final step.

**Motivation:** During a summer internship at [Firm Name], I manually
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

---
*Status: Week 1 — data ingestion (Bronze layer) in progress.*
