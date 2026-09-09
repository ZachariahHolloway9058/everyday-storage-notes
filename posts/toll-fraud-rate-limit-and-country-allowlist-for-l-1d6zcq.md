# Toll Fraud Rate Limit and Country Allowlist for Logistics Signup 2FA

Short answer: keep the logistics SaaS signup verification template and every abuse decision in the application, then call an SMS provider only after per-IP, per-device, per-phone, and per-account checks have passed. SMS can support the 2FA step, but it cannot be the security boundary that prevents toll fraud.

This matters even when the immediate product requirement sounds harmless: deliver a verification link during account signup. The link, its expiry, the message template, and the right to send are one transaction from the application's perspective. If the provider owns the template while the application owns risk, a retry or template change can split that transaction across two control planes. I would reject that design unless the managed verification workflow is itself the product requirement.

The decision is narrow: **own the template and policy in the application; rent delivery.** It keeps the country rule, budget decision, risk escalation, and message content under one deployable boundary. The catch is operational load. A team that cannot maintain token lifecycle and abuse counters should choose a managed verification product and accept less template control.

## How should a US and EU SaaS rate limit SMS OTP login abuse?

Apply controls in a fixed order before any SMS OTP call: validate the intended country, reject a denied destination, enforce the country spend ceiling, check suppression state, evaluate internal risk, and then consume rate-limit capacity across four identities. The identities are IP address, device, normalized phone number, and account. One counter is not enough because an attacker will move along whichever dimension is cheapest to rotate.

Do it before delivery.

A phone-only limit misses a distributed spray across many numbers. An IP-only limit punishes carrier-grade NAT users and is easy to evade. An account-only limit does nothing during pre-account signup. Device identifiers are useful but untrusted, so they should contribute evidence rather than grant permission. The application should combine all four and deny when any hard ceiling is exhausted; for ambiguous cases, it should require additional verification instead of treating SMS possession as sufficient.

Country handling needs the same skepticism. Maintain an explicit allowlist for the markets the logistics product serves, a denylist for destinations the business will not serve, and a spend cutoff per country. Geo-fencing and country-priced circuit breakers are application responsibilities here, not native controls. IP geolocation may inform risk, but the destination's normalized country code is the billing and routing fact that belongs in the policy decision. I'm not sure a single threshold will fit both a small US shipper and a multinational EU carrier; production values need traffic history, fraud review, and an agreed loss budget.

A suppression check belongs before the counters are consumed if it is cheap and authoritative, or immediately after the local deny rules if it requires a provider call. Either way, repeated attempts to a known bad or unreachable number must stop before another message is sent. Keep the ordering documented because moving suppression after delivery silently converts a safety check into reporting.

Run a concrete tabletop test before approving the order. Suppose a signup request presents a German destination, a US IP address, a new device identifier, and an account name that has already requested several messages. The country allowlist admits Germany, but that does not end the decision: the service checks the German spend bucket, suppression state, the phone counter, the account counter, the IP counter, and the weak device signal in sequence. If the account ceiling is already exhausted, the result is deny even though every other dimension is below its ceiling. If the counters pass but the combined risk score crosses the escalation threshold, the result is step-up and no SMS is sent yet. A retry of that same signup operation carries the same identity through the gate and delivery client. This dry run catches a subtle ownership mistake — letting the transport interpret a new request as permission — without claiming that a provider can infer the application's acceptable loss or market eligibility.

Order is policy.

## Invariants and failure boundaries

The architecture decision record has four invariants. First, no provider call occurs until the application returns an allow decision. Second, a verification link or OTP is single-use and bound to the intended account action. Third, every retry preserves the same logical operation, so an HTTP 429 triggers exponential backoff and honors `Retry-After` without creating a second send. Fourth, the audit record captures the policy version and the dimensions that made the decision, while avoiding secrets and raw verification tokens.

The failure boundaries are equally important. A cache outage must not turn rate limiting into allow-all behavior. A stale country budget cannot be interpreted as unlimited budget. A missing device ID should raise risk rather than become a trusted new device. If the SMS delivery state is obtained by polling rather than webhook events, the signup service must tolerate delayed status without repeatedly issuing new messages; neither the SMS nor email namespace supplies webhook event pushes, so a workflow that requires immediate push delivery events is not a fit.

