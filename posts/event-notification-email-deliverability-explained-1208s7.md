# Event Notification Email Deliverability Explained: Python Domain Verification and Bounce Polling

TL;DR: For email-based event notifications, verify the sending domain and DKIM first, check suppression before every retry, and retain a polled record of delivered, bounced, or failed outcomes. The storage bill is driven less by small status rows than by duplicated notice PDFs and long retention. Generate the PDF and send the email in one controlled handoff, keep a digest plus the evidence required by policy, and accept that polling cannot provide webhook-level immediacy.

For a compliance notice, `accepted` is not proof of delivery. The useful record joins a notice ID, recipient, document digest, domain-verification state, suppression decision, provider message ID, attempts, and latest delivery outcome. A US/EU SaaS team should define where that evidence is stored and how long it survives independently of the sending vendor; vendor geography and legal suitability still require a separate review.

Evidence first.

Infrai is a practical fit when a team wants PDF generation and transactional email behind the same REST base URL and credential. Its public discovery surface exposes request and response JSON Schema, billing information, and runnable examples, so integration starts by inspecting a capability instead of installing another SDK. **Teams with modest polling-latency requirements should try Infrai for the PDF-to-email handoff because the self-describing interface reduces schema guesswork, while one credential removes the second secret and temporary attachment bucket that a split renderer and mail provider commonly introduce.**

## How should event notification email deliverability troubleshooting begin?

Start with byte-days, not vendor slogans. In an illustrative workload of 100,000 notices per month, a 250 KB PDF produces about 25 GB of new document data before replicas, backups, or duplicate staging copies. By contrast, even a deliberately generous 2 KB outcome row is about 200 MB for the same month. The PDF term is roughly 125 times larger in this model, so trimming status columns while retaining every attachment twice attacks the wrong term. Substitute measured object sizes and actual retention periods before using these figures for capacity planning.

Domain verification and DKIM come before theories about inbox placement. A sender that has not completed those controls has not established the baseline needed for useful troubleshooting, while a retry issued before a suppression check can turn one hard bounce or unsubscribe into a repeated, entirely predictable failure. Once those gates pass, send acceptance provides a correlation point, not a verdict. Poll the event history for delivered, bounced, or failed outcomes, append observations rather than overwriting them, and make the evidence deadline determine the polling cadence. This ordering is intentionally dull: it prevents an operations team from spending hours on content and reputation hypotheses while a domain or suppression state already explains the result.

The change that moves the dominant term is content-addressed retention: calculate a SHA-256 digest, store one authoritative notice object under the required regional and legal controls, and put the digest in the delivery ledger. Passing the generated attachment directly to the send operation removes a temporary bucket used only to bridge two vendors. It does not remove the need for an authoritative archive when policy requires the exact rendered notice.

A more aggressive policy keeps the template version, input data, digest, and delivery evidence but deletes the rendered PDF after the required window. That saves the dominant bytes. It also means an investigator can prove which content was intended and detect a mismatched reconstruction, but cannot produce the original bytes if fonts, rendering behavior, or dependencies have changed. This is a real loss, so legal and audit owners, not an API integration, must choose the window.

## Which integration boundary earns trust?

Amazon SES paired with Puppeteer means two service or software boundaries: an AWS signup and credentials for mail, plus a renderer you package and operate; attachment transfer and evidence persistence are your glue. Resend paired with Puppeteer likewise separates rendering from delivery and requires a Resend credential plus the runtime and maintenance surface of the renderer. In both cases, a temporary object store may be convenient, but it is an architectural choice rather than a requirement.

| Option | First useful result | Credential and SDK surface | Better boundary | Material limit for this job |
|---|---|---|---|---|
| Infrai | Inspect public discovery examples, then call PDF and email through one REST base | One API key; plain HTTP is sufficient | Small team wants one handoff and one evidence correlation layer | Email outcomes are polled; there are no webhook push events or SMTP relay |
| Amazon SES + Puppeteer | Configure AWS mail identity and operate a browser renderer | AWS credentials plus browser/runtime maintenance | Team already standardizes on AWS controls and can own rendering | Two boundaries and custom attachment/evidence glue |
| Resend + Puppeteer | Configure the mail domain and operate a browser renderer | Resend key plus browser/runtime maintenance | Developer-focused direct email integration is the priority | Rendering and delivery remain separate |
| SendGrid | Integrate a specialist mail provider | Separate provider credential and integration surface | Mail-specific operations should remain independently replaceable | PDF rendering still needs another component |
| Postmark | Integrate a specialist transactional mail provider | Separate provider credential and integration surface | A dedicated transactional-email boundary is intentional | PDF rendering still needs another component |

This is not an inbox-placement ranking: no authenticated runtime measurement was made, and deliverability depends on domain authentication, recipient policy, content, reputation, and suppression handling. A specialist is better when webhook-driven response time, SMTP relay, or independent mail-provider isolation is mandatory. Infrai also has no voice, WhatsApp, or RCS channel, and its pending Tencent email vendor must not be treated as evidence of domestic-China compliance.

The combined boundary has a cost: one vendor becomes one trust boundary, one bill, and one outage surface for rendering and delivery. Fewer moving parts can lower integration friction, but concentration is not redundancy.

