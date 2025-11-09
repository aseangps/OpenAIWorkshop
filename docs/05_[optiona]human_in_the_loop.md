# Human-in-the-Loop (HITL) — Fraud Detection Workflow

Continue on in this optional task for advanced users: explore the human-in-the-loop concept in 
`agentic_ai/workflow/fraud_detection/` folder. 

## Workflow details

`fraud_detection_workflow.py` implements a complete, end-to-end fraud detection and escalation
workflow for Contoso. It demonstrates a typical enterprise pattern for combining automated
specialist analysis agents, an LLM-based aggregator, and optional human review when risk is high.

High-level flow:

- An upstream monitoring system emits a `SuspiciousActivityAlert`.

- `AlertRouterExecutor` fans the alert out to three specialist executors:
  - `UsagePatternExecutor` — analyzes usage patterns (data spikes, unexpected traffic).
  - `LocationAnalysisExecutor` — inspects geolocation and authentication logs for impossible-travel or multi-country logins.
  - `BillingChargeExecutor` — reviews billing history and charges for suspicious purchases.

- The three specialists stream back analysis results (risk indicators and scores) to the
  `FraudRiskAggregatorExecutor`.

Note on parsing: the current implementation in the workflow extracts `RISK_SCORE` and
`RISK_INDICATORS` by simple line-based text parsing of the agent responses. The aggregator
also parses `OVERALL_RISK_SCORE`, `RISK_LEVEL`, and `RECOMMENDED_ACTION` using similar
string matching. This is fragile for production — prefer structured outputs (JSON/schema)
from agents when possible.

- The aggregator synthesizes the results into a single `FraudRiskAssessment` (overall risk score,
  risk level, recommended action and reasoning).

- A `ReviewGatewayExecutor` decides whether to route the assessment for human review (if risk
  ≥ 0.6) or to proceed with automatic clearing. The workflow's switch/case uses a threshold of
  0.6 to treat assessments with overall_risk_score >= 0.6 as high risk. The aggregator prompt in
  the workflow also suggests the following risk-level buckets: low (<0.3), medium (0.3-0.6),
  high (0.6-0.8), critical (>0.8).

- If human review is required, a `RequestInfoExecutor` pauses the workflow and sends a review
  request to a fraud analyst. The workflow is checkpointed while waiting for the analyst's
  decision. After the analyst responds, the workflow resumes and executes the selected mitigation
  action, then sends final notification and audit logging.

## File-level logic and structure

Key components in the file:

- Data models: Lightweight dataclasses like `SuspiciousActivityAlert`, `UsageAnalysisResult`, and
  `FraudRiskAssessment` define the payloads that flow between executors. Dataclasses are used
  here for simple serialization compatibility with checkpoint storage.

- Executors: Each executor is a small unit of computation with a single responsibility (routing,
  specialist analysis, aggregation, gateway logic, action execution, final notification). Executors
  are decorated with `@handler` and communicate by sending typed messages through the workflow
  runtime.

- Fan-out / Fan-in: The file demonstrates a fan-out pattern (one alert → many analysts) and a
  corresponding fan-in (many analysts → aggregator). The builder creates these edges using
  `WorkflowBuilder.add_fan_out_edges` and `add_fan_in_edges`.

 - Human-in-the-loop (RequestInfoExecutor): When the aggregator determines the assessment is high
   risk, the `ReviewGatewayExecutor` triggers a `RequestInfoExecutor`. In the sample workflow the
   human-review executor is created with id `analyst_review`. This executor emits a `RequestInfoEvent`
   that represents a pause in automated processing and awaits a human's response. The example
   console app captures the checkpoint id (internally available as `workflow._runner_context._last_checkpoint_id`)
   when checkpointing is enabled. By default the sample uses `FileCheckpointStorage("./checkpoints/fraud_detection")`
   to persist checkpoints. Once an analyst provides input, the example resumes the workflow by
   sending the collected responses with `workflow.send_responses_streaming(responses)` (the sample
   builds a `responses` map of request_id -> `AnalystDecision` and resumes the runner).

- Checkpointing: The workflow optionally configures a checkpoint store (e.g. `FileCheckpointStorage`)
  so long-running workflows can persist state while waiting for human input or external events.

## Workflow vs Agent — what's the difference?

- Agent (single LLM agent): Typically encapsulates a single conversational or reasoning task. An
  agent manages its own prompt, can call tools, and returns a response. Agents are useful for
  point-in-time tasks such as summarization, answering a question, or performing a tool-backed
  lookup.

- Workflow (orchestration of executors): Represents business-level control flow across multiple
  steps, branches, and participants. The workflow coordinates multiple agents and non-LLM logic,
  implements fan-out/fan-in patterns, applies conditional routing (switch/case), and supports
  long-running, checkpointed operations that may wait for external inputs (like human review).

Why use a workflow here:

- The fraud detection scenario requires multiple specialist analyses (parallel work) and an
  aggregator to make a business decision — a single agent would struggle to handle coordination,
  stateful branching, and checkpointing reliably.

- Checkpointing and pause/resume semantics are critical when waiting for offline human analysts
  to respond.

- Workflows make control flow explicit, auditable, and easier to test end-to-end.

## Importance of Human-in-the-Loop (HITL)

Human review is included for several reasons:

- Safety and Accountability: High-risk decisions (locking accounts, reversing charges) have
  customer impact and legal/regulatory implications. Human analysts provide an approval gate that
  ensures accountability.

- Edge Cases & Context: LLMs and automated tools can miss contextual signals or misinterpret
  business rules. Humans can use additional context (phone calls, cross-team info) to make better
  decisions.

- Explainability: Analysts can review the aggregated reasoning and either confirm or override the
  recommended action while adding audit notes that become part of the record.

- Compliance & Audit Trail: Human approvals and analyst notes provide a clear, auditable trail
  required for investigations and compliance reviews.

## This is optional — try the Quickstart to experiment

Human review flows are optional for experimentation. If you want to test implementing the human
review code path or run the scenario locally, follow the Quickstart for the fraud detection
scenario. The repository includes a `QUICKSTART.md` in the same workflow folder that contains the
step-by-step instructions for running the workflow and simulating analyst responses.

- Modify the `ReviewGatewayExecutor` threshold or the aggregator's risk-to-action mapping to test
  different policies. Note: the current default high-risk threshold in the workflow is 0.6.

## Where to look next

- The workflow implementation: `agentic_ai/workflow/fraud_detection/fraud_detection_workflow.py`.
- Quickstart / run instructions: `agentic_ai/workflow/fraud_detection/QUICKSTART.md` (or the
  `QUICKSTART.md` in the same folder) — this file includes runnable examples for testing the
  human-in-the-loop pause/resume path.
- Checkpoint storage: Look for usages of `FileCheckpointStorage` in the workflow to learn how
  checkpointing is wired.

## Summary

- `fraud_detection_workflow.py` orchestrates parallel specialist agents, aggregates results with
  an LLM, and optionally pauses for human review before taking high-impact remediation steps.
- Use workflows (not single agents) for long-running, stateful business processes that require
  branching, parallelism, checkpointing, and human oversight.
- Human-in-the-loop is optional but recommended for high-impact actions; test it using the
  provided `QUICKSTART.md` in the workflow directory.
