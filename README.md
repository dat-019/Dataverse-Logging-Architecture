# Dataverse-Logging-Architecture
# 🏗️ High-Throughput Dataverse Logging Architecture

![Architecture Diagram](DataverseLogging.jpg)

## 📖 Overview

When building enterprise applications on the Power Platform, logging is non-negotiable. We need to know what’s happening, when it's happening, and where errors are popping up. However, directly routing high-volume logs from thousands of clients into Microsoft Dataverse in real-time is a fast track to API throttling and degraded application performance.

This repository demonstrates a **Queue-Based Load Leveling** architecture pattern. By decoupling the log generation from the log writing, this pattern uses **Power Automate, Azure Storage Queues, and Azure Functions** to protect Dataverse while ensuring no log is ever lost.

---

## 🏗️ The Architecture at a Glance

Instead of forcing Dataverse to swallow a firehose of data, this pattern introduces a **decoupling buffer** to absorb traffic spikes and level the load.

```text
[Clients] ➔ [Power Automate] ➔ [Azure Storage Queue] ➔ [Azure Function] ➔ [Dataverse]
                                           ↓
                                  [Poison Queue (Errors)]
```

---

## ⚙️ Key Components & Workflow

### 1. The Producer: Power Automate Flow
The journey begins at the client or device layer. Whether it's a web app, a mobile device, or a background process, client events are routed through a load balancer into a **Power Automate Flow**.
* **The Workflow:** The flow triggers on a client event, constructs a lightweight JSON payload, and uses the *Direct-to-Queue Premium Connector* to push the message out.
* **The Constraint:** To keep things lightning-fast and cost-effective, the payload is kept under **64KB**.

### 2. The Shock Absorber: Azure Storage Queue
Instead of waiting around for Dataverse to process the log, Power Automate drops the message into an **Azure Storage Queue** and immediately finishes its job. 
* This serves as a **Queue-Based Load Leveling** mechanism. 
* If your application experiences a massive spike in user activity, the queue grows, absorbing the pressure and completely isolating your frontend from your backend data store.

### 3. The Governor: Storage-Triggered Azure Function
This is where the magic happens. An Azure Function sits on the other side of the queue, waiting for messages via a **Queue Trigger**. 
* **Controlled Concurrency:** Instead of processing everything at once, the Azure Function scales its worker instances intentionally. This controls the data transformation and write speed.
* By managing the concurrency, the function acts as a governor, ensuring that downstream calls to Dataverse stay well within safe throughput limits.

### 4. The Destination: Microsoft Dataverse
The Azure Function transforms the raw JSON data and writes the log securely into your targeted **Dataverse Table** (e.g., `ApplicationLogs`). Because the Azure Function is drip-feeding the data at a controlled pace, Dataverse remains highly responsive and protected from API throttling.

---

## 🛡️ Handling Failures: The Poison Queue Safety Net

What happens if a log payload is corrupted or Dataverse becomes temporarily unreachable? In a naive system, that log is lost forever. 

In this architecture, if the Azure Function fails to process a log message a specific number of times (e.g., when the `maxDequeueCount` is reached), the system automatically routes the message to an **Azure Storage Queue - Poison**. This ensures that failed logs are safely quarantined and moved for developer review without blocking the rest of the pipeline.

---

## 🏆 Why This Pattern Wins

| Feature | Direct Writing to Dataverse | Queue-Based Logging Pattern |
| :--- | :--- | :--- |
| **Throttling Risk** | 🔴 High (Spikes can lock API limits) | 🟢 Low (Controlled write throughput) |
| **Client Latency** | 🔴 Slower (Must wait for Dataverse write) | 🟢 Instant (Only waits for Queue drop) |
| **Fault Tolerance** | 🔴 Poor (Failed requests are lost) | 🟢 Excellent (Retries & Poison Queue) |
| **Scalability** | 🔴 Limited by Dataverse API limits | 🟢 Highly scalable cloud architecture |

## 🚀 Getting Started

1. Provision an Azure Storage Account and create two queues: `log-queue` and `log-queue-poison`.
2. Import the Power Automate Solution provided in the `/flows` directory.
3. Deploy the Azure Function from the `/src` directory and configure the `AzureWebJobsStorage` connection string.
4. Ensure your Dataverse environment has the required `ApplicationLogs` table.

---
*Architected for Resilience and Scale on the Power Platform.*
