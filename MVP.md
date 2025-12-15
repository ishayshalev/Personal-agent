# Personal AI Agent - MVP Specification

## Overview

A personal AI assistant controlled via Slack that can take actions on your behalf across various platforms (starting with Notion). Built with Vercel AI SDK v6 and Node.js, designed for extensibility with full visibility into the agent's chain of thought and actions.

## Architecture

```
┌─────────────────┐     ┌──────────────────────┐     ┌─────────────────┐
│   Slack App     │────▶│   Node.js Server     │────▶│   MCP Client    │
│   (DM / @mention)│◀────│   (Bolt SDK)         │◀────│                 │
└─────────────────┘     └──────────────────────┘     └─────────────────┘
        │                         │                          │
        │ HTTP POST               │                          ▼
        │ (Events API)            │                 ┌──────────────────┐
        │                         ▼                 │  Notion MCP      │
        │               ┌──────────────────┐        └──────────────────┘
        │               │   Vercel AI SDK  │                 │
        │               │   v6 (Agent)     │                 ▼
        │               └──────────────────┘        ┌──────────────────┐
        │                         │                 │  Future MCP      │
        │                         ▼                 │  Servers...      │
        │               ┌──────────────────┐        └──────────────────┘
        └──────────────▶│   LLM Provider   │
                        │ (OpenAI/Anthropic)│
                        └──────────────────┘
```

### Slack Connection Mode

We use the **Events API** (HTTP webhooks) exclusively - same protocol from development to production:

| Aspect | Events API (HTTP) |
|--------|-------------------|
| **Protocol** | HTTP POST requests |
| **Public URL** | Required (ngrok for local dev) |
| **Architecture** | Stateless, request/response |
| **Scalability** | Excellent (serverless-friendly) |

**Why HTTP-only:**
- Same architecture from dev to production
- Stateless = simpler to debug and scale
- Works with serverless (Vercel, Railway, etc.)
- No WebSocket connection management

> For local development, use ngrok or similar tunneling service.

### Why MCP (Model Context Protocol)?

MCP is an open standard (created by Anthropic) that provides a universal way for AI agents to interact with external tools and services. Benefits:

- **Standardized Interface**: One protocol to connect to many services
- **Official Support**: Notion provides an official hosted MCP server
- **AI-Optimized**: Notion's MCP uses "Notion-flavored Markdown" - more token-efficient
- **Extensible**: Easy to add more MCP servers (Calendar, Email, etc.) later
- **Less Code**: No need to write/maintain custom API integrations

## Tech Stack

| Component | Technology |
|-----------|------------|
| Runtime | Node.js 20+ |
| Language | TypeScript |
| AI Framework | Vercel AI SDK v6 (beta) |
| Tool Protocol | MCP (Model Context Protocol) |
| Slack Integration | Bolt SDK (`@slack/bolt`) |
| Notion Integration | Official Notion MCP Server (`mcp.notion.com`) |
| Database | SQLite (local) / PostgreSQL (production) |
| LLM Provider | OpenAI / Anthropic (configurable) |

## Core Features (MVP Scope)

### 1. Slack Integration
- Receive messages via DM or @mentions
- Events API (HTTP webhooks) for all environments
- Rich message formatting with Slack Blocks
- Thread support for conversations
- Stateless architecture (serverless-ready)

### 2. AI Agent Core (Vercel AI SDK v6)
- Agent built using the new `Agent` interface from AI SDK v6
- Tool-calling capabilities for executing actions
- Conversation memory (context window management)
- Human-in-the-loop approval for sensitive actions
- **Streaming responses with step visibility**

### 3. Chain of Thought & Action Visibility
- Real-time visibility into agent reasoning
- Step-by-step action display in Slack
- Tool call notifications before execution
- Result summaries after each action
- Collapsible detail views for complex operations

### 4. Notion Integration via MCP
- Uses official Notion MCP server (hosted at `mcp.notion.com/mcp`)
- All Notion tools provided out-of-the-box via MCP
- Query databases, create/update pages, manage blocks
- Search across workspace
- AI-optimized "Notion-flavored Markdown" format

### 5. Security
- Slack workspace restriction (only your workspace)
- User ID whitelist (only you can interact)
- API key management via environment variables
- Action confirmation for destructive operations

## Chain of Thought Display

The agent shows its thinking process in real-time using Slack message updates:

### Example: Multi-step Task

