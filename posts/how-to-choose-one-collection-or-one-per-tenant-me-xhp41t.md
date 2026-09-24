# How to Choose One Collection or One per Tenant — Metadata Filter Tradeoffs

**Short answer:** For a multi-tenant SaaS help center, start with one collection and mandatory tenant metadata filters; move to one collection per tenant only when measured workload isolation, retention, or restore requirements justify the extra index cost.

The dominant cost is usually duplicated searchable state: vectors, text, metadata, and index structures multiplied by replicas and retained generations. A collection per tenant can turn a modest corpus into thousands of tiny indexes whose fixed overhead and rebuild work are harder to amortize.

**The practical rule is to partition for an operational boundary, not merely because a tenant exists.** Keep a recoverable source of truth outside the search index, treat filtered retrieval as an authorization-sensitive operation, and make the migration unit explicit from day one. This preserves a path from shared to isolated storage without paying the isolated-layout cost for every small account.

## What is the index bill actually made of?

Model the bill before choosing a topology. Marketing labels such as “collection” and “namespace” obscure the terms that matter: the number of stored chunks, bytes per vector, indexed metadata, text retained for responses, replication factor, and the number of complete generations kept during rebuilds. Provider-specific compression and graph overhead vary, so a portable estimate should expose those as measured inputs rather than pretend that one universal multiplier exists.

For an e-commerce help center that aggregates manuals, marketplace policies, seller FAQs, and returns guidance, duplicate content is an immediate warning sign. If a common shipping policy is copied into 800 tenant collections, its embeddings and index entries are stored 800 times. In a shared collection, one cannot automatically share that record if each tenant may customize or delete it; deduplication must respect provenance and access. The cost question is therefore not “How many tenants?” but “How many independently retained copies and index generations?”

The following estimator deliberately reports bytes, not currency. Rates change. Storage shape is the useful comparison.

That distinction matters.

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class IndexPlan:
    chunks: int
    dimensions: int
    bytes_per_component: int
    payload_bytes_per_chunk: int
    measured_index_multiplier: float
    replicas: int
    retained_generations: int

    def estimated_bytes(self) -> int:
        vector = self.dimensions * self.bytes_per_component
        searchable_generation = int(
            self.chunks
            * (vector + self.payload_bytes_per_chunk)
            * self.measured_index_multiplier
        )
        return searchable_generation * self.replicas * self.retained_generations


shared = IndexPlan(
    chunks=4_000_000,
    dimensions=768,
    bytes_per_component=4,
    payload_bytes_per_chunk=900,
    measured_index_multiplier=1.25,
    replicas=2,
    retained_generations=2,
)

isolated = IndexPlan(
    chunks=4_600_000,  # Includes duplicated cross-tenant source material.
    dimensions=768,
    bytes_per_component=4,
    payload_bytes_per_chunk=900,
    measured_index_multiplier=1.25,
    replicas=2,
    retained_generations=2,
)

for name, plan in {"shared": shared, "isolated": isolated}.items():
    print(name, round(plan.estimated_bytes() / 2**30, 1), "GiB")
```

The numbers above are inputs to a scenario, not a benchmark. Replace `measured_index_multiplier` with a value observed from a representative build, and count staging or blue-green generations if they coexist with the active one. That last term is easy to omit from a spreadsheet and expensive to discover during a full re-embedding.

## Should one collection use metadata filters or one collection per tenant?

A metadata filter is useful only if it is mandatory and applied before candidates can escape the tenant boundary. Post-filtering a global top-k result is wrong: relevant in-tenant chunks may already have been displaced, and an unfiltered candidate can leak into logs, caches, rerankers, or prompts. The retrieval interface should accept tenant context from trusted application state, never from an unconstrained model-generated string.

One collection and one collection per tenant make different failures cheap:

| Decision axis | Shared collection with tenant filter | Collection per tenant |
|---|---|---|
| Searchable-state duplication | Lower when common infrastructure and generations are shared | Higher when common records or fixed index overhead repeat |
| Noisy-neighbor control | Requires quotas, admission control, and workload measurements | Easier to isolate per-tenant rebuild and query work |
| Restore scope | A shared generation may be rebuilt while tenant visibility is gated logically | A tenant can be rebuilt or restored as a separate unit |
| Small-tenant efficiency | Usually better because fixed overhead is amortized | Often poor when many collections remain tiny |
| Deletion and retention | Requires a verified tenant-scoped delete plus rebuild policy | Physical unit can align with a distinct retention boundary |
| Operational cardinality | Few collections, stronger filter invariants | Many collections, more lifecycle and reconciliation work |

Choose isolation when a tenant has a genuinely distinct retention clock, encryption boundary, residency constraint, restore objective, or sustained workload that would distort shared capacity. Choose sharing when tenants are numerous and small, policy is uniform, and the team can prove filter enforcement at every retrieval path. Neither layout removes the need for authorization; separate names are not a substitute for checking who may address them.

The limitations are concrete. A shared collection is unsuitable when physical separation is a contractual requirement or when one tenant's rebuild must never contend with another tenant's queries. Per-tenant collections are unsuitable when the platform cannot reliably create, monitor, upgrade, and delete a high cardinality of small indexes. Those trade-offs remain even if the query API makes both layouts look identical.

There is also a hybrid worth designing explicitly: pooled collections for the long tail and dedicated collections for tenants that cross a documented threshold. Avoid a threshold based on account tier alone. Use observed searchable bytes, query concurrency, ingestion rate, rebuild duration, and policy boundaries, because those variables create the operational pressure.

## How do you make the shared path hard to misuse?

Put tenant scope in the repository boundary, then test the boundary rather than relying on every caller to remember a filter. The adapter below is intentionally generic. Its key property is that public search has no unscoped form.

```python
from dataclasses import dataclass
from typing import Any, Protocol, Sequence


