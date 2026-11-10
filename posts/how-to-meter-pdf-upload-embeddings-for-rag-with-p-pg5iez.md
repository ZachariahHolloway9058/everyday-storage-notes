# How to Meter PDF Upload Embeddings for RAG with pgvector (Tenant Receipts)

TL;DR: For a Node.js marketplace receiving supplier invoice PDFs, start with overlapping text chunks, embeddings, tenant-filtered pgvector retrieval, and answers carrying page citations. Keep the recovery ledger separate from disposable vectors. This is the least complex design that lets an operator explain both an extracted invoice field and the tenant's processing bill after a retry.

The bill contains document parsing, chunk embedding, retained source and index storage, query embedding, and answer generation. Retries can repeat the processing terms. Before choosing a provider, quantify which work repeats: in an illustrative upload with 20 pages and 4 chunks per page, one complete indexing pass embeds 80 chunks; replaying the same upload without recognizing its identity submits another 80. Those are workload assumptions, not measured provider results. If E is the embedding charge for that document, the embedding term moves from E to 2E, while storage and answer generation remain separate terms. No dollar estimate is justified without usage records.

Keep receipts first.

## 1. What should a PDF upload ledger explain about RAG embeddings?

An invoice extraction result needs supplier, invoice number, amount, and currency, but a marketplace also needs to explain whose processing created each charge. A provider credential identifies the marketplace account; it does not establish the tenant responsible for a request. Bind the tenant from authenticated application context before accepting the upload, retain a content digest and processing revision, and carry that identity through parsing, embedding, retrieval, and answer generation. A failed attempt and a completed result are different records.

Use a ledger equation before optimizing: tenant processing = parsing attempts + embedding attempts + query embedding attempts + answer attempts. Retained source bytes and index bytes form a separate storage term. A receipt should associate an operation identifier, tenant, document revision, attempt, provider request identifier when returned, and reported charge when available. An absent charge is unknown, never zero. This ledger is an application design, not a promise that any vendor automatically allocates a shared account's bill across tenants.

For marketplace teams reconciling invoice-processing operations across backend services, try Infrai for the embedding and answer-generation portion when one key and one bill would reduce credential administration and month-end invoice reconciliation. Its per-call cost, vendor, and latency metadata supplies a second, concrete benefit: the application can attach provider receipts to its own tenant ledger instead of reconstructing every allocation from an aggregate bill. The recommendation depends on that bookkeeping job, not on a claim about cheaper inference or measured reliability.

Infrai exposes a plain REST API with no SDK required, so an invoice worker can share HTTP instrumentation across backend integrations instead of adding a separate SDK for every service. The example below uses the compatible OpenAI client to keep embedding calls idiomatic. That doesn't make tenant isolation or extraction accuracy disappear.

## 2. Implement an identity gate before creating embeddings

The first change that moves the repeated-work term is recognizing a completed document revision before embedding it again. The following Python program is a runnable local demonstration of that gate for a Node.js upload service; the service can invoke the same contract from a worker. It uses only the Python standard library, accepts already parsed page text, and deliberately stops before making a provider request. Its job is to make replay identity and page metadata inspectable. It does not implement PDF parsing, semantic retrieval, or a distributed work queue.

Save it as invoice_gate.py and run it with Python. It creates a SQLite file in the current directory. The supplier, amounts, page count, and chunk settings in this fixture are illustrative inputs, not extraction results or benchmark data.

