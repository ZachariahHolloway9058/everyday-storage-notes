# Auditable Gaming Password Reset Email API and SMTP Relay Boundaries

A gaming password reset email has two clocks: the reset credential expires, while an email API or SMTP relay may still be retrying. The transport choice has to preserve that distinction and leave an auditable record without storing the credential itself.

Short answer: start a beginner Node.js application with an HTTP email API when the team wants the smaller operational surface and can accept a provider-specific adapter. Choose an SMTP relay when an existing mail abstraction, infrastructure policy, or portability requirement outweighs the weaker request-level feedback. Neither transport proves inbox delivery. Put a durable job, an idempotency key, an expiry deadline, and a provider-neutral event ledger around either one.

For a gaming service that must send a compliance notice alongside an account-recovery flow, reliability means reconstructing what the application intended, what the transport accepted, and what later delivery events reported. A successful API response or SMTP acceptance is one piece of evidence, not the whole audit story.

## Should a password reset email use an API or SMTP relay?

The most expensive mistake is collapsing several states into `sent`. There is a durable business event, a queued delivery attempt, transport acceptance, and a later outcome such as delivery, delay, bounce, or complaint. Those states arrive on different timelines and may disagree temporarily.

Be precise.

Acceptance is narrow.

An HTTP API can return a structured request identifier immediately. An SMTP conversation can return a successful completion after the relay accepts responsibility for the message. In both cases, a downstream server can still defer or reject delivery later. The application therefore needs its own immutable message identifier and an append-only sequence of observations; a mutable `delivered = true` column loses the evidence needed to explain retries and late events.

SPF does not repair this state model. RFC 7208 defines authorization for hosts using a domain in SMTP identity, and it also warns that SPF alone cannot authenticate the message content. Treat domain authentication as a deliverability and abuse-control concern, separate from proof that a particular player received a particular notice.

For reset mail, store the recovery request ID, account ID, template version, recipient-address hash, creation time, expiry time, attempt number, transport message ID, and normalized outcome. Do not put the raw reset token, reset URL, or rendered message body in routine logs. NIST SP 800-63B treats authenticator-related material as sensitive; the audit trail can identify the transaction without becoming a second credential store.

## Derive the transport from the failure boundary

The useful comparison starts after asking who will own retries and how much ambiguity the application can tolerate. A thin synchronous call from the password-reset handler to either transport is fragile: a timeout can occur after remote acceptance, leaving the caller unsure whether another attempt will duplicate the message. The handler should commit the recovery request and an outbox record atomically, then let a worker submit the message.

| Decision boundary | HTTP email API | SMTP relay | Consequence for the app |
|---|---|---|---|
| Submission result | Usually structured and easy to correlate | Protocol reply must be mapped into local states | Keep a local message ID in both cases |
| Ambiguous timeout | Request may have reached the service | Relay may have accepted before the connection failed | Retry only through an idempotent job policy |
| Portability | Adapter often exposes provider-specific fields | Widely implemented protocol boundary | Normalize only the events the audit model needs |
| Later outcomes | Commonly delivered by signed callbacks or polling | Commonly represented by delivery-status mail or relay-specific events | Ingest asynchronously and retain raw evidence separately |
| Operational burden | HTTP client, credentials, callback verification | Connection security, credentials, reply parsing, and pool behavior | Pick the burden the team can test |

The API path is usually easier for a small application because JSON request validation, bounded HTTP timeouts, and correlation IDs fit ordinary web-service tooling. That is a developer-experience judgment, not a claim of greater durability. Its limitation is coupling: provider-specific request fields and event shapes can make a later move expensive, so it is unsuitable when relay portability is the binding requirement. An SMTP relay remains reasonable when the organization already operates and observes that boundary, or when the application must swap relays without rewriting message submission. Its trade-off is operational surface; it is a poor choice for a beginner team that cannot test connection security, ambiguous replies, and delivery-status processing. Protocol portability can still be overstated because bounce classification, metadata propagation, and event feeds vary outside the basic submission exchange.

