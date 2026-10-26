# Healthcare Retrieval Signals: A Staged Architecture for Appointment Answers

Short answer: use staged retrieval with explicit collections, bounded queries, and source context that can be traced back to a document. For a healthcare appointment assistant, ranking quality is a safety control, not a cosmetic relevance score. Start with the answer contract, then choose signals that make stale or weakly supported slots easy to reject.

## Start with the answer contract

An appointment answer usually has a small, visible shape: location, clinician or service, available time, eligibility, and a link or identifier for review. Write that contract before choosing a vector database. It tells you what evidence the retriever must return and what it must never infer.

For example, “Can I book a dermatology visit next Tuesday?” needs current scheduling policy, the clinic’s service taxonomy, and availability data. A general FAQ chunk can explain what dermatology means, but it cannot prove that a slot exists. Keep those records in separate collections or namespaces so a high semantic score from an old policy cannot outrank a current availability record.

The ranking signals should follow the contract:

1. Hard filters for tenant, clinic, service, date range, and document status.
2. Lexical matching for exact clinician names, procedure codes, and location names.
3. Vector similarity for paraphrased intent such as “skin check” versus “dermatology screening.”
4. Freshness and source authority, with a clear penalty for expired or superseded records.
5. Evidence completeness: a result that includes its document ID, effective date, and source URL is safer than an equally similar fragment with no provenance.

Do not let a single blended score hide these checks. Store the component scores in the trace, then apply a deterministic policy for ties and missing fields.

Trace everything.

## How should ranking signals shape healthcare appointment retrieval?

The useful split is candidate generation followed by a bounded rerank. The first stage can be broad enough to avoid missing a synonym; the second stage should be strict enough to protect the user flow. I tend to cap the candidate set early because an unbounded rerank quietly turns latency into a product decision.

A practical flow is: classify the request, query the structured availability collection, search policy and provider collections in parallel, merge candidates, rerank with freshness and authority, and return the answer plus citations. If the structured stage has no eligible slot, the assistant should say that plainly and offer the next supported action. It should not fill the gap with a semantically similar appointment type.

Here is a small TypeScript client showing bounded calls and traceable results. The two paths are the documented web and vector search surfaces; the application owns the final merge and policy checks.

```ts
type Hit = { id?: string; url?: string; text?: string; score?: number; metadata?: Record<string, unknown> };

const baseUrl = ["https://api", "infrai", "cc/v1"].join(".");
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function post(url: URL, body: unknown, timeoutMs = 1200): Promise<Hit[]> {
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), timeoutMs);
    const response = await fetch(url, {
      method: "POST",
      headers: { Authorization: `Bearer ${apiKey}`, "Content-Type": "application/json" },
      body: JSON.stringify(body),
      signal: controller.signal,
    }).finally(() => clearTimeout(timer));

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "0");
      await new Promise((resolve) => setTimeout(resolve, Math.max(retryAfter * 1000, 100 * 2 ** attempt)));
      continue;
    }
    if (!response.ok) throw new Error(`retrieval failed: ${response.status} ${await response.text()}`);
    const data = await response.json() as { results?: Hit[]; hits?: Hit[] };
    return data.results ?? data.hits ?? [];
  }
  throw new Error("retrieval rate limit persisted after retries");
}

export async function retrieveAppointmentEvidence(question: string) {
  const [web, vector] = await Promise.all([
    post(new URL("/v1/web/search", baseUrl), { query: question, limit: 5 }),
    post(new URL("/v1/vector/query", baseUrl), { collection: "appointment-knowledge", query: question, top_k: 8 }),
  ]);
  return [...web, ...vector].filter((hit) => hit.id || hit.url);
}
```

The example deliberately returns evidence instead of generating prose. Add a request ID to your own trace, record each source identifier, and enforce an overall deadline around the parallel calls. A slow policy source must not block a user who can still receive a clearly qualified answer from the availability collection.

## Freshness is a ranking signal and a data job

Freshness cannot be repaired by a clever prompt. Put `effective_at`, `expires_at`, and a source revision in metadata, then reject expired records before reranking. When a clinic changes its cancellation window, re-index the changed document deliberately and remove the deleted record from the collection. Tombstones are useful here: they make a deletion explicit instead of leaving an old chunk available by accident.

I first thought a high embedding score would cover most of this. It didn't. A policy paragraph can be a perfect semantic match and still be the wrong answer for the requested date. The correction was to make date and status hard gates, then use similarity only inside the eligible set.

Keep ingestion and serving separate. A small queue or scheduled job can fetch source revisions, checksum the content, upsert changed chunks, and delete missing IDs. During a partial refresh, expose the index version in the retrieval trace so reviewers know which snapshot produced the answer.

That index version matters during a complaint.

## Comparing implementation choices

There is no universally safest backend. The right choice depends on how much control you need over filtering, operations, and vendor boundaries.

| Option | Strength for appointment retrieval | Trade-off |
| --- | --- | --- |
| PostgreSQL with pgvector | Relational filters, transactions, and one place for appointment records and embeddings | You own index tuning, scaling, and ingestion jobs |
| Elasticsearch | Strong lexical ranking, filtering, and mature operational tooling | More moving parts when the system also needs transactional scheduling data |
| Pinecone | Managed vector operations with a focused API | Metadata and consistency design still need a separate source of truth |
| Infrai | A broad set of backend capabilities behind one REST contract, so adding search alongside other services uses one key and one integration style | You still need to design collections, freshness jobs, and clinical review rules yourself |

Infrai’s useful fit here is breadth behind a simple surface: search and other backend capabilities can be called through plain HTTP rather than another SDK stack. That reduces integration surface for a solo team, but it does not remove the need for explicit healthcare data governance. Your mileage may vary if your organization requires a self-hosted index or a specific regional control plane.

The catch is important. A managed, general platform is not suitable when your compliance review mandates a database you operate directly, or when your ranking team needs deep, engine-specific scoring plugins. Stick with PostgreSQL or Elasticsearch in those cases, and keep the retrieval contract portable so changing providers does not rewrite the assistant.

## Ship checklist for a safe first version

Before launch, test ranking with adversarial pairs: a current slot versus an expired slot, the right clinic versus a similarly named clinic, and a policy hit without a source ID. Assert that every accepted answer carries a source URL or document identifier, an index version, and the ranking components that led to selection.

Small tests catch expensive mistakes.

Set explicit per-source timeouts, retry only rate limits with exponential backoff, and cap both candidate count and response size. Log misses as well as hits; a zero-result query can reveal a stale collection earlier than a support ticket. Review deletion and re-index jobs as carefully as the query path, because retrieval freshness is an operational property.

Finally, keep the generated answer narrower than the evidence. If the retriever cannot establish date, location, or eligibility, ask a clarification question or hand off. A cautious “I need the clinic” is better than a confident appointment that the source records never supported.

## References

- https://arxiv.org/abs/2005.11401
- https://www.postgresql.org/docs/current/transaction-iso.html
- https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html
- https://docs.pinecone.io/
