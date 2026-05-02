# Notification System Design

This document outlines the design of a campus notification platform. The goal is to
support both near real‑time delivery and long‑term persistence of messages such as
events, exam results and placement offers. The following sections correspond to
the stages described in the assignment.

## Stage 1 – API contract

At its core the notification service exposes a RESTful API. The API is
hierarchical and uses nouns rather than verbs. All resources are encoded as
JSON and timestamps follow the ISO 8601 format (e.g. `2026‑05‑02T10:30:00Z`).

### Endpoints

| Method & path            | Purpose                                                  | Request body                                                                                                                                             | Response & status codes                                                                                                                                                         |
|--------------------------|----------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `POST /notifications`    | Create a notification for one or more recipients         | `{ "recipientIds": ["1042", …], "type": "event", "title": "Workshop", "message": "Cloud computing workshop at 11am", "metadata": {"location": "Auditorium"} }` | `201 Created` with `{ "id": "<uuid>", "createdAt": "…" }` on success. Returns `400 Bad Request` when required fields are missing and `401 Unauthorized` for unauthenticated callers. |
| `GET /notifications`     | Retrieve notifications for a user                        | Query parameters: `studentId` (required), `isRead` (optional), `limit` (default 50), `offset` (default 0).                                            | `200 OK` with an array of notifications ordered by `createdAt DESC`. Pagination metadata (total count and next cursor) is included.                                                  |
| `POST /notifications/read` | Mark notifications as read                               | `{ "studentId": "1042", "notificationIds": ["a2f4…", "d7c3…"] }`                                                                            | `200 OK` with an empty body when updates succeed. Returns `404` if any referenced notification does not belong to the user.                                                      |
| `POST /notifications/broadcast` | Broadcast a message to all active students           | `{ "type": "event", "title": "Holiday", "message": "Campus will be closed on Friday", "metadata": {} }`                                    | `202 Accepted` to acknowledge the request. The actual fan‑out of messages is performed asynchronously in the background.                                                       |

### Near real‑time delivery

To push new messages to connected clients without polling, the system uses WebSockets.
When a user opens the portal the client establishes a persistent socket connection
and authenticates using a JWT. On the server side, new notifications are
published to an in‑process message broker (e.g. Redis Pub/Sub). A WebSocket
gateway subscribes to the channels corresponding to recipient IDs and relays
messages to connected clients. This design decouples message creation from
delivery and scales horizontally.

## Stage 2 – Persistence & schema

### Choice of database

A relational database such as **PostgreSQL** is well suited for notifications.
The schema is simple, strong consistency is desirable and the database’s
query planner can leverage composite indexes effectively. While a document store
like MongoDB could also work, SQL provides predictable performance and rich
query capabilities. For temporary data such as WebSocket sessions a volatile
store like Redis is used.

### Schema

```sql
CREATE TABLE notification (
  id            UUID PRIMARY KEY,
  student_id    TEXT NOT NULL,
  type          TEXT NOT NULL CHECK (type IN ('event','result','placement')),
  title         TEXT NOT NULL,
  message       TEXT NOT NULL,
  metadata      JSONB,
  is_read       BOOLEAN NOT NULL DEFAULT FALSE,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  read_at       TIMESTAMPTZ
);

-- Efficient retrieval of unread notifications for a student ordered by recency
CREATE INDEX idx_notification_student_unread_created
  ON notification (student_id, is_read, created_at DESC);

-- Index to find all recipients of a certain notification type quickly
CREATE INDEX idx_notification_type_created
  ON notification (type, created_at DESC);
```

### Queries

- Retrieve unread notifications for a user:

  ```sql
  SELECT id, type, title, message, metadata, created_at
  FROM notification
  WHERE student_id = $1 AND is_read = FALSE
  ORDER BY created_at DESC
  LIMIT $2 OFFSET $3;
  ```

- Mark notifications as read:

  ```sql
  UPDATE notification
  SET is_read = TRUE, read_at = NOW()
  WHERE student_id = $1 AND id = ANY($2::UUID[]);
  ```

- Insert notifications for multiple recipients (performed in a single statement):

  ```sql
  INSERT INTO notification (id, student_id, type, title, message, metadata)
  SELECT gen_random_uuid(), student_id, $1, $2, $3, $4
  FROM UNNEST($5::TEXT[]) AS student_id;
  ```

### Scaling considerations

As the volume grows beyond millions of rows the table may no longer fit in
memory and sequential scans become expensive. To mitigate this:

1. **Partitioning:** Partition the notification table by `created_at` (e.g.
   monthly) or by hash of `student_id`. This confines writes and reads to
   smaller partitions and improves vacuum efficiency.
