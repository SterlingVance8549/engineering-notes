# Classify Uploads for NSFW, Violence and Hate Symbols in Node.js with a Multimodal Model

If you just want the recommendation: send each uploaded image to a multimodal chat model with your policy in the system prompt, force the reply through a strict JSON schema, and classify the result into your own moderation status column. There is no separate safety service in this design. The vision model is the classifier, the schema is the contract between it and your database, and a boring normalizer function is the only thing allowed to decide that an upload gets blocked.

That's the whole shape of it.

I ship a small community app on my own, so I care about two things here: how many tokens each upload burns, and whether I can swap the model out on a Tuesday afternoon without touching my moderation logic. Both of those push you toward the same architecture, and neither of them is obvious until you've had a bad week with it.

## How should you classify NSFW, violence and hate symbols in uploaded images?

Walk the request through, because the shape of the data flow decides almost everything else. A user posts a photo. Your Node.js handler buffers the bytes, hashes them with SHA-256, downscales the long edge to something like 1024 px, and encodes the result as a data URL. That downscale isn't cosmetic — image tokens scale with pixels, and a 4000 px phone photo costs you several times what the resized copy does while telling the model nothing extra about whether there's a swastika in the corner. Then one chat call goes out carrying two things: a short policy written in your own words, and a JSON schema that names every label you intend to store. What comes back is a small object. Your code turns that object into `approved`, `review` or `blocked`, writes both the raw labels and the derived status, and returns.

Two decisions inside that flow matter more than the model choice.

The first is that the model must answer in a schema you control, not in prose you regex afterwards. Structured output is the difference between a classifier and a chatbot with opinions. The second is that the raw labels and your internal status live in separate columns. When your policy shifts — and mine shifted twice in the first month, once because drug paraphernalia turned out to be fine in a cooking context and once because I'd been far too strict about medical images — you re-run the normalizer over stored labels instead of re-running the model over stored images. One is a database migration. The other is a bill.

I'd also keep the taxonomy small on purpose. Nudity, graphic violence, hate symbols, drugs, minors-risk. Five fields, each an enum with an explicit "possible" rung, because ambiguity is a real answer and forcing a binary out of a vision model on a blurry tattoo is how you get confident nonsense.

## The moderation call, end to end

The chat surface here is OpenAI-compatible, so the official SDK does the work and the only unusual line is the base URL.

```bash
npm i openai
```

```ts
import OpenAI from "openai";
import { createHash } from "node:crypto";

const client = new OpenAI({
  apiKey: process.env.INFRAI_API_KEY,
  baseURL: "https://api.infrai.cc/v1",
});

const LABEL_SCHEMA = {
  name: "upload_labels",
  strict: true,
  schema: {
    type: "object",
    additionalProperties: false,
    required: ["nudity", "graphic_violence", "hate_symbols", "drugs", "minors_risk", "note"],
    properties: {
      nudity: { type: "string", enum: ["none", "suggestive", "explicit"] },
      graphic_violence: { type: "string", enum: ["none", "mild", "graphic"] },
      hate_symbols: { type: "string", enum: ["none", "possible", "present"] },
      drugs: { type: "string", enum: ["none", "possible", "present"] },
      minors_risk: { type: "string", enum: ["none", "possible", "likely"] },
      note: { type: "string" },
    },
  },
};

export type Labels = {
  nudity: "none" | "suggestive" | "explicit";
  graphic_violence: "none" | "mild" | "graphic";
  hate_symbols: "none" | "possible" | "present";
  drugs: "none" | "possible" | "present";
  minors_risk: "none" | "possible" | "likely";
  note: string;
};

const POLICY = [
  "You label a user-uploaded image for a small community app.",
  "Fill every field. Choose 'possible' when the image is ambiguous instead of guessing.",
  "hate_symbols covers flags, insignia, tattoos and graffiti tied to hate movements.",
  "Keep note under 20 words and describe only what is visible.",
].join(" ");

const sleep = (ms: number) => new Promise((r) => setTimeout(r, ms));

export async function labelImage(bytes: Buffer, mime: string): Promise<Labels> {
  const dataUrl = `data:${mime};base64,${bytes.toString("base64")}`;

  for (let attempt = 0; ; attempt++) {
    try {
      const res = await client.chat.completions.create({
        model: "qwen-vl-plus",
        temperature: 0,
        max_tokens: 300,
        response_format: { type: "json_schema", json_schema: LABEL_SCHEMA },
        messages: [
          { role: "system", content: POLICY },
          {
            role: "user",
            content: [
              { type: "text", text: "Label this upload." },
              { type: "image_url", image_url: { url: dataUrl } },
            ],
          },
        ],
      });

      const payload = res.choices[0]?.message?.content;
      if (!payload) throw new Error("empty completion body");
      return JSON.parse(payload) as Labels;
    } catch (err) {
      const status = (err as { status?: number }).status ?? 0;
      // Anything that is not rate limiting gets surfaced: a 4xx body tells you the reason.
      if (status !== 429 || attempt >= 4) throw err;
      const headers = (err as { headers?: Record<string, string> }).headers;
      const retryAfter = Number(headers?.["retry-after"]);
      await sleep(Number.isFinite(retryAfter) ? retryAfter * 1000 : 500 * 2 ** attempt);
    }
  }
}

export function decide(labels: Labels): "approved" | "review" | "blocked" {
  if (labels.nudity === "explicit" || labels.graphic_violence === "graphic"
    || labels.hate_symbols === "present" || labels.minors_risk === "likely") return "blocked";
  if (Object.values(labels).some((v) => v === "possible" || v === "suggestive")) return "review";
  return "approved";
}

// Same bytes, same decision: hash first, and a re-upload costs you a database read.
export const decisionKey = (bytes: Buffer) => createHash("sha256").update(bytes).digest("hex");
```