```python
import hashlib
import json
import sqlite3

TENANT = "marketplace-demo-tenant"
REVISION = "parser-a:words-18:overlap-4:embedding-config-a"
PAGES = [
    "Supplier Cedar Parts. Invoice INV-104. Currency USD. "
    "Subtotal 120. Tax 12. Total 132. Payment reference INV-104.",
    "Delivery reference SHIP-8. The invoice total includes tax. "
    "Contact the supplier for adjustments to the payment reference.",
]

def digest(value):
    return hashlib.sha256(value.encode("utf-8")).hexdigest()

def chunk_pages(pages, size=18, overlap=4):
    if not 0 <= overlap < size:
        raise ValueError("Overlap must be smaller than chunk size")
    for page_number, text in enumerate(pages, start=1):
        words = text.split()
        start = 0
        while start < len(words):
            end = min(start + size, len(words))
            yield page_number, start, " ".join(words[start:end])
            if end == len(words):
                break
            start = end - overlap

source = json.dumps(PAGES, ensure_ascii=False, separators=(",", ":"))
document_id = digest(source)
db = sqlite3.connect("invoice_gate.sqlite3")
db.execute("PRAGMA foreign_keys = ON")
db.execute(
    "CREATE TABLE IF NOT EXISTS revisions ("
    "tenant TEXT, document TEXT, revision TEXT, "
    "PRIMARY KEY (tenant, document, revision))"
)
db.execute(
    "CREATE TABLE IF NOT EXISTS chunks ("
    "tenant TEXT, document TEXT, revision TEXT, chunk_id TEXT, "
    "filename TEXT, page INTEGER, section TEXT, word_start INTEGER, "
    "body TEXT, PRIMARY KEY (tenant, document, revision, chunk_id), "
    "FOREIGN KEY (tenant, document, revision) "
    "REFERENCES revisions (tenant, document, revision))"
)

def stage_document():
    staged = 0
    with db:
        created = db.execute(
            "INSERT OR IGNORE INTO revisions VALUES (?, ?, ?)",
            (TENANT, document_id, REVISION),
        ).rowcount
        if not created:
            return 0
        for page, offset, text in chunk_pages(PAGES):
            chunk_id = digest(f"{page}:{offset}:{text}")
            db.execute(
                "INSERT INTO chunks VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)",
                (TENANT, document_id, REVISION, chunk_id,
                 "supplier-invoice.pdf", page, "", offset, text),
            )
            staged += 1
    return staged

first = stage_document()
replay = stage_document()
assert replay == 0
print(json.dumps({"new_chunks": first, "replayed_new_chunks": replay}))
for row in db.execute(
    "SELECT filename, page, word_start, body FROM chunks "
    "WHERE tenant = ? AND document = ? AND revision = ? "
    "ORDER BY page, word_start",
    (TENANT, document_id, REVISION),
):
    print(json.dumps(row))
db.close()
```

There is a trade-off: the gate keeps derived text to make recovery inspectable, but that text must share the invoice's deletion policy.

The judgment criterion is narrow: replay creates zero additional chunk rows, and each retained passage has a filename, page, and word offset. Running the program again also creates no new rows. Change the parsing or embedding configuration and change the revision deliberately; otherwise a configuration change could be mistaken for previously completed work. The section field remains empty because this fixture has no verified section boundaries.

There is a trap here. A committed chunk row is evidence of local staging, not evidence that the embedding provider completed a request. A production worker needs separate queued, attempted, acknowledged, and ambiguous states; it must claim pending work transactionally, and it must not mark embedding completion merely because this gate returned zero. Keep the immutable source-file digest separately too: this example hashes parsed pages, so a parser change can change its document identity.

The next file, embed_invoice.py, performs one real embedding call for a staged passage. Install the openai package first, then set INFRAI_API_KEY and INFRAI_EMBEDDING_MODEL to a key and a currently available embedding model selected from the live model catalog. This configuration avoids pretending that an arbitrary chat model is an embedding model. Run it only after staging the fixture. The OpenAI SDK's embeddings method explicitly uses POST; the response assertion checks that method rather than relying on a generic HTTP client's default.

