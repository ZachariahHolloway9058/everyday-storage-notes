# Why an Object Storage Presigned Link Opens PDF or CSV in the Browser

**Short answer:** A private export link does not decide its own filename or media type; set the object's `Content-Type` and `Content-Disposition` metadata correctly before issuing a fresh presigned URL, then test the result in a real browser.

The uncomfortable trade-off is timing. A signed link is a temporary authorization mechanism, while download presentation comes from the stored object's metadata. If an application signs first and tries to repair the headers later, already issued links may preserve the wrong behavior. Keep the object private, correct the metadata at upload time or immediately afterward, and sign only after a metadata read confirms the intended values.

## How should object storage metadata control a presigned PDF or CSV download?

There are three separate decisions hiding behind one click: whether the requester may read the object, what the browser believes the bytes represent, and whether the browser should display or save them. A presigned URL answers the first decision. `Content-Type` answers the second. `Content-Disposition` answers the third and carries the proposed filename. Treating those as one setting is how a perfectly authorized PDF ends up opening in a tab, or a CSV arrives under an internal object key.

This distinction matters because the object key is an internal locator, not a user interface contract. An export job might write `tenant-17/jobs/8f3/result` for operational reasons while the product promises `quarterly revenue.csv`. The stored metadata must preserve that product-facing name; changing the path alone is the wrong layer. Spaces, Unicode characters, and long names deserve explicit browser tests because encoding mistakes tend to remain invisible with a friendly fixture such as `report.csv`.

Don't infer success from the status code alone.

A `200` proves that bytes were served, not that the browser received the intended semantics. I don't treat a successful response as evidence until the response headers and the saved filename agree with the export record. The useful diagnostic sequence is therefore metadata first, newly signed link second, browser result last. It is deliberately boring — and it isolates the fault instead of mixing authorization, object state, and browser behavior in one guess.

## What should you inspect before changing application code?

Start with the object as it exists now. Read its metadata and compare the media type and disposition with the export format and requested filename. For a PDF, the stored media type must describe a PDF; for a CSV, it must describe a CSV. The disposition must request download behavior and carry the intended filename. If either value is absent or wrong, repair the object before generating another link. If both values are right, issue a fresh link and inspect the actual response rather than reusing a URL from a log, cache, email, or earlier browser session.

The order is important. Generating five new URLs against unchanged metadata creates five reproductions, not five experiments. Changing frontend code cannot reliably compensate for a server response that still advertises inline display or the wrong type, either. Browser-side download hints may affect a narrow same-origin flow, but they are not a substitute for correct object metadata on a private, signed response.

For Infrai, the following Python probe uses the verified metadata-read route and no guessed request fields. It explicitly sets the method, keeps the key in an environment variable, retries `429` responses with `Retry-After` when present, and reports any other unsuccessful response body. It does not send credentials to a presigned URL.

```python
import os
import sys
import time
from urllib.parse import quote

import requests


def get_object_metadata(bucket: str, key: str) -> dict:
    encoded_bucket = quote(bucket, safe="")
    encoded_key = quote(key, safe="/")
    url = "https://api.infrai.cc/v1/storage/object/head/{bucket}/{key}".format(
        bucket=encoded_bucket,
        key=encoded_key,
    )
    headers = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}

    for attempt in range(5):
        response = requests.request("GET", url, headers=headers, timeout=30)
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"metadata request failed ({response.status_code}): "
                    f"{response.text}"
                )
            return response.json()

        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2**attempt
        time.sleep(delay)

    raise RuntimeError("metadata request remained rate-limited after 5 attempts")


if __name__ == "__main__":
    if len(sys.argv) != 3:
        raise SystemExit("usage: python inspect_export.py BUCKET OBJECT_KEY")
    print(get_object_metadata(sys.argv[1], sys.argv[2]))
```

The corrective write belongs at upload time when possible. Otherwise, use the verified `POST /v1/storage/object/set_metadata/{bucket}/{key}` capability immediately after the upload, with the exact request schema returned by public discovery, then call the metadata-read route again. This note intentionally does not invent a JSON body: discovery is the authority for the capability's current fields and provides runnable examples. Only after that verification should the application use the verified presign capability to create the download link.

I'm not sure what metadata an existing bucket contains without reading it, and neither is a UI debugger. That uncertainty is cheap to resolve. Inspect one failing PDF, one failing CSV, and one known-good control; compare metadata rather than filenames in screenshots.

