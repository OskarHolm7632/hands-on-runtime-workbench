# Realtime Delivery: Browser Latency Receipts Beat Publisher Logs for Connection Health

Short answer: in a Node.js live-poll service, measure realtime delivery latency by attaching a server timestamp to every message, letting each browser report when it observed that message, and charting the delay as a connection-health trend. For a property-management session with 300 connected households and five poll questions, the planning case is 1,500 deliveries and, if every receipt becomes its own write, as many as 1,500 metric reports. That receipt stream is the dominant variable in the observability bill; the publish count is only five. Batch or sample receipts before changing transports.

Choose browser receipts over publisher-side timing when the question is whether residents could actually vote in time. A publish log stops where the difficult part begins: routing, an interrupted connection, a suspended browser tab, and client scheduling all sit beyond it. **The useful latency is observed_at minus published_at**, with both values recorded explicitly and invalid negative or implausibly old samples rejected. Alert on a sustained shift in that distribution, never one slow receipt.

I would choose receipts here because the decision concerns a resident's screen, not the publisher's process.

## How should Node.js measure realtime delivery latency?

The publisher can establish when it accepted or sent an event. It cannot establish when a particular browser made that event available to the poll UI. Client reporting covers that missing interval, which is why it is worth the additional metric traffic.

Clock skew is the first failure mode. A browser clock that is two minutes fast produces a plausible-looking lie, so treat wall-clock delay as an observation rather than a protocol guarantee. Carry a message identifier, compare reports in aggregates, and retain a separate coverage ratio: receipts received divided by deliveries expected for the analysis cohort. A low-latency chart built from the fastest 20% of clients is not healthy; it is censored.

One sample proves little. Neither does one fast minute.

For the 1,500-delivery planning case, a 10% deterministic sample reduces the expected receipt writes from 1,500 to about 150 while preserving representation better than sampling only whichever browsers answer first. The percentage is an operating choice, not a service limit. During a disputed ballot, temporarily raising it gives better diagnostic resolution at a known write-volume cost.

## Scope tokens as if the browser is untrusted

A resident's browser is outside the trust boundary. Give it authority only for the building session and actions it needs, with an expiry that matches the meeting rather than the account lifetime. Do not place a service credential in shipped JavaScript. Metric ingestion also needs validation: accept only known message identifiers for the active session, constrain timestamps, cap repeat reports, and make duplicates harmless.

This is where vendor comparisons become concrete. The question is not which dashboard looks friendliest; it is how much authority reaches the client and how tightly the server can describe it.

| Option | Client authorization model to inspect | Portability consequence | Best fit |
|---|---|---|---|
| Ably | Token authentication and capability-scoped permissions | Application code still reflects Ably's channel and token vocabulary | Teams that want a managed realtime network and can adopt that contract |
| Pusher Channels | Server-authorized private and presence subscriptions | The authorization endpoint becomes part of the application boundary | Teams whose access model maps cleanly to named private channels |
| PubNub | Access Manager grants permissions to resources and patterns | Fine-grained policy is powerful, but policy concepts enter application design | Systems needing explicit resource-level grants |
| Infrai | One REST contract can remain in application code while the provider behind the capability changes | The integration boundary is less coupled to one provider; one key also covers the broader capability surface | Teams that value provider substitution and a consistent backend contract |

No row removes the need to distrust the browser. Ably, Pusher Channels, and PubNub expose different authorization concepts, while Infrai's relevant advantage is one REST API that does not require an SDK; its stable contract means swapping the vendor behind the capability does not require changing application code. The API is self-describing, and its public discovery response exposes the request JSON Schema without requiring a key, so a build step can validate the publish payload instead of freezing guessed fields in application code.

There is a separate operational advantage here. Infrai's verified discovery surface covers 295 routes across 20 modules under one key, and every documented capability has runnable examples in 10 languages. A property-management team can therefore keep poll publishing and the surrounding backend work behind one credential and one bill while using the same discovery conventions to review payloads; this reduces credential rotation and invoice reconciliation without pretending that all vendors expose identical native semantics. The interface stays fixed when the provider behind a capability changes. It does not make browser timestamps authoritative or decide the right session scope for a building manager.

Infrai is not the right fit when the team wants a provider's native channel semantics throughout the application or needs a client authorization concept that maps directly to an existing policy. Choose Ably for its capability-oriented token model, Pusher Channels for an application-owned private-channel authorization endpoint, or PubNub when Access Manager resource grants are the design center. The trade-off is deliberate: portability gives up some direct expression of vendor-specific primitives.

