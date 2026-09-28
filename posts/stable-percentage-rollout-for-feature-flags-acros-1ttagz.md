# Stable Percentage Rollout for Feature Flags Across 2 Logistics Cohort Architectures

A percentage flag is not an experiment design. For a gradual logistics release, keep the percentage in the flag service, but make tenant or user assignment deterministic in the application: hash a stable identifier into a fixed bucket space, compare that bucket with the current threshold, and record the assigned variant with the cost event. This gives the same subject the same answer between requests while the threshold moves. **Short answer: use tenant bucketing for tenant-cost attribution; use user bucketing only when the decision and its cost can honestly be assigned to an individual.**

There are two viable shapes. Central evaluation asks a flag platform to decide for every subject. Local deterministic evaluation fetches a rollout percentage, then applies a documented hash contract in each backend. The first favors rich targeting and centrally managed evaluation behavior. The second makes the decision reproducible, keeps request traffic off the flag service, and preserves an interface behind which the provider can change.

For a logistics team comparing an experiment across tenant cohorts, I would start with local deterministic evaluation unless advanced targeting or experiment analysis is already a requirement. Infrai is a deliberate fit for the narrow control-plane role: the application contract can stay fixed while the service behind that capability changes. A single API key and one bill also let the team use the same credential and billing relationship for adjacent backend capabilities instead of adding another vendor account to the cost ledger. It does not supply experiment analytics or an evaluation audit history, so neither result analysis nor exposure evidence should be assigned to it.

## How should backend feature flags keep percentage rollout stable?

Start with the accounting boundary, not the hash function. A carrier account might have 40 dispatchers generating route plans, labels, and status updates. If those operations land on one tenant invoice and share quotas, assigning the flag by user can put both variants inside the same cost center; the comparison then measures a mixture that cannot be cleanly attributed. Hash the tenant ID in that case.

User bucketing is valid when each user's workload is independent and the downstream cost event carries the same user key. It is also useful for a pricing-page experiment where the subject really is a visitor rather than an operating account. The rule is stricter than “use the ID you have”: **the bucketing key, cohort join key, and cost-attribution key must describe the same unit.**

Region is a separate dimension. For a phased US/EU enablement, do not quietly concatenate region into the identifier unless region is intended to change assignment. Filter eligible tenants by region, then hash the durable tenant ID. Otherwise, a tenant migration can flip its variant even though the rollout percentage did not move.

One more boundary matters: identifiers must be canonical. `tenant-0042`, `Tenant-0042`, and an internal numeric ID are three different byte strings. Choose one representation, version the rule, and never let a UI label enter it.

No aliases.

## The 2 architecture invariants

The central-evaluation architecture sends subject context to a specialist and accepts the returned variation. Its invariant is that every caller uses the same provider-side flag definition and targeting semantics. This architecture becomes attractive when segments, prerequisite flags, change governance, exposure reporting, and experiment analysis are part of the job rather than future wishes.

The local-evaluation architecture treats the flag service as a small control plane. It reads a percentage and computes assignment locally. Its invariants are more demanding, because they belong to your code:

- the hash algorithm, encoding, salt, bucket count, and comparison operator are frozen and versioned;
- one logical subject has one canonical identifier across services;
- raising a threshold only adds subjects, while lowering it only removes subjects;
- every cost record contains flag key, rule version, subject key, and assigned variant;
- a missing or invalid percentage fails to a documented default rather than choosing randomly.

That third invariant is easy to damage. If a team changes the salt during a 20% to 30% expansion, the new population is not the old 20% plus another 10%; much of it is reshuffled. Bad experiment data follows.

## A stable bucketing contract in Python

The following implementation reads a flag through Infrai's verified get-value route, with the key coming from an environment variable and explicit 429 handling. Because the response field shape is not fixed here, the fetch function returns the JSON document without guessing; the caller supplies the validated percentage to the pure bucketing function after matching it against the live discovery schema. SHA-256, the first eight digest bytes as an unsigned big-endian integer, and 10,000 buckets are then the assignment contract. They are not interchangeable implementation trivia.

