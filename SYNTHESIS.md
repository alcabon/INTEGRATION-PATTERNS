
<img width="1472" height="1480" alt="image" src="https://github.com/user-attachments/assets/6d3cac14-aa17-4771-861c-6e72cf965460" />


Here is the complete reading of the diagram from top to bottom.

---

## The chain, in order

**UX layer — block the problem at source.** Inline edit is disabled via Field-Level Security (read-only on all user profiles, edit kept only for the integration profile). The user can only save through full Edit mode — one deliberate action, one transaction.

**Apex Trigger — thin, no logic.** Fires `after update`. Delegates immediately to the `TriggerHandler` class which compares `Trigger.oldMap` vs `Trigger.newMap` on the watched field list only.

**Pending_Sync__c — the coalescing queue (Transactional Outbox).** The handler does not call `System.enqueueJob()` directly. It does a `Database.upsert` keyed on `ExternalId__c`. If the user saves three times in five seconds, three upserts hit the same row — only the latest payload survives. This is the serialisation point that eliminates the race condition structurally. It also increments `Sync_Sequence__c` in the same transaction.

**SyncDispatcher Queueable — the Polling Publisher.** Reads `Pending_Sync__c` rows where `Status__c = 'Pending'`, locks them with `FOR UPDATE`, marks them `Processing`, then fires one HTTP `PATCH` callout per row via Named Credential (OAuth JWT, no secrets in code). The idempotency key in the request header is a SHA-256 of `salesforceId + lastModifiedDate` — the same event always produces the same key, making retries safe.

**External DB — Optimistic Concurrency Control.** The conditional `WHERE seq_db < seq_in` is the final guard. Even if two payloads somehow arrive out of order, the database itself rejects the stale one. Returns `200 OK` on apply, `409 Conflict` on discard.

**Error path — Dead Letter Channel.** Any `5xx` or timeout writes a `Sync_Error_Log__c` record. A scheduled job or retry flow re-enqueues failed records up to three times with exponential backoff. After that, the record stays flagged for human intervention.

**Platform Events — Pub-Subscribe Channel.** The trigger publishes `SyncStarted__e` immediately (same transaction as the upsert). The Queueable publishes `SyncResult__e` after the callout. An LWC on the record page subscribes via `empApi` and shows a toast in both cases. The user has real-time visibility without polling.

---

## Summary table

| Element | Role | Pattern |
|---|---|---|
| FLS read-only | Prevents rapid repeated saves | UX constraint |
| `after update` Apex Trigger | Entry point, delegates only | — |
| `TriggerHandler` | Detects watched field changes | — |
| `Pending_Sync__c` | Staging table, upsert by ExternalId | Transactional Outbox |
| `Sync_Sequence__c` | Monotonic version per record | Optimistic Concurrency Control |
| `SyncDispatcher` Queueable | Reads and dispatches pending rows | Polling Publisher |
| Named Credential | Credentials outside code | Externalized Configuration |
| `Idempotency-Key` header | Safe retries | Idempotent Receiver |
| External DB `WHERE seq_db < seq_in` | Rejects stale out-of-order writes | OCC / Last Write Wins |
| `Sync_Error_Log__c` + retry | Captures and replays failures | Dead Letter Channel |
| `SyncStarted__e` / `SyncResult__e` | Real-time user feedback | Pub-Subscribe Channel |
| LWC `empApi` subscriber | Toast notifications on record page | Event-Driven Consumer |
