---
title: SDKs overview
description: 1claw provides a TypeScript/JavaScript SDK (@1claw/sdk); curl and fetch examples work with any language.
sidebar_position: 0
---

# SDKs overview

- **JavaScript/TypeScript** — Official `@1claw/sdk`. Built from the same OpenAPI 3.1 spec as the API; supports agent auth, secrets, billing, and optional x402. See [JavaScript SDK](/docs/sdks/javascript).
- **Python** — No official SDK yet; use [curl examples](/docs/sdks/curl-examples) or implement a thin client with `requests`/`httpx`. See [Python](/docs/sdks/python).
- **curl / HTTP** — All endpoints are REST; you can use curl, fetch, or any HTTP client. See [curl examples](/docs/sdks/curl-examples).

**API contract:** The canonical source of truth for the API is the **OpenAPI 3.1** spec. It is published as the npm package [@1claw/openapi-spec](https://www.npmjs.com/package/@1claw/openapi-spec) (YAML and JSON) and is used to generate the SDK’s types and to validate requests. For a concise endpoint list, see [API reference](/docs/reference/api-reference).
