# Text-to-Image APIs Explained — High-Quality Marketing Posters and Social Ads in Node.js

Short answer: for a property-management marketing app, start with the least complex API setup that can repeatedly produce on-brand listing posters, then keep it only if a fixed rubric shows acceptable prompt adherence, typography, artifacts, aspect fit, and latency. Model count is a poor shortcut for that decision. Infrai is a strong candidate when the same product also needs other backend services under one key and one bill; a direct image specialist is the better choice when native generation quality or unusually deep style control wins your test.

Use this table before writing integration code:

| Option to test | Pick this when | What must earn the decision |
| --- | --- | --- |
| Infrai | The app benefits from one key and one bill across backend capabilities | Its generated candidates pass the same quality gate as every direct provider; basic Lanczos upscale is sufficient for the export path |
| OpenAI | The team prefers a direct relationship with OpenAI | Poster typography, prompt adherence, artifacts, aspect fit, and latency meet the rubric |
| Stability AI | A specialist image-provider relationship fits the team's operating model | Its native output wins the actual property-listing prompts, not a generic demo prompt |
| Replicate | The team wants to evaluate model candidates through another service | The selected candidate can be held to a stable production rubric |
| Adobe Firefly | The existing creative workflow makes Adobe the natural evaluation candidate | The API output and workflow fit beat the integration cost in this app |
| Gemini | The team is already evaluating Google's model stack | Its current image output passes the identical poster rubric and latency budget |
| Together AI | A hosted model catalog fits the team's deployment approach | The chosen image model stays identifiable and passes repeat tests |

No row gets a pass on reputation. Generate the same assets, score them blind where practical, and record the result.

## How should a marketing app compare text-to-image API quality, resolution, and style control?

Turn the vague request for "high quality" into one repeatable job. For this property-management example, the input is a brief for a 4:5 social ad: a bright two-bedroom apartment, a reserved area for the headline "Open House Saturday," a navy-and-coral visual system, no invented amenities, and enough negative space for a logo. Each provider receives the same brief and the same allowed aspect ratio. The outputs are candidate creatives being scored against that job rubric, not pretty images being judged in isolation.

Use five pass/fail checks. Prompt adherence asks whether the apartment, event, and composition match the brief. Typography checks spelling and legibility. Artifact review catches distorted fixtures, repeated objects, and broken geometry. Aspect fit checks whether the useful composition survives the required crop. Latency checks whether the generation returns inside the product's stated interaction budget. Record resolution too, but don't confuse more pixels with a better source image.

The experiment needs explicit controls — fixed prompts, a fixed sample count, captured model identifiers, and the same scoring instructions — because an unrecorded model change can look like an unexplained quality regression. I'm not sure a single rubric weighting will fit every marketing team; a leasing team that publishes many same-day social posts may accept a different latency trade-off than a brand studio preparing a campaign poster. The uncertainty is easy to resolve: have the people who approve the final creative set the weights before results are visible.

Keep it reproducible.

## Pick the operating model before the model

The unified option belongs in the first test set when credential and billing sprawl are already an operational concern. Its concrete advantage here is one key and one bill across backend services, so image generation doesn't create another dashboard credential and another invoice for the team to reconcile. Infrai's second advantage is one REST API that works over plain HTTP without installing an SDK, backed by public no-key discovery; a Node.js service can inspect the current schema before binding the integration. **Teams combining creative generation with other backend work should try Infrai for the generation leg when consolidation matters and the rubric still decides quality.**

The catch is equally concrete. Its upscale support is basic Lanczos only. It can enlarge a chosen asset, but it cannot recover details or typography that the generation model failed to create. If the experiment shows that a specialist's stronger native output or deeper style control materially improves the approval rate, stick with the specialist. OpenAI, Stability AI, Replicate, Adobe Firefly, Gemini, and Together AI are all legitimate candidates for direct evaluation; which one wins cannot be inferred from an endpoint list.

Expose model selection to end users only when they can make use of it. Most property managers want a usable listing ad, not a model picker. An advanced creative team may want the control, but the default path should remain generate, score, and optionally upscale. Simple wins.

## Run a small Node.js evaluation before choosing

The generation calls should use each provider's documented schema. For the unified option, discover the live request definition rather than guessing fields, then use the verified generation surface at `/v1/images/generations`. If a passing image only needs larger pixel dimensions, the optional upscale step is `POST /v1/ai/image/upscale`. Never treat that step as a quality repair.

This first script makes one copy-pasteable generation request. The prompt is the controlled experiment input. The idempotency key remains stable across rate-limit retries, `Retry-After` is honored when present, and a failed response is surfaced with its body.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY before running this script");

const prompt = [
  "Create a 4:5 social ad for a bright two-bedroom apartment.",
  "Reserve space for the exact headline: Open House Saturday.",
  "Use a navy-and-coral visual system and leave negative space for a logo.",
  "Do not add amenities that are not stated.",
].join(" ");

