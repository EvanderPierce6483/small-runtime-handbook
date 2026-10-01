# Grep vs Embeddings for 6-Source Listing Code Search (Choose Hybrid)

Freshness favors grep; intent matching favors embeddings. **TL;DR: choose a hybrid pipeline for an internal tool that searches code snippets behind a six-source B2B SaaS listing aggregator, but keep grep as the authority for exact names, identifiers, and newly merged code.** Generate semantic candidates only from versioned chunks, merge both result sets, then show commit and path beside every answer. If the corpus is small and queries usually contain symbols, grep alone is the better choice.

| Option | Pick it when | Chunking burden | Freshness behavior | Main failure mode |
|---|---|---|---|---|
| Grep only | Engineers know an identifier, field, route, or error text | None | Reads the current checkout or index | Misses conceptually related code with different words |
| Embeddings only | Most queries describe behavior rather than syntax | High | Depends on re-embedding changed chunks | Can return a plausible but stale or structurally incomplete snippet |
| Hybrid | Queries mix exact source names with concepts such as deduplication or normalization | Medium | Exact search covers the newest text while semantic search catches paraphrases | More ranking and observability work |

This is a retrieval decision, not a model contest. The hard parts are where a snippet begins, how quickly an edit becomes searchable, and whether a result can be traced to the same revision that the developer is reading.

## When is grep enough?

Choose grep when the internal tool answers questions like `normalizePartnerListing`, `external_listing_id`, or the literal text of a parsing error. Exact search preserves syntax, needs no semantic chunk boundaries, and makes freshness easy to reason about: search the target revision and the result belongs to that revision.

It is also the clean baseline. Before adding embeddings, collect a small query set from the work itself. Separate lookup queries from intent queries. If nearly every successful query contains a symbol, filename fragment, source label, or copied log line, semantic retrieval adds machinery without fixing the dominant job.

The weakness appears when vocabulary drifts. One connector may call a record a `listing`, another an `offer`, and a third an `inventoryItem`. A developer asking “where do we reject expired inventory?” may not know the predicate name or the status value used by that source. Imagine that three of the six connectors filter expiration during parsing, two do it during normalization, and one delegates it to a shared validator. An exact query for `expired` might find four useful locations and miss the connector whose code compares `validUntil` with the ingestion clock. The miss is not an outage or a search defect. It is the predictable limit of literal matching: grep cannot infer that the query and the comparison describe related behavior.

Keep this option boring. That is a feature.

## Pick embeddings when intent outruns vocabulary

Embeddings become useful when developers search in descriptions: “combine duplicate listings across partners,” “map remote work eligibility,” or “discard records with missing tenant ownership.” Those questions may have no literal overlap with the implementation. Semantic candidates can bridge that gap.

But embeddings search chunks, not repositories. Chunking determines what the retriever is even allowed to find. Fixed-width slices can split a function signature from its validation branch or attach a comment to the next declaration. Whole files avoid those cuts but often mix unrelated responsibilities. For this tool, declaration-aware chunks are the practical unit: include the signature, its body, the nearest useful comment, the file path, and the commit identifier. Cap oversized declarations rather than quietly swallowing an entire generated file.

Freshness is the sharper edge. A vector produced from commit A is evidence about commit A. After commit B changes the parser, a matching old chunk should not masquerade as current code. Store content hashes and revision metadata with every vector; invalidate changed chunks; exclude vectors whose revision is outside the requested snapshot. Exact search can still cover the narrow interval before updated embeddings are ready.

That last rule matters. Fast retrieval of yesterday's code is still wrong.

## Build the hybrid around revisions, not scores

The merge should not pretend that a grep rank and a cosine similarity are the same measurement. They are different signals. Use explicit buckets first, then stable tie-breakers inside each bucket: exact symbol matches, exact text matches, semantic matches from the requested revision, and finally any permitted older material clearly labeled as such. For a code assistant, the last bucket is usually better omitted.

Here is a compact TypeScript shape. It assumes that parsing and embedding happen elsewhere, so the retrieval boundary stays testable. The constants are example policy choices, not universal quality thresholds.

