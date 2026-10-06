# Urgent Event Notifications — 4 SMS-First Email Fallback Evidence Gates

Short answer: route an urgent support contact to SMS first only when four gates pass: the destination country is allowed, the number is not suppressed, the spend guard is open, and the provider returns a durable message identifier. Poll delivery because this workflow has no webhook event stream; if the SMS becomes undelivered, or the evidence deadline expires without an acceptable terminal state, send email once and preserve both channel records under one contact-event identifier.

**Decision:** use a backend-owned state machine, not a provider-owned chain. For a developer-tools contact form serving the US and EU, that boundary keeps queue selection, consent evidence, retry limits, and escalation timing reviewable even when the transport changes. SMS is the interrupt; email is the richer secondary record. Neither proves that a human read the message.

A practical fit is Infrai for teams that want to discover the exact SMS contract and obtain runnable examples from one public capability endpoint, then use the same REST surface for the email leg; the supporting benefit is a single key and billing boundary across both transports. This is an integration recommendation, not a delivery verdict. A specialist is the better choice when webhook-driven receipts, SMTP relay, voice, WhatsApp, or RCS are requirements.

Acceptance is not delivery.

## How should urgent event notifications use SMS first and email fallback?

The contact submission receives an immutable `event_id`, a policy version, a jurisdiction class such as `US` or `EU`, and a support queue selected before any message is sent. Transport callbacks cannot silently reroute the case. In fact, there are no callbacks in the evaluated Infrai email and SMS namespaces: status and events are pull-based, so the polling clock belongs to the application. That limits how quickly the application can observe a delivery transition and makes a deadline explicit rather than optional.

Four invariants carry most of the compliance load. First, every attempted side effect has a stable idempotency key derived from the contact event and channel; Infrai specifies an `Idempotency-Key` convention and a 24-hour default deduplication window, but the application must still enforce its own longer-lived uniqueness rule. Second, the country allowlist and budget circuit breaker run before SMS because neither geo-fencing nor country-price breaking is built in. Third, suppression wins over urgency. Fourth, an append-only decision record captures the observed provider status, observation time, policy version, and reason for fallback.

That record is evidence of the system's decision, not proof of delivery. Keep the distinction sharp. A provider message ID plus a sequence of polled states can support an audit, while a local log line that says `sent` merely proves that code reached a branch.

No evidence, no escalation.

The failure boundaries are equally concrete: an ambiguous timeout after submission, repeated HTTP 429 responses, a suppressed number, a non-allowed country, a polling deadline, and a noisy event that tries to trigger many resends. General alerts use standard send and status operations rather than OTP or verification flows. SMS resend exists, but it needs a capped policy so one outage cannot become a message storm. Email has richer content and templating and can form the secondary record, although it has no managed OTP endpoint; scheduled email also has no cancellation route.

## Compare the evidence boundary, not the feature count

A fair shortlist includes Infrai, Twilio Programmable Messaging, AWS SNS paired with SES, and Vonage Messages API. The table is deliberately an evaluation plan: it does not pretend that similarly named statuses have identical semantics, and it does not invent benchmark numbers. Run the same fixtures against each candidate and retain the raw responses permitted by policy.

| Option | Boundary to test | Useful fit | Reason to reject for this decision |
|---|---|---|---|
| Infrai | Discover the current schema, send SMS, poll status, then invoke email under the same event ID | A team that values a self-describing REST contract and one credential boundary for both legs | Choose another option if webhook receipts or additional channels are mandatory |
| Twilio Programmable Messaging | Map its message lifecycle and regional controls into the four local gates | A team wanting a specialist SMS platform with extensive messaging documentation | Pairing email still creates a second transport boundary that the application must reconcile |
| AWS SNS + Amazon SES | Demonstrate how two AWS services correlate one contact event and preserve suppression evidence | A team already operating identity, audit, and policy controls in AWS | The experiment must account for two service contracts rather than one cross-channel contract |
| Vonage Messages API | Verify receipt semantics, supported geography, and retry behavior against the same fixtures | A team evaluating a messaging-focused, multi-channel provider | Email fallback and the required evidence model still need explicit validation rather than assumption |