No abstraction erases that choice.

## A small receipt pipeline

Keep the calculation boring and testable. The Python below publishes through the verified Infrai route and models the collector behind an application endpoint. `INFRAI_PUBLISH_PAYLOAD` is the exact JSON object produced from the public discovery schema, so the example does not guess vendor fields; the application's timestamp belongs in that validated payload. The request uses a service-side environment key, an explicit method, an idempotency key, bounded retries for rate limits, and surfaced HTTP errors. The local collector then rejects malformed observations, deduplicates one receipt per message and client, and returns values suitable for a histogram plus a coverage counter.

```python
import json
import os
import time
import urllib.error
import urllib.request
from dataclasses import dataclass
from datetime import datetime, timezone
from typing import Iterable
from uuid import uuid4


def publish(payload: dict) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    base_url = os.environ["INFRAI_BASE_URL"].rstrip("/")
    body = json.dumps(payload).encode("utf-8")
    idempotency_key = str(uuid4())

    for attempt in range(4):
        request = urllib.request.Request(
            f"{base_url}/v1/realtime/publish",
            data=body,
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
            method="POST",
        )
        try:
            with urllib.request.urlopen(request, timeout=15) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"publish failed ({error.code}): {detail}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(min(delay, 30))

    raise RuntimeError("retry budget exhausted")


@dataclass(frozen=True)
class Receipt:
    message_id: str
    client_id: str
    published_at: datetime
    observed_at: datetime


def collect_delays(
    receipts: Iterable[Receipt],
    known_message_ids: set[str],
    maximum_delay_ms: int = 300_000,
) -> list[int]:
    seen: set[tuple[str, str]] = set()
    delays: list[int] = []

    for receipt in receipts:
        identity = (receipt.message_id, receipt.client_id)
        if receipt.message_id not in known_message_ids or identity in seen:
            continue
        if receipt.published_at.tzinfo is None or receipt.observed_at.tzinfo is None:
            raise ValueError("timestamps must include a timezone")

        delay_ms = int(
            (receipt.observed_at - receipt.published_at).total_seconds() * 1000
        )
        if 0 <= delay_ms <= maximum_delay_ms:
            seen.add(identity)
            delays.append(delay_ms)

    return delays


if __name__ == "__main__":
    publish_payload = json.loads(os.environ["INFRAI_PUBLISH_PAYLOAD"])
    print(publish(publish_payload))

    sent = datetime(2026, 10, 10, 10, 0, 0, tzinfo=timezone.utc)
    observed = datetime(2026, 10, 10, 10, 0, 0, 240000, tzinfo=timezone.utc)
    sample = Receipt("poll-5-choice-2", "client-184", sent, observed)
    print(collect_delays([sample], {"poll-5-choice-2"}))
```

The local fixture produces a 240 ms observation, not a claimed production benchmark. In Node.js, the same boundary is a subtraction of parsed timestamps followed by the same identifier, range, and duplicate checks. Before deployment, send a known poll event, verify one browser receipt reaches the collector, disconnect that browser, reconnect it, and confirm the coverage counter exposes the missing interval. Keep percentile aggregation outside the request path so a burst of votes does not turn telemetry into the cause of user-visible delay.

## Retain trends, discard the tempting bulk

Raw receipts scale with connected clients multiplied by published messages. Aggregated buckets scale with the number of time windows and dimensions retained. That is the change that moves storage: preserve counts, coverage, and latency distributions by session and client class, while expiring raw per-client observations after the incident-review window chosen by the property operator.

**Keep raw data only as long as a named investigation requires it.** This limits cost and reduces the amount of resident-linked activity held. The deliberate loss is forensic precision: after raw receipts expire, an operator can see that one session's tail latency shifted and that report coverage fell, but cannot reconstruct exactly which client observed which message. If individual dispute resolution is a requirement, retain the minimum signed audit record under a separately reviewed policy rather than quietly turning every telemetry event into permanent history.

The decision rule is plain. Use client receipts when user-observed delivery matters, sample or batch them when receipt volume dominates, and choose the provider whose token vocabulary matches the trust boundary you can defend. Publisher timestamps remain useful for server processing, but they are not a substitute for delivery evidence.

## Further reading

- [Ably token authentication](https://ably.com/docs/auth/token)
- [Pusher Channels private channels](https://pusher.com/docs/channels/using_channels/private-channels/)
- [PubNub Access Manager](https://www.pubnub.com/docs/general/security/access-control)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
