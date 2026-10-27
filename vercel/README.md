# SomeSandbox: Vercel Sandbox egress OIDC tokens prove "a Vercel sandbox" instead of the receiver's tenant

Found by nichoc0.

## Abstract

The Vercel Sandbox network policy forwards matching egress requests to a
receiver URL chosen by the developer. The egress firewall attaches a signed
OIDC token (`vercel-sandbox-oidc-token`) so the receiver can verify the
request. The documented verifier (`defineSandboxProxy`) checks issuer,
signature, audience, and expiry, then hands the token identity to the
handler as `meta` with `teamId`, `projectId`, and `sandboxId`.

Two design facts break tenant isolation.

First, `oidc.vercel.com/.well-known/jwks` is byte-identical to
`oidc.vercel.com/<ANY-TEAM>/.well-known/jwks`. One global signing key serves
every per-team issuer path, and the SDK fetches the JWKS from the token's
own `iss`. A token minted for any team verifies on any receiver.

Second, any tenant can set a forwarding rule naming any public receiver URL.
Nothing binds the rule target to the sandbox owner's team or project. The
receiver URL is an attacker-chosen string and only the scheme is checked.

The result: any customer's sandbox produces requests that pass the full
cryptographic verification of `defineSandboxProxy` against any other
customer's receiver, presenting an arbitrary team, project, and sandbox
identity. The token proves that a Vercel sandbox exists. It does not prove
whose.

## Reproduction

Two accounts, both owned. The receiver runs on a prod account: a sandbox
with a public route and an HTTP server that forwards requests through the
documented `defineSandboxProxy` example and logs `meta` verbatim.

Step 1. The attacker sets a forwarding rule naming the victim's receiver.
HTTP 200:

```http
POST /v2/sandboxes/sessions/<attacker-sandbox-session>/network-policy
{"allow":{"example.com":[{"forwardURL":"https://<victim-receiver>.vercel.run/p"}]}}
```

Step 2. The attacker triggers matching egress from inside their sandbox.
HTTP 200:

```json
{"command":"curl","args":["-sS","-m","15","https://example.com/"],"wait":true}
```

The firewall mints an RS256 JWT with `iss
https://oidc.vercel.com/<attacker-team>`, `sub team:project:sandbox`, `aud`
set to the receiver URL, and a 24-hour expiry. The request is forwarded with
the token attached.

Step 3. The victim receiver's verification passes and the handler runs.
Receiver-side log on the prod account:

```text
REQ url=/p/  aud="https://<victim-receiver>.vercel.run/p"
ACCEPTED meta={"teamId":"<ATTACKER_TEAM>","projectId":"<ATTACKER_PROJECT>",
"sandboxId":"<ATTACKER_SANDBOX>","sandboxName":"sandbox-b"}
```

The `teamId` in the victim's meta is the attacker account.

## Enforcement matrix

| receiver implementation | foreign-team token | mechanism |
|---|---|---|
| SDK `defineSandboxProxy` (documented default) | ACCEPTED, live twice | JWKS fetched from the token's own `iss` path; the global key serves every path; `aud` is the receiver URL, which the attacker chose, so it auto-matches |
| issuer-hostname plus signature, no `aud` check | ACCEPTED, local jose PoC | the global key verifies from any per-team `iss` path |
| exact per-team `iss` plus per-team JWKS plus own `aud` | REJECTED | the only binding discipline. The receiver must hand-pin its own issuer path. The SDK does not |

## The token is a plain bearer credential

Each item below reproduces separately. The impact is the composition.

1. Any tenant's sandbox sets a forwarding rule naming any receiver. The
   firewall mints a 24-hour JWT for that sandbox's team and attaches it.
2. The token materializes at the receiver as a bearer artifact. A receiver
   running inside a Vercel sandbox holds it as a file readable by any read
   primitive. Receiver-side loggers materialize it the same way.
3. A captured token replays from anywhere for 24 hours. A captured token
   POSTed directly from an external machine with handcrafted firewall-shape
   headers passed full SDK verification, returned HTTP 200, and executed the
   handler with the captured identity. No channel binding. No rotation
   inside the window.

Receivers cannot tenant-scope what they accept. The shipped
`defineSandboxProxy` accepts any team's token. A verifier without an `aud`
check accepts everything. Only a hand-rolled exact per-team issuer pin
rejects.

## Impact

Any customer's sandbox can send requests to any customer's
`defineSandboxProxy` receiver that pass the complete cryptographic
verification of the SDK, presenting an arbitrary team, project, and sandbox
identity. Receivers that authorize on verification passing accept
cross-tenant traffic, and the documented example does exactly that. Real
deployments use this pattern to let customer sandboxes call internal APIs
with per-team authorization. The verification boundary reads as proof that
the sandbox belongs to the receiver's team. It proves only the existence of
a Vercel sandbox.

The per-team issuer URL plus a single shared JWKS implies tenant isolation
to SDK consumers while providing none.

## Documentation gap

The firewall docs call `vercel-sandbox-oidc-token` "A Vercel-issued OIDC
token that the proxy can use to authenticate the request". They list
`team_id` as "the ID of the Vercel team the sandbox belongs to". They
instruct the proxy to "verify the token signature, issuer, expiry, and that
`aud` exactly matches its expected forwardURL" as the security check. The
changelog states the token lets the proxy "authenticate the request and
identify the source team / project / sandbox". Following the documented
recipe accepts any tenant's token.

## Fix

Per-team signing keys so each JWKS path serves only its own team. Bind the
forwarding rule target to the sandbox owner's team. Bind the token to a
channel: rotation, receiver nonce, or mTLS. Until then, receivers must
hand-pin their exact per-team issuer path. The SDK default accepts
everything.

## Credit

nichoc0
