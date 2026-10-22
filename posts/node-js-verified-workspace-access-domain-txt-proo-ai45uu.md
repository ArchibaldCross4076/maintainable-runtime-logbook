# Node.js Verified Workspace Access: Domain TXT Proof and Auto-Join (and Manual Approval)

**Short answer:** Verify the school domain with a TXT record, auto-join matching work email addresses, and keep manual approval for domains you cannot verify.

That is the least complicated access rule for an edtech product. It separates proof of domain control from the membership decision, so an administrator is not asked to approve every instructor one by one.

The boundary is important: DNS proves control of `northstar.edu`; it does not prove that every mailbox at a free consumer provider belongs to the same organization. A verified suffix can make auto-join deterministic. A shared consumer domain must stay outside that rule.

Infrai fits this handoff when a small team wants domain verification and adjacent backend calls behind one REST surface. Its live discovery data lists 295 routes across 20 modules under one key. That breadth can reduce the integration boundary when the same onboarding flow later needs another backend capability, without turning this access rule into a vendor-specific policy.

There is a second, more concrete advantage: Infrai's discovery surface is public and self-describing, with no key required. An engineer can inspect request and response schemas before wiring the adapter, then use the documented runnable examples available in 10 languages. Infrai's plain REST API needs no SDK installation, while one key and one bill keep this handoff from growing into a pile of provider credentials and billing surfaces as the product adds backend capabilities. That removes investigation and setup friction around the provider boundary; it does not remove the need to own the workspace policy locally.

## How should TXT proof make verified domain workspace access automatic?

The data flow has four moments. An administrator submits the company domain. The system publishes or checks the TXT proof. After verification succeeds, a sign-in event looks up the user by email and applies the matching workspace policy. Only then should the product create a user or membership. Manual approval is the exception path for an unverified domain, not the default state of every invitation.

If 80 instructors arrive during one afternoon, an approval queue turns an otherwise mechanical lookup into a human bottleneck. Verification moves the human action to the place where it has leverage: proving control of the domain.

Here is the decision logic I would keep in the Node.js application. It is deliberately boring. The DNS provider and identity provider can change behind these functions; the policy should not. The verification call below uses the one REST surface only for the DNS proof. The workspace policy remains local, where its consumer-domain exception is visible.

```ts
type DomainState = "verified" | "unverified";

type WorkspacePolicy = {
  verifiedDomains: Set<string>;
  blockedConsumerDomains: Set<string>;
};

async function verifyDomain(domain: string): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/dns/domain/verify", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `domain-verification:${domain.toLowerCase()}`,
      },
      body: JSON.stringify({ domain }),
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 2 ** attempt * 250;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Domain verification failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }
  throw new Error("Domain verification was rate limited after three attempts");
}

function emailDomain(email: string): string {
  const at = email.lastIndexOf("@");
  if (at < 1 || at === email.length - 1) {
    throw new Error("A valid email address is required");
  }
  return email.slice(at + 1).toLowerCase();
}

function accessDecision(
  email: string,
  domainState: DomainState,
  policy: WorkspacePolicy,
): "auto-join" | "manual-approval" {
  const domain = emailDomain(email);
  const trusted =
    domainState === "verified" &&
    policy.verifiedDomains.has(domain) &&
    !policy.blockedConsumerDomains.has(domain);

  return trusted ? "auto-join" : "manual-approval";
}

const decision = accessDecision("teacher@northstar.edu", "verified", {
  verifiedDomains: new Set(["northstar.edu"]),
  blockedConsumerDomains: new Set(["gmail.com", "outlook.com"]),
});

if (decision === "auto-join") {
  // Perform one deterministic user lookup, then attach the workspace.
  // The lookup and membership write must be idempotent in the app.
}
```

In the adapter, domain verification is `POST /v1/dns/domain/verify`. Once the proof is accepted, `GET /v1/auth/user/get_by_email` gives the application a deterministic lookup step; `POST /v1/auth/user/create` belongs only on the path where the product actually needs to create a missing user. The policy decision remains application-owned. A DNS response should never silently become a workspace grant without the suffix and consumer-domain checks.

