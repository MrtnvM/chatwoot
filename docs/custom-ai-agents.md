# Building custom AI agents on Chatwoot self-hosted

Chatwoot's open-source Community Edition provides everything needed to build sophisticated AI agent systems — **unlimited agents, inboxes, and conversations** under the MIT license, a native AgentBot framework with webhook-driven event delivery, and a full REST API covering conversations, messages, contacts, and assignments. The most effective architecture uses Chatwoot's AgentBot system, which automatically routes new conversations to your external AI service via webhooks and lets you respond through the API, with a clean `pending` → `open` status toggle for bot-to-human handoff. This guide covers every component: API surface, licensing boundaries, deployment in Coolify, architectural patterns, and real community projects you can build on.

---

## Chatwoot exposes three API tiers and a native bot framework

Chatwoot provides **three distinct API categories**, each with different authentication and scope. Understanding which to use is critical for AI integration.

| API type | Base path | Auth method | Purpose |
|---|---|---|---|
| **Application API** | `/api/v1/accounts/{account_id}/…` | `userApiKey` or `agentBotApiKey` | Account-level CRUD: conversations, messages, contacts, teams, labels |
| **Platform API** | `/platform/api/v1/…` | `platformAppApiKey` | Installation-wide admin: create accounts, users, global bots. **Self-hosted only.** |
| **Client API** | `/public/api/v1/inboxes/{inbox_id}/…` | `inbox_identifier` + `contact_identifier` | Customer-facing: create contacts, conversations, messages from widget side |

All APIs authenticate via the `api_access_token` header. For AI agent integrations, **the Application API with an AgentBot token is the primary interface**. The AgentBot receives webhook events at its configured `outgoing_url` and responds via the Application API using its dedicated access token.

### Key endpoints for AI integrations

The endpoints most critical for building AI agents include:

- **Messages**: `POST /api/v1/accounts/{id}/conversations/{conv_id}/messages` — send bot replies with `message_type: "outgoing"`. Supports `content_type` values of `text`, `input_select` (quick-reply buttons), `cards` (carousels), and `form`.
- **Status toggle**: `POST …/conversations/{conv_id}/toggle_status` — switch between `pending` (bot handling), `open` (human handling), `resolved`, and `snoozed`. This is the handoff mechanism.
- **Assignments**: `POST …/conversations/{conv_id}/assignments` — assign to a specific agent (`assignee_id`) or team (`team_id`).
- **Typing indicators**: `POST …/conversations/{conv_id}/toggle_typing_status` — show/hide typing bubbles during LLM processing.
- **Labels**: `POST …/conversations/{conv_id}/labels` — add classification labels (overwrites existing; requires `userApiKey`).
- **Custom attributes**: `POST …/conversations/{conv_id}/custom_attributes` — store AI metadata like intent, confidence scores (requires `userApiKey`).
- **Conversation history**: `GET …/conversations/{conv_id}/messages` — fetch full message history for LLM context window (requires `userApiKey`).

### Bot tokens have restricted endpoint access

A critical design constraint: **AgentBot tokens can only access a subset of endpoints**. From Chatwoot's source code (`access_token_auth_helper.rb`):

| Operation | Bot token | User token |
|---|---|---|
| Create message | ✅ | ✅ |
| Toggle status (handoff) | ✅ | ✅ |
| Toggle typing indicator | ✅ | ✅ |
| Toggle priority | ✅ | ✅ |
| Assign conversation | ✅ | ✅ |
| Add/update labels | ❌ | ✅ |
| Get message history | ❌ | ✅ |
| Update custom attributes | ❌ | ✅ |

**Practical recommendation**: maintain a **dual-token strategy** — use the bot's `access_token` for message sending and status changes, and a dedicated admin user's `api_access_token` for label management, conversation history retrieval, and custom attribute updates. Store both securely in environment variables.

### Rate limiting is configurable

Chatwoot uses Rack::Attack for rate limiting, defaulting to **300 requests per minute** per IP. For self-hosted deployments, this is fully configurable via environment variables: `RACK_ATTACK_LIMIT=300`, `ENABLE_RACK_ATTACK=true/false`, and `RACK_ATTACK_ALLOWED_IPS` for trusted IPs that bypass throttling. The Reports API has a separate limit of **1,000 requests per minute** per account.

---

## The AgentBot framework is your primary AI integration point

Chatwoot has a **native AgentBot concept** specifically designed for connecting external bot services. This is not a workaround — it's a first-class feature available in the free Community Edition.

### How AgentBots work

