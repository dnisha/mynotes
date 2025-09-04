---
longform:
  format: single
  title: Performace Test
title: Azure Event Hub Producer Performance Tuning
---
# Kafka Producer Performance Test Configuration Guide

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

## Command Overview

```bash
/opt/homebrew/Cellar/kafka/4.0.0/libexec/bin/kafka-producer-perf-test.sh \
    --topic adttohis \
    --throughput -1 \
    --num-records 10000000 \
    --record-size 1024 \
    --producer-props \
      bootstrap.servers=demo-kafka.servicebus.windows.net:9093 \
      security.protocol=SASL_SSL \
      sasl.mechanism=PLAIN \
      linger.ms=100 \
      batch.size=1048576 \
      acks=1 \
      buffer.memory=67108864 \
      request.timeout.ms=30000 \
      client.id=perf-test-producer \
    --print-metrics
```

## Flag Specifications

### 1. Basic Test Parameters

#### `--topic adttohis`
- **Purpose**: Specifies the target Kafka topic for message production
- **Description**: The topic where all test messages will be sent
- **Azure Event Hubs Equivalent**: Event Hub name
- **Required**: Yes

#### `--throughput -1`
- **Purpose**: Controls message production rate throttling
- **Description**: 
  - `-1` = Disables throttling (maximum possible throughput)
  - Positive values = Maximum messages per second
- **Default**: Varies (typically limited)
- **Production Consideration**: Use `-1` for capacity testing, set limits for load testing

#### `--num-records 10000000`
- **Purpose**: Defines total number of messages to produce
- **Description**: 10,000,000 messages will be generated and sent
- **Recommended Values**: 
  - 100,000+ for quick tests
  - 1,000,000+ for sustained load testing
  - 10,000,000+ for thorough performance analysis

#### `--record-size 1024`
- **Purpose**: Sets the size of each message payload
- **Description**: 1024 bytes = 1 KB per message
- **Impact on Performance**:
  - Smaller sizes → Higher messages/second, lower MB/second
  - Larger sizes → Lower messages/second, higher MB/second
- **Azure Consideration**: Event Hubs billing based on MB ingested

### 2. Producer Configuration Properties

#### `bootstrap.servers=demo-kafka.servicebus.windows.net:9093`
- **Purpose**: Connection endpoint for Kafka cluster
- **Azure Event Hubs**: Use your namespace endpoint with port 9093
- **Format**: `[namespace].servicebus.windows.net:9093`
- **Required**: Yes

#### `security.protocol=SASL_SSL`
- **Purpose**: Specifies security mechanism
- **Description**: Use SASL authentication over SSL encryption
- **Azure Requirement**: Mandatory for Event Hubs Kafka endpoint
- **Alternatives**: PLAINTEXT (insecure), SSL (encryption only)

#### `sasl.mechanism=PLAIN`
- **Purpose**: Authentication mechanism
- **Description**: Simple username/password authentication
- **Azure Requirement**: Required for Event Hubs
- **Configuration**: 
  - Username: `$ConnectionString`
  - Password: Event Hubs connection string

### 3. Performance Tuning Properties

#### `linger.ms=100`
- **Purpose**: Batch creation waiting time
- **Description**: Maximum time (ms) to wait for additional messages before sending a batch
- **Default**: 0 (send immediately)
- **Impact**: 
  - Higher values → Better batching → Higher throughput
  - Higher values → Increased latency
- **Recommended Range**: 20-100ms for production

#### `batch.size=1048576`
- **Purpose**: Maximum batch size in bytes
- **Description**: 1,048,576 bytes = 1 MB maximum batch size
- **Azure Limit**: 1 MB maximum for Standard tier
- **Default**: 16,384 bytes (16 KB)
- **Optimization**: Set to maximum allowed value for throughput

#### `acks=1`
- **Purpose**: Message acknowledgment requirement
- **Values**:
  - `0` = No acknowledgment (fastest, least reliable)
  - `1` = Leader acknowledgment (balanced)
  - `all` = All replicas acknowledgment (slowest, most reliable)
- **Production Recommendation**: `1` for most use cases

#### `buffer.memory=67108864`
- **Purpose**: Total memory for buffering unsent messages
- **Description**: 67,108,864 bytes = 64 MB buffer memory
- **Default**: 32 MB
- **Calculation**: Should be ≥ (number of producers × batch.size)

#### `request.timeout.ms=30000`
- **Purpose**: Maximum time to wait for request response
- **Description**: 30,000ms = 30 second timeout
- **Default**: 30,000ms
- **Consideration**: Increase for high-latency networks

#### `client.id=perf-test-producer`
- **Purpose**: Client identifier for monitoring
- **Description**: Unique identifier for this producer instance
- **Usage**: Helpful for debugging and monitoring specific clients

### 4. Output Configuration

#### `--print-metrics`
- **Purpose**: Enable detailed metrics output
- **Description**: Prints comprehensive performance metrics after test completion
- **Output Includes**:
  - Messages/second
  - MB/second
  - Average latency
  - Maximum latency
  - Batch statistics
  - Network metrics

## Performance Metrics Interpretation

### Key Output Metrics:
1. **Records/sec**: Message throughput rate
2. **MB/sec**: Data throughput rate (compare against Azure TU limits)
3. **Avg latency**: Average request round-trip time
4. **Max latency**: Worst-case request time
5. **Batch size**: Average messages per batch

### Azure Event Hubs Capacity Planning:
- 1 Throughput Unit (TU) = 1 MB/sec ingress
- Calculate required TUs: `Peak MB/sec ÷ 1`
- Always include 20-30% buffer for traffic spikes

## Recommended Production Values

| Parameter | Development | Production | High-Throughput |
|-----------|-------------|------------|-----------------|
| `linger.ms` | 0-50 | 20-100 | 50-200 |
| `batch.size` | 16384 | 524288 | 1048576 |
| `acks` | 1 | 1 | 1 or 0 |
| `buffer.memory` | 33554432 | 67108864 | 134217728 |
| `request.timeout.ms` | 30000 | 30000 | 60000 |

## Security Considerations

1. **Never hardcode credentials** in commands or scripts
2. Use environment variables or configuration files
3. Regularly rotate SAS keys and passwords
4. Use minimal required permissions (Send only)
5. Enable Azure Monitor for tracking and alerts

## Troubleshooting Common Issues

### Throttling Errors:
- Symptoms: `ServerBusyException` or high throttled requests
- Solution: Increase Throughput Units or optimize batching

### Timeout Errors:
- Symptoms: `RequestTimeoutException`
- Solution: Increase `request.timeout.ms` or check network latency

### Memory Errors:
- Symptoms: `OutOfMemoryException`
- Solution: Increase `buffer.memory` or reduce batch size

## Example Output Analysis

```
1000000 records sent, 125423.200851 records/sec (122.48 MB/sec), 
35.80 ms avg latency, 452.00 ms max latency.
```

- **Throughput**: 125,423 msg/sec → Excellent performance
- **Data Rate**: 122.48 MB/sec → Requires ~123 TUs in Azure
- **Latency**: 35.8ms average → Good for most applications
- **Max Latency**: 452ms → Investigate occasional delays

This configuration provides a comprehensive performance testing setup for Azure Event Hubs with optimal throughput settings while maintaining reasonable reliability guarantees.