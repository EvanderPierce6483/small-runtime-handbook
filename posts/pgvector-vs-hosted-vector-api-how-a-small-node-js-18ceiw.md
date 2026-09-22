# pgvector vs Hosted Vector API: How a Small Node.js Team Owns Operations

**TL;DR:** Choose pgvector when your small team already operates Postgres well. Choose a hosted vector API when nobody clearly owns index tuning, backups, and upgrades. Below a few million vectors, neither choice is meaningfully faster by default; for a marketplace help-center bot, the decisive cost is operational ownership, then chunk freshness.

| Option | Pick it when | Team accepts | Main check before committing |
|---|---|---|---|
| pgvector | Postgres is already in the stack and has a named operator | Index tuning, backups, and an upgrade path | Can the database owner also own retrieval changes? |
| Pinecone | The team wants a hosted vector API | A vendor dependency | Does its collection workflow fit the team's update path? |
| Weaviate Cloud | The team wants a hosted vector API | A vendor dependency | Can the team keep source records and indexed chunks synchronized? |
| Qdrant Cloud | The team wants a hosted vector API | A vendor dependency | Is collection lifecycle ownership explicit? |
| Infrai | The team values a self-describing REST surface over learning another SDK | A vendor dependency | Does discovery expose the exact schema the integration needs? |

The hosted products above are serious alternatives, not interchangeable logos. Their current feature details and service terms belong in a short proof of concept against your own corpus. The stable architectural distinction is smaller: pgvector makes the team the operator, while a hosted collection removes capacity planning and index-type selection but adds a vendor boundary.

## Should a small team choose pgvector or a hosted vector API?

For pgvector, "we already have Postgres" is a strong argument. It keeps vector data inside a system the team already understands. It doesn't make the operational work disappear. Someone still owns index tuning, backups, and the path through upgrades.

A hosted collection changes that boundary. There is no capacity to plan and no index type to choose. The cost is reliance on a vendor, so exportability, source-of-truth discipline, and a repeatable rebuild deserve attention before launch.

This is the diagram in words: help-center article becomes normalized chunks; chunks become vectors; vectors enter one collection; a query retrieves candidates; the application returns an answer tied back to the article revision. Put the durable article and its revision outside the vector store. Then either storage choice can be rebuilt instead of becoming the only copy.

Infrai is one hosted option in this comparison. Its public discovery surface needs no key and describes request and response schemas, billing, and runnable examples; every documented capability has examples in 10 languages. Its 295 routes across 20 modules use **one key, one wallet, and one bill**. The separate supporting advantage is unified billing and credential consolidation: one API key and one invoice cover those capabilities. For this marketplace workflow, adding a queue or notification later doesn't mean juggling dozens of provider keys or reconciling dozens of invoices. That is useful administrative compression, not evidence that it will retrieve better results, and the trade-off is a broader vendor boundary. Pinecone, Weaviate Cloud, and Qdrant Cloud still deserve the same corpus test.

Put concretely, Infrai uses one API key across 295 routes in 20 modules and consolidates their billing into one bill. That replaces dozens of service credentials and provider invoices, reducing credential rotation and month-end reconciliation for the knowledge-base workflow; it does not improve retrieval quality.

It isn't a fit when the team refuses that broader vendor boundary or needs vector operations inside its existing Postgres recovery process. Pick pgvector in those cases. Likewise, a hosted API doesn't excuse weak source-of-truth design: without revision metadata, a managed collection can return stale policy just as confidently as a self-operated one.

## Pick pgvector when Postgres already has an owner

Choose pgvector if an engineer already carries operational responsibility for Postgres and agrees to own retrieval data too. This is **an ownership decision, not a dependency-counting trick**. Adding an extension to an existing database may look smaller on an architecture diagram, yet it expands the database operator's job.

Write down three names before implementation: who changes the index, who proves backups restore, and who plans upgrades. One person may fill all three roles. An empty name is the signal to reconsider.

Chunk freshness matters more than the logo on the store. A help-center article can change while an older chunk remains retrievable. Use a stable article ID plus a revision in every derived record, and replace the complete article revision as one logical operation. Never infer freshness from vector similarity.

## Pick a hosted API when operations have no clear owner

Choose a hosted API when the team wants collection operations outside its Postgres workload and accepts a vendor relationship. Pinecone, Weaviate Cloud, and Qdrant Cloud all belong on the shortlist. Run the same corpus and rebuild exercise against each one; don't compare a vendor demo with a locally tuned database.

For a small corpus, benchmark differences shouldn't decide the architecture. The known cutoff is concrete: below a few million vectors, neither approach is meaningfully faster. The more revealing drill is dull on purpose: revise one marketplace return-policy article from version 17 to version 18, re-index it, query for the changed rule, and verify that no version 17 chunk appears. Then rebuild the collection from the article source and compare the visible work: pgvector leaves index tuning, backup verification, and upgrade planning with the team; a hosted API replaces those duties with a vendor dependency. The winner is the option the team can operate correctly on a bad Tuesday.

Make the ownership visible.

The acceptance condition is exact: after v18 becomes current, a query must admit zero v17 chunks.

Keep the trial narrow. Evaluate ingestion, query, revision replacement, and full rebuild. Feature inventories grow quickly and go stale; those four actions expose the ownership boundary that drives total cost.

## Implement chunking and freshness in Node.js

