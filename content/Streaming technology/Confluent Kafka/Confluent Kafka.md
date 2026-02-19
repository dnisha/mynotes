---
longform:
  format: scenes
  title: Confluent Kafka
  sceneFolder: /
  scenes: []
  ignoredFiles: []
title: Confluent Kafka
---
# Production-Grade Confluent Kafka Self-Managed Setup

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PRODUCTION ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐          │
│  │  KRaft Ctrl  │    │  KRaft Ctrl  │    │  KRaft Ctrl  │          │
│  │   + MDS #1   │    │   + MDS #2   │    │   + MDS #3   │          │
│  │  (Quorum)    │    │  (Quorum)    │    │  (Quorum)    │          │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘          │
│         └───────────────────┼───────────────────┘                  │
│                    KRaft Raft Quorum                                │
│                             │                                       │
│  ┌──────────┐ ┌──────────┐ ┌┴─────────┐ ┌──────────┐              │
│  │ Broker 1 │ │ Broker 2 │ │ Broker 3 │ │ Broker 4 │              │
│  │ (IO/Stor)│ │ (IO/Stor)│ │ (IO/Stor)│ │ (IO/Stor)│              │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘              │
│       └────────────┼────────────┼────────────┘                     │
│                    │                                                │
│  ┌─────────────────┼─────────────────────────────────────┐         │
│  │                 │   DATA PLANE                        │         │
│  │  ┌──────────────┴──────┐  ┌────────────────────────┐ │         │
│  │  │  Schema Registry    │  │   Kafka Connect (x3)   │ │         │
│  │  │  SR-1  │  SR-2      │  │  CPU-intensive workers │ │         │
│  │  └─────────────────────┘  └────────────────────────┘ │         │
│  │                                                        │         │
│  │  ┌──────────────────────┐  ┌────────────────────────┐ │         │
│  │  │  ksqlDB (x2)         │  │   Control Center (x1)  │ │         │
│  │  │  Stream Processing   │  │   Monitoring Isolation │ │         │
│  │  └──────────────────────┘  └────────────────────────┘ │         │
│  └───────────────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Component Summary Table

|#|Component|Count|ZooKeeper|Dedicated|Reason|
|---|---|---|---|---|---|
|1|Kafka Brokers|4|No|Yes|IO heavy, storage intensive|
|2|KRaft Controllers + MDS|3|No|Yes|Quorum stability critical|
|3|Schema Registry|2|No|Yes|HA for schema governance|
|4|Control Center|1|No|Yes|Monitoring isolation|
|5|Kafka Connect|3|No|Yes|CPU intensive connectors|
|6|ksqlDB|2|No|Yes|Stream processing isolation|

---

## Phase 1 — Infrastructure & OS Hardening

### Step 1.1 — Node Sizing (Recommended)

```
Kafka Brokers      : 4 × (32 vCPU | 128GB RAM | 8TB NVMe SSD | 25Gbps NIC)
KRaft Controllers  : 3 × (16 vCPU | 64GB RAM  | 500GB SSD    | 10Gbps NIC)
Schema Registry    : 2 × (8 vCPU  | 16GB RAM  | 100GB SSD    | 10Gbps NIC)
Control Center     : 1 × (8 vCPU  | 32GB RAM  | 500GB SSD    | 10Gbps NIC)
Kafka Connect      : 3 × (32 vCPU | 64GB RAM  | 200GB SSD    | 25Gbps NIC)
ksqlDB             : 2 × (16 vCPU | 64GB RAM  | 500GB SSD    | 10Gbps NIC)
```

### Step 1.2 — OS Configuration (All Nodes)

