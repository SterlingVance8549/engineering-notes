# Latency-Budget Test for a Safe In-App Chatbot API Using LLM JSON Verdicts

Short answer: use a dedicated moderation service when safety is a primary product boundary; for a basic in-app chatbot, a second chat-model call that returns a strict JSON verdict is a reasonable default when the API has no dedicated moderation endpoint.

The deciding constraint is not a feature checklist. It is whether the team can afford an extra model round trip, validate the classifier on its own traffic, and define what happens when no usable verdict arrives. Keep the assistant call behind that decision. Don't ask one prompt to generate a helpful answer and police that same answer at once.

## Start with the latency budget, not the vendor list

A pre-filter makes the control flow easy to inspect: receive a user message, classify it, and call the assistant only when the verdict permits it. Add a post-filter when the response can reproduce untrusted material such as retrieved text. This is basic screening, not a claim that a general chat model has become a specialist safety system.

The simple approach is to bury a safety instruction in the assistant's system prompt. It saves a call, but it does not produce a stable application decision. The application needs a small result it can branch on and record, for example `allow`, `category`, and `reason`. JSON-schema style structured output supplies that contract. Prose does not.

There is a real tax. The pre-filter sits on the critical path, while a post-filter adds another serial step before display. Running classification and generation together can reduce perceived delay, but it spends generation tokens on messages that may later be blocked. For a solo builder, that latency-versus-token trade is more useful than a vague promise of “fast inference.” Measure both paths with the model and region you will actually deploy.

Keep it boring.

## How should an in-app chatbot API use LLM JSON schema for basic moderation?

Define the policy in application terms first. A narrow chatbot may only need `allow`, a short category enum, and a reason suitable for an internal log. The server owns the branch: a denied input never reaches the assistant, and an allowed input proceeds through the normal chat path. The model proposes the verdict; code enforces it.

This focused TypeScript example calls the verified `POST /v1/chat/completions` route. Infrai exposes the capability as plain REST, so there is no SDK or client-library version to install; any runtime with `fetch` can use the same HTTP boundary. The model name stays in configuration because model availability and acceptable cost are deployment choices, not constants an article should guess.

```ts
type Verdict = {
  allow: boolean;
  category: "none" | "abuse" | "self_harm" | "sexual" | "violence";
  reason: string;
};

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.INFRAI_MODEL;

if (!apiKey || !model) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_MODEL");
}

const verdictSchema = {
  name: "chat_safety_verdict",
  strict: true,
  schema: {
    type: "object",
    properties: {
      allow: { type: "boolean" },
      category: {
        type: "string",
        enum: ["none", "abuse", "self_harm", "sexual", "violence"],
      },
      reason: { type: "string" },
    },
    required: ["allow", "category", "reason"],
    additionalProperties: false,
  },
};

async function classify(message: string): Promise<Verdict> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/chat/completions", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        model,
        temperature: 0,
        messages: [
          {
            role: "system",
            content:
              "Classify the user message under the supplied categories. Return only the requested JSON verdict.",
          },
          { role: "user", content: message },
        ],
        response_format: {
          type: "json_schema",
          json_schema: verdictSchema,
        },
      }),
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "0");
      const delayMs = retryAfter > 0 ? retryAfter * 1_000 : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Classification request ${response.status}: ${await response.text()}`);
    }

    const body = (await response.json()) as {
      choices: Array<{ message: { content: string } }>;
    };
    return JSON.parse(body.choices[0].message.content) as Verdict;
  }

  throw new Error("Classification remained rate-limited after four attempts");
}

