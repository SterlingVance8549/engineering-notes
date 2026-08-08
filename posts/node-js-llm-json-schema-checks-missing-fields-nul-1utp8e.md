# Node.js LLM JSON Schema Checks: Missing Fields, Null Values, and Enum Mismatch

The operational constraint is simple: a parser can reject an object, but it cannot tell you whether the source was missing a fact or the model ignored your contract. Treat those as separate signals. In a small Node.js extraction service, I use a schema as the boundary, preserve the raw response for diagnosis, and make repair a bounded, side-effect-free step.

Short answer: define missingness and nullability explicitly in JSON Schema, validate every response before business logic, and use a focused repair prompt only for the validation errors you can name. Prompt wording helps, but it is not a substitute for a contract.

## The experiment note: change one boundary at a time

The tempting approach is to keep expanding the extraction prompt: repeat “return every field,” add a second example, then ask for valid enum values again. That may improve formatting, yet it leaves the failure unclassified. A better experiment freezes the input sample and model settings, then compares three boundaries: parse-only, parse-plus-validation, and parse-plus-validation-plus-repair. The useful output is not a single success percentage. It is a per-keyword error count and the number of model calls per accepted object. I also keep the schema hash beside each result, because a validation failure from yesterday's contract is a misleading comparison for today's prompt. When the corpus changes, I record that as a new run instead of smoothing the numbers together; otherwise a larger batch of easy messages can make a weaker extractor look better.

For a support-message extractor, the contract might contain `intent`, `urgency`, `customer_id`, and `refund_requested`. The source may omit `customer_id`; an ambiguous refund request may deserve `null`; `intent` may be close to an allowed enum member but not equal to one. Those are different product decisions, even if all three arrive as a failed validation result.

I keep the first implementation deliberately boring. A response is text until `JSON.parse` proves otherwise, and it is untrusted data until the validator accepts it. That gives the rest of the application one stable handoff.

The boundary is the feature.

## How should a Node.js extraction prompt handle missing fields, null values, and enum mismatch?

Start with the schema, then make the prompt mirror it. A required property answers “must the key exist?”; a union such as `["string", "null"]` answers “may its value be null?”; an `enum` answers “which exact values are legal?” Do not ask the model to infer those distinctions from prose.

```ts
import Ajv, { type JSONSchemaType } from "ajv";

type Ticket = {
  intent: "billing" | "technical" | "account" | "unknown";
  urgency: "low" | "normal" | "high";
  customer_id: string | null;
  refund_requested: boolean | null;
};

const ticketSchema: JSONSchemaType<Ticket> = {
  type: "object",
  properties: {
    intent: { type: "string", enum: ["billing", "technical", "account", "unknown"] },
    urgency: { type: "string", enum: ["low", "normal", "high"] },
    customer_id: { type: ["string", "null"] },
    refund_requested: { type: ["boolean", "null"] },
  },
  required: ["intent", "urgency", "customer_id", "refund_requested"],
  additionalProperties: false,
};

const ajv = new Ajv({ allErrors: true });
const validate = ajv.compile(ticketSchema);

export function validateTicket(value: unknown): Ticket {
  if (!validate(value)) {
    const details = (validate.errors ?? []).map((error) => ({
      keyword: error.keyword,
      path: error.instancePath || "/",
      params: error.params,
    }));
    throw new Error(`Schema validation failed (422): ${JSON.stringify(details)}`);
  }
  return value as Ticket;
}
```

The `unknown` enum member is not a trick for making the validator lenient. It is a declared business state. Use it only when downstream consumers can act on “unknown”; otherwise route that case to review. Likewise, making a field nullable is honest only if `null` carries meaning distinct from an empty string or an omitted key.

The prompt should state “emit the object only,” list the exact enum members, and say what to do when evidence is absent. Include one complete example, not a gallery of partial fragments. Keep the schema and prompt in the same versioned module so a field change cannot silently leave the instructions behind.

## Parse, validate, then repair without replaying work

Repair belongs between validation and business side effects. It should receive the original text, the rejected JSON, and a compact list of actionable errors. Cap attempts, preserve each payload, and return a typed failure when the cap is reached.

