# How to Build One Key Chatbot API Fallback for Moderation Models

Route marketplace moderation reports through an application-owned decision envelope, and permit fallback only before a model has produced a valid classification. The SaaS application should hold one internal gateway credential; the gateway may hold several upstream credentials, but provider responses must be converted into one versioned schema before anything enters the human-review queue.

**Short answer:** choose the API layer by testing whether it preserves evidence during substitution, not by counting model names on a compatibility page. A fallback that changes category semantics, loses the original report identifier, or silently retries after a valid answer is worse than a visible failure. For report triage, the durable unit is a recorded attempt plus a normalized decision, not a chat transcript.

This is the architecture decision: keep report identity, policy version, prompt version, routing history, and normalized output under application control. Treat provider portability as a bounded adapter problem. Never treat nominally similar chat APIs as proof that the models are behaviorally interchangeable.

Teams often describe the desired SaaS app as one chatbot API key across OpenAI, Anthropic's Claude, and Google's Gemini, with fallback models ready behind it. That description captures credential convenience, but not the moderation contract. Those are three separately operated model families; putting them behind one interface cannot establish equivalent labels, confidence, refusal behavior, or retention terms. Each route remains a candidate until the same governed report set has tested it.

## Should one chatbot API key control fallback models?

The first invariant is identity. One marketplace report receives one application-generated `report_id`, while every upstream call receives a distinct `attempt_id`. If an operator replays a report after changing policy, the replay is a new decision operation linked to the old report; it is not an overwrite. UUIDs are suitable identifiers when generated and handled according to RFC 9562, but the important property here is the separation of business identity from transport attempts.

The second invariant is meaning. A model may return prose, a function call, or structured JSON, yet the review queue accepts only a small internal vocabulary such as `allow`, `escalate`, and `urgent`. The gateway rejects unknown labels. It does not guess. This deliberately narrow contract prevents a provider-specific safety label from leaking into marketplace policy.

The third invariant is evidence. Store the exact request envelope used for the decision, the unmodified response body or a content-addressed reference to it, the selected provider alias, timestamps, and the normalization result. RFC 8785 defines a JSON canonicalization scheme that can support repeatable hashing; without canonicalization, insignificant differences in member ordering or number representation can produce different byte sequences. A hash is an integrity signal, however, not proof that the classification was correct.

Keep four failure boundaries explicit:

1. Authentication and transport failure occur before a usable response exists and may qualify for fallback.
2. Rate limiting and declared temporary unavailability may qualify, subject to a total deadline and retry budget.
3. Schema failure produces evidence but no valid classification; policy must decide whether one alternate attempt is acceptable or whether the report goes directly to a human.
4. A valid but low-confidence or severe classification is a business result. It must not trigger provider fallback merely to seek a friendlier answer.

That last distinction matters. Fast retries can turn model disagreement into accidental policy shopping.

Stop there.

## Decision record and option boundaries

The comparison is between ownership models, not brands. Model catalogs change; these failure boundaries do not.

| Option | Credential boundary | Evidence ownership | Substitution risk | Valid use case |
|---|---|---|---|---|
| Application adapters | Application stores every upstream credential | Full, if implemented deliberately | Adapter drift and duplicated retry logic | Small provider set with strict internal controls |
| Self-hosted gateway | Application uses one internal credential; gateway stores upstream credentials | Full when request and response records use application storage | Gateway upgrades can alter normalization | Teams able to operate and audit a control plane |
| Managed gateway | Application uses one gateway credential | Depends on export, retention, and raw-response access | Contract or routing behavior may constrain replay | Teams that have verified evidence export and isolation requirements |
| Single-provider client | Application stores one upstream credential | Usually straightforward | No cross-provider fallback | Workloads whose failure policy is queueing rather than substitution |

An open-source gateway such as LiteLLM is evidence that a proxy can expose a common interface across multiple model APIs, but its repository is not evidence that any particular pair of models is equivalent. The same caution applies to a managed gateway or handwritten adapter. Evaluate the deployed version and configuration with the marketplace's own labeled reports.

I would reject any option that cannot answer three questions from stored records alone: Which policy and prompt produced this queue item? Which attempts preceded it? Can an auditor verify that the recorded payload has not changed? That is a stricter bar than “one key,” and it should be.