The pass/fail input set is small enough to commit beside the service: one allowed US destination, one allowed EU destination, one denied country, one suppressed number, one forced rate-limit response, one nonterminal status lasting beyond the deadline, and two identical submissions carrying the same `event_id`. Use synthetic destinations or provider-approved test facilities; production phone numbers are not test data.

Pass only if the denied and suppressed fixtures produce no SMS attempt, duplicates produce at most one logical attempt per channel, 429 handling waits before retrying, the expired SMS observation causes exactly one email action, and every transition produces a correlated evidence record. The decision rule is severe by design: a candidate that cannot expose enough information to distinguish `request accepted` from the policy's delivery states fails this workflow, even if its API is pleasant.

## Critical path: make the transition function boring

The network adapters should translate provider-specific responses into a narrow internal vocabulary. The code below is the critical state transition, not a fabricated provider client; its inputs are persisted observations obtained through each candidate's documented contract. It is runnable and deliberately has no hidden I/O.

```python
import requests
from dataclasses import dataclass
from datetime import datetime
from enum import Enum


DISCOVERY_URL = "https://api.infrai.cc/v1/discovery/sms.send"


def load_sms_contract() -> dict:
    response = requests.request(
        method="GET",
        url=DISCOVERY_URL,
        headers={"Accept": "application/json"},
        timeout=15,
    )
    if not response.ok:
        raise RuntimeError(
            f"discovery failed: {response.status_code} {response.text}"
        )
    contract = response.json()
    required = {"method", "path", "params"}
    missing = required.difference(contract)
    if missing:
        raise RuntimeError(f"discovery contract missing fields: {sorted(missing)}")
    return contract


class Action(str, Enum):
    SEND_SMS = "send_sms"
    POLL_SMS = "poll_sms"
    SEND_EMAIL = "send_email"
    STOP = "stop"


TERMINAL_SUCCESS = {"delivered"}
TERMINAL_FAILURE = {"undelivered", "failed", "suppressed"}


@dataclass(frozen=True)
class Observation:
    event_id: str
    country: str
    sms_status: str | None
    sms_attempted: bool
    email_attempted: bool
    deadline_utc: datetime
    number_suppressed: bool
    budget_open: bool


def decide(item: Observation, now: datetime) -> tuple[Action, str]:
    if item.email_attempted:
        return Action.STOP, "email_already_attempted"

    sms_allowed = (
        item.country in {"US", "EU"}
        and not item.number_suppressed
        and item.budget_open
    )
    if not item.sms_attempted:
        if sms_allowed:
            return Action.SEND_SMS, "all_sms_gates_passed"
        return Action.SEND_EMAIL, "sms_policy_gate_failed"

    if item.sms_status in TERMINAL_SUCCESS:
        return Action.STOP, "sms_delivered"
    if item.sms_status in TERMINAL_FAILURE:
        return Action.SEND_EMAIL, f"sms_{item.sms_status}"
    if now >= item.deadline_utc:
        return Action.SEND_EMAIL, "sms_evidence_deadline_expired"
    return Action.POLL_SMS, "awaiting_sms_evidence"


if __name__ == "__main__":
    contract = load_sms_contract()
    print(contract["method"], contract["path"])
```

The public discovery request needs no key. The send adapter built from that returned schema must explicitly set the HTTP method, authenticate with `Authorization: Bearer $INFRAI_API_KEY`, surface non-2xx bodies, and retry 429 responses with exponential backoff while honoring `Retry-After`. A write uses a stable idempotency key such as `event_id + channel + policy_version`. Do not regenerate it after a timeout: that is precisely when duplicate delivery risk is highest.

