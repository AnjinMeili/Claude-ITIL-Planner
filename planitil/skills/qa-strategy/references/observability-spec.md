# Observability Specification Reference

Structured logging format, the 3-tier metric hierarchy, alerting threshold patterns, and the operational distinction between logging for debugging vs. logging for operations.

---

## Two Modes of Logging

Before writing a log statement, identify its purpose:

**Logging for debugging** answers: "What happened during this request so I can reproduce and fix a bug?"
- Audience: developer actively investigating an incident
- Contains: internal state, function arguments, intermediate results, timing
- Level: DEBUG
- Volume: High — fine-grained detail
- Production behavior: Disabled by default; enabled temporarily during active incident investigation

**Logging for operations** answers: "Is the system behaving correctly right now? Did this business event happen?"
- Audience: on-call engineer monitoring the system, or an alert rule evaluating conditions
- Contains: system boundary crossings, errors with context, key business events
- Level: INFO, WARN, ERROR
- Volume: Low — signal, not noise
- Production behavior: Always enabled

The most common observability mistake is writing debug logging at INFO level in production. The result is a log stream that is too noisy to monitor and too expensive to retain.

---

## Structured Logging Format

Log in JSON. Every log event carries these top-level fields:

```json
{
  "timestamp": "2025-06-01T14:23:11.042Z",
  "level": "ERROR",
  "service": "payments",
  "trace_id": "abc123def456",
  "message": "Payment authorization failed",
  "context": {
    "order_id": "ord_789",
    "error_type": "CardDeclined",
    "provider_code": "do_not_honor",
    "duration_ms": 342
  }
}
```

**Field definitions:**

| Field | Type | Required | Description |
|---|---|---|---|
| `timestamp` | ISO 8601 UTC string | Yes | Log event timestamp. Always UTC. |
| `level` | string | Yes | DEBUG / INFO / WARN / ERROR / FATAL |
| `service` | string | Yes | The service or application name. Consistent across all logs for this service. |
| `trace_id` | string | Yes | Correlation ID for the request or operation. Propagate from the initiating request. Generate a UUID if no trace context exists. |
| `message` | string | Yes | Human-readable description. Do not put structured data in the message; put it in `context`. |
| `context` | object | No | Key-value pairs of relevant state. Do not include PII. |

**What goes in `context` vs `message`:**

- `message`: "Payment authorization failed" (constant; useful for log search and alert matching)
- `context.order_id`: "ord_789" (variable; useful for debugging a specific case)

Putting variable data in `message` breaks log aggregation — every unique value creates a new log fingerprint.

---

## Minimum Required Log Events

### System Boundary Events (INFO)

Log every request and response at system boundaries — inbound HTTP, outbound HTTP, database queries, queue messages. Minimum fields for each direction:

**Inbound request:**
```json
{
  "message": "Request received",
  "context": {
    "method": "POST",
    "path": "/api/payments",
    "trace_id": "abc123",
    "user_id": "usr_456"
  }
}
```

**Outbound response:**
```json
{
  "message": "Request completed",
  "context": {
    "method": "POST",
    "path": "/api/payments",
    "status_code": 200,
    "duration_ms": 187
  }
}
```

Log the request on receipt and the response on completion. Do not log request bodies by default — they may contain secrets or PII. Log body shape (field names, not values) if needed for debugging.

### Error Events (ERROR)

Log every error with enough context to reproduce the problem without further investigation. Required fields in `context`:

- `error_type`: The exception class or error category
- `error_message`: The exception message
- `input_summary`: A sanitized description of the input that triggered the error (no PII, no secrets)
- Any entity IDs relevant to the error (user_id, order_id, job_id)

```json
{
  "level": "ERROR",
  "message": "Database write failed",
  "context": {
    "error_type": "UniqueViolation",
    "error_message": "duplicate key value violates unique constraint 'users_email_key'",
    "operation": "create_user",
    "input_summary": "email domain: example.com"
  }
}
```

### Business Event Logs (INFO)

Log key business events as they occur. A business event is a state change that matters to the business — not an implementation detail.

