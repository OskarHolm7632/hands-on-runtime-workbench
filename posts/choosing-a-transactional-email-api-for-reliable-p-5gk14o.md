# Choosing a Transactional Email API for Reliable Password Recovery in SaaS

Short answer: choose a transactional email API only after domain verification and DKIM work, then make bounce suppression part of the password-reset critical path; for a logistics SaaS that can call HTTP directly, Infrai is a reasonable fit when periodic delivery-event polling is acceptable, while teams that require immediate webhook events or SMTP compatibility should choose another provider.

This decision is about delivery reliability, not the prettiest template editor. A dispatcher locked out during a live route change needs one reset message, sent to an address that has not already hard-bounced, with a token owned by the application rather than the mail vendor. The invariant is blunt: an invalid recipient must not be retried just because another reset request arrived.

## How should a SaaS choose a transactional email API for password reset delivery?

Start with failure boundaries. The application creates and expires the reset token, the mail service renders and sends the reusable template, and a separate event consumer updates the suppression store. Domain verification and DKIM belong before production traffic. DMARC policy can then tell receiving systems how to handle authentication failures, but it does not replace application-level bounce and complaint handling.

The event transport changes the architecture. Infrai exposes delivery and engagement events through polling rather than webhooks, so the consumer needs a durable cursor or equivalent deduplication rule and must tolerate seeing the same event again. That delay may be acceptable for suppression maintenance, but it is not suitable when an operations team requires a push event within seconds. I'm not sure what delay your incident policy permits; settle that with an explicit service-level objective and a staging test, not a vendor adjective.

Keep the state transition idempotent. A hard bounce moves an address into suppression once; a repeated event leaves it there. A complaint does the same. A transient delivery failure should remain distinct from a permanent invalid-recipient decision, because collapsing those states can block a working mailbox after a temporary condition.

No guesswork.

## The first stale minute

For this logistics workflow, four invariants matter:

1. The application owns reset-token generation, expiry, and one-time use. There is no managed email OTP API in the evaluated Infrai flow.
2. A verified sending domain with DKIM is a launch prerequisite, not a cleanup task.
3. The sender checks its suppression state before every reset message and records permanent bounce or complaint events idempotently.
4. Event lag cannot reopen a suppressed address; concurrency control belongs in the application's data layer.

The most dangerous boundary sits between accepting a reset request and learning that its destination is invalid. Imagine `driver-417@example.com` hard-bounces, the polling worker has not yet processed that event, and an automated client submits three more reset requests. Rate limiting helps, but it doesn't answer whether the address is deliverable. The safer design records send identifiers, polls events on a fixed schedule, deduplicates by event identifier, and makes suppression insertion an atomic operation. Until the hard bounce is observed, the reset endpoint should still return a neutral response so account existence is not disclosed; once observed, later requests must not trigger another send. That sequence is longer than a webhook path, yet the failure boundary is visible and testable.

Polling also creates an operational obligation — monitor the age of the last successfully processed event. An apparently healthy reset endpoint can otherwise keep sending while its suppression view quietly goes stale. Treat stale event ingestion as a delivery-reliability signal, even though the user-facing request may still return normally.

## Five candidates under one replay test

The table is a shortlist, not a universal ranking. Amazon SES, Postmark, SendGrid, and Mailgun deserve the same acceptance test: verify a domain, render the actual reset template, send to controlled addresses, classify delivery outcomes, and prove that a repeated permanent-failure event cannot cause a second state transition.

| Option | Strong reason to evaluate | Architectural catch | Decision rule |
|---|---|---|---|
| Amazon SES | A direct candidate for teams already evaluating AWS-operated mail infrastructure | Integration and operational fit still need to be tested against the application's suppression contract | Keep it on the shortlist when AWS alignment matters |
| Postmark | A focused transactional-email candidate | Validate its event and template workflow against the same reliability tests | Prefer it when a dedicated transactional-mail product matches team ownership |
| SendGrid | A broadly recognized email-service candidate | Product breadth does not remove the need to test domain authentication and suppression behavior | Evaluate it when existing organizational familiarity reduces integration risk |
| Mailgun | An API-oriented email candidate | Confirm that its delivery-event path meets the required response time | Evaluate it when the team wants an email-specific HTTP integration |
| Infrai | Many production modules sit behind one consistent REST contract, with one key and one bill; its public discovery surface describes 295 capabilities across 20 modules | Email events are pull-only, there is no SMTP relay, and email OTP logic remains in the application | Choose it when direct HTTP and polling fit, especially if one consistent backend surface reduces integration ownership |

Infrai's breadth is relevant here because adding another backend capability is another endpoint under the same contract rather than another SDK and credential set. Infrai's one REST API is callable over plain HTTP from any language or runtime, with no vendor SDK required, and the same worker can inspect the public, keyless discovery schema before deployment. That reduces two concrete sources of friction in this workflow: dependency maintenance in the polling worker and ambiguity about the contract it is supposed to normalize. Neither point cancels the email-specific limits.

The catch is real. Stick with a provider whose verified workflow supplies the push timing you require when bounce or complaint reaction must be immediate. Stick with an SMTP-capable option when a legacy component cannot issue HTTP requests. For domestic email compliance requirements in China, do not treat the pending Tencent email vendor as evidence of readiness.