```bash
# Disable swap — Kafka is heavily memory-reliant
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

# Set kernel parameters
cat >> /etc/sysctl.conf <<EOF
# Network
net.core.rmem_max=134217728
net.core.wmem_max=134217728
net.ipv4.tcp_rmem=4096 65536 134217728
net.ipv4.tcp_wmem=4096 65536 134217728
net.core.netdev_max_backlog=300000
net.ipv4.tcp_congestion_control=bbr

# VM
vm.swappiness=1
vm.dirty_ratio=80
vm.dirty_background_ratio=5
vm.max_map_count=262144

# File descriptors
fs.file-max=1000000
EOF
sysctl -p

# File descriptor limits
cat >> /etc/security/limits.conf <<EOF
kafka  soft  nofile  1000000
kafka  hard  nofile  1000000
kafka  soft  nproc   65536
kafka  hard  nproc   65536
EOF
```

### Step 1.3 — Disk Setup (Broker Nodes)

```bash
# RAID-0 for throughput OR separate mount per disk for Kafka log dirs
mkfs.xfs -f /dev/nvme0n1
mkfs.xfs -f /dev/nvme1n1
mkfs.xfs -f /dev/nvme2n1
mkfs.xfs -f /dev/nvme3n1

mkdir -p /data/kafka/{disk1,disk2,disk3,disk4}
mount -o noatime,nodiratime,nobarrier /dev/nvme0n1 /data/kafka/disk1
mount -o noatime,nodiratime,nobarrier /dev/nvme1n1 /data/kafka/disk2
mount -o noatime,nodiratime,nobarrier /dev/nvme2n1 /data/kafka/disk3
mount -o noatime,nodiratime,nobarrier /dev/nvme3n1 /data/kafka/disk4

# Persist in fstab
echo "/dev/nvme0n1 /data/kafka/disk1 xfs noatime,nodiratime,nobarrier 0 0" >> /etc/fstab
```

---

## Phase 2 — TLS & Security Setup

### Step 2.1 — Generate CA and Certificates

```bash
# Create CA
openssl req -new -x509 -keyout ca-key.pem -out ca-cert.pem \
  -days 3650 -subj "/CN=kafka-ca" -nodes

# For each node (broker, controller, schema-registry, connect, ksqldb, c3)
for NODE in broker1 broker2 broker3 broker4 \
            controller1 controller2 controller3 \
            schema-registry1 schema-registry2 \
            connect1 connect2 connect3 \
            ksqldb1 ksqldb2 control-center; do

  # Generate keystore
  keytool -genkey -noprompt -alias $NODE \
    -dname "CN=$NODE,OU=kafka,O=company,C=US" \
    -keystore $NODE.keystore.jks \
    -storepass changeit -keypass changeit \
    -keyalg RSA -keysize 4096 -validity 3650

  # Generate CSR
  keytool -certreq -alias $NODE \
    -keystore $NODE.keystore.jks \
    -file $NODE.csr \
    -storepass changeit

  # Sign with CA
  openssl x509 -req -CA ca-cert.pem -CAkey ca-key.pem \
    -in $NODE.csr -out $NODE-signed.pem \
    -days 3650 -CAcreateserial

  # Import CA cert into keystore
  keytool -import -alias CARoot -noprompt \
    -keystore $NODE.keystore.jks \
    -file ca-cert.pem -storepass changeit

  # Import signed cert
  keytool -import -alias $NODE -noprompt \
    -keystore $NODE.keystore.jks \
    -file $NODE-signed.pem -storepass changeit

  # Create truststore
  keytool -import -alias CARoot -noprompt \
    -keystore $NODE.truststore.jks \
    -file ca-cert.pem -storepass changeit
done
```

### Step 2.2 — JAAS Configuration (SASL/PLAIN or SCRAM)

```properties
# /etc/kafka/kafka_server_jaas.conf (Broker)
KafkaServer {
  org.apache.kafka.common.security.scram.ScramLoginModule required
  username="kafka-broker"
  password="broker-secret";
};

KafkaClient {
  org.apache.kafka.common.security.scram.ScramLoginModule required
  username="kafka-broker"
  password="broker-secret";
};
```

---

## Phase 3 — KRaft Controllers Setup

### Step 3.1 — Generate Cluster UUID

```bash
KAFKA_CLUSTER_ID=$(kafka-storage random-uuid)
echo $KAFKA_CLUSTER_ID  # Save this — used across ALL nodes
```