2. **Archival:** Move old read notifications to an archive table or
   cold storage (e.g. S3/Glacier) after a configurable retention period.
3. **Caching:** Store the count of unread notifications per student in Redis.
   When users check their dashboard this counter is returned immediately
   without hitting the database; the full list is fetched on demand.
4. **Horizontal scaling:** Use a read replica for servicing `GET` requests
   while writes continue to go to the primary. An ORM or query router can
   direct read queries appropriately.

## Stage 3 – Query optimisation

The provided query:

```sql
SELECT *
FROM notification
WHERE student_id = 1042
  AND is_read = false
ORDER BY created_at DESC;
```

performs a full scan of the `notification` table and sorts the results.
With five million rows this is prohibitively slow. Creating a composite index
on `(student_id, is_read, created_at DESC)` solves both the filtering and
ordering in a single step. PostgreSQL will perform an index scan instead of a
sequential scan.

```sql
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_notification_user_unread
  ON notification (student_id, is_read, created_at DESC);
```

After adding the index you can verify the execution plan using `EXPLAIN ANALYZE`.
The query should now use an `Index Scan` and complete in milliseconds. If
there is still a measurable delay consider increasing `work_mem` so that the
entire result set fits in memory and avoid disk‐based sorts.

To find all students who have received a *placement* notification in the last
seven days, the following query leverages the index on `type` and `created_at`:

```sql
SELECT DISTINCT student_id
FROM notification
WHERE type = 'placement'
  AND created_at >= NOW() - INTERVAL '7 days';
```

## Stage 4 – Reducing database load

Fetching notifications on every page load can generate millions of database
queries per day. Several strategies can help:

1. **Client side caching:** The API returns an `ETag` header and clients
   include `If-None-Match` in subsequent requests. If the data has not
   changed the server responds with `304 Not Modified` and the client uses
   the cached copy.
2. **Server side caching:** Use Redis to cache the latest N unread
   notifications per student. The cache key could be `notifications:<id>` with
   a TTL of a few minutes. When a notification is created or marked as read
   the corresponding cache entry is invalidated.
3. **Long polling / WebSockets:** For dashboard widgets displaying a small
   number of recent messages you can avoid polling entirely by pushing
   updates over a socket as described in Stage 1. The dashboard subscribes
   to its own channel and receives a new payload only when something has
   changed.

## Stage 5 – Broadcasting to all students

The pseudocode provided performs 50 000 synchronous HTTP calls in a loop.
This serial approach is inefficient and prone to timeout. Instead:

1. **Batch writes:** Insert the notifications into the database in bulk using
   a single `INSERT…SELECT` statement (see Stage 2). Separating persistence
   from delivery removes the need to iterate over each student.
2. **Asynchronous delivery:** Publish a single message to a queue (e.g.
   RabbitMQ, Kafka or Redis Streams) that describes the broadcast. A pool of
   worker processes consumes this message, fans out emails using concurrency
   primitives and logs results. This allows you to control the degree of
   parallelism and handle retries.
3. **Rate limiting:** When sending emails through external providers apply
   back pressure so you do not exceed their throughput limits. Libraries such
   as [p-limit](https://github.com/sindresorhus/p-limit) can restrict the
   number of concurrent promises.

A simple Node example using concurrency control might look like this:

```js
const pLimit = require('p-limit');
const limit = pLimit(25); // run up to 25 email sends concurrently
await Promise.all(
  recipients.map(recipient => limit(() => sendEmail(recipient)))
);
```

However, in production a proper message queue and worker pool should be used.

## Stage 6 – Ranking unread notifications

To surface the most important unread notifications we first assign a weight to
each type: `placement` (3), `result` (2) and `event` (1). Higher weights
indicate greater significance. Within each type we order by recency using the
`createdAt` timestamp. The composite comparator therefore sorts by weight
descending and then by timestamp descending. The top N notifications are
selected after sorting.

The backend implementation in `notification_app_be/src/priority.js` encodes
this logic. Given an array of notifications and an integer `topN` it returns
the highest ranked subset. A simplified example:

```js
const { prioritize } = require('./priority');
const input = [
  { type: 'event',    timestamp: '2026-04-01T10:00:00Z' },
  { type: 'placement',timestamp: '2026-03-31T09:00:00Z' },
  { type: 'result',   timestamp: '2026-04-02T08:00:00Z' }
];
const best = prioritize(input, 2);
// best will contain the placement and result notifications in that order
```

Because the number of unread notifications per user is relatively small (tens
to hundreds), a simple in‑memory sort is sufficient. For huge lists you could
push the prioritisation down to the database by adding a computed column
representing the weight and using a `ORDER BY weight DESC, created_at DESC` clause.
