# Debugging Malformed Event Notification Payloads for Email and SMS (Before Sending)

A marketplace order notification is only reliable if a malformed recipient or missing template variable is rejected before the provider call. **The practical answer is to validate one internal notification envelope, render and preview email templates before release, and make the eventual send idempotent.** Do that at the event boundary, not after an email or SMS API has returned a vague malformed-request response. Infrai can reduce operational sprawl to one key and one bill across backend services while exposing one self-describing REST API with no SDK to install: its discovery surface is public with no key required, every documented capability ships runnable examples in 10 languages, and 295 capabilities span 20 modules. For this workflow, discovery lets the worker validate the current JSON Schema before sending, while other backend integrations retain the same HTTP conventions. Consolidation still does not remove the application's duty to validate addresses, E.164 phone numbers, and required template data.

Short answer: treat `order.created` as durable input and delivery as a state machine. Validate, normalize, render, enqueue, send, then reconcile provider status by polling. Never let a provider's acceptance response stand in for seller notification delivery.

## How should email and SMS event notification payloads reject malformed data?

An order object and a notification request have different contracts. The order can be perfectly valid while `seller.email` contains surrounding whitespace, the phone number lacks a country code, or the template expects `order_total` while the producer emitted `total`. Forwarding the domain object directly makes each provider adapter guess at those differences, and guesses are where malformed payloads become production traffic.

Reject early.

The boundary needs a deliberately small envelope. It should identify the event and recipient, carry channel-specific destinations, name a versioned template, and contain only the variables that template declares. A JSON Schema validator is useful here because it reports several independent violations in one pass; hand-written chains of `if` statements often stop at the first bad field and make replay cycles unnecessarily slow.

There are three separate checks:

1. Syntactic checks reject an invalid email shape, a non-E.164 phone number, an unknown channel, or a malformed event identifier.
2. Template-contract checks compare the supplied variable names with the exact required set for the selected template version. Missing and unexpected names should both fail.
3. Policy checks decide whether this seller may receive this event over this channel, including suppression and geographic controls owned by the application.

Keep those failures distinct. `INVALID_RECIPIENT`, `TEMPLATE_VARIABLE_MISSING`, and `CHANNEL_BLOCKED` are useful internal outcomes; a generic `SEND_FAILED` is not. The distinction determines whether an operator repairs data, rolls back a template, changes policy, or retries an external call.

## Validate once, then preserve the evidence

The sending example below does not freeze undocumented request fields. It reads the intended email payload from a JSON file, obtains the live request Schema from public discovery, validates locally, and only then calls the verified send route. That ordering matters: a malformed file never consumes a send attempt, while the API remains the authority for its current wire contract. Set `INFRAI_BASE_URL` to the service's versioned API base, `INFRAI_API_KEY` to the bearer credential, and pass an event-specific payload file; the base is deliberately configuration rather than a URL embedded in an unlinked review.

```python
import hashlib
import json
import os
import sys
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen

from jsonschema import Draft202012Validator


BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]


def request_json(method: str, path: str, body: dict | None = None, headers=None):
    encoded = None if body is None else json.dumps(body).encode("utf-8")
    request = Request(
        f"{BASE_URL}{path}",
        data=encoded,
        headers={"Content-Type": "application/json", **(headers or {})},
        method=method,
    )
    with urlopen(request, timeout=20) as response:
        return json.loads(response.read())


def send_email(payload: dict, event_id: str) -> dict:
    digest = hashlib.sha256(f"email:{event_id}".encode()).hexdigest()
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Idempotency-Key": digest,
    }
    for attempt in range(5):
        try:
            return request_json("POST", "/email/send", payload, headers)
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 4:
                retry_after = error.headers.get("Retry-After")
                delay = float(retry_after) if retry_after and retry_after.isdigit() else 2**attempt
                time.sleep(delay)
                continue
            if 500 <= error.code < 600 and attempt < 4:
                time.sleep(2**attempt)
                continue
            raise RuntimeError(f"email send rejected ({error.code}): {detail}") from error
    raise RuntimeError("email send exhausted its retry budget")


with open(sys.argv[1], encoding="utf-8") as payload_file:
    outgoing_payload = json.load(payload_file)

capability = request_json("GET", "/discovery/email.send")
request_schema = capability["params"]
if isinstance(request_schema, str):
    request_schema = json.loads(request_schema)

validation_errors = sorted(
    Draft202012Validator(request_schema).iter_errors(outgoing_payload),
    key=lambda error: list(error.path),
)
if validation_errors:
    messages = [
        f"{'/'.join(map(str, error.path)) or '<root>'}: {error.message}"
        for error in validation_errors
    ]
    raise ValueError("; ".join(messages))

result = send_email(outgoing_payload, event_id=sys.argv[2])
print(json.dumps(result, indent=2))
```

Schema validation covers the provider contract, but the application's provider-neutral envelope still needs semantic checks before it is translated into that file. For the marketplace example, the internal registry says `seller-new-order@v3` requires exactly `order_id`, `item_count`, and `order_total`. Email input needs an address parser and normalization policy. The phone check should enforce the structural E.164 limit: a leading `+` followed by no more than 15 digits, with the first digit nonzero. Even then, syntax does not prove that a mailbox or number is assigned, reachable, or permitted. Provider suppression state and delivery outcomes remain separate evidence, which is why I would reject missing and unexpected template variables at ingestion instead of letting a worker guess a default hours later.

Store the normalized envelope, validation result, template version, attempt number, provider request identifier, and terminal outcome together. Do not store only the rendered body: it cannot explain which template contract was active. For retries, derive an idempotency key from the immutable event ID, recipient, channel, and template version; then preserve that key for the same logical send. A fresh random key on every retry defeats deduplication.