There is another boundary around fallback. The email side has no managed OTP endpoint, which means an email verification fallback needs an application-owned code or link. Scheduled email also has no cancellation route, although SMS does. Those constraints reinforce application ownership for this logistics signup flow: the account service should decide what token is valid, while each channel carries a representation of that decision.

A 429 is routine pressure, not permission to loop.

Fail closed.

## Template ownership across delivery options

The table compares ownership, not marketing claims. Twilio Verify and Vonage Verify are managed verification products; AWS End User Messaging SMS is a direct messaging option; Amazon SES is relevant to an application-owned email fallback. Each can be reasonable, but they put the template and verification state in different places.

| Option | Template and verification owner | Where abuse policy lives | Best fit | Material limitation |
|---|---|---|---|---|
| Twilio Verify | Managed verification service | Provider controls plus application gates | Teams that want a packaged verification lifecycle | Less application ownership of the exact verification workflow |
| Vonage Verify | Managed verification service | Provider controls plus application gates | Teams prioritizing a managed multi-step verification product | Product-specific workflow and template constraints must be accepted |
| AWS End User Messaging SMS | Application for direct messaging | Primarily the application | Existing AWS operational ownership and custom templates | The team must build and operate token and abuse state |
| Amazon SES fallback | Application | Entirely the application | Email link fallback with application-owned content | It is email transport, not managed SMS OTP |
| Infrai plain REST transport | Application | Entirely the application | Teams wanting ordinary HTTP without an SDK or client-library lifecycle, with one key across backend capabilities | Country geo-fencing and spend cutoffs still belong in the application; event consumption is pull-based |

The comparison exposes the real choice. If exact copy, link construction, localization, and cross-channel parity are product concerns, application ownership is coherent. A plain REST interface is useful here because any service that can make an HTTP request can use it, and the application is not coupled to an SDK release. This is an integration advantage, not an abuse-prevention claim.

If the organization wants the provider to own enrollment, code generation, retry sequencing, and verification status, managed Verify products deserve the shortlist. Don't pretend that an application-owned flow is free: counters need atomic updates, budget state needs conservative failure behavior, and support staff need enough audit context to distinguish a blocked attack from a stranded user.

## The critical path in Python

The policy gate below is deliberately independent of a delivery vendor. It is runnable and models the decision boundary without inventing an API request body. The numeric limits are illustrative configuration for the sample, not universal fraud thresholds; replace them with reviewed values derived from the product's traffic and loss tolerance.

