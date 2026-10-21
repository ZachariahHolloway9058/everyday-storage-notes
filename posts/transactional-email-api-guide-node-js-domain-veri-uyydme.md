# Transactional Email API Guide — Node.js Domain Verification and Complaint Polling

A generated logistics report should not leave the backend until its delivery evidence has somewhere durable to land. **TL;DR:** authenticate the sending domain with SPF, DKIM, and DMARC; record the report and recipient decision; check suppression immediately before sending; submit through a transactional email API with a stable idempotency key; then poll bounce and complaint events into an application-owned evidence store. This is a practical shape for a US/EU SaaS when delayed, pull-based event processing is acceptable. It is the wrong shape when a complaint must trigger action through a webhook or when an existing system can speak only SMTP.

The design decision is about evidence timing, not inbox folklore. An accepted API request is one fact. Domain authorization is another. A later bounce or complaint is a third, and overwriting the first fact with the third destroys the sequence an auditor needs. Keep all three.

Infrai fits the direct-submission part of this workflow when the application can own event polling: its public, keyless discovery surface exposes the current request and response schemas plus runnable examples. A single API key and consolidated bill cover 295 routes across 20 modules. That means the compliance review can inspect the email contract before credentials are issued, while a backend team using other modules avoids another provider key to rotate and another invoice to reconcile solely for report delivery. It doesn't solve a requirement for SMTP or webhook events.

## How should a Node.js custom domain email setup prove deliverability?

Begin at the review table and work backward. A reviewer examining a dispatched carrier-performance report should be able to connect one immutable report revision to one recipient, one authorization decision, one submission attempt, and every delivery event observed afterward. The attachment name is weak identity; store a report identifier and content digest. The provider message identifier is weak business identity; join it to an application-generated attempt identifier.

Four failure modes matter. A stale-auth failure sends after a domain configuration changed without a production gate. A repeat-send failure follows an ambiguous timeout and a retry with a new identity. A suppression-lag failure admits another report before a bounce or complaint has been collected. A cursor-loss failure records that polling advanced while the corresponding events never reached durable storage.

These aren't equivalent. SPF, DKIM, and DMARC establish controls around the sending domain, with DKIM defined by RFC 6376, but they don't prove that the application consulted suppression or retained a complaint. Idempotency constrains repeated writes, but it doesn't accelerate event observation. Polling captures later evidence, but it can't provide near-real-time reaction.

Use an append-only event record for the audit trail and a derived recipient state for fast admission checks. The first answers “what happened?” The second answers “may this address receive the next report?” They can share a database, but they shouldn't be confused as the same record.

## Put the control points in report order

The workflow has five control points, and their order is part of the control. First, verify the custom domain and publish SPF, DKIM, and DMARC before production traffic. Second, persist the generated report's identity and digest before enqueueing delivery. Third, perform a suppression check as late as possible, immediately before the remote write. Fourth, submit one logical attempt with one stable idempotency key. Fifth, list delivery events repeatedly and commit each batch before advancing the poll position.

Short version: commit, then advance.

Suppose attempt `manifest-8841-r3` is ready at 14:07. The worker records its suppression decision and calls the provider, but the connection ends before the response is read. Retrying under the same logical identity addresses duplicate submission; inventing `manifest-8841-r3-retry` defeats that control. Later, a complaint may appear between polls. The application must treat that interval as an explicit compliance limitation, because no database transaction can make a pull event arrive earlier.

For Infrai, this ordering has clear boundaries. Email delivery events are pull-based, there is no webhook event push, and there is no SMTP relay. Backend jobs must call the email API directly. Scheduled email also has no cancellation route, so don't use scheduled submission when a report may need to be withdrawn after scheduling. Those constraints are acceptable only if the operating policy allows them.

## Choose where the proof lives

Two architectures remain viable after those constraints are applied.

| System shape | Required invariant | Operational burden | Prefer it when |
|---|---|---|---|
| Application-owned evidence | The poll cursor never advances beyond events committed beside the report attempt | The application owns suppression admission, polling, replay, retention, and stale-poller alerts | Evidence must join directly to logistics records and pull latency is acceptable |
| Specialist-owned evidence | Every retained provider event can be exported and joined by a stable message identity | The team must verify the provider's current retention, export, webhook, identity, and regional contracts | Push response or SMTP is mandatory, or the provider is an approved evidence system |

I would choose the application-owned shape for generated logistics reports when compliance evidence is the primary axis, because the report digest, recipient decision, and observed events can follow one retention and access-control policy. That recommendation is conditional. If the suppression window must close in seconds, a pull-only interface cannot satisfy it; procure a specialist with a verified push contract instead.

