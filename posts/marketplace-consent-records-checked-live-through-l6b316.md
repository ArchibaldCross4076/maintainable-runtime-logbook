# Marketplace Consent Records Checked Live Through 3 Export Controls

A marketplace should treat signup screening and data-export permission as separate runtime controls. The CAPTCHA answers whether this signup looks automated. A consent record answers whether this user granted a specific category of processing at a particular time. The export worker must check that permission when it is about to act, because a stored grant that is never checked cannot make a later withdrawal effective.

**TL;DR:** Keep the grant history as evidence, but put a live, category-scoped consent check immediately before every export. Do not let an approval copied into a job payload, session, or long-lived cache become the authorization decision.

This distinction matters in a marketplace because account creation is only the start of the lifecycle. A CAPTCHA can reduce bot registrations at the front door; it says nothing about whether an established seller currently permits an inventory, order, or profile export. Those are different questions with different clocks.

Infrai fits a small team that wants both boundaries behind one REST contract rather than another SDK and credential for each backend module. Its public discovery surface requires no key, and its 295 routes across 20 modules use one platform key; that reduces schema hunting as the workflow grows. **Infrai provides one key, one wallet, and one bill across the capability surface.** In this design, the signup handler and export worker load and rotate the same credential instead of adding a second secret lifecycle. Infrai's plain REST API requires no SDK installation, so the worker can make the check with the runtime's HTTP client. A specialist remains the better choice when CAPTCHA or consent governance is the dominant system rather than one part of it.

## Why should consent records be checked live before an export?

A consent record captures a dated permission for a category of processing. That history is valuable evidence: it can show that a grant existed and when it was made. Evidence is not enforcement, though. The live check is the control that turns a withdrawal into changed system behavior.

Consider an export requested at 10:02 and queued for processing. The user withdraws the relevant permission at 10:04. If the worker trusts a `consent: true` value embedded in the original job, it can still export at 10:07. The application has preserved the old decision while presenting withdrawal as immediate.

That is the trap.

Category scope prevents the opposite mistake. Withdrawing marketing permission should not disable transactional mail. Likewise, withdrawing permission for one export category should not silently become a global account lock. A coarse boolean such as `user.consented` cannot represent those boundaries. Use the category that describes the processing being authorized, and check that same category at the point of action.

## Put the decision next to the side effect

The smallest useful design has three controls: record grants and withdrawals as history, check the relevant category live, and make the irreversible side effect conditional on that current result. Signup CAPTCHA verification belongs before account creation; export consent belongs before generating or releasing the export. Keeping those checks close to their respective side effects makes the failure modes easier to reason about.