### Step 3.2 — Controller Configuration

```properties
# /etc/kafka/kraft/controller.properties (controller1 example)

########################
# Process Roles
########################
process.roles=controller
node.id=1
controller.quorum.voters=1@controller1:9093,2@controller2:9093,3@controller3:9093

########################
# Listeners
########################
listeners=CONTROLLER://controller1:9093
inter.broker.listener.name=CONTROLLER
controller.listener.names=CONTROLLER

########################
# Security
########################
listener.security.protocol.map=CONTROLLER:SSL

ssl.keystore.location=/etc/kafka/ssl/controller1.keystore.jks
ssl.keystore.password=changeit
ssl.key.password=changeit
ssl.truststore.location=/etc/kafka/ssl/controller1.truststore.jks
ssl.truststore.password=changeit
ssl.client.auth=required

########################
# MDS (Metadata Service)
########################
confluent.metadata.server.listeners=https://controller1:8090
confluent.metadata.server.ssl.keystore.location=/etc/kafka/ssl/controller1.keystore.jks
confluent.metadata.server.ssl.keystore.password=changeit
confluent.metadata.server.ssl.truststore.location=/etc/kafka/ssl/controller1.truststore.jks
confluent.metadata.server.ssl.truststore.password=changeit

########################
# KRaft Logs
########################
log.dirs=/data/kraft-controller-logs
metadata.log.dir=/data/kraft-controller-logs
```

### Step 3.3 — Format Controller Storage

```bash
# Run on each controller node
kafka-storage format \
  --config /etc/kafka/kraft/controller.properties \
  --cluster-id $KAFKA_CLUSTER_ID
```

---

## Phase 4 — Kafka Brokers Setup

### Step 4.1 — Broker Configuration

```properties
# /etc/kafka/server.properties (broker1 example — node.id=101)

########################
# Process Roles
########################
process.roles=broker
node.id=101
controller.quorum.voters=1@controller1:9093,2@controller2:9093,3@controller3:9093
controller.listener.names=CONTROLLER

########################
# Listeners
########################
listeners=INTERNAL://broker1:9092,EXTERNAL://broker1:9094,REPLICATION://broker1:9095
advertised.listeners=INTERNAL://broker1:9092,EXTERNAL://broker1:9094,REPLICATION://broker1:9095

listener.security.protocol.map=\
  INTERNAL:SASL_SSL,\
  EXTERNAL:SASL_SSL,\
  REPLICATION:SSL,\
  CONTROLLER:SSL

inter.broker.listener.name=REPLICATION

########################
# Security — SSL
########################
ssl.keystore.location=/etc/kafka/ssl/broker1.keystore.jks
ssl.keystore.password=changeit
ssl.key.password=changeit
ssl.truststore.location=/etc/kafka/ssl/broker1.truststore.jks
ssl.truststore.password=changeit
ssl.client.auth=required
ssl.enabled.protocols=TLSv1.3,TLSv1.2
ssl.cipher.suites=TLS_AES_256_GCM_SHA384,TLS_CHACHA20_POLY1305_SHA256

########################
# Security — SASL
########################
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
sasl.enabled.mechanisms=SCRAM-SHA-512

########################
# Storage — Multi-disk setup
########################
log.dirs=/data/kafka/disk1,/data/kafka/disk2,/data/kafka/disk3,/data/kafka/disk4

########################
# Replication & Durability
########################
default.replication.factor=3
min.insync.replicas=2
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=2

########################
# Performance Tuning
########################
num.network.threads=16
num.io.threads=32
num.partitions=12
socket.send.buffer.bytes=1048576
socket.receive.buffer.bytes=1048576
socket.request.max.bytes=104857600

# Log retention
log.retention.hours=168
log.retention.bytes=1073741824
log.segment.bytes=1073741824
log.cleanup.policy=delete

# Compression
compression.type=lz4

# Replica fetch
num.replica.fetchers=8
replica.fetch.max.bytes=10485760

########################
# JVM Heap (set in env)
########################
# KAFKA_HEAP_OPTS="-Xms32g -Xmx32g -XX:+UseG1GC -XX:MaxGCPauseMillis=20"

########################
# Confluent MDS
########################
confluent.metadata.server.listeners=https://broker1:8090
confluent.metadata.server.advertised.listeners=https://broker1:8090
confluent.metadata.server.ssl.keystore.location=/etc/kafka/ssl/broker1.keystore.jks
confluent.metadata.server.ssl.keystore.password=changeit
confluent.metadata.server.ssl.truststore.location=/etc/kafka/ssl/broker1.truststore.jks
confluent.metadata.server.ssl.truststore.password=changeit

########################
# Confluent RBAC
########################
confluent.authorizer.access.rule.providers=CONFLUENT
confluent.metadata.server.token.max.lifetime.ms=3600000
confluent.metadata.server.authentication.method=BEARER
```