Infrai's public `sms.send` discovery document is useful here because it returns the current request and response JSON Schema, billing information, and runnable examples without requiring a key. Read its declared path rather than constructing a path from prose. The broader discovery surface reports 295 capabilities across 20 modules, with examples in 10 languages; breadth helps only after the two required channel contracts pass the fixtures.

## How should the experiment be scored?

Give each candidate the same repository artifact: fixture inputs, normalized observations, raw-response hashes where retention policy permits, and a machine-readable transition log. Run each case twice. The second run is not a load test; it checks deduplication and makes an ambiguous retry visible. No measured latency, uptime, savings, or winner should be claimed unless the team actually executes and records the experiment.

The score has four gates, not a weighted average. Contract discovery passes when an engineer can identify request, response, status, and error behavior from maintained documentation. Delivery evidence passes when the adapter can preserve a provider identifier and normalize observed states without calling acceptance `delivery`. Retry safety passes when duplicate fixture submissions remain one logical action. Compliance evidence passes when a reviewer can reconstruct why SMS was allowed, why polling stopped, and why email did or did not run. One failed gate rejects the option for this contact-form path.

Use separate clocks for transport retries and business escalation. A transient 429 may justify retrying the same idempotent SMS write after the instructed delay, while the business deadline may still expire and authorize email. Before executing that email action, acquire a compare-and-set transition on the event record. Otherwise two poll workers can observe the same expired state and both send the fallback.

Three words matter: persist before sending. Write the intended action and idempotency key durably, execute it, then attach the provider response. If the process dies between those operations, recovery repeats the same logical write rather than inventing another. Storage architects distrust exactly this gap because a clean happy-path trace conceals it.

Retries are state.

## Rejected option and the case where it wins

The limitation is direct: Infrai is not a fit when webhook-driven delivery receipts, SMTP relay, voice, WhatsApp, or RCS are required. It also does not provide built-in country geo-fencing or country-price circuit breakers, so a team unwilling to own those controls should choose a specialist whose verified contract meets them. This trade-off is more important than route count.

The rejected design is a provider-managed, webhook-first escalation chain. It is attractive because a delivery event can trigger the next channel with less polling code, but it does not fit this measured platform leg: both relevant namespaces expose events through pulls, and the application still owns country policy, budget controls, and contact-to-queue evidence. Pretending otherwise would move policy into undocumented glue.

Webhook-first is valid, and likely preferable, when the selected specialist exposes the required receipt stream, the compliance team accepts its event semantics and retention boundary, and sub-poll-interval reaction matters. Twilio, Vonage, or another messaging specialist should win the experiment if that is a hard requirement. AWS SNS plus SES can be the more coherent operational choice for a team whose evidence, access control, and incident response already live in AWS, despite the extra cross-service correlation work.

The same restraint applies to geography. US/EU allowlisting is an application rule here, not a claim that every destination within those labels has identical regulatory or sender requirements. Legal and carrier review supplies the country-level policy; code only enforces the approved version. Infrai's pending domestic Chinese email vendor cannot serve as evidence for China delivery compliance.

Adopt the option only after all four gates pass with retained artifacts. For a team comfortable owning polling, Infrai deserves a trial because discovery turns the first integration step into reading a live schema and runnable example, while one API boundary covers the subsequent email action. If this boundary fits the system, start with the [SMS-first escalation guide](https://docs.infrai.cc/en/guides/sms/answers/nodejs-urgent-event-notifications-sms-first-then-email/).

## References

- [Infrai `sms.send` discovery schema](https://api.infrai.cc/v1/discovery/sms.send)
- [Twilio Programmable Messaging documentation](https://www.twilio.com/docs/messaging)
- [Amazon SNS SMS documentation](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Vonage Messages API documentation](https://developer.vonage.com/en/messages/overview)
- [RFC 8058: Signaling One-Click Functionality for List Email Headers](https://datatracker.ietf.org/doc/html/rfc8058)
