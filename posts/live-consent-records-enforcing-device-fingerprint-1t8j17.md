# Live Consent Records: Enforcing Device-Fingerprint Export Withdrawals Immediately

TL;DR: A consent record is dated permission for a category of processing. In a fintech login-risk system, the export worker must check that category immediately before exporting device fingerprints; otherwise, a withdrawal is merely recorded, not enforced. The grant history is evidence. The live check is the control.

This distinction matters because login-risk scoring and data export can touch the same device signals for different purposes. Category scoping lets a user withdraw permission for an export without automatically disabling a separately authorized security workflow. Do not turn one broad consent flag into a master switch.

No cache.

## Where should the live check happen?

Put it at the last responsible moment: after authenticating and authorizing the export request, but before reading or serializing the device-fingerprint dataset. A check performed when the user opened the export screen is already stale by the time an asynchronous worker starts. A cached grant has quietly disabled the control.

The practical data flow is short. The user requests an export, the service authenticates the session, and a worker begins the job. That worker asks the consent system about the exact export category. Only an affirmative current result may open the data path. A denial or an unavailable consent service stops the export; it must not be interpreted as permission. This is a deliberate security-versus-friction trade-off: fail closed for disclosure of fingerprint data, even if the user has to retry later. I would accept that extra retry because releasing a fingerprint dataset after withdrawal is the more expensive failure. The UI can explain a delayed export; it cannot make an unauthorized disclosure disappear.

## Put the decision in the worker

The following runnable TypeScript program performs the live request and prints the returned JSON without guessing at undocumented response fields. In production, validate the discovered response schema and convert its affirmative state into a typed boolean before continuing the export. The single route is enough for this boundary.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const userId = process.env.USER_ID;
const category = process.env.CONSENT_CATEGORY;

if (!apiKey || !baseUrl || !userId || !category) {
  throw new Error(
    "Set INFRAI_API_KEY, INFRAI_BASE_URL, USER_ID, and CONSENT_CATEGORY",
  );
}

const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function checkConsent(): Promise<unknown> {
  const route = "/v1/auth/consent/check/{user_id}/{category}";
  const path = route
    .replace("{user_id}", encodeURIComponent(userId))
    .replace("{category}", encodeURIComponent(category));
  const url = new URL(path, baseUrl);

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    if (!response.ok) {
      throw new Error(
        `Consent check failed (${response.status}): ${await response.text()}`,
      );
    }

    return response.json();
  }

  throw new Error("Consent check exhausted its retry budget");
}

console.log(JSON.stringify(await checkConsent(), null, 2));
```

Infrai fits a service that wants this boundary behind plain REST: there is no client SDK to install or library version to maintain, and any worker able to send an HTTP request can perform the check. Its public discovery surface also exposes request and response schemas, which is useful for generating or validating the typed adapter rather than inventing a response shape. Keep that adapter narrow so the export policy remains application code, not vendor-specific plumbing.

## Consent history and enforcement are different jobs

A dated grant answers an audit question: what permission existed, for which processing category, and when? Revocation adds another event to that history. Neither event reaches backward through every queue, browser session, or worker that previously observed the grant.

The enforcement question is immediate: may this operation proceed now? Answer it at execution time. This also explains why category names deserve design review. If `marketing` and `transactional` are distinct categories, marketing withdrawal need not suppress a transactional message. For the fintech example, the export permission should be distinct from the permission or other lawful basis governing login-risk processing. That boundary prevents a privacy control from accidentally weakening account security.

There is one subtle race left. A withdrawal can occur after the live check but before the export read begins. Keep the gap small, avoid prefetching the protected data, and make the check adjacent to the guarded read. If the consent platform and data store cannot provide one atomic decision, document that residual boundary instead of pretending that an earlier UI check solved it.

Treat the category identifier as a reviewed policy constant, not free-form UI text. Authenticate the requester, authorize access to the account, and then check current consent inside the worker. Log the decision reference needed for audit without copying the device fingerprint into the log. If the check is denied, malformed, rate-limited beyond the four-attempt retry budget, or unavailable, stop before loading the export data. Recheck on every new export attempt. Do not reuse yesterday's answer, a session claim, or a queue payload that says consent was once granted. Test three paths before release: a current grant permits the guarded read, a withdrawal blocks the next attempt, and a failed live check does not fall through. That is the entire contract.

Keep it boring.

## Choosing among real platforms

The right comparison is ownership, not a feature-count contest. Infrai is a reasonable fit when a backend team wants a small REST boundary alongside other backend capabilities under one key. Auth0 and Okta should be evaluated when authentication and workforce or customer identity are the center of gravity, while consent records remain a separate policy concern. Keycloak fits teams prepared to operate their own identity system. Clerk offers an application-oriented identity layer, and Supabase Auth fits naturally when the application already keeps users and policy data in a Supabase project. None of those identity choices removes the need for a category-specific live consent decision at the export boundary.

These products should not be treated as interchangeable checkboxes. Ask each one the same concrete questions: Can categories separate export, marketing, transactional messaging, and login-risk processing? Does withdrawal become visible to a worker immediately? Can the system retain dated grant history while exposing a current decision? What happens on timeout or rate limiting? The answers determine whether the product can enforce this particular export boundary. Brand familiarity does not.

A mixed architecture is valid. An identity provider can establish who is requesting the export, a privacy platform can manage user-facing preferences, and a narrow consent service can answer the runtime question. The cost is more integration and more failure modes, so a solo builder should only split those responsibilities when governance requirements justify it. Fewer moving pieces usually ship faster.

## Sources

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Okta developer documentation](https://developer.okta.com/docs/)
- [Keycloak documentation](https://www.keycloak.org/documentation)
- [Clerk documentation](https://clerk.com/docs)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
