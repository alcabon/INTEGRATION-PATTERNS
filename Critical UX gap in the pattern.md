This is the critical UX gap in the pattern — and most teams discover it too late. Here is exactly what happens.The short answer is: **nothing**. The save feels instant, the sync is completely invisible, and failures are silent. Here is the full step-by-step experience — and then what you should build instead.Step through each state with the buttons or pills. Steps 1–6 show what happens with the basic pattern. Steps 7–9 show the production-grade fix.

---

## What you need to add to make the UX acceptable

The fix requires three extra pieces: a `Sync_Status__c` field, two Platform Events, and a small Lightning Web Component.

### Step 1 — two Platform Events (in Setup)

Create `SyncStarted__e` and `SyncResult__e`, each with these fields:

```
SyncStarted__e
  RecordId__c     (Text)
  ObjectName__c   (Text)

SyncResult__e
  RecordId__c     (Text)
  Success__c      (Checkbox)
  ErrorMessage__c (Text)
```

### Step 2 — publish from the trigger handler and Queueable

Add this to `AccountSyncTriggerHandler`, inside the `if (!payloads.isEmpty())` block, before `System.enqueueJob()`:

```apex
// Publish "sync started" immediately — the LWC shows a toast right away
EventBus.publish(new SyncStarted__e(
    RecordId__c   = String.valueOf(payloads[0].salesforceId),
    ObjectName__c = 'Account'
));
```

And at the end of `ExternalSyncQueueable.sendPayload()`, after a successful response:

```apex
// Publish success result — LWC swaps toast to green
EventBus.publish(new SyncResult__e(
    RecordId__c = String.valueOf(payload.salesforceId),
    Success__c  = true
));
```

And in the `catch` blocks inside `execute()`:

```apex
EventBus.publish(new SyncResult__e(
    RecordId__c    = String.valueOf(payload.salesforceId),
    Success__c     = false,
    ErrorMessage__c = errorMessage
));
```

### Step 3 — the Lightning Web Component

```javascript
// accountSyncStatus.js
import { LightningElement, api, wire } from 'lwc';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';
import { subscribe, unsubscribe } from 'lightning/empApi';

export default class AccountSyncStatus extends LightningElement {
    @api recordId;
    _startSub;
    _resultSub;

    connectedCallback() {
        // Subscribe to both channels, filter to this record
        subscribe('/event/SyncStarted__e', -1, (msg) => {
            if (msg.data.payload.RecordId__c !== this.recordId) return;
            this.toast('Syncing to external system…', 'info', 'dismissable');
        }).then(sub => this._startSub = sub);

        subscribe('/event/SyncResult__e', -1, (msg) => {
            const p = msg.data.payload;
            if (p.RecordId__c !== this.recordId) return;

            if (p.Success__c) {
                this.toast('External system updated successfully.', 'success', 'dismissable');
            } else {
                this.toast(
                    `Sync failed — IT has been notified. Your Salesforce data is saved.`,
                    'error',
                    'sticky'   // sticky so user doesn't miss it
                );
            }
        }).then(sub => this._resultSub = sub);
    }

    disconnectedCallback() {
        unsubscribe(this._startSub, () => {});
        unsubscribe(this._resultSub, () => {});
    }

    toast(message, variant, mode) {
        this.dispatchEvent(new ShowToastEvent({ title: '', message, variant, mode }));
    }
}
```

```html
<!-- accountSyncStatus.html -->
<template></template>
<!-- This component is invisible — it only listens and dispatches toasts -->
```

```xml
<!-- accountSyncStatus.js-meta.xml -->
<LightningComponentBundle>
    <targets>
        <target>lightning__RecordPage</target>
    </targets>
    <targetConfigs>
        <targetConfig targets="lightning__RecordPage">
            <property name="recordId" type="String" datasource="$recordId"/>
        </targetConfig>
    </targetConfigs>
</LightningComponentBundle>
```

Drop this component anywhere on the Account Lightning page in App Builder — it is invisible, zero height, and handles everything automatically.

---

## The key principle

The `after update` trigger fires synchronously and its transaction commits before the user sees "Record saved." The Queueable runs seconds later in a separate transaction. Those are **two completely different moments** in time — and the standard Salesforce "Record saved" toast only confirms the first one. Your UX must explicitly bridge that gap, otherwise the user is flying blind on every sync.
