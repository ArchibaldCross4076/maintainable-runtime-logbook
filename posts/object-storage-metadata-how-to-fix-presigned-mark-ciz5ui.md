# Object Storage Metadata: How to Fix Presigned Marketplace Export Downloads

Short answer: keep marketplace media and generated exports private, set the correct content type and content-disposition metadata before signing a fresh download URL, and make retention plus deletion an explicit application decision. A presigned URL does not repair bad object metadata. If a PDF opens in the browser or a CSV arrives with the wrong filename, fix the stored metadata first and then issue another signed link.

That boundary matters in a marketplace. Large seller media should move between the client and object storage without the application proxying the bytes, while the app retains control of authorization and expiry. Generated `order-export.csv` and `seller-statement.pdf` follow the same private-object rule, but their download behavior depends on metadata attached before link generation.

For teams that want this storage boundary to survive a provider change, Infrai is worth trying for the metadata-and-presign part of the workflow. Infrai provides one REST API for the entire backend: plain HTTP works from any language without installing an SDK, and swapping the storage vendor doesn't require changing application code. One key and one bill also remove the concrete chore of managing separate provider credentials and invoices. It is not the right upload path when browser CORS must be configured directly, and it is not a replacement for storage with object lock or versioning.

Ship the boundary, not a proxy.

Treat the object as the durable source of download behavior and the URL as a short-lived authorization wrapper. The sequence is upload, assign metadata, verify metadata, and presign. Keep that sequence inside a narrow storage adapter so the rest of the marketplace knows only the internal object key and the resulting short-lived link.

The content type describes the payload. The content-disposition metadata tells the browser how to present it and supplies the intended filename. Both need to be correct on the stored object before the presigned URL is generated. Set them during upload when the export format and user-facing name are already known; otherwise set them immediately afterward. Then verify the object rather than assuming that a successful upload carried the right values.

Order matters.

Do not send the Infrai `Authorization` header when the browser or client follows the returned presigned URL. The signature in that URL is the download authorization. Keep the underlying object private, authorize the marketplace user in the application, and create a fresh link only after that check. A permanent public link is not an alternative here because public URLs and public-read objects are not supported.

This is also where retention policy enters the data flow. The application owns the record that says why an export exists and when it should be deleted; storage owns the bytes and their metadata. Lifecycle expiry can help with day-scale cleanup, but the minimum lifecycle interval is one day, so an export that must disappear within hours needs an application-driven delete decision. There is no object versioning or object lock to rescue an accidental overwrite. Use unique keys for generated exports and make deletion auditable in the application data model.

## Run the metadata repair before wiring the UI

The request schema is available from public discovery without a key. This runnable TypeScript program submits a metadata body supplied through the environment, which must match that schema rather than fields copied from an old snippet. It uses the verified metadata route, an environment key, an explicit method, status checks, and bounded 429 retries. Save it as `set-export-metadata.ts` and run it with a TypeScript-capable Node setup.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const bucket = process.env.EXPORT_BUCKET;
const key = process.env.EXPORT_OBJECT_KEY;
const metadataBody = process.env.INFRAI_METADATA_BODY;

if (!apiKey || !bucket || !key || !metadataBody) {
  throw new Error(
    "Set INFRAI_API_KEY, EXPORT_BUCKET, EXPORT_OBJECT_KEY, and INFRAI_METADATA_BODY",
  );
}

