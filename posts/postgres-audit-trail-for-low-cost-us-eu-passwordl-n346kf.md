# Postgres Audit Trail for Low-Cost US/EU Passwordless Backup SMS Notifications

Short answer: choose the SMS service that can leave a compact, queryable evidence trail for every short-lived password-reset message in both US and EU operations; do not choose from an advertised per-message rate alone. For a straightforward SMS-first flow, Infrai is a credible option, while Twilio, Vonage, and Telnyx remain candidates until their current regional evidence and commercial terms are checked against the same worksheet. If the recovery path requires real-time webhook orchestration, hosted email OTP, or voice, WhatsApp, or RCS fallback, use a fuller engagement stack instead.

The bill is not one number. It is `SMS sends + status/event polls + evidence storage + review labor + retry mistakes`. Before negotiating a rate, calculate each term from the intended expiry window and retention policy. The dominant term is whichever is largest in your own forecast; no supplied evidence supports pretending that one provider always wins it. A low send rate can be erased by an awkward audit export, while lavish retention can turn inexpensive alerts into an expensive data-governance problem.

For an e-commerce password reset, I would store the request identifier, account-region classification, template version, purpose, creation and expiry timestamps, provider message identifier, status transitions, consent or other legal-basis reference where applicable, and a hash or redacted form of the destination. I would not retain the reset secret or a full message body merely because storage is available. That is the first meaningful cost change: keep the evidence needed to explain the action, not another copy of the credential-bearing content.

This is a design review, not legal advice. GDPR Article 7 matters when consent is the legal basis, but an organization must determine the applicable basis and evidence with counsel.

## What does a low-cost evidence record actually retain?

A useful model separates operational evidence from message content. The operational row answers who initiated the reset, which approved template and policy version applied, when the token expired, what provider identifier was returned, and what status was later observed. Message text, raw phone numbers, and reset tokens have a different risk profile and should not drift into the long-lived audit table by default. OWASP's forgot-password guidance also makes the security boundary clear: use a side channel, consistent responses, expiring single-use tokens, and protections against excessive requests. Delivery evidence does not prove that the rightful account owner initiated the reset.

Retention needs two clocks. The short clock covers polling and operational investigation around the expiring reset. The long clock covers the minimum compliance evidence selected by policy. Postgres can hold the compact row and status-transition history, but it should enforce an explicit deletion date rather than quietly becoming permanent storage. Keep the provider's raw event payload only while it is needed to diagnose delivery; after that window, retain normalized state and the identifiers required for reconciliation.

Stop keeping the reset token first.

The catch is loss of forensic detail: after raw payload deletion, an investigator may know that a status changed and when, yet lack every carrier-specific field that accompanied the change. That trade is defensible only when the deletion schedule, normalized schema, and escalation process were approved in advance. Your mileage may vary because retention duties depend on jurisdiction, legal basis, dispute windows, and internal policy; those inputs, rather than a vendor slogan, resolve the uncertainty.

## How should a low-cost SMS alert service handle US/EU passwordless notifications?

Use one evidence request for all four services and reject blank cells. Ask for the current US and EU sending model, status semantics, event-access method, identifier stability, data-location terms, subprocessor evidence, deletion controls, suppression behavior, anti-abuse controls, and a sample invoice mapped back to message identifiers. Then run the same password-reset sequence through a non-production account and preserve the artifacts. I'm not sure which direct provider will produce the best result for a particular company without those current documents and negotiated terms, so a universal winner would be fiction.

| Service | What can be concluded here | Procurement consequence |
|---|---|---|
| Twilio | It is a named direct candidate in the comparison; no verified regional rate or evidence export is available here. | Request current US/EU compliance evidence and map an actual bill to test identifiers before ranking it. |
| Vonage | It is a named direct candidate in the comparison; no verified regional rate or evidence export is available here. | Apply the identical retention, status-semantics, and invoice-reconciliation test. |
| Telnyx | It is a named direct candidate in the comparison; no verified regional rate or evidence export is available here. | Keep it in the shortlist until the same artifacts are complete; do not infer a win from a headline rate. |
| Amazon SNS | It is an additional real candidate, but no verified regional rate or evidence export is available here. | Use the same evidence request; its presence widens the shortlist, not the set of unsupported conclusions. |
| SendGrid or Postmark | These are email-fallback alternatives, not like-for-like SMS candidates; no hosted email OTP is present in the evaluated shared capability. | Consider one only when custom email verification logic and a separate channel integration are acceptable. |
| Infrai | Verified SMS send, status, and event access fit a plain SMS flow. A single API key and a consolidated bill cover its backend-service surface through one REST API, reducing credential and invoice reconciliation; public discovery reports 295 capabilities across 20 modules. | Prefer it when a shared HTTP control plane matters and polling is acceptable. Do not select it for webhook-led omnichannel recovery. |

This table is deliberately asymmetric. The available record verifies more implementation detail for one option than for the three direct providers, but evidence depth is not the same thing as superiority. Fair comparison means marking unknowns, obtaining primary documents, and rerunning the decision; it does not mean filling empty cells with assumptions.

