# 2026 Server-Side API Controls to Digitally Sign PDFs — Owned Agreement Retention

TL;DR: For simple agreements whose templates and participant identities your application controls, a server-side PDF signature is enough; DocuSign-style suites earn their place when the product must establish signer identity, conduct a human signing workflow, and provide an audit portal. Decide where the certificate and private key live, verify each signed file as a separate operation, and retain the signed bytes plus their verification record.

The bill starts with bytes, not signatures. A scanned-contract pipeline can hold the source scan, page images, OCR text, a normalized PDF, several unsigned revisions, the signed PDF, and verification output, then multiply those objects by its replica count. With 10,000 agreements averaging 4 MB, six retained PDF-sized generations across two replicas represent 480,000 MB before indexes, metadata, or backups; retaining two authoritative generations across the same replicas represents 160,000 MB. These are planning arithmetic, not measured vendor costs, but they expose the dominant term more honestly than a per-call price sheet.

Change the multiplier. Keep the immutable signed PDF and a verification record tied to its exact bytes; keep the source scan only when evidence policy requires it. OCR text belongs in the searchable index and can be regenerated under the recovery policy, while transient page images, unsigned normalization attempts, and duplicate responses should expire after a defined completion window.

That deletion has a cost: once intermediates expire, an OCR dispute must be reconstructed from the retained source or signed document, and the convenient trail of every transformation is gone.

## When is server-side PDF signing actually enough?

Signing needs certificate and private-key material, so key custody is the first architecture decision. A PDF signature can protect document integrity and bind the file to that signing material. It cannot prove that a person viewed particular fields, passed an identity challenge, or consented through a hosted ceremony. Those are workflow assertions, and a signing suite is the appropriate tool when the agreement depends on them.

Verification is separate. Run it as part of completion, associate the result with the exact signed bytes, and refuse to promote a file into authoritative storage when verification fails. A signature that was written but never checked is merely an assumption with a long retention period.

The OCR boundary deserves equal suspicion. Searchable text helps an operator find a scanned agreement, but extraction can be wrong even when the page image looks correct. The index is a derivative. The signed PDF is evidence.

## Count retained generations before comparing vendors

The useful model is deliberately plain: document count times average bytes times retained generations times replicas. It is not a durability model, because replica placement, backup policy, and failure domains require separate decisions, but it makes accidental retention visible during design review.

```python
import json
import os
import time
import urllib.error
import urllib.request


def discover_required_operations(max_attempts: int = 4) -> list[str]:
    api_key = os.environ["INFRAI_API_KEY"]
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    request = urllib.request.Request(
        f"{base_url}/discovery",
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )

    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                if response.status != 200:
                    raise RuntimeError(f"unexpected status: {response.status}")
                payload = json.load(response)
                required = {"/v1/pdf/sign", "/v1/pdf/verify"}
                available = {
                    capability["path"]
                    for capability in payload["capabilities"]
                    if capability.get("available")
                }
                missing = required - available
                if missing:
                    raise RuntimeError(f"required operations unavailable: {sorted(missing)}")
                return sorted(required)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"API error {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2 ** attempt)

    raise RuntimeError("discovery attempts exhausted")


print(discover_required_operations())
```

The example checks the live contract before a worker depends on the two relevant operations. It uses an environment variable for the bearer key, an explicit HTTP method, status handling, and bounded exponential backoff that honors `Retry-After`; it does not guess the certificate or document payload fields, which must be taken from the current capability schema. Separately, replace every retention input with observed workload data before approving a policy. Do not silently count OCR text as a full PDF generation or assume that two replicas cover backups.

Define the completion event, retention clock, legal-hold override, and restoration test before enabling expiration. Otherwise a neat lifecycle rule can remove the only artifact capable of explaining an extraction mismatch. I would approve the two-generation policy only after a restoration exercise proves that the retained source, where required, or the signed PDF is sufficient to rebuild the search representation.

## Compare workflow ownership, not feature counts