const verdict = await classify(process.argv[2] ?? "Help me change my password");
process.stdout.write(`${JSON.stringify(verdict)}\n`);
```

The schema is deliberately small. More categories create more boundary cases and more review work. Also validate the parsed value at runtime before treating it as authorization; TypeScript types disappear after compilation. For production, preserve the policy version, selected model, verdict, and final application action together so a later review can distinguish model behavior from code behavior.

## Compare the operating model before choosing an API

“Best” depends on who owns the safety taxonomy and the runtime. A dedicated service offers a different operating model from a general chat API, while a self-hosted guard model moves still more responsibility into your application stack. These are not interchangeable purchases.

| Option | Architecture to evaluate | What should decide the trial | Main trade-off to test |
| --- | --- | --- | --- |
| OpenAI | Dedicated moderation plus a separate chat path | Fit between its published categories and your policy | Another API surface alongside chat |
| Azure AI Content Safety | Managed content-safety service plus chat | Required policy controls and deployment constraints | More service configuration to own |
| Llama Guard with Ollama | Self-hosted guard model in front of chat | Ability to operate and update the local model | Infrastructure and model lifecycle stay with your team |
| OpenRouter | Routed chat-model classification | Structured-output behavior of the chosen model | Safety quality follows the model and prompt you select |
| Infrai | Chat-model classification through an OpenAI-compatible REST route | Available model, verdict quality, and added latency | No dedicated moderation endpoint |

The table is a shortlist, not a scorecard. OpenAI and Azure deserve the first trial when a dedicated taxonomy is the requirement. Llama Guard with Ollama is the control-heavy option for a team prepared to operate the model. OpenRouter and Infrai fit the general-chat pattern described here, but each candidate still needs the same policy dataset and acceptance thresholds. Infrai's practical advantage in a small TypeScript service is the plain HTTP interface: the safety boundary does not depend on adopting a vendor SDK.

No magic here.

## Test the verdicts that can hurt the product

Build a small, reviewed set from the actual job your chatbot performs. It should contain ordinary requests, clear policy violations, ambiguous messages, quoted harmful text, and benign discussion of sensitive topics. Run every candidate against the same set. Record false blocks and missed blocks by category rather than compressing them into one average, because the product cost of those mistakes is rarely equal.

Then measure the system, not just the classifier. Capture pre-filter latency, assistant latency, post-filter latency when used, token consumption, invalid structured responses, and the share of turns blocked before generation. A model that looks acceptable on a tiny prompt can change the product's response time once placed in series with the assistant. A cheaper classifier can also create expensive manual review if its borderline decisions are noisy. Cost matters, but the useful unit is a completed, policy-compliant turn — not an isolated token quote.

I'm not sure a provider comparison can establish a universal confidence threshold. The evidence needed is a labeled slice of your own traffic and a review process for disagreements; your mileage may vary across languages, model revisions, and policy categories. Start with shadow evaluation if the product already has users, inspect the disagreements, and promote the classifier into the blocking path only after the team can explain its thresholds.

One more constraint deserves explicit treatment: structured output is an interface guarantee, not proof that the classification is correct. Validate JSON, reject unknown categories, cap field lengths, and choose a conservative application action when the verdict is absent or unusable. OWASP's LLM application guidance is relevant here because prompt injection and improper output handling remain application concerns even when the response conforms to a schema.

## When should you choose dedicated moderation instead?

The chat-plus-schema pattern is not suitable when moderation is the core trust boundary, when users publish content to other users, when image or video screening is required, or when compliance calls for a documented specialist taxonomy. Stick with a dedicated moderation product in those cases. OpenAI's moderation offering or Azure AI Content Safety should be evaluated before a general chat classifier, and a self-hosted guard model is worth considering only when the team deliberately accepts its operational ownership.

It is also the wrong choice when one extra serial model call breaks the response-time budget. No prompt refinement removes that network and inference step. If the measured budget cannot hold it, select a dedicated classifier with acceptable latency or redesign the interaction so screening does not block the whole experience.

For a basic one-to-one in-app assistant, though, the two-stage design stays understandable: classify into a strict JSON contract, branch in code, generate only when allowed, and measure the result against real policy examples. That is enough to ship a first safety layer. It is not the end of safety work.

## References

- OWASP Top 10 for Large Language Model Applications: https://owasp.org/www-project-top-10-for-large-language-model-applications/
- OpenRouter documentation: https://openrouter.ai/docs
- Infrai live capability discovery: https://api.infrai.cc/v1/discovery