The limitations are concrete. Application adapters place credential rotation, SDK changes, and retry correctness on the product team. A self-hosted gateway adds an operational component whose configuration and upgrades enter the audit scope. A managed gateway may be inappropriate when raw evidence export, storage location, tenant isolation, or retention controls cannot satisfy the application's requirements. A single-provider client gives up cross-provider fallback, yet it is the cleaner choice when the team cannot operate another control plane and can safely hold reports in the human queue during an outage. My decision rule favors evidence ownership because a delayed classification is recoverable; an enforcement decision with an unreconstructable route is not.

## Implement the critical path as a narrow state machine

The following Python program is runnable with the standard library. Its provider functions are injected, so the moderation contract has no dependency on an upstream SDK. The example deliberately permits fallback for transport failure and throttling, but sends malformed model output to human review instead of asking successive models until one returns an agreeable shape.

```python
from __future__ import annotations

import hashlib
import json
import time
import uuid
from dataclasses import asdict, dataclass
from typing import Any, Callable


class TransportFailure(Exception):
    pass


class Throttled(Exception):
    pass


@dataclass(frozen=True)
class Decision:
    report_id: str
    operation_id: str
    attempt_id: str
    provider_alias: str
    policy_version: str
    prompt_version: str
    label: str
    confidence: float
    evidence_hash: str


Provider = Callable[[dict[str, Any], float], dict[str, Any]]
ALLOWED_LABELS = {"allow", "escalate", "urgent"}


def canonical_bytes(value: dict[str, Any]) -> bytes:
    # This is deterministic for this constrained envelope; use an RFC 8785
    # implementation if every JSON edge case must be canonicalized.
    return json.dumps(
        value, sort_keys=True, separators=(",", ":"), ensure_ascii=False
    ).encode("utf-8")


def normalize(raw: dict[str, Any]) -> tuple[str, float]:
    label = raw.get("label")
    confidence = raw.get("confidence")
    if label not in ALLOWED_LABELS:
        raise ValueError("unknown moderation label")
    if isinstance(confidence, bool) or not isinstance(confidence, (int, float)):
        raise ValueError("confidence must be numeric")
    if not 0.0 <= float(confidence) <= 1.0:
        raise ValueError("confidence must be between 0 and 1")
    return label, float(confidence)


def classify(
    report: dict[str, Any],
    providers: list[tuple[str, Provider]],
    *,
    policy_version: str,
    prompt_version: str,
    total_timeout_s: float = 8.0,
) -> tuple[Decision | None, list[dict[str, Any]]]:
    started = time.monotonic()
    operation_id = str(uuid.uuid4())
    attempts: list[dict[str, Any]] = []

    for provider_alias, call_provider in providers:
        remaining = total_timeout_s - (time.monotonic() - started)
        if remaining <= 0:
            break

        attempt_id = str(uuid.uuid4())
        request = {
            "report_id": report["report_id"],
            "operation_id": operation_id,
            "attempt_id": attempt_id,
            "policy_version": policy_version,
            "prompt_version": prompt_version,
            "report": report,
        }
        record: dict[str, Any] = {
            "attempt_id": attempt_id,
            "provider_alias": provider_alias,
            "request": request,
        }

        try:
            raw = call_provider(request, remaining)
            record["raw_response"] = raw
            label, confidence = normalize(raw)
        except (TransportFailure, Throttled) as exc:
            record["outcome"] = type(exc).__name__
            attempts.append(record)
            continue
        except ValueError as exc:
            record["outcome"] = "schema_failure"
            record["reason"] = str(exc)
            attempts.append(record)
            return None, attempts

        record["outcome"] = "accepted"
        evidence_hash = hashlib.sha256(canonical_bytes(record)).hexdigest()
        attempts.append(record)
        return Decision(
            report_id=report["report_id"],
            operation_id=operation_id,
            attempt_id=attempt_id,
            provider_alias=provider_alias,
            policy_version=policy_version,
            prompt_version=prompt_version,
            label=label,
            confidence=confidence,
            evidence_hash=evidence_hash,
        ), attempts

    return None, attempts


def unavailable(_: dict[str, Any], __: float) -> dict[str, Any]:
    raise TransportFailure("connection ended before a response")


def alternate(_: dict[str, Any], __: float) -> dict[str, Any]:
    return {"label": "escalate", "confidence": 0.81}


if __name__ == "__main__":
    sample = {
        "report_id": "report-1842",
        "listing_id": "listing-771",
        "reason": "suspected counterfeit branding",
        "report_text": "Logo and packaging differ from the catalog images.",
    }
    decision, audit_log = classify(
        sample,
        [("primary", unavailable), ("alternate", alternate)],
        policy_version="marketplace-policy-7",
        prompt_version="report-triage-12",
    )
    print(json.dumps({"decision": asdict(decision), "attempts": audit_log}, indent=2))
```

