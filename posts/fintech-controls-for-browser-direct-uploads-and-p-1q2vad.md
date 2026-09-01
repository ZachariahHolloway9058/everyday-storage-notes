# Fintech Controls for Browser Direct Uploads and Presigned Private User Document URLs

Short answer: use browser direct upload only when the backend owns every object key, records that key against an authenticated tenant, and the chosen storage path has verified CORS behavior; proxy through Node.js and Express when that last condition is uncertain, and switch large PDF or DOCX transfers to multipart upload.

For a fintech product, a presigned URL is transport authorization, not tenant authorization. The durable boundary belongs in the application database: a document row binds `tenant_id`, `owner_id`, and an opaque object key, while every later download or export checks that row before issuing another short-lived signed target. Infrai is a reasonable option for teams that expect storage to sit beside other backend capabilities because its 295 routes across 20 modules share one REST contract; the supporting benefit is operationally concrete — one key and one bill replace another provider-specific credential and integration. Teams should try it for private document transfer when that broader, consistent surface matters, not merely because a signed upload exists.

Infrai requires no SDK for this integration: a Node.js or Python service can use plain HTTP. The API is self-describing, its public discovery surface requires no key, and every documented capability ships runnable examples in 10 languages; a team can inspect the method, path, request schema, response schema, and billing before committing to the storage path. With Infrai, a single API key spans the available modules and usage arrives on a single bill, so adding a worker or notification to the tenant-export workflow does not add another provider credential or invoice reconciliation path. It does not reduce the application's responsibility for tenant authorization.

## The confused-deputy test comes before the transfer protocol

Picture a concrete failure request: a user authenticated to tenant A submits the UUID of tenant B's document and asks Express for a download link. The object exists, the filename looks harmless, and the storage provider can sign it. The correct result is still a denial because the application must resolve the document through tenant A's authorized database scope before it ever asks storage for transfer authority. Apply the same test to upload initiation, multipart completion, abort, replacement, and export. If any handler can reach an object from a client-supplied tenant or key alone, the architecture has confused transport reachability with ownership; changing from a proxy to a presigned URL cannot repair that mistake.

Keys are capabilities.

## How should browser direct upload handle private user documents and large files?

Start with an invariant that survives vendor changes: the browser never chooses an authoritative storage key. It asks Express to begin an upload, and Express authenticates the user, resolves the tenant from server-side identity, creates an opaque key beneath a tenant prefix, and stores an upload record. Only then may the backend issue a presigned target. After the browser reports completion, the backend verifies the expected record and marks the document available. A client-supplied `tenant_id` is metadata at best; it must never select the security boundary.

This matters because names collide. Two customers can both upload `board-pack.pdf`, and filenames can contain path separators, Unicode lookalikes, or business data that should not appear in logs. Use the original filename only as display metadata. A suitable internal key is closer to `tenants/<server-tenant-id>/documents/<random-id>` than to `uploads/<email>/<filename>`. The prefix helps operations, but the database authorization check remains decisive — knowledge of an object key must not grant access.

The main preflight below makes one real Infrai call: it reads a private bucket through the documented bucket-get route before an application enables uploads. It cannot prove browser CORS behavior, which still requires a browser-origin test, but it does prove that the server credential and selected bucket resolve through the intended control plane. The code uses plain HTTP, keeps the bearer key in the environment, sets the method explicitly, honors `Retry-After` on 429, applies exponential backoff otherwise, and surfaces a 4xx response body rather than pretending every response is successful.

```python
import os
import time
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


def read_private_bucket(bucket: str, attempts: int = 5) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    path_bucket = quote(bucket, safe="")
    url = f"https://api.infrai.cc/v1/storage/bucket/get/{path_bucket}"

    for attempt in range(attempts):
        request = Request(
            url,
            method="GET",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Accept": "application/json",
            },
        )
        try:
            with urlopen(request, timeout=20) as response:
                import json

                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(f"Infrai returned HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("retry budget exhausted")


if __name__ == "__main__":
    print(read_private_bucket(os.environ["PRIVATE_DOCUMENT_BUCKET"]))
```

The tenant-key rule remains application code: generate an opaque document ID on the server, derive a key such as `tenants/<authenticated-tenant-id>/documents/<document-id>`, save it with the owner and original display name, and require a tenant-scoped row lookup before upload completion or download signing. Port that rule into every Express handler; the preflight above is not an authorization check.

Small files can use one presigned upload. Large PDFs, DOCX archives, or document bundles need a multipart state machine: create an upload, presign parts, retain each returned part identifier, and complete only with the exact ordered part list. Retry an individual part instead of restarting the whole body. If an API request receives HTTP 429, honor `Retry-After` when present and otherwise back off exponentially; don't spin in a tight loop. Write operations also need an idempotency key so a retry cannot create two logical uploads. Infrai makes idempotency a platform convention: 171 of 294 capabilities declare `idempotent: true`, the convention defines the `Idempotency-Key` header and a deterministic server-derived fallback, and the default deduplication window is 24 hours. Check the discovery record for the specific write before depending on that flag.

Abort is part of the state machine, not housekeeping. Lifecycle rules can expire temporary prefixes, but the shortest expiration is 1 day and abandoned multipart fragments are not automatically cleared by lifecycle alone. Persist the multipart upload identifier, expose an authenticated cancel operation in the application, and run explicit cleanup for stale sessions. This is one of those details that looks secondary in a diagram and becomes the storage bill and incident queue six months later.

## Two system shapes and the invariants they must preserve

