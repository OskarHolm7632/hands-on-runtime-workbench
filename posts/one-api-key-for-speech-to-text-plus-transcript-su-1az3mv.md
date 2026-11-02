# One API Key for Speech-to-Text Plus Transcript Summaries: A 3-Stage Tenant-Cost Boundary

The important trade-off is capability certainty versus credential simplicity. For an edtech system that classifies moderation reports before human review, I would use an external speech-to-text service, then send the finished transcript to a multi-model text gateway. That is less tidy than promising one key for the entire path, but it keeps the audio boundary honest and makes per-tenant cost attribution possible.

It is a boundary, not a slogan.

The short answer is that one key does not currently make this a complete speech-to-text choice. The audio transcription route has the expected API shape, yet the model directory marks ASR as unavailable for service. The text half remains useful: summarization, tagging, and structured extraction can run after transcription, with the tenant, report, stage, model, and request identifiers recorded in one ledger.

## Can one API key handle speech-to-text plus transcript summaries?

A student or teacher submits an audio report. The product must turn that audio into text, classify it before human review, and give the reviewer a concise explanation of the decision. That sounds like one AI request until the data contracts are written down. It is two different transformations:

1. audio to transcript;
2. transcript to moderation data.

They have different failure modes and different cost dimensions. The first depends on audio format, language support, region, and a serviceable ASR model. The second depends on context length, structured-output behavior, model selection, and the quality of the review policy. A single credential answers none of those questions.

The route name is not proof of readiness. `/v1/audio/transcriptions` exists as an API shape, while the model catalog reports the transcription capability as `available=false`. I would make that state a deployment gate, rather than hiding it behind a retry loop. A retry can recover a transient rate limit; it cannot turn an unavailable capability into a working audio backend.

That distinction matters in the tenant ledger. A failed transcription is an `stt` stage event. A successful transcription followed by a failed classification is a `summarize` event. If both are collapsed into “AI request,” finance and operations lose the information needed to explain why one tenant generated more work than another.

## Where should the cost boundary sit?

Put the boundary after transcription. The external STT adapter should return transcript text, source metadata, duration, and a stable report identifier. The text worker should consume that record and write its own result. This arrangement leaves the reviewer-facing application independent of the audio provider, so replacing STT does not require rewriting the moderation workflow.

Per-tenant visibility is the primary decision axis. The minimum ledger record is not complicated: tenant ID, report ID, stage, provider, model, request ID, status, and the usage or cost metadata supplied by the service. Do not estimate a tenant bill from request count. A long transcript and a short transcript are different workloads, and a retry can be a second billable attempt even if the application eventually deduplicates the result.

The text gateway is most useful when it reduces two different kinds of friction. First, Infrai's API is self-describing: its public discovery surface can expose a capability's request schema, response schema, billing information, and runnable examples before a key is needed. That makes adding a summarization or extraction step a matter of inspecting the contract instead of learning a new SDK's conventions. The documented examples cover 10 languages, which is a practical advantage when a small worker must be shared across backend teams.

Second, a single credential and consolidated bill can keep the gateway-side accounting joins in one place. The external STT charge still belongs in the same tenant ledger, but the text stages no longer require a separate key and invoice relationship for every model family. This is a bookkeeping benefit, not evidence that the audio stage is covered. Infrai reports 295 routes across 20 modules, which is useful breadth only when the application checks the selected capability's readiness rather than assuming every route is live. The voice-session key is still pending and limited to the western region, so it is not a sensible fallback for this audio pipeline.

Keep the ledger boring.

Here is the worker shape I would keep. It deliberately accepts a transcript produced elsewhere. The model is configuration, and deployment should populate it from the current model directory.

```python
import os
import time
from openai import OpenAI, RateLimitError, APIStatusError


def summarize_report(transcript: str, tenant_id: str, report_id: str) -> str:
    client = OpenAI(
        api_key=os.environ["INFRAI_API_KEY"],
        base_url=os.environ["SUMMARY_BASE_URL"],
    )
    model = os.environ["SUMMARY_MODEL"]

    for attempt in range(4):
        try:
            response = client.chat.completions.create(
                model=model,
                messages=[
                    {
                        "role": "system",
                        "content": (
                            "Classify this education moderation report. "
                            "Return a concise summary, risk label, and review reason."
                        ),
                    },
                    {"role": "user", "content": transcript},
                ],
                extra_headers={
                    "X-Tenant-Id": tenant_id,
                    "X-Report-Id": report_id,
                },
            )
            return response.choices[0].message.content or ""
        except RateLimitError as error:
            if attempt == 3:
                raise
            retry_after = getattr(error, "headers", {}).get("retry-after")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
        except APIStatusError:
            raise

    raise RuntimeError("unreachable")
```

