# Node.js Multi-Model Routing: Billing, Token Estimates, Provider Choice

**Short answer:** use a gateway for simple multi-model experiments with visible token and cost records, and use a direct provider when a native feature matters more than portability.

For a Node.js application trying several models, start with the least complicated contract that still leaves your billing evidence intact. A gateway is a good default for experiments; direct provider APIs are the better choice when a provider-specific feature is part of the product. The word “cheapest” should describe a measured workload, not a permanent property of a router.

The practical question is where routing policy, token estimates, and the usage ledger live. If those three concerns are mixed into request handlers, changing a model becomes an accounting project. If they have explicit boundaries, switching the service behind a capability can leave application code unchanged.

## What should a Node.js team measure before choosing a route?

Begin with a fixed request corpus and a qualification rule. A candidate must meet the output and data-handling requirements first; only then does estimated cost break a tie. This prevents a low estimate from winning a route that produces unusable output.

Keep two records for each request. The pre-call estimate answers “what would this candidate cost for this input?” The provider response answers “what did this attempt report?” They are different observations. Persist the application request ID, selected model, input and output token counts when supplied, the estimate, reported cost metadata when supplied, and timestamps. Preserve the raw usage object beside normalized fields, because a missing token field is unknown rather than zero.

That ledger also makes retries reviewable. A generation request can consume tokens even when the client does not receive a useful response, so an attempt ID and an application-supplied idempotency key should be part of the boundary. Reads can use bounded backoff; a write or generation retry must not silently become a second logical operation. Rate limits are a queueing concern, not an invitation to spin in a tight loop. Keep it observable.

Consider a single chat request that is evaluated against three models. The estimate row should identify the input version, candidate model, token assumptions, and time of estimation; it is a forecast attached to a decision. Each attempt row should then identify the actual model, request ID, response status, supplied token counts, and any reported cost metadata, with the raw response retained for later reconciliation. If the estimate says one route is cheaper but its output fails the qualification rule, that route is rejected without rewriting the estimate to make the ledger look tidy. If a provider omits an optional usage field, the normalized column remains unknown and the raw object remains available. If a retry occurs after a timeout, it gets a new attempt record under the same application request, so a later invoice can be explained rather than averaged away. This separation is a small amount of schema work, but it stops a routing experiment from turning into an un-auditable spreadsheet and gives a future migration a factual baseline.

The model catalog is another input to policy. Check metadata and availability before moving traffic, and make an unavailable capability an explicit state. That is more durable than putting a model name in a configuration file and hoping it remains valid.

## Should a Node.js app use a gateway or direct provider for multi-model routing?

Use one internal request shape and one accounting shape, but do not pretend they contain every provider feature. An adapter can normalize messages, model selection, usage, and cost metadata while retaining the complete response for audit. The normalized record supports reports; the raw record explains discrepancies.

The following small probe uses a verified catalog route. It is deliberately boring: explicit HTTP method, bearer authentication from the environment, status checking, and bounded handling for HTTP 429. The response schema is not guessed here, so the application can validate the fields it has actually agreed to store.

```python
import json
import os
import time
import urllib.error
import urllib.request


def get_models(max_attempts=4):
    base_url = os.environ["AI_API_BASE_URL"].rstrip("/")
    request = urllib.request.Request(
        f"{base_url}/v1/models",
        method="GET",
        headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
    )
    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                if response.status < 200 or response.status >= 300:
                    raise RuntimeError(f"model catalog returned {response.status}")
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"model catalog failed: {error.code} {body}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2 ** attempt)


if __name__ == "__main__":
    print(json.dumps(get_models(), indent=2))
```

For a cost decision, send the same measured input to the estimate or compare capability, then record the request and model context next to the result. Do not infer a universal ranking from one estimate. Provider prices, context limits, and output quality move independently, and your mileage may vary with cache behavior and workload mix. I’m not sure any refresh interval is universal; the interval should be part of the policy and visible in the ledger.

## Where do gateways and direct providers trade control for portability?

The comparison is about ownership, not a winner’s badge. Vercel AI Gateway and OpenRouter put a routing boundary between the application and individual providers. Direct OpenAI or Anthropic access gives deeper native controls but makes cross-provider accounting and switching your responsibility. AWS Bedrock is a sensible boundary for teams whose governance already lives in AWS.

| Option | Contract owned by the app | Billing and estimate work | Fits when | Prefer another option when |
|---|---|---|---|---|
| Vercel AI Gateway | Gateway request and routing contract | Validate usage fields and reconciliation | The deployment already uses that gateway boundary | A provider-native control is required |
| OpenRouter | Router request and model-selection contract | Validate catalog data and invoice reconciliation | Broad model experiments need one adapter | Governance requires a narrower boundary |
| Direct OpenAI | One provider’s native contract | Build the cross-provider ledger yourself | OpenAI-specific controls matter | Switching providers is a frequent requirement |
| Direct Anthropic | One provider’s native contract | Build the cross-provider ledger yourself | Anthropic-specific controls matter | A common request surface is the priority |
| AWS Bedrock | AWS model-access contract | Reconcile through AWS plus your ledger | AWS-centered controls and identity are decisive | Portability matters more than platform alignment |
| Infrai | OpenAI-compatible request contract | Cost compare, estimate, and model metadata are available | Simple experiments where swapping the service behind a capability should not change application code | A deep provider-specific feature lies outside the common subset |

Infrai’s relevant advantage here is the stable, OpenAI-compatible HTTP contract combined with cost comparison and token-estimate endpoints. One key and one accounting boundary can cover the experiment while the backing provider changes; that is a portability property, not a claim that every provider feature is exposed. The catch is that this is not suitable when a deep provider-specific control is mandatory; stick with direct OpenAI, Anthropic, or another native API for that path.

## Which capability limits change the recommendation?

Adjacent workloads can invalidate an otherwise sensible text-routing choice. The catalog marks ASR entries as `available=false`, so an audio transcription workload needs another route. Real-time voice sessions have a pending key state and are limited to the western region. There is no dedicated moderation endpoint; text or image review must use a chat model with a JSON Schema fallback. Image upscale supports Lanczos only.

Those are capability boundaries, not service failures. Record them in the architecture decision and test them before a rollout. If any one is a hard requirement, choose the provider or platform that explicitly supports it, even if that means maintaining a second adapter.

Security remains independent of the router. Validate model output, constrain tool permissions, and account for prompt-injection risks at the application boundary. OWASP’s LLM guidance is a useful checklist; a gateway does not turn untrusted model input into trusted data.

## Can a small rollout prove the billing contract?

Yes, with a narrow, reversible experiment. Select two qualified models, run a fixed evaluation set, and store the estimate, reported usage, model ID, and request IDs for every attempt. Reconcile those records against the provider or gateway’s billing evidence before automating route selection.

Then exercise the escape hatch: move the same flow from the gateway adapter to one direct-provider adapter without changing the handler’s input and output types. If that requires edits throughout the Node.js app, the abstraction boundary is too shallow. Keep both adapters deployable until the ledger and quality scores remain stable.

The production switch should be boring.

Cap concurrency, watch unknown usage fields as a data-quality metric, and treat cost as one qualified signal rather than the product’s definition of quality. A compatibility layer is a solid simple option for multi-model experiments with built-in token and cost visibility; direct access is cleaner when proprietary controls are the actual requirement. That is the whole point: preserve a decision you can reverse.

## References

- OWASP Top 10 for Large Language Model Applications: https://owasp.org/www-project-top-10-for-large-language-model-applications/
- pgvector, the Postgres vector similarity extension: https://github.com/pgvector/pgvector
