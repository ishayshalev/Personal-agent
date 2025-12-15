# Personal AI Agent - MVP Specification

## Overview

A personal AI assistant controlled via WhatsApp that can take actions on your behalf across various platforms (starting with Notion). Built with Vercel AI SDK v6 and Node.js, designed for extensibility.

## Architecture

```
┌─────────────────┐     ┌──────────────────────┐     ┌─────────────────┐
│    WhatsApp     │────▶│   Node.js Server     │────▶│   MCP Client    │
│  (Your Phone)   │◀────│   (AI Agent Core)    │◀────│                 │
└─────────────────┘     └──────────────────────┘     └─────────────────┘
                                  │                          │
                                  ▼                          ▼
                        ┌──────────────────┐       ┌──────────────────┐
                        │   Vercel AI SDK  │       │  Notion MCP      │
                        │   v6 (Agent)     │       │  (hosted/local)  │
                        └──────────────────┘       └──────────────────┘
                                  │                          │
                                  ▼                          ▼
                        ┌──────────────────┐       ┌──────────────────┐
                        │   LLM Provider   │       │  Future MCP      │
                        │ (OpenAI/Anthropic)│       │  Servers...      │
                        └──────────────────┘       └──────────────────┘
```

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
| WhatsApp Integration | Official WhatsApp Cloud API |
| Notion Integration | Official Notion MCP Server (`mcp.notion.com`) |
| Database | SQLite (local) / PostgreSQL (production) |
| Web Framework | Express.js or Fastify |
| LLM Provider | OpenAI / Anthropic (configurable) |

## Core Features (MVP Scope)

### 1. WhatsApp Integration
- Receive messages via WhatsApp Cloud API webhooks
- Send responses back to your WhatsApp
- Support for text messages (media support in v2)
- Message queue for handling rate limits

### 2. AI Agent Core (Vercel AI SDK v6)
- Agent built using the new `Agent` interface from AI SDK v6
- Tool-calling capabilities for executing actions
- Conversation memory (context window management)
- Human-in-the-loop approval for sensitive actions

### 3. Notion Integration via MCP
- Uses official Notion MCP server (hosted at `mcp.notion.com/mcp`)
- All Notion tools provided out-of-the-box via MCP
- Query databases, create/update pages, manage blocks
- Search across workspace
- AI-optimized "Notion-flavored Markdown" format

### 4. Security
- Whitelist of allowed phone numbers (only you)
- API key management via environment variables
- Action confirmation for destructive operations

## Project Structure

```
Personal-agent/
├── src/
│   ├── index.ts                 # Entry point
│   ├── config/
│   │   └── env.ts               # Environment configuration
│   ├── agent/
│   │   ├── index.ts             # Agent setup with AI SDK v6
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
│   │   └── whatsapp/
│   │       ├── client.ts        # WhatsApp API client
│   │       ├── webhook.ts       # Webhook handler
│   │       └── types.ts         # WhatsApp types
│   ├── server/
│   │   ├── index.ts             # Express/Fastify server
│   │   └── routes/
│   │       └── webhook.ts       # Webhook routes
│   └── storage/
│       ├── conversation.ts      # Conversation history
│       └── db.ts                # Database connection
├── tests/
├── .env.example
├── package.json
├── tsconfig.json
└── README.md
```

## API Endpoints

### Webhook Endpoints

| Method | Path | Description |
|--------|------|-------------|
| GET | `/webhook` | WhatsApp webhook verification |
| POST | `/webhook` | Receive WhatsApp messages |

### Health Check

| Method | Path | Description |
|--------|------|-------------|
| GET | `/health` | Server health check |

## Environment Variables

```env
# Server
PORT=3000
NODE_ENV=development

# WhatsApp Cloud API
WHATSAPP_API_TOKEN=your_whatsapp_api_token
WHATSAPP_PHONE_NUMBER_ID=your_phone_number_id
WHATSAPP_BUSINESS_ACCOUNT_ID=your_business_account_id
WHATSAPP_WEBHOOK_VERIFY_TOKEN=your_verify_token

# Allowed Users (comma-separated phone numbers)
ALLOWED_PHONE_NUMBERS=+1234567890

# LLM Provider
OPENAI_API_KEY=your_openai_key
# Or use Anthropic
ANTHROPIC_API_KEY=your_anthropic_key

# Notion MCP (OAuth handled by MCP server)
# For hosted MCP: OAuth flow via mcp.notion.com
# For self-hosted MCP: provide integration token
NOTION_MCP_URL=https://mcp.notion.com/mcp
# NOTION_API_KEY=your_notion_integration_token  # Only if self-hosting

# Database (optional for MVP, can use in-memory)
DATABASE_URL=file:./data/agent.db
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
| Google Calendar | `@anthropic/mcp-google-calendar` (example) |
| GitHub | Official GitHub MCP |
| Slack | Community MCP servers available |
| Custom APIs | Build your own MCP server |

## Message Flow

```
1. You send WhatsApp message
         │
         ▼
2. WhatsApp Cloud API sends webhook to server
         │
         ▼
3. Server validates sender (whitelist check)
         │
         ▼
4. Message passed to AI Agent
         │
         ▼
5. Agent processes message with context
         │
         ▼
6. Agent decides: respond or use tool?
         │
         ├──▶ Tool needed: Execute tool → Get result → Continue
         │
         ▼
7. Generate response
         │
         ▼
8. Send response via WhatsApp API
         │
         ▼
