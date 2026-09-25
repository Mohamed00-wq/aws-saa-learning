# Amazon MSK Course  Fully Managed Apache Kafka Streaming

## 1. Purpose

Amazon Managed Streaming for Apache Kafka (**Amazon MSK**) is the **fully managed Apache Kafka service**  you run production Kafka without operating brokers/ZooKeeper it's 100% Apache Kafka API-compatible so existing producers/consumers/tools keep working. Choose **MSK Provisioned** (brokers you size: Standard or Express) or **MSK Serverless** (auto-scales, capacity-less). For SAA it's the answer to **"Apache Kafka / streaming data platform / event streaming / CDC/backbone for streaming pipelines"**.

## 2. How it works

- **Broker clusters in your VPC**  MSK provisions, patches, replaces failed brokers, auto-scales storage, manages metadata (they've moved all clusters to **KRaft** ZooKeeper managed/minimal)
- **Native Apache Kafka APIs**  produce/consume standard topics/partitions, consumer groups, exactly-once, offsets **no app code changes**
- **Broker types (Provisioned)**  **Standard** (flexible sizing/durability) and **Express** (elastic, virtually unlimited storage, up to **3× throughput per broker**, scales ~20× faster, ~90% faster recovery for stateless loads)
- **Storage**  managed EBS-backed storage with **auto-expansion** + **tiered storage** (move older data to S3 while keeping it queryable), i.e., "infinite" retention options
- **MSK Serverless**  no capacity planning cluster scales (compute+storage+partitions) automatically pay per provisioned/virtualized partition-hour + data good for spiky/unpredictable streaming
- **MSK Connect**  fully managed **Kafka Connect** workers, bring-your-own connectors (S3, Postgres, Debezium, etc.)
- **MSK Replicator**  effort-less cross-Region/cross-account replication for DR/multi-region streaming
- **Security**  IAM auth, SASL/SCRAM, mutual TLS, Kafka ACLs **encryption at rest (KMS)** + TLS in transit private/VPC connectivity incl. multi-VPC
- **Integrations**  **AWS Glue Schema Registry**, **Lambda** (also on serverless), **Amazon Managed Service for Apache Flink**, S3, Redshift, OpenSearch streaming-loads
- Monitoring  CloudWatch (DEFAULT/PER_BROKER/PER_TOPIC…) + **Open Monitoring with Prometheus** (also used for Cruise Control rebalances)

```
Producers (apps/connectors/CDC loggers) → MSK cluster (brokers in your VPC, KRaft)
  ├─ Consumers (Lambda / Flink / custom) read topics with Kafka APIs
  ├─ MSK Connect: managed Connector workers (S3, JDBC…) 
  ├─ Replicator: cross-Region copy tiered storage → S3 for long retention
  └─ Serverless option: auto-scaling, pay-per-usage
```

## 3. When to use

- **You need Apache Kafka's model**: topics/partitions/consumer-groups, replay + long retention, ordering per partition, multi-consumer fan-out, stream processing
- **Event streaming backbone / data-in-motion architecture**  the "air traffic control" for events across the enterprise
- **Change data capture (CDC)** via Debezium (DBs → Kafka), log/telemetry streaming to data lakes
- **Migrate existing Kafka** (self-managed or Confluent) to managed AWS **without code changes**
- **Large-scale, multi-consumer, replay-able streams** where SQS/SNS queues fall short

## 4. When NOT to use

- **Simple decoupling of microservices**  → **SQS** (simpler, serverless, FIFO available) or **SNS** (pub/sub)
- **Short-lived messages, no replay/retention** needed → SQS
- **Very small/low-traffic workloads** where a Kafka cluster is overkill → SQS/Kinesis depending on needs
- **You specifically avoid the Kafka ecosystem/Java-ish clients** and want AWS-native streams → **Amazon Kinesis Data Streams**
- **Teams without Kafka expertise** just use SQS/SNS to reduce complexity

## 5. Important features

- **100% Apache Kafka API compatibility**  standard clients, tools, connectors unchanged
- **Serverless** deployment  fully auto-scaling cluster pay-per-use **or Provisioned** for control (Standard/Express brokers)
- **Express brokers**  elastic storage, 3× throughput/broker, faster scaling & recovery
- **Tiered storage** (broker-hot → S3) + storage auto-expansion  long retention economics
- **MSK Connect** (managed Kafka Connect), **MSK Replicator** (multi-Region/account replication)
- **Security**  IAM / SASL-SCRAM / mTLS + Kafka ACLs KMS at rest TLS in transit VPC & multi-VPC private access (PrivateLink-based)
- **KRaft metadata mode** (no ZooKeeper burden) managed version upgrades **Glue Schema Registry** integration
- **Monitoring**  CloudWatch + **Prometheus (Open Monitoring)**, consumer-lag metrics Lambda/Managed Flink/S3/Redshift integrations

## 6. Limitations

- **Provisioned clusters cost & sizing**  brokers are instances you pay for partition/broker sizing needs planning (Express reduces it)
- **Not a simple message queue**  Kafka's ordering/retention model is different consumer management is on you
- **Stream processing engines** (Flink, etc.) add separate managed services/complexity
- Networking: brokers live in VPC access across VPCs/accounts needs private connectivity setup (supported)
- Kafka knowledge required (topics, partitions, consumer groups, exactly-once settings)

## 7. Trade-offs

- **MSK vs SQS/SNS**  streaming platform with replay/retention/consumer-groups/multi-consumer (MSK) vs serverless queues/topics with single-pull Semantics, FIFO, simpler (SQS/SNS)  **usually the default for new decoupling**
- **MSK vs Kinesis Data Streams**  Kafka-compatible ecosystem & migration path (MSK) vs AWS-native shard model (Kinesis)
- **MSK Provisioned vs Serverless**  control/sizing (cheaper at steady high load) vs auto-scale pay-per-use (spiky/low)
- **Standard vs Express brokers**  configurable/durable vs elastic 3× throughput/fast-scaling
- **vs self-managed Kafka / Confluent**  fully managed cluster ops + AWS integrations vs full control (the migration guide: move with MM2/replicator)

## 8. Architecture

```
Streaming backbone:
  Apps / IoT / CDC (Debezium) → MSK Provisioned (Express brokers, tiered storage→S3)
  ├─ consumers: Lambda (event triggers), Managed Flink (processing), S3/Redshift/OpenSearch (sinks via MSK Connect)
  ├─ MSK Replicator → DR Region cluster (failover read/write)
  └─ Glue Schema Registry validates Avro/Protobuf/JSON
Security: IAM/mTLS auth, KMS at rest, TLS, private VPC access
```

## 9. SAA-C03 Perspective

- **"Apache Kafka / Kafka-compatible / streaming platform, fully managed"** → **Amazon MSK**
- **"Migrate existing Kafka to AWS without code changes"** → **MSK**
- **"Streaming events with replay/retention/multiple consumers"** → **MSK** (vs SQS for simple queues)
- **"Elastic, no-capacity-planning Kafka"** → **MSK Serverless** "throughput + elastic storage + recovery speed" → **Express brokers**
- **"Cross-Region Kafka replication for DR"** → **MSK Replicator** "managed Kafka Connect" → **MSK Connect**
- **"Simple decoupling / FIFO queue"** → **SQS** "pub/sub events to many subscribers" → **SNS**

Exam traps: "MSK is a managed SQS-like queue" → **no, Apache Kafka** "you run ZooKeeper yourself" → **no, fully managed (KRaft now)** "MSK = serverless only" → **two modes: Provisioned & Serverless** "no replay/retention" → **Kafka is built for long retention/replay** (esp. with tiered storage).