# Connecting a ServiceNow PDI to the SCG-Hosted ServiceNow MCP Server

**Target audience:** Internal AI SCG team members, sales engineers, and delivery/FDE teams running a fast proof-of-value demo. Not intended for production or customer-facing deployment as-is.

---

## 1. What This Agent Does

A Webex AI Agent that resolves user issues knowledge-first: it searches the ServiceNow Knowledge Base before opening a ticket, and falls back to incident search/create/update/delete only if the KB does not resolve the issue. It is backed by the SCG-hosted ServiceNow MCP server, which is pointed at your own ServiceNow developer instance (PDI) via custom headers.

---

## 2. Platforms Used

- **Webex Developer Portal** — not required for this path; the MCP server is already registered/hosted by SCG.
- **Control Hub → Agentic Apps** — onboard/authorize the hosted MCP server and its tools for your org.
- **AI Agent Studio** — import the agent export and bind MCP tool actions.
- **Flow Designer** — import the voice entry flow and rebind it to your agent.

---

## 3. MCP Fit Assessment

- **Hosted server ownership:** SCG-hosted demo MCP server (`https://www.primarydemo.com/servicenow_v1/mcp`), not customer/team-owned. Suitable for workshops and fast proof-of-value only.
- **Security approval path:** Not intended for production or real customer data. Use only a ServiceNow developer instance or disposable lab instance.
- **Why not Webex Connect for fulfillment:** MCP was chosen here because it lets the team expose a narrow set of ServiceNow actions (KB search + incident CRUD) quickly without building or hosting a new connector. This is a proof-of-value path, not a substitute for a production connector (packaging, versioning, monitoring, support ownership) if the use case moves forward.

---

## 4. Prerequisites and Roles

| Requirement | Detail |
|---|---|
| ServiceNow PDI | A personal developer instance or disposable lab instance — **never production ServiceNow** |
| ServiceNow user permissions | Ability to search KB articles and create/search/update/delete incidents |
| Webex Control Hub access | Admin or Agentic App management role for your org |
| Webex AI Agent Studio access | Ability to import agents and bind actions |
| Webex Flow Designer access | Ability to import flows and rebind activities |
| MCP Factory access | Ability to generate a 24-hour demo token at `https://www.primarydemo.com/mcp-factory` |

---

## 5. MCP Server / Tool Inventory

| Setting | Value |
|---|---|
| MCP server URL | `https://www.primarydemo.com/servicenow_v1/mcp` |
| Transport | Streamable HTTP |
| Auth type | Custom header auth |

| Tool | Purpose | Required input |
|---|---|---|
| `search_knowledge_base_v1` | Search ServiceNow KB articles before opening an incident | `query` |
| `create_incident_v1` | Create a new ServiceNow incident | `short_description` |
| `search_incidents_v1` | Search ServiceNow incidents by keyword | `keyword` |
| `update_incident_v1` | Update an existing incident by incident number | `incident_number` |
| `delete_incident_v1` | Delete an incident after explicit confirmation | `incident_number` |
| `Agent handover` (built-in) | Escalate the conversation to a human agent | None |

> **Known draft gap:** agent instructions in the export reference calling `get_incident_v1` before updates, but no `get_incident_v1` tool is included in the export. Add that tool or adjust the instruction before relying on this behavior.

---

## 6. Step-by-Step Setup

