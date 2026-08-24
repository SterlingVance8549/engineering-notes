# Choosing a Cheap, Simple Support Ticket Classifier Without Fine-Tuning

Short answer: use zero-shot classification to prove the taxonomy, move repeatable cases to an embeddings classifier once reviewed examples exist, and add reranking only for label pairs that retrieval repeatedly confuses.

That is an operating sequence, not a permanent ranking. The deciding constraint is the evidence available today: label descriptions, corrected tickets, or a stable training corpus. A cheap model call attached to a vague taxonomy is still an expensive routing system once human corrections, retries, and missed queues enter the bill.

Start reversible.

## How should a cheap support ticket classifier combine embeddings, zero-shot LLMs, and reranking?

A zero-shot LLM is the shortest path from written label definitions to a testable classifier. Give it the ticket, an allowlisted set of label IDs, and precise inclusion and exclusion rules. Then validate its response in application code. This is a useful starting point when there are no reviewed examples, because the team can learn whether humans even agree on the taxonomy before maintaining a training pipeline. The catch is that each classification has to carry enough taxonomy context for the model to distinguish the labels, so runtime work grows with the number and length of those definitions. It is also a poor fit for a strict latency budget or for a workflow that cannot tolerate nondeterministic free-form output.

An embeddings classifier needs a different kind of evidence: reviewed historical tickets that represent current policy. It embeds a new ticket, retrieves similar reviewed examples, and aggregates their labels. This keeps taxonomy changes in data rather than in a newly trained model artifact. It also exposes the evidence behind a prediction, which is handy during review. But similarity does not repair a bad labeling policy. If agents use `login_problem` and `account_access` interchangeably, the nearest neighbors preserve that ambiguity. A new label with no examples has the opposite problem: there is nothing useful to retrieve.

Reranking belongs between candidate generation and the final decision. An embeddings search can cheaply narrow a large label set or example set; a reranker then evaluates the ticket against only those candidates. That second stage is worth testing when the confusion matrix shows that the correct label usually appears in the shortlist but rarely appears first. It is not suitable when retrieval already orders candidates correctly, when the shortlist often omits the right answer, or when another inference step breaks the tail-latency budget. Reranking cannot recover a candidate it never receives.

Fine-tuning becomes reasonable when the label policy is stable, representative reviewed data exists, and repeated evaluation shows a material distinction that the simpler cascade cannot make reliably. It brings a dataset lifecycle, training job, versioned artifact, and rollback decision. Don't pay that operational cost merely because fine-tuning sounds more committed.

| Method | Minimum useful evidence | Main strength | Boundary that matters |
| --- | --- | --- | --- |
| Zero-shot LLM | Clear label descriptions | Works before reviewed examples accumulate | Taxonomy context and validation happen on every request |
| Embeddings classifier | Representative reviewed tickets | Corrections become retrieval evidence | Similarity repeats inconsistent labels and struggles with new classes |
| Retrieve, then rerank | Reviewed candidates and discriminating descriptions | Reorders a plausible shortlist | Adds latency and cannot rescue missing candidates |
| Fine-tuned classifier | Stable policy and a representative labeled corpus | Encodes a mature classification task | Requires training, evaluation, deployment, and rollback discipline |

None is the best alternative in isolation. For a small team, the useful default is the least complicated method that meets a measured error budget while retaining an explicit path to human triage.

## The evaluation set matters more than the model label

Randomly splitting support tickets can make an experiment look better than production. Near-duplicate replies, recurring incidents, and copied ticket text may land on both sides of the split. A time-based split asks the more honest question: could yesterday's reviewed evidence classify tomorrow's arrivals? Keep entire conversations together, freeze the taxonomy version used by the test set, and exclude machine-generated agent replies if production classification happens before those replies exist.

Accuracy alone is too blunt. Record per-label precision and recall, a confusion matrix, coverage, abstention rate, and correction rate. Coverage is the share of tickets accepted automatically at a chosen policy threshold; selective accuracy is the accuracy on that accepted share. Read them together. A system can appear wonderfully accurate by sending nearly everything to review, or impress with high coverage by forcing guesses on cases that should have been rejected.

The cost denominator should be an accepted, correct tag. Count inference calls, embedding work, vector queries, storage, retries, and review time. I don't treat a lower per-call price as a result if the method sends twice as many tickets through manual triage. Tail latency deserves the same treatment: measure queue wait, embedding, retrieval, reranking, and fallback independently instead of reporting one warm-loop median.

Use a small shadow run before changing thresholds. Log the old and proposed decisions without letting the proposed path route tickets, then review disagreements by label. Thresholds are policy, not universal constants — a value that works for a clean billing taxonomy says nothing about a security queue. I'm not sure how much confidence calibration will transfer between two taxonomies without seeing their label prevalence and annotation agreement; a held-out, time-based set resolves that uncertainty.

There is a deeper failure mode here. The incoming ticket is untrusted text. It may contain pasted email, logs, credentials, or sentences that look like instructions to a model. OWASP's guidance for LLM applications identifies prompt injection and sensitive-information disclosure as distinct risks. Classification code should therefore treat the ticket as data, constrain output to known label IDs, reject unexpected structure, and avoid copying raw sensitive bodies into diagnostic logs. The model never gets authority to invent a queue or change the policy.

