# Hybrid Search Accounting Explained — Node.js Traces for Tenant Moderation Docs

| Retrieval path | Pick this when | What to record per tenant |
| --- | --- | --- |
| Keyword search | Reports contain exact policy terms or rule identifiers | Search calls and returned candidates |
| Embedding search | Reports paraphrase the wording in policy documents | Query embedding usage and returned candidates |
| Keyword plus embeddings, then rerank | Exact terms and paraphrases both matter | Each branch, merged candidates, and reranking usage |

TL;DR: For a B2B SaaS moderation assistant, start with the smallest retrieval path that finds the right policy evidence. Combine keyword and embedding candidates when both kinds of matching are needed, then rerank a bounded union. Attach tenant accounting to every stage before letting a docs chatbot use the retrieved passages to suggest a report category for human review.

Make work visible.

The useful distinction is between finding evidence and spending work on it. A chatbot response might use six passages, yet the retrieval path could have processed many more. Recording only the final answer hides that difference. The design below makes those intermediate counts explicit without treating any illustrative limit as a measured optimum.

## When is keyword search enough?

Pick keyword search when your moderation reports and rubric documents share exact vocabulary. Consider an illustrative tenant, `tenant-cobalt`, whose report mentions policy rule `HAR-17`. An exact identifier gives the retrieval layer something concrete to match. The human reviewer still needs the associated policy passage, including any qualifications around that rule.

For an ask-your-docs interface, the query can be a report summary or a reviewer's question about the report. Keep retrieval focused on the tenant's moderation rubric. A report is input to classify; it is not permission to change which documents the assistant can access.

Start with this path if exact terminology covers the intended workload. It gives the team fewer stages to instrument, although language that differs from the document wording is a reason to test another path. Track the search duration, candidate count, and eventual reviewer correction separately. A fast search with irrelevant evidence is still a poor result.

## Pick embedding search for paraphrased reports

Pick this path when the report describes behavior in words that differ from the policy's language. An illustrative report saying “they keep sending threats after I block them” might need a passage labeled harassment or intimidation. An embedding adapter supplies a semantic candidate list; its presence alone does not establish that the passage supports the proposed category.

That boundary matters. Use labeled report-to-policy examples to decide whether this branch helps, and keep exact rule identifiers in the test set too. Record query embedding usage separately from document indexing usage. The former belongs to a request; the latter needs an explicit allocation policy if the team wants to attribute shared ingestion work to tenants.

These are accounting choices.

Document them before turning usage into a monetary estimate.

## How should hybrid search combine keyword matches and embeddings for moderation docs?

For mixed reports, run the keyword and embedding branches against the same authorized document scope. Merge their candidates, remove duplicate chunks, and use a reranker to order that bounded set for the report query. This note assumes both adapters return results in descending relevance order. The reranker is a generic interface, not a particular service.

Diagram in words: authenticated tenant → report query → two retrieval branches → unique policy chunks → reranker → cited evidence → suggested category → human reviewer. Alongside that path, stage events flow into an accounting ledger. The ledger answers which tenant requested the work and how much usage was reported; the trace answers where the request spent time.

The trade-off is concrete. Two branches add work, and reranking adds another operation. They deserve that complexity only if the resulting evidence improves the report-classification workflow. Use an explicit merge policy so that a busy keyword branch cannot silently consume every reranking slot. In the example below, round-robin selection reserves opportunities for both branches; it deliberately avoids combining scores whose scales have no defined relationship. Consider the specified limits: two branches can return 40 candidates before deduplication, but only 30 unique chunks can reach the reranker. When both branches return the same policy chunk, count it once in that union while preserving each branch's original workload event. Otherwise, duplicate removal could make the accounting appear to erase work that already happened. A small union is useful for bounding the next stage, but it doesn't describe the full retrieval effort.

Before: one response duration and a final passage count. After: branch durations, branch candidate counts, union size, and reranker usage attributed to the tenant. Now an unexpectedly large workload has a place to start investigating.

## A minimal implementation with stage accounting

Here is a TypeScript orchestration function for a Node.js application. The example limits each branch to 20 candidates, the union to 30, and the returned evidence to six passages. Those are illustrative workload controls. Tune them using your own report-to-policy test cases.

The interfaces are intentionally small. Implementations must enforce the tenant scope during retrieval, return their actual usage when available, and supply a stable key for each document-version-and-chunk combination. An empty usage list means accounting is unknown, not free. The output is evidence for the classification step; this function does not generate a category or authorize an action.

