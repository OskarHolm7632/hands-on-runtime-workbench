# Nodejs Password Reset Email API: Transactional Evidence Without Retaining Token Links

TL;DR: In a US/EU fintech system that sends an order receipt after payment settles, use the same API-based transactional-mail boundary for password recovery: authenticate a custom sending domain, keep the reset token in application-owned state, check suppression before sending, and record delivery outcomes separately. The cheapest evidence record to operate is also the safer one: retain identifiers, decisions, template versions, and outcomes, but discard the rendered message, plaintext token, and reset URL.

Infrai's concrete interface advantage is one plain REST API with no SDK to install, and its separate operational advantage is one API key and one bill across 295 routes in 20 modules instead of separate credentials and invoices. For the receipt and recovery workflows, that means one credential-rotation control and one billing trail rather than another pair of exceptions. The API is genuinely self-describing, and its discovery surface is public with no key required; it supplies request and response schemas, billing information, and runnable examples, so a control owner can inspect the contract before approving it. Every documented capability also includes runnable examples in 10 languages. That breadth, credential consolidation, and inspectability reduce integration inventory and review friction; they do not remove the need for application-owned tokens, polling, or regional and contractual diligence.

Start with the storage bill. Its dominant term is rarely the compact delivery event; it is the accidental copy of message content attached to every audit row. For an illustrative 10 million events per month retained for 400 days, a 2 KB normalized record is about 263 GB before indexes and replicas, while a 100 KB rendered-message copy is about 13.2 TB. These are capacity-model inputs, not vendor measurements. Removing bodies changes the dominant term by 50:1, which matters more than arguing over an email API's changing unit price.

The least complex defensible design is one application-owned reset-grant table, one immediate email send, and one append-only evidence stream shared with settled-payment receipts. Password recovery and receipts may share delivery infrastructure, but they must not share retention assumptions: a receipt is business content; a reset link is a bearer credential.

## How should Nodejs send a transactional password reset email through an API?

An auditor needs to establish who or what initiated the workflow, which policy was applied, which sending identity and template version were selected, when the provider accepted the request, and what delivery or bounce state was later observed. The evidence record should also show whether the application consumed, expired, or invalidated the reset grant. None of those questions requires preserving the live link.

The sender below keeps the provider payload in an environment variable because the application should validate it against the public discovery schema rather than embedding undocumented fields in a shared client. It makes the write idempotent, states the HTTP method, reports the response body on failure, and backs off on HTTP 429. Set `EMAIL_API_BASE_URL` to the service base, provide a JSON `EMAIL_SEND_PAYLOAD` containing the verified template and recipient fields, and use a reset-request identifier that remains stable across retries.

```python
import json
import os
import time
import urllib.error
import urllib.request
from urllib.parse import urljoin


def send_reset_email() -> dict:
    base_url = os.environ["EMAIL_API_BASE_URL"].rstrip("/") + "/"
    payload = os.environ["EMAIL_SEND_PAYLOAD"].encode("utf-8")
    request_id = os.environ["RESET_REQUEST_ID"]
    url = urljoin(base_url, "v1/email/send")

    for attempt in range(5):
        request = urllib.request.Request(
            url,
            data=payload,
            method="POST",
            headers={
                "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
                "Content-Type": "application/json",
                "Idempotency-Key": request_id,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(
                    f"email send failed ({error.code}): {body}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(min(delay, 30.0))

    raise RuntimeError("email send retry budget exhausted")


if __name__ == "__main__":
    print(json.dumps(send_reset_email(), indent=2))
```

A practical record contains a pseudonymous account reference, workflow type, request and correlation identifiers, template version, sending domain, provider message identifier, timestamps, suppression decision, and normalized outcome. The token store is separate. It holds a cryptographic digest, purpose, expiry, and an atomic consumed marker; issuing a replacement should invalidate the earlier live grant. The request endpoint should return the same public response for known and unknown addresses, and logs should exclude query strings.

One-time means one-time. Keep it small.

SPF, DKIM, and DMARC belong in the control evidence, not merely in a launch checklist. Retain approval of the expected DNS configuration and periodically compare observed state with that expectation. DMARC adds identifier alignment, policy, and reporting semantics; a vendor-console badge alone does not explain which identifiers aligned when a message was evaluated.

For payment receipts, retaining an approved content snapshot may be justified by the business record policy. For reset mail, retain the immutable template version and a non-secret merge-field manifest if reconstruction is required, under restricted access, but deliberately stop keeping the rendered body and bearer URL. The cost during an investigation is clear: operators can prove issuance, provider acceptance, later outcome, and token consumption, but cannot replay the exact URL the user received. That loss is desirable unless a specific legal requirement says otherwise.

## The dominant retention term is under application control

Treat retention as a multiplication problem before choosing a database: events per period times bytes per event times retained periods, then add indexes, replicas, and backup policy. Provider charges remain a separate operating expense tied to sends and the current billing model. Combining the two in one vague "email cost" number hides the part engineering can reduce without changing vendors.