An AgentBot is an external web service registered in Chatwoot with a name and a webhook URL (`outgoing_url`). When you connect a bot to an inbox, the system changes behavior: **all new conversations in that inbox start with `pending` status** instead of `open`. Chatwoot sends webhook events (`message_created`, `webwidget_triggered`, `message_updated`) to your bot's URL. Your service processes these events, calls an LLM or any logic you want, and responds via the Chatwoot API.

Two types exist: **Account Bots** (scoped to one account, created via Settings → Bots or the Application API) and **Global Bots** (scoped to the entire installation, created via the Platform API). For most AI agent use cases, Account Bots are sufficient.

### Creating and connecting an AgentBot

Create a bot via the Application API:

```bash
POST /api/v1/accounts/{account_id}/agent_bots
Header: api_access_token: <admin_user_token>
Body: {
  "name": "AI Support Agent",
  "description": "LLM-powered auto-responder",
  "outgoing_url": "https://your-ai-service.com/chatwoot-webhook"
}
```

The response includes an `access_token` — **save this immediately**, as it's how your bot authenticates API calls back to Chatwoot. Then connect the bot to a specific inbox via the Chatwoot UI (Inbox Settings → Bot Configuration) or via the Rails console with `AgentBotInbox.create!(inbox: Inbox.find(id), agent_bot: bot)`.

### Bot-to-human handoff is built in

The handoff mechanism uses conversation status transitions. When the bot determines it cannot handle a query, it calls `toggle_status` with `{"status": "open"}`, which moves the conversation from the bot queue to the human agent queue. Normal auto-assignment or round-robin takes over. **Agents can return conversations to the bot** by setting status back to `pending` — useful for multi-step workflows where the bot handles routine steps and humans handle exceptions.

---

## Webhook events and payload structures

Chatwoot's webhook system delivers real-time events via HTTP POST. You can configure **account-level webhooks** (Settings → Integrations → Webhooks, firing for all inboxes) or use the **AgentBot's `outgoing_url`** (per-inbox events only). Both support HMAC-SHA256 signature verification via the `X-Chatwoot-Signature` header.

Available events include `message_created`, `message_updated`, `conversation_created`, `conversation_updated`, `conversation_status_changed`, `webwidget_triggered`, and typing indicator events. The most important for AI agents is `message_created`, which delivers:

```json
{
  "event": "message_created",
  "id": "1",
  "content": "Hi, I need help with my order",
  "message_type": "incoming",
  "content_type": "text",
  "private": false,
  "sender": {"id": "1", "name": "John", "type": "contact"},
  "conversation": {"display_id": "42", "status": "pending", "inbox_id": 1},
  "account": {"id": "1", "name": "My Company"},
  "inbox": {"id": 1, "name": "Website Chat"}
}
```

The `conversation_updated` event includes a `changed_attributes` array showing previous and current values — essential for detecting status changes, assignment changes, or label updates. **Always verify webhook signatures** using `sha256=HMAC-SHA256(secret, "{timestamp}.{body}")` against the `X-Chatwoot-Signature` header.

---

## Community Edition licensing gives you full freedom to build

Chatwoot uses a **dual-licensing model**: everything outside the `/enterprise/` directory is **MIT-licensed** (maximum permissive freedom), while the `/enterprise/` directory is proprietary. This has been stable since version 2.0, and Chatwoot has publicly pledged never to move CE features into EE.

### What the free Community Edition includes for AI builders

**No artificial limits on self-hosted CE**: unlimited agents, inboxes, conversations, and data retention. The restrictive limits (2 agents, 1 inbox, 500 conversations/month) apply only to the free cloud "Hacker" tier, not self-hosted. The complete feature set available in CE includes:

- Full AgentBot framework with webhook delivery
- Complete REST API with no endpoint restrictions
- Account-level webhooks with all event types
- Automation rules for rule-based routing
- Custom attributes, labels, teams, and canned responses
- All messaging channels (website, email, WhatsApp, Telegram, Facebook, Instagram, SMS, API channel)
- Help Center / knowledge base
- Full reporting suite (agent, inbox, label, team, CSAT reports)
- Dialogflow native integration
- Slack integration and Dashboard Apps

### What requires a paid plan