### Step 4.2 — Format Broker Storage & Start

```bash
# Format broker storage
kafka-storage format \
  --config /etc/kafka/server.properties \
  --cluster-id $KAFKA_CLUSTER_ID

# Create SCRAM credentials (run once from any broker)
kafka-configs --bootstrap-server broker1:9092 \
  --alter --add-config \
  'SCRAM-SHA-512=[iterations=8192,password=broker-secret]' \
  --entity-type users --entity-name kafka-broker

# Start broker (systemd)
systemctl start confluent-kafka
systemctl enable confluent-kafka
```

### Step 4.3 — Broker JVM Settings

```bash
# /etc/kafka/kafka-env.sh
export KAFKA_HEAP_OPTS="-Xms32g -Xmx32g"
export KAFKA_JVM_PERFORMANCE_OPTS="\
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=20 \
  -XX:InitiatingHeapOccupancyPercent=35 \
  -XX:+ExplicitGCInvokesConcurrent \
  -XX:G1HeapRegionSize=16m \
  -Djava.awt.headless=true \
  -Dcom.sun.jndi.rmiURLParsing=none"
```

---

## Phase 5 — Schema Registry Setup

### Step 5.1 — Schema Registry Configuration

```properties
# /etc/schema-registry/schema-registry.properties (SR-1)

listeners=https://schema-registry1:8081
host.name=schema-registry1

########################
# Kafka Backend
########################
kafkastore.bootstrap.servers=broker1:9092,broker2:9092,broker3:9092,broker4:9092
kafkastore.topic=_schemas
kafkastore.topic.replication.factor=3
kafkastore.security.protocol=SASL_SSL
kafkastore.ssl.keystore.location=/etc/schema-registry/ssl/schema-registry1.keystore.jks
kafkastore.ssl.keystore.password=changeit
kafkastore.ssl.truststore.location=/etc/schema-registry/ssl/schema-registry1.truststore.jks
kafkastore.ssl.truststore.password=changeit
kafkastore.sasl.mechanism=SCRAM-SHA-512
kafkastore.sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule \
  required username="schema-registry" password="sr-secret";

########################
# SR SSL (client-facing)
########################
ssl.keystore.location=/etc/schema-registry/ssl/schema-registry1.keystore.jks
ssl.keystore.password=changeit
ssl.truststore.location=/etc/schema-registry/ssl/schema-registry1.truststore.jks
ssl.truststore.password=changeit

########################
# HA — Leader Election
########################
# Both SR nodes run; leader elected via Kafka topic
schema.compatibility.level=FULL_TRANSITIVE

########################
# MDS / RBAC
########################
confluent.schema.registry.auth.mechanism=JETTY_AUTH
confluent.metadata.bootstrap.server.urls=https://controller1:8090,https://controller2:8090
confluent.metadata.http.auth.credentials.provider=BASIC
confluent.metadata.basic.auth.user.info=schema-registry:sr-mds-secret
```

---

## Phase 6 — Kafka Connect Setup

### Step 6.1 — Connect Worker Configuration

