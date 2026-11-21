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


# functional responsbility

1. Receive application logs and metrics (Telemetry) from external applications.

   Requirements:

   - Accepts telemetry from external source via HTTP APIs.
   - Validate incoming events from defined event schema.
   - Support structured log and metric events.
   - Attach metadata such as service name, environment,timestamp and severity.
   - Rejection of malformed events.
   - Support batch ingestion.
   - Publish accepted events to kafka.
   - Persist telemetry in clickhouse for investigation and querying.

2. System must process incomming event telemetry asynchronously 

# non functional responsibility