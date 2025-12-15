# Feature Specification: Gmail Integration

## Overview

Enable the personal AI agent to read, send, manage, and organize emails in Gmail through natural language commands via Slack.

**Status**: Post-MVP Feature (Phase 3)
**Priority**: High
**Complexity**: Medium-High (OAuth 2.0 setup required)

## Use Cases

### Reading & Searching
```
You: "Do I have any unread emails from the last hour?"
Agent: 🔧 Searching Gmail: newer_than:1h is:unread
Agent: ✅ You have 3 unread emails:
       1. From: boss@company.com - "Q4 Planning Meeting" (10 min ago)
       2. From: client@example.com - "Invoice Question" (25 min ago)
       3. From: newsletter@tech.io - "Weekly Digest" (45 min ago)
```

### Sending Emails
```
You: "Send an email to john@example.com saying I'll be 15 minutes late to our meeting"
Agent: 🔧 Composing email...
       To: john@example.com
       Subject: Running Late
       Body: Hi John, I'll be about 15 minutes late to our meeting. See you soon!

Agent: ⚠️ Ready to send. Confirm? [Send] [Edit] [Cancel]
You: *clicks Send*
Agent: ✅ Email sent to john@example.com
```

### Draft Management
```
You: "Draft a response to the client invoice question - say we'll review and get back by Friday"
Agent: 🔧 Creating draft reply to client@example.com...
Agent: ✅ Draft saved. You can review and send from Gmail.
       [Open Draft in Gmail →]
```

### Organization
```
You: "Label all emails from newsletter@tech.io as 'Newsletters' and archive them"
Agent: 🔧 Searching: from:newsletter@tech.io
Agent: 📋 Found 47 emails
Agent: 🔧 Creating label "Newsletters"...
Agent: 🔧 Applying label and archiving...
Agent: ✅ Done! 47 emails labeled and archived.
```

### Smart Summaries
```
You: "Summarize my unread emails"
Agent: 🔧 Fetching unread emails...
Agent: ✅ You have 8 unread emails:

       🔴 Urgent (2):
       • Boss wants Q4 budget by EOD
       • Client escalation on Project X

       📋 Action Required (3):
       • Meeting invite for tomorrow 2pm
       • PR review requested
       • Expense report approval needed

       📰 FYI (3):
       • 2 newsletters, 1 notification
```

## Architecture

### Option A: Community Gmail MCP Server (Recommended for Initial Implementation)

```
┌─────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   AI Agent      │────▶│   MCP Client     │────▶│  Gmail MCP       │
│                 │◀────│                  │◀────│  Server          │
└─────────────────┘     └──────────────────┘     └──────────────────┘
                                                          │
                                                          ▼
                                                 ┌──────────────────┐
                                                 │   Gmail API      │
                                                 │   (OAuth 2.0)    │
                                                 └──────────────────┘
```

**Using**: `@gongrzhe/server-gmail-autoauth-mcp`
- Community-maintained, actively developed
- Auto-authentication support
- Full Gmail API coverage
- Works with existing MCP client

### Option B: Official Google MCP (Future)

Google announced official MCP support (Dec 2025) for:
- Google Maps
- BigQuery
- Compute Engine
- Kubernetes Engine

**Gmail not yet supported** - but likely coming. Design for easy migration.

### Option C: Custom Gmail MCP Server (If Needed)

Build our own using:
- `@modelcontextprotocol/sdk` for MCP server
- `googleapis` for Gmail API
- Custom OAuth 2.0 handling

## Gmail MCP Tools

### Tools Provided by Gmail MCP Server

| Tool | Description | Parameters |
|------|-------------|------------|
| `gmail_search` | Search emails with Gmail syntax | `query: string`, `maxResults?: number` |
| `gmail_get_message` | Get full email content | `messageId: string` |
| `gmail_get_thread` | Get email thread/conversation | `threadId: string` |
| `gmail_send` | Send an email | `to, subject, body, cc?, bcc?, attachments?` |
| `gmail_create_draft` | Create email draft | `to, subject, body, replyTo?` |
| `gmail_send_draft` | Send existing draft | `draftId: string` |
| `gmail_reply` | Reply to an email | `messageId, body, replyAll?` |
| `gmail_forward` | Forward an email | `messageId, to, body?` |
| `gmail_add_label` | Add label to message | `messageId, labelName` |
| `gmail_remove_label` | Remove label | `messageId, labelName` |
| `gmail_create_label` | Create new label | `name, color?` |
| `gmail_list_labels` | List all labels | - |
| `gmail_mark_read` | Mark as read | `messageId` |
| `gmail_mark_unread` | Mark as unread | `messageId` |
| `gmail_archive` | Archive message | `messageId` |
| `gmail_trash` | Move to trash | `messageId` |
| `gmail_get_attachment` | Download attachment | `messageId, attachmentId` |

### Gmail Search Syntax (Passed to `gmail_search`)