```properties
# /etc/kafka/connect-distributed.properties (connect1 example)

########################
# Worker Identity
########################
bootstrap.servers=broker1:9092,broker2:9092,broker3:9092,broker4:9092
group.id=connect-cluster-production

########################
# Security
########################
security.protocol=SASL_SSL
ssl.keystore.location=/etc/kafka/ssl/connect1.keystore.jks
ssl.keystore.password=changeit
ssl.truststore.location=/etc/kafka/ssl/connect1.truststore.jks
ssl.truststore.password=changeit
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule \
  required username="connect-worker" password="connect-secret";

# Producer/Consumer security (same as above)
producer.security.protocol=SASL_SSL
producer.sasl.mechanism=SCRAM-SHA-512
consumer.security.protocol=SASL_SSL
consumer.sasl.mechanism=SCRAM-SHA-512

########################
# REST Interface
########################
rest.host.name=connect1
rest.port=8083
rest.advertised.host.name=connect1
rest.advertised.port=8083

########################
# Schema Registry Integration
########################
key.converter=io.confluent.connect.avro.AvroConverter
value.converter=io.confluent.connect.avro.AvroConverter
key.converter.schema.registry.url=https://schema-registry1:8081,https://schema-registry2:8081
value.converter.schema.registry.url=https://schema-registry1:8081,https://schema-registry2:8081
key.converter.schema.registry.ssl.truststore.location=/etc/kafka/ssl/connect1.truststore.jks
value.converter.schema.registry.ssl.truststore.location=/etc/kafka/ssl/connect1.truststore.jks

########################
# Internal Topics (replicated)
########################
config.storage.topic=connect-configs
config.storage.replication.factor=3
offset.storage.topic=connect-offsets
offset.storage.replication.factor=3
offset.storage.partitions=25
status.storage.topic=connect-status
status.storage.replication.factor=3
status.storage.partitions=5

########################
# Performance
########################
task.shutdown.graceful.timeout.ms=10000
offset.flush.interval.ms=10000
connector.client.config.override.policy=All

# Worker threads (tune per connector CPU load)
# num.worker.threads=<CPU count>
```

### Step 6.2 — JVM for Connect (CPU-heavy)

```bash
export KAFKA_HEAP_OPTS="-Xms16g -Xmx16g"
export KAFKA_JVM_PERFORMANCE_OPTS="\
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=100 \
  -XX:+UseStringDeduplication"
```

---

## Phase 7 — ksqlDB Setup

### Step 7.1 — ksqlDB Server Configuration

```properties
# /etc/ksqldb/ksql-server.properties (ksqldb1)

########################
# Server
########################
bootstrap.servers=broker1:9092,broker2:9092,broker3:9092,broker4:9092
ksql.service.id=production-ksqldb-cluster
listeners=https://ksqldb1:8088

########################
# Security
########################
security.protocol=SASL_SSL
ssl.keystore.location=/etc/ksqldb/ssl/ksqldb1.keystore.jks
ssl.keystore.password=changeit
ssl.truststore.location=/etc/ksqldb/ssl/ksqldb1.truststore.jks
ssl.truststore.password=changeit
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule \
  required username="ksqldb" password="ksql-secret";

########################
# Schema Registry
########################
ksql.schema.registry.url=https://schema-registry1:8081,https://schema-registry2:8081
ksql.schema.registry.ssl.truststore.location=/etc/ksqldb/ssl/ksqldb1.truststore.jks
ksql.schema.registry.ssl.truststore.password=changeit

########################
# Processing
########################
ksql.streams.num.stream.threads=8
ksql.streams.cache.max.bytes.buffering=10000000
ksql.streams.commit.interval.ms=2000

########################
# State Store (RocksDB)
########################
ksql.streams.state.dir=/data/ksqldb/state-store

########################
# HA — Both nodes join same service.id
########################
# Both ksqldb1 and ksqldb2 use same ksql.service.id
# Load balanced via external LB or round-robin DNS
```

---

## Phase 8 — Control Center Setup

### Step 8.1 — Control Center Configuration

