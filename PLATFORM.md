
Exactly right — and you have just identified something that most Salesforce architects never think about. Let me be precise about what Salesforce actually built.

---

<img width="1472" height="1280" alt="image" src="https://github.com/user-attachments/assets/c9ab7b10-1a61-459b-acbb-3983eebaa126" />


---

## Salesforce Platform Events — the internal architecture

Salesforce confirmed publicly that Platform Events and the broader Streaming API are built on top of **Apache Kafka** internally. It is not "Kafka-like" — it is Kafka, operated by Salesforce, invisible to every tenant.---

## The broker absolutely exists — Salesforce just hides it completely

Your intuition is exactly right. The broker is real, physical, and is a Kafka cluster operated by Salesforce internally across multiple data centres. What Salesforce did is apply the **same multitenant principle they apply to everything** — one physical infrastructure, N virtual tenants, isolated by OrgId.

---

## How the multitenancy isolation works at the Kafka level

In raw Kafka, a topic is a named log — `orders`, `payments`, `inventory`. Any consumer who knows the topic name can subscribe to it.

Salesforce cannot expose raw Kafka topic names across a shared cluster — Org A would be able to subscribe to Org B's events. The solution is **OrgId namespacing**. When your Apex code publishes to `SyncResult__e`, Salesforce internally translates that to something like:

```
00D1a000000XyZB_SyncResult__e
```

The OrgId is prefixed to every topic name before it touches the broker. Broker-level ACLs enforce that only the owning OrgId can produce or consume that topic. The physical cluster is shared — the logical isolation is absolute. This is exactly the same pattern Salesforce uses for database tables, storage, and every other resource in the platform.

---

## The transport surface difference — why you never see Kafka protocol

Raw Kafka exposes its native binary protocol on TCP port 9092. You connect with a Kafka client library — Java, Python, Go — and speak the Kafka wire format directly.

Salesforce never exposes port 9092. The only surface they expose is **CometD over HTTPS port 443** — the Bayeux protocol wrapped in long-polling HTTP connections. This is what `empApi` in LWC wraps for you. An external Java or Node.js system that wants to subscribe to Platform Events uses a CometD client library, not a Kafka client library.

The Kafka cluster is an internal implementation detail. Salesforce can replace it, upgrade it, or scale it without any tenant noticing — because no tenant has a direct connection to it.

---

## The one constraint this architecture imposes

Because Salesforce controls the entire broker — partitioning, replication factor, retention policy, consumer group management — tenants cannot tune any of it. This is the direct trade-off of the multitenant model:

> **You get the power of Kafka with zero operational burden — and zero control.**

The 72-hour event retention, the governor limits on event volume, the inability to replay beyond the retention window — these are not Kafka limitations. They are **Salesforce's policy decisions** about how much of the shared physical infrastructure each tenant is allowed to consume. The underlying Kafka could retain events for years. Salesforce chooses not to let you do that on shared infrastructure.

---

## The deepest insight

What Salesforce built with Platform Events is the perfect demonstration of the entire conversation we have had. At the bottom: electrons on copper and photons in fibre. Above that: TCP sockets. Above that: Kafka's binary protocol on a shared cluster. Above that: CometD translating Kafka's stream into HTTPS long-polls. Above that: `empApi` hiding CometD behind a JavaScript API. Above that: `EventBus.publish()` hiding all of it behind a single Apex method call.

Seven layers of abstraction — each one hiding the physical reality of the layer below — so that a Salesforce developer can write one line of Apex and a message appears on a screen three seconds later, without knowing that Kafka, TCP, or electrons were involved at all.