Examples:
- `"User registered"` — with user_id
- `"Order placed"` — with order_id, item count, total_amount (not card details)
- `"Payment processed"` — with order_id, provider, result (success/failure)
- `"Job completed"` — with job_id, job_type, duration_ms, record_count

Business event logs are the basis for business metrics and the audit trail for incident post-mortems.

---

## The 3-Tier Metric Hierarchy

### Tier 1: System Metrics (Infrastructure)

Provided by the deployment platform (AWS CloudWatch, Datadog, GCP Monitoring). Do not instrument these in application code.

- CPU utilization (%)
- Memory utilization (%)
- Disk utilization (%)
- Network throughput (bytes/sec)
- Instance health (up/down)

### Tier 2: Service Metrics (Application)

Instrument in application code. These metrics characterize the health and performance of the service itself.

**Request rate:** Requests per second or per minute. Track as a counter; compute rate in your metrics aggregator.

**Error rate:** `(error_count / total_request_count) * 100`. This is the most important service health metric. Alert threshold: >1% over any 5-minute window.

**Latency distribution:** Measure p50, p95, p99 for all request types. p50 reflects median user experience; p99 reflects tail latency; p50 alone hides tail problems.

**Active connections / concurrency:** Current number of in-flight requests. Useful for diagnosing saturation.

### Tier 3: Business Metrics (Domain)

Define these specifically for the project. They answer: "Is the business behaving correctly?" — not "Is the infrastructure healthy?"

Examples by domain:

| Domain | Business metric |
|---|---|
| E-commerce | Orders per hour, payment success rate, cart abandonment rate |
| Job processing | Queue depth, job completion rate, average job duration |
| User-facing app | New registrations per day, active sessions, feature usage rate |
| API product | API calls per minute by endpoint, API error rate by client |

Business metrics require application-level instrumentation. They are usually derived from business event logs (see above) or directly incremented as counters when the event occurs.

---

## Alerting Threshold Patterns

### Error Rate Alert

```
ALERT if: error_rate > 1% sustained over 5 minutes
ACTION: Page on-call
CONTEXT: Include service name, current error rate, top 3 error types in alert body
```

1% is a conservative threshold for established services. For a new service in its first week, use 5% and tighten as the baseline is established. Do not use an absolute error count — "more than 10 errors" is not meaningful without knowing request volume.

### Latency Alert

```
ALERT if: p99 latency > 2x baseline sustained over 5 minutes
ACTION: Page on-call
CONTEXT: Include service name, current p99, baseline p99, top affected endpoints
```

"2x baseline" requires knowing the baseline. Establish baseline during load testing or after the first stable week of production. Store the baseline value in the alert configuration.

### Queue Depth Alert

```
ALERT if: queue depth > 1,000 items AND growing
ACTION: Notify team (not page)
ESCALATE TO PAGE if: queue depth > 5,000 items
CONTEXT: Include queue name, current depth, processing rate, estimated drain time
```

Alert on queue depth plus trend. A queue of 1,000 items draining at 100/sec will be clear in 10 seconds — not an incident. A queue of 1,000 items growing at 50/sec is an incident.

### Post-Deployment Monitoring Alert

```
MONITOR for: 30 minutes after every deployment
ALERT if: error rate > 0.5% (tighter than baseline threshold)
ACTION: Notify release manager; trigger rollback review
CONTEXT: Include deployment timestamp, deployer, service version, current error rate
```

The post-deployment window uses a tighter threshold because any error rate increase immediately after a deployment is likely caused by the deployment. The deployment is the most recent change; it is the primary suspect.

---

## What Not to Log

**PII.** Email addresses, names, phone numbers, payment card data, and government ID numbers must never appear in logs. If you need to reference a user in a log, use the user_id (an opaque internal identifier).

**Secrets.** Environment variable values, API keys, tokens, passwords. If logging a configuration for debugging, log the key names, not the values.

**Verbose success paths.** "User session validated successfully" on every authenticated request produces log volume with no signal. Log the session event at INFO once on creation; do not re-log on every use.

**Framework internals.** ORM SQL queries in development are useful. In production, they fill the log stream with noise. If you need query logs for performance analysis, enable them in a time-limited debugging session, not permanently.
