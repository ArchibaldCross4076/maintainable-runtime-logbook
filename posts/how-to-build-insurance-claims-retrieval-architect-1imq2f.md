# How to Build Insurance Claims Retrieval Architecture in Node.js: Traceable Access Rules

Short answer: use a staged retrieval design with explicit collections, bounded queries, and source context that survives all the way to the claim handler. In an insurance claims intake service, access rules are part of retrieval quality, not a post-processing detail.

I learned this the hard way in a prototype that put every document in one vector index. A query for “water damage deductible” returned a broker FAQ before the policy endorsement that actually governed the claimant. The top-k score looked fine; the citation was wrong. That is a grounding failure, and a confident answer makes it worse.

## How should retrieval architecture enforce insurance claims intake access rules?

Start with a retrieval contract for each user-visible answer. Define the retrieval unit (clause, endorsement, adjuster note), permitted collection, metadata filters, and freshness requirement before choosing an endpoint. For example, a claimant-facing deductible answer might allow only the `policy-endorsement` collection, filter by `policy_id`, and require content indexed after the latest policy change.

Keep collections explicit. Separate policy wording, filed endorsements, claim correspondence, and public guidance. The separation is an authorization boundary that a similarity score cannot provide. A collection name is not a permission system by itself, so enforce the same allow-list in the API layer and record it in the retrieval trace.

The web stage is for discovery, not authority. It can find a regulator page or a carrier notice, but the answer should label that material as external context and never let it override a policy document. A small, bounded candidate set is easier to inspect than an unbounded crawl.

Keep it boring.

## A minimal staged query in TypeScript

The example below searches public context first, then queries the already-authorized vector collection. It keeps the source identifiers and snippets needed for a citation. The API is plain HTTP, so a Node.js service does not need a vendor SDK; Infrai is one option when a single REST surface and key should cover both stages.

```ts
type Hit = { id: string; text: string; source_url?: string; score?: number };

const base = process.env.INFRAI_BASE_URL;
const key = process.env.INFRAI_API_KEY;
if (!base || !key) throw new Error("INFRAI_BASE_URL and INFRAI_API_KEY are required");

async function requestWeb(body: unknown): Promise<any> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(`${base}/web/search`, {
      method: "POST",
      headers: { Authorization: `Bearer ${key}`, "Content-Type": "application/json" },
      body: JSON.stringify(body),
    });
    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, Math.min(retryAfter * 1000, 8000)));
      continue;
    }
    if (!response.ok) throw new Error(`retrieval failed (${response.status}): ${await response.text()}`);
    return response.json();
  }
  throw new Error("rate limit persisted after retries");
}

async function requestVector(body: unknown): Promise<any> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(`${base}/vector/query`, {
      method: "POST",
      headers: { Authorization: `Bearer ${key}`, "Content-Type": "application/json" },
      body: JSON.stringify(body),
    });
    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, Math.min(retryAfter * 1000, 8000)));
      continue;
    }
    if (!response.ok) throw new Error(`retrieval failed (${response.status}): ${await response.text()}`);
    return response.json();
  }
  throw new Error("rate limit persisted after retries");
}

const question = "What deductible applies to this water-damage claim?";
const web = await requestWeb({ query: question, limit: 5 });
const policy = await requestVector({
  collection: "policy-endorsement",
  query: question,
  top_k: 8,
  filters: { policy_id: "POL-20481", status: "active" },
});

const sources: Hit[] = [...(policy.hits ?? []), ...(web.results ?? [])].map((hit: Hit) => ({
  id: hit.id,
  text: hit.text,
  source_url: hit.source_url,
  score: hit.score,
}));
console.log(JSON.stringify({ answer_context: sources, citation_required: true }));
```

The field names in a production adapter should be checked against the discovery schema for the deployed capability. The important behavior is stable: bound `top_k`, apply an allow-listed filter, and return source IDs with the text. Never generate a citation from the model's memory when the trace is empty.

For a concrete access rule, imagine a claim opened at 09:10 and a policy endorsement published at 09:30. The intake worker should mark the earlier answer as provisional, enqueue a re-index event for the endorsement, and invalidate the old document ID once the new hash is searchable. A later question must carry the same `policy_id` filter; relying on the claimant's free-text name is too loose because family policies can share surnames. For an adjuster role, add a second collection for internal notes, but keep that collection out of claimant requests even if its passages score higher. The trace should record the role, collection, filter, document version, and retrieval request ID. When a reviewer opens the claim six weeks later, those fields explain the answer without requiring a replay against a moving index. This is extra plumbing, yet it is cheaper than debating an untraceable denial.

That's the decision rule.

## What changes between common vector options?

There is no universally best collection service. The right choice depends on where access policy and freshness logic already live.

| Option | Useful fit for claims intake | Trade-off to test |
| --- | --- | --- |
| Pinecone | Managed vector collections with metadata filtering | External control plane and vendor-specific API semantics |
| Weaviate | Schema-rich objects and hybrid search | More schema and operational choices to own |
| Elasticsearch | Teams already using lexical search, ACLs, and audit tooling | Vector tuning shares resources with broader search workloads |
| Infrai vector API | Plain HTTP integration when one key should span web and vector calls | You still own collection allow-lists, re-index jobs, and evaluation |

The catch is operational ownership. A hosted index does not decide whether an adjuster may see a fraud-investigation note. Keep that decision in your application, pass only the permitted collection and filters, and log the request ID beside the claim event. Stick with Elasticsearch when your existing audit and field-level security are more valuable than a new vector control plane. Choose Pinecone or Weaviate when their managed indexing and retrieval ergonomics match your team.

## Freshness, deletion, and evaluation are one release gate

Re-index changed content deliberately. Treat a policy endorsement update as a new version, then remove the deleted record from the collection instead of leaving a stale embedding that can still win retrieval. Store `effective_at`, `supersedes`, and a content hash as metadata so a reviewer can explain why a passage was selected.

Before rollout, build a small labeled evaluation set: real-shaped questions, the allowed collection, expected source IDs, and an acceptable freshness window. Measure recall of the labeled source and citation precision, then inspect failures by rule (wrong collection, stale version, or weak wording). Ten carefully labeled cases beat a dashboard with an impressive but ungrounded average. Your mileage may vary; the set should grow whenever a new endorsement type or access role ships.

I would not copy the architecture until those tests pass for each role that handles claims. Latency and token cost matter, but a fast answer citing the wrong coverage is still a denied claim waiting to happen.

## References

- https://arxiv.org/abs/2005.11401
- https://docs.pinecone.io/guides/indexes/understanding-indexes
- https://weaviate.io/developers/weaviate/search/hybrid
- https://www.elastic.co/guide/en/elasticsearch/reference/current/knn-search.html
