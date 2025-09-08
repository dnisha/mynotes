---
longform:
  format: single
  title: Azure Event Hub Metrics Monitoring
title: Azure Event Hub Metrics Monitoring
---

### 1. Error Metrics (The "Red Alert" Metrics)

These are your top-priority metrics. Any non-zero value here typically requires immediate investigation.

| Metric | What it Measures | Kafka/Event Hubs Concept | How to Monitor for Production Issues |
| :--- | :--- | :--- | :--- |
| **Server Errors** | Internal errors from the Event Hubs service (HTTP 5xx). | Broker failures, internal errors on the Event Hubs nodes. | **Alert on:** **Any value > 0**. This indicates a platform issue. You should have a **critical alert** that triggers for any Server Error. Contact Azure support if this is sustained. |
| **User Errors** | Errors caused by invalid client requests (e.g., malformed request, unauthorized). | Producer/consumer misconfiguration, invalid API calls, auth failures (SASL/SSL issues). | **Alert on:** **Any value > 0**. This almost always indicates a bug or misconfiguration in your client applications. Log the specific error from the client side for debugging. |
| **QuotaExceededErrors** | A subset of User Errors where the request was valid but exceeded the allocated quota. | **This is throttling.** The client is being rate-limited by the broker. | **Alert on:** **Any value > 0**. This is your primary scaling signal. It means your current Throughput Units (TUs) or Processing Units (PUs) are insufficient. |
| **Throttled Requests** | The *number of requests* that were throttled due to exceeding the throughput unit limit. | The manifestation of `QuotaExceededErrors` at the request level. | **Alert on:** **A sustained value > 0 over 5 minutes.** This confirms that throttling is happening. Use this to correlate with client-side performance degradation. |

**Production Monitoring Strategy:**
*   **Create a high-severity alert** for `Server Errors`, `User Errors`, and `QuotaExceededErrors` that triggers on any occurrence (`> 0`) over a 5-minute window. Send this to your on-call system (PagerDuty).
*   **Create a medium-severity alert** for `Throttled Requests` that triggers if the total over 5 minutes exceeds a threshold (e.g., > 100). This is a warning to scale up before it becomes critical.

---

### 2. Throughput & Capacity Metrics (The "Scaling" Metrics)

These metrics tell you about the load on your system and are crucial for right-sizing and scaling.

| Metric | What it Measures | Kafka/Event Hubs Concept | How to Monitor for Production Issues |
| :--- | :--- | :--- | :--- |
| **Incoming Messages** | Number of events sent to the Event Hub. | Producer throughput (messages/second). | **Monitor Trend:** Graph this. Watch for unexpected dips (producer failure) or spikes (load testing/new traffic).<br>**Scale Warning:** If this consistently approaches **1000 msg/sec per TU**, you will start throttling. |
| **Incoming Bytes** | Volume of data sent to the Event Hub. | Producer throughput (bytes/second). | **Most Important Scaling Metric:** The ingress limit is **1 MB/sec per TU**.<br>**Alert on:** `Average value > (0.8 * [TU Limit] MB/sec)` over 5 minutes. (e.g., Alert at 8 MB/s avg if you have 10 TUs). This is a proactive scaling alert. |
| **Incoming Requests** | Number of requests made to the service. | Producer API call rate. High numbers can indicate many small batches. | **Correlation Metric:** Correlate spikes here with `Throttled Requests`. A high number of requests with low `Incoming Bytes` might suggest your producers are not batching efficiently (check `linger.ms` and `batch.size`). |
| **Successful Requests** | Number of requests that succeeded. | Healthy producer/consumer communication. | **Use in Dashboards:** Good for health display. A drop while `Incoming Messages` holds steady can indicate a problem. |
| **Size** | The total size of data stored in the Event Hub. | The total disk usage on the broker for the topic. | **Alert on:** `Value > X GB`. Set a threshold based on your retention policy to avoid unexpectedly high storage costs. Monitor the growth rate. |