```ts
type SearchQuery = {
  text: string;
  revision: string;
  source?: string;
};

type Hit = {
  id: string;
  path: string;
  revision: string;
  contentHash: string;
  snippet: string;
  exactRank?: number;
  semanticScore?: number;
};

interface ExactIndex {
  search(query: SearchQuery, limit: number): Promise<Hit[]>;
}

interface SemanticIndex {
  search(query: SearchQuery, limit: number): Promise<Hit[]>;
}

const EXACT_LIMIT = 12;
const SEMANTIC_LIMIT = 18;
const RESULT_LIMIT = 10;

export async function retrieveCurrentSnippets(
  query: SearchQuery,
  exact: ExactIndex,
  semantic: SemanticIndex,
): Promise<Hit[]> {
  const [exactHits, semanticHits] = await Promise.all([
    exact.search(query, EXACT_LIMIT),
    semantic.search(query, SEMANTIC_LIMIT),
  ]);

  const currentSemanticHits = semanticHits.filter(
    (hit) => hit.revision === query.revision,
  );

  const merged = new Map<string, Hit>();
  for (const hit of [...exactHits, ...currentSemanticHits]) {
    const key = `${hit.path}:${hit.contentHash}`;
    const prior = merged.get(key);
    merged.set(key, prior ? { ...prior, ...hit } : hit);
  }

  return [...merged.values()]
    .sort((a, b) => {
      const aExact = a.exactRank === undefined ? 1 : 0;
      const bExact = b.exactRank === undefined ? 1 : 0;
      if (aExact !== bExact) return aExact - bExact;
      if (a.exactRank !== b.exactRank) {
        return (a.exactRank ?? Infinity) - (b.exactRank ?? Infinity);
      }
      return (b.semanticScore ?? -Infinity) - (a.semanticScore ?? -Infinity);
    })
    .slice(0, RESULT_LIMIT);
}
```

Read the flow as a diagram in words: query enters; exact and semantic branches run together; the revision gate closes on stale vectors; content hashes collapse duplicates; exact evidence gets the first lane; the tool returns ten traceable snippets. No generated answer should erase those paths and revisions. Retrieval-augmented generation was introduced as a way to combine generated output with retrieved external memory, but retrieval does not guarantee that the selected evidence is current or sufficient. The application still owns that contract.

Deployment needs two independent paths. The query path serves the current exact index. The indexing path parses changed files, computes content hashes, embeds only changed chunks, and marks a revision ready after all expected chunks arrive. Do not flip the semantic alias file by file. Flip it once the revision is complete, or keep semantic retrieval on the prior complete revision while exact search handles the new checkout.

Observability should follow those same stages. Record counts and latency separately for exact candidates, semantic candidates, revision rejections, deduplicated hits, and empty results. Alert on indexing lag in revisions or elapsed time, not only on request failures. A perfectly healthy query service can otherwise keep serving an aging semantic index.

Testing is equally concrete. Keep a versioned set of real query shapes with expected paths, including exact identifiers, paraphrased intent, renamed functions, deleted files, and two connectors that implement the same concept differently. Run it against a pinned repository revision. Add one freshness test that changes a function, removes its old chunk, and verifies that the old hash cannot return. This catches a more dangerous bug than a small ranking shuffle.

## Should an internal code snippet search use grep or embeddings?

Start with grep and instrument zero-result and reformulated queries. Add embeddings only after the missed queries show a recurring vocabulary gap. Once semantic search exists, **choose hybrid whenever freshness and paraphrase recall are both requirements**; do not make embeddings the sole path for exact code facts.

For the six-source listing aggregator, apply one operational rule: every visible snippet must carry `path`, `revision`, `contentHash`, and `source` metadata. The source field distinguishes connector-specific logic. The other three fields let the tool prove that the displayed code belongs to the requested snapshot.

Evaluate with task-shaped slices rather than one blended score. Look separately at symbol lookup, cross-source normalization, deduplication behavior, and freshly changed code. Also inspect failures. A top result from the wrong connector can look semantically excellent while sending an engineer into the wrong mapping rules.

The limitations are clear. Grep will not solve paraphrase. Embeddings will not prove freshness. Hybrid search will not repair poor chunks, missing revision metadata, or an evaluation set that ignores recent edits. It is also the wrong choice when every query is an identifier lookup, when the repository is too small to justify a second index, or when the team cannot operate revision-aware re-indexing. In those cases, pick grep. The trade-off is less conceptual recall in exchange for a smaller freshness surface and fewer moving parts.

For this mixed-query tool, **pick hybrid, with exact retrieval as the freshness backstop and revision-gated semantic chunks as the recall layer.**

## Sources and References

- https://arxiv.org/abs/2005.11401
