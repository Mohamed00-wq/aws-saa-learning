# Amazon MQ Course — Managed Message Broker (ActiveMQ / RabbitMQ)

## 1. Purpose

Amazon MQ is a **fully managed message broker** that runs **Apache ActiveMQ and RabbitMQ** — the classic **JMS/AMQP/MQTT/OpenWire** messaging protocols — so you can **migrate existing on-prem broker applications to AWS without rewriting code**. If you need the protocols and semantics of a traditional broker (JMS topics, AMQP exchanges, MQTT pub/sub) rather than AWS-native SQS/SNS, that's Amazon MQ. For SAA it's the answer to **"existing message broker / JMS / AMQP / MQTT / migrate to managed AWS / network-of-brokers"**.

## 2. How it works

- **Broker instances you provision** (not serverless) — choose **single-instance** (cheap, dev) or **active/standby** (HA across AZs) with **Amazon EBS storage** in a VPC
- **Two engines** — **ActiveMQ**: JMS, AMQP 0-9-1, MQTT, OpenWire, STOMP; **RabbitMQ** (v3/4 lines): AMQP 0-9-1, MQTT, STOMP; accessible via existing clients/SDKs
- Managed HA, patch/config management, **network-of-brokers** to scale beyond one instance (or a cluster of brokers)
- **ActiveMQ cross-Region data replication (CRDR)** — replicate messages to a standby broker in another Region for DR; **RabbitMQ private networking** (VPC Lattice / RAM sharing / PrivateLink) for cross-VPC access
- **Security** — TLS/mTLS, KMS encryption (SSE supports RabbitMQ), IAM for control plane, managed credentials; **IAM-based authentication option** for ActiveMQ and RabbitMQ (users via IAM)
- **Integrations** — S3/EFS-less: console, CloudWatch metrics, EventBridge events (broker state), **JMS topics & virtual topics** (publish/subscribe) and RabbitMQ with **Prometheus** metrics, logs to CloudWatch Logs
- In-place major upgrades now supported (ActiveMQ 5.19.x, RabbitMQ 3.13 → 4.x), feature banners

```
Existing broker app (JMS/AMQP/MQTT client) → Amazon MQ broker (ActiveMQ or RabbitMQ)
  ├─ single-instance (dev) or active/standby (HA, multi-AZ)
  ├─ network of brokers for scale; ActiveMQ CRDR for DR (cross-Region)
  └─ CloudWatch/Prometheus metrics, KMS/TLS security, VPC access
```

## 3. When to use

- **You already have ActiveMQ / RabbitMQ brokers** on-prem or on EC2 — migrate to AWS **without rewriting applications** (protocol-level drop-in)
- **Need JMS semantics** (Java message service, topics, queues, message selectors) or **AMQP exchanges/bindings**, **MQTT for IoT/pub-sub**
- **Rounding out hybrid/migration scenarios** — lift-and-shift of middleware when you can't change the client stack
- **Pub/sub with full broker features** (JMS topics, RabbitMQ fanout/topic/direct/headers exchanges) rather than SNS's simpler model

## 4. When NOT to use

- **New, cloud-native apps** → prefer **SQS (+ SNS)**: serverless, no broker to provision, battle-tested, cheaper at scale, simpler ops
- **Exactly-once / FIFO ordering semantics** → SQS **FIFO** (Amazon MQ has no FIFO queue type)
- **Bare-minimum pub/sub on AWS-native events** → SNS (no broker management)
- **Event/streaming at Kafka scale** → **Amazon MSK** (Apache Kafka)
- **You never need JMS/AMQP/MQTT and can write to SQS easily** — don't add a broker

## 5. Important features

- **ActiveMQ & RabbitMQ engines** — native JMS; AMQP 0-9-1; MQTT; OpenWire/STOMP (ActiveMQ); RabbitMQ AMQP/MQTT/STOMP; client SDKs unchanged
- **HA** — active/standby brokers (active it also gives replication and quick failover); multi-AZ
- **Network of brokers / RabbitMQ cluster** — scale horizontally
- **ActiveMQ cross-Region data replication (CRDR)** — DR standby; **RabbitMQ private networking** + **VPC Lattice / RAM / PrivateLink** access
- **Security** — TLS/mTLS, KMS at rest (SSE), IAM auth option, managed users/certs, AD/LDAP supported for ActiveMQ
- **Monitoring** — CloudWatch metrics, **RabbitMQ Prometheus** metrics, logs to CloudWatch, EventBridge state events
- **In-place major-version upgrades** (ActiveMQ 5.19, RabbitMQ 3.13→4.2), fix/security patches managed

## 6. Limitations

- **Not serverless** — you provision broker instance sizes (pay per instance), scaling means resizing/re-architecting (network-of-brokers/cluster)
- **Single primary Region for HA pair** — cross-Region DR needs CRDR (ActiveMQ) / manual failover (RabbitMQ)
- **No FIFO/once-only** semantics like SQS FIFO (JMS/AMQP are exactly-once-in-queue concepts differ)
- Broker management still has some tuning (connection pooling, message TTL, sizing)
- Multiple broker engines = more moving parts; quotas on broker size/connections

## 7. Trade-offs

- **Amazon MQ vs SQS/SNS** — protocol/JMS/AMQP/MQTT compatibility & lift-and-shift (Amazon MQ) vs AWS-native serverless, auto-scale, FIFO, cheaper, no provisioning (SQS/SNS). AWS's message: **new apps → SQS/SNS; existing brokers → Amazon MQ**
- **Amazon MQ vs Amazon MSK** — JMS/AMQP broker (MQ) vs Apache Kafka streaming platform (MSK): different event/queue models, client ecosystems, scaling approaches
- **ActiveMQ vs RabbitMQ engine** — JMS/OpenWire/Java-heavy + CRDR (ActiveMQ) vs AMQP-first/exchange routing + Prometheus (RabbitMQ) — pick per existing workload
- **Single-instance vs active/standby** — cheapest/dev vs HA with failover (higher cost)

## 8. Architecture

```
Migrate on-prem JMS middleware:
  Existing JMS/AMQP/MQTT apps → Amazon MQ active/standby broker (VPC)
  ├─ topics for pub/sub + queues for work distribution
  ├─ ActiveMQ CRDR → standby broker in DR Region (RTO < RPO-friendly)
  └─ after migration keep protocol semantics: no app changes
Monitor: CloudWatch + EventBridge; secure: TLS, KMS, IAM, PrivateLink (RabbitMQ)
```

## 9. SAA-C03 Perspective

- **"Existing ActiveMQ/RabbitMQ broker; JMS/AMQP/MQTT client code you can't change"** → **Amazon MQ**
- **"Migrate messaging middleware to AWS without rewriting"** → **Amazon MQ**
- **"New app, fully managed serverless message queue"** → **SQS**, pub/sub → **SNS**
- **"FIFO exactly-once ordering"** → **SQS FIFO** (Amazon MQ has no FIFO)
- **"Apache Kafka streaming"** → **MSK**

Exam traps: "Amazon MQ is serverless like SQS" → **no, you provision broker instances**; "Amazon MQ FIFO queues" → **no, use SQS FIFO**; "MQ is the always-best choice" → **only when you need broker protocols/migration; otherwise SQS/SNS**; "MQ = Kafka" → **no, ActiveMQ/RabbitMQ broker vs MSK's Kafka**.