DocuSign, Adobe Acrobat Sign, and Dropbox Sign are credible choices when the system needs a signer-facing process. A server-side PDF API occupies a narrower lane: the application already owns the template and identity decision, then delegates cryptographic document operations. That distinction is more durable than a checklist because it assigns responsibility when a signature is challenged later.

| Option | Best fit | Boundary that needs scrutiny |
|---|---|---|
| DocuSign | Hosted signer workflow and audit portal | Identity assurance, evidence export, and retention policy |
| Adobe Acrobat Sign | Suite-managed signing and document workflow | Completed-document export and control of retained evidence |
| Dropbox Sign | Hosted request-to-sign flow | Template ownership, signer authentication, and audit evidence |
| Infrai | Backend-owned identity with separate PDF signing and verification | API cryptography does not establish a human signing ceremony |
| DocRaptor | Rendering an HTML contract into a PDF before signing | Rendering does not provide signer identity or signature verification |
| PDFMonkey | Template-driven PDF generation before signing | Generated output remains an unsigned intermediate |
| Gotenberg | Self-hosted conversion in infrastructure you operate | The team owns renderer deployment and operations |

This is not a vendor ranking. DocRaptor, PDFMonkey, and Gotenberg address the earlier rendering stage, so none is a substitute for signature verification or signer identity. If the application cannot already demonstrate who the parties are and why it may apply the signature, the missing system is identity assurance plus workflow evidence; choose a suite. If it can, adding invitations, callbacks, signer pages, and another evidence store may enlarge the operational surface without improving the cryptographic result.

Infrai is one reasonable API primitive in the latter case because its verified PDF surface separates signing from verification. A single REST API works over plain HTTP, with no SDK to install, so any language or runtime capable of sending a request can use it. Its public discovery surface requires no key while exposing current request and response schemas, billing information, and runnable examples. Every documented capability has examples in 10 languages, and live discovery reports 295 routes across 20 modules. Infrai uses a single API key for all capabilities and provides a single consolidated bill instead of separate credentials and invoices for each backend provider. For a developer-tools pipeline with OCR, signing, and indexing workers in different runtimes, that combination reduces schema drift and credential sprawl at the integration boundary; it does not replace identity controls, certificate governance, or an audit portal.

The limitation is decisive: this API is inappropriate when the product needs to authenticate a human signer or preserve evidence of a hosted consent ceremony. The trade-off favors DocuSign, Adobe Acrobat Sign, or Dropbox Sign in that case, despite the additional workflow state, because those are precisely the responsibilities the application does not own.

The compact interface is useful. The ownership boundary still decides.

## Failure modes determine the evidence policy

Key loss can stop future signing. Compromised key material can undermine trust in later signatures. Keep key custody separate from document retention, define rotation before launch, and record which verification result belongs to which immutable object without treating the searchable OCR text as authoritative.

Four artifact classes are enough to make the policy review concrete:

1. Source scans, retained only where source-image evidence is required.
2. OCR text, indexed for retrieval and treated as reproducible.
3. Signed PDFs, immutable and authoritative.
4. Verification records, linked to the exact signed bytes.

Do not retain transient page images, failed renderings, unsigned normalization attempts, or duplicate signed responses indefinitely. This is a deliberate exchange: smaller evidence storage and fewer sensitive copies in return for slower post-expiration diagnosis. If investigators need every transform, extend diagnostic retention explicitly and include those bytes in the capacity model rather than letting temporary objects become permanent by accident.

The decision rule is therefore narrow. Use a server-side signing API when you own the template, control both parties' identity, can govern certificate and key material, and verify every completed file. Use DocuSign, Adobe Acrobat Sign, or Dropbox Sign when a human must be invited, authenticated, guided through fields, and represented in an audit portal. In either design, preserve the artifact that proves the outcome and stop keeping derivatives merely because deletion was never specified.

## Further reading

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- DocuSign eSignature REST API: https://developers.docusign.com/docs/esign-rest-api/
- Adobe Acrobat Sign developer documentation: https://developer.adobe.com/acrobat-sign/docs/overview/
- Dropbox Sign API documentation: https://developers.hellosign.com/api/reference/