```properties
# /etc/confluent-control-center/control-center-production.properties

########################
# Kafka Connection
########################
bootstrap.servers=broker1:9092,broker2:9092,broker3:9092,broker4:9092
confluent.controlcenter.id=1

########################
# Security
########################
confluent.controlcenter.streams.security.protocol=SASL_SSL
confluent.controlcenter.streams.ssl.keystore.location=/etc/c3/ssl/control-center.keystore.jks
confluent.controlcenter.streams.ssl.keystore.password=changeit
confluent.controlcenter.streams.ssl.truststore.location=/etc/c3/ssl/control-center.truststore.jks
confluent.controlcenter.streams.ssl.truststore.password=changeit
confluent.controlcenter.streams.sasl.mechanism=SCRAM-SHA-512
confluent.controlcenter.streams.sasl.jaas.config=\
  org.apache.kafka.common.security.scram.ScramLoginModule \
  required username="c3" password="c3-secret";

########################
# REST
########################
confluent.controlcenter.rest.listeners=https://control-center:9021
confluent.controlcenter.rest.ssl.keystore.location=/etc/c3/ssl/control-center.keystore.jks
confluent.controlcenter.rest.ssl.keystore.password=changeit

########################
# Schema Registry
########################
confluent.controlcenter.schema.registry.url=\
  https://schema-registry1:8081,https://schema-registry2:8081

########################
# Connect Integration
########################
confluent.controlcenter.connect.connect-cluster.cluster=\
  https://connect1:8083,https://connect2:8083,https://connect3:8083

########################
# ksqlDB Integration
########################
confluent.controlcenter.ksql.ksqldb-cluster.url=\
  https://ksqldb1:8088,https://ksqldb2:8088

########################
# Internal Topics
########################
confluent.controlcenter.internal.topics.replication=3
confluent.controlcenter.command.topic.replication=3
confluent.monitoring.interceptor.topic.replication=3

########################
# Storage
########################
confluent.controlcenter.data.dir=/data/control-center
```

---

## Phase 9 — RBAC (Role-Based Access Control) Setup

```bash
# Create service accounts via MDS
# Authenticate to MDS
confluent login --url https://controller1:8090 \
  --ca-cert-path /etc/kafka/ssl/ca-cert.pem

# Create users/principals
confluent iam rbac role-binding create \
  --principal User:schema-registry \
  --role ResourceOwner \
  --resource Topic:_schemas \
  --kafka-cluster-id $KAFKA_CLUSTER_ID

confluent iam rbac role-binding create \
  --principal User:connect-worker \
  --role ResourceOwner \
  --resource Topic:connect-configs \
  --kafka-cluster-id $KAFKA_CLUSTER_ID

confluent iam rbac role-binding create \
  --principal User:ksqldb \
  --role ResourceOwner \
  --resource KsqlCluster:production-ksqldb-cluster \
  --kafka-cluster-id $KAFKA_CLUSTER_ID

confluent iam rbac role-binding create \
  --principal User:c3 \
  --role SystemAdmin \
  --kafka-cluster-id $KAFKA_CLUSTER_ID
```

---

## Phase 10 — Topic Governance & Quotas

```bash
# Create critical internal topics with right replication
kafka-topics --bootstrap-server broker1:9092 \
  --create --topic _schemas \
  --partitions 1 --replication-factor 3 \
  --config cleanup.policy=compact \
  --config min.insync.replicas=2

# Producer quotas per client
kafka-configs --bootstrap-server broker1:9092 \
  --alter --add-config 'producer_byte_rate=104857600,consumer_byte_rate=209715200' \
  --entity-type clients --entity-name high-throughput-app

# Topic-level config
kafka-configs --bootstrap-server broker1:9092 \
  --alter --add-config \
  'retention.ms=604800000,min.insync.replicas=2,compression.type=lz4' \
  --entity-type topics --entity-name your-production-topic
```

---

## Phase 11 — Monitoring Setup

### Step 11.1 — JMX + Prometheus JMX Exporter