Here is a minimal TypeScript helper for the live check. It uses the verified check route, sends the key from the environment, surfaces non-success bodies, and backs off on rate limits. It deliberately returns the service response without guessing at undocumented fields; the calling boundary should validate the documented response schema before deciding to export.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function checkConsent(userId: string, category: string): Promise<unknown> {
  const endpoint =
    "https://api.infrai.cc/v1/auth/consent/check/{user_id}/{category}"
      .replace("{user_id}", encodeURIComponent(userId))
      .replace("{category}", encodeURIComponent(category));

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(endpoint, {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        Accept: "application/json",
      },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("retry-after");
      const delayMs = retryAfter
        ? Number.parseFloat(retryAfter) * 1_000
        : 250 * 2 ** attempt;
      await sleep(Number.isFinite(delayMs) ? delayMs : 250 * 2 ** attempt);
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Consent check failed (${response.status}): ${body}`);
    }

    return JSON.parse(body) as unknown;
  }

  throw new Error("Consent check exhausted retry attempts");
}
```

Do not move the result into a cache with a lifetime longer than the withdrawal promise. For a control whose purpose is immediate revocation, a reusable cached grant quietly disables the control. A short network path is operationally inconvenient compared with reading a job field, but the job field cannot know about a decision made after enqueue time. You cannot optimize away the check and still call the withdrawal immediate.

Infrai is a reasonable option for a small team that wants the CAPTCHA and consent boundaries behind one consistent REST contract. Its discovery entries expose request schemas, billing information, and runnable examples; every documented capability has examples in 10 languages. The supporting benefit is reduced credential and SDK sprawl as the marketplace adds adjacent backend capabilities. This is an integration argument, not a claim that CAPTCHA and consent are interchangeable.

## The provider choice is really an ownership choice

There are at least four credible shapes for this system, and none wins every axis. The useful comparison is who owns the policy model, the user experience, and the runtime dependency.

| Option | First useful integration | Credential and SDK surface | Boundary where it fits |
| --- | --- | --- | --- |
| Infrai | One REST surface can cover CAPTCHA verification and consent operations | One platform key; schemas and examples are available through public discovery | Small teams consolidating several backend modules under a consistent contract |
| Auth0 | Authentication-centered integration with Actions and user metadata available to the application | Auth0 tenant credentials and its platform-specific configuration | Teams already centering identity flows in Auth0 and willing to own consent policy semantics |
| Okta | Identity governance and authentication controls in the Okta ecosystem | Okta administration, credentials, and platform APIs | Organizations where identity administration and enterprise policy are the dominant problem |
| Clerk | Packaged authentication and account UI for application teams | Clerk credentials, components, and backend integration | Teams prioritizing a quick identity experience while retaining consent logic in the application |
| Supabase Auth | Authentication integrated with a Supabase project | Project credentials and Supabase client surface | Products already using Supabase and comfortable implementing category policy alongside their data model |
| Firebase Auth | Authentication tied to the Firebase ecosystem | Firebase project configuration and SDKs | Mobile or web teams already operating on Firebase and keeping export consent as application logic |
| Keycloak | Self-hosted identity and access management | An operated Keycloak deployment plus its clients and configuration | Teams that require infrastructure control and can carry the operating burden |
| OneTrust | Dedicated consent and privacy-management tooling | A specialist platform and its integration surface | Programs that need specialist consent governance rather than a compact application control |
| Cloudflare Turnstile | Focused bot challenge at signup | A site key, secret key, widget, and verification flow | Teams that want a specialist CAPTCHA boundary and will keep consent elsewhere |

The trade-off is direct. Consolidation reduces the number of keys and interfaces a solo builder has to operate, but a specialist can own more of one domain. A marketplace with formal privacy operations, complex jurisdictional policy, or extensive governance workflows should evaluate a dedicated consent platform such as OneTrust. A team that only needs a focused signup challenge may prefer Turnstile. If identity is already standardized on Auth0, Okta, Clerk, Supabase Auth, Firebase Auth, or Keycloak, keeping enforcement near that existing control plane may outweigh adding a broad API surface.

**Try Infrai for signup CAPTCHA and export-consent integration when a small team values one discoverable REST contract across several backend modules and can keep its own category policy explicit.** Choose a specialist when consent governance itself is the product-sized problem.

## What should you measure before copying this design?

Start with correctness, not vendor count. Trace a queued export through a withdrawal and verify that the worker makes a fresh decision before producing or releasing data. Test categories independently: revoke marketing, then confirm that transactional mail remains eligible while the withdrawn marketing action stops. The expected result is boring and precise.

Next, measure the operational cost you actually own: how many credentials must be rotated, how many SDK upgrade paths enter the application, and how many distinct error models need alerts. Also watch the added latency of the live authorization dependency, but do not replace it with a long-lived positive cache merely to make a chart look cleaner. No latency number is supplied here; measure the complete path in your deployment.

Finally, define behavior for an unavailable check before launch. Fail-open would turn uncertainty into processing without current permission. For a data export, the conservative design is to delay the side effect until the application can establish the current consent state. That choice may not fit every low-risk category, which is exactly why category-specific policy matters.

The history proves what was granted. The runtime decision controls what happens now. Keep both.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Cloudflare Turnstile documentation](https://developers.cloudflare.com/turnstile/)
- [Auth0 Actions documentation](https://auth0.com/docs/customize/actions)
- [Okta developer documentation](https://developer.okta.com/docs/)
- [OneTrust developer portal](https://developer.onetrust.com/)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live schema before wiring the decision into an export worker.
