---
title: "Apache Kafka Fundamentals for Beginners"
description: "Understanding Kafka's core concepts — topics, partitions, producers, and consumers — with simple examples from real-time billing systems."
date: 2026-04-05
tags: ["Kafka", "Distributed Systems", "Java"]
cover: "images/kafka-cover.png"
---

## What is Kafka?

Apache Kafka is a distributed event streaming platform. Think of it as a highly scalable, fault-tolerant message queue that can handle millions of events per second.

In the billing domain at Amdocs, we use Kafka to process real-time usage events — every call, every data session gets streamed through Kafka topics.

## Core Concepts

### Topics and Partitions

A **topic** is a named stream of records. Each topic is split into **partitions** for parallelism:

```
Topic: billing-events
  ├── Partition 0: [event1, event4, event7, ...]
  ├── Partition 1: [event2, event5, event8, ...]
  └── Partition 2: [event3, event6, event9, ...]
```

### Producers

Producers publish messages to topics. A simple Java producer:

```java
Properties props = new Properties();
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

KafkaProducer<String, String> producer = new KafkaProducer<>(props);

producer.send(new ProducerRecord<>("billing-events", "user-123", eventJson));
```

### Consumers

Consumers read messages from topics, organized into **consumer groups** for load balancing:

```java
@KafkaListener(topics = "billing-events", groupId = "billing-processor")
public void processEvent(String event) {
    BillingEvent billingEvent = objectMapper.readValue(event, BillingEvent.class);
    billingService.process(billingEvent);
}
```

## When to Use Kafka

- **Event-driven architectures** — decouple services
- **Real-time data pipelines** — stream processing
- **Log aggregation** — centralize logs from multiple services
- **Activity tracking** — user behavior, audit trails

## Tips from Production

1. **Choose partition keys wisely** — they determine ordering and distribution
2. **Monitor consumer lag** — it tells you if consumers are falling behind
3. **Set appropriate retention** — don't store forever unless you need to
4. **Use schemas** — Avro or JSON Schema with Confluent Schema Registry

Kafka has a steep learning curve, but once you understand the fundamentals, it becomes an indispensable tool in your distributed systems toolkit.