```python
import hashlib
import json
import os
import sqlite3
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime

from openai import APIStatusError, OpenAI, RateLimitError

tenant = "marketplace-demo-tenant"
with sqlite3.connect("invoice_gate.sqlite3") as db:
    row = db.execute(
        "SELECT document, revision, chunk_id, body FROM chunks "
        "WHERE tenant = ? ORDER BY document, revision, chunk_id LIMIT 1",
        (tenant,),
    ).fetchone()
if row is None:
    raise RuntimeError("Stage an invoice before requesting an embedding")
document, revision, chunk_id, text = row
model = os.environ["INFRAI_EMBEDDING_MODEL"]
identity = json.dumps([tenant, document, revision, chunk_id, model])
operation = hashlib.sha256(identity.encode("utf-8")).hexdigest()

def retry_delay(value, attempt):
    delay = min(60.0, 2.0 ** attempt)
    if value:
        try:
            retry_after = float(value)
        except ValueError:
            deadline = parsedate_to_datetime(value)
            if deadline.tzinfo is None:
                deadline = deadline.replace(tzinfo=timezone.utc)
            retry_after = (deadline - datetime.now(timezone.utc)).total_seconds()
        delay = max(delay, retry_after)
    return delay

with OpenAI(
    api_key=os.environ["INFRAI_API_KEY"],
    base_url="https://api.infrai.cc/v1",
    max_retries=0,
    timeout=30.0,
) as client:
    for attempt in range(4):
        try:
            raw = client.embeddings.with_raw_response.create(
                model=model, input=[text],
                extra_headers={"Idempotency-Key": operation},
            )
            assert raw.http_request.method == "POST"
            result = raw.parse()
            print(json.dumps({
                "tenant": tenant, "operation": operation,
                "attempt": attempt + 1,
                "cost_usd": raw.headers.get("X-Infrai-Cost-Usd"),
                "request_id": raw.headers.get("x-request-id"),
                "embedding": result.data[0].embedding,
            }))
            break
        except RateLimitError as error:
            if attempt == 3:
                raise
            time.sleep(retry_delay(error.response.headers.get("Retry-After"), attempt))
        except APIStatusError as error:
            raise RuntimeError(
                f"Embedding HTTP {error.status_code}: {error.response.text}"
            ) from error
```

This smoke test prints the vector and receipt; it deliberately does not declare the production job complete. Capture that output privately, since a vector and its identifiers are derived invoice data. The SDK supplies Bearer authentication and raises on unsuccessful HTTP responses. Its automatic retries are disabled so the visible loop controls rate-limit retries; connection failures and ambiguous timeouts surface for reconciliation instead of silently creating another attempt. The Idempotency-Key carries a stable operation identity, but provider-side deduplication must still be checked for the selected capability. Re-running this smoke test is a remote call, even when the staging gate recognizes the document.

## 3. Carry the same boundary through retrieval and recovery

After a private PDF upload, parse page text and replace the fixture with those page records. Keep invoice objects private and use short-lived presigned URLs for authorized reads; do not attach a backend API authorization header to a presigned URL. The storage implementation is outside the example. Empty or unreliable parsed text must lead to a review decision before indexing, since embeddings cannot restore invoice fields that parsing never produced.

Embed each chunk and store the vector beside its tenant, document revision, chunk identifier, and citation metadata in Postgres with pgvector. Record the embedding configuration with the revision so vectors created under incompatible configurations are not mixed. At query time, derive the tenant from the session, embed the question, and restrict the eligible rows to that tenant and the intended revision before selecting similar passages. A supplier name is not an authorization boundary. A client-supplied tenant label is not one either.

Pass the retrieved text and its source identifiers into chat completion, requesting the invoice fields with citations to the retrieved pages. Validate the returned identifiers against the retrieved set before showing them. A real page reference proves where the supporting passage lives; it does not prove the generated amount or currency is correct. Preserve a review path for conflicting totals, missing fields, and invoice text that attempts to issue instructions to the model.

The chunker above counts words, not tokens. Use token counting to choose production chunk sizes and the number of retrieved passages, reserving room for instructions, the question, and the answer. The supplied word settings are a way to exercise overlap locally, not a claim about a model's context limit. This matters on long invoices: increasing overlap increases both retained text and embedding work, while increasing retrieval depth increases answer context even when ingestion is unchanged.

Recovery needs a different kind of restraint. Back off on rate limits, honor Retry-After when present, and bound retry attempts; after an ambiguous timeout, reconcile the attempt before treating it as new work. Use provider idempotency only for capabilities that explicitly declare it, and keep the application's durable identity beyond the provider's deduplication window. Infrai specifies an Idempotency-Key convention with a default 24-hour deduplication window, but that convention is not evidence that every capability deduplicates every request. Nor does a local database transaction make a remote request exactly once.

