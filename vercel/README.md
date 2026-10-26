# SomeSandbox: Vercel Sandbox egress OIDC tokens attest "a Vercel sandbox" — not the receiver's tenant

## Abstract

The Vercel Sandbox network policy forwards matching egress requests to a
developer-chosen receiver URL and attaches a signed OIDC token
(`vercel-sandbox-oidc-token`) so the receiver can verify the request is
legitimate. The documented verifier (`defineSandboxProxy`) checks issuer,
signature, audience, and expiry — and then hands the token identity to the
handler as `meta {teamId, projectId, sandboxId}`. Two design facts break
tenant isolation:

1. `oidc.vercel.com/.well-known/jwks` is byte-identical to
   `oidc.vercel.com/<ANY-TEAM>/.well-known/jwks`. One global signing key
   serves every per-team issuer path, and the SDK fetches the JWKS from the
   token's own `iss` — so a token minted for any team verifies on any
   receiver.
2. Any tenant can set a forwarding rule naming any public receiver URL.
   Nothing binds the rule target to the sandbox owner's team or project; the
   receiver URL is an attacker-chosen string and only the scheme is validated.

Result: any customer's sandbox produces requests that pass
`defineSandboxProxy`'s complete cryptographic verification against any other
customer's receiver — presenting an arbitrary team/project/sandbox identity.
It proves only "some Vercel sandbox".

## Reproduction (two accounts, both owned)

Receiver side (prod account): a sandbox with a public route and an HTTP
server that passes requests through the documented `defineSandboxProxy`
example and logs `meta` verbatim.

Step 1 — attacker sets a forwarding rule naming the victim's receiver
(HTTP 200):

```http
POST /v2/sandboxes/sessions/<attacker-sandbox-session>/network-policy
{"allow":{"example.com":[{"forwardURL":"https://<victim-receiver>.vercel.run/p"}]}}
```

Step 2 — attacker triggers matching egress from inside their sandbox
(HTTP 200):

```json
{"command":"curl","args":["-sS","-m","15","https://example.com/"],"wait":true}
```

The egress firewall mints an RS256 JWT (`iss
https://oidc.vercel.com/<attacker-team>`, `sub team:project:sandbox`, `aud`
the receiver URL, 24h expiry) and forwards the request with the token
attached.

Step 3 — the victim receiver's verification PASSES and the handler executes.
Receiver-side log, prod account:

```text
REQ url=/p/  aud="https://<victim-receiver>.vercel.run/p"
ACCEPTED meta={"teamId":"<ATTACKER_TEAM>","projectId":"<ATTACKER_PROJECT>",
"sandboxId":"<ATTACKER_SANDBOX>","sandboxName":"sandbox-b"}
```

`teamId` in the victim's `meta` is the attacker account.

## The enforcement matrix

| receiver implementation | foreign-team token | mechanism |
|---|---|---|
| SDK `defineSandboxProxy` (documented default) | **ACCEPTED** (live, twice) | JWKS fetched from the token's own `iss` path; the global key serves every path; `aud` comes from the receiver URL, which the attacker chose, so it auto-matches |
| issuer-hostname + signature, no `aud` check | **ACCEPTED** (local jose PoC) | global key verifies from any per-team `iss` path |
| exact per-team `iss` + per-team JWKS + own `aud` | REJECTED | the only binding discipline; the receiver must hand-pin its own issuer path — the SDK does not |

## The token is a plain bearer credential

Each link below is reproduced separately; the impact is their composition.

1. Any tenant's sandbox sets a forwarding rule naming any receiver. The
   firewall mints a 24h JWT for that sandbox's team and attaches it.
2. The token materializes at the receiver as a bearer artifact — including a
   receiver running inside a Vercel sandbox, where it is file-readable by any
   read primitive. Any receiver-side logger materializes it the same way.
3. A captured token replays from anywhere for 24h: a captured token POSTed
   directly from an external machine with handcrafted firewall-shape headers
   passed full SDK verification (HTTP 200, handler executed with the captured
   identity). No channel binding, no rotation inside the window.

Receivers cannot tenant-scope what they accept: the shipped
`defineSandboxProxy` accepts any team's token, a naive no-`aud` verifier
accepts everything, and only a hand-rolled exact per-team issuer pin rejects.

## Impact

Any customer's sandbox can send requests to any customer's
`defineSandboxProxy` receiver that pass the SDK's complete cryptographic
verification, presenting an arbitrary team/project/sandbox identity.
Receivers that authorize on verification passing — which the documented
example does — accept cross-tenant traffic. Real deployments use this
pattern to let customer sandboxes call internal APIs with per-team
authorization. The verification boundary was understood to prove the sandbox
belongs to the receiver's team. It proves only "some Vercel sandbox".

The per-team issuer URL plus a single shared JWKS implies tenant isolation to
SDK consumers while providing none.

## Documentation gap

Vercel's own documentation frames the token as an authentication guarantee:
the firewall docs call `vercel-sandbox-oidc-token` "A Vercel-issued OIDC
token that the proxy can use to authenticate the request", list `team_id` as
"[t]he ID of the Vercel team the sandbox belongs to", instruct the proxy to
"verify the token signature, issuer, expiry, and that `aud` exactly matches
its expected forwardURL", and recommend `defineSandboxProxy` "with automatic
OIDC validation". The changelog states the token lets the proxy
"authenticate the request and identify the source team / project / sandbox".
Following the documented recipe accepts any tenant's token.

## Remediation

Per-team signing keys (JWKS per issuer path), bind the forwarding rule
target to the sandbox owner's team, and/or bind the token to a channel
(rotation, receiver-nonce, or mTLS). Until then, receivers must hand-pin
their exact per-team issuer path — the SDK's default accepts everything.

## Credit

nichoc0
