# 🛒 E-commerce Lakehouse ML Pipeline: Solution Design

**Architecture design for a Databricks lakehouse that turns e-commerce clickstream into purchase predictions and product recommendations.**

![Status](https://img.shields.io/badge/status-architecture%20design-blue?style=flat-square)
![Databricks](https://img.shields.io/badge/Databricks-FF3621?style=flat-square&logo=databricks&logoColor=white)
![Delta Lake](https://img.shields.io/badge/Delta%20Lake-00ADD4?style=flat-square)
![Apache Spark](https://img.shields.io/badge/Apache%20Spark-E25A1C?style=flat-square&logo=apachespark&logoColor=white)
![MLflow](https://img.shields.io/badge/MLflow-0194E2?style=flat-square&logo=mlflow&logoColor=white)

> **Scope:** this repository is a **design document**. It covers the architecture, pipeline flow and model-retraining strategy, written as a solution-design exercise. It contains no implementation code.

---

## Problem

An e-commerce business wants to know **which users are likely to buy** and **what to recommend to them**, using behavioural events (views, clicks, add-to-cart, purchases), with a pipeline that keeps the models accurate as behaviour changes.

## Architecture

```mermaid
flowchart LR
    A[Clickstream events<br/>views · clicks · cart · purchases] --> B[Ingestion<br/>batch / Auto Loader]
    B --> C[(Bronze<br/>raw Delta, append-only)]
    C --> D[(Silver<br/>cleaned + user features)]
    D --> E[Feature Store]
    E --> F[Training<br/>MLflow tracking]
    F --> G[Model Registry<br/>staging → production]
    G --> H[Batch scoring]
    H --> I[(Gold<br/>predictions + recommendations)]
    I --> J[Dashboards · marketing · recommendation engine]
    G -. drift / performance triggers .-> F
```

## Pipeline flow

| Stage | Responsibility | Key outputs |
|---|---|---|
| **Ingestion** | Land raw events in Delta (batch, or Auto Loader for incremental files) | append-only event tables |
| **Bronze** | Keep events exactly as received, for traceability and replay | `user_events`, `product_clicks`, `transaction_logs` |
| **Silver** | Deduplicate, handle nulls, standardise timestamps, aggregate per user | `total_events`, `total_spent`, `number_of_purchases`, `average_product_price`, `distinct_products_viewed` |
| **Feature Store** | Serve the same user features to training and scoring | versioned feature tables |
| **Training** | Logistic Regression / Random Forest for purchase propensity; ALS collaborative filtering for recommendations; experiments tracked in MLflow | candidate models with metrics |
| **Model Registry** | Lifecycle: development → staging → production; promote only if the metric improves | production model version |
| **Batch scoring** | Apply the production model to fresh user features | `purchase_probability`, `recommended_products` |
| **Gold** | Curated outputs for the business | `customer_purchase_predictions`, `product_recommendations`, aggregated metrics |

## Retraining strategy

- **Scheduled:** retrain weekly or monthly on new activity data.
- **Data drift:** monitor feature distributions and trigger retraining on significant shift.
- **Performance:** trigger retraining when AUC or accuracy falls below a threshold.
- **Versioning:** every retrain registers a new model version. Compare it with production, promote if better, otherwise discard.

## Design decisions

- **Medallion layers** separate raw retention (Bronze) from business logic (Silver/Gold), so features can be rebuilt from history.
- **A feature store** keeps training and scoring features identical, avoiding training/serving skew.
- **Batch, not real-time, scoring:** propensity and recommendations change slowly, so a scheduled batch job is cheaper and simpler than an online endpoint.

## Stack

Databricks · Apache Spark · Delta Lake · Spark MLlib · MLflow (tracking + registry) · Databricks Feature Store

## Next step

Implement the design end to end in a Databricks notebook project: Auto Loader ingestion, Silver feature tables, an MLflow-tracked propensity model, and a scheduled batch-scoring job.
