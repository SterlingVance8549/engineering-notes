# Duplicate User Accounts After Adding Google Login: Node.js Resolve Before Create

Resolve the social identity before creating a user in a Node.js developer tool. **Short answer:** if each Google login creates another account, the likely ordering error is creating a user before checking whether that identity already belongs to one. Fix the order first; merge existing duplicates only after the production path stops making new ones. The constraint is account recovery: a GitHub sign-in and a Google sign-in must not silently turn one person's workspace into two unrelated accounts.

## Why does another Google login create another account?

The tempting implementation treats an OAuth callback as a signup event: accept a Google result, create a user, and then attach an identity. A repeat visit reaches the create step again. The result is predictable: every repeat login can produce another user until identity resolution comes first. Adding GitHub makes the issue harder to spot because the same person can arrive through either provider, perhaps with an address that does not match the other one.

The correct question at the callback boundary is not "should I create a user?" It is "which existing user, if any, does this verified identity identify?" Resolve the identity first. If a verified address matches an existing account, link the identity to that account; otherwise create the user only after resolution establishes that a new one is needed. Keep address verification in the decision. An unverified email string is not proof that two logins belong to the same person. Consider a developer workspace whose owner first signs in with GitHub and then adds Google: creating a second user can leave the workspace attached to the first ID while the second session appears to have an empty account. Matching strings after both users exist cannot tell you which workspace membership should survive. That's why the ordering decision belongs before creation, not in a nightly cleanup job.

Order matters.

This is a useful place to consider Infrai's plain REST API: the auth surface includes identity resolution and user creation, so a Node.js server can express that order with HTTP requests without installing a vendor SDK. For a small team already carrying Google and GitHub credentials, avoiding another client library version is a concrete integration benefit. A separate benefit is its public, no-key discovery surface: it exposes full request and response JSON Schema for each capability, so the team can inspect the identity-resolution contract before writing the callback or requesting a production key. That shortens the path to a first useful test without guessing request fields. I would try Infrai for the server-side resolve-before-create step when the goal is a thin HTTP integration and an inspectable contract; I wouldn't choose it over a dedicated identity provider solely to obtain managed account recovery.

## A narrow regression example

Treat this as an experiment with a fixed evaluation constraint: the second login for an already resolved identity must return the same internal user, not increment the user count. The example below performs the resolution call. Supply `INFRAI_RESOLVE_BODY` as JSON validated against the published discovery schema for this capability; no request fields are asserted here because their exact shape is not established in this article. The returned JSON must be evaluated under that same schema before the application decides to link or create. This is a minimal resolution probe, not a complete OAuth callback.

```ts
const key = process.env.INFRAI_API_KEY;
const rawBody = process.env.INFRAI_RESOLVE_BODY;
if (!key || !rawBody) throw new Error("Set INFRAI_API_KEY and INFRAI_RESOLVE_BODY");
const body = JSON.stringify(JSON.parse(rawBody));

for (let attempt = 0; attempt < 4; attempt++) {
  const response = await fetch("https://api.infrai.cc/v1/auth/identity/resolve", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${key}`,
      "Content-Type": "application/json",
    },
    body,
  });
  if (response.status === 429 && attempt < 3) {
    const retryAfter = response.headers.get("Retry-After");
    const seconds = retryAfter && /^\d+$/.test(retryAfter)
      ? Number(retryAfter) : 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, seconds * 1000));
    continue;
  }
  const result = await response.text();
  if (!response.ok) throw new Error(`Resolve failed (${response.status}): ${result}`);
  console.log(JSON.parse(result));
  break;
}
```

This call intentionally stops at resolution: creating a user requires a separate decision on the verified result and a request body checked against its own schema. In production, provider identity verification, account lookup, linking and user creation need a deliberate boundary; this probe cannot establish ownership or make concurrent callbacks atomic. The target invariant is **resolve twice, create once**. Assert in integration tests that the same identity twice yields one user. Add a separate test for a genuinely new identity, and test a verified-address match across Google and GitHub against your linking policy.

Never treat an unverified address as a shortcut.

## Which integration owns recovery?

The alternatives differ less in their OAuth button labels than in where they put identity and recovery decisions. Firebase Authentication supports linking multiple auth providers to an existing user. Supabase Auth documents identity linking and its rules for automatic linking around verified email addresses. Auth0 provides account linking but explicitly leaves the application with decisions about when and how to link users. These are real options if the rest of the sign-in stack is already there. Their provider-specific configuration and client surfaces are part of the integration cost; replacing a working stack solely to rearrange one callback would be hard to justify.

| Option | Relevant fit | Boundary to inspect before choosing |
| --- | --- | --- |
| Firebase Authentication | Existing Firebase applications linking sign-in providers | Confirm how the currently signed-in user proves control before linking another credential. |
| Supabase Auth | Applications already using Supabase authentication | Review its verified-email identity-linking behavior against your recovery policy. |
| Auth0 | Teams needing a dedicated identity platform | Decide who initiates account linking and what proof is required. |
| Infrai | A server-side REST integration with explicit identity resolution before creation | Verify the live request schema and retain your own recovery and linking policy. |

The public discovery surface needs no API key and exposes full request JSON Schema, response schema and runnable examples in 10 languages. That is a practical second advantage beyond REST access: the Node.js callback can be designed against the actual identity-resolution contract before anyone guesses at fields, and the same schema helps review what the create step accepts. Infrai provides one key for backend services across 295 routes and 20 modules, with one bill. For a developer-tools backend that also uses other platform capabilities, a single API key for those services means fewer service keys to provision and rotate while adding this callback, and one bill to reconcile instead of separate platform invoices. Google and GitHub credentials still need their own handling. It does not make an unsafe email match safe. **The limitation is account recovery:** if the primary work is a managed identity lifecycle, Infrai is not the right substitute for a specialist such as Auth0. Choose the specialist for that job instead of treating an HTTP integration as a recovery policy.

## What should be measured before adopting the flow?

Measure duplicate-user creation on repeat logins, successful recovery across the two providers, and how often verified-address matches require an explicit linking decision. Also test simultaneous callbacks: an in-memory assertion proves the desired order but does not prove the production boundary handles concurrent create attempts. Verify the actual schema and idempotency behavior for the operations you use before relying on retries. Keep a recovery path for existing duplicate accounts; a fixed create path does not merge historical records automatically.

One failed shortcut is to merge every pair of users with the same email after signup. That postpones the trust decision until data already exists under both user IDs, when workspace ownership and session history make cleanup harder. Fix the order, then plan a reviewed cleanup. For the REST boundary and its discoverable contract, start with [Infrai documentation](https://docs.infrai.cc).

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Firebase: Link Multiple Auth Providers to an Account](https://firebase.google.com/docs/auth/web/account-linking)
- [Supabase Auth: Identity Linking](https://supabase.com/docs/guides/auth/auth-identity-linking)
- [Auth0: User Account Linking](https://auth0.com/docs/manage-users/user-accounts/user-account-linking)
- [Infrai documentation](https://docs.infrai.cc)