| Query | Description |
|-------|-------------|
| `is:unread` | Unread emails |
| `from:email@example.com` | From specific sender |
| `to:email@example.com` | Sent to specific recipient |
| `subject:meeting` | Subject contains "meeting" |
| `has:attachment` | Has attachments |
| `newer_than:1d` | From last day |
| `older_than:1w` | Older than 1 week |
| `in:inbox` | In inbox |
| `in:sent` | In sent folder |
| `label:important` | Has label |
| `is:starred` | Starred emails |

## Authentication

### OAuth 2.0 Flow

```
1. User initiates Gmail connection in Slack
         │
         ▼
2. Agent generates OAuth URL
         │
         ▼
3. User clicks link, authorizes in Google
         │
         ▼
4. Google redirects with auth code
         │
         ▼
5. Agent exchanges code for tokens
         │
         ▼
6. Tokens stored encrypted in database
         │
         ▼
7. Agent uses access token for Gmail API
         │
         ▼
8. Refresh token used when access token expires
```

### Required OAuth Scopes

```typescript
const GMAIL_SCOPES = [
  'https://www.googleapis.com/auth/gmail.readonly',    // Read emails
  'https://www.googleapis.com/auth/gmail.send',        // Send emails
  'https://www.googleapis.com/auth/gmail.compose',     // Create drafts
  'https://www.googleapis.com/auth/gmail.modify',      // Modify labels, archive
  'https://www.googleapis.com/auth/gmail.labels',      // Manage labels
];
```

**Note**: We intentionally avoid `https://mail.google.com/` (full access) to follow least-privilege principle.

### Token Storage

```typescript
// src/db/schema.ts
export const oauthTokens = pgTable('oauth_tokens', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: text('user_id').notNull(),           // Slack user ID
  provider: text('provider').notNull(),         // 'gmail'
  accessToken: text('access_token').notNull(),  // Encrypted
  refreshToken: text('refresh_token').notNull(), // Encrypted
  expiresAt: timestamp('expires_at').notNull(),
  scopes: text('scopes').array(),
  createdAt: timestamp('created_at').defaultNow(),
  updatedAt: timestamp('updated_at').defaultNow(),
});
```

## Project Structure Updates

```
src/
├── mcp/
│   ├── client.ts
│   ├── notion.ts
│   └── gmail.ts              # NEW: Gmail MCP connection
├── integrations/
│   └── gmail/
│       ├── auth.ts           # NEW: OAuth handling
│       ├── oauth-callback.ts # NEW: OAuth callback handler
│       └── types.ts          # NEW: Gmail types
└── db/
    └── schema.ts             # Add oauth_tokens table
```

## Environment Variables

```env
# Gmail OAuth (Google Cloud Console)
GOOGLE_CLIENT_ID=your-client-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your-client-secret
GOOGLE_REDIRECT_URI=https://your-domain.com/oauth/gmail/callback

# Gmail MCP Server (if using community server)
GMAIL_MCP_COMMAND=npx
GMAIL_MCP_ARGS=@gongrzhe/server-gmail-autoauth-mcp
```

## Setup Requirements

### Google Cloud Console Setup

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create new project or select existing
3. Enable Gmail API:
   - APIs & Services → Enable APIs → Search "Gmail API" → Enable
4. Create OAuth credentials:
   - APIs & Services → Credentials → Create Credentials → OAuth client ID
   - Application type: Web application
   - Authorized redirect URIs: `https://your-domain.com/oauth/gmail/callback`
5. Download credentials JSON
6. Configure OAuth consent screen:
   - User type: External (or Internal for Workspace)
   - Add scopes listed above
   - Add test users (your email) for development

### Gmail MCP Server Setup

**Option 1: Use community server**
```bash
# Install globally
npm install -g @gongrzhe/server-gmail-autoauth-mcp

# Or run via npx (recommended)
npx @gongrzhe/server-gmail-autoauth-mcp

# Authenticate
npx @gongrzhe/server-gmail-autoauth-mcp auth
```

**Option 2: Docker**
```bash
docker run -d \
  -e GOOGLE_CLIENT_ID=$GOOGLE_CLIENT_ID \
  -e GOOGLE_CLIENT_SECRET=$GOOGLE_CLIENT_SECRET \
  -v ~/.gmail-mcp:/root/.gmail-mcp \
  gmail-mcp-server
```

## Security Considerations

### Data Protection
1. **Token Encryption**: All OAuth tokens encrypted at rest using AES-256
2. **Minimal Scopes**: Only request necessary permissions
3. **Token Rotation**: Automatically refresh tokens before expiry
4. **Audit Logging**: Log all Gmail operations for security review

### Confirmation Flows
Require explicit confirmation for:
- Sending emails
- Deleting emails (not just trash)
- Bulk operations (>10 emails)
- Operations on emails from important contacts