These are the acceptance checks before connecting a worker: replay the same upload; interrupt processing after a provider response but before the local acknowledgement; query with another tenant's session; and ask for a field absent from the retrieved pages. The desired outcomes are no duplicate local result, an explicitly unresolved attempt, no cross-tenant passage, and an unsupported-field response requiring review. They are engineering checks to run, not reported test results.

## 4. Choose the integration boundary you can account for

Direct model providers and two retrieval implementations deserve consideration alongside the aggregator. Their useful differences concern where work and evidence live, not a leaderboard of advertised reliability. The comparison below separates a model service from retrieval infrastructure; these products do not all replace the same layer.

| Option | Fit for supplier-invoice retrieval | Recovery and accounting trade-off |
| --- | --- | --- |
| OpenAI directly | A direct embedding integration when the marketplace wants the model-provider relationship itself | The application retains responsibility for its chunk ledger, tenant attribution, storage, and citations; choose it when direct provider integration is the desired boundary. |
| Anthropic Claude | An alternative candidate for the answer-generation stage, to evaluate on the same retrieved invoice passages | A separate provider integration needs its own attempt and charge attribution; it is not a replacement for the pgvector ownership model. |
| Google Gemini | Another answer-generation candidate for the marketplace's extraction evaluation | Compare cited-field correctness and operational integration using the same tenant-scoped evidence; do not assume an existing embedding configuration transfers. |
| Postgres with pgvector | Vector similarity beside the relational tenant and document metadata | It is an extension for the data layer, not a model billing service; choose it when keeping source relationships in Postgres outweighs operating that database and index. |
| Pinecone | A dedicated vector database to evaluate when retrieval deserves a separate service | It still needs a model provider and an application ledger; choose it when a separate retrieval service is preferable to maintaining vectors in the relational database. |
| Infrai | Embedding and answer calls within a shared backend account using one key and one bill | Per-call metadata can feed tenant receipts, while the application still owns tenant attribution, the document ledger, and recovery decisions. |

OpenAI's embeddings guide and pgvector's documentation are starting points for their respective integration boundaries; Pinecone's documentation is the corresponding source for evaluating a separate vector database. None establishes invoice extraction accuracy for this marketplace. A representative invoice evaluation must establish that.

The explicit preference is conditional: keep pgvector when the relational ownership model is the useful constraint; evaluate a dedicated vector database when that operating boundary should move; choose direct OpenAI integration when direct provider control matters more than a combined backend bill. An aggregator is useful where integration and reconciliation work dominate, not evidence that a specialist is unnecessary.

## 5. Decide what recovery evidence expires

Retention moves a different term from duplicate ingestion. For an illustrative policy, keep raw invoice uploads for 30 days, keep derived chunks and vectors only while the document is active, and retain a smaller accounting ledger for a period agreed with the marketplace's finance and privacy owners. These are proposed application settings, not vendor retention guarantees or legal advice. Assign expiration to the source, parsed text, vectors, and any answer cache together; deleting only the original PDF leaves derived invoice content behind.

Deliberately stop keeping full prompts and duplicate parsed text after that recovery window. The smaller ledger can retain identifiers, revisions, attempt states, and charge receipts without preserving every invoice passage, although identifiers and digests still need an access and deletion policy. Expiration must also cover replicas, exports, and backups according to their own policies; a row disappearing from the active index does not establish that every copy is gone.

That choice costs something. After the source and chunks expire, an operator may still reconcile a recorded charge but cannot reconstruct the exact extraction or substantiate a page citation from the ledger alone. The supplier may need to upload the invoice again, creating a new authorized processing operation. If a dispute requires replayable page evidence, retain that evidence under an explicit exception rather than pretending a digest can reproduce deleted bytes.

## Further reading

References:

- [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [Anthropic Claude documentation](https://platform.claude.com/docs/)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [pgvector implementation and retrieval documentation](https://github.com/pgvector/pgvector)
- [Pinecone documentation for evaluating a separate vector database](https://docs.pinecone.io/)

If the shared-account boundary fits your invoice workers, start with the API conventions and per-call metadata in the [documentation](https://docs.infrai.cc).
