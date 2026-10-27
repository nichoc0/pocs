# django CMS: missing authorization in render_object_structure discloses non-PageContent placeholder structure to low-privileged staff

CVE-2026-61663. GHSA-8qj2-c6q4-f399. CWE-862.
CVSS 4.3, CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:L/I:N/A:N.
Affects django CMS before 5.0.9. Fixed in 5.0.9.

## Abstract

django CMS is an enterprise content management system built on Django.
Before 5.0.9, `render_object_structure` fails to authorize non-PageContent
objects that use PlaceholderRelationField. An active staff user without
`cms.use_structure` permission and without model-level view or change
permission can request
`/admin/cms/placeholder/object/<content_type_id>/structure/<object_id>/`
with guessed values. The response discloses placeholder slot names, the
plugin tree, plugin identifiers, labels, and object existence for
frontend-editable objects. The fix applies
`user_can_view_placeholder_source` to the non-PageContent branch and keeps
the structure board read-only for view-only users.

## Reproduction

An active staff user with no `cms.use_structure` permission and no
view or change permission on the target model sends:

```http
GET /admin/cms/placeholder/object/<content_type_id>/structure/<object_id>/ HTTP/1.1
```

`content_type_id` values are enumerable through the standard Django
contenttypes framework. `object_id` values are guessable for common models
with auto-incrementing keys. The structure response discloses placeholder
slot names, the plugin tree, plugin identifiers, labels, and object
existence for objects the staff user has no permission to view or change.

## Root Cause

`render_object_structure` authorizes PageContent objects and takes no
authorization action for non-PageContent objects using
PlaceholderRelationField. The permission checks that protect the
PageContent branch, `cms.use_structure` and model-level view and change,
are absent on the other branch. Any active staff user enumerates the
placeholder structure of arbitrary content types by id.

## Fix

The fix applies `user_can_view_placeholder_source` to the non-PageContent
branch and keeps the structure board read-only for view-only users. Upgrade
to django CMS 5.0.9.

Advisory: GHSA-8qj2-c6q4-f399.
Patch: django-cms/django-cms#8703.
Commit: 9c82abfeb25471583e23906ea1ebef9202527b04.
Release: django-cms/django-cms/releases/tag/5.0.9.

## Credit

Reported by nichoc0 and doanmanhducz.
