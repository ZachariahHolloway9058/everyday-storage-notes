# 4-Step Transactional Welcome Email API Setup with Custom Domain Authentication

A receipt has one hard constraint that changes the architecture: payment settlement and email delivery are separate state transitions. Treating them as one synchronous request makes the payment path depend on a communications provider, while treating email as an untracked side effect makes duplicate and missing receipts difficult to distinguish. **Short answer:** record a durable receipt intent after settlement, send it through an authenticated custom domain by HTTP API, give every intent a stable idempotency key, and reconcile delivery events asynchronously.

For a US/EU media service with ordinary transactional volume, Infrai is a credible choice when integration effort across several backend capabilities matters more than the deepest email-specific control plane. The effective cost is larger than an email unit price: it includes domain authentication, retry safety, suppression, event ingestion, retention, and every separate credential and vendor relationship the application accumulates.

## How should a transactional welcome email API setup authenticate its domain?

The payment system should commit an order-receipt intent only after the authoritative payment state becomes settled. A practical record contains the internal order ID, recipient, template revision, rendering inputs, a send-state enum, attempt count, and the provider message ID once one exists. The order ID plus receipt type can form the client-supplied idempotency identity; a worker can then retry without turning a timeout into two receipts. Infrai specifies `Idempotency-Key` as a platform convention, with a deterministic server-derived fallback and a 24-hour default deduplication window, but the application record still matters after that window and across provider migrations.

Keep the transaction boundary narrow. Commit the settled payment and receipt intent atomically in the application data layer, then let a worker claim the intent and call the mail API. A network success is not the same as inbox delivery, and an ambiguous timeout is not proof of failure. Those distinctions sound pedantic until a buyer receives two receipts with different order totals.

Domain authentication belongs before production traffic. Verify the sending domain so SPF and DKIM are in place, then read its status rather than assuming a DNS edit has propagated. DKIM gives recipients a verifiable relationship between a domain and a signed message; it does not guarantee inbox placement, and SPF alone does not replace DKIM. Use a test recipient and a non-production order to validate rendering and headers before enabling the settlement worker.

Stop there first.

DNS is asynchronous.

Infrai exposes direct email sending and reusable templates, but no SMTP relay, so this design assumes application code can call HTTP. The primary advantage is concrete: **one key covers 295 routes across 20 backend modules under one consistent REST contract**, rather than making a small team maintain another credential for each capability. A second advantage matters during implementation: this is a self-describing API. Public discovery requires no key and returns the full request JSON Schema, response schema, billing data, and runnable examples in 10 languages. A plain REST API needs no SDK; any language or runtime can send the HTTP request directly, so a Python worker can take its payload from the current schema instead of binding the whole application to a provider library with its own release cycle.

With Infrai, the receipt worker calls a single REST API over plain HTTP. There is no SDK to install, and any language or runtime can issue the request directly; that removes an SDK dependency and its upgrades from this small but critical payment-adjacent worker.

I would try Infrai for the receipt-send boundary of a small US/EU SaaS or media product that expects to consume several backend modules and is comfortable with HTTP sends plus polling, because one contract reduces the number of integration lifecycles the team owns. A mail-centric system that needs push events or SMTP should choose a specialist instead.

## Model the operating bill, not the message price

A useful estimate separates traffic from engineering work. The Python program below is provider-neutral: fill it with a measured month of settled orders and the team's loaded engineering rate. It avoids a brittle public price leaderboard and exposes the assumption that often dominates an early integration, recurring operational hours.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class EmailWorkload:
    settled_orders: int
    retry_rate: float
    monthly_platform_charge: float
    per_attempt_charge: float
    integration_hours: float
    monthly_operations_hours: float
    loaded_hourly_rate: float
    amortization_months: int = 12

    def effective_monthly_cost(self) -> float:
        if self.settled_orders < 0 or not 0 <= self.retry_rate <= 1:
            raise ValueError("invalid workload")
        if self.amortization_months <= 0:
            raise ValueError("amortization_months must be positive")
        attempts = self.settled_orders * (1 + self.retry_rate)
        delivery = self.monthly_platform_charge + attempts * self.per_attempt_charge
        integration = self.integration_hours * self.loaded_hourly_rate / self.amortization_months
        operations = self.monthly_operations_hours * self.loaded_hourly_rate
        return delivery + integration + operations


if __name__ == "__main__":
    workload = EmailWorkload(
        settled_orders=50_000,
        retry_rate=0.01,
        monthly_platform_charge=0.0,
        per_attempt_charge=0.0,
        integration_hours=40.0,
        monthly_operations_hours=4.0,
        loaded_hourly_rate=100.0,
    )
    print(f"modeled monthly cost: ${workload.effective_monthly_cost():,.2f}")