**Production Monitoring Strategy:**
*   **Set proactive alerts on `Incoming Bytes`** to warn you when you're at 80% of your purchased capacity.
*   **Enable Auto-Inflate** on your Standard tier namespace to automatically scale TUs when `Incoming Bytes` is high. Your alert then informs you that auto-scale happened.
*   **Graph `Incoming Messages` and `Incoming Bytes`** together on a dashboard to understand your typical traffic pattern.

---

### 3. Consumer & Lag Metrics (The "Backpressure" Metrics)

These are critical for ensuring your consumers are healthy and keeping up with the producers. Failure here means delayed data processing.

| Metric | What it Measures | Kafka/Event Hubs Concept | How to Monitor for Production Issues |
| :--- | :--- | :--- | :--- |
| **Outgoing Messages** | Number of events read from the Event Hub. | Consumer throughput (messages/second). | **Correlation:** Should roughly match `Incoming Messages` over time. A sustained divergence indicates lag is building. |
| **Outgoing Bytes** | Volume of data read from the Event Hub. | Consumer throughput (bytes/second). | **Correlation:** Should roughly match `Incoming Bytes` over time. |
| **Capture Backlog** | Number of events not yet archived to Azure Blob Storage. | Lag for the built-in Event Hubs Capture feature. | **Alert on:** `Value > X`. If using Capture, a growing backlog means the archive process is falling behind, potentially due to slow blob storage or insufficient throughput. |
| **Captured Messages/Bytes** | The rate of messages/bytes being archived. | Throughput of the Capture process. | **Monitor Trend:** Ensure it can keep up with `Incoming` metrics. |

**Production Monitoring Strategy:**
*   **This dashboard is missing the most critical consumer metric: `Consumer Lag`.** You must add it.
    *   **Metric Name:** `Consumer Lag` (or similar, it might be under a different name like `MessageLag`).
    *   **What it is:** The number of messages in a partition that have been published but not yet consumed by a consumer group.
    *   **How to Monitor:** **Alert on this above all else!** Set a critical alert: **`Maximum Consumer Lag across all partitions > 1000`** (or a value acceptable for your use case). A growing lag means your consumers are failing or cannot keep up. This is the number one reason to scale out your consumer applications.

---

### 4. Connection Metrics (The "Client Health" Metrics)

These metrics help you understand the connectivity of your producers and consumers.

| Metric | What it Measures | Kafka/Event Hubs Concept | How to Monitor for Production Issues |
| :--- | :--- | :--- | :--- |
| **Active Connections** | The number of active AMQP or Kafka connections. | Number of connected producers and consumers. | **Alert on:** `Value < [Expected Minimum]`. A sudden drop to zero likely indicates a network connectivity issue between your VNet and Event Hubs or a widespread application failure. |
| **Connections Opened/Closed** | The rate of new connections being established and terminated. | Producer/consumer churn. | **Monitor Trend:** A very high rate can indicate client misconfiguration (e.g., not reusing connections), which adds overhead and can lead to throttling. |

**Production Monitoring Strategy:**
*   **Alert on a sudden drop in `Active Connections`** if you expect a constant number of clients (e.g., a stable pool of consumer service instances).

---

### Summary: Your Production Alerting Strategy

1.  **Critical Alerts (Page Someone):**
    *   `Server Errors > 0`
    *   `User Errors > 0`
    *   `Consumer Lag > [Your Threshold]` **(MUST ADD THIS)**
2.  **Warning Alerts (Investigate ASAP):**
    *   `QuotaExceededErrors > 0`
    *   `Incoming Bytes > (0.8 * TU Limit)`
    *   `Active Connections < [Expected Minimum]`
3.  **Informational Dashboards (Trend Analysis & Scaling):**
    *   Graph `Incoming Bytes` vs `Outgoing Bytes`
    *   Graph `Incoming Messages` vs `Outgoing Messages`
    *   Graph `Throttled Requests` and `Incoming Requests`
    *   Track `Size` to monitor storage growth.