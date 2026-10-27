# FromForge: LiveKit SIP trunk source-IP allowlist matched against the caller's From header host

GHSA-82qx-7757-5jqv. CWE-346.
Affected: github.com/livekit/protocol >= 1.35.0.
Reproduced against livekit-server v1.13.5, livekit/sip v1.11.0, protocol at
commit d74e7cd.

Found by nichoc0.

## Abstract

The SIP trunk field `allowed_addresses` is documented as "CIDR or IPs that
traffic is accepted from". The trunk matching function compares every
allowlist entry against two values: the network source address, which is
correct, and the host in the call's From URI, which the caller writes. A
caller from any network places an allowlisted IP in their From header and
passes the check. Reproduced against livekit-server v1.13.5 with
livekit/sip v1.11.0 and protocol at commit d74e7cd: an unauthenticated
caller joins a room behind an IP allowlist trunk, publishes a microphone
track, and their audio reaches a livekit-agents worker STT input.

## The A and the B

Trunk ST_oWSGRvYxGQCU: number +15550001, allowed_addresses [9.9.9.9], no
auth username, no auth password. Dispatch rule dispatch_rule_direct, room
sip-victim, no pin.

Both INVITEs came from the same real source 172.19.0.8, not in the
allowlist, and are identical except the From URI host.

Control, From host not allowlisted:

```text
INVITE sip:+15550001@sip SIP/2.0
From: <sip:attacker@1.2.3.4>;tag=fd0277ee

sip-1  | ... "fromHost": "1.2.3.4" ... Rejecting inbound flood
sip-1  | ... reason: "flood" ... "call was not authorized by trunk configuration"
```

No trunk matched. Call dropped. No participant joined.

Attack, From host equals the allowlisted IP:

```text
INVITE sip:+15550001@sip SIP/2.0
From: <sip:attacker@9.9.9.9>;tag=acb44afb

sip-1  | ... "fromIP": "172.19.0.8", "fromHost": "9.9.9.9" ...
         "sipTrunk": "ST_oWSGRvYxGQCU"
sip-1  | ... SIP participant joined room ... "room": "sip-victim",
         "participant": "sip_attacker"
sip-1  | ... published track ... "name": "sip_attacker", "source": "MICROPHONE"
sip-1  | ... subscribing to existing room participants
```

The trunk matched. The caller joined the room, published a microphone
track, and subscribed to the other participants.

## Root Cause

In protocol, sip/sip.go, the function matchAddrMasks:

```go
func matchAddrMasks(addr string, host string, masks []string) bool {
	ip, err := netip.ParseAddr(addr)
	if err != nil {
		return true
	}
	masks = filterInvalidAddrMasks(masks)
	if len(masks) == 0 {
		return true
	}
	for _, mask := range masks {
		if mask == host || matchAddrMask(ip, mask) {
			return true
		}
	}
	return false
}
```

matchAddrMask(ip, mask) is the real check: exact IP or CIDR against the
network source. The branch `mask == host` compares every allowlist entry,
including plain IPs, against the From URI host. The caller writes that
host.

lk-sip pkg/sip/inbound.go builds the call info from the INVITE unchanged:

```go
callInfo := &rpc.SIPCall{
	...
	SourceIp:  src.Addr().String(),
	From:      ToSIPUri("", from),
```

ToSIPUri copies u.Host through as Host. In the attack above the network
source is 172.19.0.8. matchAddrMask(172.19.0.8, "9.9.9.9") is false, so the
match could only come from `mask == host`. The trunk matching loop calls it
with the caller-controlled value:

```go
if !matchAddrMasks(call.SourceIp, call.From.Host, tr.AllowedAddresses) {
```

The same loop also gates on matchNumbers(call.From.User,
tr.AllowedNumbers), so a caller-number allowlist is checked against the
caller's own From user. Caller-ID spoofing is expected in telephony.
Matching a network-origin allowlist against a header value is not. It is
the same class as IP ACLs checked against X-Forwarded-For, and those get
fixed wherever they surface.

An empty or unparseable SourceIp also returns true from matchAddrMasks, and
GetSIPTrunkAuthentication only warns on an empty SourceIp.

## Why the `mask == host` branch exists, and what it broke

Commit d8a31f2, 2025-02-25, "Handle SIP hostname in address filters. Expose
attribute." (#987) added the branch, together with a loosened isValidMask
that accepts some.host.name. The purpose was hostname entries: an operator
lists sip.provider.com, and a call whose From host is that name matches the
trunk.

Before that commit the field had no host matching at all:

```go
func matchAddrMasks(addr netip.Addr, masks []string) bool {
	...
	for _, mask := range masks {
		if matchAddrMask(addr, mask) {
			return true
		}
	}
```

isValidMask rejected anything that was not an IP or CIDR. So an operator who
listed IPs had source-IP enforcement until #987, and has had a
caller-supplied header as an alternative ever since. The proto comment for
allowed_addresses still reads "CIDR or IPs that traffic is accepted from"
and was never updated to describe From-host matching.

The hostname goal does not require the current shape. An entry that does not
parse as an IP or CIDR could be compared to the host, leaving IP and CIDR
entries matched against the network source only.

## Impact

An unauthenticated caller who can reach the SIP port joins rooms behind an
IP-ACL trunk. They hear the room and they speak into it.

With digest auth configured on the trunk this does not apply. The trunk must
rely on allowed_addresses alone. With a pin on the dispatch rule the caller
still matches the trunk but needs the pin, and the room join requires a
no-pin dispatch rule, which is a documented-supported configuration.

In the same lab, a livekit-agents worker in the room received 951 audio
frames from the caller's track:

```text
AGENT AUDIO_FRAMES from track=TR_AMmriuZSHCsTb2 count=951
(room sip-victim3, job AJ_oScHN7FP9KnX)
```

For deployments running voice agents, the caller's audio reaches the agent
input, and from there the STT and tool chain.

## Fix

Scope the host comparison to entries that are not an IP or CIDR. Keep
matchAddrMask(ip, mask) for IP and CIDR entries. Compare `mask == host` only
for the rest. That preserves the hostname feature from #987 and restores
source enforcement for IP entries.

## Reproducer

```text
docker compose up -d
# livekit-server v1.13.5, livekit/sip v1.11.0, redis
bash sip_setup.sh
# creates the IP-ACL trunk with allowed_addresses [9.9.9.9] and no auth,
# plus dispatch_rule_direct with no pin
docker run --rm --network livekit-lab_lk -v $PWD/sip_invite.py:/s.py:ro \
	python:3.12-slim python /s.py sip 9.9.9.9 ATTACK
# same command with 1.2.3.4 for the control
```

Full lab and both transcripts available on request.

## Credit

nichoc0