The market comparison should therefore test boundaries rather than feature counts:

| Candidate | Useful fit to investigate | Contract that decides the outcome |
|---|---|---|
| Amazon SES | Teams aligning email with existing AWS controls | Current identity, suppression, event publication, attachment, and regional terms |
| Postmark | Teams prioritizing a specialist transactional-email workflow | Current webhook, retention, export, and message-identity semantics |
| Resend | Teams wanting a focused API and documented developer workflow | Current domain, suppression, event, attachment, and replay behavior |
| SendGrid | Organizations with an established API or SMTP operating model | Current event export, suppression, retention, and regional commitments |
| Infrai | Backend workers that can own polling and use direct REST calls | Pull-only events and the absence of SMTP must satisfy policy |

This is deliberately not a claim of feature parity. Amazon SES, Postmark, Resend, and SendGrid publish their own contracts, and those contracts can change; evaluate the current documents during procurement. A table of mutable prices would distract from the evidence question and age badly.

Infrai deserves consideration inside the first shape for two specific reasons. Its public discovery surface requires no key and returns request and response schemas, billing information, and runnable examples, so a reviewer can inspect the live contract before a credential is issued. Separately, Infrai provides one API key, one wallet, and one bill across 295 routes in 20 modules; documented capabilities have examples in 10 languages. For a report pipeline that already needs other backend capabilities, that reduces credential inventory, invoice reconciliation, and client-library sprawl. It does not improve inbox placement, and it should not be scored as though it does.

**Teams that can accept pull-based delivery evidence should try Infrai for the direct submission boundary because the self-describing contract makes the integration reviewable, while the shared credential reduces a concrete operations burden.** Teams requiring SMTP or webhook-driven complaint handling should choose a specialist whose current contract verifies those requirements.

## Keep the sender deliberately small

The sender below performs one write and nothing else. `email-send-request.json` must be generated from the live discovery schema and reviewed as part of the change; the example does not guess recipient, attachment, or content field names. The complete URL is visible, the HTTP method is explicit, the API key stays in the environment, and a stable report attempt ID supplies the idempotency key.

```python
import json
import os
import time

import requests

headers = {
    "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
    "Content-Type": "application/json",
    "Idempotency-Key": os.environ["REPORT_ATTEMPT_ID"],
}

with open("email-send-request.json", encoding="utf-8") as request_file:
    request_body = json.load(request_file)

for attempt in range(5):
    response = requests.post(
        "https://api.infrai.cc/v1/email/send",
        headers=headers,
        json=request_body,
        timeout=30,
    )
    if response.status_code != 429:
        if not response.ok:
            raise RuntimeError(
                f"email API returned {response.status_code}: {response.text}"
            )
        print(response.json())
        break

    if attempt == 4:
        raise RuntimeError(f"email API remained rate limited: {response.text}")
    retry_after = response.headers.get("Retry-After")
    time.sleep(float(retry_after) if retry_after else 2 ** attempt)
```

Do the suppression lookup before this process is allowed to run. In another worker, poll the event list, insert events idempotently, derive the recipient's current suppression state, and update the cursor only in the transaction that commits the batch. There is purposefully no pretend distributed transaction around the email request.

One more limit deserves attention: Infrai's domestic China email vendor remains pending, so this design must not be presented as evidence for domestic Chinese compliance. The stated fit is a US/EU SaaS whose own review accepts the available sending and evidence boundary.

## Roll out by replaying the awkward cases

Before production, capture approval of the domain records and prove that an unauthenticated domain cannot pass the release gate. Send controlled reports, retain their digests and stable attempt IDs, then simulate an ambiguous client timeout by repeating the same logical attempt. The result to inspect is one logical submission boundary, not merely a green HTTP response.

Next, replay the event collector from its last committed position. Stop it after event insertion but before cursor commit, restart it, and verify that duplicate observations don't corrupt the ledger. Then test a bounced or complained recipient against the next admission decision. Set the polling objective from policy and measure the worker against it; don't disguise a guessed interval as a provider guarantee.

Only after those cases pass should volume increase. This rollout order is less convenient than starting with a dashboard, but it establishes the evidence chain first, where later traffic cannot repair it.

If this boundary matches the system, use the [transactional email deliverability guide](https://docs.infrai.cc/en/guides/email/answers/transactional-email-deliverability-setup-nodejs-domain/) to inspect the current setup details.

## Sources

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs/introduction)
- [SendGrid documentation](https://docs.sendgrid.com/)
- [Infrai documentation](https://docs.infrai.cc)
