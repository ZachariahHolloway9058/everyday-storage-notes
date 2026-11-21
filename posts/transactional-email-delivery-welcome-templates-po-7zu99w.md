# Transactional Email Delivery: Welcome Templates, Polling, and Suppression Evidence

Short answer: for a B2B marketplace, send the seller's new-order notice from an asynchronous worker and retain a compact, immutable evidence record, not every rendered message body. The useful record ties an internal order event to a template revision, recipient policy decision, regional route, provider message identifier, and later delivery or suppression events. This is the least complex design that can answer who triggered the notice, what rules applied, and what the delivery system reported without turning email content into a second customer-data archive.

The bill is mostly a retention equation: event count multiplied by bytes per retained record multiplied by retention time and replica count, plus query and retrieval charges. A 2 KB evidence record retained beside a 120 KB rendered MIME message is smaller by a factor of 60; that ratio is arithmetic from the example sizes, not a storage benchmark or a promise about any provider. Measure actual payload distributions before choosing a policy. The change that moves the dominant term is refusing to retain rendered bodies by default.

## What should Next.js and Node.js record for transactional email delivery?

Start with the business assertion, not the transport receipt. An SMTP acceptance response says that a downstream system accepted responsibility for a message; it does not prove that a human read it. Likewise, a polling endpoint that reports `delivered` reflects a transport event under that system's definition. Calling either one "customer notification proof" quietly overstates the evidence.

Framework choice does not change that boundary. A Next.js route can commit the marketplace action and enqueue work, while a Node.js worker owns custom template rendering and transport submission; neither process should call an acceptance response proof of delivery. The same evidence model can cover a welcome email sent when a seller account is approved and the transactional email sent when that seller receives a new order, but their purposes, template revisions, and retention rules must remain distinct.

For each seller notice, preserve a stable internal event ID, tenant and seller pseudonymous IDs, order ID, purpose, template revision, locale, destination region, suppression decision, attempt number, provider message ID, timestamps, and normalized outcome. Store a hash of canonical template inputs when later verification matters. Do not store secrets, authentication codes, or an entire order payload merely because the mail worker received them. NIST SP 800-63B is especially relevant when a welcome message also carries an authenticator or recovery flow: authenticator handling deserves a separate threat model from routine marketplace copy.

A compact schema makes the boundary visible:

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Literal

Outcome = Literal["queued", "accepted", "delivered", "bounced", "suppressed"]

@dataclass(frozen=True)
class DeliveryEvidence:
    event_id: str
    tenant_id: str
    seller_id: str
    order_id: str
    template_revision: str
    locale: str
    processing_region: Literal["us", "eu"]
    suppression_checked_at: datetime
    provider_message_id: str | None
    outcome: Outcome
    observed_at: datetime
    input_digest: str
