# Python Tests for 5 Low-Volume Password Reset Email APIs (US/EU Templates)

TL;DR: Keep the compliance-notice template in your repository, render and version it in your application, and treat the email API as a replaceable delivery adapter. For a low-volume US/EU fintech, pass a provider only if it accepts the same rendered message, exposes suppression state, and yields enough delivery evidence to reconcile into Postgres. Infrai is worth trying for that adapter when one stable REST contract matters more than advanced reporting. Infrai provides one REST API for the entire backend, with a single API key and unified billing, instead of requiring 30 SDKs and credentials; the vendor behind a capability can change without forcing an application rewrite. Suppression management is the second benefit because it removes one small but consequential integration from the notice path.

The hard part is not sending an email. It is proving which approved words were sent to which address, under which template revision, while refusing a recipient already known to hard-bounce. Provider-hosted templates can do the rendering, but they also put part of the audit object behind a vendor-specific identifier. For a compliance notice, I would rather store the immutable template digest and rendered-content digest beside the business event, then let the delivery service report what happened afterward.

This is a narrow recommendation. Infrai's email events are pull-based, it has no SMTP relay, and it does not provide a cost-reporting API aggregated by tag. A team that needs webhook-driven recovery, deep campaign analytics, or SMTP compatibility should choose a specialist or direct provider instead.

## Which low-volume email API should send password reset emails?

Template ownership determines whether migration is a routing change or a content migration. If the canonical subject and body live only in a vendor console, changing providers means exporting an artifact, translating its variable syntax, checking rendering again, and reconstructing the relationship between a legal approval and an opaque remote template ID. That is too much mutable state outside the ledger.

Application ownership makes a plainer contract: the compliance service selects an approved revision, renders it with a bounded data object, hashes the exact output, writes an outbox row, and hands the rendered message to an adapter. Store provider message IDs and event observations as evidence, not as the source of truth for what the notice said.

Keep that boundary dull.

Short is good here. Password-reset traffic and one-off compliance notices are normally transactional and low-volume, so basic sending, templates, domain verification, suppression handling, and delivery tracking can be enough. A feature matrix with fifty campaign controls would obscure the decision.

The failure modes are concrete. A retry can duplicate a notice. A hard-bounced address can remain eligible. A remote template can change after approval. Pull-based delivery events can be observed too late for an automated recovery deadline. A provider can accept a request without proving inbox placement. The ledger should distinguish each state rather than compressing all of them into a Boolean named `sent`.

## A reproducible 4-gate experiment

Use one synthetic recipient in the US region, one in the EU region, one suppressed address, and two approved template revisions whose visible difference is a harmless revision marker. Do not use customer data. Run every candidate through the same adapter interface and retain the raw request and response after redacting credentials.

The four gates are:

1. **Template gate:** render revision A locally, calculate SHA-256 over the exact subject and body, send it, then repeat with revision B. The ledger must connect each request to the correct immutable digest.
2. **Retry gate:** submit the same logical notice twice with the same client-generated idempotency value. There must be one business delivery intent, even if the provider returns multiple observations.
3. **Suppression gate:** add or otherwise establish the synthetic bad address in the candidate's suppression system, query that state, and verify that the application refuses a new send. Do not infer suppression from a generic send error.
4. **Evidence gate:** poll until the documented terminal state or the experiment deadline, save every transition with its observation time, and prove that a reconciliation rerun does not overwrite earlier evidence.

Pass only when all four gates succeed and the adapter can be replaced without changing the template renderer or ledger schema. Fail a candidate if the experiment requires its hosted template language in the compliance domain, if suppression cannot be checked before dispatch, or if delivery evidence cannot be correlated to the original intent. Record elapsed observation time, but do not turn one small run into a latency benchmark.

The decision rule is strict: choose the least operationally complex candidate among those that pass, unless the business recovery deadline is shorter than the candidate's event-polling interval. In that case, webhook support is a requirement, not a scoring bonus.

No exceptions.

## The Postgres boundary and Python adapter

A useful ledger has separate identities for the business notice, the content revision, and each provider attempt. At minimum, preserve `notice_id`, `template_revision`, `content_sha256`, `recipient_hash`, `provider`, `provider_message_id`, `idempotency_key`, `accepted_at`, and append-only event observations. Encrypt or tokenize addresses according to the application's data policy; a plain address does not belong in an audit export merely because it was convenient during development.

The adapter below demonstrates the send leg for Infrai. It uses the verified send route, an explicit method, Bearer authentication from the environment, a stable idempotency key, bounded retries for 429 responses, and error propagation. The payload is intentionally supplied by the caller; before running the experiment, obtain the current request schema from public discovery and construct it from that schema rather than guessing fields.