**Captain AI** (Chatwoot's built-in AI assistant, copilot, and memory system) requires the Premium Support plan ($19/agent/month) or Enterprise plan ($99/agent/month). Captain provides built-in RAG-powered auto-responses, agent copilot suggestions, conversation summarization, and label suggestions using your own OpenAI API key (BYOK model). **However, you can build equivalent functionality yourself** using the AgentBot framework + your own LLM backend — which is exactly what this guide enables.

Other EE-only features include SSO/SAML, advanced roles and permissions, SLA policies, audit logs, agent capacity management, and custom branding (removing "Powered by Chatwoot"). None of these restrict AI agent development.

Under the MIT license, you **can** build commercial products, SaaS offerings, and derivative works on Chatwoot CE. You must include the MIT copyright notice. The "Chatwoot" trademark is protected — use your own branding for products built on top. Telemetry is enabled by default (anonymized metadata to `hub.chatwoot.com`) but can be disabled with `DISABLE_TELEMETRY=true`.

---

## Architectural patterns for four AI agent use cases

### Pattern A: LLM-powered auto-responder

The foundational pattern uses a webhook-driven pipeline with a message queue for reliability:

```
Customer → Chatwoot → Webhook → Queue (Redis) → AI Worker → LLM → Chatwoot API → Customer
```

The critical implementation details: **filter incoming messages** to prevent response loops (only process `message_type: "incoming"`, non-private, `pending` status conversations). Fetch conversation history via the user token to build the LLM context window. Toggle typing indicator ON before the LLM call, OFF after. Send the response via the bot token. For reliability, your webhook endpoint should **immediately enqueue and return 200** within 5 seconds — do LLM processing asynchronously via queue workers.

Use Redis (BullMQ or Celery) as the message queue for small-to-medium deployments — Chatwoot already requires Redis, so no new infrastructure. Process messages **sequentially per conversation** (use `conversation_id` as a partition key) to maintain message ordering. Implement exponential backoff for LLM API failures and automatic handoff to humans after N consecutive errors.

### Pattern B: Automatic ticket routing and classification

On `conversation_created` or the first `message_created`, classify the customer's intent using an LLM prompt that returns structured JSON (`{"category": "billing", "priority": "high", "team": "billing_team"}`). Then apply the classification via API: add labels (user token), assign to team (bot token), and set priority (bot token). For high-volume deployments, consider **embedding-based classification** — pre-compute embeddings for category exemplars, store in pgvector (which Chatwoot already uses internally), and do cosine similarity matching. This is faster and cheaper than LLM calls for simple routing.

### Pattern C: Agent copilot with summarization

Write AI-generated summaries and suggestions as **private notes** (`"private": true` in the message creation payload). This requires a user token since private notes are outgoing messages that only agents can see. Trigger summarization on handoff (`conversation_status_changed` from `pending` to `open`) or periodically when agents are active. For a richer experience, build a **Dashboard App** — Chatwoot supports embeddable iframes in the agent dashboard that can show a custom copilot UI with suggestion buttons, knowledge base matches, and real-time AI analysis.

### Pattern D: Fully autonomous resolution agent

Combine patterns A and B: classify intent, assess confidence, query external systems (CRM, order database, knowledge base) using LLM function-calling / tool-use, generate a response, and if confidence exceeds a threshold (e.g., **0.85**), send the reply and resolve the conversation via `toggle_status` with `{"status": "resolved"}`. Implement safety guardrails: maximum autonomous turns before escalation (e.g., 5), sentiment monitoring for frustration detection, topic blacklists that always escalate, and audit trails via private notes logging every AI decision. Track CSAT scores separately for bot-resolved versus human-resolved conversations to measure quality.

### The handoff pattern ties everything together

When the AI determines human help is needed, it should: post a private note summarizing the conversation, the customer's intent, and the escalation reason; optionally assign to a specific team; toggle status to `open`; and send a customer-facing message ("I'm connecting you with a team member"). The reverse also works — agents can push conversations back to `pending` to re-engage the bot for routine follow-up steps.

---

## Deploying Chatwoot in Coolify

Coolify includes Chatwoot as an **official one-click service** in its service catalog, handling the multi-container orchestration (Rails app, Sidekiq worker, PostgreSQL with pgvector, Redis) automatically. For production deployments, the community-maintained `fullhouseit/chatwoot-coolify-template` on GitHub adds Active Storage and S3-compatible storage configuration to the base template.

### Essential environment variables

The minimum configuration requires `SECRET_KEY_BASE` (generate with `head /dev/urandom | tr -dc A-Za-z0-9 | head -c 63`), `FRONTEND_URL` (your public domain like `https://chat.yourdomain.com`), `POSTGRES_PASSWORD`, `REDIS_PASSWORD`, SMTP settings for email notifications, and `RAILS_ENV=production`. For Chatwoot v4+, also set `ACTIVE_RECORD_ENCRYPTION_PRIMARY_KEY`, `ACTIVE_RECORD_ENCRYPTION_DETERMINISTIC_KEY`, and `ACTIVE_RECORD_ENCRYPTION_KEY_DERIVATION_SALT` for MFA support.

**Key Coolify gotcha**: environment variables must be defined in the Docker Compose file itself using `${VARIABLE_NAME}` syntax to reference Coolify UI variables — hardcoded values in the compose file won't appear in Coolify's GUI editor. Coolify supports magic variables like `SERVICE_PASSWORD_POSTGRES` for auto-generated secrets.

### Persistent volumes and storage

Define named volumes for PostgreSQL data (`/var/lib/postgresql/data`), Redis data (`/data`), and Chatwoot storage (`/app/storage`) — Coolify auto-prefixes service UUIDs for isolation and persists volumes across redeployments. For production, configure S3-compatible storage (`ACTIVE_STORAGE_SERVICE=s3_compatible`) with MinIO or any S3 provider to offload file attachments from the server.

### Networking for AI service integration

If your AI service runs in the **same Coolify Docker Compose stack**, services communicate via container names on the shared network (e.g., the AI service calls Chatwoot at `http://rails:3000`). Set the webhook URL to `http://ai-agent:8080/webhook`. If the AI service is a **separate Coolify resource**, connect both to the same Docker network. For **external AI services** (cloud functions, separate servers), use public URLs and ensure the AI service can reach Chatwoot's `FRONTEND_URL`.

### Known issues and minimum requirements

A known issue (Coolify GitHub #7524, December 2025) involves Docker Hub authentication failures pulling the `pgvector/pgvector:pg12` image — switch to `pgvector/pgvector:pg16` which is the current recommended version. Another issue (#2238) reports deployment failures from missing environment variables. **Minimum server specs**: 4 GB RAM and 2 CPU cores for light use, but **8 GB RAM and 4 cores are recommended** when running Coolify overhead plus an AI service alongside Chatwoot. Always run `rails db:chatwoot_prepare` (not `db:migrate`) after updates.

---

## Existing community projects and integration examples

The ecosystem of ready-made AI integrations for Chatwoot is substantial and growing.

**Official Chatwoot repositories** include `chatwoot/implementation-examples` (containing a LangChain agent bot with SQL chat, a Botpress bridge, and a Rasa demo), `chatwoot/ai-agents` (a Ruby AI Agents SDK with multi-agent handoffs, tool calling, and support for OpenAI/Anthropic/Gemini — **107 GitHub stars**, actively developed in 2026), and demo repos for Dialogflow and Rasa integration. Dialogflow has a **native first-party integration** built into Chatwoot's UI (Settings → Applications → Dialogflow).

**n8n integration** is the most popular no-code approach. The template "Multichannel AI Assistant with Chatwoot & OpenRouter" (n8n.io/workflows/8260) demonstrates the complete pattern: Chatwoot webhook → n8n → conversation history retrieval → AI agent node → response via HTTP. The community n8n node package `devlikeapro/n8n-nodes-chatwoot` simplifies API operations. Note: a known compatibility issue (Chatwoot GitHub #12720) affects agent bots after v4.6.0 — test thoroughly.

**Evolution API** (`EvolutionAPI/evolution-api`) is widely used in the Brazilian/Portuguese-speaking community, providing a WhatsApp integration layer with built-in connectors to Chatwoot, Dify AI, OpenAI, Typebot, and n8n. This creates a powerful pipeline: WhatsApp → Evolution API → Chatwoot + AI. **Make.com** offers Chatwoot as a community app with AI module combinations; Zapier has no official integration (use webhooks as workaround).

For building from scratch, the most flexible approach is a **custom webhook service** using any language/framework (Python FastAPI, Node.js Express, Ruby) that receives Chatwoot webhook events, processes them with your preferred LLM (OpenAI, Anthropic Claude, local models via Ollama), and responds via the Chatwoot API. No dedicated LangChain or LlamaIndex plugins exist, but the webhook + API pattern makes integration straightforward with any AI framework.

---

## Conclusion

The free Chatwoot Community Edition is a remarkably complete foundation for custom AI agent development. The **AgentBot framework + webhook system + full REST API** provides everything needed without licensing restrictions — Captain AI is the only significant AI feature locked behind a paywall, and you can replicate its capabilities with your own LLM backend. The dual-token strategy (bot token for messages and status, user token for labels and history) is the key architectural detail most tutorials miss. Coolify's one-click Chatwoot template eliminates deployment complexity, though budget **8 GB RAM minimum** when running an AI service alongside. Start with the n8n multichannel template or the official `implementation-examples` repo for rapid prototyping, then graduate to a custom webhook service with Redis queuing for production workloads. The conversation status state machine (`pending` → `open` → `resolved`) is the backbone of every pattern — master it, and the rest follows naturally.