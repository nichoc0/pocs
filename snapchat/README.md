# GhostWrite: Snapchat creativeContents PUT is an upsert keyed on a client-supplied id with no ownership check — cross-org overwrite, ownership strip, global listing

## Abstract

`PUT /api/v1/creativeContents/` is an upsert keyed on an `id` taken from the
request body. It does not check that the caller owns that id. An account in
one organization can overwrite another organization's ad creative with its
own content; the write strips the original `created_by`, which flips the
object to globally readable; a stripped object then appears in the global
listing every account receives; and any third account can overwrite or delete
it from there. The same handler shape exists on `creativeFields` and `media`,
making this a shared write-layer defect across at least three resource types
— including uploaded media, so it reaches actual creative assets, not just
metadata.

## Reproduction (three accounts, three unrelated organizations)

Account Q creates an object; `created_by = Q`:

```text
Q  POST /api/v1/creativeContents/  -> id a37aaf9e-…
```

Account P (different org, no relationship to Q) requests the same id:

```text
P  GET /api/v1/creativeContents/a37aaf9e-…   -> 404
```

P sends a PUT with that same id in the body:

```text
P  PUT /api/v1/creativeContents/ {"id":"a37aaf9e-…", …P's content}  -> 200
```

Q's content is gone on the next read, and Q's `created_by` field is gone
with it:

```text
Q  GET /api/v1/creativeContents/a37aaf9e-…   -> 200, returns P's content
```

The same steps on `creativeFields` and `media` produce the same result — same
accounts, same sequence, two more resource types.

## The ownership-strip cascade

Once `created_by` is stripped, the object becomes readable by any signed-in
account, and for `creativeFields` it also appears in the global
`GET /api/v1/creativeFields/` list every account receives — no id is even
needed to find it. A globally listed, `created_by`-null object can then be
overwritten or deleted by a third unrelated account:

```text
R (third org)  PUT  {"id":"<stripped object>"}              -> 200
R (third org)  DELETE /api/v1/creativeFields?ids=<stripped>  -> 200
```

## The global template exposure (proven primitive, not run live)

A real global template pulled from that same global list carries no
protection flag — no `is_system`, no `is_locked`, nothing. It differs from an
ordinary `creativeField` only by `organization_id`/`publisher_id` being
GLOBAL, and the write path checks neither field. This step was not run
against a live template (that would deface something every advertiser sees);
the primitive that would let an attacker overwrite it is proven above on
canary objects.

## Root Cause

The PUT handler treats the body's `id` as the upsert key and performs no
ownership lookup: no check that the id belongs to the caller's organization,
no check on `organization_id`/`publisher_id`, no preservation of `created_by`
on overwrite. Read-path authorization keys on the same fields, so the
stripped object becomes world-readable as a side effect of the write.

## Impact

An advertiser can overwrite another organization's ad creative with their own
content, and the original cannot be recovered through the API. The same
write path lets an unrelated third account later overwrite or delete that
same object once it has been stripped of ownership. Nothing in the data model
distinguishes a shared global template from an ordinary creativeField, and
the write and delete paths both ignore `organization_id` — the same primitive
that lands on a canary object lands on a template every advertiser depends
on. Media objects (uploaded images and video) are affected the same way as
the text-based types.

## Remediation

Key the upsert on the server-side object lookup (id must resolve to an
object owned by the caller's organization, else 404); preserve `created_by`
and `organization_id` on update; mark and protect global templates
(`is_system`/`is_locked`) in the write and delete paths; exclude unowned
objects from the global listing.

## Credit

nichoc0