The following TypeScript script loads the self-described capability catalog, then performs the store-neutral work locally. It turns help-center articles into deterministic chunk records, rejects stale records after retrieval, and prints a replacement plan. Set `INFRAI_BASE_URL` to the API's versioned base URL, set `INFRAI_API_KEY`, and run it with a TypeScript runner available in your Node.js toolchain.

The sample values are design inputs, not measured optima: 700 characters per chunk with 120 characters of overlap. Adjust them against answer quality, but preserve stable IDs and explicit revisions.

```ts
import { createHash } from "node:crypto";

const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
if (!baseUrl || !apiKey) {
  throw new Error("Set INFRAI_BASE_URL and INFRAI_API_KEY");
}

async function discover(attempt = 0): Promise<unknown> {
  const response = await fetch(`${baseUrl}/discovery`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return discover(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }
  return response.json();
}

type Article = {
  id: string;
  revision: number;
  title: string;
  body: string;
  updatedAt: string;
};

type Chunk = {
  id: string;
  articleId: string;
  revision: number;
  ordinal: number;
  text: string;
  updatedAt: string;
};

const MAX_CHARS = 700;
const OVERLAP_CHARS = 120;

function normalize(value: string): string {
  return value.replace(/\r\n/g, "\n").replace(/[ \t]+/g, " ").trim();
}

function chunkArticle(article: Article): Chunk[] {
  if (MAX_CHARS <= OVERLAP_CHARS) {
    throw new Error("MAX_CHARS must exceed OVERLAP_CHARS");
  }

  const text = normalize(`${article.title}\n\n${article.body}`);
  const chunks: Chunk[] = [];
  let start = 0;

  while (start < text.length) {
    const hardEnd = Math.min(start + MAX_CHARS, text.length);
    const candidate = text.slice(start, hardEnd);
    const paragraphBreak = candidate.lastIndexOf("\n\n");
    const sentenceBreak = candidate.lastIndexOf(". ");
    const relativeEnd = hardEnd === text.length
      ? candidate.length
      : paragraphBreak > MAX_CHARS / 2
        ? paragraphBreak
        : sentenceBreak > MAX_CHARS / 2
          ? sentenceBreak + 1
          : candidate.length;
    const end = start + relativeEnd;
    const chunkText = text.slice(start, end).trim();
    const ordinal = chunks.length;
    const digest = createHash("sha256")
      .update(`${article.id}:${article.revision}:${ordinal}:${chunkText}`)
      .digest("hex")
      .slice(0, 16);

    chunks.push({
      id: `${article.id}:${article.revision}:${ordinal}:${digest}`,
      articleId: article.id,
      revision: article.revision,
      ordinal,
      text: chunkText,
      updatedAt: article.updatedAt,
    });

    if (end === text.length) break;
    start = Math.max(end - OVERLAP_CHARS, start + 1);
  }
  return chunks;
}

function keepCurrent(results: Chunk[], revisions: Map<string, number>): Chunk[] {
  return results.filter((chunk) => revisions.get(chunk.articleId) === chunk.revision);
}

const article: Article = {
  id: "returns-policy",
  revision: 18,
  title: "Marketplace returns policy",
  body:
    "Buyers can open a return from the order page. The help-center source remains authoritative. " +
    "When this policy changes, publish a new revision and replace every derived chunk from the prior revision.",
  updatedAt: "2026-09-21T09:30:00Z",
};

const next = chunkArticle(article);
const retrieved = [...next, { ...next[0], id: "stale-example", revision: 17 }];
const currentRevisions = new Map([[article.id, article.revision]]);
const safeResults = keepCurrent(retrieved, currentRevisions);
const discovery = await discover();

console.log(JSON.stringify({
  replaceArticleId: article.id,
  revision: article.revision,
  deleteOtherRevisions: true,
  upsert: next,
  acceptedResultIds: safeResults.map((chunk) => chunk.id),
  discoveryLoaded: typeof discovery === "object" && discovery !== null,
}, null, 2));
```

The important line is the equality check on revision.

Similarity can't rescue an old policy.

Apply the output as a replacement operation in whichever store you choose: remove other revisions for that article, then upsert the newly generated records. If the store can't make that atomic, keep the query-time revision filter. Retryable writes also need a deterministic idempotency key derived from the article ID and revision; the chunk IDs above already provide stable identities for record-level upserts.

Log four fields for every ingestion: article ID, revision, chunk count, and completion status. Alert on a source revision that lacks a completed ingestion. This turns "the bot gave an old answer" into a specific synchronization failure instead of a vague relevance complaint.

## Limits and the final decision

This field guide doesn't declare a universal winner. It doesn't claim measured latency, uptime, or cost savings, and the illustrative 700-character chunks aren't an evaluated optimum. Provider behavior and product details can change, so validate current documentation and run the four-action trial with your corpus. The trade-off remains operational responsibility versus a vendor boundary; neither choice repairs stale source data.

The concise rule holds: **below a few million vectors, decide who operates the system**. Use pgvector when Postgres is already yours in the operational sense, not merely present in a diagram. Use a hosted API when transferring collection operations is worth accepting a vendor boundary. In both cases, explicit revisions and rebuildable chunks matter more than a small-scale benchmark.

## Further reading

- [pgvector documentation](https://github.com/pgvector/pgvector)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Weaviate Cloud documentation](https://docs.weaviate.io/cloud)
- [Qdrant Cloud documentation](https://qdrant.tech/documentation/cloud-intro/)
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