```
You: "What are my overdue tasks and add them to today's standup notes"

Agent: 🤔 Thinking...
       ├─ I need to find overdue tasks in Notion
       └─ Then append them to today's standup notes

Agent: 🔧 Action: Querying tasks database...
       └─ Filter: due_date < today, status != done

Agent: 📋 Found 3 overdue tasks:
       • Fix login bug (2 days overdue)
       • Review PR #42 (1 day overdue)
       • Update docs (3 days overdue)

Agent: 🔧 Action: Searching for today's standup notes...
       └─ Query: "Standup" + today's date

Agent: 🔧 Action: Appending to standup notes...
       └─ Adding overdue tasks section

Agent: ✅ Done! I've added your 3 overdue tasks to today's
       standup notes. Here's what I added:

       ## Overdue Tasks
       - [ ] Fix login bug (2 days overdue)
       - [ ] Review PR #42 (1 day overdue)
       - [ ] Update docs (3 days overdue)
```

### Visibility Levels

| Level | Shows | Use Case |
|-------|-------|----------|
| **Minimal** | Final result only | Quick answers |
| **Standard** | Actions + results | Default |
| **Verbose** | Full reasoning chain | Debugging/learning |

Configurable per-request: "verbose: what's on my calendar?"

## Project Structure

```
Personal-agent/
├── src/
│   ├── index.ts                 # Entry point
│   ├── config/
│   │   └── env.ts               # Environment configuration
│   ├── agent/
│   │   ├── index.ts             # Agent setup with AI SDK v6
│   │   ├── executor.ts          # Step executor with visibility
│   │   ├── tools/
│   │   │   ├── index.ts         # Tool registry (local tools only)
│   │   │   └── system.ts        # System tools (memory, time, etc.)
│   │   └── prompts/
│   │       └── system.ts        # System prompts
│   ├── mcp/
│   │   ├── client.ts            # MCP client setup
│   │   ├── notion.ts            # Notion MCP server connection
│   │   └── types.ts             # MCP types
│   ├── integrations/
│   │   └── slack/
│   │       ├── app.ts           # Bolt app setup
│   │       ├── listeners/
│   │       │   ├── messages.ts  # DM message handler
│   │       │   ├── mentions.ts  # @mention handler
│   │       │   └── actions.ts   # Button/action handlers
│   │       ├── blocks/
│   │       │   ├── thinking.ts  # Thinking indicator blocks
│   │       │   ├── action.ts    # Action display blocks
│   │       │   └── result.ts    # Result display blocks
│   │       └── types.ts         # Slack types
│   └── storage/
│       ├── conversation.ts      # Conversation history
│       └── db.ts                # Database connection
├── manifest.json                # Slack app manifest
├── tests/
├── .env.example
├── package.json
├── tsconfig.json
└── README.md
```

## Environment Variables

```env
# Server
PORT=3000
NODE_ENV=development

# Slack (Events API)
SLACK_BOT_TOKEN=xoxb-your-bot-token
SLACK_SIGNING_SECRET=your-signing-secret

# Allowed Users (comma-separated Slack user IDs)
ALLOWED_USER_IDS=U0123456789

# LLM Provider
OPENAI_API_KEY=your_openai_key
# Or use Anthropic
ANTHROPIC_API_KEY=your_anthropic_key

# Notion MCP (OAuth handled by MCP server)
# For hosted MCP: OAuth flow via mcp.notion.com
# For self-hosted MCP: provide integration token
NOTION_MCP_URL=https://mcp.notion.com/mcp
# NOTION_API_KEY=your_notion_integration_token  # Only if self-hosting

# Chain of Thought visibility (minimal | standard | verbose)
DEFAULT_VISIBILITY=standard

# Database (optional for MVP, can use in-memory)
DATABASE_URL=file:./data/agent.db
```

## Slack App Manifest

```yaml
display_information:
  name: Personal AI Agent
  description: Your personal AI assistant
  background_color: "#1a1a2e"

features:
  bot_user:
    display_name: AI Agent
    always_online: true
  app_home:
    home_tab_enabled: true
    messages_tab_enabled: true
    messages_tab_read_only_enabled: false

oauth_config:
  scopes:
    bot:
      - chat:write
      - im:history
      - im:read
      - im:write
      - app_mentions:read
      - users:read

settings:
  event_subscriptions:
    request_url: https://your-domain.com/slack/events
    bot_events:
      - message.im
      - app_mention
      - app_home_opened
  interactivity:
    is_enabled: true
    request_url: https://your-domain.com/slack/actions
  socket_mode_enabled: false
```

## Agent Tools (MVP)

### Notion Tools (via MCP - provided automatically)

The Notion MCP server provides these tools out-of-the-box:

| Tool | Description |
|------|-------------|
| `notion_search` | Search across workspace |
| `notion_get_page` | Retrieve a page's content |
| `notion_create_page` | Create a new page |
| `notion_update_page` | Update page content/properties |
| `notion_query_database` | Query a database with filters |
| `notion_create_database_item` | Add item to a database |
| `notion_get_comments` | Get comments on a page |
| `notion_add_comment` | Add a comment to a page |