Two implementation details are intentionally severe. There is one operation ID across attempts, which makes the chain reconstructable, and the deadline shrinks rather than resetting for each provider. With an 8-second budget, the second call gets only what the first call leaves behind. Otherwise, three individually reasonable timeouts can create an unreasonable queue delay.

Eight seconds is a policy example, not a universal target.

The code keeps raw evidence in memory only to make the example executable. A production design should write immutable attempt records to durable storage, define retention from legal and marketplace requirements, encrypt sensitive report content, and restrict who can retrieve it. Do not place report text in metrics labels or exception messages. Logs, metrics, and evidence have different access patterns.

## Test substitution before trusting it

A gateway integration test proves that calls move. It does not prove that moderation meaning survives. Build a fixed, access-controlled evaluation set from appropriately governed reports and include sparse descriptions, multilingual text, quoted abuse, prompt-injection strings, and categories whose false negatives demand immediate human attention. NIST's AI Risk Management Framework describes testing, evaluation, verification, and validation as part of managing AI risk; the local test design still has to reflect the harms of this marketplace.

Run each candidate route with fallback disabled first. Record schema-valid rate, label confusion against human-reviewed outcomes, abstention or escalation behavior, and latency distributions. Then inject transport failures and throttling to verify attempt order, total deadlines, and duplicate suppression. A test should fail if two accepted decisions are emitted for one operation.

Use counts with denominators. “Nine disagreements” means little without the number and composition of evaluated reports. Confidence values also require calibration against held-out outcomes before they can set automation thresholds; a decimal emitted by a model is not a portable probability merely because every adapter calls the field `confidence`.

This is the trade-off: stricter substitution rules reduce automatic completion during upstream trouble because they deliberately preserve ambiguity for a reviewer. I choose that failure mode for pre-review moderation because the queue already represents a human decision boundary. A customer-facing conversational assistant, where the immediate response itself is the product, could reasonably choose a different deadline and a broader retry policy.

Deployment should be staged by tenant or deterministic report cohort, with a kill switch that queues reports for people rather than dropping them. Monitor accepted decisions, human overrides, schema failures, fallback attempts, deadline exhaustion, and queue age separately. Provider aliases belong in bounded metric dimensions; report IDs do not.

## Why reject transparent retry after every bad answer?

The rejected design is a general “try the next model on any error” loop. It is attractive because completion rates rise on a dashboard, yet it collapses transport errors, contract violations, safety refusals, and valid adverse classifications into one bucket. It also makes the final answer depend on route order in a way that is hard to explain during an appeal.

There is a valid use case for broader retry: low-stakes drafting where a user sees the generated text, can reject it, and no automated enforcement follows. Marketplace moderation before human review has a different failure cost. Queueing an uncertain report is an available safe state, so the system need not manufacture certainty.

This ADR therefore accepts provider substitution only for enumerated pre-decision failures, under one shrinking deadline, with every attempt attached to the same operation. The practical selection criterion is plain: adopt an API layer only if the team can export, verify, replay, and explain that record without relying on the provider's current dashboard. Portability without evidence is a routing feature, not a moderation architecture.

## References

- https://www.rfc-editor.org/rfc/rfc8785
- https://www.rfc-editor.org/rfc/rfc9562
- https://www.nist.gov/itl/ai-risk-management-framework
- https://github.com/BerriAI/litellm
- https://platform.openai.com/docs/api-reference
- https://docs.anthropic.com/en/api/overview
- https://ai.google.dev/gemini-api/docs
