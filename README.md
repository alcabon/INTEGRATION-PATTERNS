# INTEGRATION-PATTERNS

<img width="1472" height="2218" alt="image" src="https://github.com/user-attachments/assets/71809848-f1e2-4546-b140-a2691b479550" />


These are all canonical, named patterns — they come from two foundational books and distributed systems theory. Let me map each technique to its formal name and origin.The patterns are entirely generic — the Salesforce code is just one implementation. Here is the intellectual lineage:

---

## The two books that name everything

**Enterprise Integration Patterns** (Hohpe & Woolf, 2003) is the foundational reference. It catalogues 65 patterns with canonical names, icons, and a common vocabulary. If you are in an architecture discussion and say "Idempotent Receiver" or "Dead Letter Channel," any senior integration engineer anywhere knows exactly what you mean, regardless of the technology stack.

**Microservices Patterns** (Chris Richardson, 2018) extended that vocabulary for the era of distributed services. The Transactional Outbox is its most important contribution — it solves precisely the problem you identified: how do you guarantee that a write to your own DB and a message to an external system are always consistent, even if the process crashes between the two?

---

## The hierarchy of the patterns we used

The three solutions map onto a classic progression in distributed systems design:

**Optimistic Concurrency Control** is the oldest and most primitive defence. It does nothing to prevent the race — it just detects and discards the loser after the fact. It is cheap and stateless at the sender, but it requires the receiver to be smart.

**Transactional Outbox + Polling Publisher** eliminates the race structurally. There is no longer a race because there is only ever one message in flight per record. The outbox table is the serialisation point — upsert is atomic, so the last write always wins before anything is ever sent. This is the correct architectural solution.

**Idempotent Receiver + Dead Letter Channel** are orthogonal to both of the above. They are safety nets for a different class of failure — network failures and retries — not ordering failures. A well-designed system needs all three layers.

---

## One nuance worth knowing

The **Last Write Wins** pattern looks simple but is dangerous in practice. It relies on clock synchronisation between the two systems. If Salesforce's clock and the external DB's clock drift by even a few seconds — which is entirely possible in a cloud multi-region setup — LWW can silently apply a genuinely older write because its timestamp happens to be higher. The sequence number (OCC) avoids this entirely because it is a logical clock, not a wall clock. This distinction is a central theme in Leslie Lamport's 1978 paper *Time, Clocks, and the Ordering of Events in a Distributed System*, which is the academic root of everything in this space.
