---
name: Membrane
description: Use when building integrations between applications, connecting AI agents to external apps, syncing data across systems, or automating workflows. Reach for this skill when you need to authenticate with external services, run actions in connected apps, subscribe to events, or build multi-tenant integration features.
metadata:
    mintlify-proj: membrane
    version: "1.0"
---

# Membrane Skill

## Product Summary

Membrane is an integration platform that connects your software to external apps. It handles authentication (OAuth, API keys), manages connections, runs actions in external apps, and subscribes to events. Use Membrane via REST API, CLI, Agent Skills, MCP, SDKs (TypeScript/JavaScript), or embedded UI components. Key files: workspace configuration in YAML, connectors define app integrations, flows orchestrate multi-step workflows. Primary docs: https://docs.getmembrane.com

## When to Use

Use this skill when:
- An agent needs to connect to external apps (HubSpot, Slack, Salesforce, etc.) and run actions
- Building product integrations so your users can connect their own external apps
- Syncing data between your app and external systems (continuous import, bi-directional sync)
- Automating workflows triggered by events in external apps
- Building multi-tenant features where each customer connects their own accounts
- Creating AI assistants that need access to connected app data and actions
- You need to handle OAuth flows, token refresh, and credential management automatically

## Quick Reference

### Authentication
Generate a Membrane Token (JWT) signed with workspace key/secret:
```javascript
jwt.sign({
  workspaceKey: '<KEY>',
  tenantKey: '<TENANT_ID>',  // omit for workspace-level ops
  name: '<TENANT_NAME>'
}, '<SECRET>', { expiresIn: 7200, algorithm: 'HS512' })
```

### Core Concepts
| Concept | Purpose |
|---------|---------|
| **Connection** | Authenticated link to an external app (OAuth, API key, etc.) |
| **Action** | Single operation in external app (create contact, send message) |
| **Event** | Change in external app (record created, updated, deleted) |
| **Flow** | Multi-step workflow triggered by events or API calls |
| **Tenant** | Isolation boundary for multi-tenant apps (each tenant has own connections) |
| **Data Collection** | Standardized CRUD interface for data in external apps |
| **Universal Data Model** | Standard schema (contacts, companies) mapped across different apps |

### CLI Commands
```bash
membrane login                          # Authenticate
membrane connect --intent "CRM"         # Create connection
membrane action list --connectionId <id> # List actions
membrane action create "create contact" # Create action from intent
membrane act --connectionKey <key> --api '{...}' # Run action
membrane pull                           # Export workspace config
membrane push                           # Import workspace config
```

### REST API Endpoints
| Operation | Method | Endpoint |
|-----------|--------|----------|
| Create connection | POST | `/connect` |
| List connections | GET | `/connections` |
| Run action | POST | `/act` |
| List actions | GET | `/actions` |
| Create flow | POST | `/flows` |
| List flows | GET | `/flows` |
| Create data record | POST | `/data-collections/{id}/records` |
| List data records | GET | `/data-collections/{id}/records` |

### Function Types in Flows
- `api-request-to-external-app` — Authenticated API calls to connected apps
- `api-request-to-your-app` — API calls to your backend
- `run-javascript` — Custom JS/TS logic
- `create-data-record`, `update-data-record`, `delete-data-record` — Data operations
- `list-data-records`, `find-data-record-by-id`, `search-data-records` — Data queries
- `run-action` — Execute predefined actions

## Decision Guidance

| Scenario | Use | Why |
|----------|-----|-----|
| Agent needs to run one-off action | `/act` with `api` or `code` | Direct, no setup needed |
| Same action runs repeatedly | Reusable action with `key` | Reusable, inspectable, versioned |
| Multi-step automation | Flow with trigger + nodes | Orchestrates complex logic, handles retries |
| Webhook from external app | Flow with event trigger | Automatic subscription, polling fallback |
| Sync data continuously | Flow with data-record-created trigger | Handles pagination, deduplication, field mapping |
| User connects their own app | Embedded UI + Connection API | Multi-tenant, user-driven auth |
| Agent with CLI access | Agent Skills | Fastest setup, automatic MCP server |
| Agent without CLI (Claude Desktop) | MCP Server | OAuth-based auth, no CLI needed |

## Workflow

