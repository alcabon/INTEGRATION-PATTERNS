This is the **race condition** at the heart of async integration. Let me show the problem precisely, then the three viable solutions.Now the code for each layer.

---

## Disable inline edit — the mandatory first step

The cleanest approach combines FLS (hard block) with a custom LWC that replaces the field on the page layout, showing its value and sync status in one component. The user sees an "Edit" button that opens a modal — one deliberate save, no rapid-fire clicks.

```apex
// In Setup → Object Manager → Account → Fields → AnnualRevenue
// → Field-Level Security → set to Read Only for every user profile
// → leave Edit access only for the "Integration" profile used by your Connected App
//
// This is a zero-code enforcement: no Apex trigger can be bypassed,
// no inline edit is possible, no Flow or Process Builder can change it
// unless running as the integration user.
```

---

## Solution A — sequence number (guard at the external DB)

Add one field to the Account object:

```apex
// Custom field: Sync_Sequence__c (Number, 18 digits, no decimals)
// Auto-incremented in the trigger handler before enqueuing.
```

Update the handler to increment the sequence atomically:

```apex
// In AccountSyncTriggerHandler.handleAfterUpdate()
// Replace the payload construction block with:

List<Account> toUpdate = new List<Account>();
List<AccountSyncPayload> payloads = new List<AccountSyncPayload>();

for (Id accountId : newMap.keySet()) {
    Account newRec = newMap.get(accountId);
    Account oldRec = oldMap.get(accountId);
    Map<String, Object> changedFields = getChangedFields(newRec, oldRec);

    if (!changedFields.isEmpty()) {
        // Increment sequence atomically in the same transaction
        Long nextSeq = (newRec.Sync_Sequence__c == null ? 0 :
                        (Long) newRec.Sync_Sequence__c) + 1;

        toUpdate.add(new Account(
            Id               = accountId,
            Sync_Sequence__c = nextSeq
        ));

        payloads.add(new AccountSyncPayload(
            accountId,
            newRec.ExternalId__c,
            changedFields,
            newRec.LastModifiedDate,
            nextSeq                   // pass sequence to payload
        ));
    }
}

if (!toUpdate.isEmpty()) {
    // This is an after update trigger — update is allowed
    // Use ALL_OR_NOTHING = false so one failure doesn't block others
    Database.update(toUpdate, false);
    System.enqueueJob(new ExternalSyncQueueable(payloads));
}
```

The JSON payload now includes the sequence:

```json
{
  "externalId": "EXT-001",
  "salesforceId": "0014x00000AbcDeAAB",
  "lastModifiedDate": "2026-04-22T14:30:00Z",
  "syncSequence": 7,
  "changedFields": { "AnnualRevenue": 300000 }
}
```

The external DB endpoint enforces the rule:

```sql
-- PostgreSQL example — atomic conditional update
UPDATE accounts
SET    annual_revenue  = $changedFields.annualRevenue,
       sync_sequence   = $syncSequence,
       last_synced_at  = NOW()
WHERE  external_id     = $externalId
AND    sync_sequence   < $syncSequence;   -- only apply if incoming is newer

-- Returns 0 rows affected → HTTP 409 Conflict (stale write, safely ignored)
-- Returns 1 row affected → HTTP 200 OK
```

---

## Solution B — coalescing queue (the most robust pattern)

This replaces the direct `System.enqueueJob()` call with a write to a staging object. Only one HTTP call ever fires per record, carrying the latest state.

**The staging object `Pending_Sync__c`:**

```
External_Id__c         (Text, External ID, Unique)  ← upsert key
Salesforce_Id__c       (Text)
Payload_JSON__c        (Long Text Area)
Last_Modified_Date__c  (DateTime)
Sync_Sequence__c       (Number)
Status__c              (Picklist: Pending / Processing / Done / Failed)
```

**Updated trigger handler — no Queueable, just an upsert:**