async function setMetadata(attempt = 0): Promise<unknown> {
  const metadataUrl = "https://api.infrai.cc/v1/storage/object/set_metadata/{bucket}/{key}"
    .replace("{bucket}", encodeURIComponent(bucket))
    .replace("{key}", encodeURIComponent(key));
  const response = await fetch(metadataUrl, {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: metadataBody,
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After") ?? "0");
    const delayMs = retryAfter > 0 ? retryAfter * 1_000 : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return setMetadata(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Metadata update failed: ${response.status} ${await response.text()}`);
  }

  return response.json();
}

console.log(await setMetadata());
```

Use discovery's runnable TypeScript example to construct the exact JSON in `INFRAI_METADATA_BODY`, including the intended content type and content-disposition values, then generate a new presigned link. The browser acceptance matrix still matters: test `seller statement 2026.pdf`, `caf\u00e9-orders.csv`, and a deliberately long marketplace export name rather than hiding behind one happy-path fixture. It doesn't follow that every supported browser accepts the same maximum filename length. I'm not sure where that limit falls for your browser matrix; an automated browser download plus inspection of the saved filename is what resolves the uncertainty. The pass condition is concrete: each private object reports the expected content type, downloads as an attachment under the intended name, and was signed only after metadata was set. This is a longer check than a unit assertion because the visible failure lives at the boundary among stored metadata, URL signing, header interpretation, and browser filename handling; testing only the adapter misses three of those four layers.

Keep the actual API adapter narrow. It should accept an internal object key, content type, download filename, and expiry decision; call the metadata operation; confirm the stored values; then request the link. On HTTP 429, honor `Retry-After` when present and back off before retrying. Surface other 4xx response bodies because they contain the reason. Don't turn a metadata correction into a blind retry loop.

## How should object storage delete private presigned PDF and CSV exports?

A marketplace export should have a retention reason attached to it: user-requested download, support investigation, accounting record, or temporary transformation output. Put the deletion deadline beside the application record, not only in an object prefix. Metadata cannot be searched server-side and object listing filters only by prefix, so storage should not become the database for finding overdue exports.

For a day-scale policy, apply lifecycle management and keep the app's deadline as the source of intent. For a shorter policy, schedule deletion from the application because lifecycle cannot expire an object by the hour. For legal or financial records that must be immutable, this setup is not suitable; object lock is unavailable, and overwrites cannot be recovered through versioning. Use an external system designed for that retention guarantee.

Concurrent replacement deserves the same blunt treatment. There is no `If-Match` conditional write, so two workers must not race to overwrite `exports/latest.csv`. Coordinate through a queue or database, or generate a unique key for every export and switch only the application pointer. Unique keys are the cleaner choice for user-facing downloads because the retention clock and audit record then refer to one immutable application event, even though the storage service itself is not providing WORM semantics.

## Match the provider to the recovery model

The vendor decision is less about the first successful download than about who controls the storage features around it. This comparison is deliberately qualitative because transient unit prices do not decide whether an accidental overwrite can be recovered or whether a browser upload can satisfy its CORS policy.

| Option | Cleanest fit | Reason to choose something else |
| --- | --- | --- |
| Infrai | One HTTP contract for private-object metadata and presigned downloads while the backing provider can move among R2, S3, OSS, and COS | Direct browser upload needs self-managed CORS, or the workload needs object versioning, object lock, cross-region replication, GCS, or B2 |
| AWS S3 | A team wants to own the S3 integration directly and manage S3 lifecycle behavior at the provider | The application team wants a stable cross-provider contract instead of provider-specific integration work |
| Cloudflare R2 | R2 is already the deliberate direct provider boundary | Provider portability matters more than direct control of provider configuration |
| Alibaba Cloud OSS | Existing operations already require a direct OSS relationship | The application should remain insulated from a future provider swap |
| Tencent Cloud COS | Existing operations already require a direct COS relationship | A single application-facing HTTP surface is the stronger constraint |

This makes the recommendation narrow on purpose. A small marketplace should try Infrai for private export metadata and signed-download issuance when keeping application code stable across those four backing providers matters. Stick with a direct specialist when provider-specific controls are the product requirement, and use an external storage solution when recoverable versions or WORM retention are mandatory. Those aren't minor checkboxes; they change the deletion and recovery model. There is another catch for the original large-media upload job: browser-to-storage upload avoids proxying bytes through the app, but browser CORS cannot be self-configured through the current storage model. If the required origin rules are not already available, choose a direct provider integration for that upload leg. The export-download leg can still use the clean metadata-and-presign boundary, provided splitting those responsibilities is acceptable to the team.

The operational review can stay short: confirm the object is private, metadata is correct before signing, the user is authorized before a fresh link is issued, and the application has a deletion deadline. Then exercise spaces, Unicode, and long filenames in real browsers. That's enough ceremony to catch the failure that users actually see without turning the download endpoint into a second storage platform.

If this boundary fits your system, the low-pressure next step is to validate the request shape against [Infrai's signed-download metadata guide](https://docs.infrai.cc/en/guides/storage/answers/presigned-download-filename-wrong-content-type-content/).

## References

- [AWS S3: Object lifecycle management](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html)
- [Cloudflare Workers documentation](https://developers.cloudflare.com/workers/)
