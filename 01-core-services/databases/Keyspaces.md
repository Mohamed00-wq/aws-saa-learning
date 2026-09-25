# Amazon Keyspaces Course  Serverless Apache Cassandra–Compatible Database

## 1. Purpose

Amazon Keyspaces (for Apache Cassandra) is a **scalable, serverless, fully managed Apache Cassandra–compatible database**  you use the same **Cassandra Query Language (CQL)**, drivers, and tools you already use, just pointed at the AWS endpoint. Tables auto-scale, are replicated **3× across 3 AZs**, and are fully managed. For SAA it's the answer to **"Cassandra / wide-column database / CQL / serverless keyspace"**.

## 2. How it works

- **Keyspace = the database** (namespace of tables) **Multi-Region keyspaces** replicate cross-Region with LOCAL_QUORUM writes
- **Capacity modes per table**:
  - **On-demand (default)**  pay per request scales instantly to any previously-reached traffic level (fully serverless)
  - **Provisioned**  you set read/write capacity units with Auto Scaling (cheaper for predictable load)
- Capacity is metered like DynamoDB: **RRUs** (read request units) and **WRUs** (write request units)  sized by data + consistency level
- **Consistency**  writes always `LOCAL_QUORUM` (durable) reads at `LOCAL_ONE` or `LOCAL_QUORUM` (stronger costs more RRUs)
- **Fully managed TTL**  column/row expiry without tombstone/compaction management (Keyspaces deletes expired data automatically)
- **Backups**  continuous **point-in-time recovery up to 35 days** + on-demand snapshots, no performance impact
- Just change the Cassandra **hostname/endpoint** to the Keyspaces endpoint  existing Cassandra 2.x drivers (Java, Python, Ruby, .NET, Node, PHP, C++, Perl) & tools work

```
Cassandra app (CQL) → change endpoint → Keyspaces (serverless, replicate ×3 across AZs)
  tables with on-demand or provisioned capacity TTL auto-cleanup PITR 35 days
  99.99% SLA in-Region multi-Region keyspaces for global replication
```

## 3. When to use

- **Migrating existing Cassandra workloads** to AWS with minimal code change (CQL + drivers)
- **Wide-column data** with partition-key access patterns: IoT/sensor data, time-series, telemetry
- **Write-heavy, high-throughput applications** (Cassandra's sweet spot: append-heavy, partition-key reads)
- **Cassandra-esque data with TTL expiry** (metrics, events, device state, personalization)
- **Serverless-to-operate Cassandra**  no nodes, no patching, no tombstone tuning

## 4. When NOT to use

- **Key-value serverless at huge scale / DynamoDB-native features** (DAX, global tables LWW) → **DynamoDB** is usually the simpler AWS-native choice for new apps
- **Relational data with joins / SQL / ACID** → **RDS/Aurora**
- **Rich document/aggregation queries** → DocumentDB/MongoDB
- **Need full Cassandra feature set** (some advanced Cassandra APIs/ecosystem features aren't supported) → self-managed Cassandra (e.g., on EC2) or EKS
- **Analytics / columnar DW queries** → Redshift/Athena

## 5. Important features

- **CQL compatibility**  DDL/DML, SELECT/LWT (lightweight transactions), `ALLOW FILTERING` limits, counters, UDTs (partial)
- **On-demand (default) vs provisioned capacity** with table-level auto scaling burst capacity
- **Fully managed TTL**  no tombstone wrangling cheap deletion via TTL
- **~99.99% availability SLA in-Region data replicated 3× across AZs**
- **PITR (≤ 35 days) + on-demand snapshots**, encrypted at rest by default
- **Multi-Region keyspaces** for global applications
- **CloudWatch metrics** IAM access control + VPC endpoints price reduction (up to 75% across several dimensions, making on-demand the recommended default)

## 6. Limitations

- **Not 100% Cassandra**  CQL subset not every Cassandra feature/ecosystem tool is supported
- **Single Region writes default** (multi-Region=keyspace replication eventually-consistent cross-Region reads unless configured)
- Capacity planning still matters in **provisioned** mode (sizing, hot partitions)
- **No joins / relational queries**  data must be modeled for partition-key access
- In-Region **99.99% SLA** (multi-Region adds more moving parts)

## 7. Trade-offs

- **Keyspaces vs DynamoDB**  Cassandra CQL/driver-compatibility & wide-column style vs DynamoDB's richer AWS-native feature set (DAX, global tables, Streams ecosystem). New serverless key-value apps usually pick **DynamoDB** migrate/keep Cassandra → **Keyspaces**
- **On-demand vs provisioned**  variable/spiky/no-planning (pay per request) vs predictable load (cheaper), like DynamoDB
- **Keyspaces vs self-managed Cassandra on EC2**  zero-ops, serverless, SLA vs full control/custom config + ops burden
- **vs DocumentDB**  wide-column partition-key vs document/aggregation model

## 8. Architecture

```
IoT/telemetry pipeline:
  Devices → Kinesis → Lambda → Keyspaces (wide-column, partition key = device_id,
  clustering = timestamp, TTL 30 days on rows)
  Dashboards read via LOCAL_ONE (cheap) for hot device state provisioning on-demand
  Backup: PITR 35 days multi-Region keyspaces replicate to DR Region
```

## 9. SAA-C03 Perspective

- **"Apache Cassandra compatible / CQL / wide-column, fully managed"** → **Keyspaces**
- **"Migrate Cassandra to AWS serverless with same drivers"** → **Keyspaces** (hostname change only)
- **"In-Region 99.99% SLA, data ×3 across AZs, TTL auto-purge"** → Keyspaces
- **"Cassandra / migrate a Cassandra app to AWS fully managed"** → **Keyspaces**
- Compare: **Cassandra/migrate = Keyspaces**, **key-value/serverless = DynamoDB**, **document/MongoDB = DocumentDB**

Exam traps: "Keyspaces = DynamoDB" → **no, Cassandra-compatible** "you manage nodes/compaction/tombstones" → **no, fully managed TTL** "strongly consistent default" → **writes LOCAL_QUORUM, reads default LOCAL_ONE (choose QUORUM)** "unlimited joins" → **no, partition-key model, no joins**.