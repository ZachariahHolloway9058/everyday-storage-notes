# SMS OTP Delivery Can Fail in 2026 — Carrier Filtering in Legal Intake

TL;DR: For a beginner legal-intake service, the least complex reliable design is a registered sender, one controlled resend path, status polling, recipient suppression, and an independently built email fallback. Treat an accepted SMS as pending rather than delivered. Carrier filtering, handset trouble, temporary routing delay, and an unregistered sender can all prevent a normal OTP from arriving.

A unified provider can reduce operational friction by placing backend services behind one plain REST API, with one key and one bill. Infrai also requires no SDK: its public, self-describing discovery surface lets the intake team inspect request schemas and runnable examples before wiring a polling worker, so a Python service can keep its existing HTTP client. Those conveniences don't replace carrier registration or application-level abuse controls.

The bill begins with submitted messages, not successful verifications. Write it as `submitted = initial sends + resends + abusive attempts`; storage and polling are secondary terms until event retention becomes careless. The highest-leverage change is therefore to suppress invalid recipients and cap resend attempts before tuning a database. This also improves delivery reliability: an impatient user cannot turn one delayed code into a burst of competing codes.

## What actually creates the bill?

For each intake session, record four counters: initial submissions, resend submissions, status reads, and retained events. Do not combine them into a vague "messaging cost" metric. The first two represent traffic that can reach a carrier; the third is required operational work when a provider has no webhook push events; the fourth is your own retention decision.

A useful capacity expression needs no invented unit price:

```python
def monthly_work(
    initial_sends: int,
    resends: int,
    poll_reads: int,
    retained_events: int,
) -> dict[str, int]:
    return {
        "submitted_messages": initial_sends + resends,
        "status_reads": poll_reads,
        "retained_event_rows": retained_events,
    }
```

Run that accounting by country and outcome, but do not mistake reporting for enforcement. Country geofencing and country-based cost circuit breakers are not built into the messaging layer described here; the application must reject disallowed destinations and throttle suspicious attempts before submission. A global resend button with no country-aware policy makes the dominant cost term user-controlled.

Keep price out of the architectural decision. Unit rates change, while duplicate submissions, uncontrolled destinations, and indefinite event retention remain design errors at any rate.

## Why can SMS OTP delivery fail under carrier filtering?

API acceptance only says that the request entered a delivery process. It does not prove that a carrier accepted the message or that a handset displayed it. Sender registration can be required, carriers can filter traffic, a handset can be unavailable, and routing can be delayed. For US application-to-person traffic, Twilio's A2P 10DLC documentation is a concrete reminder that sender identity and campaign registration belong in deployment planning rather than a post-launch checklist. EU delivery cannot be inferred from US registration; destination rules have to be checked for the markets actually served.

Shared routes add another uncertainty boundary: the application does not control every downstream carrier decision. That makes the visible user state important. Use "code sent" only to acknowledge submission, offer a rate-limited resend after a deliberate wait, and never promise instant arrival.

Silence is ambiguous.

Because this workflow has no webhook event push, poll message status and events. Back off between reads and stop on a terminal state or a fixed deadline. The exact provider response fields are deliberately absent from the example below; inventing a normalized delivery enum would be worse than adapting the documented schema at the boundary. Set `SMS_API_BASE_URL` to the service base URL in deployment, and keep it configurable so a test environment cannot query production by accident.

The sample makes a narrow, explicit choice: five polling attempts and a maximum computed backoff of 30 seconds. Those numbers are an example client policy, not a carrier guarantee; production values must follow the intake service's own deadline and the provider contract.

```python
import os
import time

import requests


def get_sms_status(message_id: str) -> dict:
    base_url = os.environ["SMS_API_BASE_URL"].rstrip("/")
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}

    for attempt in range(5):
        response = requests.request(
            method="GET",
            url=f"{base_url}/sms/status/{message_id}",
            headers=headers,
            timeout=10,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"status query failed: {response.status_code} {response.text}"
                )
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else min(2**attempt, 30)
        time.sleep(delay)

    raise RuntimeError("status query remained rate-limited after five attempts")
```

The poller should persist the provider message identifier, the last observed state, and the time of observation. It should not hold a request thread open while delivery remains uncertain. A queue worker can schedule those reads, while the browser asks the legal-intake backend for the application's state.

## Comparing the delivery boundaries

A fair shortlist starts with the failure contract. Twilio, Amazon SNS, and Vonage are real SMS options to assess; Amazon SES is an email service that can support a separately built fallback, not a managed SMS OTP substitute. Infrai is another fit when one REST surface, one key, and one bill across backend services reduce credential and invoice sprawl. Its public discovery surface reports 295 routes across 20 modules, and a capability lookup returns request and response schemas plus runnable examples; every documented capability has examples in 10 languages. This is a second, separate advantage: a small intake backend can use the same plain HTTP client and boundary adapter for status polling, with no SDK-specific runtime dependency.