```ts
type Chunk = { key: string; tenantId: string; text: string };
type Usage = { unit: string; quantity: number };
type Batch = { chunks: Chunk[]; usage: Usage[] };
type Scope = { tenantId: string; requestId: string; query: string };
type Search = (scope: Scope, limit: number) => Promise<Batch>;
type Event = {
  tenantId: string;
  requestId: string;
  stage: string;
  status: "ok" | "error";
  durationMs: number;
  inputCount?: number;
  outputCount?: number;
  usage?: Usage[];
};
type Ports = {
  keyword: Search;
  semantic: Search;
  rerank: (scope: Scope, chunks: Chunk[]) => Promise<Batch>;
  record: (event: Event) => Promise<void>;
};

export async function retrieve(scope: Scope, ports: Ports) {
  async function stage(
    name: string,
    run: () => Promise<Batch>,
    inputCount?: number,
  ): Promise<Batch> {
    const started = performance.now();
    let batch: Batch;
    try {
      batch = await run();
      if (batch.chunks.some(c => c.tenantId !== scope.tenantId)) {
        throw new Error("Candidate scope mismatch");
      }
    } catch (error) {
      await ports.record({
        tenantId: scope.tenantId, requestId: scope.requestId,
        stage: name, status: "error", inputCount,
        durationMs: performance.now() - started,
      });
      throw error;
    }
    await ports.record({
      tenantId: scope.tenantId, requestId: scope.requestId,
      stage: name, status: "ok", inputCount,
      durationMs: performance.now() - started,
      outputCount: batch.chunks.length, usage: batch.usage,
    });
    return batch;
  }

  const [keyword, semantic] = await Promise.all([
    stage("keyword", () => ports.keyword(scope, 20)),
    stage("semantic", () => ports.semantic(scope, 20)),
  ]);
  const unique = new Map<string, Chunk>();
  for (let i = 0; i < 20 && unique.size < 30; i++) {
    for (const batch of [keyword, semantic]) {
      const chunk = batch.chunks[i];
      if (chunk && !unique.has(chunk.key) && unique.size < 30) {
        unique.set(chunk.key, chunk);
      }
    }
  }
  const candidates = [...unique.values()];
  if (candidates.length === 0) return [];
  const ranked = await stage(
    "rerank", () => ports.rerank(scope, candidates), candidates.length,
  );
  if (ranked.chunks.some(c => !unique.has(c.key))) {
    throw new Error("Reranker returned an unknown candidate");
  }
  return ranked.chunks.slice(0, 6);
}
```

Supply `tenantId` from authenticated application context. A matching field on a returned chunk is a defensive check; it cannot substitute for enforcing permissions inside the search adapters. If a tenant has finer document permissions, extend the scope and enforce them in both branches before candidates enter the union. The reranker should only reorder the supplied chunks and preserve their identities and content.

The accounting sink also needs a contract. Here, an awaited recording failure rejects the request, so a successful return requires successful stage recording. That makes missing accounting visible, but couples availability to the sink. A production deployment could instead durably enqueue events and process them asynchronously. Decide explicitly. Use stable event identifiers and idempotent ledger writes before adding retries; otherwise one retry can look like a second chargeable operation. The sample request identifier correlates events but is not, by itself, that idempotency scheme.

Don't sum unlike units.

Keep embedding tokens, reranking units, and search calls in separate ledger fields. For monetary attribution, reconcile them with the applicable billing records and a versioned allocation policy; candidate counts explain workload but do not establish a bill. An errored operation may still have consumed resources, so request failure must not automatically become zero usage. Adapters need a way to reconcile that usage even though this small function only records an error event.

For logs, prefer request identifiers, stage names, counts, and status over report text. Put tenant detail in access-controlled accounting records and diagnostic events; use aggregate stage metrics for the shared dashboard. This separation also gives retention policies something concrete to govern. GDPR Article 5 describes data minimization and storage limitation, while OWASP's LLM guidance covers sensitive information disclosure and prompt injection. Those concerns apply to the evidence and telemetry paths as well as the final chatbot response.

## Limits to settle before deployment

This implementation stops at retrieving evidence. Add adapter deadlines, cancellation, and an explicit fallback policy before deployment. Concurrent promises do not cancel the other branch when one fails. If you allow a keyword-only fallback, record that path so its latency and reviewer outcomes are not blended into hybrid results.

Test the accounting as well as relevance: one report with an exact rule identifier, one paraphrase, one empty result, duplicate chunks across branches, an out-of-scope candidate, and a failed usage write. These are proposed fixtures, not benchmark results. Compare the passages and suggested categories with human labels before increasing the reranking budget, and verify ledger reconciliation with the same care as response quality.

Retrieved policy text and submitted reports should remain data, not instructions that override the assistant's rules. Keep source references attached to the suggested category so a reviewer can inspect the actual passage. A retrieved match does not justify an automatic moderation decision.

The remaining limit is attribution. Request traces describe work along this path; shared indexing, storage, and operations require separate allocation rules. Per-tenant cost visibility is only as complete as that ledger's coverage.

## References

- OWASP Top 10 for Large Language Model Applications: https://owasp.org/www-project-top-10-for-large-language-model-applications/
- GDPR full text, including Article 5 on processing principles: https://gdpr-info.eu
