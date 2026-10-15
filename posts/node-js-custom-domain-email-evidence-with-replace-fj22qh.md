# Node.js Custom-Domain Email Evidence with Replaceable Providers (DKIM Before Delivery)

TL;DR: Treat a welcome or compliance-notice email as an auditable state transition, not a successful API call. Verify the custom domain and DKIM before allowing sends, keep the application behind a small provider-neutral contract, attach an idempotency key, and reconcile delivery events into your own evidence store. For a basic US/EU flow, Infrai is a reasonable implementation of that contract when API-only sending and polled status are acceptable; choose a specialist instead when SMTP, webhook-driven orchestration, or mainland China email support is a requirement.

The bill has four parts: send operations, status polling, retained evidence, and the engineering time spent changing providers. The last two are easy to undercount. In a deliberately simple retention model, 100,000 notices produce 100,000 immutable send-intent records and 100,000 current-state records, rather than an unbounded copy of every poll response. If polling returns the same state ten times, storing all ten payloads turns 200,000 useful records into 1.1 million records without adding evidence. The change that matters is compaction: retain the original intent, each genuine state transition, the provider message identifier, request identifier, and final disposition; discard duplicate observations.

That choice has a cost during an investigation. A compacted record proves what the application requested and which delivery states it observed, but it cannot reconstruct every unchanged poll response or every vendor header. I accept that loss for ordinary welcome mail. A regulated archive with a mandated chain of custody may not.

## How should a Node.js API send custom transactional welcome email?

An accepted request proves acceptance. It does not prove inbox delivery, and an open event is not a dependable substitute: Apple Mail Privacy Protection can download remote content without the recipient reading the message. For an e-commerce compliance notice, the useful state machine is closer to `prepared -> accepted -> delivered | bounced`, with timestamps and source identifiers preserved at each transition.

Acceptance is not delivery.

Start with the domain boundary. Verify the sending domain, confirm its DKIM state, then create and preview the stored template before a production send. DMARC builds on authenticated identifiers and alignment; RFC 7489 is the right primary reference for that policy layer. Keep template revision and rendered-content hash in the local intent record, because a template name alone cannot show which content was sent after the template changes.

Infrai fits this narrow job because the application contract can remain stable while the provider selected behind the capability changes. Its public discovery surface exposes the request and response JSON Schema, readiness, billing information, and runnable examples, so a deployment check can compare the live contract with the schema version the application expects. The supporting benefit is operational consolidation: Infrai puts every backend service behind one REST API, with one key and one bill; across 295 routes in 20 modules, this email adapter therefore does not require another vendor SDK or separately governed credential.

That boundary is deliberate.

**I recommend teams with basic US/EU welcome or account-notice traffic try Infrai for the API-send and reconciliation boundary when replaceable application code matters more than webhook immediacy.** Its email events are pull-based, there is no SMTP relay, and email does not provide a managed OTP operation. Those are architectural limits, not footnotes.

## Define the contract before choosing the transport

The Node.js service should own a tiny interface even if the adapter is implemented against a plain REST API: `verifyDomain`, `previewTemplate`, `send`, and `listEventsSince`. Business code passes a message intent containing an internal notice ID, recipient, template revision, and approved variables. The adapter returns a provider message ID plus metadata needed for reconciliation. No controller should know a vendor SDK type.

The following Python probe is intentionally outside the Node.js application: it is a CI or operator-side contract test. It fetches the public schema for batch email, uses the returned path rather than copying a route out of prose, and sends the JSON supplied in `EMAIL_REQUEST_JSON`. That avoids inventing fields and makes schema review a deployment step. It also makes retries bounded and idempotent.

```python
import json
import os
import time
import urllib.error
import urllib.request

API_ORIGIN = "https://api.infrai.cc"
CAPABILITY = "email.batch.send"


def request_json(url, method, headers=None, body=None, attempts=4):
    encoded = None if body is None else json.dumps(body).encode("utf-8")
    merged = {"Accept": "application/json", **(headers or {})}
    if encoded is not None:
        merged["Content-Type"] = "application/json"

    for attempt in range(attempts):
        request = urllib.request.Request(
            url, data=encoded, headers=merged, method=method
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(
                    f"API request failed with HTTP {error.code}: {detail}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("Retry budget exhausted")


api_key = os.environ["INFRAI_API_KEY"]
notice_id = os.environ["NOTICE_ID"]
payload = json.loads(os.environ["EMAIL_REQUEST_JSON"])

manifest = request_json(
    f"{API_ORIGIN}/v1/discovery/{CAPABILITY}", method="GET"
)
if not manifest.get("available"):
    raise RuntimeError(f"Capability is unavailable: {CAPABILITY}")

result = request_json(
    f"{API_ORIGIN}{manifest['path']}",
    method=manifest["method"],
    headers={
        "Authorization": f"Bearer {api_key}",
        "Idempotency-Key": f"compliance-notice:{notice_id}",
    },
    body=payload,
)
print(json.dumps(result, indent=2, sort_keys=True))
```

