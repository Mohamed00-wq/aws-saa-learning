# AWS Data Exchange  Third-Party Data Marketplace

## Purpose

AWS Data Exchange is **AWS's data marketplace**  find, subscribe to, and use **third-party data products** (financial, healthcare, geospatial, consumer/media, maps, etc.) while **billing on your AWS invoice** instead of starting external vendor contracts. Catalogs contain **3,500+ data sets from 300+ providers** (incl. 1,000+ free/open ones). On the flip side, **providers** can publish and monetize data products with automated subscription, entitlement, billing, and delivery. For SAA: "the place to **buy/acquire third-party reference data** (or **sell** your data) via AWS"  a marketplace, not a compute/ETL/warehouse service.

## Main use cases

- **Subscribing** to third-party/enrichment data  market/alternative data, demographics, weather, geo, property, healthcare  and joining it with your own analytics
- **Exporting bought data into your S3 data lake / Redshift** for immediate Athena/Redshift/QuickSight analysis (no FTP/shipped media)
- **Licensing YOUR data**  publishers/ISVs monetize datasets via files, **Amazon Redshift datashares**, or **APIs** with AWS-managed billing/entitlements
- **Governed data share for data mesh / data exchange programs** (e.g., from Lake Formation governed tables)
- **Automating refresh**  pick up **new revisions** automatically via EventBridge/S3 export jobs (no manual downloads)

## Key features

- **Discovery & subscription**  Marketplace catalog with standard/custom/private offers 1–36 month terms free & paid products
- **Delivery methods (subscribable)**  **export files to S3**, access provider S3 bucket via access points, **query via Redshift datashare** (live, no ETL), **invoke via API** (API Gateway), and **Lake Formation data share** for governed lake access
- **Automated updates**  CloudWatch Events/EventBridge auto-retrieve **new revisions** (Data Sets → Revisions → Assets model)
- **Provider tooling**  publish S3/on-prem data as products, private offers, custom agreements, automated entitlements/billing, commerce reports
- **AWS-native**  consolidated AWS billing, IAM integration, encryption, Private Marketplace for org-approved catalogs

## When to use

- Need **third-party vendor data** and want procurement via AWS (contracts + billing in one place)
- Want third-party data delivered **directly into S3/Redshift/analytics** with automated refresh
- Enrich internal data lakes/warehouses for ML, BI, risk, geo, or marketing use cases
- You're a **data provider** wanting a storefront with automated monetization

## Important limitation

- It's a **marketplace**  you do **not** build ETL here you subscribe, then **delivery/export is set-and-forget** but querying/processing still happens in Athena/Redshift/EMR. **Data licensing/terms** are each provider's not all products are free, and provider updates define freshness (subscribe to revisions, some are periodic). You're limited to **catalog availability** (product must exist 300+ providers, but not every dataset you can imagine). Not for building your own bespoke data-pipeline ingestion of arbitrary third-party feeds  that's Glue/AppFlow/S3.

## SAA relevance

- "**Third-party datasets / subscribe to vendor data / exchange data** for analytics" → **AWS Data Exchange**
- "Dataset + **automated updates into S3/Redshift** with consolidated AWS billing" → Data Exchange (delivery methods)
- "Provider monetizing data via AWS" → publish product with shares/APIs/revisions
- "**Billing/entitlement/invoicing** solved by AWS Marketplace mechanism" → Data Exchange
- Exam traps: Data Exchange = **marketplace for data products** (≠ ETL, ≠ query engine, ≠ data lake storage) know the **Data Set → Revision → Asset** model for fine-grained sharing *within your org* **use Lake Formation ressource links/Data Exchange export**, for public third-party data **use Data Exchange subscriptions**. Mnemonic: **“data from outside → Data Exchange govern inside → Lake Formation.”**