```

`processing_region` is evidence about the route selected by your application. It is not, by itself, proof of every place a processor stored or transferred data. That distinction belongs in the data-processing assessment and contract review.

## Retention changes the architecture

Duplicates are evidence failures.

The transactional path should be retryable without sending twice merely because a web request timed out. Commit the order and an outbox event in one database transaction, let a regional worker claim the event by its stable ID, check the current suppression state, render the pinned template revision, submit the message, then append observations to the evidence ledger. The web handler can return after committing business state; it should not wait for inbox delivery.

Polling is appropriate when the transport exposes status by identifier and callbacks are unavailable or need reconciliation, but aggressive polling creates write amplification and can still miss a later event after the polling window closes. Use bounded backoff, persist the last observed state, and treat terminal labels as provider observations rather than universal truth. A callback consumer needs the same idempotency rule: deduplicate on the source event identifier and retain the raw source identifier even after normalizing its status. Consider the full failure chain: the worker submits a message, loses the response, and retries while the first attempt is already moving downstream; later, a callback arrives before the retry's polling record exists, and a stale suppression-list replica still says the address is eligible. A single mutable `status` field cannot explain that sequence. Separate attempts from observations, attach both to one business event, record which suppression snapshot each attempt consulted, and permit observations to arrive out of order. This costs more rows and more careful queries. The deliberate trade-off buys an audit trail that does not erase ambiguity.

| Decision | Evidence value | Storage and operational cost | Failure mode to name |
|---|---|---|---|
| Keep normalized events | Searchable chronology with small records | Index growth and regional replicas | A new source status is mapped incorrectly |
| Keep rendered bodies | Exact historical content | Large sensitive-data footprint | Retention outlives the business purpose |
| Keep template revision plus input digest | Verifies the chosen artifact and inputs | Requires immutable template releases | A mutable template destroys reproducibility |
| Poll delivery state | Reconciles missing callbacks | Repeated reads and scheduler load | The polling horizon ends before a late bounce |
| Apply suppression before rendering | Avoids unnecessary content processing | Requires fresh, shared policy state | A stale replica permits another attempt |

The table exposes an uncomfortable trade-off: exact replay and data minimization pull in opposite directions. For routine order notices, the template artifact, revisioned subject, selected locale, minimal variable digest, and transport events will often answer operational questions without preserving the body. Legal requirements differ by jurisdiction and contract, so counsel and the data owner must set the retention schedule; engineering should make expiration enforceable rather than encode an indefinite default.

## How should suppression and templates interact?

Suppression is a policy decision made before submission, not a cosmetic status applied afterward. A hard bounce, complaint, tenant-level block, or verified address change can alter that decision, but the precise categories and required treatment depend on the mail system and organizational policy. Keep the source reason and effective time so an auditor can distinguish "not attempted by policy" from "attempted and rejected."

The suppression list is shared policy state, not an address dump.

Race conditions matter. If two workers read an unsuppressed address and then both send, a later suppression write cannot repair the duplicate. Claim the outbox record atomically, key attempts by the business event, and make policy checks part of the recorded attempt. Exactly-once delivery across a database and an external mail service is not a realistic assumption; idempotent application state and explicit reconciliation are the defensible tools.

Templates need the same discipline as schema migrations. Assign an immutable revision, review changes, render representative US and EU locales in CI, and test absent optional fields, long seller names, escaped user content, and plain-text output. Never infer what was sent from today's template. It may have changed yesterday.

## Test the evidence, not just the email

A preview that looks correct proves very little about the workflow. Contract tests should verify that order creation emits one stable event, retries reuse its idempotency key, suppression prevents submission, and every transition appends rather than overwrites evidence. Inject a timeout after remote acceptance; the worker must reconcile before blindly resubmitting. Inject a late bounce after the normal polling window and verify that the callback path still updates suppression.

Deployment deserves a staged template revision and a reversible worker release. Observe queue age, attempt counts, suppression decisions, unmatched callbacks, polling exhaustion, and transitions that violate the state machine. Do not put recipient addresses or rendered subjects in metric labels; high-cardinality telemetry is costly, and those fields can disclose customer data. Alerts should point to an event ID that an authorized operator can use to retrieve narrowly scoped evidence.

This also clarifies provider evaluation without turning it into a feature contest. Amazon SES documents sending, identities, monitoring, and deliverability concepts, but an application still owns its order-event semantics and retention policy. Other transports can expose different event vocabularies and retention boundaries. Compare candidates against a written matrix: regional processing commitments, suppression semantics, event authenticity, status retention, export capability, documented limits, and the ability to reconcile by a stable message identifier. Verify those claims in current primary documentation and contracts before selection.

## What do we deliberately stop keeping?

Keep less.

Delete rendered HTML, MIME bodies, transient personalization data, and verbose transport payloads when their short diagnostic window expires. Expire normalized evidence on the approved schedule, including replicas and secondary indexes, while retaining only records that have a documented legal hold. Backups need the same lifecycle reasoning; deletion from the primary database does not erase an unexpired backup.

The cost is real. Without a retained body, an investigation may prove which immutable template and input digest were used but may not reconstruct the exact bytes after rendering dependencies disappear. Preserve deterministic renderers and released template artifacts when byte-level reconstruction is required, or accept that limitation explicitly. **A smaller evidence ledger improves privacy and cost only if its evidentiary limits are written down.**

For a seller's new-order notice, the durable design is therefore a regional outbox, a pre-send suppression decision, immutable template revisions, idempotent attempts, and append-only delivery observations with finite retention. The evidence should support precise claims. It should never pretend that transport acceptance proves attention, or that a region label alone proves data residency.

## Further reading

- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://pages.nist.gov/800-63-3/sp800-63b.html