class VectorIndex(Protocol):
    def query(
        self,
        *,
        vector: Sequence[float],
        limit: int,
        filters: dict[str, Any],
    ) -> list[dict[str, Any]]: ...


@dataclass(frozen=True)
class TenantContext:
    tenant_id: str


class HelpCenterSearch:
    def __init__(self, index: VectorIndex) -> None:
        self._index = index

    def search(
        self,
        context: TenantContext,
        vector: Sequence[float],
        limit: int = 8,
    ) -> list[dict[str, Any]]:
        if not context.tenant_id:
            raise ValueError("tenant_id is required")
        if not 1 <= limit <= 50:
            raise ValueError("limit must be between 1 and 50")

        rows = self._index.query(
            vector=vector,
            limit=limit,
            filters={"tenant_id": {"eq": context.tenant_id}},
        )
        if any(row.get("tenant_id") != context.tenant_id for row in rows):
            raise RuntimeError("retrieval crossed the tenant boundary")
        return rows
```

That defensive result check is not the primary control; it is a tripwire. The underlying index must evaluate the filter during candidate selection, and integration tests should seed near-identical chunks for two tenants, query using each identity, and assert that results, traces, caches, and reranker inputs contain only the allowed tenant. Also test an absent tenant, a malformed identity, concurrent deletion, and a rebuild in progress. These are failure modes, not edge decoration.

No topology makes this optional.

Cache keys need the tenant identifier and every retrieval-affecting policy version. Metrics should expose query count, filtered candidate count where available, p95 latency, indexed bytes, ingestion backlog, deletion lag, and rebuild age by pool or dedicated tenant. Do not put raw help-center text into routine telemetry; identifiers and bounded counters are enough to diagnose most capacity failures.

## Change the layout without rewriting ingestion

The ingestion record should carry stable document and chunk identifiers, tenant ownership, source revision, content hash, and policy timestamps before it reaches any index. Route that same envelope to a pooled or dedicated physical target through a small placement map. Then moving a tenant is a controlled copy-and-verify operation rather than a change to parsing, chunking, or embedding semantics.

During migration, build the destination from the durable source, compare counts and sampled retrieval invariants, switch reads through a versioned placement record, and keep writes ordered through one authoritative path. Dual writing can shorten a transition, but it creates partial-success states that require reconciliation: the source write can succeed while the pooled index succeeds and the dedicated index fails, or the placement record can switch while an ingestion worker still holds an older route. A replayable event log or source table is easier to reason about than assuming two index writes always succeed together. Record a migration epoch on every write, reject stale epochs at the destination, and reconcile document identifiers before retiring the old copy. The trade-off is temporary duplicated state and more write coordination in exchange for a bounded, observable cutover.

The retrieval-augmented generation pattern combines a generator with retrieved external knowledge; its value depends on retrieving the right evidence in the first place. Tenant placement should therefore remain below the retrieval contract. Evaluation stays stable: for each tenant, measure whether expected supporting chunks appear, whether forbidden chunks never appear, and whether source revisions propagate within the stated freshness objective.

## What should you deliberately stop keeping?

Retain the authoritative source records, normalized ingestion envelopes, active index generation, and enough prior state to meet a written rollback objective. Do not retain every historical embedding generation by default. Embeddings are derived data; if the source, chunking configuration, embedding model identifier, and transformation code are reproducible, old generations can expire after the rollback window.

This choice has a cost during failure. If both retained generations are unusable, recovery requires reprocessing source material, consuming embedding capacity, and rebuilding index structures before retrieval is complete. A per-tenant layout may constrain that rebuild to one tenant, while a pooled layout needs placement gates or partial publication so unaffected tenants are not exposed to an incomplete generation. Measure full rebuild time with production-shaped data before setting retention, because source durability without a tested reconstruction path is only half a recovery plan.

My decision rule is deliberately dull: begin pooled, make tenant scope impossible to omit, record the variables that dominate searchable bytes, and isolate only where measurements or policy create a separate operational boundary. The bill stays legible, and the system retains an escape hatch when one tenant stops behaving like the long tail.

## Further reading

- Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks: https://arxiv.org/abs/2005.11401
