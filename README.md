# opspilot

## overview

When a production system starts behaving abnormally, OpsPilot should detect the problem, create an incident, investigate the available operational data, and provide engineers with an evidence-backed explanation of what might have gone wrong.

## scenario

Imagine you operate an e-commerce application.

Your application has:
- User service
- Payment service
- Order service
- Inventory service
- Notification service
At 2:00 PM, users start reporting that payments are failing.

Today, an engineer might have to:
1. Notice an alert or customer complaint.
2. Open Grafana or another monitoring dashboard.
3. Search application logs.
4. Check metrics for error rates and latency.
5. Look at recent deployments.
6. Correlate events across multiple services.
7. Identify the likely root cause.
8. Create and update an incident.
9. Communicate findings to the team.
This process can be time-consuming, especially when information is spread across different tools.

*OpsPilot aims to automate and simplify parts of this process.*

## responsbilities

Let's define its responsibilities as five capabilities.

| Capability | What it does | Example |
|---|---|---|
| 1. Data collection | Collects logs and metrics from applications | Receives payment-service errors |
| 2. Incident detection | Evaluates incoming data against alert rules | Detects payment failure rate above 10% |
| 3. Incident management | Creates and tracks incidents | Opens a Payment Service incident |
| 4. AI investigation | Investigates logs, metrics, and deployment history | Finds that failures started after a recent deployment |
| 5. Root-cause summary | Produces an explanation with supporting evidence | Reports that a recent deployment may have introduced the failure |

*the AI should not simply generate a plausible explanation. It should investigate actual data and support its conclusions with evidence.*


# functional responsibility

Plain-language requirements for what the system must do. Technical terms (Kafka, ClickHouse, OAuth2, and so on) name the chosen building blocks; the behavior below is what matters.

1. **Collect logs and metrics from other applications** (telemetry).

   Other services send operational data into OpsPilot so it can be stored, searched, and used for alerts and investigations.

   Requirements:

   - Accept telemetry over HTTP from external applications.
   - Reject data that does not match the agreed event format; accept only well-formed events.
   - Support both log-style messages and numeric metrics.
   - Require useful labels on each event: which service, which environment (e.g. production), when it happened, and how serious it is.
   - Allow sending many events in one request (batch ingestion).
   - After validation, hand off accepted events to Kafka for background processing.
   - Store accepted telemetry in ClickHouse so people and the AI can query it later.

2. **Process incoming telemetry in the background** (asynchronously).

   Ingestion should stay fast; heavy work happens on workers, not in the request that received the data.

   Requirements:

   - Read events from Kafka and process them on separate workers.
   - Do not block incident investigation or APIs while telemetry is being processed.
   - When a downstream step fails, retry where it makes sense instead of dropping data silently.
   - Treat duplicate deliveries safely so the same event twice does not corrupt state.
   - Send events that cannot be processed to a dead-letter queue (DLQ) for later inspection.
   - Keep the original timestamp and metadata on every event end to end.

3. **Let users define and manage alert rules** (“when should we worry?”).

   Engineers configure rules that describe unhealthy conditions (for example, too many payment errors).

   Requirements:

   - Create, update, read, and delete alert rules.
   - Base rules on metric values or on how many matching events occurred in a period.
   - Set a threshold (e.g. above 10%) and a time window over which to measure (e.g. last 5 minutes).
   - Turn rules on or off without deleting them.
   - Tie each rule to a specific service and environment.

   Example:

   ```
   name: payment-error-rate
   service: payment-service
   metric: http_5xx_error_rate
   condition: greater_than
   threshold: 10
   evaluation_window: 5m
   severity: critical
   ```

4. **Continuously check live telemetry against alert rules** (alert engine).

   The system watches incoming data and raises or clears alerts when rules say so.

   Requirements:

   - Evaluate rules on an ongoing basis as new telemetry arrives.
   - Support “value above/below a threshold over a time window” style conditions.
   - Notice when a situation goes from normal to alerting, and back to normal.
   - Do not spam repeated alerts for the same ongoing problem; one logical episode should not flood the channel.
   - Clear or resolve alerts when the condition is healthy again.
   - Record which service and environment each alert belongs to.

5. **Turn alerts into incidents** (tracked outages or issues).

   When rules fire, OpsPilot opens or updates an incident so the team has a single place to coordinate.

   Requirements:

   - Automatically open an incident when an alert meets the criteria for one.
   - Attach new related alerts to an existing incident when it is the same underlying issue.
   - Avoid opening multiple incidents for one continuing outage.
   - Set how severe the incident is and which service is affected.
   - Support status changes through the incident lifecycle.
   - Keep a timeline of what happened and when.
   - Record important actions taken during the incident.

   ```
     OPEN
      |
      v
   ACKNOWLEDGED
      |
      v
   RESOLVED
   ```