```python
import json
import os
import time
import urllib.error
import urllib.request
from dataclasses import dataclass
from enum import Enum


class Action(str, Enum):
    ALLOW = 'allow'
    DENY = 'deny'
    STEP_UP = 'step_up'


@dataclass(frozen=True)
class Attempt:
    country: str
    ip_count: int
    device_count: int
    phone_count: int
    account_count: int
    country_spend_cents: int
    suppressed: bool
    risk_score: int


@dataclass(frozen=True)
class Policy:
    allowed_countries: frozenset[str]
    denied_countries: frozenset[str]
    country_budget_cents: dict[str, int]
    max_ip: int
    max_device: int
    max_phone: int
    max_account: int
    step_up_score: int


def authorize_send(attempt: Attempt, policy: Policy) -> tuple[Action, str]:
    if attempt.country in policy.denied_countries:
        return Action.DENY, 'country_denied'
    if attempt.country not in policy.allowed_countries:
        return Action.DENY, 'country_not_allowed'

    budget = policy.country_budget_cents.get(attempt.country)
    if budget is None or attempt.country_spend_cents >= budget:
        return Action.DENY, 'country_budget_closed'
    if attempt.suppressed:
        return Action.DENY, 'destination_suppressed'

    counters = {
        'ip_rate_limited': (attempt.ip_count, policy.max_ip),
        'device_rate_limited': (attempt.device_count, policy.max_device),
        'phone_rate_limited': (attempt.phone_count, policy.max_phone),
        'account_rate_limited': (attempt.account_count, policy.max_account),
    }
    for reason, (observed, limit) in counters.items():
        if observed >= limit:
            return Action.DENY, reason

    if attempt.risk_score >= policy.step_up_score:
        return Action.STEP_UP, 'additional_verification_required'
    return Action.ALLOW, 'policy_passed'


def send_sms_otp(payload: dict, idempotency_key: str) -> dict:
    api_key = os.environ['INFRAI_API_KEY']
    api_origin = os.environ['INFRAI_API_ORIGIN'].rstrip('/')
    request = urllib.request.Request(
        f'{api_origin}/v1/sms/otp',
        data=json.dumps(payload).encode('utf-8'),
        headers={
            'Authorization': f'Bearer {api_key}',
            'Content-Type': 'application/json',
            'Idempotency-Key': idempotency_key,
        },
        method='POST',
    )

    for attempt_number in range(4):
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f'Unexpected HTTP status: {response.status}')
                return json.loads(response.read())
        except urllib.error.HTTPError as error:
            body = error.read().decode('utf-8', errors='replace')
            if error.code != 429 or attempt_number == 3:
                raise RuntimeError(f'HTTP {error.code}: {body}') from error
            retry_after = error.headers.get('Retry-After')
            delay = float(retry_after) if retry_after else 2 ** attempt_number
            time.sleep(delay)

    raise RuntimeError('Retry limit reached')


if __name__ == '__main__':
    sample_policy = Policy(
        allowed_countries=frozenset({'US', 'DE', 'FR'}),
        denied_countries=frozenset(),
        country_budget_cents={'US': 5000, 'DE': 3000, 'FR': 3000},
        max_ip=8,
        max_device=5,
        max_phone=3,
        max_account=4,
        step_up_score=70,
    )
    sample_attempt = Attempt(
        country='DE',
        ip_count=2,
        device_count=1,
        phone_count=1,
        account_count=1,
        country_spend_cents=1200,
        suppressed=False,
        risk_score=35,
    )
    action, reason = authorize_send(sample_attempt, sample_policy)
    print(action, reason)
    if action is Action.ALLOW:
        otp_payload = json.loads(os.environ['INFRAI_SMS_OTP_JSON'])
        operation_id = os.environ['OTP_IDEMPOTENCY_KEY']
        print(send_sms_otp(otp_payload, operation_id))
```

Set `INFRAI_API_ORIGIN` to the documented API origin, `INFRAI_SMS_OTP_JSON` to a request object that matches the live discovery schema, `INFRAI_API_KEY` to the bearer key, and `OTP_IDEMPOTENCY_KEY` to the stable identity of the signup operation. Keeping the JSON outside the article avoids freezing undocumented fields into a copyable example. The code sets an explicit method, rejects non-success responses, and retries 429 responses with bounded exponential backoff while honoring `Retry-After`.

In production, the counter check and increment need one atomic operation per dimension, and the final allow record needs the same stable request identity used by the sending client. Reuse that idempotency key so network retries represent one logical action.

Notice what the function does not do: it does not infer country permission from IP alone, it does not reset one counter because another identity changed, and it does not send SMS. That separation makes the policy testable. It also permits a managed verification provider later without moving the business's loss boundary into vendor-specific code.

## Rejected option and the case for using it

For this logistics signup system, the rejected option is provider-owned templates and verification state. It weakens the primary requirement: the application cannot treat the verification link, localized copy, country eligibility, and email fallback as one owned artifact. It is also not suitable when policy changes must ship with the account service or when the same link semantics must survive a delivery-provider change.

Still, rejection is contextual. Stick with Twilio Verify or Vonage Verify when the team explicitly wants a managed verification lifecycle, can accept the provider's template model, and would otherwise implement token state poorly. Stick with AWS messaging when AWS account governance and direct application ownership matter more than a unified backend interface. Use Amazon SES for the email fallback when the application is prepared to own that verification link; it does not remove the need for app-side token logic.

The final decision rule is blunt: choose managed verification to transfer workflow ownership, or choose transport to retain it. In either case, retain per-IP, per-device, per-phone, and per-account limits, suppression checks, country allowlist and denylist rules, country budget cutoffs, and risk-based step-up in the application. Those controls are the toll-fraud boundary. The SMS API is downstream.

Templates are policy too.

## References

- https://www.twilio.com/docs/verify/api
- https://developer.vonage.com/en/verify/overview
- https://docs.aws.amazon.com/sms-voice/latest/userguide/what-is-service.html
- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms
- https://pages.nist.gov/800-63-4/sp800-63b.html