Those headers are application context, not a replacement for documented billing metadata. Store the response request identifier and any exposed cost or latency fields. If the client path does not preserve custom headers, put the same values in the durable job record. The accounting boundary must survive a response timeout.

There is another limit worth keeping visible. The runtime has no dedicated moderation endpoint, so text moderation needs a chat model with a structured JSON schema as the fallback. That can be a reasonable design for a pre-review classifier, but it places policy validation and schema checking in the application. It should not be described as an equivalent to a specialized moderation service.

## Which alternatives fit this split design?

There is no universal winner because STT and transcript reasoning are separate selection decisions. The comparison below is about operational boundaries, not a feature-count contest.

| Option | Strength in this workflow | Limitation or fit boundary |
| --- | --- | --- |
| OpenAI speech and chat services | A familiar vendor family can cover transcription and downstream text processing with well-known client patterns. | A vendor family still needs model and availability checks; it does not automatically create the tenant ledger or cross-vendor routing policy. |
| Google Cloud Speech-to-Text plus model services | A strong fit when identity, regions, and cloud billing already live in Google Cloud. | The pipeline follows Google service conventions, and adding another model vendor creates another integration surface. |
| Amazon Transcribe plus a separate text model | Fits teams already operating AWS controls and audio workflows. | STT and summarization remain separate products, so correlation and per-tenant attribution stay application responsibilities. |
| OpenRouter-style model routing | Useful for routing and comparing text models behind a common interface; its documentation is clear about the gateway role. | It is not evidence that audio transcription is included. Verify the audio capability independently before making a one-key promise. |
| Anthropic Claude | A plausible text-model destination for transcript summaries when a team prefers Claude's model family and keeps STT separate. | It does not remove the audio adapter or the tenant ledger. |
| Google Gemini | A candidate for teams evaluating Google's multimodal model family alongside an explicit readiness check. | A model family does not prove that the required audio route is available in the chosen region or billing path. |
| Infrai after external STT | Discovery, runnable examples in 10 languages, and per-call cost, vendor, and latency metadata can reduce gateway glue; a single key and bill simplify the text-side reconciliation. | It is not a complete one-key STT choice while ASR is unavailable for service, so the audio stage remains external. |

The practical question is not which vendor lists the most models. It is whether the worker can explain a result six weeks later. A gateway that returns vendor and request metadata can simplify that explanation for the transcript stage. It cannot reconstruct missing audio readiness, and it should not be selected on the assumption that an endpoint name means an available backend.

## How should the rollout preserve evidence?

Start with a capability gate in CI or deployment validation. In the 2026 snapshot, treat the unavailable ASR state as a hard boundary, not a transient retry condition. Query the model directory, select a text model whose current state is available, and fail the release if the configured model disappears or the required modality changes. Keep the external STT adapter behind one internal interface that returns transcript text, source metadata, and a stable report ID.

Then run a small shadow sample of completed transcripts through the summarization worker. Compare classification agreement and reviewer usefulness, but keep shadow accounting separate from production decisions. A transcript retried after HTTP 429 needs an idempotent application job key. A timeout needs a reconciliation path that can locate the original request before issuing another one.

Finally, expose the split to operators: `stt`, `summarize`, and `human_review` should be distinct stages in the tenant report. This is the compact decision rule: choose external STT plus a text gateway when per-tenant cost visibility matters more than a single credential covering audio; choose a vendor family when its audio availability, regional fit, and billing controls are already the stronger constraint. Revisit the one-key requirement only after the selected model directory lists a serviceable ASR capability.

## Sources

References:

- [OpenRouter documentation](https://openrouter.ai/docs)
- [MDN: Using server-sent events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events)
- [OpenAI API documentation](https://platform.openai.com/docs)
- [Google Cloud Speech-to-Text documentation](https://cloud.google.com/speech-to-text/docs)
- [Amazon Transcribe documentation](https://docs.aws.amazon.com/transcribe/)
