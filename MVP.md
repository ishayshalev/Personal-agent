# Personal AI Agent - MVP Specification

## Overview

A personal AI assistant controlled via WhatsApp that can take actions on your behalf across various platforms (starting with Notion). Built with Vercel AI SDK v6 and Node.js, designed for extensibility.

## Architecture

```
┌─────────────────┐     ┌──────────────────────┐     ┌─────────────────┐
│    WhatsApp     │────▶│   Node.js Server     │────▶│   Notion API    │
│  (Your Phone)   │◀────│   (AI Agent Core)    │◀────│                 │
└─────────────────┘     └──────────────────────┘     └─────────────────┘
                                  │
                                  ▼
                        ┌──────────────────┐
                        │   Vercel AI SDK  │
                        │   v6 (Agent)     │
                        └──────────────────┘
                                  │
                                  ▼
                        ┌──────────────────┐
                        │   LLM Provider   │
                        │ (OpenAI/Anthropic)│
                        └──────────────────┘
```

## Tech Stack

| Component | Technology |
|-----------|------------|
| Runtime | Node.js 20+ |
| Language | TypeScript |
| AI Framework | Vercel AI SDK v6 (beta) |
| WhatsApp Integration | Official WhatsApp Cloud API + `@vercel/ai` agents |
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

### 3. Notion Integration (First Platform)
- Query databases
- Create new pages
- Update existing pages
- Add content blocks to pages
- Search across workspace

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
│   │   │   ├── index.ts         # Tool registry
│   │   │   ├── notion.ts        # Notion tools
│   │   │   └── system.ts        # System tools (memory, etc.)
│   │   └── prompts/
│   │       └── system.ts        # System prompts
│   ├── integrations/
│   │   ├── whatsapp/
│   │   │   ├── client.ts        # WhatsApp API client
│   │   │   ├── webhook.ts       # Webhook handler
│   │   │   └── types.ts         # WhatsApp types
│   │   └── notion/
│   │       ├── client.ts        # Notion API client
│   │       └── types.ts         # Notion types
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

# Notion
NOTION_API_KEY=your_notion_integration_token

# Database (optional for MVP, can use in-memory)
DATABASE_URL=file:./data/agent.db
```

## Agent Tools (MVP)

### Notion Tools

| Tool | Description | Parameters |
|------|-------------|------------|
| `notion_search` | Search across Notion workspace | `query: string` |
| `notion_query_database` | Query a Notion database | `database_id: string, filter?: object` |
| `notion_create_page` | Create a new page | `parent_id: string, title: string, content?: string` |
| `notion_update_page` | Update page properties | `page_id: string, properties: object` |
| `notion_append_blocks` | Add content to a page | `page_id: string, blocks: Block[]` |

### System Tools

| Tool | Description | Parameters |
|------|-------------|------------|
| `remember` | Store information for later | `key: string, value: string` |
| `recall` | Retrieve stored information | `key: string` |
| `get_current_time` | Get current date/time | - |

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
Agent: [Uses notion_query_database tool]
Agent: "You have 5 tasks in your Projects database:
        1. Finish MVP spec (Due: Today)
        2. Set up CI/CD (Due: Tomorrow)
        ..."
```

### Create Note
```
You: "Add a note to my Ideas page: Build a habit tracker app"
Agent: [Uses notion_append_blocks tool]
Agent: "Done! I've added 'Build a habit tracker app' to your Ideas page."
```

### Natural Conversation
```
You: "Remind me about that startup idea I mentioned last week"
Agent: [Uses recall tool]
Agent: "Last week you mentioned wanting to build a personal finance
        app that connects to banks and uses AI for spending insights."
```

## Setup Requirements

### WhatsApp Business Setup
1. Create a Meta Developer account
2. Create a Meta App with WhatsApp product
3. Set up a WhatsApp Business Account
4. Get API credentials (Access Token, Phone Number ID)
5. Configure webhook URL (needs HTTPS - use ngrok for dev)

### Notion Setup
1. Go to notion.so/my-integrations
2. Create a new integration
3. Get the Internal Integration Token
4. Share relevant pages/databases with your integration

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
    "@notionhq/client": "^2.2.0",
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

## Development Phases

### Phase 1: Foundation (Current MVP)
- [x] Project setup with TypeScript
- [ ] Basic Express server with webhook endpoints
- [ ] WhatsApp Cloud API integration
- [ ] AI Agent setup with Vercel AI SDK v6
- [ ] Notion integration with basic tools
- [ ] Simple conversation memory

### Phase 2: Enhanced Intelligence
- [ ] Improved context management
- [ ] Better tool selection
- [ ] Confirmation flow for actions
- [ ] Error handling and retries

### Phase 3: More Integrations
- [ ] Google Calendar
- [ ] Email (Gmail/Outlook)
- [ ] Todoist / Linear
- [ ] Custom webhooks

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
- **More Integrations**: Calendar, Email, Task managers
- **Proactive Agent**: Scheduled check-ins, reminders
- **Voice Interface**: Process voice messages
- **Local LLM Option**: Ollama for privacy-sensitive tasks
- **Plugin System**: Easy way to add new tools/integrations

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

- [Vercel AI SDK v6 Documentation](https://ai-sdk.dev/docs/introduction)
- [AI SDK v6 Beta Announcement](https://ai-sdk.dev/docs/announcing-ai-sdk-6-beta)
- [WhatsApp Cloud API Documentation](https://developers.facebook.com/docs/whatsapp/cloud-api)
- [Official WhatsApp Node.js SDK](https://github.com/WhatsApp/WhatsApp-Nodejs-SDK)
- [Notion API Documentation](https://developers.notion.com)
- [Notion Node.js SDK](https://github.com/makenotion/notion-sdk-js)
