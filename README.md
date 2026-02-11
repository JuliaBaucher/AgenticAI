# AgenticAI Hybrid Orchestration Framework  
**Agent with Tools + Template-Based AWS Orchestration**  
Accessible via a single entry point:  

```bash
kiro-cli chat --agent mcps
```

---

# 1. Overview

AgenticAI is a hybrid automation framework that combines:

1. **Dynamic Agent-Based Tool Orchestration** (MCP-powered)
2. **Deterministic Template-Based Orchestration** (AWS Step Functions + Lambda)

Both execution models are available from the same Q CLI interface (`kiro-cli chat --agent mcps`). The user does not switch systems. The system dynamically decides whether a request is handled by:

- The **LLM agent using tools**, or  
- A **durable, template-driven AWS workflow**

This architecture balances flexibility and reasoning power with production-grade control, reliability, and governance.

---

# 2. High-Level Architecture

```
User
  ↓
kiro-cli chat --agent mcps
  ↓
mcps agent (multi-system agent)
  ↓
+---------------------------------------------+
| MCP Servers                                 |
| - gmail                                     |
| - microsoft (Outlook Graph)                 |
| - jira-rovo                                 |
| - motherduck                                |
| - n8n                                       |
| - AgenticAI (custom orchestration MCP)      |
+---------------------------------------------+
                     ↓
              AgenticAI MCP
                     ↓
           AWS Step Functions
                     ↓
              Lambda Workers
                     ↓
External Systems (Graph, Jira, Bedrock, etc.)
```

---

# 3. Dual Execution Model

## 3.1 Agent with Tools (Dynamic Orchestration)

The original framework provides a multi-system agent that:

- Uses MCP servers (Gmail, Outlook, Jira, MotherDuck, n8n)
- Allows tool composition inside a reasoning loop
- Enables cross-system queries and reporting
- Generates documentation and narrative outputs

Characteristics:

- LLM-driven tool selection  
- Context-based reasoning  
- Prompt-controlled behavior  
- Session-based memory  
- Non-durable execution  

This mode is ideal for:
- Data research
- Analytics
- Narrative reporting
- Cross-system exploration

However, it does not guarantee:
- Durable step state
- Explicit execution order
- Retry isolation
- Strict side-effect control

For structured workflows, a second layer is used.

---

## 3.2 Template-Based Orchestration (Deterministic Control Plane)

Structured business workflows (e.g., intake, approvals, triage) are implemented using:

- **AWS Step Functions** (execution authority)
- **Lambda workers** (atomic execution units)
- **IAM roles** (security boundaries)
- **Secrets Manager** (credential isolation)

Each workflow is a **state machine template**.  
Each execution is a durable instance of that template.

This provides:

- Explicit step ordering  
- Named states  
- Per-step retry logic  
- Catch & fallback paths  
- Durable execution state  
- Execution history & audit trail  
- Idempotency via execution naming  

---

# 4. AgenticAI MCP Integration in Q CLI

## 4.1 What AgenticAI MCP Does

AgenticAI MCP is a custom MCP server integrated into `mcps.json`.  
It exposes three tools to the agent:

- `list_workflows`
- `execute_workflow`
- `get_status`

When the user says:

> “Run Intake workflow”

The LLM calls:

```
@AgenticAI/execute_workflow
```

Which triggers:

```
StepFunctions.start_execution(...)
```

The workflow then runs entirely inside AWS.

---

## 4.2 Local MCP Implementation (Current)

Current implementation:

- Python MCP server
- Runs locally via STDIO
- Uses `boto3` to call Step Functions
- Launched automatically by Q CLI

Characteristics:

- Requires local AWS credentials
- Simple to debug
- No additional infrastructure
- Suitable for development / controlled environments

Flow:

```
Q CLI → Local AgenticAI MCP → AWS Step Functions
```

---

## 4.3 Remote MCP (Production-Grade Alternative)

In enterprise deployment, AgenticAI MCP can be:

- Containerized (ECS/Fargate)
- Exposed via API Gateway
- Registered as remote MCP
- Secured via IAM or OAuth

Flow:

```
Q CLI → Remote MCP API → Step Functions
```

Advantages:

- No local AWS credentials required
- Centralized logging
- Multi-user environment
- Horizontal scalability
- Stronger governance

Trade-off:

- Additional infrastructure complexity

---

# 5. Lambda Workers Design

Each Lambda worker:

- Performs one atomic action
- Is stateless
- Has explicit timeout
- Uses minimal IAM privileges
- Reads secrets only from Secrets Manager

Workers do NOT:
- Decide execution order
- Implement retry policies
- Manage branching logic

All control logic resides in Step Functions.

---

# 6. Example Workflows

## 6.1 Intake Workflow (Deterministic)