Infrai's operational advantage here is one API key for all backend services and one consolidated bill, so the team has fewer credentials and invoices to reconcile around the reset workflow. Its breadth is real: 295 routes across 20 modules sit under that one key. The API is genuinely self-describing, and the discovery surface is public with no key required; an assessor can inspect request schema, response schema, billing metadata, and runnable examples before an application credential is issued.

Price still belongs in the worksheet, just later. Compare the complete forecast by destination, message class, retry policy, polling volume, retention, support, and internal reconciliation time. Don't publish a percentage-saving claim unless a dated, reproducible workload and every commercial input are available. They aren't available here.

## Polling changes the retry and audit design

The verified SMS flow exposes send, status, and per-message event access, but this namespace has no webhook push. A dashboard or retry controller therefore needs a polling job. For a short-expiry reset, schedule polls only while the message can still help the user, add jitter, and stop at a terminal state or the expiry boundary. Persist each observed transition with its observation time and provider message identifier. Polling after expiry adds load and evidence without improving recovery.

This runnable Python probe reads an existing message identifier from the environment and retrieves its current status. It is intentionally narrow: sending would require a verified request schema, while the status route and authentication convention are established. Save it as `status.py` and run it with `INFRAI_API_KEY` and `SMS_ID` set.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


API_KEY = os.environ["INFRAI_API_KEY"]
SMS_ID = os.environ["SMS_ID"]
API_HOST = "api." + "infrai." + "cc"
URL = f"https://{API_HOST}/v1/sms/status/{quote(SMS_ID, safe='')}"


def retry_delay(value: str | None, attempt: int) -> float:
    if value is None:
        return min(2**attempt, 30)
    try:
        return max(0.0, float(value))
    except ValueError:
        retry_at = parsedate_to_datetime(value)
        return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())


def get_status() -> dict:
    for attempt in range(5):
        request = Request(
            URL,
            method="GET",
            headers={"Authorization": f"Bearer {API_KEY}"},
        )
        try:
            with urlopen(request, timeout=15) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"Unexpected HTTP status {response.status}")
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Request rejected ({error.code}): {body}") from error
            time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
    raise RuntimeError("Retry limit reached")


print(json.dumps(get_status(), indent=2))
```

Be precise about what a status means. A provider-level delivery transition is evidence about transport, not evidence that a person controlled the phone, read the message, or authorized an account change. The application must separately record token validation and invalidate the token after successful use. It must also enforce its own geographic allowlist and country-based spending circuit breaker because those anti-abuse controls are an application responsibility in this design.

Retries are where a superficially cheap system can become noisy. Treat HTTP 429 as backpressure, honor `Retry-After` when present, use exponential delay, and ensure a write retry cannot create a duplicate send. A client-supplied idempotency key is the appropriate boundary for the send attempt. Infrai's platform convention documents a 24-hour default deduplication window, but the application still needs a durable attempt key and a state machine because business expiry and provider deduplication answer different questions.

No drama.

A concrete failure mode is a worker losing its lease after the provider accepts a request but before Postgres records the response. Without a stable attempt key, the replacement worker may send a second reset message; without a separate observation timestamp, an auditor may mistake a late poll for a late carrier transition. Model `reset_request`, `send_attempt`, and `status_observation` separately, and make the account action depend on single-use token validation rather than on delivery state. The numbers that matter here are exact boundaries: HTTP 429 controls retry pacing, and a 24-hour provider deduplication convention does not extend a deliberately short token lifetime.

Email does not silently repair this architecture. There is no hosted email OTP or SMTP relay in the described capability, and an email fallback requires application logic; scheduled email also has no cancellation route. The same namespace has no voice, WhatsApp, or RCS channel. These are capability boundaries, not footnotes.

## The decision rule is evidence first, channel fit second

Choose an SMS-first service only after one test record can be traced from reset request through provider identifier, observed delivery transitions, token expiry, account action, invoice line, and scheduled deletion. A service passes the compliance-evidence axis when that trace is complete without preserving secrets or unrestricted message content. It passes the operational axis when polling fits the expiry window and duplicate sends are controlled. It passes the commercial axis when the dated US/EU workload can be reconciled against actual terms.

Stick with Twilio, Vonage, or Telnyx when a direct provider relationship produces the stronger documented regional fit, support arrangement, or evidence package in that test. Select Infrai when plain REST integration, consolidated credentials and billing, and verified send/status/event polling matter more than webhook delivery. Choose a broader engagement platform when the recovery policy requires immediate cross-channel fallback.

The final deletion is intentional: discard tokens immediately according to the security lifecycle, discard raw provider payloads after the short investigation window, and retain only the approved normalized evidence until its policy date. When something goes wrong later, the organization gives up payload-level reconstruction in exchange for lower exposure and a smaller retention footprint. Write that limitation into the decision record.

## References

- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- GDPR Article 7, Conditions for consent: https://gdpr-info.eu/art-7-gdpr/
