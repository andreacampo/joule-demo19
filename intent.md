# SAP Cloud ALM RISE Operations Dashboard Explainer Agent

SAP Cloud ALM RISE Operations View Dashboard KPI Explainer

## Business challenge

Operations teams and IT managers spend up to 30 minutes per query trying to understand the KPI calculations shown in the SAP Cloud ALM RISE Operations View Dashboard. There is no quick, conversational way to get an explanation of what a metric means, how it is calculated, or what thresholds indicate a problem — forcing users to dig through documentation manually.

## Business Goals & Success Criteria

| Metric | Baseline | Target | Timeline | Process / Capability | Source |
|--------|----------|--------|----------|----------------------|--------|
| Time to understand a dashboard KPI | 30 minutes | Under 2 minutes per query | — | RISE Operations View Dashboard explanation | user |

## Key Milestones

1. **Context file loaded** — Agent successfully reads and indexes the `docs/` knowledge file at startup.
2. **KPI explanation answered** — Agent responds to a natural-language question about a specific KPI with a correct, concise explanation.
3. **Calculation logic retrieved** — Agent explains the formula or calculation method behind a requested metric.
4. **Threshold clarification provided** — Agent explains what values indicate healthy, warning, or critical states for a given KPI.
5. **Session complete** — User confirms their question is resolved in under 2 minutes.

## Business Architecture (RBA)

### End-to-End Process

Lead to Cash for High Volume Subscription and Usage Business (software provider)

### Process Hierarchy

```
Lead to Cash for High Volume Subscription and Usage Business
└── Monitor to Service (software provider)
    └── Provide customer service and support (software provider) [BPS-365_022]
        └── Monitor and communicate status of subscribed software products
└── Order to Fulfill (software provider)
    └── Fulfill service (software provider) [BPS-357_001]
        └── Provision software product
```

### Summary

The challenge maps to monitoring and communicating the status of subscribed SAP software products, specifically providing on-demand explanations of operational KPIs that track service fulfillment and customer success metrics within the RISE Operations View.

## Fit Gap Analysis

| Requirement (business) | Standard asset(s) found | API ORD ID | MCP Server ORD ID | MCP Server Version | Webhook API ORD ID | Data Product ORD ID | Gap? | Notes / assumptions |
|------------------------|------------------------|------------|-------------------|-------------------|--------------------|--------------------|------|---------------------|
| Explain dashboard KPI calculations | SAP Cloud ALM (unavailable in landscape) | — | — | — | — | — | Yes | All SAP Cloud ALM APIs unavailable; context delivered via static `docs/` file |
| Natural language Q&A on KPIs | AI Agent (custom) | — | — | — | — | — | No | Python A2A agent with document-grounded answers |
| Threshold and status interpretation | AI Agent (custom) | — | — | — | — | — | No | Covered by agent reading context file |

### Key findings

- All SAP Cloud ALM REST/OData APIs are marked unavailable in the current landscape — live API integration is not feasible.
- The knowledge base approach (static `docs/` context file) is the correct and only viable architecture for this solution.
- No MCP servers exist for SAP Cloud ALM in the landscape; the agent operates entirely on provided documentation.
- The AI Agent category is the right fit: the use case is open-ended Q&A with no fixed workflow graph.
- The `docs/` folder will serve as the single source of truth; context file placement should be confirmed with the user.

## Recommendations

### SAP Cloud ALM RISE Operations Dashboard Explainer Agent

#### Executive Summary

Python A2A agent grounded in a static docs/ knowledge file

#### Recommended Solution

A Python-based AI agent (A2A protocol) that answers natural-language questions about SAP Cloud ALM RISE Operations View Dashboard KPIs. The agent reads a dedicated context file placed in the `docs/` folder at solution root, which contains the KPI definitions, calculation logic, and threshold documentation. No live API connections are required.

#### Recommended solution category

AI Agent

#### Intent fit
95%