| Evidence choice | What it proves | Retention consequence | Failure cost |
| --- | --- | --- | --- |
| Normalized decision and outcome | Policy, request, template, provider ID, state transition | Small, stable rows | Cannot reconstruct visual rendering |
| Immutable template plus non-secret fields | Approved content version | Moderate and deduplicated | Requires a renderer to reconstruct content |
| Full rendered message | Exact body sent | Message size dominates storage | Expands sensitive-data scope |
| Plaintext token or reset URL | Exact bearer credential | Small in bytes, severe in exposure | A reader may gain account-reset authority |

Do not optimize away the correlation fields. A compact record that cannot join application decisions to provider outcomes is cheap but useless. Conversely, storing every MIME body "just in case" makes backups, deletion requests, and access reviews carry data that the evidence question did not require.

The same distinction helps with settled-payment receipts. Store the settlement identifier, approved receipt-template version, send request, and delivery state in the shared evidence stream; apply the receipt's content-retention policy in a separate governed store. That prevents the password-reset subsystem from inheriting broad access merely because both workflows send email.

## Delivery observation is delayed, so design for it

Before a reset send, check the recipient against suppression state. Repeated attempts to a blocked or bounced address should not cause repeated provider calls, although the public recovery endpoint should still give its neutral response. Use a stable idempotency key for the write because a timeout leaves the caller unable to know whether the provider accepted the first attempt.

Email events on the unified platform are pull-based rather than webhook-pushed. A poller therefore needs a durable checkpoint or overlapping time window, idempotent upserts, and an explicit staleness objective. Delivery state will lag by the polling interval. Password acceptance must not wait for a later delivery event, because provider acceptance does not prove that a person read the message; event data is for operations, suppression maintenance, and compliance evidence.

There is no managed email OTP endpoint, so the application must generate and validate the reset credential. There is also no SMTP relay. Scheduled email has no cancellation route, which makes future scheduling a poor match for short-lived recovery credentials: send immediately and let application expiry remove authority. These limits are architectural boundaries, not details to discover during an incident review.

## Compare operating models, not feature counts

All four choices below can deliver a password-reset message while the application owns token security. The meaningful difference is the evidence and event-processing machinery the team must operate around the send.

| Service | Strong fit | Boundary that affects this design |
| --- | --- | --- |
| Amazon SES | Teams already standardizing identity, event publication, and operations in AWS | The team assembles several AWS controls around the mail service |
| Twilio SendGrid | Teams wanting an email-focused API, hosted templates, and pushed event notifications | Webhook authentication, deduplication, ingress availability, and another vendor contract remain application concerns |
| Postmark | Teams prioritizing transactional-email focus and webhook delivery events | Its narrower scope means adjacent backend services keep separate credentials and bills |
| Unified multi-service REST platform | Teams reducing integration inventory across email and other backend modules | Email outcomes require polling; SMTP relay and managed email OTP are unavailable |

SES makes sense when IAM, SNS, and AWS-native operational assembly are already accepted organizational machinery. SendGrid and Postmark can reduce observation lag with event webhooks, but a webhook is not free evidence: the receiver must authenticate callbacks, survive retries, deduplicate events, and prove durable processing. Pulling removes a public callback receiver while adding checkpoint recovery and bounded status lag. Neither mechanism is inherently more compliant. The tested chain from decision to durable evidence is what matters.

The main limitation is explicit: Infrai is a poor fit when SMTP relay, push-based delivery events, managed email OTP, voice, WhatsApp, or RCS is mandatory. Choose SendGrid or Postmark when pushed email events are a firm requirement, or SES when the organization already accepts the AWS-native assembly. It is also insufficient evidence for China compliance while the domestic email vendor remains pending. Provider availability never proves residency, regulated-outsourcing, subprocessors, transfer, deletion, or retention terms; legal and security owners still need those artifacts for the actual processing arrangement.

Polling is the trade-off.

## Set a retention rule that admits what it loses

For recovery mail, keep security decisions and provider outcomes long enough to satisfy the approved audit and investigation policy, then delete them on schedule. Keep reset grants only through their short operational lifetime plus the narrowly justified security window, preferably as digests throughout. Receipt records can follow a separate financial-record policy even when both workflows use the same sender and evidence schema.

Test four gaps before launch: a send timeout followed by an idempotent retry; a suppressed recipient; a poller outage followed by checkpoint recovery; and two concurrent attempts to consume the same grant. The final case must permit one password change, not two. Also test deletion against replicas and backups, because a retention statement that covers only the primary table is incomplete.

The deliberate decision is to stop retaining rendered reset messages, plaintext tokens, and URLs. This lowers the dominant storage term and shrinks the credential-exposure surface. It also means support and audit staff cannot reconstruct the exact link after the fact. Preserve the decision, template version, non-secret inputs, delivery observation, and consume result instead; those are the facts needed to explain the workflow without preserving authority to repeat it.

## Further reading

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [Twilio SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark webhook overview](https://postmarkapp.com/developer/webhooks/webhooks-overview)
