# Documentation: Durable Orchestration with Temporal.io

This document outlines the implementation of **Temporal.io** as the core orchestration engine for our agentic-first mortgage servicing platform.

---

## 1. The Philosophy: Durable Execution

In mortgage servicing, workflows are measured in years, not milliseconds. Standard microservices are stateless and "forget" where they are if a network glitch or pod restart occurs.

**Temporal** provides **Durable Execution**: it preserves the full state of a function (the "Workflow") across process restarts, server failures, and even code deployments. In our platform, **the Loan is the Workflow.**

### Key Benefits for Mortgage Servicing

* **Zero Data Loss:** If a pod crashes during a complex escrow calculation, Temporal resumes it on a different pod at the exact same line of code.
* **Deterministic Timers:** Handles 30-day grace periods and 90-day trial modifications natively using `workflow.sleep()`.
* **Implicit State:** No more "State Machines" in a database. The state is maintained in the workflow code itself.

---

## 2. Core Components of the Temporal Stack

| Component | Description |
| --- | --- |
| **Temporal Cluster** | The backend service (Persistence, History, and Matching). We use the **Self-Hosted (K8s)** version for maximum data sovereignty. |
| **Workflow Worker** | The "Skeleton." It orchestrates the sequence of events (e.g., `BillingCycle -> LateFeeCheck -> LossMitEvaluation`). |
| **Activity Worker** | The "Muscle." These are non-deterministic tasks like calling a LangGraph agent, querying Databricks, or sending an email. |
| **Visibility (Elasticsearch)** | Allows us to query the state of all 100k+ active loans using SQL-like syntax (e.g., `Status = 'Delinquent' AND Balance > 50000`). |

---

## 3. The "Loan Lifecycle" Workflow Pattern

Every loan is initiated as a single `LoanLifecycleWorkflow`.

### The Workflow Loop

```python
# Conceptual Python SDK Example
@workflow.defn
class LoanLifecycleWorkflow:
    @workflow.run
    async def run(self, loan_id: str):
        while not self.is_paid_off:
            # Wait for next billing cycle (30 days)
            await workflow.sleep(timedelta(days=30))
            
            # Trigger Agentic Billing Activity
            result = await workflow.execute_activity(
                "ProcessMonthlyStatement", 
                args=[loan_id],
                start_to_close_timeout=timedelta(minutes=5)
            )
            
            if result.payment_missed:
                # Spawn a Child Workflow for Delinquency
                await workflow.execute_child_workflow(DelinquencyWorkflow(loan_id))

```

---

## 4. Integrating with LangGraph Agents

Temporal acts as the **Supervisor** that calls the **Agents**.

1. **Activity Trigger:** Temporal initiates an activity called `EvaluateLossMitigation`.
2. **Agent Reasoning:** The activity pod spins up a **LangGraph** instance. The agent "reasons" through the borrower's financial data stored in **Databricks**.
3. **Durable Return:** Once the agent decides (e.g., "Recommend Deferment"), it returns the result to Temporal.
4. **Audit History:** Temporal records the input, output, and the exact timestamp in its **Immutable Event History**, providing a perfect audit trail.

---

## 5. Reliability & Security (2026 Standards)

### The "Sagent-Killer" Resilience

* **Retries with Backoff:** If the Credit Bureau API is down, Temporal retries automatically for hours or days without blocking other loan processes.
* **Version Tracking:** Use **Worker Versioning** to deploy new servicing logic. Existing loans continue running on the "Old Logic" until they finish their current cycle, while new loans use the "New Logic."
* **Encryption:** All data in the Temporal History is encrypted using a **Data Converter** (with keys managed in our own KMS), ensuring borrower NPI (Non-Public Information) is never visible in the Temporal UI.

---

## 6. Monitoring & Auditing

Every action taken on a loan is recorded in the **Temporal History**. This history is:

* **Immutable:** Cannot be edited by humans or agents.
* **Traceable:** Shows exactly which Activity (Agent) made which decision and when.
* **Exportable:** History can be archived to **Databricks Delta Lake** for long-term regulatory storage and "Judge Agent" evaluation.

---