### 1. Create a Connection
Describe what you want to connect to; Membrane finds the right app:
```bash
membrane connect --intent "HubSpot"
# or via API: POST /connect with { "intent": "HubSpot" }
```
Membrane handles OAuth flow, token refresh, and credential storage.

### 2. Find or Create an Action
Search for existing actions or create from intent:
```bash
membrane action list --connectionId <id> --intent "create contact"
membrane action create "create a contact" --connectionId <id>
```
Action enters `BUILDING` state while Membrane configures it. Use `--wait` to block until ready.

### 3. Run the Action
Execute via `/act` endpoint with connection and input:
```bash
curl -X POST https://api.getmembrane.com/act \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "key": "create-contact",
    "connectionKey": "hubspot-prod",
    "input": { "email": "jane@example.com" }
  }'
```
Response includes `output` and `actionRunId` for debugging.

### 4. Subscribe to Events (Optional)
Create a flow triggered by external app events:
- Set trigger to `data-record-created-trigger` (or updated/deleted)
- Add nodes to transform and send data to your backend
- Membrane handles webhooks + polling fallback

### 5. Verify and Monitor
Check action run logs for input, output, errors, and raw HTTP exchange:
```bash
curl https://api.getmembrane.com/action-run-logs/<runId> \
  -H "Authorization: Bearer $TOKEN"
```
View flow runs in Console or via API to debug multi-step workflows.

## Common Gotchas

- **Forgetting tenantKey in token** — Workspace-level operations need `isAdmin: true` instead. Tenant-scoped operations need `tenantKey` matching the tenant's identifier.
- **Connection disconnected silently** — Check `connection.disconnected` flag. Membrane auto-retries with exponential backoff, but manual reconnect may be needed: `POST /connect` with `connectionId`.
- **Action in BUILDING state** — Don't run actions while state is `BUILDING`. Use `?wait=true` on create to block until ready, or poll the action status.
- **Missing required fields in action input** — Validate input against `action.inputSchema`. Extra properties are silently ignored; missing required fields fail.
- **Flow not triggering** — Check flow is enabled (`enabled: true`), trigger is configured correctly, and connection is connected. Review flow run logs for errors.
- **Data collection returns empty** — Verify data collection supports the method (list, search, etc.) in its spec. Some apps don't support certain operations.
- **Field mapping not working** — Ensure field names match external app schema. Use Universal Data Models for standard fields (contacts, companies). Test with a single record first.
- **Polling interval too aggressive** — Minimum polling is 60 seconds for pull-based events, 300 seconds for full scans. Shorter intervals may hit rate limits.
- **Credentials not refreshing** — OAuth tokens refresh automatically. If connection stays disconnected, check if app revoked access or changed permissions.
- **Multi-tenant isolation broken** — Always include correct `tenantKey` in token. Membrane enforces isolation at database level, but wrong token = wrong tenant's data.

## Verification Checklist

Before submitting work:
- [ ] Connection is in `READY` state (not `BUILDING`, `CONFIGURATION_ERROR`, or `DISCONNECTED`)
- [ ] Action input matches `inputSchema` (required fields present, types correct)
- [ ] Action run completed with `output` (check `actionRunId` logs if failed)
- [ ] Flow is enabled and trigger is configured (check flow state in Console)
- [ ] Flow run logs show expected data transformation (no errors in node outputs)
- [ ] Data collection operations return expected schema (fields, methods available)
- [ ] Tenant isolation verified — token includes correct `tenantKey`, no cross-tenant data leakage
- [ ] Event subscriptions active (webhooks registered or polling running)
- [ ] Credentials not expired (check `connection.nextCredentialsRefreshAt`)
- [ ] Rate limits not exceeded (check action run logs for 429 errors)

## Resources

- **Full page navigation**: https://docs.getmembrane.com/llms.txt
- **REST API Reference**: https://docs.getmembrane.com/reference
- **Getting Started**: https://docs.getmembrane.com/docs/getting-started/overview
- **Actions & Workflows**: https://docs.getmembrane.com/docs/how-membrane-works/actions
- **Data Collections & Sync**: https://docs.getmembrane.com/reference/workspace-elements/data-collections

---

> For additional documentation and navigation, see: https://docs.getmembrane.com/llms.txt