> Note: Exact tool names/parameters defined by the MCP server. Tools are discovered at runtime via the MCP protocol.

### Local System Tools (custom)

| Tool | Description | Parameters |
|------|-------------|------------|
| `remember` | Store information for later | `key: string, value: string` |
| `recall` | Retrieve stored information | `key: string` |
| `get_current_time` | Get current date/time | - |

### Future MCP Servers

Easy to add more capabilities by connecting additional MCP servers:

| Service | MCP Server |
|---------|------------|
| Google Calendar | Community MCP servers available |
| GitHub | Official GitHub MCP |
| Linear | Community MCP servers available |
| Custom APIs | Build your own MCP server |

## Message Flow

```
1. You send Slack DM or @mention
         │
         ▼
2. Slack sends HTTP POST to your server
         │
         ▼
3. Bolt SDK receives and validates signature
         │
         ▼
4. Server validates user (whitelist check)
         │
         ▼
5. Message passed to AI Agent
         │
         ▼
6. Agent starts processing
         │
         ├──▶ Send "🤔 Thinking..." to Slack
         │
         ▼
7. Agent decides: respond or use tool?
         │
         ├──▶ Tool needed:
         │    ├─ Send "🔧 Action: ..." to Slack
         │    ├─ Execute tool via MCP
         │    ├─ Send "📋 Result: ..." to Slack
         │    └─ Continue processing
         │
         ▼
8. Generate final response
         │
         ▼
9. Send "✅ Done: ..." to Slack
         │
         ▼
10. You see the full chain in Slack thread
```

## Example Interactions

### Query Notion (with Chain of Thought)
```
You: "What tasks do I have in my Projects database?"

Agent: 🤔 Understanding your request...
       └─ Looking for tasks in Projects database

Agent: 🔧 Action: notion_query_database
       └─ Database: Projects
       └─ Filter: type = "task"

Agent: ✅ Found 5 tasks in your Projects database:
       1. Finish MVP spec (Due: Today)
       2. Set up CI/CD (Due: Tomorrow)
       3. Write tests (Due: Friday)
       4. Code review (No due date)
       5. Deploy v1 (Due: Next week)
```

### Multi-step Action
```
You: "Create a new page called 'Meeting Notes' and add today's date"

Agent: 🤔 Planning actions...
       ├─ 1. Create new page "Meeting Notes"
       └─ 2. Add today's date to the page

Agent: 🔧 Action: notion_create_page
       └─ Title: "Meeting Notes"
       └─ Parent: Workspace root

Agent: 📋 Page created successfully
       └─ ID: abc-123-def

Agent: 🔧 Action: notion_update_page
       └─ Adding content: "## December 15, 2025"

Agent: ✅ Done! Created "Meeting Notes" with today's date.
       [View in Notion →]
```

### Verbose Mode
```
You: "verbose: summarize my week"

Agent: 🧠 Verbose mode enabled

Agent: 🤔 Analyzing request...
       ├─ User wants a summary of their week
       ├─ I should check: tasks, meetings, notes
       └─ Time range: last 7 days

Agent: 🔧 Step 1/3: Querying tasks...
       ├─ Tool: notion_query_database
       ├─ Database: Tasks
       ├─ Filter: modified_time > 7 days ago
       └─ Response: 12 tasks found

Agent: 🔧 Step 2/3: Querying calendar...
       ├─ Tool: notion_query_database
       ├─ Database: Calendar
       └─ Response: 8 events found

Agent: 🔧 Step 3/3: Generating summary...
       └─ Analyzing patterns and highlights

Agent: ✅ Your Week in Review:

       📊 Productivity: 8/12 tasks completed (67%)
       📅 Meetings: 8 events, 12 hours total
       🎯 Top focus: MVP development

       Highlights:
       • Completed MVP spec
       • 3 PR reviews done
       • Started CI/CD setup
```

## Setup Requirements

### Slack App Setup
1. Go to api.slack.com/apps → Create New App
2. Choose "From manifest" and paste the manifest above
3. Install to your workspace
4. Get credentials:
   - `SLACK_BOT_TOKEN` (OAuth & Permissions → Bot User OAuth Token)
   - `SLACK_SIGNING_SECRET` (Basic Information → App Credentials → Signing Secret)
5. Set Event Subscriptions Request URL to your server endpoint
   - For local dev: Use ngrok URL (e.g., `https://abc123.ngrok.io/slack/events`)
   - For production: Your deployed server URL

### Notion MCP Setup

**Option A: Hosted MCP (Recommended for MVP)**
1. Connect via OAuth at `mcp.notion.com`
2. Authorize access to your workspace
3. MCP client handles token management automatically

**Option B: Self-Hosted MCP**
1. Go to notion.so/my-integrations
2. Create a new integration
3. Get the Internal Integration Token
4. Share relevant pages/databases with your integration
5. Run your own MCP server (open-source available)