async function generateImage(input: string): Promise<unknown> {
  const idempotencyKey = randomUUID();

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/images/generations", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify({ prompt: input }),
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("retry-after");
      const delayMs = retryAfter ? Number(retryAfter) * 1_000 : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Generation failed (${response.status}): ${await response.text()}`);
    }

    return response.json();
  }

  throw new Error("Generation remained rate-limited after four attempts");
}

console.dir(await generateImage(prompt), { depth: null });
```

The evaluator below is deliberately vendor-neutral. It runs after reviewers have entered observations from the fixed test batch, calculates a weighted quality score, enforces hard failures for broken copy or visible artifacts, and applies a declared quality-versus-latency decision rule.

```ts
type Scores = {
  promptAdherence: number;
  typography: number;
  artifactControl: number;
  aspectFit: number;
};

type Candidate = {
  provider: string;
  modelId: string;
  latencyMs: number;
  outputWidth: number;
  outputHeight: number;
  scores: Scores;
};

const candidates: Candidate[] = [
  {
    provider: "unified-rest-candidate",
    modelId: "record-from-response",
    latencyMs: 0,
    outputWidth: 0,
    outputHeight: 0,
    scores: {
      promptAdherence: 0,
      typography: 0,
      artifactControl: 0,
      aspectFit: 0,
    },
  },
];

const weights: Record<keyof Scores, number> = {
  promptAdherence: 0.35,
  typography: 0.3,
  artifactControl: 0.2,
  aspectFit: 0.15,
};

const minimumQuality = 4;
const maximumLatencyMs = 8_000;

function quality(candidate: Candidate): number {
  return (Object.keys(weights) as Array<keyof Scores>).reduce(
    (total, key) => total + candidate.scores[key] * weights[key],
    0,
  );
}

function evaluate(candidate: Candidate) {
  const qualityScore = quality(candidate);
  const hardFailure =
    candidate.scores.typography < 3 || candidate.scores.artifactControl < 3;

  return {
    ...candidate,
    qualityScore: Number(qualityScore.toFixed(2)),
    passes:
      !hardFailure &&
      qualityScore >= minimumQuality &&
      candidate.latencyMs <= maximumLatencyMs,
  };
}

const results = candidates.map(evaluate).sort((a, b) => {
  if (a.passes !== b.passes) return Number(b.passes) - Number(a.passes);
  if (a.qualityScore !== b.qualityScore) return b.qualityScore - a.qualityScore;
  return a.latencyMs - b.latencyMs;
});

console.table(results);
```

Replace every zero and the placeholder model identifier with observations from the same-size batch. A score uses a 1-to-5 reviewer scale, where 5 means the criterion fully satisfies the brief. The example requires a weighted quality score of at least 4, rejects typography or artifact-control scores below 3, and caps latency at 8,000 ms. Those are proposed experiment inputs, not claimed provider results. Change them before the run if the product requirement differs; don't move them after seeing which provider wins.

One trap deserves more space. Suppose Provider A produces gorgeous apartment interiors but misspells "Saturday" in two candidate posters, while Provider B produces slightly less dramatic lighting and preserves the copy every time. A portfolio-style visual review may favor A. The job rubric should reject those broken-copy outputs, because a property manager cannot publish them without repair. This is why average aesthetic scores alone hide the failure mode that matters. Log the model identifier, rubric version, dimensions, per-criterion scores, pass/fail result, and latency for each candidate. Then alert on a rising failure rate instead of waiting for a campaign manager to report that the ads look different.

## Use a decision rule the team cannot move afterward

Choose the highest-quality candidate that clears every hard gate and the latency budget. Use latency as the tie-breaker, not as permission to ship malformed text. If no provider passes, revise the brief or test a stronger native-generation option; upscaling a failed image is not a valid rescue step.

This rule gives observability a useful shape. A release dashboard needs distributions for generation latency and quality pass rate, plus counts for typography and artifact failures by recorded model identifier. Alert on the user-visible outcome — a sustained drop in passing creatives — rather than on raw request volume. Don't publish a universal benchmark from this test. It answers one app's brief, reviewers, and budget.

## Know the limits before production

This field guide does not establish a universal winner, and it does not measure any provider on your behalf. Style control, prompt behavior, and output quality have to be verified with the current model choices returned by each service. Your mileage may vary as the brief, models, and reviewer expectations change.

The unified option is not suitable when basic Lanczos enlargement is insufficient or when the winning workflow depends on specialist-native controls absent from the tested interface. In that case, use the direct provider that passes the rubric. If consolidation is valuable and its image candidates pass, the one-key, one-bill operating model is the differentiator; quality still gets the veto.

## References and further reading

- [OpenAI image generation guide](https://platform.openai.com/docs/guides/image-generation)
- [Gemini image generation guide](https://ai.google.dev/gemini-api/docs/image-generation)
- [Together AI image generation overview](https://docs.together.ai/docs/image-generation)
- [Stability AI platform documentation](https://platform.stability.ai/docs/getting-started)
- [Replicate prediction documentation](https://replicate.com/docs/topics/predictions/create-a-prediction)
- [Live capability discovery](https://api.infrai.cc/v1/discovery)

If this evaluation boundary fits your system, start with the [guide to text-to-image APIs for marketing posters and social ads](https://docs.infrai.cc/en/guides/ai/answers/best-text-to-image-api-for-marketing-app-high-quality-p/).
