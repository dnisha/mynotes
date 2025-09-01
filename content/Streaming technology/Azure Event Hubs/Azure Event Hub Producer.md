---
longform:
  format: single
  title: Producer
title: Azure Event Hub Producer
---

## ✅ 1. **High-Throughput Enhancements**

Here's how you can **optimize your existing `EventHubProducer`** class for **high-throughput** messaging:

### 🔧 Key Changes:

- **Reduce `Task.Delay`** or eliminate it if needed.
- **Increase `BatchSize` and buffer settings.**
- **Use asynchronous batching and less frequent flushes.**
- **Optimize producer config for throughput.**
- **Disable delivery reports if not needed.**
- **Parallelize production better.**

---

### 🔁 Updated `EventHubProducer.cs` (High-Throughput Optimized)


```csharp
using System;
using System.Threading.Tasks;
using Confluent.Kafka;
using System.Text.Json;
using System.Collections.Generic;
using System.Text;

namespace kafkaProject
{
    public class EventHubProducer
    {
        private readonly string _bootstrapServers = "max-pre-prod.servicebus.windows.net:9093";
        private readonly string _saslPassword = "Endpoint=sb://kafkademo.servicebus.windows.net/;SharedAccessKeyName=MyManageSharedAccessKey;SharedAccessKey=test";
        private readonly string _eventHubName = "adttohis";

        private static int _idCounter = 1;

        public async Task ProduceBatchAsync(int count)
        {
            var config = new ProducerConfig
            {
                BootstrapServers = _bootstrapServers,
                SecurityProtocol = SecurityProtocol.SaslSsl,
                SaslMechanism = SaslMechanism.Plain,
                SaslUsername = "$ConnectionString",
                SaslPassword = _saslPassword,

                // High-throughput tuning
                LingerMs = 100,                      // Wait up to 100ms to batch messages
                BatchSize = 100_000,                 // 100 KB batch size
                QueueBufferingMaxMessages = 100_000, // Allow up to 100k messages buffered
                QueueBufferingMaxKbytes = 102400,    // 100 MB total buffer
                CompressionType = CompressionType.Lz4, // Fast compression
                Acks = Acks.Leader,
                EnableIdempotence = false, // Not needed for Event Hubs
                ClientId = "eventhub-high-throughput-producer"
            };

            using var producer = new ProducerBuilder<Null, string>(config).Build();

            var tasks = new List<Task>();
            var successfulCount = 0;
            var failedCount = 0;

            var parallelism = 10; // Use multiple parallel producers
            var batchSize = count / parallelism;

            var produceTasks = new List<Task>();

            for (int i = 0; i < parallelism; i++)
            {
                produceTasks.Add(Task.Run(async () =>
                {
                    for (int j = 0; j < batchSize; j++)
                    {
                        var messageId = Interlocked.Increment(ref _idCounter);
                        var message = new MyMessage
                        {
                            Id = messageId,
                            Content = $"Message {messageId}",
                            Timestamp = DateTime.UtcNow
                        };

                        string jsonMessage = JsonSerializer.Serialize(message);

                        try
                        {
                            var kafkaMessage = new Message<Null, string>
                            {
                                Value = jsonMessage,
                                Headers = new Headers
                                {
                                    new Header("message-type", Encoding.UTF8.GetBytes("application/json"))
                                }
                            };

                            await producer.ProduceAsync(_eventHubName, kafkaMessage)
                                .ContinueWith(deliveryReport =>
                                {
                                    if (deliveryReport.Exception != null)
                                    {
                                        Console.WriteLine($"❌ Message {message.Id} failed: {deliveryReport.Exception.InnerException?.Message}");
                                        Interlocked.Increment(ref failedCount);
                                    }
                                    else
                                    {
                                        // Commented for performance; log only if needed
                                        // Console.WriteLine($"✅ Message {message.Id} delivered");
                                        Interlocked.Increment(ref successfulCount);
                                    }
                                });
                        }
                        catch (Exception ex)
                        {
                            Console.WriteLine($"❌ Exception on message {message.Id}: {ex.Message}");
                            Interlocked.Increment(ref failedCount);
                        }
                    }
                }));
            }

            await Task.WhenAll(produceTasks);

            producer.Flush(TimeSpan.FromSeconds(10));

            Console.WriteLine($"\n📊 Summary: {successfulCount} successful, {failedCount} failed");
        }
    }

    public class MyMessage
    {
        public int Id { get; set; }
        public string Content { get; set; }
        public DateTime Timestamp { get; set; }
    }
}
```

---

## 📄 2. **Kafka Producer Config Documentation**

Here’s a table explaining each config parameter used in your producer:

|**Config Key**|**Value**|**Purpose**|
|---|---|---|
|`BootstrapServers`|Azure Event Hub FQDN|Event Hub Kafka endpoint|
|`SaslUsername`|`$ConnectionString`|Required for Azure Event Hubs Kafka authentication|
|`SaslPassword`|Event Hub connection string|SASL PLAIN password (your full connection string)|
|`SecurityProtocol`|`SaslSsl`|Azure Event Hub requires SASL over SSL|
|`SaslMechanism`|`Plain`|Use PLAIN auth mechanism|
|`LingerMs`|`100`|Wait up to 100ms before sending to allow batching|
|`BatchSize`|`100000`|Max size in bytes for a message batch (~100 KB)|
|`QueueBufferingMaxMessages`|`100000`|Max number of messages buffered in memory|
|`QueueBufferingMaxKbytes`|`102400`|Total memory used to buffer messages (in KB)|
|`CompressionType`|`Lz4`|Use LZ4 compression for speed and size efficiency|
|`Acks`|`Leader`|Wait for leader ack; Event Hub doesn’t support `all`|
|`EnableIdempotence`|`false`|Not needed in Azure Event Hubs context|
|`ClientId`|`"eventhub-high-throughput-producer"`|Useful for tracking clients in logs|

---

## ⚠️ Best Practices for Azure Event Hubs (Kafka)

- **Partitioning**: Use multiple partitions in Event Hub for parallelism.
- **Message Size**: Keep messages under 1 MB.
- **Compression**: LZ4 balances speed and size; avoid `gzip` unless needed.
- **Avoid Flush in Loops**: Only call `Flush()` once after all messages are sent.

---