Cost belongs in the worksheet, but not at the top. Compare retained event volume, callback processing, support requirements, and engineering ownership before comparing per-message charges. A cheap submission path that cannot explain an expired compliance notice has the wrong total cost model.

That boundary matters more than syntax.

## Make retries expire with the recovery request

A queue retry policy must know the business deadline. Retrying a reset message after its token expires produces a perfectly delivered dead end, while retrying a compliance notice may still be required under a different retention and escalation policy. Those are separate job classes even if they share a transport adapter.

The following Python sketch shows the boundary; a Node.js implementation can preserve the same contract. The important detail is that the caller supplies stable identity and expiry, while the adapter returns evidence rather than declaring delivery.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Protocol

@dataclass(frozen=True)
class MailJob:
    message_id: str
    recipient: str
    template_version: str
    expires_at: datetime

@dataclass(frozen=True)
class SubmissionEvidence:
    transport_id: str
    accepted_at: datetime

class MailTransport(Protocol):
    def submit(self, job: MailJob) -> SubmissionEvidence: ...

def submit_if_live(job: MailJob, transport: MailTransport) -> SubmissionEvidence:
    now = datetime.now(timezone.utc)
    if now >= job.expires_at:
        raise ValueError("recovery message expired before submission")
    return transport.submit(job)
```

The worker records `attempt_started` before the network call and `transport_accepted` afterward. If the call times out, it records `outcome_unknown`; it does not manufacture a failure. A retry then uses the same message ID and checks existing evidence before submitting. Where a transport supports idempotency, pass that stable ID through. Where it does not, the system must accept a bounded duplicate risk or add reconciliation before retrying ambiguous attempts.

Unknown means unknown.

Keep the ledger append-only. Corrections should be new events, and access to recipient mappings should be narrower than access to aggregate delivery metrics. For the gaming compliance notice, retain the template version and policy identifier so an auditor can establish what category of notice was requested without exposing personalized recovery content.

## Test the ugly paths, not the happy response

A transport evaluation should use failure injection. Cut the connection after submission bytes leave the process but before the response arrives. Deliver the same callback twice. Deliver a bounce before an acceptance event is visible to the reporting worker. Rotate credentials while jobs remain queued. Then advance the clock beyond token expiry and confirm that the worker records expiration rather than sending.

Four measures are more informative than a single success rate: queue age at submission, time from request to transport acceptance, time to terminal delivery observation, and the count of unknown outcomes awaiting reconciliation. Break them down by message class and domain only where the resulting labels do not expose recipient data.

Alerts also need distinct meanings. Growing queue age points to local capacity or transport availability. A rise in hard failures may indicate bad addresses or authentication trouble. Missing callbacks can make reporting stale while mail continues to move. Combining these into one red light prevents useful diagnosis.

Runbook ownership decides more than the protocol. Someone must know how to pause recovery sends without suppressing required compliance notices, replay verified events, rotate signing secrets, and reconcile an accepted message whose later outcome never arrived. If a beginner team cannot name that owner, the smaller API integration is the safer initial scope, provided the adapter and event ledger keep migration possible.

## Roll out without betting the audit trail

Start with shadow recording: create outbox and ledger entries while the existing sender remains authoritative. Next, route a small internal cohort through the new adapter, compare acceptance and terminal-event coverage, and test rollback by switching only the transport binding. Do not dual-send to real recipients as a migration check; duplicates make both support and evidence harder to interpret.

Promote traffic by message class. Recovery mail has a hard expiry and user-visible latency, while compliance notices may have different retry and retention rules. Keep those policies outside the transport adapter so changing from API submission to SMTP relay, or the reverse, does not rewrite the audit semantics.

The decision rule is compact: use the HTTP API when operational simplicity and structured correlation are the binding constraints; use SMTP relay when an already-operated protocol boundary and portability are stronger constraints. In either case, durable intent, deadline-aware retries, and independently recorded outcomes are what make delivery defensible.

## Sources

References:

- https://datatracker.ietf.org/doc/html/rfc7208
- https://pages.nist.gov/800-63-3/sp800-63b.html
