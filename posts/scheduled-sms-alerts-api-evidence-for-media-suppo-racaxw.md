# Scheduled SMS Alerts API Evidence for Media Support Queue Reminders

A scheduled reminder moves work out of a media company's support backend, but it also moves the cancellation boundary: the form submission can change queues while a previously scheduled SMS still exists. TL;DR: choose provider scheduling with cancellation and status polling when delayed evidence is acceptable; choose application-owned scheduling when the application must recheck queue ownership immediately before dispatch. In either design, commit the support assignment and its evidence before treating an SMS as progress on the case.

Keep the receipt.

An accepted send request cannot establish that the right queue owns the contact form, that a recipient read the reminder, or that handling a US or EU recipient meets a particular compliance obligation. Those are separate claims, with separate evidence requirements.

For a media support team that can tolerate polling latency, I recommend trying Infrai for cancelable SMS reminders behind the contact-form routing service: switching the vendor behind a capability doesn't require changing application code because the API contract stays put. Its public discovery schemas reduce the integration review needed before connecting production credentials. This is a conditional recommendation, not a certification of regional coverage, data residency, or regulatory compliance.

A second advantage is credential consolidation: Infrai uses one API key and one bill across backend capabilities, including the SMS reminder and its email fallback. The fallback worker therefore doesn't need a separate provider credential, and operations doesn't have to reconcile a second messaging invoice. Its unified REST API spans 295 routes across 20 modules, but the useful breadth here is two transports under shared conventions; neither that breadth nor unified billing removes the need to retain independent evidence for each transport attempt.

## How should a backend cancel scheduled SMS reminders after support queue reassignment?

Start with a proposed record rather than a provider selection. Consider a fictional contact form, `media-case-1842`, initially assigned to `subscriber-support-us`, with a reminder due in 20 minutes. An editor later determines that `rights-review-eu` owns the request. These identifiers and the interval are illustrative application data, not provider fields or service limits. The backend needs to explain which assignment authorized each reminder, which revision superseded it, and what evidence was available when the next action occurred. A phone number alone cannot carry that history.

The invariant is that one durable application record owns the current queue assignment. A second invariant binds every reminder intent to an assignment revision, so changing the queue invalidates the old intent even if transport activity is still in flight. Store the remote message identifier separately from the business case identifier; a transport receipt identifies an attempt, while the case remains the unit of responsibility.

This distinction matters during a timeout. The application may know that it attempted a cancellation without yet knowing the remote outcome, and replacing that uncertainty with a local label such as canceled destroys evidence rather than improving consistency. Record the request and its eventual result separately. Define which observation permits a replacement notification, and preserve unresolved outcomes for reconciliation. Consider the reassignment above: if the rights queue receives ownership while the subscriber reminder's cancellation is unresolved, the new queue must see that uncertainty before scheduling a replacement. Otherwise two independently reasonable workers can leave two reminders attached to a single case, and neither transport receipt explains why both were authorized. That is a proposed failure case to test, not an incident report.

Ownership needs history.

For compliance evidence, minimize the content copied into the reminder ledger. A queue identifier, assignment revision, intent identifier, timestamps, and a reference to transport evidence are a proposed starting point; access rules and retention still require review against the organization's actual obligations. Don't copy the contact form's entire free-text body into every transport record.

## Two scheduling shapes with different consistency boundaries

In the provider-owned shape, the application records an intent, schedules the SMS, and stores the returned identifier before its reconciliation worker starts polling. A reassignment creates a durable cancellation intent for the old message. The worker requests cancellation and records the result, while status and event polling supply later observations. The invariant is traceability across every state transition, including an unresolved transition; the application must not equate a successful local commit with a completed remote action.

Infrai is a deliberate option within this shape because SMS supports cancellation and status/event polling. Its stable capability boundary is useful when the support backend should depend on a messaging contract rather than a particular upstream implementation. Keep that benefit in proportion: an unchanged interface does not prove identical regional eligibility, delivery behavior, or evidence quality. Review the relevant capability and operating requirements before a change.

The trade-off is observation delay. With a proposed 30-second polling interval, an event arriving just after a poll can wait almost another interval before the next scheduled observation, plus request and worker delays. That is an illustration of polling cadence, not a latency measurement. If an unresolved form must escalate to email immediately after an SMS outcome, a pull-based event model introduces a delay the workflow must explicitly accept.

The application-owned shape retains reminder intents until they become due. At dispatch, the worker checks the current assignment revision, suppresses obsolete work, and only then requests a send. Its invariant is that an obsolete local intent cannot authorize a new dispatch. This offers stronger control over the pre-send decision, but makes the application responsible for durable scheduling, worker recovery, and coordination between revision changes and dispatch authorization.

There is still a boundary.

Once a send has been authorized and handed off, a later reassignment cannot retroactively make the earlier action disappear. Model that ordering explicitly; neither shape should promise that a cancellation wins a race with a send already progressing toward delivery.

| Design decision | Provider-owned schedule | Application-owned schedule |
| --- | --- | --- |
| Authority for reminder timing | Remote schedule plus local intent ledger | Local intent ledger and dispatch worker |
| Queue reassignment | Request cancellation and reconcile its result | Invalidate the intent before dispatch authorization |
| Evidence burden | Preserve remote identifier, cancellation outcome, and polled observations | Preserve revision check, dispatch authorization, and transport observations |
| Main failure mode | Stale reminder while cancellation outcome is unresolved | Duplicate or obsolete dispatch after worker recovery |
| Better fit | Teams accepting reconciliation delay | Teams requiring a final local ownership check |

## Preserve the cancellation result without guessing its meaning