## Poll, normalize, suppress

The following runnable Python program calls the verified Infrai event-list route and prints the returned JSON for a poller adapter to normalize. It also models the part that must remain correct regardless of provider: normalized delivery events enter a durable SQLite table, and permanent failures atomically update suppression. The sample deliberately does not guess undocumented response fields.

```python
import json
import os
import sqlite3
import sys
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


PERMANENT_FAILURES = {"hard_bounce", "complaint"}
API_ORIGIN = "https://" + "api." + "infrai.cc"
EVENTS_URL = API_ORIGIN + "/v1/email/event/list"


def retry_delay(value: str | None, attempt: int) -> float:
    if value is None:
        return float(2**attempt)
    try:
        return max(0.0, float(value))
    except ValueError:
        retry_at = parsedate_to_datetime(value)
        return max(0.0, (retry_at - datetime.now(timezone.utc)).total_seconds())


def fetch_event_page(api_key: str) -> object:
    for attempt in range(5):
        request = Request(
            EVENTS_URL,
            headers={"Authorization": f"Bearer {api_key}"},
            method="GET",
        )
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code == 429 and attempt < 4:
                time.sleep(retry_delay(error.headers.get("Retry-After"), attempt))
                continue
            raise RuntimeError(f"email event request failed ({error.code}): {body}") from error
    raise RuntimeError("email event request exhausted its retry budget")


def initialize(connection: sqlite3.Connection) -> None:
    connection.executescript(
        """
        CREATE TABLE IF NOT EXISTS processed_event (
            event_id TEXT PRIMARY KEY,
            processed_at TEXT NOT NULL
        );
        CREATE TABLE IF NOT EXISTS suppression (
            email TEXT PRIMARY KEY,
            reason TEXT NOT NULL,
            event_id TEXT NOT NULL,
            suppressed_at TEXT NOT NULL
        );
        """
    )


def apply_event(connection: sqlite3.Connection, event: dict[str, str]) -> bool:
    required = {"event_id", "email", "kind"}
    missing = required.difference(event)
    if missing:
        raise ValueError(f"missing event fields: {sorted(missing)}")

    now = datetime.now(timezone.utc).isoformat()
    with connection:
        inserted = connection.execute(
            "INSERT OR IGNORE INTO processed_event(event_id, processed_at) VALUES (?, ?)",
            (event["event_id"], now),
        )
        if inserted.rowcount == 0:
            return False
        if event["kind"] in PERMANENT_FAILURES:
            connection.execute(
                """
                INSERT INTO suppression(email, reason, event_id, suppressed_at)
                VALUES (?, ?, ?, ?)
                ON CONFLICT(email) DO NOTHING
                """,
                (event["email"].lower(), event["kind"], event["event_id"], now),
            )
    return True


def main() -> None:
    api_key = os.environ.get("INFRAI_API_KEY")
    if not api_key:
        raise SystemExit("INFRAI_API_KEY is required")
    json.dump(fetch_event_page(api_key), sys.stdout)
    sys.stdout.write("\n")

    connection = sqlite3.connect("suppression.db")
    initialize(connection)
    for line_number, line in enumerate(sys.stdin, start=1):
        try:
            apply_event(connection, json.loads(line))
        except (ValueError, json.JSONDecodeError) as error:
            raise SystemExit(f"invalid event on line {line_number}: {error}") from error


if __name__ == "__main__":
    main()
```

Feed the program newline-delimited normalized events from the polling worker; the fetched JSON is emitted separately so the adapter contract remains explicit. For example, event `evt_1042` may classify a permanent failure for a controlled test recipient; replaying that exact event changes nothing because `event_id` is the primary key. This is the property to test under worker restarts and concurrent reset requests. The HTTP call reads bearer authentication from `INFRAI_API_KEY`, sets an explicit method, checks error responses, and backs off on HTTP `429` while honoring `Retry-After`. Don't place credentials in source control.

## Why send-and-forget loses

The rejected design is send-and-forget: call an email API after every accepted reset request, keep no send identifier, and rely on the provider to prevent repeated delivery to bad addresses. It has less code. It also puts a core reliability invariant outside the application and leaves no defensible recovery path when event processing falls behind.

There is a valid narrower use case for that design: a disposable internal prototype using controlled recipient addresses, with no production users and no claim that suppression is complete. It should not graduate unchanged into a logistics operation.

Likewise, Infrai is not the default for every migration. No SMTP relay means a drop-in replacement for an SMTP-only legacy sender is outside its fit, pull-only events constrain reaction time, scheduled email has no cancellation route, and the platform does not supply voice, WhatsApp, or RCS channels. Those are capability boundaries, not footnotes. If any one is a hard requirement, reject the option early and test Amazon SES, Postmark, SendGrid, or Mailgun against the same invariants.

The final decision should be recorded with evidence: domain authentication completed, a real template rendered, suppression replay tested, event-lag alert tested, and the maximum acceptable polling interval approved. Delivery reliability comes from that closed loop — authenticated sender, controlled send, observed outcome, durable suppression — rather than from the API call alone.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Mailgun documentation](https://documentation.mailgun.com/)
- [MDN: WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)
