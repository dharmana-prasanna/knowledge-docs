Here are side-by-side sequence diagrams for both use cases, drawn from the actual flows in the running app.

## Use Case 1 — REF-827261 (validation error that's really a routing failure)

### Without AI (scripted workflow)

```mermaid
sequenceDiagram
    participant Ops as Analyst
    participant Script as Scripted Workflow
    participant MCP as Tools/APIs
    Ops->>Script: Fix REF-827261
    Script->>MCP: get_payment_status
    MCP-->>Script: VALIDATION_FAILED
    Script->>MCP: get_validation_errors
    MCP-->>Script: MISSING_INTERMEDIARY_BIC
    Note over Script: Rule match: missing field -> fill it
    Script->>MCP: repair_payment_fields(intermediaryBic)
    MCP-->>Script: FIELD_SET
    Script->>MCP: retry_payment
    MCP-->>Script: FAILED (primary route DOWN)
    Script-->>Ops: Still failed -> escalate to human
    Note over Ops,MCP: Treated the SYMPTOM. Root cause never checked. No human gate.
```

### With AI + MCP

```mermaid
sequenceDiagram
    participant Ops as Analyst
    participant Agent as AI Agent (reasoning)
    participant MCP as MCP Server (governed)
    participant KB as Knowledge Base
    participant Audit as Audit Log
    Ops->>Agent: Investigate REF-827261
    Agent->>MCP: get_payment_status
    MCP-->>Agent: VALIDATION_FAILED
    Agent->>MCP: get_validation_errors
    MCP-->>Agent: MISSING_INTERMEDIARY_BIC
    Agent->>MCP: get_payment_route
    MCP-->>Agent: primary route DOWN, BIC null
    Agent->>MCP: check_screening_status
    MCP-->>Agent: CLEAR
    Agent->>KB: get_error_code_definition
    KB-->>Agent: "often a SYMPTOM of routing failure"
    Agent->>KB: check_repair_eligibility(ROUTING_PATH_DOWN)
    KB-->>Agent: repairable -> route_payment_alternative (ops)
    Note over Agent: Correlates: missing BIC is a symptom of route DOWN
    Agent-->>Ops: Root cause ROUTING_PATH_DOWN; recommend re-route (approval needed)
    Ops->>Agent: Approve
    Agent->>MCP: route_payment_alternative + approval token
    MCP->>Audit: log (guardrail validated)
    MCP-->>Agent: SUCCESS (settled via alternate)
    Agent-->>Ops: Fixed first time + full audit trail
```

**Advantage:** the script blindly fixed the symptom and failed; the agent gathered routing + knowledge evidence, identified the *true* cause, and the MCP layer enforced human approval + audit before any money moved.

## Use Case 2 — REF-994201 (ledger imbalance that's really an FX-timing mismatch)

### Without AI (scripted workflow)

```mermaid
sequenceDiagram
    participant Ops as Analyst
    participant Script as Scripted Workflow
    participant MCP as Tools/APIs
    Ops->>Script: Fix imbalance on REF-994201
    Script->>MCP: get_payment_status
    MCP-->>Script: COMPLETED / IMBALANCED
    Script->>MCP: get_ledger_entries
    MCP-->>Script: delta = 3,142
    Note over Script: Rule match: imbalance -> post balancing entry
    Script->>MCP: post_balancing_entry(3142)
    MCP-->>Script: BALANCED
    Script-->>Ops: "Fixed"
    Note over Ops,MCP: Ledger balances but suspense residual orphaned -> FX P&L misstated
```

### With AI + MCP

```mermaid
sequenceDiagram
    participant Ops as Analyst
    participant Agent as AI Agent (reasoning)
    participant MCP as MCP Server (governed)
    participant KB as Knowledge Base
    participant Audit as Audit Log
    Ops->>Agent: Investigate imbalance on REF-994201
    Agent->>MCP: get_payment_status
    MCP-->>Agent: IMBALANCED (delta 3,142)
    Agent->>MCP: get_ledger_entries
    MCP-->>Agent: debits != credits
    Agent->>MCP: get_journal_postings
    MCP-->>Agent: PRINCIPAL + FX_FEE legs
    Agent->>MCP: get_fx_rate(principal)
    MCP-->>Agent: 88.40 (EOD)
    Agent->>MCP: get_fx_rate(fee)
    MCP-->>Agent: 87.85 (stale intraday)
    Agent->>MCP: get_account_balance(SUSPENSE-FX-014)
    MCP-->>Agent: holds 3,142
    Agent->>KB: check_repair_eligibility(FX_TIMING_DELTA)
    KB-->>Agent: re-rate + sweep (ops + finance)
    Note over Agent: Root cause = FX-rate timing mismatch, not a missing posting
    Agent-->>Ops: Recommend re-rate fee leg + sweep suspense (approval needed)
    Ops->>Agent: Approve (ops + finance)
    Agent->>MCP: repost_journal_leg + token
    MCP->>Audit: log
    Agent->>MCP: sweep_suspense + token
    MCP->>Audit: log
    MCP-->>Agent: BALANCED and correct
    Agent-->>Ops: Books balanced AND FX P&L correct + audit trail
```

**Advantage:** the script made the ledger *balance* but left the books *wrong*; the agent reasoned about *why* the legs differed (comparing FX rates across systems), chose the correct re-rate-and-sweep playbook, and required dual ops+finance approval — all logged.

## The pattern both diagrams reveal

| | Without AI | With AI + MCP |
|---|---|---|
| Decision logic | fixed `if/else` symptom match | reasons over correlated evidence |
| Cross-system correlation | none (one signal) | routing/ledger/FX + knowledge base |
| Symptom vs. root cause | treats symptom | identifies true cause |
| Outcome | fails / corrupts books | correct fix, first time |
| Governance | self-acts, no gate | approval-gated + full audit |

Want me to add these diagrams to the `README.md` (e.g., a "Sequence comparison" section)? I can drop them in.