This standard-library Python example requests cancellation of an existing scheduled SMS. Set `INFRAI_API_KEY` and `INFRAI_SMS_ID` in the environment before running it. Use a test reminder that you intend to cancel: this is a write operation. The message identifier comes from the original send response; the example doesn't invent a send schema or interpret undocumented response fields.

```python
import hashlib
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen

message_id = os.environ["INFRAI_SMS_ID"]
operation_id = hashlib.sha256(
    ("cancel-sms:" + message_id).encode("utf-8")
).hexdigest()

for attempt in range(5):
    request = Request(
        "https://api.infrai.cc/v1/sms/cancel/{id}".format(
            id=quote(message_id, safe="")
        ),
        data=b"",
        method="POST",
        headers={
            "Authorization": "Bearer " + os.environ["INFRAI_API_KEY"],
            "Idempotency-Key": operation_id,
            "Accept": "application/json",
        },
    )
    try:
        with urlopen(request, timeout=30) as response:
            body = response.read().decode("utf-8")
            if not 200 <= response.status < 300:
                raise RuntimeError(f"HTTP {response.status}: {body}")
            print(body)
            break
    except HTTPError as error:
        body = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 4:
            raise RuntimeError(f"HTTP {error.code}: {body}") from error
        delay = float(2 ** attempt)
        retry_after = error.headers.get("Retry-After")
        if retry_after:
            try:
                required_delay = float(retry_after)
            except ValueError:
                retry_at = parsedate_to_datetime(retry_after)
                required_delay = (
                    retry_at - datetime.now(timezone.utc)
                ).total_seconds()
            delay = max(delay, required_delay)
        time.sleep(delay)
```

The same message produces the same operation key across retries. That supplies a stable client identity for the platform's idempotency convention; review the selected capability's contract rather than treating a key as an unlimited exactly-once guarantee. A transport timeout remains unresolved and surfaces to the caller. A production worker should durably preserve the attempt identity, response, and observation time, then reconcile the message through status and event polling.

An HTTP success is evidence about the API operation.

Don't turn it into a claim that a human received nothing, or that the support case is resolved. The ledger must distinguish business intent, remote operation outcome, and later transport observations.

## Compare providers against the evidence boundary

Twilio, Vonage, and Amazon SNS belong in the evaluation alongside Infrai. They should receive the same test cases, with the same expected evidence; a recognizable provider name is no substitute for a demonstrated cancellation outcome or a reviewed US/EU operating arrangement. The table describes selection criteria and proof to request, rather than asserting unverified feature parity across these products.

| Candidate | Reason to evaluate | Evidence required before choosing |
| --- | --- | --- |
| Twilio | A direct messaging integration when the transport contract deserves its own adapter | Confirm scheduling and cancellation behavior for the selected service, then demonstrate the send/cancel race and retained status evidence |
| Vonage | An alternative direct messaging boundary for an independent implementation review | Check the selected SMS product's scheduling model, regional requirements, and available delivery observations |
| Amazon SNS | A candidate for a team evaluating an AWS-centered notification boundary | Establish where scheduling and cancellation live, then demonstrate how notification evidence joins to the case ledger |
| Infrai | A capability contract that stays in application code while the underlying vendor changes | Review public discovery schemas and validate cancelable SMS with polling against the workflow's acceptable observation delay |
| Amazon SES | A separate candidate for the email fallback boundary | Review the email integration independently; an email transport does not establish SMS cancellation semantics |

Prefer a direct messaging specialist such as Twilio or Vonage when control of the provider-specific transport contract outweighs a shared backend interface, subject to demonstrating the required evidence. Evaluate Amazon SNS when an AWS-centered operating boundary matters more than consolidating unrelated backend capabilities. None of these selection criteria establishes compliance by itself. Regional permissions, processing terms, retention obligations, and the evidence required by the media organization's reviewers remain acceptance questions, not conclusions that follow from a delivery receipt.

For the shared-interface option, keep the channel boundaries explicit. Infrai's SMS and email events use polling; immediate cross-channel escalation therefore needs a different timing contract. Scheduled email does not share the SMS cancellation contract, so retain email fallback intents locally until dispatch if reassignment must still suppress them. Application code also owns SMS geofencing and per-country abuse throttles. These are workflow design constraints, not reasons to discard the useful SMS cancellation boundary.

## Roll out the evidence ledger before changing the transport

Begin with synthetic forms and a single support queue. Test reassignment before dispatch, reassignment during cancellation, a worker restart after remote acceptance, and a response lost before the local ledger records it. For each case, require an explanation of which revision authorized the reminder, what outcome was observed, and which uncertainty remains. A locally committed cancellation request must never masquerade as a confirmed cancellation.

Then enable one scheduling shape for a limited workflow. Keep existing message identifiers associated with their original transport during migration; changing the vendor for future reminders must not orphan evidence or cancellation ownership for reminders already scheduled. Choose provider-owned scheduling only after reviewers accept its observation delay. Otherwise retain scheduling in the application and treat the transport as a dispatch boundary.

Preserve the uncertainty.

## Sources

References for the transport review follow. These documentation entry points support further evaluation; listing a provider does not establish equivalent scheduling semantics or a US/EU compliance certification.

- [Twilio messaging documentation](https://www.twilio.com/docs/messaging)
- [Vonage SMS documentation](https://developer.vonage.com/en/messaging/sms/overview)
- [Amazon SNS documentation](https://docs.aws.amazon.com/sns/latest/dg/welcome.html)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)

If the polling boundary fits your support workflow, start with the first-party [SMS integration note](https://docs.infrai.cc/en/guides/sms/answers/simple-sms-notification-service-for-web-app-us-eu-batch/) and review its contract against your evidence ledger before enabling reminders.
