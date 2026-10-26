# MapJack: unrestricted Google Maps API key shipped in the Priceline Android app — any third party bills Google API usage to Priceline's cloud project

## Abstract

The Priceline Android app ships a Google API key in plaintext in
`res/values/strings.xml`. The key carries no application, referrer, or API
restriction configured in Google Cloud. Because it is unrestricted, it can be
read directly out of the public Play Store APK and used by anyone, from any
host, to call billable Google Maps web-service APIs — authenticating as, and
billing to, Priceline's Google Cloud project.

Shipping a Maps SDK key inside a client app is normal; the issue here is
specifically the absence of a usage restriction on this key.

## Reproduction

Pull the key from the public APK — `res/values/strings.xml` →
`google_api_key` — then call a billable Maps API from an arbitrary host with
no app context:

```bash
curl "https://maps.googleapis.com/maps/api/geocode/json?address=NewYork&key=AIzaSyBynTYctp3pg9FR0KJd6DYZSBzoWNmH3Sg"
```

The response is `"status": "OK"` with geocoding results, returned to a
request that has no Referer and no Android app headers — confirming the key
carries no restriction. A single request is sufficient to confirm; no volume
or load testing performed.

## Impact

The key is a Priceline credential published in a public artifact with no
usage restriction: a third party can authenticate to Google Maps Platform as
Priceline and have those calls charged to Priceline's billing account. This
is a security-misconfiguration / exposed-credential issue. It does not
disclose Priceline user data and has no confidentiality or integrity impact
on Priceline systems or customers.

## Remediation

Restrict the key in Google Cloud: Android app restriction (package name +
signing cert SHA-1) plus API restriction (only the Maps SDK, not the billable
web-service APIs), and rotate the currently-published key.

## Credit

nichoc0
