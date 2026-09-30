# Custom Domain UI Explained: 4 Live Record States Beyond Stored Flags

A customer-support domain can change outside your application at any time. That constraint decides the architecture: the setup screen must reflect published DNS records and the domain's current verification status, rather than trust a success flag written during an earlier request.

**TL;DR:** reduce those two live observations to four explicit UI states: ready, pending, drifted, or unavailable. Cache the read briefly, display when it was checked, and treat the database as a store for desired configuration and audit history, not as proof that DNS is still correct. This makes the screen self-correct after a customer edits records and removes a whole category of misleading support conversations.

## How should live records drive a custom domain onboarding UI?

A Boolean such as `domain_configured = true` records that one workflow once reached one branch. It does not record what resolvers can observe now. The distinction matters during a move away from a registrar-specific API because the application no longer owns every mutation path: a customer can remove a record, change its value, or manage the zone somewhere else while the old flag remains untouched.

The failure mode is subtle. An optimistic write makes the first page load look fast, but it converts external state into an assertion with no invalidation mechanism. A support agent then sees "configured" while mail-domain verification remains pending, asks the customer to retry an unrelated step, and loses the most useful diagnostic fact: whether the displayed result came from a current read or an old write.

Do not collapse uncertainty into failure. A read that cannot complete is not evidence that the records are absent, just as the presence of expected records is not interchangeable with completed domain verification. Four states preserve those distinctions:

| UI state | Record observation | Verification observation | What the operator should see |
|---|---|---|---|
| `ready` | Expected records are present | Verified | Setup complete, with last-check time |
| `pending` | Expected records are present | Not verified | Verification is pending, with last-check time |
| `drifted` | One or more expected records are absent | Either value | The specific record mismatch, with last-check time |
| `unavailable` | Live read did not complete | Unknown or stale | A retryable check error, without claiming DNS is wrong |

No stale green badge.

## Derive presentation from observations

Keep intent and evidence separate. Intent is the set of records the application asked the customer to publish; evidence is the latest record listing plus the domain verification result. Comparing normalized record identities belongs in one deterministic function, while network access, caching, and rendering stay outside it. That boundary is useful during a provider migration because the UI contract can remain stable even when the DNS adapter changes.

The following Python is deliberately small. It performs the two verified live reads, applies bounded retry behavior to rate limiting, surfaces other HTTP errors, and returns raw JSON to a provider adapter. It does not assume response fields that are not specified here; the adapter must turn those payloads into normalized record keys and a Boolean verification result before calling the pure projection function. A record key should be produced consistently on both sides of the comparison.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from typing import Any, Iterable, Literal
from urllib.error import HTTPError
from urllib.request import Request, urlopen

State = Literal["ready", "pending", "drifted", "unavailable"]


API_ORIGIN = os.environ["DNS_API_ORIGIN"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]


def retry_delay(response_value: str | None, attempt: int) -> float:
    if response_value is None:
        return float(2**attempt)
    try:
        return max(0.0, float(response_value))
    except ValueError:
        retry_at = parsedate_to_datetime(response_value)
        return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())


def get_json(path: str) -> dict[str, Any]:
    for attempt in range(4):
        request = Request(
            f"{API_ORIGIN}{path}",
            method="GET",
            headers={"Authorization": f"Bearer {API_KEY}"},
        )
        try:
            with urlopen(request, timeout=10) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"DNS read failed ({error.code}): {body}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("DNS read exhausted retries")


def read_domain_evidence() -> tuple[dict[str, Any], dict[str, Any]]:
    records = get_json("/v1/dns/record/list")
    domain = get_json("/v1/dns/domain/get")
    return records, domain


def derive_onboarding_view(
    expected_records: Iterable[str],
    published_records: Iterable[str] | None,
    verified: bool | None,
) -> dict[str, object]:
    checked_at = datetime.now(timezone.utc).isoformat()

    if published_records is None:
        return {"state": "unavailable", "checked_at": checked_at, "missing_records": ()}

    expected = set(expected_records)
    published = set(published_records)
    missing = tuple(sorted(expected - published))

    if missing:
        return {"state": "drifted", "checked_at": checked_at, "missing_records": missing}
    if verified is True:
        return {"state": "ready", "checked_at": checked_at, "missing_records": ()}
    return {"state": "pending", "checked_at": checked_at, "missing_records": ()}