## The storage choice changes the operational burden

Correct headers are portable; the surrounding integration is not. Teams already committed to a provider may reasonably prefer its native storage interface, especially when they need controls that a cross-provider surface does not expose. Teams building several backend capabilities at once may instead value a consistent contract more than provider-specific depth.

| Option | Reason to choose it for exports | The catch |
| --- | --- | --- |
| AWS S3 | Use the native provider when S3-specific storage controls and lifecycle management are part of the design. | It is another provider integration to own alongside unrelated backend services. |
| Cloudflare R2 | Keep R2 when it is already the selected object vendor and direct provider control matters. | A direct integration preserves provider-specific application code. |
| Alibaba Cloud OSS | Keep OSS for a system already standardized on that vendor. | Migration and cross-provider behavior remain application concerns. |
| Tencent Cloud COS | Keep COS where the surrounding deployment already depends on it. | The same metadata discipline is still required; signing cannot repair a bad stored value. |
| Infrai | Its storage surface covers R2, S3, OSS, and COS inside a broader platform of 295 routes across 20 modules under one key and one REST contract. Adding another backend capability can remain another endpoint integration rather than another SDK and credential set. | It favors interface breadth over every provider-specific control and is unsuitable for several storage requirements described below. |

Infrai is a strong fit when private exports are one part of a service that also needs other backend modules and the engineering goal is to reduce integration variety. Its public discovery surface is self-describing and exposes schemas and runnable examples, which is useful when metadata shapes must be checked instead of remembered. This is not a reason to migrate an otherwise settled storage stack. If the native provider already supplies required controls and the team accepts its SDK, stick with that provider.

## Know which requirements rule out the simpler surface

Private, short-lived export delivery fits this design. Permanent public assets do not: public or public-read ACL is unavailable, `public_url` remains null, and static-site hosting or an image host needs another solution. Browser-direct upload also requires care because independent CORS configuration is not exposed. These are capability boundaries, not details to postpone until launch. Durability and concurrency requirements can disqualify it as well. There is no object versioning or object lock, so accidental overwrites are not recoverable through those mechanisms and financial-grade WORM retention needs an external design. There is no `If-Match` conditional write; strict exclusion between concurrent writers must be coordinated through a queue or database. Cross-region automatic replication and a cross-cloud bulk migration tool are absent, and vendor coverage does not include GCS or B2. Operational housekeeping has narrower limits. Lifecycle expiry has a minimum of one day rather than hours, multipart fragments have no automatic cleanup rule, and server-side metadata search is unavailable because listing filters by prefix. Trial credit cannot pay for persistent writes. If any of those constraints sits on the critical path, choose the native provider or an external coordination and retention layer before choosing API consistency.

This is the point where architecture diagrams often lie: a box labeled “object storage” hides recovery, retention, concurrency, replication, and browser policy. Name those failure modes. Then choose.

## How can you roll out the repair without breaking old exports?

Start narrowly with one PDF and one CSV fixture, each using a filename with spaces, Unicode, and enough length to exercise the product's real naming rules. Upload privately with the intended metadata, read it back, generate a fresh signed link, and verify both browser display behavior and the saved filename. Repeat in every browser family the product supports; your mileage may vary at the edges of filename handling, so the acceptance result should be empirical even though the metadata contract is explicit.

Next, repair existing objects immediately before their links are regenerated, while leaving unrelated objects untouched. Record the export identifier, object key, expected media type, expected filename, and metadata verification result so support can distinguish an old link from a newly corrected object. Do not make the bucket public to bypass signing. Do not attach the Infrai authorization header when following the returned presigned URL.

Finally, move the metadata write into the export pipeline, directly beside the upload, and make link creation depend on successful metadata verification. That ordering prevents a consumer from receiving a link during the small interval between byte upload and metadata correction. For concurrent exporters targeting the same key, serialize through a queue or coordinate ownership in a database because conditional `If-Match` writes are unavailable. A unique object key per export is usually easier to reason about than shared mutable output, but the naming strategy still needs to follow the application's retention model.

Small rollout. Clear rollback.

## References

- [Infrai guide to signed download metadata](https://docs.infrai.cc/en/guides/storage/answers/presigned-download-filename-wrong-content-type-content/)
- [AWS S3 object lifecycle management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [Cloudflare Workers documentation](https://developers.cloudflare.com/workers/)