Steps:

1. Fetch email (Microsoft Graph)
2. Analyze content
3. If "requirement" present → create Jira ticket
4. Else → request missing info
5. Catch & fallback if Jira fails
6. Succeed or fail explicitly

Characteristics:

- Execution name = messageId (idempotent)
- Retry only failed step
- Explicit branching
- Durable execution state

---

## 6.2 AI-Powered Triage Workflow (Scalable Extension)

New workflow example:

Instead of checking only for the word “requirement”, use **Amazon Bedrock agent** for intelligent email triage.

Steps:

1. Fetch email
2. Invoke Bedrock Agent for classification
3. Based on AI output:
   - Create ticket
   - Route to different team
   - Request clarification
4. Persist decision
5. Audit output

This is implemented as:

- New Step Function template
- Reuse existing fetch and Jira workers
- Add a Bedrock worker

---

# 7. Managing Boundaries for Bedrock Agent

Introducing AI inside deterministic orchestration requires strict boundaries.

Boundaries are enforced at multiple layers:

### 7.1 Prompt-Level Boundaries
- Structured system prompt
- Explicit allowed output schema
- No free-form execution authority
- AI cannot call external systems directly

### 7.2 Schema Enforcement
- Lambda validates Bedrock output
- Only expected fields allowed
- Invalid outputs fail the step

### 7.3 IAM Boundaries
- Bedrock worker role limited to:
  ```
  bedrock:InvokeModel
  ```
- No write access to Jira or Graph
- Separation of reasoning and execution

### 7.4 Step Function Authority
- AI only influences branching
- Cannot alter workflow order
- Cannot bypass approval states

This ensures:

> AI proposes — system enforces.

---

# 8. Alignment with Orchestration Principles

## Execution Authority
- Step Functions define explicit step order
- LLM cannot reorder steps
- Named states ensure clarity

## Boundaries
- Tool allow-list in `mcps.json`
- IAM-scoped Lambda roles
- Secrets isolated in Secrets Manager
- Prompt-controlled Bedrock usage

## Shared Memory & State
- Durable state stored in Step Functions
- Execution context passed between steps
- Local agent context isolated from AWS workflow state

## Error Handling
- Per-step retries
- Catch blocks
- Fallback states
- Controlled failure paths

## Policies & Permissions
- IAM least-privilege model
- Explicit Lambda invoke permissions
- Bedrock invocation restricted
- No implicit cross-service access

## Observability & Auditability
- Step Functions execution history
- CloudWatch logs per worker
- Input/output trace capture
- Retry metadata
- Execution replay capability

## Human-in-the-Loop
- Optional approval states
- Callback pattern supported
- Manual override via Fail states
- Escalation branch support

## Compliance & Governance
- Region-controlled deployment (e.g., eu-central-1)
- Secrets centrally managed
- Execution logs retained
- Explicit business rules encoded in ASL
- No hidden AI side effects

---

# 9. Scalability Model

Scalability is achieved via:

- Lambda auto-scaling
- Step Functions concurrency
- Stateless workers
- Decoupled control plane

### Adding New Workflows

Each new workflow:

- Has its own Step Function definition
- Reuses existing workers where possible
- Adds new workers only if needed

Example:

| Workflow | Step Function | Workers |
|-----------|---------------|----------|
| Intake | agenticAI_Intake | fetch, check, create_jira |
| AI Triage | agenticAI_Triage | fetch, bedrock_classify, create_jira |

This ensures:

- Clear separation of execution models
- Reusable components
- Explicit versioning per workflow
- No monolithic orchestration logic

---

# 10. Why This Hybrid Architecture Matters

Pure Agent Systems:
- Flexible
- Hard to govern
- Hard to audit
- Risk of uncontrolled side effects

Pure Workflow Systems:
- Deterministic
- Rigid
- Limited reasoning capability

AgenticAI combines:

| Agent Strength | Workflow Strength |
|----------------|------------------|
| Cross-tool reasoning | Deterministic order |
| AI narrative generation | Durable state |
| Flexible exploration | Retry control |
| Contextual adaptation | Governance enforcement |

Result:

> Controlled agent autonomy inside deterministic orchestration.

---

# 11. Summary

This framework delivers:

- Single CLI entry point
- Multi-system tool orchestration
- Durable AWS-backed workflow control
- Secure credential management
- Explicit IAM boundaries
- AI integration with guardrails
- Horizontal scalability
- Enterprise-grade observability
- Compliance-ready design

It is designed for environments where:

- AI must operate within policy constraints
- Business workflows require durability
- Human oversight may be required
- Side effects must be controlled
- Governance and auditability are mandatory

AgenticAI is not just an automation system.  
It is a structured, policy-aligned, AI-enabled orchestration platform.
