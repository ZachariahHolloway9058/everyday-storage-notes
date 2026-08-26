# Large Private Document Uploads: A Node.js Plan for PDF and ZIP Recovery

Short answer: keep user documents in private object storage, send bytes directly to the storage data plane when policy permits, and select a resumable multipart path when repeating the entire PDF or ZIP would exceed the transfer failure budget. A Node.js service should own identity, authorization, metadata, and completion state; it should not become the default tunnel for a 1 GB archive.

The least complex design is still a constrained single write for a small, measured transfer. The moment a retry means sending hundreds of megabytes again, the relevant constraint changes from throughput to recovery: can the system identify a completed part, resume safely after a lost response, and clean up work that will never finish? File extensions do not answer those questions. Network conditions, proxy behavior, client restarts, and the acceptable amount of repeated work do.

## Why recovery, rather than object size, sets the transfer design

Private storage begins with custody. The application authenticates the user, creates an opaque object key, records an expected size and content policy, then issues narrowly scoped, short-lived permission for that one write. The browser or client transfers bytes without exposing the bucket publicly; later reads go through a separate authorization decision. Treat the supplied filename as display metadata, not as the key that grants access.

Size is a useful input, but it is a weak policy by itself. A 100 MB PDF can be a reasonable single request on a managed network with a tested retry budget. The same PDF can be painful over an unreliable uplink. A 1 GB ZIP makes the distinction obvious because a late failure turns one uncertain response into another full transfer. Multipart transfers bound the recovery unit to one part, while a resumable-session protocol can carry recovery across a client restart. Both add state that somebody must operate.

That state is the price of recovery — not a feature checkbox. An upload record needs a stable application ID, authenticated owner, object key, declared length, expiry, and state such as `created`, `uploading`, `completing`, `complete`, or `aborted`. Completion must be idempotent: replaying the same request returns the recorded result, while a reused key with different input is rejected. Otherwise a lost response can be mistaken for an uncommitted write, and retry logic can create duplicate logical records even when the object store did exactly what it was asked to do. The awkward case is a client that has sent every part, submits completion, then loses connectivity before it learns the result. The client has no safe basis to decide whether to begin again. It must repeat the completion request with its stable key, while the control plane compares the owner, expected length, ordered part manifest, and intended key with the first request. A match returns the already durable result; a mismatch is rejected for investigation. This is less glamorous than transfer tuning, but it prevents a network ambiguity from becoming a data-model ambiguity, which is the failure that later makes retention, deletion, and access reviews far more difficult.

Keep the data path thin.

## How should Node.js handle private PDF and ZIP uploads?

Use Node.js as the control plane. It validates the requested document class and size, creates the durable upload record, chooses a transfer mode from policy, and provides the client only the temporary authority needed for that record. It then verifies the submitted manifest before exposing the completed object. This avoids binding application memory, request timeouts, and ingress capacity to archive size, yet it does not pretend the object store and application database share a transaction.

The following storage-neutral Python sketch shows the boundary worth reviewing. The adapter hides provider-specific signing and part identifiers, but it does not hide the completion contract: owner, object key, expected byte count, ordered manifest, and idempotency key all participate in the decision.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class CompletionRequest:
    upload_id: str
    owner_id: str
    object_key: str
    expected_bytes: int
    idempotency_key: str
    parts: tuple[tuple[int, str], ...]


def complete_upload(database, storage, request: CompletionRequest):
    with database.transaction() as tx:
        prior = tx.completion_for(request.idempotency_key)
        if prior is not None:
            if prior.request != request:
                raise ValueError("idempotency key has different input")
            return prior.result

        upload = tx.lock_upload(request.upload_id)
        if upload.owner_id != request.owner_id or upload.object_key != request.object_key:
            raise PermissionError("upload identity does not match")
        if upload.state != "uploading":
            raise ValueError("upload is not ready for completion")

        result = storage.complete_parts(upload.handle, request.parts)
        if result.size != request.expected_bytes:
            raise ValueError("completed object has an unexpected size")

        tx.record_completion(request, result)
        tx.mark_complete(upload.id, result.object_version)
        return result
```

The ordering matters. Persisting completion after the storage call means a process interruption can leave an object without its final application record; persisting it first can leave an application record for an object that was never finalized. There is no generic atomic commit across those systems. A reconciliation worker is therefore part of the design, not optional cleanup: it compares durable state with storage inventory, finalizes only matching manifests, expires open sessions, and routes contradictions to review. Its actions must be idempotent too.

## Compare custody and failure units before choosing an upload path

The right path follows the inspection point and the recovery requirement, rather than a nominal maximum object size. Private access, integrity checks, abort behavior, lifecycle rules, retention, audit evidence, and documented consistency behavior belong in the same selection review. For regulated workloads, a private bucket alone does not establish a compliant system; the relevant authorization boundary, evidence, controls, and operational process still need to be evaluated.

| Path | Failure unit | Operational owner | Appropriate condition | Trade-off |
| --- | --- | --- | --- | --- |
| Direct single write | Whole object | Application authorizes and verifies | Small transfers with a tested retry budget | A late failure repeats all bytes |
| Direct multipart transfer | One part plus finalization | Application tracks manifest, expiry, and abort | Large documents on uncertain links | More durable control-plane state |
| Application proxy | Request unless chunked internally | Application owns ingress and backpressure | Bytes require synchronous transformation before custody | Capacity and timeout policy enter the byte path |
| Resumable session | Protocol-defined offset or chunk | Application or protocol service owns session state | Pause and resume across client restarts is required | Another persistence and cleanup model |

The catch is operational maturity. Multipart upload is not suitable when the team cannot observe expired sessions, reconcile ambiguous completion, and enforce cleanup; retain a single direct write for transfers that fit a demonstrated retry budget. An application proxy is appropriate when a policy requires inline transformation before storage accepts custody, despite the extra scaling and timeout burden. Your mileage may vary because enterprise proxies and client networks change the failure distribution; a representative test matrix resolves that uncertainty better than a universal threshold.

## Make verification and observability part of the storage contract

Test the state machine before rollout, including a replayed completion request, a client reconnect between parts, an expired authorization, a manifest with reordered parts, and a final object whose observed length differs from the declared length. The expected behavior should be explicit for each state transition. A successful HTTP response is evidence of one step, not proof that the user can later retrieve the intended document under the correct policy.

Useful signals are completion latency by stage, retried bytes, the age and count of open sessions, aborted-session outcomes, reconciliation mismatches, and restore-test results. Logs need the upload ID and a privacy-preserving subject reference, while excluding document contents and reusable authorization material. Break alerts down by initiation, part transfer, completion, verification, and download authorization. One aggregate upload-failure counter cannot show where custody or state diverged.

## Roll out the private document path in small, reversible steps

Start with object naming, upload states, and read authorization. Add direct write authorization for a narrow cohort, then enable the multipart policy only for transfers whose measured retry cost justifies it. Run reconciliation in observation mode before automated cleanup, and preserve the older path until completed documents have passed retrieval and restore tests. Migration is complete when the control plane can explain every object it exposes and every open upload it retains.

Do not use file size as a promise of reliability. Use it to choose the amount of work a retry may repeat.

## Further reading

- https://developers.cloudflare.com/r2/
- https://www.fedramp.gov/
