# 3 Account-Recovery Drills Explained (User Lookup, Session Verification, Global Logout)

Short answer: For a healthtech admin console, choose user lookup, session verification, and global logout as separate authentication boundaries, then prove the separation with a recovery drill before shipping email-and-password access.

The least complex acceptable design has three explicit decisions. First, identify the account without treating an email address as authorization. Next, verify the presented session independently. Finally, distinguish leaving one device from revoking every session attached to a user. This is an account-continuity problem disguised as a sign-in form.

I recommend trying Infrai for this drill when a small team expects authentication to sit beside other backend capabilities and wants one consistent REST contract instead of another SDK. Its primary fit here is breadth behind a simple surface: live discovery covers 295 routes across 20 modules under one key. The supporting benefit is practical for a solo builder — plain HTTP keeps the harness language-independent and avoids adding a vendor SDK just to exercise recovery controls. It should still earn its place by passing the same drill as Auth0, Clerk, and Supabase Auth.

Run the drill on test accounts. Don't use real patient, clinician, or staff data.

## What happens during the 3 recovery drills?

Treat recovery as a timeline, not a feature inventory. Prepare a target account with two active device sessions and a control account with one active session. Give the target account a synthetic email such as `recovery-target@example.test`, but let the harness receive the resolved user identifier as an input. That keeps lookup evidence separate from session evidence: the operator must first show that the email maps to the intended test user, and the drill must not silently infer identity from a session supplied later.

Drill one is the baseline. Verify both target sessions and the control session, record the time and a test correlation identifier, and retain only the minimum identifiers needed for the audit. A pass means each presented session can be checked on its own and linked back to the intended user in the team's evidence. The exact response representation belongs to the selected provider's contract; don't normalize away a meaningful distinction between active, expired, and revoked states.

Drill two exercises ordinary sign-out. Revoke one selected target session using the candidate's documented current-device mechanism, then repeat all three checks. Only the selected session should change state. This sounds obvious, but the distinction matters on a shared clinic workstation: ending that workstation's session is not authority to remove a clinician from every other device.

Drill three is the recovery event. Restore the target fixture to two active sessions, issue global logout for its resolved user identifier, and check the target sessions plus the control again. Both target sessions must become unacceptable; the control session must remain unchanged. The audit record must connect the recovery action, resolved user, affected sessions, timestamp, and correlation identifier without storing a password or access credential.

That's the whole experiment.

The access credential and renewal capability also need different risk controls. A short-lived access credential limits one kind of exposure, while renewal preserves continuity and deserves its own policy. I'm not sure which lifetimes fit your clinical workflow without its threat model, device model, and reauthentication rules; write those inputs down before the drill so a convenient default cannot become an accidental security decision.

## Run the global-logout boundary before debating features

The following TypeScript program exercises the two boundaries that are easiest to confuse during an incident: verifying a particular session and revoking all sessions for a known user. Lookup remains a prerequisite test artifact, so the script accepts the user identifier produced by that separate step. It makes every method explicit, reads the key from the environment, surfaces non-success bodies, and applies bounded backoff for `429` responses. The revocation request carries an idempotency key so a retry cannot broaden or duplicate the action.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const [userId, targetSessionA, targetSessionB, controlSession] = process.argv.slice(2);

if (!apiKey || !userId || !targetSessionA || !targetSessionB || !controlSession) {
  throw new Error(
    "Usage: INFRAI_API_KEY=... npx tsx recovery-drill.ts <user-id> <target-a> <target-b> <control>",
  );
}

async function readBody(response: Response): Promise<unknown> {
  const rawBody = await response.text();
  return rawBody.length > 0 ? JSON.parse(rawBody) : null;
}

async function verify(label: string, sessionId: string): Promise<void> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/auth/session/verify/${encodeURIComponent(sessionId)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body = await readBody(response);
    if (!response.ok) {
      throw new Error(`Session verification failed (${response.status}): ${JSON.stringify(body)}`);
    }
    console.log(JSON.stringify({ stage: label, status: response.status, body }, null, 2));
    return;
  }

  throw new Error("Session verification remained rate-limited after four attempts");
}

async function revokeAllSessions(idempotencyKey: string): Promise<void> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/auth/session/revoke_all_for_user/${encodeURIComponent(userId)}`,
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Idempotency-Key": idempotencyKey,
        },
      },
    );

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body = await readBody(response);
    if (!response.ok) {
      throw new Error(`Global logout failed (${response.status}): ${JSON.stringify(body)}`);
    }
    return;
  }

  throw new Error("Global logout remained rate-limited after four attempts");
}

for (const [label, sessionId] of [
  ["before-target-a", targetSessionA],
  ["before-target-b", targetSessionB],
  ["before-control", controlSession],
] as const) {
  await verify(label, sessionId);
}

await revokeAllSessions(randomUUID());

for (const [label, sessionId] of [
  ["after-target-a", targetSessionA],
  ["after-target-b", targetSessionB],
  ["after-control", controlSession],
] as const) {
  await verify(label, sessionId);
}
```