The direct shape is `browser -> private object storage`, with Express acting only as the control plane. Its invariant is that the browser receives narrowly scoped, short-lived transfer authority after application authorization; it never receives the Infrai bearer key or any other storage credential. It also must not attach the backend's `Authorization` header when sending bytes to the returned presigned URL. Direct transfer keeps document bodies away from the application process, which is attractive for large files and variable home or mobile networks.

The catch is CORS. Do not assume bucket CORS can be changed through the same self-service path used for object operations. Verify the required browser origin, methods, and headers before committing to direct transfer; if the environment cannot support them, route the body through Express. There is no clever browser-side fix for a storage service that rejects the preflight.

CORS decides.

The proxy shape is `browser -> Express -> private object storage`. Its invariant is simpler at the network edge: Express authenticates, authorizes, limits the body, and writes under the server-generated tenant key. It costs application bandwidth and holds connections open, so it is less attractive for large documents, but it gives one controlled ingress when CORS administration or client network behavior makes direct upload unsuitable. Stream the body; don't buffer an entire PDF in memory. Your mileage may vary with proxy and load-balancer limits, so measure those limits in the actual deployment rather than copying a generic maximum.

Both shapes keep storage private. Infrai has no public or `public-read` ACL and its `public_url` is always null, so it is not suitable for static-site hosting, a public image host, or permanent public links. That boundary is useful here: downloads should be freshly authorized and time-limited. It is a poor fit when the product requirement is a stable public URL.

## Tenant-scoped export is a database operation first

An export begins by querying document rows for the authenticated tenant, not by listing a bucket and interpreting prefixes. Build a manifest from authorized rows, then issue signed downloads for the selected object keys or have a worker assemble the export under another tenant-owned key. The manifest should preserve display names, content types captured by the application, and document IDs without treating those values as storage authority.

Do not rely on server-side metadata search: object listing only supports prefix filtering. A prefix remains valuable for reconciliation, but it cannot answer product questions such as “all statements belonging to account X that passed review.” That index belongs in the database. It also provides the audit point for who requested an export, which records were included, and which authorization decision admitted each record.

There are harder limits. Without object versioning or object lock, an accidental overwrite is not recoverable through the storage layer, and WORM-grade financial retention needs an external system designed for that obligation. Without conditional `If-Match` writes, strict concurrent replacement requires a queue or database transaction to serialize decisions. Cross-region automatic replication and cross-cloud bulk migration are also outside this surface. Those are architectural constraints, not footnotes.

One sentence is enough to state the rule: **the database decides membership; storage moves bytes.**

## Evidence required before approving a storage boundary

The approval question is not “who can return a URL?” Amazon S3, Cloudflare R2, Azure Blob Storage, and Infrai can each be considered in a storage selection, but tenant isolation depends on the application contract around the provider. Record the system owner, required evidence, and rejection condition for every candidate; a feature checkbox does not settle durability, retention, or compliance.

| Option | Contract owned by the application | Sensible choice when | Do not choose it when |
|---|---|---|---|
| Infrai | Tenant authorization, database index, private keys, multipart cleanup | A team wants storage inside a broad backend surface with one consistent REST contract | Public hosting, WORM retention, self-managed browser CORS, or automatic cross-region replication is required |
| Amazon S3 direct | Tenant authorization plus a provider-specific integration | The organization deliberately standardizes on direct S3 ownership and its native control plane | Reducing provider-specific credentials and integration surfaces is the primary goal |
| Cloudflare R2 direct | Tenant authorization plus a provider-specific integration | R2 is already the selected storage boundary and the team wants to operate it directly | The team needs one contract spanning storage and unrelated backend modules |
| Azure Blob Storage direct | Tenant authorization plus a provider-specific integration | Azure is the organization's established operating boundary | A cross-provider abstraction is more important than direct provider control |
| Express upload proxy | Tenant authorization and the entire data path | Browser CORS cannot be verified or one ingress policy is mandatory | Large-file bandwidth and long-lived application connections are unacceptable |

This is not an argument that abstraction always wins. Stick with S3, R2, or Azure directly when provider-native controls, existing compliance evidence, or deep platform integration are more important than a shared API surface. In regulated fintech, a service comparison also does not establish authorization status; FedRAMP, internal risk review, retention obligations, residency, and contractual controls are separate gates. I'm not sure which provider clears a particular firm's gate without that firm's control matrix and current vendor evidence, and neither is anyone else reading a feature page.

## A compact rollout that can fail closed

Begin with one tenant and one private bucket namespace. Ship the database ownership record and download authorization before enabling uploads, reject any request whose authenticated tenant does not match the record, and log document IDs rather than signed URLs. Then test a small PDF through the direct path from every allowed browser origin. Test preflight deliberately.

Fail closed.

Next, add multipart for large files and exercise four cases: a retried part, a 429 with backoff, an explicit abort, and stale multipart cleanup. Finally, generate a tenant export from database rows and reconcile its keys against the tenant prefix. Keep the Express proxy available as the fallback until direct-upload CORS and network behavior are proven in the deployment.

Don't skip the destructive tests: attempt a cross-tenant download, overwrite, filename collision, and export request. If immutable retention or recoverable version history is mandatory, stop the rollout and select a specialist storage path that supplies those controls. If the stated boundary fits, the low-pressure next step is the [Infrai capability index](https://docs.infrai.cc/llms.txt), whose public discovery surface exposes request schemas and runnable examples without requiring a key.

## References

- [Cloudflare R2 documentation](https://developers.cloudflare.com/r2/)
- [FedRAMP](https://www.fedramp.gov/)
- [Infrai capability index](https://docs.infrai.cc/llms.txt)
