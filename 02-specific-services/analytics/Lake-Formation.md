# AWS Lake Formation — Data Lake Governance & Fine-Grained Security

## Purpose

AWS Lake Formation makes it easy to **set up and centrally govern a secure data lake** on S3. It centralizes metadata + permissions on the **AWS Glue Data Catalog** and enforces **fine-grained access control (FGAC)** — at **database, table, column, row, and cell** level — across analytics engines, using RDBMS-style **GRANT/REVOKE** instead of hand-writing S3 bucket policies per user. It also provides catalogs for **governed data sharing** (data mesh, cross-account, via AWS Data Exchange) and access auditing by user/role.

## Main use cases

- **Governed data lakes on S3** — register locations, crawl & catalog data, grant teams access in one place
- **Fine-grained security** — column/row-level (row filters), cell-level masking to protect PII/regulated data from analytics users
- **Central permissions across tools** — one permission model enforced consistently on **Athena, Redshift Spectrum, EMR, AWS Glue, QuickSight** (single set of grants applied at scale)
- **Scalable permission administration** — **LF-Tags** (attribute-based access control) to grant on logical attributes instead of thousands of policies
- **Data sharing** — **resource links** & cross-account/cross-Region catalog sharing, and export of governed tables via **AWS Data Exchange** (data mesh without moving data)
- **Auditing compliance** — track *who accessed what data when* by user and role

## Key features

- **Central permissions model** on Glue Data Catalog resources (databases/tables/columns) + S3 locations
- **FGAC** — column, row, and cell-level data filtering across Athena, Redshift Spectrum, EMR, Glue ETL, QuickSight
- **RDBMS grant/revoke** (GRANT SELECT ON table…) — replaces complex S3 bucket + IAM policy sprawl
- **LF-Tags (TBAC)** — attribute-based access control & cross-engine sharing of governed data
- **Data lake admin persona** (up to 30), hybrid access mode (Lake Formation + IAM policies together), IAM Identity Center integration
- **Blueprints & workflows** — automated ingestion/cataloging from S3, RDS, etc.
- **Catalog unification** — multi-level federated catalogs; external sources (Redshift, DynamoDB, Snowflake, 30+ via federated connectors)
- **Audit** — CloudTrail logging of data access

## When to use

- Any S3 data lake needing central, auditable, fine-grained (column/row) permissions for many users across many analytics engines
- PII/regulated data that must be restricted per user beyond "whole bucket/bucket-policy" level
- Multi-account or org-wide data sharing without duplicating data (resource links / Data Exchange export)
- Replacing brittle S3 bucket policies + per-service IAM with a single manageable grants model
- Scaling grants via tags when you have hundreds/thousands of tables

## Important limitation

- **Two "doors" must both pass**: a caller needs **both** Lake Formation grants **and** IAM permissions (e.g., `glue:*`, `s3:*`) — misconfiguration leaves users denied; grants are **per-Region**. Fine-grained governance really shines with **Glue Data Catalog-backed tables** (S3 + catalog); **cross-engine cell/masking** nuance & hybrid-mode complexity add operational overhead. Not a replacement for protecting *compute* (that's IAM/Network/Firewall) — it governs *data*. Lake Formation is **not a separate S3 storage service**; S3 still stores the data (its policies get replaced by LF for governed locations).

## SAA relevance

- "**Data lake on S3 + centralized, fine-grained (column/row) permissions** across Athena/Redshift/EMR" → **AWS Lake Formation**
- "**GRANT/REVOKE-style** data permissions / **LF-Tags** / cross-account **data sharing** without copying" → Lake Formation
- "Encrypt/rotate data lake keys" → add **KMS**; "discover PII" → **Macie**; "centrally govern multiple accounts' S3" → Lake Formation (with Organizations)
- Exam traps: LF is **governance on top of S3 + Glue Catalog** (not storage, not a query engine); both **IAM + LF** must grant access; LF ≠ Macie (that finds PII) / ≠ Athena (that queries) / ≠ Glue ETL (though crawlers build its catalog). "**Configure S3 bucket policies per user/table**" → **wrong; centralize in Lake Formation**.