---
title: Audit and compliance
description: 1claw records access and failures in an audit log; query events via GET /v1/audit/events; secret values are never logged.
sidebar_position: 5
---

# Audit and compliance

All API access (and relevant failures) can be recorded in an **audit log**. Secret **values** are never stored in audit events; only metadata (e.g. path, action, actor, timestamp) is recorded.

## Querying audit events

**GET /v1/audit/events** — Returns events for the caller’s organization. Query parameters may include:

- `resource_id` — Filter by resource (e.g. secret path).
- `actor_id` — Filter by actor (user or agent UUID).
- `action` — Filter by action type.
- `from`, `to` — Time range (ISO 8601).
- `limit`, `offset` — Pagination.

Response shape: `{ "events": [ ... ], "count": N }`. Each event typically has id, org_id, actor_type, actor_id, action, resource_type, resource_id, metadata, timestamp.

## Typical events

- Secret read/write/delete (path and actor; no value).
- Auth success/failure (no credentials).
- Policy create/update/delete.
- Agent registration, key rotation, deactivation.

## Tamper-evident hash chain

Every audit event includes a cryptographic hash chain that links it to the previous event:

- `prev_event_id` — UUID of the immediately preceding event in the org's audit log.
- `integrity_hash` — SHA-256 of `prev_hash|event_id|actor_id|action|resource_type|resource_id|timestamp`.

This makes the audit log **append-only and tamper-evident**: modifying or deleting any event breaks the hash chain for all subsequent entries. You can verify integrity by walking the chain from the latest event back to the first (`prev_event_id = NULL`).

## Best practices

Use the audit log for compliance reviews, incident response, and access analysis. Export or forward events to your SIEM or logging pipeline as needed.