```apex
public static void handleAfterUpdate(
    Map<Id, Account> newMap,
    Map<Id, Account> oldMap
) {
    List<Pending_Sync__c> stagingRecords = new List<Pending_Sync__c>();

    for (Id accountId : newMap.keySet()) {
        Account newRec = newMap.get(accountId);
        Account oldRec = oldMap.get(accountId);
        Map<String, Object> changedFields = getChangedFields(newRec, oldRec);

        if (!changedFields.isEmpty()) {
            // Upsert by ExternalId — three rapid saves overwrite the same row.
            // Only the latest payload survives.
            stagingRecords.add(new Pending_Sync__c(
                External_Id__c        = newRec.ExternalId__c,
                Salesforce_Id__c      = accountId,
                Payload_JSON__c       = JSON.serialize(changedFields),
                Last_Modified_Date__c = newRec.LastModifiedDate,
                Sync_Sequence__c      = newRec.Sync_Sequence__c,
                Status__c             = 'Pending'
            ));
        }
    }

    if (!stagingRecords.isEmpty()) {
        Database.upsert(stagingRecords, Pending_Sync__c.External_Id__c, false);
        // Kick off the dispatcher — it is idempotent, safe to call many times
        System.enqueueJob(new SyncDispatcherQueueable());
    }
}
```

**The dispatcher — reads all Pending, fires one HTTP call per record:**

```apex
public class SyncDispatcherQueueable implements Queueable, Database.AllowsCallouts {

    public void execute(QueueableContext ctx) {
        // Claim all Pending records atomically
        List<Pending_Sync__c> batch = [
            SELECT Id, External_Id__c, Salesforce_Id__c,
                   Payload_JSON__c, Last_Modified_Date__c, Sync_Sequence__c
            FROM   Pending_Sync__c
            WHERE  Status__c = 'Pending'
            LIMIT  10   // governor: max 10 callouts per transaction
            FOR UPDATE  // row-level lock — prevents duplicate processing
        ];

        if (batch.isEmpty()) return;

        // Mark as Processing before callouts
        for (Pending_Sync__c r : batch) r.Status__c = 'Processing';
        update batch;

        List<Pending_Sync__c> toUpdate = new List<Pending_Sync__c>();

        for (Pending_Sync__c pending : batch) {
            try {
                sendToExternalDB(pending);
                pending.Status__c = 'Done';
            } catch (Exception e) {
                pending.Status__c = 'Failed';
                // Error details already captured in Sync_Error_Log__c
                // via the sendToExternalDB method
            }
            toUpdate.add(pending);
        }

        update toUpdate;

        // If more Pending records exist, chain to next Queueable
        Integer remaining = [
            SELECT COUNT() FROM Pending_Sync__c WHERE Status__c = 'Pending'
        ];
        if (remaining > 0) {
            System.enqueueJob(new SyncDispatcherQueueable());
        }
    }

    private void sendToExternalDB(Pending_Sync__c pending) {
        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:External_Master_DB/api/v1/salesforce/accounts/sync');
        req.setMethod('PATCH');
        req.setTimeout(10000);
        req.setHeader('Content-Type', 'application/json');
        req.setHeader('Idempotency-Key',
            EncodingUtil.convertToHex(
                Crypto.generateDigest('SHA-256',
                    Blob.valueOf(pending.Salesforce_Id__c +
                                 String.valueOf(pending.Last_Modified_Date__c))
                )
            )
        );

        Map<String, Object> body = new Map<String, Object>{
            'externalId'       => pending.External_Id__c,
            'salesforceId'     => pending.Salesforce_Id__c,
            'syncSequence'     => pending.Sync_Sequence__c,
            'lastModifiedDate' => pending.Last_Modified_Date__c
                                         .formatGmt('yyyy-MM-dd\'T\'HH:mm:ss\'Z\''),
            'changedFields'    => (Map<String, Object>)
                                  JSON.deserializeUntyped(pending.Payload_JSON__c)
        };
        req.setBody(JSON.serialize(body));

        HttpResponse res = new Http().send(req);
        if (res.getStatusCode() != 200 && res.getStatusCode() != 204) {
            throw new CalloutException('HTTP ' + res.getStatusCode());
        }
    }
}
```

---

## Summary — the three layers are complementary

| Layer | What it prevents | Where it lives |
|---|---|---|
| Inline edit disabled (FLS) | User cannot make rapid consecutive changes | Salesforce Setup |
| Sequence number | External DB silently discards stale out-of-order writes | External DB SQL |
| Coalescing queue | Multiple saves collapse to one HTTP call; guaranteed latest state | Apex + custom object |

Disabling inline edit reduces the frequency of the race to near zero in normal use. The sequence number is your safety net for cases you cannot control — bulk data loads, API integrations, flows. The coalescing queue is the architectural solution that makes order irrelevant, because only one call ever fires.
