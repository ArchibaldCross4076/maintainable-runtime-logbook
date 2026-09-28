# Domain Verification vs Email Confirmation: How to Control Workspace Joining

Short answer: use email confirmation to prove that a person can receive mail at one address. Use a DNS challenge when the decision grants control over an organisation's workspace or authorizes a marketplace sender domain. The least complex defensible design keeps those proofs separate, stores the intended SPF, DKIM, and DMARC state, and continuously compares that intent with public DNS. A clicked link is not evidence that someone controls the domain.

For a marketplace, that distinction affects more than sign-up. A seller may confirm 'ops@northwind.example' while a different team controls '_dmarc.northwind.example', the DKIM selector, or the infrastructure that publishes SPF. If mailbox confirmation automatically admits every address ending in the same string, one accessible inbox can become an organisation-wide privilege. If a DNS claim is treated as permanent, a later record deletion can leave authorization detached from the evidence that created it.

Keep the proofs separate.

## Should workspace joining use domain verification or email confirmation?

Email confirmation answers, "Can this claimant receive a message at this address now?" It is useful for activating an individual account, recovering it, and checking a notification route. It says nothing by itself about who administers sibling mailboxes, the DNS zone, or the marketplace's sending policy.

A DNS challenge answers a narrower control question: "Can this claimant arrange publication beneath the domain?" The service generates a high-entropy, single-purpose token and asks for it at a designated TXT owner name. Matching that exact value is evidence of effective publication control at verification time. It still does not prove legal ownership, and delegated subzones mean the proof applies to the verified DNS name, not automatically to every parent or child.

That scope is the first design decision. Normalize domain names to lowercase and remove one trailing dot before comparison, but do not infer an organisational boundary from a string suffix. 'store.example' and 'fraudstore.example' plainly differ; 'team.store.example' may also have a separately delegated administrator. For auto-join, bind a verified domain claim to one workspace, record the exact name and proof time, and require an explicit policy that says which account addresses qualify. Keep individual email confirmation as a second gate.

**Mailbox access establishes identity reachability; DNS publication establishes a domain-control signal. Neither substitutes for the other.**

## Implement the published-state check first

The data flow is small. An administrator declares the expected records, DNS publishes them, and a worker resolves each owner name through the normal public resolver path. The worker compares semantic values, stores the observation and timestamp, then changes authorization only through a deliberate policy. This makes drift visible without coupling every sign-in to DNS latency or cache state.

Here is a runnable checker using Node's promise-based DNS API. Save it as 'check-mail-dns.ts' and run it with a TypeScript runner in an environment that can resolve public DNS. Replace the example names and values with configuration from your own workspace record; the reserved '.example' names below are intentionally non-production placeholders.

```ts
import { resolveTxt } from "node:dns/promises";

type ExpectedRecord = {
  purpose: "domain-claim" | "spf" | "dkim" | "dmarc";
  name: string;
  expected: string;
};

const records: ExpectedRecord[] = [
  {
    purpose: "domain-claim",
    name: "_workspace-claim.seller.example",
    expected: "workspace-claim=replace-with-generated-token",
  },
  {
    purpose: "spf",
    name: "seller.example",
    expected: "v=spf1 include:mail.example -all",
  },
  {
    purpose: "dkim",
    name: "market1._domainkey.seller.example",
    expected: "v=DKIM1; k=rsa; p=replace-with-public-key",
  },
  {
    purpose: "dmarc",
    name: "_dmarc.seller.example",
    expected: "v=DMARC1; p=none; rua=mailto:dmarc@seller.example",
  },
];

function normalizeTxt(value: string): string {
  return value.trim().replace(/\s+/g, " ");
}

async function observe(record: ExpectedRecord) {
  try {
    const answers = await resolveTxt(record.name);
    const published = answers.map((parts) => parts.join(""));
    const matches = published.some(
      (value) => normalizeTxt(value) === normalizeTxt(record.expected),
    );
    return { ...record, matches, published, error: null };
  } catch (error) {
    const message = error instanceof Error ? error.message : String(error);
    return { ...record, matches: false, published: [], error: message };
  }
}

const observations = await Promise.all(records.map(observe));
console.log(JSON.stringify({ checkedAt: new Date().toISOString(), observations }, null, 2));
process.exitCode = observations.every((item) => item.matches) ? 0 : 1;
```

TXT character-strings can be returned as multiple chunks; joining the chunks in each answer is therefore important. Do not concatenate separate TXT answers into one policy. The strict whole-value comparison is appropriate for a claim token and for detecting drift from a stored desired value, but it is not a complete SPF, DKIM, or DMARC validator. Full validation requires parsing each protocol's grammar and, for mail authentication, evaluating a real message and its identifiers.

The sample deliberately makes a failed lookup an observation rather than an exception that aborts the batch. DNS absence, timeout, and mismatch are operationally different, so production storage should retain the resolver error code as structured data. Retry transient resolution failures with bounded backoff; do not silently convert one timeout into revocation.

## Publish authentication without confusing the layers

SPF, DKIM, and DMARC solve related but distinct problems. SPF publishes which hosts may use a domain in the SMTP envelope identity. DKIM attaches a cryptographic signature and identifies the signing domain and selector. DMARC evaluates alignment between the visible From domain and an authenticated SPF or DKIM domain, then applies the published policy and reporting instructions. RFC 7489 also specifies the '_dmarc' owner-name convention and the 'p' policy tag.

