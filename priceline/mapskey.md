# MapJack: unrestricted Google Maps API key shipped in the Priceline Android app

Found by nichoc0.

## Abstract

The Priceline Android app ships a Google API key in plaintext at
`res/values/strings.xml`. The key has no application restriction, no
referrer restriction, and no API restriction in Google Cloud. Anyone can
read it out of the public Play Store APK and call billable Google Maps
web-service APIs from any host. The calls authenticate as Priceline and bill
to Priceline's Google Cloud project.

Shipping a Maps SDK key in a client app is normal. The issue here is the
absence of a usage restriction on this key.

## Reproduction

Pull the key from the public APK at `res/values/strings.xml`, key name
`google_api_key`. Then call a billable Maps API from an arbitrary host with
no app context:

```bash
curl "https://maps.googleapis.com/maps/api/geocode/json?address=NewYork&key=AIzaSyBynTYctp3pg9FR0KJd6DYZSBzoWNmH3Sg"
```

The response is `"status": "OK"` with geocoding results. The request carried
no Referer and no Android app headers, which confirms the key has no
restriction. One request confirmed it. No volume or load testing was
performed.

## Impact

A third party can authenticate to Google Maps Platform as Priceline and have
the usage charged to Priceline's billing account. This is an exposed
credential and a security misconfiguration. It does not disclose Priceline
user data.

## Fix

Restrict the key in Google Cloud: Android app restriction (package name plus
signing cert SHA-1) and API restriction (Maps SDK only, not the billable
web-service APIs). Rotate the published key.

## Credit

nichoc0