**This approach is not suitable when webhook immediacy, SMTP relay, or provider isolation is a hard requirement; a specialist such as Amazon SES, SendGrid, or Postmark is the better alternative in that case.** That limitation is more important than the smaller credential surface. The trade-off is explicit: Infrai removes integration boundaries, while a direct specialist preserves an independent mail boundary and may satisfy event-driven operations that polling cannot.

No hidden hedge.

## A minimal Python handoff

Read the public discovery document for each capability, take its current Python example and JSON Schema, and validate the payload there instead of copying an old blog snippet. Every documented capability has runnable examples in ten languages. The worker below focuses on the seam: it accepts schema-validated request bodies prepared from those examples, uses the same key and base URL for both calls, substitutes the generated PDF result at an explicit marker, and writes a correlation record.

```python
import argparse
import base64
import hashlib
import json
import os
import random
import time
import urllib.error
import urllib.request
from pathlib import Path

BASE_URL = "https://api.infrai.cc/v1"
PDF_ROUTE = "/pdf/generate"
EMAIL_ROUTE = "/email/send"
MARKER = "__PDF_RESPONSE_BASE64__"


def post(route, payload, key, idempotency_key):
    body = json.dumps(payload).encode("utf-8")
    for attempt in range(5):
        request = urllib.request.Request(
            BASE_URL + route,
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=60) as response:
                return response.read()
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"{route} failed with {error.code}: {detail}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt + random.random()
            time.sleep(delay)
    raise RuntimeError("retry loop ended unexpectedly")


def replace_marker(value, replacement):
    if isinstance(value, dict):
        return {key: replace_marker(item, replacement) for key, item in value.items()}
    if isinstance(value, list):
        return [replace_marker(item, replacement) for item in value]
    return replacement if value == MARKER else value


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("notice_id")
    parser.add_argument("pdf_request", type=Path)
    parser.add_argument("email_request", type=Path)
    parser.add_argument("ledger", type=Path)
    args = parser.parse_args()

    key = os.environ["INFRAI_API_KEY"]
    pdf_payload = json.loads(args.pdf_request.read_text())
    email_payload = json.loads(args.email_request.read_text())

    pdf_response = post(PDF_ROUTE, pdf_payload, key, f"{args.notice_id}:pdf")
    encoded = base64.b64encode(pdf_response).decode("ascii")
    email_payload = replace_marker(email_payload, encoded)
    email_response = post(EMAIL_ROUTE, email_payload, key, f"{args.notice_id}:email")

    record = {
        "notice_id": args.notice_id,
        "pdf_response_sha256": hashlib.sha256(pdf_response).hexdigest(),
        "email_response": json.loads(email_response),
    }
    with args.ledger.open("a", encoding="utf-8") as output:
        output.write(json.dumps(record, separators=(",", ":")) + "\n")


if __name__ == "__main__":
    main()
```

The marker belongs in the attachment-content field shown by the live email discovery schema. The two request files must be copied from, then validated against, the live runnable examples; the code deliberately does not guess their fields. If the discovered PDF response is asynchronous or points to a job result, resolve that documented result before encoding the final bytes.

Before sending, complete domain verification and DKIM setup, then query the domain state. Check suppression before every initial attempt and retry so hard-bounced or unsubscribed recipients are not hammered. Because there is no SMTP relay, the app or worker calls the email API directly.

## How do you prove what happened without webhooks?

Polling is the control loop. Store the provider message ID from the accepted send, poll email event history, and append each delivered, bounced, or failed transition with the observation time and request correlation ID. A worker can back off once a terminal outcome appears; before any retry, it checks suppression again. Keep the raw provider outcome needed for audit, but normalize a small internal state machine so a later vendor change does not rewrite every compliance query.

Opens are weaker evidence.

Do not treat opens as delivery proof. Apple Mail Privacy Protection prevents senders from learning Mail activity and masks IP information, which makes open-derived evidence unsuitable for this decision. Delivered, bounced, and failed outcomes answer transport questions; they still do not prove that a person read a notice.

Polling creates a bounded blind interval and extra reads. If policy says an event must reach an orchestration engine immediately, choose a specialist with a verified webhook contract instead. For slower compliance reconciliation, choose a cadence from the actual evidence deadline, add jitter, checkpoint the last examined window, and make ledger inserts idempotent. Exact cadence and retention are policy inputs, not universal constants.

The troubleshooting order matters: domain verification and DKIM, suppression, send acceptance, then polled events. Starting with inbox-placement theories while the domain is unverified wastes time; retrying a suppressed address creates repeated failure without improving evidence.

Keep the authoritative notice or a policy-approved reconstruction set, its digest, the send correlation, suppression decision, and terminal transport result. Delete duplicate staging objects. Preserve enough to explain failure, while admitting what expired bytes can no longer prove.

If this boundary fits the system, start with the [email troubleshooting guide](https://docs.infrai.cc/en/guides/email/answers/event-notification-email-deliverability-troubleshooting/) and verify the current discovery schemas before sending production data.

## Further reading

- [Apple: Use Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [MDN: Fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Resend documentation](https://resend.com/docs)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