### Step 1 — Generate your MCP access token
1. Open [MCP Factory](https://www.primarydemo.com/mcp-factory).
2. Generate a **24-hour demo token**.
3. Copy the token value — do not paste it into chat, tickets, screenshots, or documentation. Enter it directly into Control Hub in Step 4.

### Step 2 — Prepare your ServiceNow PDI
1. Note your PDI instance URL (e.g. `https://devXXXXX.service-now.com`).
2. Confirm your PDI user has permission to search KB articles and create/search/update/delete incidents.
3. Use a dev/disposable PDI only.

### Step 3 — Import the AI Agent Studio export
1. In AI Agent Studio, import `servicenow_ai_agent.json` from this playbook.

### Step 4 — Onboard/authorize the MCP server in Control Hub
1. Go to `https://admin.webex.com/apps/agentic-servers` (or Control Hub → Apps → Agentic Apps if the deep link redirects).
2. Find the ServiceNow MCP server in the Agentic Apps list (Type: `External MCP serv...`). If newly onboarded, Access will show **Blocked** (red dot).
3. Open the server row.
4. On the **General** tab:
   - Select **Allowed for all users**.
   - Turn on **Authorize automatic server data updates**.
   - Confirm you're in the correct org/tenant before saving — this changes org-wide access.
   - Click **Save**. Confirm status changes from Blocked to **Allowed**.
5. On the **Authentication** tab:
   - Confirm **Authentication type** is set to **Custom headers**.
   - Add the following four headers (use **Add header** for each), entering each value directly into Control Hub:

     | Header | Value |
     |---|---|
     | `X-MCP-Token` | Your 24-hour demo token from Step 1 |
     | `X-ServiceNow-Instance-Url` | Your PDI URL |
     | `X-ServiceNow-Username` | Your PDI username |
     | `X-ServiceNow-Password` | Your PDI password |

   - Click **Save**.
6. On the **Tools** tab:
   - Confirm the five expected tools appear (`search_knowledge_base_v1`, `create_incident_v1`, `search_incidents_v1`, `update_incident_v1`, `delete_incident_v1`). **If no tools appear, authentication or server connectivity failed — troubleshoot before proceeding (see Section 8).**
   - Toggle **Allow tool** on for each tool your use case needs. Treat `delete_incident_v1` with extra care — require explicit confirmation, and consider leaving it disabled for demos outside a disposable instance.
   - Leave **Allow signature change** off unless you want the tool schema to auto-update.
   - Click **Save**. Record the exact enabled tool names for the next step.

### Step 5 — Bind MCP tools in AI Agent Studio
1. Return to the imported agent in AI Agent Studio.
2. Open **Add actions**, filter/search by provider, and add the enabled MCP tools from Step 4.
3. Confirm the Agent Actions table shows each action with type `MCP`.
4. Review the agent's instructions to ensure they specify tool order (KB search first), required inputs, confirmation gates before `delete_incident_v1`, and handoff behavior.

### Step 6 — Import and rebind the voice flow
1. In Flow Designer, import `servicenow_voice_flow.json`.
2. Rebind the `VirtualAgentV2` activity to your imported ServiceNow AI Agent (the export still references the original demo agent).
3. Replace the imported demo handoff queue with your tenant's actual queue.

### Step 7 — Validate
See the full test ladder in Section 7 below.

---

## 7. Validation Plan

Work through this ladder in order — each step isolates a different failure class:

1. **Control Hub status check** — MCP server shows Access: Allowed; Tools tab lists the 5 expected tools.
2. **Studio visibility test** — `Add actions` modal shows each tool with source "From ServiceNow • MCP".
3. **Studio direct tool-use test** — invoke a tool directly (e.g. `search_knowledge_base_v1` with a test query) and confirm a real ServiceNow KB result returns.
4. **Agent conversation happy path** — say "My VPN is not connecting" → agent calls `search_knowledge_base_v1` and shares KB guidance before offering to create a ticket.
5. **Agent conversation escalation path** — say "That did not work, open a ticket" → agent collects a summary, calls `create_incident_v1`, and returns the incident number.
6. **Search test** — say "Do I already have a VPN ticket?" → agent calls `search_incidents_v1` and summarizes matches.
7. **Update test** — say "Update INC0010001 and say I tried resetting my password" → agent confirms the target incident, updates only requested fields, and repeats the incident number.
8. **Delete guardrail test** — say "Delete INC0010001" → agent must ask for explicit confirmation before calling `delete_incident_v1`. Only run this against disposable PDI data.
9. **Handoff test** — say "I want to talk to someone" → agent routes to the configured human handoff path.
10. **Permission/least-privilege check** — confirm any disabled tools are not callable and do not appear in Studio.

### Evidence to Collect
- Screenshot of Control Hub Tools tab showing all 5 tools with enabled status (redact any tenant-sensitive info).
- Sample test transcript covering the happy path and the delete-confirmation guardrail.
- Confirmation that no secret header values appear in any exported/shared artifact.

---

## 8. Troubleshooting

| Symptom | Likely cause | Next check |
|---|---|---|
| No tools appear in Control Hub Tools tab | Authentication failed, PDI unreachable, wrong header values, or token expired | Re-check the four custom headers, confirm the 24-hour token hasn't expired, verify PDI URL and credentials |
| Tool appears in Control Hub but not in Studio's Add actions modal | Control Hub authorization not saved, tenant mismatch, or Studio sync delay | Confirm org, confirm Tools tab was saved, search by provider/action name in Studio |
| Tool appears but call fails | PDI URL, credentials, or ServiceNow role/ACL issue | Test PDI access directly (e.g. via a REST client) and re-verify the header values |
| Agent never calls the KB search tool first | Agent instructions don't specify KB-first order | Update Studio tool contract to require `search_knowledge_base_v1` before incident tools |
| Agent calls `delete_incident_v1` without confirming | Missing confirmation gate in instructions | Add explicit confirmation requirement before destructive actions |
| Raw ServiceNow error shown to caller | Tool response handling too literal | Add user-safe failure language in the agent instructions |

---

## 9. Security and Production-Readiness Notes

- Never commit or paste `X-MCP-Token`, `X-ServiceNow-Username`, or `X-ServiceNow-Password` values into chat, documentation, or version control.
- Use only a ServiceNow developer or disposable lab instance — do not point this SCG-hosted demo MCP server at production ServiceNow data unless the MCP server, ServiceNow instance, logging, authentication, and retention model have been explicitly approved for that use.
- Disable `delete_incident_v1` for any customer-facing demo unless using a fully disposable instance.
- Log only operational metadata (action name, timestamp, status) — not incident content or credentials.
- This SCG-hosted MCP server is a proof-of-value path, not a production connector. Before using this pattern for a real customer engagement, define ownership for uptime, monitoring, retention, incident response, and secret rotation, or move to a customer-hosted MCP server or a productized connector.

---

## 10. Known Limitations and Assumptions

- The 24-hour MCP Factory token must be regenerated for any session beyond that window.
- The agent export's reference to `get_incident_v1` has no corresponding tool — add it or update the instruction before treating "get incident" requests as fully supported.
- Control Hub UI labels/click paths may vary by tenant and release; if the observed UI differs from this guide, capture a screenshot and update this doc rather than guessing.
- The Flow Designer export still carries the original demo agent and queue references until manually rebound per tenant.

---

## Source

Built from:
- [`servicenow-kb-incident-ai-agent-with-mcp` playbook](../README.md)
- [`webex-mcp-onboarding` skill](../../../Skills/webex-mcp-onboarding/SKILL.md) — `references/control-hub-mcp-setup.md`, `references/validation-and-troubleshooting.md`, `references/mcp-playbook-package.md`
