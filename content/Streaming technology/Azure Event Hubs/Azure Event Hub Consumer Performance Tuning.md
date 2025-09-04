---
longform:
  format: single
  title: Azure Event Hub Consumer
title: Azure Event Hub Consumer
---
# 🧪 Kafka Consumer Performance Test — High Throughput Configuration

# Configure Auth - kafka_jaas.conf

```
KafkaClient {

org.apache.kafka.common.security.plain.PlainLoginModule required

username="$ConnectionString"

password="Endpoint=sb://demo-kafka.servicebus.windows.net/;SharedAccessKeyName=admin;SharedAccessKey=test=;EntityPath=adttohis";

};
```

# Export Creds

```
export KAFKA_OPTS="-Djava.security.auth.login.config=/Users/deepaknishad/jmeter-kafka/kafka_jaas.conf"
```

```bash
/opt/homebrew/Cellar/kafka/4.0.0/libexec/bin/kafka-consumer-perf-test.sh \
  --bootstrap-server demo-kafka.servicebus.windows.net:9093 \
  --topic adttohis \
  --messages 10000 \
  --group testcg \
  --from-latest \
  --timeout 130000 \
  --print-metrics \
  --show-detailed-stats \
  --consumer.config consumer.properties
```

---

## 🧵 Breakdown of Each Option

|**Option**|**Description**|**Why It's Important for High Throughput**|
|---|---|---|
|`--bootstrap-server demo-kafka.servicebus.windows.net:9093`|Kafka broker (Azure Event Hubs endpoint)|Required to connect to Kafka|
|`--topic adttohis`|The topic from which messages are consumed|Determines source of data load|
|`--messages 10000`|Number of messages to consume|A larger number provides better test for sustained throughput|
|`--group testcg`|Consumer group ID|Isolates offsets for this test; avoids offset conflict|
|`--from-latest`|Start from the latest offset|Only measures performance of **new messages**, not backlog|
|`--timeout 130000`|Max time (ms) to wait for consuming|Prevents indefinite wait if throughput is low|
|`--print-metrics`|Show summary metrics after test|Displays throughput, records/sec, latency|
|`--show-detailed-stats`|Includes latency percentiles and other stats|Helps understand fetch/request patterns|
|`--consumer.config consumer.properties`|External config file for advanced tuning|Enables tuning for high performance|

---

## ⚙️ consumer.properties Explained (High Throughput Focus)

```properties
# Connection
bootstrap.servers=demo-kafka.servicebus.windows.net:9093
security.protocol=SASL_SSL
sasl.mechanism=PLAIN
```

🔐 **Security & Connection to Azure Event Hubs**

---

### 🚀 **Performance Optimization**

```properties
#fetch.min.bytes=65536
fetch.max.wait.ms=550
#max.partition.fetch.bytes=65536
max.poll.records=3000
session.timeout.ms=10000
heartbeat.interval.ms=3000
```

|**Property**|**What It Does**|**High Throughput Role**|
|---|---|---|
|`fetch.min.bytes=65536` _(commented)_|Broker waits for at least 64KB data before responding|Improves batching efficiency|
|`fetch.max.wait.ms=550`|Max time broker waits before sending fetch response|Balances latency vs batch size|
|`max.partition.fetch.bytes=65536` _(commented)_|Max data per partition per fetch|Should be increased for high-volume partitions|
|`max.poll.records=3000`|Max records returned per poll|Larger poll size = fewer requests, better throughput|
|`session.timeout.ms=10000`|Time before broker marks consumer as dead|Avoids unnecessary rebalances|
|`heartbeat.interval.ms=3000`|Interval between heartbeats|Should be < ⅓ of `session.timeout.ms`|

---

### ⏱️ **Timeouts**

```properties
request.timeout.ms=30000
metadata.max.age.ms=30000
```

|**Property**|**Purpose**|**Why It Matters**|
|---|---|---|
|`request.timeout.ms`|Max time to wait for broker responses|Prevents long stalls|
|`metadata.max.age.ms`|How often metadata is refreshed|Ensures quick adaptation to cluster changes|

---

### 🔁 **Retry Behavior**

```properties
reconnect.backoff.ms=100
reconnect.backoff.max.ms=1000
retry.backoff.ms=100
```

|**Property**|**Purpose**|**Tuning Insight**|
|---|---|---|
|`reconnect.backoff.ms`|Time before reconnect attempt|Keeps retry loops under control|
|`reconnect.backoff.max.ms`|Max wait between reconnects|Prevents overloading the broker|
|`retry.backoff.ms`|Time before retrying failed requests|Useful in cloud where transient errors occur|

---

# Final consumer.properties


```bash
# Connection

bootstrap.servers=demo-kafka.servicebus.windows.net:9093

security.protocol=SASL_SSL

sasl.mechanism=PLAIN

# Performance Optimization

#fetch.min.bytes=65536

fetch.max.wait.ms=550

#max.partition.fetch.bytes=65536

max.poll.records=3000

session.timeout.ms=10000

heartbeat.interval.ms=3000

# Timeouts

request.timeout.ms=30000

metadata.max.age.ms=30000

# Retry

reconnect.backoff.ms=100

reconnect.backoff.max.ms=1000

retry.backoff.ms=100

```