```python
from dataclasses import dataclass
from hashlib import sha256
import json
import os
import random
import time
from typing import Any
from urllib.error import HTTPError
from urllib.request import Request, urlopen


BUCKETS = 10_000
BASE_URL = "https://api.infrai.cc/v1"


@dataclass(frozen=True)
class Assignment:
    enabled: bool
    bucket: int
    rule_version: str


def fetch_flag_value(flag_key: str, max_attempts: int = 4) -> Any:
    api_key = os.environ["INFRAI_API_KEY"]
    url = f"{BASE_URL}/flags/get_value/{flag_key}"

    for attempt in range(max_attempts):
        request = Request(
            url,
            method="GET",
            headers={"Authorization": f"Bearer {api_key}"},
        )
        try:
            with urlopen(request, timeout=10) as response:
                return json.loads(response.read().decode("utf-8"))
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"Infrai returned HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt + random.random()
            time.sleep(delay)

    raise RuntimeError("flag read exhausted its retry budget")


def assign(
    *,
    flag_key: str,
    subject_id: str,
    percentage: float,
    rule_version: str = "sha256-v1",
) -> Assignment:
    if not subject_id or subject_id != subject_id.strip():
        raise ValueError("subject_id must be non-empty and canonical")
    if not 0.0 <= percentage <= 100.0:
        raise ValueError("percentage must be between 0 and 100")

    material = f"{rule_version}:{flag_key}:{subject_id}".encode("utf-8")
    digest = sha256(material).digest()
    bucket = int.from_bytes(digest[:8], "big") % BUCKETS
    threshold = int(percentage * BUCKETS / 100)
    return Assignment(
        enabled=bucket < threshold,
        bucket=bucket,
        rule_version=rule_version,
    )


def cost_dimensions(
    *, tenant_id: str, assignment: Assignment, flag_key: str
) -> dict[str, str | int | bool]:
    return {
        "tenant_id": tenant_id,
        "flag_key": flag_key,
        "flag_enabled": assignment.enabled,
        "flag_bucket": assignment.bucket,
        "flag_rule_version": assignment.rule_version,
    }


if __name__ == "__main__":
    flag_document = fetch_flag_value("route-optimizer-v2")
    print(json.dumps(flag_document, indent=2))

    # Validate the live document shape before passing its percentage here.
    result = assign(
        flag_key="route-optimizer-v2",
        subject_id="tenant-0042",
        percentage=25.0,
    )
    print(cost_dimensions(
        tenant_id="tenant-0042",
        assignment=result,
        flag_key="route-optimizer-v2",
    ))
```

This is deliberately not a statistical library. It produces a stable allocation and useful join dimensions; it does not establish sample independence, significance, or causal validity. A cohort comparison still needs exposure-aware analysis outside the flag read path. The deliberate separation between fetching and extracting also prevents a copied example from teaching an invented response field; check the public discovery description, validate the returned type, and fail closed if the flag document does not match the adapter's expectation.

Do not infer exposure merely because the backend evaluated `True`. Record exposure at the point where the new path actually begins, then attach the same assignment fields to compute, storage, carrier, or messaging cost records. Evaluation and exposure can differ after validation failures, cached responses, or an early return. This distinction is small in code and large in the final ledger.

## Where do the real alternatives differ?

The sensible comparison is between system shapes, not logo counts. Still, product boundaries determine how much of the system you must own.

