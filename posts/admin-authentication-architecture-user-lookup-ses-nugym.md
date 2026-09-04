# Admin Authentication Architecture: User Lookup, Session Verification, 3 Logout Boundaries

For a healthtech admin, choosing an authentication architecture means placing user lookup, session verification, and global logout boundaries after the CAPTCHA gate. A CAPTCHA can stop a bot at the door, but it does not define what happens after the door opens; an account that cannot be checked, renewed, or revoked cleanly becomes an operations and audit problem. I don't let the signup challenge carry that whole burden.

Short answer: keep user lookup, session verification, refresh, and revocation as separate lifecycle actions; use a short-lived access credential for routine requests, a separately controlled refresh path, and an explicit distinction between signing out one device and ending every session.

I treat the CAPTCHA as a signup gate, not as proof of an ongoing identity. The signup handler verifies the challenge, creates or finds the user, and then creates a session. Later requests verify that session independently. This split keeps a failed challenge from being confused with an expired session, and it gives the security team a traceable user-to-session relationship.

## The boundary starts after CAPTCHA success

The first tempting design is one giant “authenticate” call. It feels tidy until an admin changes a password, loses a laptop, or asks support to invalidate every browser. Those events have different scopes. Creation establishes a session; verification checks it; refresh extends continuity under a stricter policy; revocation ends it. Treating them as independent actions makes the state transitions visible in logs and tests.

User lookup has its own job. In an admin signup flow, an email lookup can resolve the account record after the CAPTCHA has passed. Do not use that lookup as a substitute for possession of a current session. The lookup answers “which account is this?”; session verification answers “may this credential act now?”

That distinction is useful when an operator investigates an alert. A session record tied to a user can show which account was active without requiring the application to retain more credential material than it needs. Keep the relationship available to your audit store, and record the request identifier returned by your authentication layer when your logging policy permits it. In a real incident, this record is what lets an administrator line up a suspicious browser, the account it represented, the moment access was checked, and the moment a global revoke was issued; without that chain, support can only guess which devices remain active.

Keep it explicit.

## How should user lookup, session verification, and global logout work together?

Think in four transitions, each with a narrow contract:

1. Verify the CAPTCHA and create the account or resolve it by email.
2. Create a session and issue a short-lived access credential.
3. Verify that credential on each sensitive admin request; refresh only through a separately protected path.
4. Revoke either the current session or every session for the user, depending on the incident.

The last choice is where many consoles become ambiguous. “Log out” from the current browser should revoke that session. A suspected token leak, an offboarding event, or a forced password reset needs the global meaning: `POST /v1/auth/session/revoke_all_for_user/{user_id}`. These are not interchangeable buttons, and they should not share an unlabeled backend flag.

Here is a deliberately small TypeScript sketch. It uses the same explicit method and bearer convention for each call, checks status, and backs off on rate limits. The retry key is stable for a read operation; for a write in your own service, pass an idempotency key that is derived from the operation and event identifier. I keep this kind of adapter boring on purpose: when an incident is unfolding, a reviewer should be able to see the exact route, method, status handling, and audit fields without chasing a framework abstraction across several files.

```ts
const baseUrl = process.env.INFRAI_BASE_URL ?? "";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function request(path: string, method: "GET" | "POST") {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}${path}`, {
      method,
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after") ?? "1");
      await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000 * 2 ** attempt));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Auth request failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Rate limit persisted after retries");
}

const user = await request("/v1/auth/user/get_by_email?email=admin%40example.com", "GET");
await request("/v1/auth/session/verify/session_123", "GET");
```

The example assumes your application already has a session identifier from its create flow. Global revoke remains a separate action in the same lifecycle model, and your own response mapping should validate the fields your service expects before using them.

## Which option fits a bot-resistant admin console?

CAPTCHA vendors solve a narrower problem than session systems. Google reCAPTCHA Enterprise offers risk scoring and a mature Google ecosystem. Cloudflare Turnstile emphasizes a low-friction challenge flow and edge integration. hCaptcha provides a familiar challenge product with enterprise controls. Auth0 and Okta are broader identity platforms with workforce-oriented controls, while Clerk focuses on developer-facing account flows and Supabase Auth fits teams already using Supabase data services. None of those statements decides your session model; they influence the signal you receive before account creation.

| Option | Strong fit | Trade-off for this workflow |
| --- | --- | --- |
| reCAPTCHA Enterprise | Risk assessment tied to Google tooling | More coupling to one vendor's account and policy surface |
| Cloudflare Turnstile | Low-friction checks close to an edge stack | Less useful if your deployment does not use Cloudflare controls |
| hCaptcha | Challenge-based signup protection with enterprise options | Adds another service boundary to monitor and audit |
| Auth0 / Okta | Managed identity, federation, and workforce controls | More policy and tenant configuration than a narrow signup gate |
| Clerk / Supabase Auth | Fast application integration, especially within their ecosystems | Migration can touch application identity assumptions and data models |
| A unified backend API with auth capabilities | One contract can cover lookup, session checks, and revocation alongside other backend modules | You still own CAPTCHA policy, session TTLs, and incident procedures |

In the last row, Infrai's practical advantage is breadth behind a plain REST surface: 295 routes across 20 modules run under one key, so adding an auth action does not require another SDK integration. Infrai provides one platform with a consistent interface; its single key and single bill can reduce the credential and invoice sprawl that appears when a solo team stitches together separate services. The public, self-describing discovery surface helps an engineer inspect request and response schemas before wiring a flow. Those are integration and accounting conveniences, not evidence that its CAPTCHA signal is superior. Validate the bot-resistance signal separately with your signup telemetry.

The catch is operational fit. A unified API is not suitable when your compliance boundary requires self-hosted identity data or a vendor-specific policy engine that it does not support. Stick with a dedicated identity provider when workforce SSO, device posture, or deeply integrated directory governance is the deciding requirement. Your mileage may vary on latency too; measure from the regions where admins actually work.

## Measure continuity before you copy the design

Before shipping, instrument four outcomes: CAPTCHA rejection rate, successful session verification rate, refresh failures, and the time from a global revoke request to the last accepted session. Break them down by browser and region. A low signup-bot rate with frequent forced logouts is not a win for an admin product.

I also test the negative paths first: an expired access credential must not become valid just because user lookup succeeds; a current-device logout must leave another explicitly authorized device alone; a global revoke must end every session associated with that user. Those tests are more revealing than a happy-path signup demo.

Authentication boundaries are a risk decision. Choose the CAPTCHA signal that matches your abuse model, then compose the smallest set of lifecycle interfaces that preserves account continuity and auditability.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://developers.google.com/recaptcha/enterprise/docs
- https://developers.cloudflare.com/turnstile/
- https://docs.hcaptcha.com/