9. You receive response on WhatsApp
```

## Example Interactions

### Query Notion
```
You: "What tasks do I have in my Projects database?"
Agent: [Uses notion_query_database via MCP]
Agent: "You have 5 tasks in your Projects database:
        1. Finish MVP spec (Due: Today)
        2. Set up CI/CD (Due: Tomorrow)
        ..."
```

### Create Note
```
You: "Add a note to my Ideas page: Build a habit tracker app"
Agent: [Uses notion_update_page via MCP]
Agent: "Done! I've added 'Build a habit tracker app' to your Ideas page."
```

### Natural Conversation with Memory
```
You: "Remind me about that startup idea I mentioned last week"
Agent: [Uses recall tool - local]
Agent: "Last week you mentioned wanting to build a personal finance
        app that connects to banks and uses AI for spending insights."
```

### Search Across Workspace
```
You: "Find my notes about machine learning"
Agent: [Uses notion_search via MCP]
Agent: "Found 3 pages mentioning machine learning:
        1. ML Course Notes (last edited 2 days ago)
        2. Project Ideas → AI Features section
        3. Reading List → 'Hands-On ML' book notes"
```

## Setup Requirements

### WhatsApp Business Setup
1. Create a Meta Developer account
2. Create a Meta App with WhatsApp product
3. Set up a WhatsApp Business Account
4. Get API credentials (Access Token, Phone Number ID)
5. Configure webhook URL (needs HTTPS - use ngrok for dev)

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
- **Local Development**: ngrok + local Node.js server
- **Production**: Railway, Render, Fly.io, or any Node.js hosting

## Dependencies

```json
{
  "dependencies": {
    "ai": "^6.0.0-beta",
    "@ai-sdk/openai": "^1.0.0",
    "@ai-sdk/anthropic": "^1.0.0",
    "@modelcontextprotocol/sdk": "^1.0.0",
    "express": "^4.18.0",
    "better-sqlite3": "^11.0.0",
    "zod": "^3.23.0",
    "dotenv": "^16.4.0"
  },
  "devDependencies": {
    "typescript": "^5.4.0",
    "@types/node": "^20.0.0",
    "@types/express": "^4.17.0",
    "tsx": "^4.7.0"
  }
}
```

> Note: `@notionhq/client` is NOT needed - we use Notion via MCP instead.

## Development Phases

### Phase 1: Foundation (Current MVP)
- [x] Project setup with TypeScript
- [ ] Basic Express server with webhook endpoints
- [ ] WhatsApp Cloud API integration
- [ ] AI Agent setup with Vercel AI SDK v6
- [ ] MCP client integration
- [ ] Connect Notion MCP server
- [ ] Simple conversation memory

### Phase 2: Enhanced Intelligence
- [ ] Improved context management
- [ ] Better tool selection
- [ ] Confirmation flow for actions
- [ ] Error handling and retries

### Phase 3: More MCP Servers
- [ ] Google Calendar MCP
- [ ] Email MCP (Gmail/Outlook)
- [ ] GitHub MCP
- [ ] Custom MCP server for your own APIs

### Phase 4: Advanced Features
- [ ] Voice messages support
- [ ] Image understanding
- [ ] Scheduled tasks/reminders
- [ ] Multi-step workflows

## Limitations (MVP)

- Single user only (you)
- Text messages only (no voice/images initially)
- No scheduled/proactive messages
- Basic conversation memory (last N messages)
- Requires always-on server or serverless deployment

## Security Considerations

1. **Phone Number Whitelist**: Only process messages from your number
2. **No Sensitive Data Logging**: Don't log message content in production
3. **API Key Security**: Use environment variables, never commit keys
4. **Webhook Verification**: Validate all incoming webhooks
5. **Rate Limiting**: Implement rate limits to prevent abuse

## Future Expansion Ideas

- **More Platforms**: Telegram, Discord, SMS
- **More MCP Servers**: Calendar, Email, Task managers, Slack, GitHub
- **Proactive Agent**: Scheduled check-ins, reminders
- **Voice Interface**: Process voice messages
- **Local LLM Option**: Ollama for privacy-sensitive tasks
- **Custom MCP Servers**: Build MCP servers for your own APIs/services
- **MCP Server Marketplace**: Easy discovery and connection of new capabilities

---

## Quick Start (After Implementation)

```bash
# Clone and install
git clone <repo>
cd Personal-agent
npm install

# Configure environment
cp .env.example .env
# Edit .env with your API keys

# Development with ngrok
ngrok http 3000
# Update WhatsApp webhook URL with ngrok URL

# Run
npm run dev
```

## References

### AI & Agent Framework
- [Vercel AI SDK v6 Documentation](https://ai-sdk.dev/docs/introduction)
- [AI SDK v6 Beta Announcement](https://ai-sdk.dev/docs/announcing-ai-sdk-6-beta)

### MCP (Model Context Protocol)
- [MCP Specification](https://modelcontextprotocol.io)
- [Notion MCP Documentation](https://developers.notion.com/docs/mcp)
- [Notion MCP Getting Started](https://developers.notion.com/docs/get-started-with-mcp)
- [Notion's Hosted MCP Server Blog](https://www.notion.com/blog/notions-hosted-mcp-server-an-inside-look)
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)

### WhatsApp
- [WhatsApp Cloud API Documentation](https://developers.facebook.com/docs/whatsapp/cloud-api)
- [Official WhatsApp Node.js SDK](https://github.com/WhatsApp/WhatsApp-Nodejs-SDK)