6. **Expose APIs for incident management** (Spring Boot application).

   Engineers and tools interact with incidents, teams, and on-call through a secured HTTP API.

   Requirements:

   - Create and fetch individual incidents.
   - List incidents with filters and page through large lists.
   - Show incident details together with linked alerts.
   - Mark incidents as acknowledged or resolved.
   - Manage teams and on-call schedules.
   - Sign users in with OAuth2 and restrict actions by role (RBAC).
   - Keep an audit trail of who changed what.

7. **Give the AI investigator controlled access to operational data** (tools, not raw database access).

   The investigation agent may only use defined operations so queries stay safe and bounded.

   Initially, the agent should have tools to:
   1. Search application logs.
   2. Query service metrics.
   3. Retrieve recent deployment history.

   Requirements:

   - Search logs by service, environment, time range, severity, and keywords.
   - Fetch metrics for a chosen service and time range.
   - List deployments within a time range.
   - Return answers in a consistent, structured format.
   - Enforce limits and timeouts so one investigation cannot overload storage or APIs.

8. **Investigate incidents using real operational data**.

   For a given incident, the system gathers evidence from logs, metrics, and deploy history, then summarizes findings without making things up.

   Requirements:

   - Start from a specific incident.
   - Load incident context (service, time, severity, linked alerts).
   - Search logs and query metrics that matter for that incident.
   - Check whether a deployment happened shortly before symptoms started.
   - Connect facts across sources (logs, metrics, deploys).
   - Suggest possible root causes with stated confidence.
   - Write an investigation summary that cites supporting evidence.
   - Clearly separate what was observed from what is only a hypothesis.
   - If evidence is missing, say so instead of inventing an explanation.

   ```
   {
   "incident_id": "INC-1001",
   "summary": "Payment API error rate increased significantly.",
   "observations": [
      "HTTP 5xx errors increased from 1% to 35%.",
      "Database timeout errors increased.",
      "A deployment occurred four minutes before the incident."
   ],
   "possible_root_causes": [
      {
         "description": "Database connection handling issue",
         "confidence": "medium",
         "evidence": [
         "Increased database timeout logs",
         "Increased payment API latency",
         "Recent deployment"
         ]
       }
   ],
   "status": "investigation_completed"
   }

   ```

9. **Attach investigation results to incidents**.

   Investigations are part of the incident record, not one-off chat output.

   Requirements:

   - Start an investigation for an incident on demand or by workflow.
   - Track whether an investigation is running, finished, or failed.
   - Save results so they can be opened later.
   - Let engineers retrieve past investigation runs for the same incident.
   - Keep history when investigations are re-run.
   - Let engineers review and validate AI-generated findings.

10. **Measure how good the investigation agent is** (evaluation harness).

    The project needs a repeatable way to score the AI against incidents whose causes are already known.

   Requirements:

   - Keep a dataset of incidents with known root causes.
   - Re-run historical or synthetic incidents through the agent.
   - Compare the agent’s conclusions to the expected causes.
   - Measure accuracy and track wrong or unsupported claims.
   - Detect regressions when the agent or its tools change.

# non functional responsibility

| ID | Category | Requirement | Proposed acceptance criteria |
|---|---|---|---|
| NFR-01 | Performance | High-throughput ingestion | Target 50,000 events/sec on one machine under a defined benchmark workload. |
| NFR-02 | Latency | Low-latency event processing | 95% of accepted events become available to the alert engine within 2 seconds. |
| NFR-03 | Scalability | Horizontal scalability | Collector and alert engine should support multiple instances. |
| NFR-04 | Reliability | No silent event loss | Accepted events must be durably queued or failures explicitly reported. |
| NFR-05 | Fault tolerance | Handle component failures | Temporary Kafka, ClickHouse, or API failures must not crash the entire pipeline. |
| NFR-06 | Backpressure | Handle traffic spikes | Bounded queues and controlled ingestion when downstream systems are slow. |
| NFR-07 | Idempotency | Prevent duplicate processing effects | Replayed events should not create duplicate incidents. |
| NFR-08 | Security | Protect APIs and data | OAuth2 authentication, RBAC, input validation, and secure service communication. |
| NFR-09 | Observability | Monitor OpsPilot itself | Expose application metrics, structured logs, health checks, and distributed traces. |
| NFR-10 | Maintainability | Modular architecture | Separate ingestion, alerting, incident management, and investigation responsibilities. |
| NFR-11 | Data integrity | Preserve event correctness | Consistent event identifiers, timestamps, and incident relationships. |
| NFR-12 | AI quality | Evidence-backed investigations | Every proposed root cause must reference retrieved evidence. |
| NFR-13 | AI reliability | Avoid unsupported conclusions | Agent must explicitly report insufficient evidence when appropriate. |
| NFR-14 | Testability | Automated verification | Unit, integration, load, failure-injection, and AI evaluation tests. |
| NFR-15 | Deployability | Reproducible deployment | Services should run using Docker Compose with documented configuration. |