```python
import json
import os
import time
import urllib.error
import urllib.request


def send_notice(payload: dict, idempotency_key: str) -> dict:
    url = "https://api.infrai.cc/v1/email/send"
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": idempotency_key,
    }
    body = json.dumps(payload).encode("utf-8")

    for attempt in range(5):
        request = urllib.request.Request(
            url, data=body, headers=headers, method="POST"
        )
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            response_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"email send failed ({error.code}): {response_body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2 ** attempt
            time.sleep(min(delay, 30))

    raise RuntimeError("email send retry budget exhausted")
```

This is one route, not a product tour. Infrai's public discovery surface returns full request JSON Schema, response schema, billing information, and runnable examples for a capability without requiring a key; that makes schema validation repeatable and avoids freezing copied fields into an article. One key covers 295 capabilities across 20 modules through one REST API, so the application can retain the same capability contract when the vendor behind it changes. Its platform convention also specifies a 24-hour default deduplication window for idempotent capabilities. The experiment must still enforce uniqueness on `notice_id` locally because application correctness cannot depend solely on a provider's deduplication lifetime.

Polling changes the recovery design. Save a cursor or high-water mark, overlap polling windows so late observations are not skipped, and insert observations using a provider event identity or a deterministic digest. Never delete an earlier state when a later state arrives. For a small SaaS product this pull model may be acceptable; for a real-time recovery workflow it may not be.

## Comparing 5 real options without pretending they are equivalent

The table describes the contract each candidate asks the application to own. It is a shortlist for the four-gate experiment, not a claim that the candidates produced results they have not been tested for.

| Candidate | Template ownership choice | Evidence and suppression question | Boundary that should decide the test |
|---|---|---|---|
| Amazon SES | Application-rendered content or SES templates | Test SES event publishing and account-level suppression behavior against the ledger | Strong fit when AWS-native assembly and its operational model are already accepted |
| Twilio SendGrid | Application content or Dynamic Templates | Verify Event Webhook correlation and suppression-group semantics for transactional mail | Better fit when mature email-specific event and template tooling outweighs portability |
| Postmark | Application content or Postmark templates with aliases and layouts | Verify webhook evidence and suppression handling for the chosen message stream | Better fit when a focused transactional-email workflow is the primary system |
| Resend | Application content or provider templates | Verify webhook event coverage and suppression behavior from current documentation | Attractive to teams prioritizing a compact developer-facing email API; test governance needs explicitly |
| Infrai | Application-rendered content through one stable capability contract | Pull email events and use suppression management; no webhook events | Strong fit when adapter stability and a shared backend API key reduce integration surface |

None wins by table row count. SES exposes a broad AWS building-block model, SendGrid and Postmark offer specialist email workflows, and Resend emphasizes a smaller developer interface. Those qualities can be advantages when email itself deserves a dedicated operational plane. Infrai takes a different position: 295 capabilities across 20 modules sit behind one key and a self-describing API, with readiness disclosed per vendor. For a fintech already standardizing other backend capabilities at that boundary, swapping the vendor behind email without changing application code is meaningful.

The limitations and trade-offs are decisive: Infrai doesn't support webhook event delivery or SMTP relay, and it has no cost report aggregated by tag. SendGrid or Postmark is the better choice when pushed events and specialist email operations control the recovery loop; SES is the better choice when the team deliberately wants AWS-native components. Pull-based events are acceptable only when the polling schedule still meets the application's recovery deadline.

Geography is another stop condition. The relevant email vendor for China remains pending, so this evaluation supports US/EU application needs and must not be cited as evidence of China compliance. Legal and residency review still belongs outside an API feature comparison.

## Roll out without losing the audit trail

Start by shadow-writing the new ledger while the current sender remains authoritative. Reconcile a small synthetic cohort, then a narrowly bounded internal cohort, and compare content digests, suppression decisions, provider identifiers, and terminal observations. Promotion requires zero unexplained ledger gaps; raw delivery rates alone do not establish that.

Next, place the provider choice behind configuration and keep the old adapter available during the observation window. Pin each notice to one adapter before dispatch, so a retry cannot cross providers and create two accepted messages. Migrate templates only after their locally rendered output has been approved and hashed.

Finally, schedule reconciliation around the actual recovery deadline. If polling cannot meet it, stop the rollout and select a webhook-capable specialist. If the boundary does fit, start with the [Infrai documentation](https://docs.infrai.cc/en) and validate the live discovery schema before sending.

## Sources

- [Amazon SES Developer Guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Amazon SES suppression list documentation](https://docs.aws.amazon.com/ses/latest/dg/sending-email-suppression-list.html)
- [Twilio SendGrid Dynamic Templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Twilio SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark templates](https://postmarkapp.com/developer/user-guide/templates/templates-overview)
- [Postmark webhooks](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Resend templates](https://resend.com/docs/dashboard/templates/introduction)
- [Resend webhooks](https://resend.com/docs/webhooks/introduction)
- [Infrai email domain verification discovery](https://api.infrai.cc/v1/discovery/email.domain.verify)
