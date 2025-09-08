---
longform:
  format: single
  title: Throttle 1 TU eventhub
title: Throttle 1 TU eventhub
---
```
/opt/homebrew/Cellar/kafka/4.0.0/libexec/bin/kafka-producer-perf-test.sh --topic adttohis --throughput 1500 --num-records 1000000000 --record-size 2024 --producer-props bootstrap.servers=kafkaDemo.servicebus.windows.net:9093 security.protocol=SASL_SSL sasl.mechanism=PLAIN batch.size=16384000 acks=1 request.timeout.ms=30000 client.id="throttle-test-producer" --print-metrics
```


```
/opt/homebrew/Cellar/kafka/4.0.0/libexec/bin/kafka-producer-perf-test.sh --topic topichistoadt --throughput 10000 --num-records 100000000 --record-size 10024 --producer-props bootstrap.servers=kafkaDemo.servicebus.windows.net:9093 security.protocol=SASL_SSL sasl.mechanism=PLAIN acks=1 request.timeout.ms=30000 client.id=perf-test-producer --print-metrics
```


```
/opt/homebrew/Cellar/kafka/4.0.0/libexec/bin/kafka-producer-perf-test.sh \
  --topic test \
  --throughput -1 \
  --num-records 100000000 \
  --record-size 1024 \
  --producer-props \
    bootstrap.servers=kafkaDemo.servicebus.windows.net:9093 \
    security.protocol=SASL_SSL \
    sasl.mechanism=PLAIN \
    acks=1 \
    request.timeout.ms=30000 \
    client.id=perf-test-producer \
    batch.size=655360 \
    linger.ms=5 \
    max.in.flight.requests.per.connection=5 \
    buffer.memory=67108864 \
  --print-metrics
  
```