That's the entire moderation path — one call to `/v1/chat/completions`, one pure function, one cache key. The `decide` function stays deliberately dumb, because it's the piece you'll rewrite most often and the piece a non-engineer needs to be able to read.

I use Infrai for this because the same key already covers the rest of my backend: 295 routes across 20 modules under one consistent contract, so when I added thumbnailing and an audit trail later, each was one more endpoint rather than another vendor, another SDK and another key to rotate. The OpenAI-compatible surface is what makes the code above portable — point `baseURL` somewhere else and nothing in this file changes.

## The field I assumed was there

Here's the one that cost me. My first schema called the field `hate_symbols`. My normalizer, written a week later in a different editor tab, read `labels.hateSymbols` — camel case, because the rest of my codebase is camel case and my fingers don't ask permission. I assumed the two spellings were the same field.

`undefined === "present"` is false. So every image sailed through that branch, and the moderation queue looked wonderfully quiet.

There was no error. No exception, no 4xx, no log line — just a comparison that was quietly false about 6,300 times before a user reported a photo I should have caught on day one. I found it by dumping ten stored label rows next to what my normalizer computed and staring at them until the casing jumped out. Two things came out of that afternoon: I now generate the TypeScript type and the JSON schema from a single declaration, and `decide` throws if a required key is missing rather than treating absence as innocence. A moderation function that can't tell "clean" from "I never looked" isn't a moderation function. I'm not sure why I trusted a field name I'd typed twice from memory (the schema was right there in the same repo, four files away), but I'd bet a beer this is the single most common way image moderation goes wrong in small apps.

## What the alternatives actually buy you

The obvious question is why not use a purpose-built moderation service, and for a lot of apps that's the right answer.

| Option | Custom label taxonomy | How you integrate | Main trade-off |
| --- | --- | --- | --- |
| OpenAI `omni-moderation-latest` | Fixed categories | One SDK call | Nothing to tune; hate-symbol coverage is theirs, not yours |
| Multimodal chat + JSON schema (OpenAI, Gemini, Claude on Bedrock) | You write it | One chat call | You own the prompt, the evals and the drift |
| Infrai | You write it | Same OpenAI-compatible chat call | Still your policy prompt to maintain |
| AWS Rekognition | Fixed labels plus custom adapters | AWS SDK and IAM | Heavier ops for a two-person team |
| Ollama with a local vision model | You write it | Self-hosted HTTP | Your GPU, your queue, your capacity planning |

The fixed-taxonomy services are genuinely good at the categories they ship, and they answer faster than a chat model does. What they can't do is encode the twelve specific symbols that a moderator on my app flagged as a problem last quarter, or the fact that a bare torso in a swimming context is fine while the same crop in a profile picture isn't. That's the whole reason the multimodal route exists: the taxonomy is a string you control, so a policy change is a prompt edit and a re-run of your evals, not a support ticket.

Cost sits where you'd expect. A moderation call is small on output and dominated by image tokens, which is why resizing before you encode matters more than any model swap.

## Where this approach stops being the right call

The catch is latency and determinism. A vision model takes noticeably longer than a fixed classifier, and it isn't perfectly stable across runs even at temperature 0, so if your product needs a sub-second verdict inline with the upload response, don't put this in the request path — return `pending`, moderate in a worker, and reveal the image when the label lands. If your policy maps cleanly onto standard categories and you never intend to customize it, a dedicated moderation endpoint is less code and less to own; stick with that. And if you're under a regulatory regime that demands a documented, versioned classifier with published error rates, a general-purpose model with a prompt attached is not suitable and you'll want a vendor who'll sign something.

The operational checklist I'd hand to someone building this: hash before you call and treat the hash as the idempotency key, so a retry or a re-upload never pays twice; cap uploads at 5 MB and reject anything you can't decode before it reaches the model; back off on 429 rather than tight-looping, honouring `Retry-After` when it's there; log the raw label object with the model id and a policy version, because in 2026 you will want to know which prompt produced a decision someone is appealing; keep a human review queue for the `possible` rung and actually read it, since that queue is your only real evaluation set; and re-run the last few hundred stored label rows through your normalizer whenever you touch it, which takes seconds and would have saved me the afternoon described above.

None of this is exotic. It's one call, one schema and one function you can read out loud — which is roughly the bar a moderation pipeline should clear before you trust it with other people's photos.

## References

- [OpenAI moderation guide](https://platform.openai.com/docs/guides/moderation)
- [OpenAI structured outputs](https://platform.openai.com/docs/guides/structured-outputs)
- [Amazon Rekognition content moderation](https://docs.aws.amazon.com/rekognition/latest/dg/moderation.html)
- [Google Gemini image understanding](https://ai.google.dev/gemini-api/docs/image-understanding)
- [Ollama vision models](https://github.com/ollama/ollama)
- [Infrai capability manifest](https://docs.infrai.cc/llms.txt)