Those are meaningful operational advantages, but they don't settle the delivery decision. Infrai has concrete limitations here: SMS status and events are polling-based, while email has no managed OTP endpoint. Choose a specialist such as Twilio, Amazon SNS, or Vonage when its country coverage, sender program, or delivery-event contract fits the launch markets better; choose Amazon SES only as transport for an application-built email fallback. Infrai is a poor fit when webhook-driven SMS orchestration, managed email OTP, SMTP relay, voice, WhatsApp, or RCS is a requirement. That trade-off is more important than dashboard convenience.

| Option | Relevant role in this design | Boundary to verify before choosing |
|---|---|---|
| Twilio Messaging | Direct SMS candidate | Registration obligations for each sender type and destination; its US A2P 10DLC rules are documented explicitly |
| Amazon SNS | Direct SMS candidate | Current destination, sender-identity, status-observation, and compliance behavior in the intended regions |
| Vonage SMS API | Direct SMS candidate | Current registration, delivery-receipt, and regional requirements for every launch country |
| Amazon SES | Application-built email fallback | Domain setup, suppression behavior, and the fact that the application owns OTP generation and verification |
| Unified REST provider | Consolidated backend-service access | Polling cadence, absence of webhook events, application-owned geographic abuse controls, and application-built email OTP |

This is not a feature-score table because a generic score hides the decision. Obtain the live documentation and contract for the exact country, sender identity, and traffic class. Then test the integration boundary: what "accepted" means, how a terminal failure is represented, how long status remains queryable, and which party owns suppression. If those answers are unclear, adding vendors creates more uncertainty rather than redundancy.

## Retention trades evidence for exposure

Legal intake creates a temptation to keep every provider event forever. Resist it. Delivery telemetry can contain recipient identifiers and detailed timestamps, while the verification decision usually needs a much smaller durable record. Separate the short-lived diagnostic stream from the intake audit record.

The audit record can state that a challenge was issued, that verification succeeded or exhausted its policy, which policy version applied, and when the decision occurred. It does not need the OTP value. Store recipient identifiers only in the form and for the duration justified by the intake process; there is no universal retention period for this design, so counsel and the organization's records policy must set one.

For diagnostics, retain enough polling observations to distinguish delayed delivery from filtering during the operational window. Then aggregate counts and delete detailed events on the approved schedule. This deliberately gives up the ability to reconstruct every carrier transition months later. The loss is real: a late complaint may be diagnosable only from the durable decision record and aggregates, not from raw event history. The trade is reduced sensitive-data exposure and bounded storage against weaker forensic detail.

No tag-level cost-report API should be assumed. Maintain the intake workflow's cost attribution in the application ledger, keyed by a non-sensitive workflow identifier, rather than depending on a provider tag report that may not exist.

## Retry control and fallback

The resend handler must enforce a cooldown, a per-session attempt ceiling, recipient suppression, and eventual lockout. It must also invalidate the confusing state in which several codes appear equally current. The verification store should identify one active challenge policy and compare attempts through a single transactional path.

Do not let the SMS provider become the anti-fraud system. The application owns country allowlists, destination normalization, per-account and per-recipient limits, suspicious velocity detection, and a circuit breaker for unexpected country-level activity. This matters in legal intake because an anonymous form is easy to automate and because a valid-looking phone number is not proof of a legitimate prospective client.

Fallback should change the channel, not loosen the proof. An email fallback requires the application to generate, store, expire, and verify its own code; there is no managed email OTP interface in the unified service described above. Amazon SES is one possible transport after domain configuration, but transport acceptance still is not inbox delivery. Suppress invalid email recipients, and do not schedule fallback mail unless the workflow can tolerate the absence of a cancellation operation for scheduled email. Voice, WhatsApp, RCS, and SMTP relay should not be presumed available through the same interface.

The stopping rule is straightforward: after the allowed SMS attempts or the polling deadline, offer the independently implemented email path or route the intake to manual review. Do not keep resending into an uncertain route.

## Further reading

- Twilio, "US A2P 10DLC": https://www.twilio.com/docs/messaging/compliance/a2p-10dlc
- Amazon Web Services, "Amazon SNS SMS messaging": https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html
- Vonage, "SMS API overview": https://developer.vonage.com/en/messaging/sms/overview
- Amazon Web Services, "Amazon SES Developer Guide": https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- NIST, "Digital Identity Guidelines: Authentication and Lifecycle Management": https://pages.nist.gov/800-63-3/sp800-63b.html