That last check is tiny. It matters.

## A focused TypeScript cascade keeps the decision reversible

The orchestration layer should own policy while adapters own model and storage calls. This keeps the zero-shot classifier, embedding index, and reranker replaceable, and it gives tests a stable surface. The following example is deliberately generic. Its candidate counts and thresholds are illustrative configuration values that must be tuned on a held-out set; they are not claimed benchmarks.

```ts
type Label = {
  id: string;
  description: string;
};

type Neighbor = {
  labelId: string;
  similarity: number;
};

type TagDecision = {
  labelId: string | null;
  route: "embedding" | "rerank" | "zero-shot" | "review";
  score: number | null;
  reason?: "UNRECOGNIZED_LABEL" | "LOW_CONFIDENCE";
};

interface ClassifierRuntime {
  nearest(ticket: string, limit: number): Promise<Neighbor[]>;
  rerank(
    ticket: string,
    candidates: Label[],
  ): Promise<Array<{ labelId: string; score: number }>>;
  classifyZeroShot(ticket: string, labels: Label[]): Promise<string>;
}

type Policy = {
  neighborLimit: number;
  candidateLimit: number;
  embeddingThreshold: number;
  rerankThreshold: number;
};

function weightedVotes(neighbors: Neighbor[]): Array<[string, number]> {
  const votes = new Map<string, number>();

  for (const neighbor of neighbors) {
    votes.set(
      neighbor.labelId,
      (votes.get(neighbor.labelId) ?? 0) + neighbor.similarity,
    );
  }

  return [...votes.entries()].sort((left, right) => right[1] - left[1]);
}

export async function classifyTicket(
  runtime: ClassifierRuntime,
  ticket: string,
  labels: Label[],
  policy: Policy,
): Promise<TagDecision> {
  const labelsById = new Map(labels.map((label) => [label.id, label]));
  const neighbors = await runtime.nearest(ticket, policy.neighborLimit);
  const votes = weightedVotes(neighbors).filter(([id]) => labelsById.has(id));
  const voteTotal = votes.reduce((sum, [, score]) => sum + score, 0);
  const topVoteShare = voteTotal > 0 ? (votes[0]?.[1] ?? 0) / voteTotal : 0;

  if (votes[0] && topVoteShare >= policy.embeddingThreshold) {
    return { labelId: votes[0][0], route: "embedding", score: topVoteShare };
  }

  const candidates = votes
    .slice(0, policy.candidateLimit)
    .map(([id]) => labelsById.get(id))
    .filter((label): label is Label => label !== undefined);

  if (candidates.length > 1) {
    const [best] = await runtime.rerank(ticket, candidates);
    if (
      best &&
      labelsById.has(best.labelId) &&
      best.score >= policy.rerankThreshold
    ) {
      return { labelId: best.labelId, route: "rerank", score: best.score };
    }
  }

  const zeroShotLabel = await runtime.classifyZeroShot(ticket, labels);
  if (labelsById.has(zeroShotLabel)) {
    return { labelId: zeroShotLabel, route: "zero-shot", score: null };
  }

  return {
    labelId: null,
    route: "review",
    score: null,
    reason: "UNRECOGNIZED_LABEL",
  };
}
```

I've made the boundaries visible: only reviewed labels enter the vote, the reranker sees a bounded candidate list, and an unknown model response becomes `UNRECOGNIZED_LABEL` rather than a database value. Production adapters should also enforce structured output, cap input according to the selected runtime, apply a separate deadline to each stage, and return `LOW_CONFIDENCE` when no route clears its calibrated threshold. The example leaves that final threshold branch to the surrounding queue policy because some teams can accept a delayed human decision while others need an immediate default queue.

Vector storage does not dictate the classifier architecture. A vector-capable relational store can keep embeddings beside ticket IDs, label IDs, review state, and taxonomy version; pgvector documents both exact and approximate nearest-neighbor search for PostgreSQL. Start with the simplest search that meets the measured latency target. An approximate index is an engineering trade-off only after corpus size and query latency justify measuring recall against exact search.

## Ship the baseline, then earn each extra stage

Begin with a versioned label file and a reviewed evaluation set. Run zero-shot classification to expose vague descriptions, but route low-confidence or invalid output to a person. Once corrections cover the active labels, test embeddings against the same frozen set. Add reranking only if candidate recall is good and candidate order is the demonstrated failure. This sequence makes every new model call answer a measured problem.

Keep the zero-shot path when examples remain scarce or the taxonomy changes weekly. Stick with embeddings when reviewed examples are plentiful, labels are separable, and corrections need to take effect without a training release. Use reranking when closely related candidates survive retrieval and the latency budget permits another stage. Return to fine-tuning when policy and data have matured enough that a trained artifact's improvement repays its lifecycle.

Before copying this design, measure annotation agreement, per-label error, candidate recall, accepted coverage, review time, p95 and p99 latency, and cost per accepted correct tag. Then choose the smallest cascade that clears the actual routing requirement.

No victory lap. Just fewer silent mistakes.

## References

- OWASP Top 10 for Large Language Model Applications: https://owasp.org/www-project-top-10-for-large-language-model-applications/
- pgvector, PostgreSQL vector similarity search extension: https://github.com/pgvector/pgvector
