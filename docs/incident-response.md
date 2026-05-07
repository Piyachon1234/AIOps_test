1. Initial Triage and Assessment
Objective: Confirm the scope and validity of the alert before taking action.

Acknowledge the Alert: Claim the ticket in the incident management portal to let the team know you are actively investigating.

Verify System State: Open the primary Grafana Dashboard (http://localhost:3000/d/aiops-agent-prod-01).

Assess Impact: Look at the "Throughput by Status" panel.

Scenario A: If total traffic volume is massive and rejections have scaled linearly, the system is likely functioning correctly by blocking a malicious bot attack or a sudden burst of out-of-policy prompts.

Scenario B: If total traffic is flat but the success line drops while the rejected line spikes, the AI logic has sustained an internal failure (e.g., prompt injection vulnerability breached, upstream model degraded, or guardrail misconfiguration).

2. Investigation Steps (Commands & Queries)
Objective: Isolate the exact microservice or model causing the rejections.

Do not guess; use telemetry. Execute the following diagnostics:

2.1 Check Rejection Reasons: Open the Prometheus UI (http://localhost:9090) and run:
sum(rate(agent_rejections_total[5m])) by (reason)

This tells you exactly why the agent is failing (e.g., is it safety_violation or json_parse_error?).

2.2 Verify Kubernetes / Container Health: Ensure the container is not deadlocked.
curl -s http://localhost:8080/health

2.3 Inspect Agent Logs: Find the specific payloads failing.

docker logs agent-api --tail 200 | grep -i -E "error|rejected"

3. Decision Framework for Mitigation vs. Escalation
Objective: Restore service as rapidly as possible. Do not attempt a comprehensive root-cause fix at 3:00 AM. Focus on operational mitigation.

When to Mitigate (Rollback): If GitHub Actions shows a deployment occurred in the last 60 minutes, immediately rollback the image tag to the previous known-good Git SHA.
# Example Mitigation
export IMAGE_TAG=<previous_working_sha>
make down && make up

When to Mitigate (Traffic Shedding): If the rejection spike is due to volumetric abuse from a specific user or IP, apply API Gateway rate limits to the offending source to protect the LLM backend.

When to Escalate: If the logs reveal that the underlying AI model has suffered a total collapse (e.g., upstream provider is returning continuous 500 errors, or the model is outputting toxic content bypassing safety filters), immediately page the Tier-2 AI Core team or the Data Science on-call engineer, as this constitutes a critical security and architectural incident.

4. Post-Incident Actions
Objective: Prevent recurrence.

Preserve Data: Take screenshots of the Grafana dashboard spanning the incident window, as high-resolution metrics may be downsampled later.

Post-Mortem: Schedule a blameless post-mortem review within 48 hours.

Update the Quality Gate: Create a ticket to permanently add the specific payload or failure condition that triggered this incident to the make eval suite in the CI/CD pipeline, ensuring this specific regression cannot be merged again.