```yaml
# jmx_exporter_config.yml
lowercaseOutputName: true
rules:
  - pattern: 'kafka.server<type=BrokerTopicMetrics, name=MessagesInPerSec><>OneMinuteRate'
    name: kafka_server_messages_in_per_sec
  - pattern: 'kafka.server<type=ReplicaManager, name=UnderReplicatedPartitions><>Value'
    name: kafka_server_under_replicated_partitions
  - pattern: 'kafka.controller<type=KafkaController, name=ActiveControllerCount><>Value'
    name: kafka_controller_active_count
  - pattern: 'kafka.network<type=RequestMetrics, name=TotalTimeMs, request=Produce><>99thPercentile'
    name: kafka_produce_total_time_ms_p99
```

```bash
# Add JMX exporter to broker startup
export KAFKA_OPTS="-javaagent:/opt/jmx_prometheus_javaagent.jar=7071:/etc/kafka/jmx_exporter_config.yml"
```

### Step 11.2 — Key Alerts (Prometheus Rules)

```yaml
groups:
  - name: kafka-critical
    rules:
      - alert: UnderReplicatedPartitions
        expr: kafka_server_under_replicated_partitions > 0
        for: 1m
        labels:
          severity: critical

      - alert: ActiveControllerCount
        expr: kafka_controller_active_count != 1
        for: 30s
        labels:
          severity: critical

      - alert: OfflinePartitionsCount
        expr: kafka_controller_offline_partitions_count > 0
        for: 1m
        labels:
          severity: critical

      - alert: ConsumerGroupLag
        expr: kafka_consumer_group_lag > 100000
        for: 5m
        labels:
          severity: warning

      - alert: BrokerDiskUsage
        expr: (node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.20
        for: 5m
        labels:
          severity: warning
```

---

## Phase 12 — Startup Order & Validation

### Step 12.1 — Correct Startup Sequence

```
1. KRaft Controllers (all 3, quorum must form)
       ↓
2. Kafka Brokers (all 4, wait for controller quorum)
       ↓
3. Schema Registry (both nodes)
       ↓
4. Kafka Connect (all 3 workers)
       ↓
5. ksqlDB (both nodes)
       ↓
6. Control Center (last — depends on everything)
```

### Step 12.2 — Validation Checklist

```bash
# 1. Verify KRaft quorum
kafka-metadata-quorum --bootstrap-server broker1:9092 describe --status

# 2. Check brokers registered
kafka-broker-api-versions --bootstrap-server broker1:9092

# 3. Verify replication health
kafka-topics --bootstrap-server broker1:9092 --describe | grep "URP\|Leader: none"

# 4. Schema Registry health
curl -k https://schema-registry1:8081/subjects

# 5. Connect cluster check
curl -k https://connect1:8083/connectors

# 6. ksqlDB health
curl -k https://ksqldb1:8088/info

# 7. Control Center
curl -k https://control-center:9021/2.0/check

# 8. End-to-end produce/consume test
kafka-producer-perf-test \
  --topic test-topic \
  --num-records 1000000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props bootstrap.servers=broker1:9092

kafka-consumer-perf-test \
  --bootstrap-server broker1:9092 \
  --topic test-topic \
  --messages 1000000
```

---

## Phase 13 — Operational Runbook

|Scenario|Action|
|---|---|
|Broker rolling restart|Reassign partitions first, restart one broker at a time|
|Controller failover|Automatic via KRaft quorum (no manual action)|
|SR leader failover|Automatic via Kafka-backed leader election|
|Connect worker failure|Tasks auto-rebalanced to remaining 2 workers|
|Disk 80% full on broker|Add disk or expand log.dirs, no restart needed|
|Consumer lag spike|Increase partitions or scale consumer group|
|Schema backward compat break|SR will reject — fix schema before deploy|

---

## Summary Flow

```
OS Hardening → TLS/Certs → KRaft Controllers → Brokers
    → Schema Registry → Kafka Connect → ksqlDB
        → Control Center → RBAC → Topics → Monitoring → Validate
```

This gives you a fully production-grade, secure, RBAC-enabled, highly available Confluent Kafka self-managed cluster with no ZooKeeper dependency, proper quorum, TLS everywhere, and full observability.