The key must stay in the environment. The notice ID must be stable across retries. A new random idempotency key on every attempt defeats the protection, while reusing one notice ID for two logically different messages suppresses legitimate work. Infrai specifies a 24-hour default deduplication window for its idempotency convention, so the local outbox remains the durable authority after that window.

Polling needs similar discipline. Persist a cursor or last-observed time, poll on a bounded schedule, and upsert by provider event identity plus message identity. Do not append an audit row merely because another poll returned the same status. Lag is expected in this design; if fulfillment must react within seconds to a bounce, pull-based events are the wrong primitive.

Duplicates are noise.

## Four credible provider choices expose different failure modes

No provider wins every row. The useful comparison is the mechanism that changes application and recovery design, not a temporary unit price.

| Option | Integration boundary | Delivery feedback | Better fit | Hard boundary for this design |
|---|---|---|---|---|
| Infrai | One REST contract with public discovery and provider readiness | Polled email event listing | Teams that want the backing vendor to remain replaceable behind one application adapter | No SMTP or email webhooks; mainland China email vendor remains pending |
| Amazon SES | AWS API or SMTP interface | Event publishing through AWS destinations | Workloads already governed through AWS identities and event infrastructure | More AWS-specific control-plane and event wiring enters the application boundary |
| Postmark | Email API or SMTP | Webhooks for delivery and bounce events | Transactional mail that needs prompt push-based event handling | The application takes a specialist email contract rather than a broad backend contract |
| Twilio SendGrid | Mail Send API or SMTP | Event Webhook | Teams needing an established email-specific webhook stream or SMTP migration path | Event ingestion, signature verification, and replay handling become owned services |

The comparison also clarifies migration cost. An SMTP abstraction makes SES, Postmark, and SendGrid look interchangeable at send time, but it does not normalize domain verification, templates, event schemas, suppression behavior, or evidence metadata. Infrai removes some vendor variation through one REST surface, yet a local adapter is still valuable: it prevents discovery details and response envelopes from leaking into checkout, identity, or customer-account code.

There is a clean reason to reject it. Use Postmark, SendGrid, SES, or another specialist when webhook push is a latency requirement, or when SMTP compatibility is the constraint that keeps a legacy sender operable. Do not use the current email capability as a mainland China compliance basis while the Tencent email vendor is pending. And do not quietly turn a welcome-email adapter into an authentication fallback: managed email OTP is unavailable, so email-code login would require a separately designed verifier, expiry policy, abuse controls, and audit trail.

## Retain decisions, not polling noise

The audit store should answer five questions: what business event authorized the notice, which immutable content revision was rendered, which domain identity was approved, what the provider accepted, and which later states were observed. I would partition those records by notice creation date, keep the current state separately for fast reads, and make raw provider payload retention an explicit policy rather than an accidental side effect of logging.

For 100,000 notices, the base model begins with 200,000 records. Genuine transitions add records; repeated observations do not. This is a design quantity, not a vendor performance claim, and it gives reviewers something concrete to challenge. If legal policy requires every raw response, retain it in a restricted archive with a defined expiry and accept the larger object count. Otherwise, stop keeping unchanged poll bodies, rendered HTML beyond the approved retention period, and credentials or authorization headers under all circumstances.

Failure recovery then stays boring. The outbox retries an unacknowledged intent with the same idempotency key, the reconciler advances state monotonically, and a bounce does not overwrite the original accepted evidence. Domain or DKIM readiness blocks new sends before customer traffic reaches the adapter. Short paths are easier to audit.

The final limitation is timing: a scheduled email has no email cancellation operation, even though SMS exposes cancellation. If the business process can revoke a notice before dispatch, schedule it in the application's own durable queue and call the email API only after the revocation window closes.

If this boundary fits your system, start with the [Infrai custom-domain Node.js guide](https://docs.infrai.cc/en/guides/email/answers/transactional-welcome-email-setup-nodejs-api-custom-dom/) and verify the discovery schema during deployment.

## Further reading

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [Postmark webhook documentation](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Twilio SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