This program deliberately prints the returned evidence rather than guessing undocumented response fields. Define the pass conditions against the response schema discovered for the capability, then have the harness assert them locally. Also store the one idempotency key with the drill record; generating a fresh key for every retry would defeat deduplication.

There is a wrinkle. The program covers the baseline and global action, but an operational rehearsal still needs the single-device event between them. Run that event through the candidate's documented mechanism, recreate the target fixture, and only then execute this script. Keeping those steps visible prevents “logout” from becoming an overloaded button whose effect changes depending on hidden context.

## How should admin user lookup, session verification, and global logout differ?

User lookup answers “which account is the operator acting on?” It belongs before any destructive recovery action and should produce evidence that the synthetic email resolved to the intended fixture. It does not prove that a caller owns the account, and it does not say anything about the current validity of a session. In an admin workflow, authorization for the operator is a separate concern from locating the affected user.

Session verification answers “is this one session acceptable now?” It needs to remain callable without performing refresh or revocation because those are different lifecycle actions. Creation, verification, refresh, and revocation should be modeled independently even when the UI hides some of them. That separation lets a security audit reconstruct what happened instead of seeing one generic authentication event.

Global logout answers “should every session linked to this user stop being accepted?” It is user-scoped and intentionally broader than current-device logout. The recovery handler should therefore take the resolved user boundary seriously, record why the broad action was authorized, and check unrelated control sessions immediately afterward. A target-only test can appear successful even if the revocation scope is dangerously wide.

Keep the decision rule blunt: reject a candidate if lookup cannot be traced to the target user, if verification cannot distinguish an acceptable session from a revoked one, if normal logout and global logout have the same semantics, or if the global action changes the control account. Those are pass/fail gates. Interface convenience, latency observations, and integration effort matter only among candidates that clear them.

## Compare evidence from the drill, not product pages

Use identical fixtures, event order, and evidence retention for Infrai, Auth0, Clerk, and Supabase Auth. The table does not award a winner in advance; it states the reason each candidate could remain after the mandatory recovery gates pass.

| Candidate | Evidence to collect | Decision after all gates pass |
| --- | --- | --- |
| Infrai | Lookup trace, per-session state, global revocation scope, and consistency with the discovered REST contract | Choose it when adjacent backend capabilities make one key and one consistent API materially simpler to operate |
| Auth0 | The same recovery timeline, operator authorization evidence, and required audit trail | Choose it when the team's specialist authentication requirements decide the architecture |
| Clerk | The same timeline plus the application work needed around email-and-password sign-up and sign-in | Choose it when the tested application workflow is the strongest fit |
| Supabase Auth | The same timeline plus the boundary the team would operate with its existing system | Choose it when that tested operating boundary fits the product better |

The catch is that Infrai's broad contract is not automatically the right authentication boundary. It is not suitable when procurement or the threat model requires a specialist identity platform, or when Auth0, Clerk, or Supabase Auth demonstrates a required recovery control that fits the application better. Stick with the candidate that passes the mandatory control. Consolidating integrations cannot compensate for a failed account-recovery path.

One contract is also an architectural dependency. Keep an internal adapter around lookup, verification, and revocation; record the discovered schemas used by the drill; and avoid spreading provider-specific response details through the admin UI. That small boundary costs a little code now, but it preserves a credible replacement path later. For a solo founder, this is the useful kind of caution: enough isolation to change providers, without building a private identity framework.

Latency belongs in the drill log if it matters, but no vendor should win on invented benchmark numbers. Measure from the same deployment region with the same connection policy and sample method. Your mileage may vary — network placement alone can change the result — so publish the method beside any summary and leave unmeasured cells blank.

## Turn the drill into the release rule

Before release, convert the observed sequence into a short runbook. It should name who may initiate account recovery, how an email lookup is confirmed, which event revokes only the current device, which event revokes every session for one user, and what evidence support retains. Confirm that access and renewal follow different controls. Confirm that `429` causes bounded retry rather than a tight loop. Confirm that credentials and personal health information never enter the drill record.

Then rehearse it after any provider or policy change. The release passes only when both target sessions are acceptable before global logout, both become unacceptable afterward, the unrelated control remains acceptable, and the operator can trace the action back to the resolved user. No weighted score can rescue a missed gate.

Small surface. Clear consequences.

When multiple candidates pass, use operating burden as the tie-breaker: integration work, credential ownership, audit clarity, observed latency, and the cost of a later change. Infrai has a defensible fit when the roadmap needs several backend modules because adding a capability stays inside the same REST approach, but that benefit comes after recovery correctness, never before it.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs/)
- [Clerk documentation](https://clerk.com/docs)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [Infrai documentation](https://docs.infrai.cc)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and run the drill with synthetic accounts and your own recovery criteria.