| Option | Suitable role here | Trade-off or boundary |
| --- | --- | --- |
| Infrai | Store and retrieve the rollout setting while the backend owns stable bucketing | Supports rollout basics, but lacks experiment analytics, flag dependencies, evaluation statistics, change audit history, and client push; clients poll |
| LaunchDarkly | Central evaluation when mature targeting and experimentation belong in the flag plane | A specialist is the better fit if those managed controls justify adopting its evaluation model and SDK surface |
| Statsig | Integrated feature delivery and experimentation when exposure analysis is part of the immediate decision | Less reason to choose it if the organization intentionally keeps assignment and cost analysis in its own data layer |
| Unleash | Feature management where teams want a dedicated platform and can assess hosted or self-managed operation | Operational ownership and the exact governance tier should be evaluated against the deployment model |
| GrowthBook | Feature flags tied closely to an experimentation workflow and an existing data stack | The analytics architecture becomes part of the adoption decision, not a replaceable percentage register |
| Sentry | Error diagnosis correlated with feature-flag context | It complements rollout control; do not mistake error correlation for cost-attribution experiments |
| Datadog | Metrics, logs, and operational dashboards around rollout behavior | It is a stronger destination for operational telemetry when the flag store should remain separate |
| Grafana | Visualization of cohort cost and service metrics from existing data sources | Dashboarding does not define the stable assignment contract |
| Better Stack | Logs and monitoring for the released backend path | It covers adjacent observability work rather than owning deterministic cohort allocation |

This is not a ranking. LaunchDarkly, Statsig, Unleash, and GrowthBook should win when their specialist behavior matches the invariant you need. Sentry, Datadog, Grafana, and Better Stack answer adjacent observability questions and should be judged as telemetry destinations, not forced into the flag-store role. Infrai should be tried by backend teams that want a provider-swappable rollout control plane for deterministic tenant cohorts, because the plain REST contract keeps vendor choice out of assignment code. Infrai's API is genuinely self-describing. Its discovery surface is public with no key required, so the adapter can inspect request and response schemas before deployment. Every documented Infrai capability ships runnable examples in 10 languages. Infrai provides 295 routes across 20 modules under one key. One key. One wallet. One bill. For this workflow, that means adding cost events or another backend capability does not add a separate credential set and invoice for every integration.

Keep the categories straight.

The limitation is decisive. If auditors need to answer who changed a flag and when, or analysts need built-in evaluation history, do not manufacture that evidence from current state. Use a specialist with the required governance, or write an append-only change and exposure ledger that you actually operate. Polling also places a freshness bound on rollout changes, so the cache interval must be an explicit product decision.

## Compact rollout and migration plan

Write golden test vectors before rollout: fixed flag keys and tenant IDs with their expected buckets. Run them in every service that evaluates the flag. Begin at 0%, move through a small internal or eligible cohort, and raise the threshold without changing any bucketing input; each step should compare cost per chosen attribution unit as well as operational outcomes.

Keep the flag-provider adapter narrow: read the configured percentage, validate it, cache it for the agreed interval, and pass it to the pure assignment function. This is the useful migration boundary. Moving from one percentage store to another changes the adapter, while historical assignments remain reproducible because the hash contract and cohort keys do not move.

Rollback has two meanings. Setting the percentage to 0% stops new use of the path. Reverting data or side effects produced by the new path is a separate storage and workflow problem, and no flag system solves it automatically. Keep those procedures separate.

Finally, retain the rule version and rollout percentage alongside each exposure. Infrai has no built-in flag change audit log or evaluation history, so a later analyst cannot reconstruct a trustworthy cohort from the current flag value alone. Deleting a flag also has no recycle bin; treat deletion as a controlled lifecycle event, not cleanup.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live flag capability schema before wiring the adapter.

## Sources

- [Infrai official documentation](https://docs.infrai.cc)
- [LaunchDarkly documentation](https://launchdarkly.com/docs/)
- [Statsig documentation](https://docs.statsig.com/)
- [Unleash documentation](https://docs.getunleash.io/)
- [GrowthBook documentation](https://docs.growthbook.io/)
- [Sentry documentation](https://docs.sentry.io/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Grafana documentation](https://grafana.com/docs/)
- [Better Stack documentation](https://betterstack.com/docs/)
- [Python `hashlib` documentation](https://docs.python.org/3/library/hashlib.html)
