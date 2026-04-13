---
title: Error Handling
deprecated: false
hidden: false
metadata:
  robots: index
---
DCE API errors use standard HTTP status codes and JSON bodies. This guide explains what to expect and how to handle failures safely.

## Response patterns

Success responses are route-specific JSON objects.

Error responses commonly include:

```json
{
  "error": "Message describing failure"
}
```

Some validation routes may include structured fields such as `details`.

## Status code guide

| Code  | Meaning                                     | Typical action                                |
| ----- | ------------------------------------------- | --------------------------------------------- |
| `200` | Success                                     | Process response                              |
| `400` | Validation or business-rule failure         | Fix request payload/parameters                |
| `401` | Missing/invalid auth                        | Check API key and header                      |
| `403` | Permission denied                           | Verify key scope/role                         |
| `404` | Resource not found                          | Validate IDs/paths                            |
| `410` | Gone (e.g., consumed/expired deposit token) | Regenerate token/session                      |
| `413` | Payload too large                           | Reduce payload size                           |
| `429` | Rate limit reached                          | Retry with backoff                            |
| `500` | Internal server error                       | Retry later and contact support if persistent |

## Retry strategy

Retry only transient failures:

- `429`, `500`, network timeout/connection issues

Do **not** blindly retry:

- `400`, `401`, `403`, most `404` (unless eventual consistency is expected)

Recommended backoff:

- Exponential backoff + jitter (for example 1s, 2s, 4s, 8s with randomness)
- Max attempts: 3-5 depending on operation criticality

## Safe integration patterns

- Treat all amount fields as decimal strings.
- Use your own idempotency reference (`referenceId`) to prevent duplicate business actions.
- Log request context (route, status, payload hash, timestamps) without logging secrets.

## Example error handling (Node.js)

```javascript
const response = await fetch(url, options);
const body = await response.json().catch(() => ({}));

if (!response.ok) {
  if ([429, 500].includes(response.status)) {
    // retry with backoff
  } else {
    // handle as non-retryable
    throw new Error(body.error || `HTTP ${response.status}`);
  }
}
```

## When to contact support

Reach out with:

- environment (`staging` or `production`)
- endpoint + approximate timestamp (UTC)
- status code + response body snippet
- correlation/request ID if available

Contact: [hello@dcepay.io](mailto:hello@dcepay.io)