For a marketplace, write desired state per seller domain before asking anyone to edit DNS. The record should include the exact domain claim, the SPF policy fragment expected from the sending architecture, active DKIM selectors and public keys, and the DMARC policy. Treat those as versioned intent. A DNS observation is evidence about that intent, not the source of truth.

Roll changes in dependency order. Publish and observe a new DKIM selector before signing with it; keep the previous selector published while messages signed with it may still be in transit or retried. For SPF, respect RFC 7208's limit of 10 terms that cause DNS queries during evaluation. Flattening every address into one generated record can trade lookup depth for a brittle maintenance job, so measure the actual sending graph and keep ownership clear.

Ten DNS-querying terms is the ceiling.

DMARC deserves a staged decision. A monitoring policy can collect aggregate feedback before enforcement, while 'quarantine' and 'reject' request receiver handling for messages that fail DMARC. Reports may expose information about mail flows, so route them to an appropriately controlled mailbox and define retention deliberately. A policy change should follow observed alignment, not a deadline picked for a launch announcement.

No click path can verify those conditions.

## Design revocation and drift as product behavior

The dangerous shortcut is a boolean such as 'domainVerified: true'. It loses which token was observed, at what owner name, when it was checked, and what should happen after drift. Use an append-only observation history plus a current claim state. A compact state model is 'pending', 'verified', 'drifting', and 'revoked', with timestamps and reason codes.

A mismatch should not instantly eject every member. DNS caches, operator mistakes, and transient resolver failures exist, and a marketplace workspace may be handling active orders. Instead, separate existing sessions from privilege expansion. During a defined grace window, stop new domain-based auto-joins and sensitive domain-admin changes, notify already verified administrators through independent channels, and continue checks from more than one resolver vantage point. Revoke the claim when the policy's evidence threshold is met. The exact interval is a risk choice; inventing one universal number would hide the trade-off.

This is where cost discipline helps. Resolution is cheap compared with model inference or outbound mail, but an unbounded per-request lookup creates needless latency and turns DNS availability into login availability. A scheduled worker, cached observations, and event-driven rechecks after configuration changes keep the hot path small. Increase check frequency while a claim is pending or drifting; stable records can be checked less aggressively according to the workspace's risk tier.

Log enough to explain a transition: workspace ID, normalized owner name, expected-value hash, observed-value hash, resolver result, check time, and state change. Do not put the raw claim token in general application logs. Expose the last successful observation and current mismatch to administrators so they can distinguish propagation from a typo.

The comparison rule must also resist duplicate claims. Before accepting a token, perform an atomic uniqueness check for the normalized domain scope. Transfers between workspaces need an explicit handoff or dispute process; whichever request happens to poll DNS last should not win.

Drift is normal. Unexplained authority is not.

## What should block workspace joining?

Use two independent predicates. The account's email must be confirmed, and the address domain must match an active claim whose workspace policy permits auto-join. For privileged roles, add an invitation or administrator approval instead of deriving authority from the address alone. Contractors, aliases, plus-addressing, and acquired domains make domain suffixes a weak role system.

A practical decision table keeps the edge cases visible:

| Email proof | DNS claim | Mail records | Result |
| --- | --- | --- | --- |
| Confirmed | Active for this workspace | At intended state | Allow policy-eligible membership; keep roles minimal |
| Confirmed | Missing or belongs elsewhere | Any state | Require invitation or an explicit claim process |
| Unconfirmed | Active | Any state | Do not auto-join; mailbox reachability is unproven |
| Confirmed | Drifting | Authentication drift detected | Pause expansion and alert; apply the documented grace policy |

Notice that mail-authentication health and workspace ownership are adjacent controls, not one combined proof. A seller can retain its claim TXT record while accidentally removing DMARC. That should raise a delivery-risk incident, not rewrite historical membership. Conversely, healthy SPF, DKIM, and DMARC records do not grant your service permission to attach users to a workspace; many domains publish them without knowing your application exists.

Before shipping, walk the entire lifecycle in prose and in tests: a new claim, an exact TXT match, duplicate contention, a delegated subdomain, a confirmed and unconfirmed mailbox, record removal, resolver timeout, token rotation, workspace transfer, and claim revocation. Verify the public answer rather than a control panel's saved form. Then send representative mail through the real path and inspect authentication results, because syntactically present records are not proof of alignment.

**Grant membership from explicit policy backed by current evidence, and keep delivery authentication as separately monitored desired state.** That boundary prevents a convenient inbox check from becoming organisational control, while making DNS drift something operators can detect and resolve before marketplace mail quietly loses trust.

## Further reading

- RFC 1035, Domain Names: Implementation and Specification: https://datatracker.ietf.org/doc/html/rfc1035
- RFC 6376, DomainKeys Identified Mail Signatures: https://datatracker.ietf.org/doc/html/rfc6376
- RFC 7208, Sender Policy Framework: https://datatracker.ietf.org/doc/html/rfc7208
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
- RFC 2606, Reserved Top Level DNS Names: https://datatracker.ietf.org/doc/html/rfc2606
- Node.js DNS documentation: https://nodejs.org/api/dns.html
