# Amazon QLDB Course  Fully Managed Ledger Database (Immutable Journal)

## 1. Purpose

Amazon Quantum Ledger Database (QLDB) is a **fully managed ledger database** with an **immutable, cryptographically verifiable journal** of all change history  a **centralized (single-owner) system of record**. Unlike a blockchain it doesn't need multi-party consensus you own the data and get tamper-evident, auditable history. For SAA it's the answer to **"ledger / audit trail / immutable history / verifiable record of changes / financial statements"**.

## 2. How it works

- **Journal-first architecture**  every write is committed to an **append-only journal** with a **SHA-256 hash chain**: each record's digest includes the previous digest, producing a **cryptographically linked, tamper-evident history**
- **Document model (JSON/Ion)**  tables of documents updated via **PartiQL** (SQL-compatible) you query *current* data and the *history* (`MOVE_NEXT`/visit journal history)
- **Digest verification**  QLDB produces a periodically published **digest** you can verify any record's integrity offline (`VerifyDigest`, GetRevision) against the journal to prove it wasn't altered
- **Streaming (CDC)**  `JournalS3` or **Kinesis Data Streams** for QLDB captures changes to downstream consumers (analytics, replication)
- **Export**  full journal/table export to **S3** (Ion/JSON/parquet via export), then query with **Athena**/Glue  historically the SAA-relevant companion
- **Serverless**  no capacity provisioning scales automatically up to a certain concurrency (writes up to 2%, reads higher  check quotas)
- ACID transactions with PartiQL, indexes (AS OF-style queries), encryption at rest (KMS)/TLS in transit

```
App → PartiQL (SQL-like) → QLDB (immutable hash-chained journal, document/Ion model)
  ├─ query current state + history (provenance/audit)
  ├─ digest verification (prove record was never altered)
  ├─ Streams (Kinesis) / JournalS3 export → downstream analytics
  └─ S3 export → Athena/Glue for reporting
```

## 3. When to use

- **Audit/compliance trail**  you must prove WHEN and HOW every change happened
- **Financial applications**  credit/debit, payment ledger, general ledger, card transactions, order commitments
- **Insurance claims**, HR records, compensation data, **supply-chain/ERP** event history, dispute resolution
- **"System of record" with verifiable history** where tamper-evidence matters (regulators)
- **Where data history is as important as current state**

## 4. When NOT to use

- **General-purpose database** (CRUD, relationships, query flexibility) → **DynamoDB / RDS/Aurora**
- **Blockchain/multi-party**  QLDB is a **single-owner** ledger no consensus trust across independent parties (that's Amazon Managed Blockchain)
- **Very high write concurrency OLTP**  QLDB serializes writes more conservatively (quota-limited writer threads)
- **Complex relational reporting/ad-hoc SQL analytics** → RDS/Redshift
- **Only need current state & speed, history unimportant** → DynamoDB/RDS is simpler

## 5. Important features

- **Immutable, append-only journal** with **SHA-256 hash chain**  tamper-evident history-by-design
- **Digest & revision verification**  cryptographically prove data integrity offline (GetDigest, VerifyRevision)
- **PartiQL** (SQL-compatible) queries over current state AND journal history
- **ACID transactions**, document (Ion/JSON) model, indexes for point lookups
- **Kinesis Data Streams integration (streams 2.0)** for CDC → real-time downstream (analytics, replication, search index)
- **S3 export** (Ion/JSON/parquet ready) + **Athena/Glue** query path for reporting
- **Serverless**  auto-scale, pay-as-you-go, no capacity management
- **Security**  IAM, KMS at rest, TLS in transit, CloudTrail

## 6. Limitations

- **Centralized ledger**  NOT a blockchain no multi-party consensus/decentralized trust
- **Concurrency ceilings**  write throughput quotas (writer threads) make it unsuitable for very high-write OLTP
- **All data in a single Region** by default (historically no native cross-Region active-active replicas/annex avaibility checked per region)
- **PartiQL subset + Ion model, not full SQL**  some complex relational queries are awkward
- Ledger size & document-size limits apply streams sent to Kinesis for durable CDC

## 7. Trade-offs

- **QLDB vs DynamoDB**  auditable immutable history + verification vs current-state key-value at massive scale (can always run BOTH: DynamoDB for hot data, QLDB for the audit journal)
- **QLDB vs RDS/Aurora**  ledger/provenance semantics vs general relational OLTP (RDS has no built-in tamper-evidence journal)
- **QLDB vs Amazon Managed Blockchain**  single-owner verifiable service vs multi-party decentralized consensus (different trust model)
- **Trade cost/complexity for provenance**  QLDB adds ledger semantics & concurrency constraints vs a plain DB

## 8. Architecture

```
Financial ledger (e.g., banking balance):
  API → Lambda → QLDB (PartiQL transaction: credit/debit both committed atomically)
  ├─ Streams (Kinesis) → aggregation/reconciliation + alerting on anomalies
  ├─ digest verification job → evidence for auditors
  └─ periodic S3 export → Athena: regulatory reports, "prove this transaction happened"
Retain full change history forever  never overwrite, only append (MOVE_NEXT history)
```

## 9. SAA-C03 Perspective

- **"Ledger / immutable / tamper-evident audit trail"** → **QLDB**
- **"Cryptographically verifiable change history (hash-chained journal)"** → **QLDB**
- **"Credit/debit, financial, insurance, HR records  history matters"** → **QLDB**
- **"Export ledger to S3 and query with Athena/Glue"** → the classic SAA architecture combo
- **"Multi-party blockchain/consensus"** → NOT QLDB  **Amazon Managed Blockchain**
- **"General DB, key-value, serverless, CRUD"** → DynamoDB/RDS, not QLDB

Exam traps: "QLDB is a blockchain with consensus" → **no, centralized single-owner ledger (no consensus) blockchain = Managed Blockchain** "QLDB overwrites data" → **no, append-only immutable journal** "QLDB for high-write CRUD" → **no, ledger semantics with throughput limits** "no way to verify integrity" → **digests + VerifyRevision/GetDigest are core features**.