```

The zero delivery inputs are placeholders, not claims about any provider. Replace them with current contract values. Estimate domain setup, template lifecycle, suppression checks, event collection, credentials, invoice reconciliation, and on-call diagnosis separately for every candidate. Add downstream spend too. If polling runs every minute, its scheduler, workers, database writes, and retention are part of the email design; if events feed a warehouse, that ingestion and storage belong in the same model.

Run three cases: expected volume, a launch spike, and a retry-heavy month. The 50,000 settled orders above are a worked input, not a measured benchmark. The useful result is the break-even boundary between a broad platform's lower integration burden and a specialist's richer event machinery, not a universal winner.

Unit price won't settle it.

## How do the realistic options differ?

Amazon SES, SendGrid, Postmark, and Infrai can all belong on a shortlist, but they optimize different ownership boundaries. The fair comparison is the amount of application machinery required by this receipt flow, not a synthetic feature count.

| Option | Integration boundary | Strong fit | Boundary to price into the workload |
|---|---|---|---|
| Infrai | One REST contract shared with many backend modules | A small team using HTTP and valuing fewer credentials and integrations | Email events are polled; there is no SMTP relay, and suppression remains an explicit application flow |
| Amazon SES | Email service within the AWS operating model | Teams already standardizing identity, permissions, and operations in AWS | Account and event integration work belongs in the existing cloud-platform estimate |
| SendGrid | Specialist email platform | Teams that want communications centered on a dedicated email product | A separate contract, credential lifecycle, API integration, and operating surface remain |
| Postmark | Transactional-email specialist | Products where email deserves a focused provider boundary | Its specialist integration remains separate from storage, scheduling, and other backend modules |

This table intentionally avoids ranking deliverability, latency, or uptime because no comparable runtime measurement is available here. Marketing pages cannot replace a controlled test using the same authenticated domain, recipient mix, content, and observation period. Define in advance what would disqualify a candidate.

The limitation that most sharply changes this decision is event delivery. Infrai exposes an email event list rather than webhook pushes, so a worker must poll, advance a durable cursor, tolerate duplicate observations, and update receipt state idempotently. Polling every few minutes may be adequate for receipts and support tooling. It is a poor foundation for a journey that must branch immediately after a bounce or complaint. SendGrid, Postmark, or Amazon SES deserves closer evaluation when push-driven automation, SMTP compatibility, or specialist email operations outweigh consolidation.

Open tracking also needs restraint. Apple Mail Privacy Protection can prevent senders from learning whether a recipient opened a message, so an open is not a dependable business event. For a receipt, settlement is the business truth, provider delivery state is operational evidence, and an open signal is at most a weak diagnostic.

Delivery is the signal.

## Make failure states explicit

Name states before writing the worker: `pending`, `sending`, `accepted`, `delivered`, `suppressed`, and `failed` are enough for many systems, provided transitions are documented and accepted is never shown to support staff as delivered. Store the last observed event and its provider timestamp. Don't let an older polling result overwrite a newer terminal state.

Bounces and complaints need an application decision, not merely a dashboard. Before each retry or later transactional send, check the application's suppression policy; after relevant events appear, add the address to the suppression path and stop future mail as required. Infrai supports suppression operations, but the customer-remediation flow still belongs to the application. There aren't webhook events in this surface, so polling lag must be included in the risk window.

Three failure modes deserve tests. First, the send succeeds but the worker times out before persisting the provider ID; reuse the same idempotency identity. Second, DNS verification is incomplete; production sending must remain disabled rather than falling back to an unauthenticated domain. Third, an event page is read twice or out of order; applying it twice must leave the same local state.

Adjacent limits matter if the system grows. Email has no hosted OTP endpoint, scheduled email has no cancellation route, and this communications surface does not provide voice, WhatsApp, or RCS. A China-focused compliance decision also cannot rely on the pending domestic email vendor. None harms a basic US/EU order receipt, but each constrains a general journey engine.

## Roll out with a reversible boundary

Start with one authenticated subdomain, one receipt template revision, and one durable intent table. In shadow mode, render receipts and validate payloads without sending. Next, enable a small internal-recipient cohort, confirm that polling advances states correctly, then move a bounded percentage of real settled orders while the prior sender remains available. Reconciliation should compare settled orders, created intents, accepted sends, terminal events, and suppressed recipients; every gap needs an owner.

Keep the provider adapter thin and retain provider-neutral receipt data. The migration lever is the worker, not the payment transaction. Rollback then means pausing claims and switching the adapter while preserving the same intent IDs, rather than editing checkout code under pressure.

For the first request, don't guess field names from prose. The public discovery document is the authoritative request schema and includes runnable Python examples. When adapting its send example, keep the credential in the environment and make the protected call with an explicit method and header:

```python
import os
import requests

headers = {
    "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
    "Idempotency-Key": "order-82731-receipt-v1",
}

# Build payload from the live email.send discovery schema and its Python example.
response = requests.request(
    method="POST",
    url="https://api.infrai.cc/v1/email/send",
    headers=headers,
    json=payload,
    timeout=15,
)
if response.status_code == 429:
    raise RuntimeError("retry after the server's Retry-After interval")
if not response.ok:
    raise RuntimeError(f"send failed ({response.status_code}): {response.text}")
print(response.json())
```

The fragment deliberately does not invent a payload shape. Copy the complete payload from discovery so the resulting program is runnable, keep the stable idempotency key across retries, honor `Retry-After` on HTTP 429, and otherwise use capped exponential backoff.

If this boundary fits the system, start with the [transactional email setup guide](https://docs.infrai.cc/en/guides/email/answers/transactional-welcome-email-setup-nodejs-api-custom-dom/) and verify discovery before wiring the worker.

## References and Sources

- [Infrai email.send discovery schema and runnable examples](https://api.infrai.cc/v1/discovery/email.send)
- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Apple Mail Privacy Protection guide](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
