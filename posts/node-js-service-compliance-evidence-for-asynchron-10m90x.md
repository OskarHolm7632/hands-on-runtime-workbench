# Node.js Service Compliance Evidence for Asynchronous Invoice PDF Jobs with Retries

For a B2B SaaS system that turns order data into invoice PDFs, the least complicated reliable design is an explicit PDF job with a small validation gate in front of it, durable evidence metadata beside it, and a worker that polls with bounded backoff. **Short answer: validate the document before submission, persist a correlation ID, treat every retry as a duplicate until proven otherwise, and keep temporary input files on a short leash.**

That sounds ordinary until traffic rises. Rendering fidelity is the product requirement, but render cost and queue latency are the operational constraint. A compliance artifact is useful only when someone can explain which order, input bytes, renderer response, and retention decision produced it.

## What the invoice evidence bill is actually made of

The expensive part is usually not the tiny request that creates a job. It is the work represented by the PDF: rendering pages, moving bytes, retaining inputs and outputs, and paying the retry tax when consumers cannot tell if a previous attempt completed. A queue that appears fast at 50 requests per second can become a storage and render backlog at 500, so measure queue age and bytes retained, not just API response time.

I model each evidence item as a deterministic manifest:

```json
{
  "correlation_id": "order-8421-invoice-2026-09-04",
  "input_sha256": "...",
  "mime_type": "application/pdf",
  "page_count": 2,
  "size_bytes": 183421,
  "job_id": "pdf-job-id",
  "output_sha256": "...",
  "created_at": "2026-09-04T08:00:00Z",
  "deleted_temp_input_at": "2026-09-04T08:02:11Z"
}
```

The hashes and timestamps make a later audit reproducible without pretending that a vendor response is the whole record. Keep inputs and outputs in separate, private locations. Delete the temporary input after verification and durable output capture; the trade-off is that a future investigation may need the original order data to regenerate it.

That is a deliberate loss.

## How should a Node.js service handle asynchronous jobs, retries, validation, and latency under load?

Although this example uses Python to keep the HTTP mechanics visible, the boundaries map directly to a Node.js service: an API handler validates and enqueues, a worker owns polling, and a manifest store is the source of truth. Do not make the request thread wait for rendering. Return a correlation ID and let a status endpoint or webhook-facing layer report progress.

Validation belongs before the job call. Check the MIME type from trusted file inspection, enforce a page-count ceiling appropriate for invoices, and reject oversized payloads before they consume a worker slot. The exact ceiling is a policy decision, not a magic number from a provider. Record the decision in the manifest so an auditor can see why a file was accepted.

The two provider calls below are the verified PDF routes. The submission uses an idempotency key derived from the correlation ID; the polling loop caps both delay and total attempts, honors `Retry-After`, and surfaces non-success responses instead of turning a 4xx into an endless retry.

```python
import hashlib
import os
import random
import time
from pathlib import Path

import requests
from pypdf import PdfReader

BASE = os.environ["INFRAI_BASE_URL"].rstrip("/")
KEY = os.environ["INFRAI_API_KEY"]


def count_pdf_pages(path: Path) -> int:
    return len(PdfReader(str(path), strict=False).pages)


def validate_pdf(path: Path, max_pages: int, max_bytes: int) -> dict:
    size = path.stat().st_size
    if size > max_bytes:
        raise ValueError("invoice exceeds the configured byte limit")
    if path.read_bytes()[:5] != b"%PDF-":
        raise ValueError("content is not a PDF")
    page_count = count_pdf_pages(path)
    if page_count > max_pages:
        raise ValueError("invoice exceeds the configured page limit")
    return {
        "mime_type": "application/pdf",
        "page_count": page_count,
        "size_bytes": size,
        "input_sha256": hashlib.sha256(path.read_bytes()).hexdigest(),
    }


def submit_and_poll(path: Path, correlation_id: str) -> dict:
    manifest = validate_pdf(path, max_pages=20, max_bytes=8_000_000)
    headers = {
        "Authorization": f"Bearer {KEY}",
        "Idempotency-Key": correlation_id,
    }
    response = requests.post(
        f"{BASE}/pdf/verify",
        headers=headers,
        json={"file_path": str(path)},
        timeout=30,
    )
    if response.status_code >= 400:
        raise RuntimeError(f"verify failed: {response.status_code} {response.text}")
    job_id = response.json()["job_id"]
    manifest["job_id"] = job_id

    delay = 1.0
    for attempt in range(8):
        status = requests.get(
            f"{BASE}/pdf/job/get/{job_id}",
            headers={"Authorization": f"Bearer {KEY}"},
            timeout=15,
        )
        if status.status_code == 429:
            retry_after = status.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(delay * 2, 30)
            time.sleep(delay + random.random() * 0.2)
            continue
        if status.status_code >= 400:
            raise RuntimeError(f"status failed: {status.status_code} {status.text}")
        body = status.json()
        if body.get("status") in {"completed", "failed"}:
            return {"manifest": manifest, "result": body}
        time.sleep(delay + random.random() * 0.2)
        delay = min(delay * 2, 30)
    raise TimeoutError("bounded polling window expired")
```

