# Query API Contract

## Endpoint

`GET /api/v1/audit-events`

## Filters

- `actor`: exact actor match.
- `from`: include events where `recordedAt >= from`.
- `to`: include events where `recordedAt < to`.
- `action`: exact action enum match.
- `outcome`: exact outcome enum match.
- `page`: zero-based page index, default `0`.
- `size`: page size, default `50`, maximum `200`.
- `snapshotId`: optional signed 64-bit integer boundary; include only events
  where `id <= snapshotId`, in both `content` and `totalElements`.
  When absent, no additional ID restriction applies.

`snapshotId` uses the existing integer query parsing rules, including whitespace
handling and signed hexadecimal input. Empty or blank values act as absent.
Malformed integers and values outside `[-9223372036854775808, 9223372036854775807]`
return `400 Bad Request` with a message naming `snapshotId`.
Negative and zero boundaries are valid. Blank `actor` remains an absent filter.

## Response

```json
{
  "content": [],
  "page": 0,
  "size": 50,
  "totalElements": 0
}
```

`content` items include:

- `id`
- `recordedAt`
- `actor`
- `action`
- `resourceType`
- `resourceId`
- `outcome`
- `reason`
- `metadata`

## Sort Order

Results are sorted by `recordedAt` descending, then unique `id` descending:
`ORDER BY recorded_at DESC, id DESC`.

## Bounded Traversal

The client supplies the same `snapshotId=S` with every page and keeps all other
filters and `size` unchanged. Events with IDs greater than S do not appear in
this traversal or change its `totalElements`. The boundary combines with all
existing filters using `AND`, for both content and count.

Choose S from known initial data before appends; the highest ID on the first
page need not be the journal's highest ID because time is the primary sort key.
The server does not choose S automatically or return it. The response has only
`content`, `page`, `size` and `totalElements`, as before.

This ID boundary is not a PostgreSQL transactional snapshot. Transactions that
start earlier but commit later are outside this contract. Requests without
`snapshotId` retain live page-based behavior, including possible page overlap
when an event is appended between requests.

## Additive Filter Rule

Filters are additive and read-only; adding a filter must never change results when its parameter is absent.

When both `action` and `outcome` are present, they combine with `AND`.

Invalid enum values return `400 Bad Request` with a message naming the invalid parameter.