### Rate Limiting
- Gmail API quota: 250 quota units/user/second
- Implement local rate limiting to stay within bounds
- Queue bulk operations

## User Experience

### Connecting Gmail (First Time)

```
You: "Connect my Gmail"
Agent: 🔗 To connect Gmail, I need your authorization.

       Click here to authorize: [Authorize Gmail →]

       This will allow me to:
       • Read your emails
       • Send emails on your behalf
       • Manage labels and organization

       Your credentials are encrypted and only used for your requests.
```

### Connection Status

```
You: "What accounts are connected?"
Agent: 📧 Connected accounts:
       • Gmail: john@gmail.com (connected 2 days ago)
       • Notion: workspace "Personal" (connected)

       [Disconnect Gmail] [Reconnect]
```

### Error Handling

```
You: "Send email to invalid-email"
Agent: ❌ Invalid email address "invalid-email"
       Please provide a valid email like: name@example.com
```

```
You: "Read my emails"
Agent: ⚠️ Gmail is not connected. Would you like to connect now?
       [Connect Gmail →]
```

## Example Interactions

### Daily Email Summary
```
You: "Give me my morning email briefing"
Agent: 🔧 Fetching emails from last 12 hours...
Agent: ✅ Good morning! Here's your email briefing:

       📬 12 new emails overnight

       🔴 Needs Response (3):
       1. Re: Project Proposal - Sarah wants feedback
       2. Meeting Request - Client call Thursday
       3. Invoice #1234 - Approval needed

       📅 Calendar (2):
       • Team standup at 9am
       • 1:1 with manager at 2pm

       📰 Newsletters (4):
       TechCrunch, Morning Brew, HN Digest, Product Hunt

       🗑️ Promotional (3):
       Auto-archived to "Promotions"

       What would you like to do first?
```

### Quick Reply
```
You: "Reply to Sarah's project proposal email - looks good, let's discuss tomorrow"
Agent: 🔧 Finding Sarah's email...
Agent: 📧 Composing reply:

       To: sarah@company.com
       Subject: Re: Project Proposal

       Hi Sarah,

       Looks good! Let's discuss tomorrow.

       Best,
       [Your name]

       [Send] [Edit] [Cancel]
```

### Email Automation
```
You: "Every email from github notifications, add label 'GitHub' and mark as read"
Agent: ⚠️ This would create an automated rule. Currently I can only
       perform one-time operations.

       Would you like me to:
       1. Apply this to existing GitHub emails now?
       2. Set up a Gmail filter (I'll guide you)?

You: "Option 1"
Agent: 🔧 Searching: from:notifications@github.com
Agent: 📋 Found 234 emails
Agent: 🔧 Creating label "GitHub"...
Agent: 🔧 Applying label and marking as read...
Agent: ✅ Done! 234 emails labeled and marked as read.
```

## Dependencies

```json
{
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0",
    "googleapis": "^144.0.0"
  }
}
```

Or if using community MCP server:
```json
{
  "dependencies": {
    "@gongrzhe/server-gmail-autoauth-mcp": "^0.1.0"
  }
}
```

## Implementation Phases

### Phase 1: Basic Integration
- [ ] Set up Google Cloud project
- [ ] Implement OAuth 2.0 flow
- [ ] Connect Gmail MCP server
- [ ] Basic read/search functionality
- [ ] Token storage and refresh

### Phase 2: Send & Draft
- [ ] Send emails with confirmation
- [ ] Create and manage drafts
- [ ] Reply to threads
- [ ] Forward emails

### Phase 3: Organization
- [ ] Label management
- [ ] Bulk operations
- [ ] Mark read/unread
- [ ] Archive functionality

### Phase 4: Advanced Features
- [ ] Attachment handling
- [ ] Smart summaries with AI
- [ ] Email templates
- [ ] Contact integration

## Migration Path

When Google releases official Gmail MCP support:

1. Update MCP server configuration to use Google's endpoint
2. Re-authenticate users with new OAuth flow
3. Verify tool compatibility (names may differ)
4. Update any custom tool wrappers
5. Remove community server dependency

## References

### Official Documentation
- [Gmail API Documentation](https://developers.google.com/gmail/api)
- [Gmail API Scopes](https://developers.google.com/workspace/gmail/api/auth/scopes)
- [OAuth 2.0 for Web Server Apps](https://developers.google.com/identity/protocols/oauth2/web-server)

### MCP Resources
- [Google Official MCP Announcement](https://cloud.google.com/blog/products/ai-machine-learning/announcing-official-mcp-support-for-google-services)
- [Gmail MCP Server (Community)](https://github.com/GongRzhe/Gmail-MCP-Server)
- [MCP Gmail Python](https://github.com/jeremyjordan/mcp-gmail)

### Google Cloud
- [Google Cloud Console](https://console.cloud.google.com/)
- [Enable Gmail API](https://console.cloud.google.com/apis/library/gmail.googleapis.com)