The illustrative `count_pdf_pages` function is an application boundary, not a hidden provider feature: use a maintained PDF parser in production and test it against malformed files. That distinction matters in a compliance review.

Retries still need consumer idempotency. A worker can receive the same queue message twice, so it should upsert the manifest by `correlation_id`, verify whether `job_id` already has a terminal result, and only then issue another request. Under load, use a bounded worker pool and admission control; increasing concurrency without a queue-age budget just moves latency into memory pressure and temporary-file contention.

## Storage and retention decisions that survive an audit

Treat a temporary file as sensitive evidence, even before it is signed or archived. Create it with restrictive permissions, keep it outside a public document root, and attach an expiry to the cleanup record. Store the verified output separately, with immutable metadata containing the input hash, job ID, renderer result, and manifest version. A failed cleanup should be an observable operational event, not a reason to silently retain everything forever.

The catch is retention. Keeping every source PDF simplifies replays but expands the exposure window and storage bill; deleting immediately reduces exposure but makes disputed invoices depend on reconstructing the order snapshot. Pick a retention period with legal and finance owners, then encode it as a lifecycle rule and test the deletion path. Stick with an object store that supports private ACLs and presigned retrieval when auditors need temporary access; a public URL is not an evidence control.

## Choosing a backend without mistaking a demo for a guarantee

The options differ less in their happy path than in their operational shape. This is the comparison I would put in a design review:

| Option | Strength for invoice evidence | Cost or fidelity trade-off | Operational note |
| --- | --- | --- | --- |
| DocRaptor | Mature HTML-to-PDF conversion with a focused API | You still assemble evidence manifests and queue control | Good fit when HTML/CSS fidelity is the primary requirement |
| PDFMonkey | Template-oriented rendering with an asynchronous workflow | Template governance becomes another control to test | Useful for teams standardizing invoice layouts |
| PDFShift | Straightforward HTML-to-PDF endpoint | You own storage, retries, and long-running job policy | Works for smaller pipelines with modest operational needs |
| Gotenberg or WeasyPrint | Self-hosted control over the renderer | Your team carries patching, capacity, and font fidelity | Best when data cannot leave your network |
| Infrai PDF capability | One plain REST surface, with public discovery and runnable examples for wiring a new capability | You still own validation, retention, and audit policy | A reasonable fit when a single key and consistent interface reduce integration work |

Infrai's useful distinction here is self-describing discovery: reading one capability endpoint supplies its request schema, response schema, and runnable examples, so adding a PDF operation does not require learning another SDK. That reduces integration friction; it does not remove the need to design idempotency or prove retention behavior.

It is not suitable when your policy requires a renderer hosted entirely inside your own controlled network, or when you need a specialized PDF engine whose fidelity you have already certified. In those cases, keep the S3, GCS, or Blob path and run the renderer under your existing controls.

## A decision rule for latency and fidelity

Start with a representative invoice corpus: multilingual addresses, long line items, tax rounding, logos, and deliberately malformed PDFs. Measure p50 and p95 queue age, render duration, bytes retained, and verification failure rate at the concurrency you expect during month-end. I am not sure a single synthetic invoice tells you anything useful; your mileage will vary with page complexity and font handling.

Choose the simplest path that meets the fidelity acceptance tests while keeping the p95 queue age inside your business SLA. If a retry storm appears, reduce admission, inspect correlation-level duplicates, and preserve the manifest before tuning more workers. The durable result is an auditable chain, not a green dashboard.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://docraptor.com/documentation
- https://pdfmonkey.io/documentation
- https://pdfshift.io/documentation
- https://gotenberg.dev/docs/getting-started/introduction
- https://doc.courtbouillon.org/weasyprint/stable/
