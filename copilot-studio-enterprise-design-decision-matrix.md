# Microsoft Copilot Studio – Enterprise Design Decision Matrix

**Topics · Agents · Knowledge Sources · Connectors · Custom Connectors · MCP Servers · Authentication · Security**

![Version](https://img.shields.io/badge/v0.1-Draft%20March%202026-blue) _Validated against Microsoft Learn_

---

## 1. Core Capability Comparison

| Dimension                              | Topic                                                                                                                                                      | Agent (Generative)                                                                                                | Knowledge Source                                                               | Connector / Action                                                     | Custom Connector                                              | MCP Server (3rd party)                                                                                         | Agent 365 MCP Server                                                                                                                      |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ | ---------------------------------------------------------------------- | ------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **Primary purpose**                    | Scripted, deterministic conversation flow                                                                                                                  | Autonomous reasoning & multi-step orchestration                                                                   | Grounded Q&A from documents / data                                             | Single action against an external system                               | Action against a proprietary or unlisted API                  | Dynamic multi-tool capability bundle from external server                                                      | Enterprise-grade M365 tool bundle (Outlook, SharePoint, Teams, Dataverse) with centralized governance                                     |
| **Who controls the path?**             | Designer always                                                                                                                                            | AI decides                                                                                                        | Designer scopes content                                                        | Designer (in Topic) / AI (in Agent)                                    | Designer (in Topic) / AI (in Agent)                           | AI decides; server owner controls tool surface                                                                 | Centrally governed by Microsoft Agent 365 control plane                                                                                   |
| **Output predictability**              | Fully deterministic                                                                                                                                        | Variable — AI generated                                                                                           | Bounded by content scope                                                       | Deterministic result                                                   | Deterministic result                                          | Tool results deterministic; AI decides which tools to call                                                     | Deterministic tools; AI orchestrates selection                                                                                            |
| **Compliance / audit suitability**     | High — fully auditable, replayable                                                                                                                         | Medium — conversation transcripts & tool call logs in Purview; reasoning not deterministically replayable         | Medium — depends on source scoping & sensitivity labels                        | High — action logs available                                           | High — action logs available                                  | Medium — tool calls logged; server-side governance varies                                                      | High — full audit support with Purview; centrally governed                                                                                |
| **Maintenance burden**                 | High at scale — every flow must be manually maintained; brittle beyond ~50 topics                                                                          | Medium–High for enterprise — requires ongoing prompt tuning, evaluation pipelines, guardrails, and safety testing | Low — document is source of truth; update doc, not the agent                   | Medium — schema changes require updates                                | Medium — you own the API definition and must maintain it      | Low-Medium — server owner updates tools; Copilot Studio picks up dynamically; governance overhead required     | Low — Microsoft owns server maintenance; Agent 365 control plane handles governance                                                       |
| **Requires Generative Orchestration?** | No — works with classic or generative                                                                                                                      | Yes — must be enabled                                                                                             | No                                                                             | No (in Topic) / Yes (in Agent)                                         | No (in Topic) / Yes (in Agent)                                | Yes — required                                                                                                 | Yes — required                                                                                                                            |
| **Best for**                           | Regulated flows; escalations; structured data collection; high-risk actions requiring deterministic control; "must act as end user" with guaranteed output | Complex, variable tasks; long-tail user intent; multi-system reasoning; tasks where the path cannot be scripted   | Policy FAQs; knowledge bases; self-service Q&A where answers live in documents | Single-system read/write triggered by conversation; process automation | Internal/proprietary APIs; systems with no prebuilt connector | Exposing complex capabilities your team owns; multi-tool bundles; cross-platform reuse (beyond Copilot Studio) | M365 workloads (mail, calendar, Teams, SharePoint, Dataverse) in enterprise scenarios requiring governance, audit, and delegated identity |
| **Avoid when**                         | Task is open-ended; user intent is unpredictable; scale exceeds manageable topic count                                                                     | Every response must be scripted; compliance requires replayable output; strict regulatory controls                | Data is real-time or transactional -- use a connector instead                  | Multi-step orchestration needed -- combine with Agent                  | A prebuilt connector already exists for the system            | You can't control the server definition; tool descriptions matter for precise orchestration                    | User doesn't have M365 Copilot license (currently required); Frontier program enrollment required                                         |

---

## 2. Channel Constraints on Authentication

> The channel determines what auth is possible. A correctly configured OAuth connector still cannot delegate if the channel does not pass a user identity.

### Microsoft Teams

Full delegated end-user auth. User identity flows natively. Best channel for enterprise agents requiring user-scoped data. Entra SSO supported. Agent 365 MCP Servers supported.

### Authenticated Web Chat

End-user auth supported when configured with Entra ID. Requires custom auth setup. Transcripts and Purview logging apply only to authenticated users.

### M365 Copilot (via Agent Builder)

Delegated auth on by default. Agent operates as the signed-in M365 user. Strongest compliance posture. Governed by M365 Copilot licensing and data policies.

### Anonymous Web Chat

No user identity. Delegated auth not possible. All connector calls run as maker/service account. Purview interaction logging not available. **Do not use for user-scoped or sensitive data.**

### Direct Line / Custom Apps

Auth depends on how the embedding app passes identity. Must explicitly pass Entra tokens. Validate end-to-end before production.

### Email / SMS Channels

No interactive identity session. All actions run as service account. No user delegation. Suitable for notification-type agents only.

### SharePoint Embedded

SharePoint user context can be passed, but requires careful auth configuration. Validate that the connector chain correctly inherits the user token end-to-end.

### Autonomous / Background Agents

No user session. Must use maker/service account for all actions. Hard platform constraint -- no background delegated identity in Copilot Studio today.

---

## 3. Authentication Identity -- "Run As" Decision

> The most important governance decision. Independent of Topic vs Agent -- determined by connector capability AND channel. Both must be met for end-user delegation to work.

### Run As End User (Delegated)

- Actions respect the user's own permissions
- Natural audit trail -- actions attributed to the individual
- Required for SharePoint, OneDrive, personal calendars, user-specific CRM records
- **Requires both:** (1) channel that passes identity AND (2) connector that supports OAuth delegation
- Supported in Topics and Agents, but only on authenticated channels (Teams, authenticated webchat, M365 Copilot)
- **Hard constraint:** Autonomous/background agents cannot use delegated identity -- no interactive session exists
- CMK environments may restrict delegated credentials in generative orchestration -- verify per tenant

### Run As Maker / Service Account

- Agent acts under the maker's or a service account's identity
- Required for autonomous agents -- no user session exists (hard platform constraint)
- Use for scheduled flows, background processing, system-to-system integrations
- **Governance risk:** Over-permissioned service account can act across all users' data
- Apply least privilege, secrets rotation, audit monitoring via Sentinel
- Users may see "runs under author's identity" warning in some channels

### Mixed / Per-Connection

- Each connection inside a Power Automate flow can independently run as user or as flow owner
- A single flow can mix delegated and service-account actions in the same execution
- MCP via OAuth auth-code flow: end-user delegation supported if server configured correctly
- MCP via API key: service-level only, no per-user identity
- Agent 365 MCP Servers: act on behalf of the signed-in user when deployed on Teams/M365 Copilot

---

## 4. Connector Authentication Support

> Assumes an authenticated channel (Teams or authenticated webchat). On anonymous channels, end-user delegation is not possible regardless of connector capability.

| Connector / System                                | Supports End User (Delegated)?                              | Supports Service Account?                  | Auth Type                                   | Enterprise Notes                                                                                                                                                                                                                                                                                                                               |
| ------------------------------------------------- | ----------------------------------------------------------- | ------------------------------------------ | ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **SharePoint**                                    | Yes                                                         | Yes                                        | OAuth 2.0 / Entra ID                        | End-user preferred — SharePoint permissions model enforced natively. Service account needs explicit site/library permissions assigned.                                                                                                                                                                                                         |
| **Microsoft Teams**                               | Yes                                                         | Yes                                        | OAuth 2.0 / Entra ID                        | Posting as user vs. posting as bot are distinct — delegated auth gives human attribution in chat history.                                                                                                                                                                                                                                      |
| **Outlook / Exchange**                            | Yes                                                         | Yes (shared mailbox)                       | OAuth 2.0 / Entra ID                        | End user = sends from their own mailbox. Service account = shared/service mailbox. Can't send from another user's mailbox without an explicit delegation grant in Exchange.                                                                                                                                                                    |
| **Dataverse**                                     | Yes                                                         | Yes                                        | OAuth 2.0 / Entra ID                        | Row-level security in Dataverse applies when running as end user — strongly preferred for data governance. Service account bypasses record-sharing rules.                                                                                                                                                                                      |
| **Dynamics 365**                                  | Yes                                                         | Yes                                        | OAuth 2.0 / Entra ID                        | D365 security roles apply for delegated users. Service account must have explicit CRM roles — avoid assigning System Administrator unless absolutely required.                                                                                                                                                                                 |
| **OneDrive for Business**                         | Yes                                                         | Limited — own drive only                   | OAuth 2.0 / Entra ID                        | Service account can only access its own OneDrive. Accessing other users' files requires admin-granted delegated permissions — a governance risk to document and justify.                                                                                                                                                                       |
| **SQL Server (On-Premises)**                      | No — gateway does not support true end-user delegation      | Yes                                        | SQL auth / Windows auth via On-Prem Gateway | Even if a username is passed, it is not delegated identity — the on-premises data gateway does not propagate end-user Entra tokens. Compensate with row-level security at the database level and audit at the SQL Server layer.                                                                                                                |
| **Azure SQL**                                     | Possible with Entra ID — rarely implemented correctly       | Yes                                        | SQL auth or Entra ID                        | Delegation via Entra is technically possible but requires both the connector and Azure SQL to be configured for Entra auth — a non-trivial setup. Most enterprise deployments use service identity. Verify connector limitations before committing to this approach.                                                                           |
| **SAP ERP / SAP OData**                           | No                                                          | Yes                                        | Basic auth / SAP OAuth                      | The standard Power Platform SAP connector doesn't support end-user OAuth delegation. Always runs as a configured SAP service/technical user. User attribution has to be managed within SAP's own audit logs. High-risk: make sure the SAP service user is least-privilege.                                                                     |
| **ServiceNow**                                    | Possible via OAuth — requires explicit impersonation config | Yes                                        | Basic auth or OAuth 2.0                     | ServiceNow's "impersonate user" capability must be explicitly enabled and audited — otherwise all actions run as the service account. Validate per instance. Many organisations default to service account due to configuration complexity.                                                                                                    |
| **Salesforce**                                    | Yes — via OAuth                                             | Yes                                        | OAuth 2.0                                   | Salesforce sharing model and record-level security apply for delegated users. Service account bypasses record-level sharing — significant governance risk if used for write operations.                                                                                                                                                        |
| **Custom Connector (REST API)**                   | Yes — if API supports OAuth 2.0                             | Yes                                        | OAuth 2.0, API Key, or Basic                | Designer's choice. If backend supports Entra ID tokens, delegated auth is achievable. API key = service-level only. Best practice: design internal APIs to accept and validate Entra tokens for delegated scenarios.                                                                                                                           |
| **MCP Server (OAuth 2.0 auth code flow)**         | Yes — if server validates user tokens                       | Yes — via client credentials               | OAuth 2.0                                   | End-user delegation requires the MCP server to implement and validate OAuth auth-code flow — not just accept a token. Validate server implementation before assuming delegation works. Client credentials = service account.                                                                                                                   |
| **MCP Server (API Key only)**                     | No — shared secret, no per-user identity                    | Yes                                        | API Key                                     | API key is a shared secret — there is no per-user identity concept. Always service-level. If user attribution matters, choose an MCP server with OAuth support instead.                                                                                                                                                                        |
| **Agent 365 MCP Servers**                         | Yes -- acts as signed-in user on Teams / M365 Copilot       | Not applicable -- user-delegated by design | Entra ID (M365 Copilot license)             | Designed for delegated identity. Requires full M365 Copilot license per user and Frontier program enrollment (as of Nov 2025). Centrally governed via Agent 365 control plane with full audit support.                                                                                                                                         |
| **HTTP Action (direct, no connector)**            | No -- zero user context                                     | API key / static token only                | API Key / Static token                      | No auth context gets passed. Don't use this for user-scoped or sensitive data in production. Fine for public APIs or internal prototype/sandbox work.                                                                                                                                                                                          |
| **Power Automate Cloud Flow (called from Topic)** | Yes                                                         | Yes                                        | Entra ID (per connection in flow)           | Each connection in the flow can independently be configured to run as end user or flow owner. This is a key enterprise pattern — Topic guarantees the flow fires at the right point; individual connections enforce the right identity. CMK environments may restrict user credentials in generative orchestration — verify per tenant config. |

---

## 5. Topic vs Agent -- Auth Identity Decision Tree

### Start here: Does the action need to be attributed to a specific user for compliance, data security, or audit?

**YES -- actions must carry user identity** (SharePoint edits, CRM updates, sending mail)

> Prerequisite: Is the channel authenticated (Teams, M365 Copilot, authenticated webchat)? Does the connector support OAuth delegation?
>
> If both yes: **Topic + Connector (end user)** for deterministic flows, or **Agent with user auth** for variable tasks.
>
> If either no: **Delegation not possible.** Redesign the channel choice or compensate at the system level.

**YES -- but the system doesn't support delegation** (SAP, on-prem SQL, legacy systems)

> **Topic + Connector (service account).** Compensate with row-level security at the source system, system-level audit logging, least-privilege service account, and formal risk acceptance documentation.

**NO -- background/autonomous process, no user session**

> **Service account required -- hard platform constraint.** No user session means no delegation in Copilot Studio today. Use Agent Flows with maker credentials. Apply strict least privilege. Monitor via Sentinel. Consider HITL approval checkpoints for high-risk actions (now in preview).

**NO -- user identity not required**

> Either approach works. Simple/scripted = **Topic + Connector**. Complex/variable = **Agent + Tool**. Pure Q&A = **Knowledge Source**.

---

## 6. Primary Enterprise Architecture Pattern

### Agent → Topic (for controlled action) → Connector (enforcing identity)

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐    ┌──────────────────┐
│  User intent │ →  │    Agent     │ →  │ Calls Topic  │ →  │ Connector fires  │ →  │ System of record │
│  open-ended  │    │  reasons &   │    │ deterministic│    │  with correct    │    │   updated /      │
│  / variable  │    │   decides    │    │  compliant   │    │    identity      │    │     read         │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────────┘    └──────────────────┘
```

> **The pattern Microsoft actively promotes.** Agent handles the unpredictable; Topic enforces guardrails; Connector enforces identity. These are not alternatives -- compose them.

---

## 7. Custom Connector vs MCP Server -- Decision Guide

| Dimension                          | Custom Connector                                                                                                                                        | MCP Server (3rd party / self-owned)                                                                                                                                      | Agent 365 MCP Server                                                                                                           |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------ |
| **Who controls tool definitions?** | The Copilot Studio maker — full control over action names, descriptions, and inputs                                                                     | The server owner — maker can enable/disable tools but cannot edit names or descriptions                                                                                  | Microsoft — tools are deterministic and auditable by design; maker cannot alter descriptions                                   |
| **Orchestration precision**        | High — maker writes the descriptions that drive AI tool selection accuracy                                                                              | Medium — accuracy depends on server owner's naming; maker cannot override. Critical governance point.                                                                    | High — Microsoft-defined descriptions optimised for M365 workloads                                                             |
| **New tool discovery**             | Static — maker adds each action explicitly; nothing changes without maker action                                                                        | Dynamic — new tools on the server become available automatically in Copilot Studio. Requires explicit governance (allow/deny) to prevent unintended capability exposure. | Governed — new tools added by Microsoft follow Agent 365 release cadence; centrally managed                                    |
| **Governance risk**                | Low — no surprises; maker controls everything                                                                                                           | Higher — server owner can add new tools; without governance controls these are exposed automatically                                                                     | Low — governed by Agent 365 control plane and M365 admin policies                                                              |
| **Rich content / resources**       | Returns structured data payloads; can return URLs to content; no native file/resource abstraction                                                       | Native MCP Resource abstraction — file-like content the agent can read for reasoning (API responses, file contents). Richer than structured data alone.                  | Full MCP resource support for M365 content (docs, mail, calendar items)                                                        |
| **Cross-platform reuse**           | Power Platform only                                                                                                                                     | Any MCP-compatible client — Azure AI Foundry, GitHub Copilot, and others                                                                                                 | M365 Copilot ecosystem only — designed for Microsoft's agent infrastructure                                                    |
| **End-user auth**                  | Yes — if backend supports Entra OAuth                                                                                                                   | Yes — if server implements OAuth auth-code flow; No if API key only                                                                                                      | Yes — delegated by design; requires M365 Copilot license                                                                       |
| **Licensing requirement**          | Standard Power Platform / Copilot Studio licensing                                                                                                      | Standard Copilot Studio licensing; server hosting costs                                                                                                                  | Full M365 Copilot license per user + Frontier program enrollment (as of Nov 2025)                                              |
| **Best for**                       | Internal APIs your team owns; precise orchestration control; existing Power Platform investments; scenarios where tool description accuracy is critical | Teams who own both server and agent and want cross-platform reuse; exposing complex enterprise systems with multiple tool surfaces                                       | Enterprise M365 workloads where governance, audit, and delegated identity are non-negotiable and M365 Copilot licensing exists |

---

## 8. Security, Guardrails & Evaluation Layer

> Agents require ongoing security investment that Topics do not. First-class concern in enterprise deployments. Microsoft now provides dedicated tooling.

### Prompt Injection & Input Validation

- Agents can be manipulated through malicious inputs from users, documents, or tool responses ("indirect prompt injection")
- Topics are significantly more resistant -- scripted flow, not subject to AI reasoning manipulation
- Mitigations: scope agent instructions tightly; validate tool inputs; use Purview DLP for Copilot prompts (Public Preview, GA planned Q1 2026) to block sensitive data in prompts
- Cloud Adoption Framework mandates dedicated AI red teaming and prompt injection testing for production agents
- Third-party runtime guardrails (e.g. Noma Security via Copilot Studio external guardrails feature -- Public Preview) provide real-time detection and blocking

### Evaluation Pipelines

- Copilot Studio now includes automated agent evaluation (Public Preview) -- build test sets, run at scale, get pass/fail scores and groundedness metrics
- Mandatory for enterprise agents -- primary defence against regression when prompts, tools, or knowledge sources change
- Integrate into CI/CD pipelines (GitHub Actions / Azure DevOps) for continuous validation
- Key metrics: Exact Match, Similarity, Groundedness, Relevance, Completeness
- Re-evaluate after every model upgrade (GPT-4.1 is current default; GPT-5 family in preview -- do not use preview models in production without re-evaluating)

### Observability & Audit

- Conversation transcripts stored in Dataverse (30-day default, extendable) and searchable via Purview DSPM for AI
- Tool call logs available -- which tools fired and with what inputs
- Application Insights integration for custom telemetry on topic triggers and agent actions
- Sentinel integration for alert-based monitoring of anomalous agent behaviour
- Caveat: observable what happened; not deterministically replayable why the AI reasoned the way it did

### Tool Access Scoping

- Do not trust the model to decide access -- restrict agents to only the tools and knowledge they need
- An over-tooled agent is a security risk: wider tool surface = wider blast radius for prompt injection or misuse
- For MCP servers: use allow/deny controls to prevent unintended tool exposure; do not use "allow all" in production
- Apply HITL approval checkpoints for high-risk actions (now in Preview, Nov 2025)
- MIP sensitivity labels are now surfaced across connectors and test chat (Preview) to help prevent oversharing

---

## 9. Knowledge Sources vs Connectors -- Where to Draw the Line

| Scenario                                                           | Use Knowledge Source                       | Use Connector / Action         | Why It Matters                                                                                                                                             |
| ------------------------------------------------------------------ | ------------------------------------------ | ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| "What is our expense policy?"                                      | Yes -- the document is the answer          | No                             | Static answer that lives in a document. No system to query. Knowledge Source keeps it current automatically when the doc changes.                          |
| "What is my current leave balance?"                                | No -- live, user-specific data             | Yes -- call HR system          | Real-time transactional query. Knowledge Sources cannot hold live, per-user data. Connector required at runtime.                                           |
| "Summarise recent changes to our travel policy"                    | Yes -- if policy is in SharePoint/OneDrive | No                             | SharePoint as a Knowledge Source means the agent always works from the latest version. Building this as a Topic or connector call is unnecessary overhead. |
| "Submit a purchase order for £5,000"                               | No -- write operation                      | Yes -- ERP connector via Topic | Write operations must not go through Knowledge Sources. Topic provides the deterministic, auditable flow; Connector executes with correct identity.        |
| "Show me all open Salesforce opportunities above £100k"            | No -- live transactional data              | Yes -- Salesforce connector    | Live CRM data cannot be grounded in a document. Connector call respects Salesforce sharing model if running as end user.                                   |
| Combination: "Explain our procurement policy and submit a request" | Yes -- for policy explanation              | Yes -- for submission action   | Agent orchestrates both: Knowledge Source for Q&A; Topic + Connector for the action.                                                                       |

---

## 10. Enterprise Design Rules of Thumb

| #   | Rule                                                                                                              | Rationale                                                                                                                                                                                                                                         |
| --- | ----------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | **If you can draw a flowchart for it, it's a Topic. If you can't, it's an Agent.**                                | Topics handle known paths; Agents handle unknown ones.                                                                                                                                                                                            |
| 2   | **Lock regulated, high-risk actions behind Topics -- even when an Agent handles the conversation around it.**     | Topics can be called from Agents. Compose them: Agent understands intent, Topic executes the controlled action, Connector enforces identity. This is the primary enterprise pattern.                                                              |
| 3   | **Prefer end-user delegated auth wherever supported and a user session exists.**                                  | Delegation means the user's own permissions are the guardrail. Service accounts that can act on behalf of all users are a governance and breach risk.                                                                                             |
| 4   | **For systems that don't support delegation (SAP, on-prem SQL), compensate at the source system.**                | Cannot fix the auth model at the connector level. Push row-level security, logging, and access control down to the database or ERP.                                                                                                               |
| 5   | **The channel determines whether delegation is possible -- check this before designing the auth model.**          | The best OAuth connector cannot delegate identity on an anonymous channel. Channel choice is an auth decision, not just a UX decision.                                                                                                            |
| 6   | **Never over-scope a service account used by an autonomous agent.**                                               | Autonomous agents have no user oversight. A wide-permission service account + autonomous AI = significant risk surface. Least privilege and monitoring are non-negotiable.                                                                        |
| 7   | **Prefer Custom Connectors over 3rd-party MCP when orchestration precision and tool description control matter.** | Custom Connectors: you write the descriptions that drive AI tool selection. 3rd-party MCP: you inherit the server owner's descriptions and cannot change them. Description quality directly affects tool selection reliability.                   |
| 8   | **Apply allow/deny governance to MCP servers. Never use "allow all" in production.**                              | MCP tools can appear dynamically as server owners add them. Without governance controls, new capabilities are automatically exposed to the AI -- unacceptable in regulated environments.                                                          |
| 9   | **Use Knowledge Sources for "find the answer in a document." Use Connectors for anything transactional.**         | Mixing these creates brittle solutions. Knowledge Sources scale; hard-coded topic Q&A does not. Live data needs a Connector; static knowledge needs a Knowledge Source.                                                                           |
| 10  | **Treat Agent evaluation pipelines as a production requirement, not a nice-to-have.**                             | Unlike Topics (where behaviour is scripted), Agent behaviour can drift when prompts, tools, knowledge sources, or models change. Automated evaluation is the only scalable defence against silent regression.                                     |
| 11  | **Always configure system event Topics (escalation, fallback, error). Never leave these to the Agent.**           | Safety net. If the AI fails or cannot handle a scenario, the fallback path must be human-designed, predictable, and compliant.                                                                                                                    |
| 12  | **Apply HITL checkpoints for autonomous agents taking high-risk actions.**                                        | HITL (now in Preview) allows agents to pause and request human approval before proceeding. Use for financial authorisation, procurement, data deletion, or any irreversible action. Governance bridge between full autonomy and full determinism. |

---

_Enterprise Copilot Studio guidance v0.1 (Draft) | Validated against Microsoft Learn, Microsoft Cloud Adoption Framework, and Microsoft Ignite 2025 announcements | March 2026_

**Key sources:** [learn.microsoft.com/microsoft-copilot-studio](https://learn.microsoft.com/microsoft-copilot-studio) · [learn.microsoft.com/azure/cloud-adoption-framework/ai-agents](https://learn.microsoft.com/azure/cloud-adoption-framework/ai-agents) · [techcommunity.microsoft.com](https://techcommunity.microsoft.com)