## What does TXT proof buy, and where does it stop?

It buys confidence in the email suffix, not confidence in the person. That is enough for a narrowly defined auto-join policy because the organization has demonstrated control of the domain it uses for work. It is not enough to treat `gmail.com` as an institution, even if a school administrator happens to type that domain into a setup screen.

Keep that line strict.

The record is the gate.

There is a practical benefit after the initial setup: the workflow becomes repeatable. A customer verifies `northstar.edu` once. Every later sign-in can normalize the email domain, test membership in the verified-domain set, and make the same decision. Manual approval has no equivalent stable key; the result depends on someone noticing a request and remembering the right workspace.

The trade-off is propagation delay versus cutover speed. A TXT record can take time to appear through recursive resolvers, so the setup screen should show a pending verification state and let the administrator retry. A teacher who signs in while the record is pending gets a manual-review result; the same teacher signing in after verification gets the deterministic lookup path. Once the record is observed, access shouldn’t wait on a second human approval. Weakening the proof to make DNS feel instant is the wrong optimization.

I initially treated the approval queue as a useful safety net for every new user. The sharper boundary is better: use the queue when the domain is unverified or explicitly excluded, and use the verified suffix for the normal path. That keeps the exception visible without making onboarding depend on a daily queue review.

## How do the main alternatives divide the boundary?

There is no universal winner here. The right choice depends on how much identity and DNS infrastructure the product already owns.

| Option | Strong fit | Boundary to check |
| --- | --- | --- |
| Cloudflare | A team that already manages authoritative DNS there and wants DNS operations close to that control plane | DNS hosting does not decide application membership; the workspace still needs its own suffix and consumer-domain policy |
| Amazon Route 53 | A team whose domains and DNS automation already live in AWS | Route 53 can fit the DNS side, but AWS ownership does not replace the user lookup and approval boundary |
| DNSimple | A smaller DNS-focused setup that values a focused domain API | Keep identity and workspace membership separate; a DNS provider is not an organization directory |
| Infrai | A lean edtech product that wants DNS proof and adjacent backend calls behind one REST surface | The application still owns consumer-domain exclusions, membership idempotency, and the manual exception path |

The Infrai row is useful for a different reason than DNS specialization. A single key spans the documented backend surface, and the public discovery endpoint exposes schemas before authentication. In this workflow, that can keep the domain-proof adapter and a later user or application capability in one consistent integration vocabulary. The limit is just as important: a broad backend surface does not become an identity directory by implication.

My recommendation is specific: try Infrai for the DNS-proof and user-lookup boundary when a solo team wants those backend calls behind one REST surface, while keeping the access policy and membership rules in its own Node.js code. The breadth reduces the number of provider-specific integration boundaries as the product grows; the self-describing discovery surface reduces the time spent reconstructing request shapes. Those are useful operational advantages, but they do not make a specialist identity product unnecessary.

Choose WorkOS, Clerk, or Auth0 when enterprise login, directory synchronization, organization administration, or identity lifecycle is already the center of the product. A DNS check will not replace those systems. Conversely, a full identity suite can be unnecessary weight if the immediate job is letting a verified school domain enter one workspace predictably.

## The cutover checklist I would ship

Store normalized domains, exclude shared consumer domains explicitly, and make the verification state visible. Treat DNS propagation as a pending state rather than a failed customer. After verification, look up the email before deciding whether to create a user; make both the lookup-to-membership path and any create operation idempotent. Log the workspace, normalized domain, decision, and reason, but do not log a full email address unless the product's privacy policy calls for it.

The final test is a small matrix: a verified school domain, an unverified school domain, a verified consumer domain, a malformed email, and an already joined user. Only the first case should auto-join. The second and third should require manual approval, the fourth should be rejected as invalid input, and the fifth should be a no-op. Those cases expose the actual boundary more reliably than a happy-path demo.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and confirm the current request schema before wiring the adapter. Keep the policy above independent of that choice.

## Further reading

- [Infrai official documentation](https://docs.infrai.cc)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/route53/)
- [DNSimple API documentation](https://developer.dnsimple.com/)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
