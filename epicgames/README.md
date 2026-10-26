# CodeSnatch: Fortnite Android confidential-client secret shipped in the app + no PKCE — intercepted authorization code mints a full victim account session

## Abstract

The Fortnite Android account-login client (`client_id 3f69e56c…`) is a
confidential OAuth client whose `client_secret` is recoverable from the
shipped app, and its authorization-code flow has no PKCE binding. Its
registered redirect is the platform-unverifiable custom scheme
`com.epicgames.fortnite://fnauth/` — an exported, BROWSABLE Activity with no
autoVerify — so a co-installed malicious app can intercept the authorization
code. With the recovered secret, an intercepted code is redeemed from a pure
attacker context (no victim cookies, no `code_verifier`) for the victim's
user access token and a 48-hour refresh token. Demonstrated end-to-end
against two distinct canary accounts.

## Recovering the client secret (wire capture)

At app startup — before any user login and before anti-cheat arms — the
client sends its own `grant_type=client_credentials` bootstrap to
`account-public-service-prod.ol.epicgames.com/account/api/oauth/token`
carrying `Authorization: Basic base64(client_id:client_secret)`. Captured via
a rooted AVD + mitmproxy + a frida trust-all agent. The server returned HTTP
200 + an `eg1~` token, proving the secret is live.

Independent live verification (three-way differential):

```text
recovered secret + client_credentials      -> 200  eg1 token (client_id echoes correct)
wrong secret (control)                     -> 400  invalid_client_credentials (18033)
authorization_code + recovered secret + dummy code
                                           -> 400  authorization_code_not_found (18059)
```

The third response is the proof: `authorization_code_not_found`, **not**
`invalid_client` — the client authenticates; only a valid code is missing.

## The PKCE wall does not exist

The `/id/api/redirect` mint endpoint refuses `code_challenge` — the flow has
no PKCE. An intercepted authorization code is therefore unbound and
redeemable by anyone holding the client secret; a stolen code requires no
`code_verifier`.

## Full account takeover, demonstrated

For two distinct canary accounts (A and B): mint the victim's authorization
code, then from a bare attacker session (Basic client:secret only, NO victim
cookies, NO code_verifier):

```http
POST /account/api/oauth/token HTTP/1.1
Host: account-public-service-prod.ol.epicgames.com
Authorization: Basic <client_id:client_secret>
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&code=<victim_code>&token_type=eg1
```

Response — HTTP 200:

```json
{"access_token":"eg1~<JWT>", "refresh_token":"eg1~<JWT>","refresh_expires":172800,
 "account_id":"<VICTIM>", "client_id":"3f69e56c…","scope":["basic_profile"]}
```

The JWT `sub` is the victim account; `am = authorization_code`. The attacker
holds the victim's access token (2h) and refresh token (48h persistence).

## Real-world chain

1. Attacker ships a benign-looking Android app registering
   `com.epicgames.fortnite://` (a plain custom scheme, not an App Link — no
   `autoVerify`, so it is claimable by any co-installed app).
2. Victim logs into Fortnite; the OS delivers the auth code to the attacker
   app.
3. Attacker redeems the code at the token endpoint with the recovered secret
   (no PKCE verifier needed) — the victim's user token.

## Honest limiters

- Interception requires a malicious app co-installed on the victim device
  (custom-scheme squatting); this is not a pure-remote ATO.
- The minted token is `scope=basic_profile`.
- A secondary EOS secret recovered from a static `.pak` file returned `18033`
  (rotated) — reported as recovered-but-unverified, not a working
  credential. Only the wire-captured account-client secret is live. (The
  live secret values are withheld from this writeup; they are recoverable by
  the wire-capture method above.)

## Remediation

Make the Fortnite Android client a public OAuth client (no shipped secret)
and require PKCE (S256) on its authorization-code flow so an intercepted
code is useless without the app's verifier; and/or use verified App Links
(`autoVerify`) instead of a claimable custom scheme for the redirect.

## Credit

nichoc0
