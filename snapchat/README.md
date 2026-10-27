# GhostWrite: Snapchat creativeContents PUT is an upsert keyed on a client-supplied id with no ownership check

Found by nichoc0.

## Abstract

`PUT /api/v1/creativeContents/` is an upsert keyed on an `id` taken from the
request body. It does not check that the caller owns that id. An account in
one organization overwrites another organization's ad creative with its own
content. The write strips the original `created_by` field, which flips the
object to globally readable. A stripped object appears in the global listing
every account receives, and any third account can then overwrite or delete
it. The same handler shape exists on `creativeFields` and `media`. The
defect is in the shared write layer, across at least three resource types.

## Reproduction

Three accounts in three unrelated organizations.

Account Q creates an object:

```text
Q  POST /api/v1/creativeContents/  ->  id a37aaf9e-…  created_by: Q
```

Account P requests the same id:

```text
P  GET /api/v1/creativeContents/a37aaf9e-…   ->  404
```

Account P sends a PUT with that id in the body:

```text
P  PUT /api/v1/creativeContents/ {"id":"a37aaf9e-…", <P's content>}   ->  200
```

Account Q reads it back:

```text
Q  GET /api/v1/creativeContents/a37aaf9e-…   ->  200, returns P's content
```

The same steps on `creativeFields` and on `media` produce the same result.
Same accounts, same sequence, two more resource types.

## The ownership-strip cascade

Once `created_by` is stripped, the object becomes readable by any signed-in
account. For `creativeFields` it also appears in the global
`GET /api/v1/creativeFields/` list that every account receives, so no id is
needed to find it. A globally listed object with a null `created_by` can
then be overwritten or deleted by a third account:

```text
R  PUT  /api/v1/creativeContents/ {"id":"<stripped object>"}       ->  200
R  DELETE /api/v1/creativeFields?ids=<stripped object>             ->  200
```

## The global template exposure

A real global template pulled from that same list has no protection flag.
No `is_system`, no `is_locked`. It differs from an ordinary creativeField
only in that `organization_id` and `publisher_id` are GLOBAL, and the write
path checks neither field. This step was not run against a live template,
because that would deface something every advertiser sees. The primitive
that reaches it is proven above on canary objects.

## Root Cause

The PUT handler treats the body's `id` as the upsert key and performs no
ownership lookup. There is no check that the id belongs to the caller's
organization, no check on `organization_id` or `publisher_id`, and no
preservation of `created_by` on overwrite. The read path keys on the same
fields, so the stripped object becomes world-readable as a side effect of
the write.

## Impact

An advertiser overwrites another organization's ad creative with their own
content, and the original cannot be recovered through the API. A third
account can later overwrite or delete the same object once it has been
stripped of ownership. Nothing in the data model distinguishes a shared
global template from an ordinary creativeField, and the write and delete
paths both ignore `organization_id`, so the same primitive that lands on a
canary object lands on a template every advertiser depends on. Media
objects, uploaded images and video, are affected the same way as the two
text-based types.

## Fix

Key the upsert on a server-side lookup: the id must resolve to an object
owned by the caller's organization, else return 404. Preserve `created_by`
and `organization_id` on update. Mark and protect global templates
(`is_system`, `is_locked`) in the write and delete paths. Exclude unowned
objects from the global listing.

## Credit

nichoc0