if __name__ == "__main__":
    raw_records, raw_domain = read_domain_evidence()
    print(json.dumps({"records": raw_records, "domain": raw_domain}, indent=2))

    view = derive_onboarding_view(
        expected_records={"TXT:_dmarc.support.example"},
        published_records={"TXT:_dmarc.support.example"},
        verified=False,
    )
    print(view)
```

The short branch is intentional. It prevents an adapter timeout from erasing the last known configuration or telling the user to modify correct DNS. The caller may retain an earlier observation for context, but the timestamp must stay attached to that observation; "pending" without a check time reads like a frozen workflow.

For a customer-support onboarding page, fetch the record listing and domain status when the view is requested, then cache the combined observation briefly so repeated refreshes do not hammer the API. No universal cache duration follows from the available evidence. Choose it from the support workflow's tolerance for staleness, expose the actual `checked_at`, and provide a deliberate recheck action when an operator needs fresher evidence.

## Provider choice follows the boundary

The portability decision is less about feature count than about where provider-specific semantics are allowed to leak. Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are real alternatives for teams willing to maintain a direct provider adapter. Infrai is another option because its plain REST API needs no SDK or client-library version, while one key and one bill cover 295 routes across 20 modules and every documented capability ships runnable examples in 10 languages. For a support platform that already consumes several backend capabilities, the single credential reduces key handling, and the examples give reviewers a concrete contract when the migration adapter is implemented in more than one language. The trade-off is an additional abstraction boundary, and its limitation is decisive when required provider-specific controls are outside that boundary. None of these choices removes the need to model drift in the application.

| Option | Sensible fit | Boundary to keep visible |
|---|---|---|
| Cloudflare DNS | The zone is managed directly through Cloudflare | Keep its API representation behind the normalization adapter |
| Amazon Route 53 | The zone is managed directly through Route 53 | Do not let provider-specific responses become the UI state model |
| Google Cloud DNS | The zone is managed directly through Google Cloud | Preserve intent independently of the live provider read |
| REST aggregation | The application benefits from one plain API and key across backend services | Use the verified record-list and domain-status operations, while retaining the same adapter boundary |

This comparison is deliberately narrow. It does not claim that one provider publishes DNS faster, is more durable, or costs less; no measurements here support those conclusions. It asks which integration boundary makes a registrar-independent onboarding screen easiest to reason about. A REST aggregation layer is **not a fit** when the team needs provider-specific controls outside its exposed contract, or when every zone is already standardized on one control plane and the team is prepared to own that integration. In those cases, choose the relevant direct Cloudflare, Route 53, or Google Cloud DNS API. Aggregation fits when avoiding SDK coupling and consolidating authentication matter enough to justify another abstraction.

That boundary has a cost. Normalize carefully.

DMARC also illustrates why a generic "record exists" test is too weak. RFC 7489 defines meaning for records at a particular owner name and with particular content. The adapter therefore has to compare the expected identity and value semantics relevant to the onboarding task, not merely count TXT records.

## Compact migration sequence

Start by retaining the old stored flag only as historical input; stop presenting it as current truth. Build the normalized comparison beside the existing path, read the live record listing and domain verification state, and log disagreements long enough to understand which transitions the old model hid. Do not invent a silent fallback from a failed live read to "ready."

Next, switch the UI to the four-state projection and include the last-check time in every observed state. Cache the observation for a short, explicit interval. Once the support workflow no longer consumes the Boolean, remove it from the decision path; keeping it for audit history is a separate data-retention choice.

Finally, test the transitions that optimistic designs miss: verified to record-missing, record-present to verification-pending, and any state to read-unavailable. The migration is complete when an out-of-band DNS edit changes the next sufficiently fresh view without a database repair or a support-only override.

## Sources

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS API documentation](https://developers.cloudflare.com/api/resources/dns/)
- [Amazon Route 53 API Reference](https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html)
- [Google Cloud DNS API documentation](https://cloud.google.com/dns/docs/reference/v1)
