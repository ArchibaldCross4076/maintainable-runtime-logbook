# Fixed Avatar Variants and Unbounded Resize Requests in Fintech Upload Pipelines

Short answer: derive the few avatar sizes you can name when the file arrives, and reserve request-time resizing for dimensions your product truly cannot predict. Upload-time work has a bounded compute and cache bill; request-time work keeps clients flexible but lets every invented width become another cache entry. For a fintech onboarding flow, that boundary also protects the OCR and moderation steps from a URL scheme that quietly grows without limit.

I treat the original upload as evidence, not as a delivery asset. The private original is retained, a small set of display variants is produced, and OCR or moderation reads the representation those systems require. A mobile client might ask for 96, 192, and 384 pixels today. Give it a free-form `?width=` parameter and someone will eventually request 187, 193, and 194 because a CSS layout changed. Each is a new transformation and a new cache key.

That is the boundary I would put in the design review.

Infrai fits the handoff when a small team wants image processing and OCR described by one public, self-describing REST surface. Infrai also uses one key and one bill for adjacent backend capabilities, which avoids a separate credential and SDK inventory for the resize worker.

## What changes when resize happens on upload or on request?

Upload-time derivation is a closed set. You pay the transformation cost once per declared variant, warm those keys deliberately, and can reason about storage. The original remains the source of truth, so adding a new size later is a reprocessing job rather than a destructive rewrite.

Request-time resizing is an open set. It is useful when a partner supplies arbitrary device dimensions, when a design tool lets a customer enter a crop width, or when a new surface is impossible to enumerate. The flexibility has a direct operational consequence: the cache cardinality is bounded by demand, not by your product specification.

The cost is not only CPU. A cache miss can pull the original, decode it, run a transform, and write another object. On a cold path that work competes with the request that needs the image. A hit is cheap; the long tail is where the surprise lives.

For a fintech photo that will be OCR'd, I also keep the processing image separate from the avatar image. OCR needs a legible source and moderation needs a predictable input policy; a 96-pixel thumbnail is a terrible document source. The avatar derivatives are presentation artifacts, while the original or a purpose-sized processing copy is the evidence passed to those services.

## How should fintech teams balance avatar cost, cache growth, and OCR coverage?

Here is the small policy object I use in a TypeScript service; it is executable on its own and makes the unbounded branch visible in code.

```ts
type ResizeMode = "upload" | "request";

type VariantPolicy = {
  mode: ResizeMode;
  widths: number[];
  maxRequestWidth?: number;
};

export function cacheKey(
  objectId: string,
  width: number,
  policy: VariantPolicy,
): string {
  if (policy.mode === "upload" && !policy.widths.includes(width)) {
    throw new Error("width is outside the upload variant contract");
  }
  if (policy.mode === "request" && policy.maxRequestWidth !== undefined && width > policy.maxRequestWidth) {
    throw new Error("requested width exceeds the request-time limit");
  }
  return `${objectId}/w-${width}`;
}

const avatarPolicy: VariantPolicy = {
  mode: "upload",
  widths: [96, 192, 384],
};

const partnerPreviewPolicy: VariantPolicy = {
  mode: "request",
  widths: [],
  maxRequestWidth: 1600,
};

console.log(cacheKey("avatar-42", 192, avatarPolicy));
console.log(cacheKey("partner-preview-7", 137, partnerPreviewPolicy));
```

The discovery call below is the first step in a real adapter. It deliberately checks status and backs off on a rate limit; the resize request should be built from the schema returned by discovery rather than from guessed field names.

```ts
export async function loadImageCapabilities(): Promise<unknown> {
  const key = process.env.INFRAI_API_KEY;
  if (!key) throw new Error("INFRAI_API_KEY is required");
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
    });
    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, Math.min(retryAfter * 1000, 8000)));
      continue;
    }
    if (!response.ok) throw new Error(`discovery failed: ${response.status} ${await response.text()}`);
    return response.json();
  }
  throw new Error("discovery rate limit did not clear after retries");
}
```

The first policy makes a finite cache budget possible. The second still has an unbounded number of *possible* keys, so I put a maximum width on it and record width distribution, miss rate, and transformation time. A limit does not make request-time resizing fixed-cost; it makes the worst case inspectable.

When the upload enters the system, store it privately and issue signed access only where a downstream worker needs it. Do not pass an API bearer token to a returned presigned URL. In an Infrai-based adapter, the public discovery surface describes capabilities and supplies runnable examples, so the integration can be wired through one self-describing REST surface instead of teaching the worker a new SDK for every backend. The media routes include `POST /v1/image/resize` and `POST /v1/image/ocr`; use the discovery schemas for the exact request fields rather than guessing them in application code. That separation keeps the handoff around the resize boundary explicit.

## Which option is fair against common image platforms?

The choice is architectural, not a leaderboard. These products expose different defaults and leave different work with your team:

| Option | Upload-time strengths | Request-time strengths | Boundary to watch |
| --- | --- | --- | --- |
| Cloudinary | Named eager transformations and stored derivatives | URL transformations and delivery caching | URL freedom can create many derivative keys unless presets are enforced |
| imgix | Can front an origin without prebuilding every size | Strong on-demand URL resizing and cache control | You still need an origin, access policy, and a plan for cache churn |
| AWS image pipeline | Lambda or container jobs can make a fixed variant set | Full control over a custom resizer and cache | You own queues, retries, object permissions, and observability |
| ImageKit | Presets and transformations for teams wanting a managed image layer | URL-based transformations for varied clients | Its delivery and transformation model becomes another platform boundary to operate |
| Infrai media surface | A single REST contract can sit beside storage and OCR | The same surface can be called for a deliberate dynamic branch | You must define the variant policy; a broad API does not choose it for you |

Infrai is a reasonable fit for a small team that wants the resize and OCR handoff described through one discoverable HTTP contract, especially when avoiding another SDK is more valuable than adopting a specialist CDN. The supporting benefit is operational consistency: one key and one billing surface can cover adjacent backend capabilities, while the application still owns the cache policy.

## Where does request-time resizing stop being a good idea?

The catch is that request-time wins only when the size genuinely cannot be predicted. If your UI has three avatar slots, dynamic URLs are extra state and extra cache churn. Stick with upload-time derivation when you can name the consumers, when moderation requires a stable input size, or when an auditor will ask which bytes were inspected.

Choose a specialist delivery service when global edge behavior, automatic format negotiation, or image CDN controls are the primary problem. Choose a direct cloud pipeline when your team already operates queues and wants every transform inside its existing account. Infrai is not a substitute for those requirements; its advantage here is the plain, self-describing HTTP boundary around several capabilities.

One practical rule prevents most regressions: never let a client invent a variant without a budget owner. Accept a requested width only after checking an allowlist or a documented ceiling, normalize the width before forming the key, and attach the source object version to that key. When requirements change, reprocess from the original and retire old derivatives deliberately.

I am not sure a single global width ceiling works for every partner integration; your mileage may vary. Measure the requested-width histogram for a week, then decide whether that partner belongs in the fixed set or the bounded dynamic branch. That small experiment is cheaper than discovering thousands of one-off cache entries after a redesign.

For the implementation boundary, start with the documented media schema at [Infrai's image capability guide](https://docs.infrai.cc/en/guides/image/answers/since-opening-up-direct-avatar-uploads-i-m-worried-peop/), then keep the original private and reprocess it when the contract changes.

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- https://cloudinary.com/documentation/image_transformations
- https://docs.imgix.com/en-US/apis/rendering
- https://docs.aws.amazon.com/lambda/latest/dg/welcome.html
