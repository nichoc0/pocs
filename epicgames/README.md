# CodeSnatch: Fortnite Android ships a confidential-client secret and its authorization-code flow has no PKCE

Found by nichoc0.

## Abstract

The Fortnite Android account-login client (`client_id 3f69e56c…`) is a
confidential OAuth client. Its `client_secret` is recoverable from the
shipped app, and its authorization-code flow has no PKCE binding. The
registered redirect is the custom scheme `com.epicgames.fortnite://fnauth/`:
an exported BROWSABLE Activity with no autoVerify, so a co-installed
malicious app receives the authorization code. With the recovered secret,
that code redeems from a bare attacker context. No victim cookies and no
code_verifier. The result is the victim's user access token and a 48-hour
refresh token. Demonstrated end to end against two canary accounts.

## Recovering the client secret

At app startup, before any user login and before anti-cheat arms, the client
sends a `grant_type=client_credentials` bootstrap to
`account-public-service-prod.ol.epicgames.com/account/api/oauth/token` with
`Authorization: Basic base64(client_id:client_secret)`. The capture setup is
a rooted AVD, mitmproxy, and a frida trust-all agent. The server returned
HTTP 200 with an `eg1~` token, which proves the secret is live.

Three-response differential confirms it:

```text
recovered secret + client_credentials   -> 200  eg1 token, client_id echoes correct
wrong secret (control)                  -> 400  invalid_client_credentials (18033)
authorization_code + recovered secret + dummy code
                                        -> 400  authorization_code_not_found (18059)
```

The third response is the proof. `authorization_code_not_found` instead of
`invalid_client` means the client authenticates and only a valid code is
missing.

## The PKCE wall does not exist

The `/id/api/redirect` mint endpoint refuses `code_challenge`. The flow has
no PKCE. An intercepted authorization code is unbound and redeemable by
anyone holding the client secret. A stolen code does not require the victim
app's code_verifier.

## Full account takeover

For two distinct canary accounts: mint the victim's authorization code, then
redeem it from a bare attacker session. No victim cookies, no code_verifier:

```http
POST /account/api/oauth/token HTTP/1.1
Host: account-public-service-prod.ol.epicgames.com
Authorization: Basic <client_id:client_secret>
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&code=<victim_code>&token_type=eg1
```

Response:

```json
{"access_token":"eg1~<JWT>", "refresh_token":"eg1~<JWT>","refresh_expires":172800,
 "account_id":"<VICTIM>", "client_id":"3f69e56c…","scope":["basic_profile"]}
```

The JWT `sub` is the victim account and `am` is `authorization_code`. The
attacker holds the victim's access token for 2 hours and a refresh token for
48 hours.

## Real-world chain

1. The attacker ships a benign app registering
   `com.epicgames.fortnite://`. It is a plain custom scheme, not an App
   Link, so it has no autoVerify and any co-installed app can claim it.
2. The victim logs into Fortnite. The OS delivers the auth code to the
   attacker app.
3. The attacker redeems the code with the recovered secret. No PKCE
   verifier is needed.

## Limiters

- Interception requires a malicious app co-installed on the victim device.
  This is not a pure-remote ATO.
- The minted token is `scope=basic_profile`.
- A secondary EOS secret recovered from a static `.pak` file returned 18033
  (rotated). It is reported as recovered but unverified. Only the
  wire-captured account-client secret is live. The live secret values are
  withheld from this writeup. They are recoverable by the wire-capture
  method above.

## Fix

Make the Fortnite Android client a public OAuth client with no shipped
secret. Require PKCE (S256) on its authorization-code flow so an intercepted
code is useless without the app's verifier. Use verified App Links
(autoVerify) instead of a claimable custom scheme for the redirect.

## Credit

nichoc0