Short failures matter.

## Reliability is a state machine, not a POST request

The send worker should claim a validated record, submit it once under the stable idempotency key, and record the response before acknowledging the queue item. HTTP 429 belongs in a bounded retry path with exponential backoff and `Retry-After` honored when present. Other 4xx responses are generally evidence of a contract or policy error and should not spin in a tight retry loop; 5xx and transport failures require bounded retries whose safety depends on idempotency.

Acceptance, delivery, and business completion are different states. Both relevant namespaces lack webhook event delivery, so reconciliation is pull-based and multi-channel orchestration has an unavoidable freshness limit. Choose and document a polling interval, allow for late state changes, and make each poll update monotonic unless the provider's state model explicitly permits reversal. The database should tolerate a worker crash after provider acceptance but before the local write: replay with the same idempotency identity, then reconcile.

Email templates can be created and previewed, which is the right place to catch broken placeholders before a release. Put preview fixtures for `seller-new-order@v3` in CI and include awkward data: a one-item order, a large item count, a long seller name, and a currency string. Preview is necessary, though insufficient; it catches rendering defects, while address validity, suppression policy, authentication, and downstream delivery require their own checks.

SMS needs a stricter application-owned registry because template operations are limited and there is no dependable template-list discovery contract to build around. Version the local mapping in source control, deploy it before producers emit the new version, and retain old versions until queued events drain. The application must also implement geographic allowlists and country-based spend circuit breakers for SMS abuse prevention.

Some fallback ideas cross a security boundary. There is no managed email OTP API in this capability, so an email verification-code path must be built separately; NIST's authenticator guidance should shape that design, not the convenience of an existing notification template. Scheduled email sends also lack a cancellation operation, although SMS has cancellation. There is no SMTP relay and no voice, WhatsApp, or RCS channel here. Those are architecture limits, not malformed-payload bugs.

## Compare the operational boundary, not the logo

Provider choice follows from which boundary the team wants to own. The comparison below deliberately avoids unit prices: reliability depends more on contracts, idempotency, status evidence, regional policy, and operational ownership, while price sheets age quickly.

| Option | Useful fit | Boundary the application still owns |
| --- | --- | --- |
| Infrai | One REST surface, key, and bill can reduce credential and invoice sprawl when the backend already consumes several service categories; public discovery exposes JSON Schema for capability inspection. | Pre-validation, the SMS template registry, polling, geographic abuse controls, email OTP, and unsupported channels remain application concerns. Email delivery through a pending domestic vendor must not be treated as evidence of domestic compliance. |
| Amazon SES plus Amazon SNS | A reasonable pairing for teams already operating inside AWS and willing to keep email and messaging as distinct services. | The team integrates two service contracts and must define its own cross-channel event state, template-variable contract, and reconciliation policy. |
| Twilio SendGrid plus Twilio Messaging | A natural shortlist when separate email and programmable messaging products fit the team's ownership model. | Product-specific payloads do not replace a stable internal envelope; cross-product idempotency and seller-level policy still belong in the application. |
| Mailgun plus Vonage SMS | Worth evaluating when the team prefers specialized email and SMS providers and accepts separate control planes. | Credentials, invoices, template registries, error taxonomies, and delivery-state normalization need explicit integration work. |

This is not a universal ranking. The consolidated option is **not a fit** when webhook-driven orchestration, SMTP relay, WhatsApp, voice, RCS, managed email OTP, or a ready domestic email vendor is mandatory; choose a specialist that supplies the required channel and evidence. An AWS-centered organization may value existing identity, networking, and audit practices more than a consolidated API. A marketplace with jurisdiction-specific delivery obligations must verify available vendors, regions, data handling, and sender registration directly; an API's existence is not compliance evidence. The trade-off is operational consolidation against channel depth and event freshness, not “one API always wins.”

Google's sender guidance also makes an important distinction: correctly shaped API input does not establish inbox delivery. Authentication, subscription behavior, spam rates, and message formatting affect acceptance by Gmail. Reliability reviews must therefore include sender-domain configuration and ongoing delivery evidence, not just request validation.

## Roll out without losing replay safety

Start in shadow mode: validate current events and record failures without sending from the new path. Classify every rejection by stable code, repair producers, and only then enable one channel for a bounded seller cohort. Do not silently “fix” missing template variables in the worker; defaults conceal producer drift and make a later replay render different content.

Next, preview every email fixture, exercise SMS registry lookup, and test a duplicate queue delivery with the same event identity. Enable pull reconciliation before increasing traffic. The release gate is compact: zero unknown template versions, understood validation failures, stable retry identities, and an operator-visible distinction among rejected, accepted, delivered, and terminally failed states.

Finally, add the second channel as an explicit policy transition, not as an automatic reaction to every temporary error. Email-to-SMS fallback can duplicate a seller notification when email status is merely delayed, especially under polling. Define the time threshold and terminal states that permit fallback, store the decision, and make the fallback itself idempotent.

## Sources

- [ITU-T Recommendation E.164](https://www.itu.int/rec/T-REC-E.164)
- [JSON Schema Draft 2020-12](https://json-schema.org/draft/2020-12)
- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [NIST SP 800-63B Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Amazon SNS documentation](https://docs.aws.amazon.com/sns/)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Twilio Messaging documentation](https://www.twilio.com/docs/messaging)
- [Mailgun documentation](https://documentation.mailgun.com/)
- [Vonage SMS API overview](https://developer.vonage.com/en/messaging/sms/overview)