### Deployment Options (MVP)
- **Local Development**: ngrok + `npm run dev`
- **Production**: Railway, Render, Fly.io, Vercel, or any Node.js hosting
- **Recommended for MVP**: Deploy early to Railway/Render (free tier) to avoid ngrok hassle

## Dependencies

```json
{
  "dependencies": {
    "ai": "^6.0.0-beta",
    "@ai-sdk/openai": "^1.0.0",
    "@ai-sdk/anthropic": "^1.0.0",
    "@slack/bolt": "^4.0.0",
    "@modelcontextprotocol/sdk": "^1.0.0",
    "better-sqlite3": "^11.0.0",
    "zod": "^3.23.0",
    "dotenv": "^16.4.0"
  },
  "devDependencies": {
    "typescript": "^5.4.0",
    "@types/node": "^20.0.0",
    "tsx": "^4.7.0"
  }
}
```

## Development Phases

### Phase 1: Foundation (Current MVP)
- [x] Project setup with TypeScript
- [ ] Slack Bolt app with Events API
- [ ] Basic DM and @mention handling
- [ ] AI Agent setup with Vercel AI SDK v6
- [ ] Chain of thought visibility (standard mode)
- [ ] MCP client integration
- [ ] Connect Notion MCP server
- [ ] Simple conversation memory

### Phase 2: Enhanced Intelligence
- [ ] Verbose mode for full visibility
- [ ] Improved context management
- [ ] Better tool selection
- [ ] Confirmation flow for destructive actions
- [ ] Error handling and retries

### Phase 3: More MCP Servers
- [ ] Google Calendar MCP
- [ ] GitHub MCP
- [ ] Linear MCP
- [ ] Custom MCP server for your own APIs

### Phase 4: Advanced Features
- [ ] App Home dashboard
- [ ] Slash commands
- [ ] Scheduled tasks/reminders
- [ ] Multi-step workflow builder
- [ ] Proactive notifications

## Limitations (MVP)

- Single user only (you)
- Text messages only (no file attachments initially)
- No scheduled/proactive messages
- Basic conversation memory (last N messages)
- Requires public URL (ngrok for local dev, or deploy to cloud)

## Security Considerations

1. **User ID Whitelist**: Only process messages from your Slack user ID
2. **Workspace Restriction**: App only installed in your workspace
3. **No Sensitive Data Logging**: Don't log message content in production
4. **API Key Security**: Use environment variables, never commit keys
5. **Token Rotation**: Rotate Slack tokens periodically

## Future Expansion Ideas

- **More Platforms**: Telegram, Discord, CLI
- **More MCP Servers**: Calendar, Email, Task managers, GitHub
- **Proactive Agent**: Scheduled check-ins, reminders
- **App Home Dashboard**: Quick actions, recent activity
- **Slash Commands**: `/ask`, `/task`, `/note`
- **Local LLM Option**: Ollama for privacy-sensitive tasks
- **Custom MCP Servers**: Build MCP servers for your own APIs/services

---

## Quick Start (After Implementation)

```bash
# Clone and install
git clone <repo>
cd Personal-agent
npm install

# Configure environment
cp .env.example .env
# Edit .env with your Slack tokens and API keys

# Start ngrok tunnel (in separate terminal)
ngrok http 3000
# Copy the HTTPS URL (e.g., https://abc123.ngrok.io)

# Update Slack app Event Subscriptions URL to:
# https://abc123.ngrok.io/slack/events

# Run server
npm run dev

# Open Slack and DM your bot!
```

**Pro tip**: Deploy to Railway/Render early to skip the ngrok step entirely.

## References

### AI & Agent Framework
- [Vercel AI SDK v6 Documentation](https://ai-sdk.dev/docs/introduction)
- [AI SDK v6 Beta Announcement](https://ai-sdk.dev/docs/announcing-ai-sdk-6-beta)

### Slack
- [Bolt for JavaScript](https://tools.slack.dev/bolt-js/)
- [Bolt TypeScript Tutorial](https://slack.dev/bolt-js/tutorial/using-typescript)
- [Socket Mode Documentation](https://api.slack.com/apis/socket-mode)
- [Events API Documentation](https://api.slack.com/apis/events-api)
- [Slack Block Kit](https://api.slack.com/block-kit)

### MCP (Model Context Protocol)
- [MCP Specification](https://modelcontextprotocol.io)
- [Notion MCP Documentation](https://developers.notion.com/docs/mcp)
- [Notion MCP Getting Started](https://developers.notion.com/docs/get-started-with-mcp)
- [Notion's Hosted MCP Server Blog](https://www.notion.com/blog/notions-hosted-mcp-server-an-inside-look)
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
