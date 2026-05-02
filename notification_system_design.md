# Notification System Design

This document describes a simple campus notification service for events, results, and placement messages.

## API

The service exposes a REST API:

- `POST /notifications` to create one or more notifications.
- `GET /notifications` to read notifications for a student.
- `POST /notifications/read` to mark notifications as read.
- `POST /notifications/broadcast` to broadcast a message to all students.

Clients receive JSON responses and timestamps use ISO 8601.

## Delivery

Connected clients use WebSockets. The server publishes new notifications to a message bus like Redis Pub/Sub and forwards them to clients using a WebSocket gateway. This keeps delivery separate from notification creation.

## Storage

Use PostgreSQL for notification storage.

Key table fields:

- `id` (UUID)
- `student_id`
- `type` (`event`, `result`, `placement`)
- `title`
- `message`
- `metadata`
- `is_read`
- `created_at`
- `read_at`

Use indexes on `(student_id, is_read, created_at DESC)` and `(type, created_at DESC)` for fast retrieval.

## Optimization

For unread notifications, query by student and `is_read = false`, ordered by `created_at DESC`. Add an index on `(student_id, is_read, created_at DESC)` to avoid full table scans.

For broadcast scale:

- write notifications in bulk,
- publish a broadcast event to a queue,
- process delivery asynchronously.

## Prioritization

Rank unread items using a type weight: `placement` > `result` > `event`, then sort by `createdAt` descending. For moderate lists, an in-memory sort is fine.