```ts
type ModelCall = (prompt: string) => Promise<string>;

function parseObject(raw: string): unknown {
  try {
    const value: unknown = JSON.parse(raw);
    return value && typeof value === "object" && !Array.isArray(value) ? value : undefined;
  } catch {
    return undefined;
  }
}

function repairPrompt(source: string, previous: string, errors: string[]): string {
  return [
    "Return one JSON object and no prose.",
    "Use only the schema's property names and enum values.",
    `Source text:\n${source}`,
    `Previous response:\n${previous}`,
    `Validation errors:\n${errors.join("\n")}`,
  ].join("\n\n");
}

export async function extractTicket(source: string, callModel: ModelCall): Promise<Ticket> {
  let raw = await callModel(`Extract a ticket as JSON.\n\nSource text:\n${source}`);

  for (let attempt = 1; attempt <= 2; attempt += 1) {
    const parsed = parseObject(raw);
    if (parsed !== undefined && validate(parsed)) return parsed as Ticket;

    const errors = parsed === undefined
      ? ["response is not a JSON object"]
      : (validate.errors ?? []).map((error) =>
          `${error.instancePath || "/"}: ${error.keyword} ${JSON.stringify(error.params)}`,
        );
    if (attempt === 2) throw new Error(`Extraction rejected after ${attempt} attempts`);
    raw = await callModel(repairPrompt(source, raw, errors));
  }

  throw new Error("unreachable");
}
```

The retry wraps only the model call. If the accepted object triggers a database write, that write happens after extraction returns and uses a document identifier as its idempotency key. An HTTP 422-style validation record is observable; a duplicated refund or email is much harder to unwind. I log attempt count, schema keyword, input identifier, and token usage, while redacting source text that contains personal data.

One short warning: never “repair” by silently deleting an invalid field. That converts a visible contract failure into an incomplete record and makes later audits guess what happened.

## Choosing the right fix for each failure class

Missing keys usually point to one of three causes: the requirement is not actually present in the source, the prompt does not define an absence value, or the validator is checking a different schema version. Compare the raw text, prompt version, and schema hash before changing the model call.

Null values need a policy. If null means “the extractor found no evidence,” keep it and document that meaning. If the business process requires a decision, replace null with an explicit enum state and send that state to a human or a later enrichment step. Do not coerce null to a plausible value just to make validation green.

Enum mismatches are often normalization problems: case, punctuation, or a synonym differs from the canonical value. A deterministic mapping table can handle known aliases, but it must run after parsing and before validation only when the mapping is lossless and audited. Otherwise, send the allowed values back in a repair request. Never accept an arbitrary new enum member because it “looks close.”

Malformed JSON is its own category. A code fence, trailing commentary, or an unescaped newline should be counted separately from schema drift. Tight output instructions can reduce it; a parser exception should still stop the record from reaching application code.

## Measure the contract before you tune the prompt

Use a fixed, representative corpus and version every change. Track first-pass acceptance, repair acceptance, failures by JSON Schema keyword, parse failures, average attempts, and tokens per accepted record. Segment those metrics by document shape; a single blended rate hides the cases that matter.

The catch is operational fit. A repair loop is unsuitable when latency must stay within one round trip, when model calls cannot receive the source safely, or when a human review queue is cheaper than repeated inference. In those situations, use provider-supported constrained decoding if it covers your schema, or choose a deterministic parser plus review for the ambiguous fields. Stick with a prompt-only approach when the output is advisory and occasional manual correction is acceptable; do not pretend it provides a hard contract.

I'm not sure one threshold generalizes across teams. Your mileage will vary with document length, language mix, and how expensive a wrong enum is. That uncertainty is a reason to publish the eval set and schema version, not a reason to hide the error counts.

## Sources

- JSON Schema Validation, draft 2020-12 — https://json-schema.org/draft/2020-12/json-schema-validation.html
- Ajv API: validation errors — https://ajv.js.org/api.html#validation-errors
- RFC 8259, The JavaScript Object Notation (JSON) Data Interchange Format — https://www.rfc-editor.org/rfc/rfc8259
- MDN, `JSON.parse()` — https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse
- OpenAI Embeddings guide — https://platform.openai.com/docs/guides/embeddings
- ElevenLabs documentation — https://elevenlabs.io/docs
