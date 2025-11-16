# 📅 Telegram AI Meeting Scheduler

An intelligent meeting scheduler that integrates Telegram, Google Calendar, and AI agents to automate meeting coordination through natural language conversations.

## Overview

This project uses a three-agent architecture built in n8n to understand user requests, check calendar availability, and automatically create meetings—all through simple Telegram messages.

## ✨ Features

- **Natural Language Processing**: Schedule meetings using conversational language like "Schedule meeting tomorrow 2pm with John"
- **Email Detection**: Automatically extracts attendee emails from messages
- **Google Calendar Integration**: Creates real calendar events with attendees and optional Google Meet links
- **Context-Aware Conversations**: Maintains conversation history using Redis
- **Conflict Detection**: Checks for scheduling conflicts and suggests alternative times
- **Multi-Attendee Support**: Handle meetings with multiple participants
- **Telegram Interface**: Simple, accessible scheduling through Telegram chat

## 🏗️ System Architecture
https://www.mermaidchart.com/d/d2e3f91c-d0e6-4901-89ad-6c7246b3a6de

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         USER LAYER                              │
│                    Telegram Mobile/Desktop                      │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ HTTPS Webhook
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                      TELEGRAM BOT API                           │
│                    api.telegram.org                             │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ POST /webhook
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                       N8N WORKFLOW ENGINE                       │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │              TELEGRAM WEBHOOK RECEIVER                   │ │
│  │  • Parse message (chat_id, text, from, date)            │ │
│  │  • Extract user context                                  │ │
│  └────────────────────┬─────────────────────────────────────┘ │
│                       │                                         │
│                       ▼                                         │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │            REDIS MEMORY RETRIEVAL                        │ │
│  │  • Fetch conversation history by chat_id                 │ │
│  │  • Load previous context (last 10 messages)              │ │
│  └────────────────────┬─────────────────────────────────────┘ │
│                       │                                         │
│                       ▼                                         │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │    AGENT 1: CONVERSATION ROUTER (Groq Llama 3)          │ │
│  │                                                          │ │
│  │  System Prompt:                                          │ │
│  │  • Understand user intent                                │ │
│  │  • Extract entities (date, time, email, subject)         │ │
│  │  • Validate completeness of information                  │ │
│  │  • Route to appropriate workflow                         │ │
│  │                                                          │ │
│  │  Input: User message + conversation history              │ │
│  │  Output: JSON with action & extracted data               │ │
│  │  Tools: NONE (pure reasoning)                            │ │
│  │  Memory: Redis-backed context window                     │ │
│  └────┬──────────────┬──────────────┬──────────────────────┘ │
│       │              │              │                         │
│       │              │              │                         │
│   (NOT MEETING)  (MISSING INFO) (COMPLETE)                    │
│       │              │              │                         │
│       ▼              ▼              ▼                         │
│  ┌─────────┐   ┌─────────┐   ┌──────────────────────────┐   │
│  │ General │   │   Ask   │   │  AGENT 2: AVAILABILITY   │   │
│  │  Chat   │   │  User   │   │  CHECKER (Groq Llama 3)  │   │
│  │ Response│   │  for    │   │                          │   │
│  │         │   │ Details │   │  System Prompt:          │   │
│  └────┬────┘   └────┬────┘   │  • Parse time range      │   │
│       │             │         │  • Call Google Calendar  │   │
│       │             │         │  • Detect conflicts      │   │
│       │             │         │  • Suggest alternatives  │   │
│       │             │         │                          │   │
│       │             │         │  Tools:                  │   │
│       │             │         │  • get_calendar_events() │   │
│       │             │         └────┬─────────────────────┘   │
│       │             │              │                         │
│       │             │              │                         │
│       │             │         (GOOGLE CALENDAR API)          │
│       │             │              │                         │
│       │             │         ┌────┴────────┐                │
│       │             │         │             │                │
│       │             │      (FREE)      (CONFLICT)            │
│       │             │         │             │                │
│       │             │         ▼             ▼                │
│       │             │    ┌─────────┐  ┌──────────┐          │
│       │             │    │ AGENT 3 │  │ Suggest  │          │
│       │             │    │ EVENT   │  │Alternate │          │
│       │             │    │ CREATOR │  │  Times   │          │
│       │             │    │         │  └────┬─────┘          │
│       │             │    │ System  │       │                │
│       │             │    │ Prompt: │       │                │
│       │             │    │ • Format│       │                │
│       │             │    │   event │       │                │
│       │             │    │ • Create│       │                │
│       │             │    │   Meet  │       │                │
│       │             │    │   link  │       │                │
│       │             │    │         │       │                │
│       │             │    │ Tools:  │       │                │
│       │             │    │ • create│       │                │
│       │             │    │  _event()│      │                │
│       │             │    └────┬────┘       │                │
│       │             │         │            │                │
│       │             │         ▼            │                │
│       │             │    (GOOGLE CALENDAR API)              │
│       │             │         │            │                │
│       └─────────────┴─────────┴────────────┘                │
│                              │                               │
│                              ▼                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │         RESPONSE AGGREGATOR & FORMATTER              │   │
│  │  • Merge outputs from all agents                     │   │
│  │  • Format user-friendly message                      │   │
│  │  • Add emojis and structure                          │   │
│  └────────────────────┬─────────────────────────────────┘   │
│                       │                                      │
│                       ▼                                      │
│  ┌──────────────────────────────────────────────────────┐   │
│  │            REDIS MEMORY UPDATE                       │   │
│  │  • Save conversation turn to history                 │   │
│  │  • Update context window                             │   │
│  └────────────────────┬─────────────────────────────────┘   │
│                       │                                      │
└───────────────────────┼──────────────────────────────────────┘
                        │
                        │ Telegram sendMessage API
                        ▼
            ┌────────────────────────┐
            │   TELEGRAM BOT API     │
            │   Send Response        │
            └────────┬───────────────┘
                     │
                     ▼
            ┌────────────────────────┐
            │   USER RECEIVES        │
            │   CONFIRMATION         │
            └────────────────────────┘
```

### Data Flow Sequence

```
┌──────┐                ┌─────────┐              ┌──────────┐           ┌────────────┐
│ User │                │Telegram │              │   n8n    │           │   Redis    │
└──┬───┘                └────┬────┘              └────┬─────┘           └─────┬──────┘
   │                         │                        │                       │
   │ "Meeting tomorrow 2pm"  │                        │                       │
   │─────────────────────────>                        │                       │
   │                         │                        │                       │
   │                         │  POST /webhook         │                       │
   │                         │───────────────────────>│                       │
   │                         │                        │                       │
   │                         │                        │  GET chat_history     │
   │                         │                        │──────────────────────>│
   │                         │                        │                       │
   │                         │                        │<──────────────────────│
   │                         │                        │  [previous messages]  │
   │                         │                        │                       │
   │                         │                        │                       │
   │                         │                  ┌─────▼─────┐                │
   │                         │                  │  Agent 1  │                │
   │                         │                  │  Analyze  │                │
   │                         │                  └─────┬─────┘                │
   │                         │                        │                       │
   │                         │                        │ Missing: email        │
   │                         │                        │                       │
   │                         │                        │  SAVE interaction     │
   │                         │                        │──────────────────────>│
   │                         │                        │                       │
   │                         │  "Who should attend?"  │                       │
   │                         │<───────────────────────│                       │
   │                         │                        │                       │
   │  "Who should attend?"   │                        │                       │
   │<─────────────────────────                        │                       │
   │                         │                        │                       │
   │ "john@example.com"      │                        │                       │
   │─────────────────────────>                        │                       │
   │                         │                        │                       │
   │                         │  POST /webhook         │                       │
   │                         │───────────────────────>│                       │
   │                         │                        │                       │
   │                         │                        │  GET chat_history     │
   │                         │                        │──────────────────────>│
   │                         │                        │                       │
   │                         │                        │<──────────────────────│
   │                         │                        │  [includes email now] │
   │                         │                        │                       │
   │                         │                  ┌─────▼─────┐                │
   │                         │                  │  Agent 1  │                │
   │                         │                  │ Complete! │                │
   │                         │                  └─────┬─────┘                │
   │                         │                        │                       │
   │                         │                  ┌─────▼─────┐                │
   │                         │                  │  Agent 2  │                │
   │                         │                  │   Check   │───────────┐    │
   │                         │                  │ Calendar  │           │    │
   │                         │                  └─────┬─────┘           │    │
   │                         │                        │                 │    │
   │                         │                        │<────────────────┘    │
   │                         │                        │   (FREE)             │
   │                         │                        │                      │
   │                         │                  ┌─────▼─────┐               │
   │                         │                  │  Agent 3  │               │
   │                         │                  │  Create   │───────────┐   │
   │                         │                  │   Event   │           │   │
   │                         │                  └─────┬─────┘           │   │
   │                         │                        │                 │   │
   │                         │                        │<────────────────┘   │
   │                         │                        │   (CREATED)         │
   │                         │                        │                     │
   │                         │                        │  SAVE interaction   │
   │                         │                        │────────────────────>│
   │                         │                        │                     │
   │                         │  "✅ Meeting created"  │                     │
   │                         │<───────────────────────│                     │
   │                         │                        │                     │
   │  "✅ Meeting created"   │                        │                     │
   │<─────────────────────────                        │                     │
   │                         │                        │                     │
```

## 🔧 Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Messaging** | Telegram Bot API | User interface and interaction |
| **Workflow Engine** | n8n (v1.72+) | Orchestrates AI agents and integrations |
| **LLM** | Groq Llama 3.3 70B | Natural language understanding |
| **Calendar** | Google Calendar API v3 | Event creation and management |
| **Memory** | Redis 6.0+ | Conversation context storage |
| **Authentication** | OAuth 2.0 | Google Calendar authorization |
| **Data Format** | JSON | Structured agent communication |
| **Protocol** | HTTPS/REST | Secure API communication |

## 🧠 Three-Agent System

### Agent 1: Conversation Router

**Model**: `llama-3.3-70b-versatile`

**Function**: Intent classification and entity extraction

**Tools**: None (pure reasoning)

**Input Schema**:
```json
{
  "message": "string",
  "chat_history": "array",
  "current_time": "ISO8601 timestamp"
}
```

**Output Schema**:
```json
{
  "action": "ask_user | create_event | reply",
  "question": "string (if ask_user)",
  "summary": "string (if create_event)",
  "start": "ISO8601 (if create_event)",
  "end": "ISO8601 (if create_event)",
  "attendees": ["email@domain.com"],
  "googleMeet": "boolean",
  "message": "string (if reply)"
}
```

**Responsibilities**:
- Parse natural language time expressions ("tomorrow at 2pm" → `2025-11-18T14:00:00Z`)
- Extract attendee emails using regex pattern: `[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}`
- Detect missing required fields: `date`, `time`, `attendee`
- Maintain conversation context for multi-turn interactions
- Route to appropriate workflow path

### Agent 2: Availability Checker

**Model**: `llama-3.3-70b-versatile`

**Function**: Conflict detection and time slot analysis

**Tools**: 
- `get_calendar_events(start_time, end_time, calendar_id)`

**Input Schema**:
```json
{
  "proposed_start": "ISO8601",
  "proposed_end": "ISO8601",
  "calendar_id": "primary"
}
```

**Output Schema**:
```json
{
  "available": "boolean",
  "conflicts": [
    {
      "summary": "string",
      "start": "ISO8601",
      "end": "ISO8601"
    }
  ],
  "suggested_times": ["ISO8601", "ISO8601", "ISO8601"]
}
```

**Algorithm**:
```python
# Pseudo-code for conflict detection
def check_availability(proposed_start, proposed_end):
    existing_events = get_calendar_events(
        start=proposed_start - 1_hour,
        end=proposed_end + 1_hour
    )
    
    conflicts = []
    for event in existing_events:
        if time_overlap(event, proposed_start, proposed_end):
            conflicts.append(event)
    
    if conflicts:
        return suggest_alternatives(proposed_start, existing_events)
    else:
        return {"available": True}
```

### Agent 3: Event Creator

**Model**: `llama-3.3-70b-versatile`

**Function**: Calendar event creation with attendees

**Tools**:
- `create_calendar_event(event_details)`

**Input Schema**:
```json
{
  "summary": "string",
  "description": "string",
  "start": {
    "dateTime": "ISO8601",
    "timeZone": "UTC"
  },
  "end": {
    "dateTime": "ISO8601",
    "timeZone": "UTC"
  },
  "attendees": [
    {"email": "string"}
  ],
  "conferenceData": {
    "createRequest": {
      "requestId": "uuid",
      "conferenceSolutionKey": {"type": "hangoutsMeet"}
    }
  }
}
```

**Google Calendar API Call**:
```http
POST https://www.googleapis.com/calendar/v3/calendars/primary/events
Authorization: Bearer {access_token}
Content-Type: application/json

{
  "summary": "standup",
  "description": "Scheduled via Telegram Meeting Assistant",
  "start": {
    "dateTime": "2025-11-18T14:00:00Z",
    "timeZone": "UTC"
  },
  "end": {
    "dateTime": "2025-11-18T15:00:00Z",
    "timeZone": "UTC"
  },
  "attendees": [
    {"email": "john@example.com"}
  ],
  "conferenceDataVersion": 1
}
```

## 🚀 Getting Started

### Prerequisites

- n8n version 1.72 or higher
- Google Calendar API credentials (OAuth2)
- Telegram Bot token
- Redis instance (recommended for memory persistence)
- Groq API key (free tier available)

### Installation

#### 1. Create a Telegram Bot

1. Message [@BotFather](https://t.me/BotFather) on Telegram
2. Send `/newbot` and follow the prompts
3. Save your bot token

#### 2. Set Up Webhook

Replace `<YOUR_BOT_TOKEN>` and `<YOUR_N8N_WEBHOOK_URL>` with your values:

```bash
curl "https://api.telegram.org/bot<YOUR_BOT_TOKEN>/setWebhook?url=<YOUR_N8N_WEBHOOK_URL>"
```

Expected response:
```json
{"ok": true, "result": true, "description": "Webhook was set"}
```

#### 3. Configure Groq API

1. Visit [Groq Console](https://console.groq.com/)
2. Create an account
3. Generate API key
4. No billing required for free tier (30 requests/minute)

#### 4. Configure Google Calendar API

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project or select an existing one
3. Enable the Google Calendar API
4. Create OAuth2 credentials (Web application)
5. Add authorized redirect URIs for n8n:
   ```
   https://YOUR_N8N_URL/rest/oauth2-credential/callback
   ```
6. Add test users during development (required for unverified apps)

#### 5. Set Up Redis

**Option A: Local Docker**
```bash
docker run -d \
  --name redis \
  -p 6379:6379 \
  redis:7-alpine
```

**Option B: Cloud Redis**
- [Upstash](https://upstash.com/) (Free tier: 10K commands/day)
- [Redis Cloud](https://redis.com/try-free/) (Free 30MB)
- [Railway](https://railway.app/) (Redis plugin)

#### 6. Import n8n Workflow

1. Open n8n
2. Import the provided workflow JSON file
3. Configure credentials:
   - **Telegram Bot**: Add bot token
   - **Groq API**: Add API key
   - **Google Calendar OAuth2**: Complete OAuth flow
   - **Redis**: Add connection URL

### Environment Variables

```env
# Telegram Configuration
TELEGRAM_BOT_TOKEN=your_telegram_bot_token

# Groq LLM Configuration
GROQ_API_KEY=your_groq_api_key
GROQ_MODEL=llama-3.3-70b-versatile

# Google Calendar Configuration
GOOGLE_CLIENT_ID=your_google_client_id
GOOGLE_CLIENT_SECRET=your_google_client_secret
GOOGLE_REDIRECT_URI=https://your-n8n.com/rest/oauth2-credential/callback

# Redis Configuration
REDIS_URL=redis://localhost:6379
REDIS_PASSWORD=your_redis_password (if applicable)
REDIS_DB=0

# n8n Configuration
N8N_WEBHOOK_URL=https://your-n8n.com/webhook/telegram
N8N_HOST=your-n8n.com
N8N_PROTOCOL=https
```

## 💬 Usage Examples

### Example 1: Simple Scheduling

**User:**
```
Schedule meeting tomorrow at 2pm with john@example.com about standup
```

**Bot:**
```
✅ Meeting scheduled!
📌 standup
📅 Nov 18, 2025 - 14:00 UTC
👤 john@example.com
🔗 No Google Meet link
```

### Example 2: Multi-Turn Conversation

**User:**
```
Schedule a meeting with alice@example.com
```

**Bot:**
```
What date and time should the meeting be scheduled for?
```

**User:**
```
Next Monday at 10am
```

**Bot:**
```
What should be the subject of the meeting?
```

**User:**
```
Q4 Planning Review
```

**Bot:**
```
✅ Meeting scheduled!
📌 Q4 Planning Review
📅 Nov 24, 2025 - 10:00 UTC
👤 alice@example.com
```

### Example 3: Conflict Detection

**User:**
```
Meeting tomorrow 10am with bob@example.com
```

**Bot:**
```
⚠️ That time slot is already booked with "Team Standup"

Available alternatives:
• 11:00 AM - 12:00 PM
• 1:00 PM - 2:00 PM
• 3:00 PM - 4:00 PM

Would you like to book one of these times?
```

### Example 4: Google Meet Request

**User:**
```
Schedule video call Friday 3pm with team@company.com, need Meet link
```

**Bot:**
```
✅ Meeting scheduled with Google Meet!
📌 Video Call
📅 Nov 21, 2025 - 15:00 UTC
👥 team@company.com
🎥 meet.google.com/xyz-abcd-123
```

## 🧪 Testing

### Test Messages

Try these sample messages to test your bot:

```
# Basic scheduling
Schedule meeting tomorrow 2pm with alex@example.com

# Multiple attendees
Book call next Monday morning with team@company.com and manager@company.com

# With Google Meet
Schedule video meeting Friday 10am with john@example.com, add Meet link

# Casual language
Set up a catch-up with sarah@example.com next week

# Time zone handling
Meeting Thursday 9am EST with client@company.com
```

### Debugging Tools

**Check Telegram webhook status:**
```bash
curl "https://api.telegram.org/bot<TOKEN>/getWebhookInfo"
```

**Test Redis connection:**
```bash
redis-cli -u redis://localhost:6379 ping
```

**Verify Google Calendar API:**
```bash
curl -H "Authorization: Bearer <ACCESS_TOKEN>" \
  "https://www.googleapis.com/calendar/v3/calendars/primary/events"
```

## 🛠️ Troubleshooting

| Issue | Diagnosis | Solution |
|-------|-----------|----------|
| Bot replies with "skip:true" | Webhook message parsing failure | Verify Telegram webhook format; check n8n Parse JSON node |
| Calendar event creation fails | OAuth token expired or invalid | Refresh Google Calendar OAuth credentials in n8n |
| JSON parsing errors | LLM output format mismatch | Update Structured Output Parser schema; add validation |
| Context not remembered | Redis connection issue | Check Redis URL; verify Memory Window node configuration |
| Webhook not receiving messages | Webhook URL not registered | Re-register webhook with Telegram Bot API |
| Time zone issues | UTC conversion error | Ensure all times converted to ISO8601 UTC format |
| Groq API rate limit | Too many requests | Implement request throttling; upgrade to paid tier |
| Duplicate events | Missing deduplication | Add event ID tracking in Redis |

### Common Error Messages

**Error**: `"event_creator_agent returned no result"`
- **Cause**: Agent 3 didn't receive proper input format
- **Fix**: Check Agent 2 output schema matches Agent 3 input

**Error**: `"Calendar API returned 401 Unauthorized"`
- **Cause**: OAuth token expired
- **Fix**: Re-authenticate Google Calendar in n8n credentials

**Error**: `"Redis connection refused"`
- **Cause**: Redis server not running or wrong URL
- **Fix**: Start Redis server or update connection string

## 📚 Technical Implementation Details

### Redis Memory Schema

```javascript
// Key pattern
const key = `telegram:chat:${chat_id}:history`;

// Value structure
{
  "messages": [
    {
      "role": "user",
      "content": "Schedule meeting tomorrow 2pm",
      "timestamp": "2025-11-17T10:30:00Z"
    },
    {
      "role": "assistant",
      "content": "Who should attend the meeting?",
      "timestamp": "2025-11-17T10:30:05Z"
    }
  ],
  "metadata": {
    "last_updated": "2025-11-17T10:30:05Z",
    "message_count": 10
  }
}

// TTL: 24 hours (conversation expires after 1 day of inactivity)
```

### n8n Workflow Structure

```
Webhook Trigger (Telegram)
  ↓
Set Variables (chat_id, message_text)
  ↓
Redis Get (fetch conversation history)
  ↓
AI Agent (Groq LLM) - Agent 1
  ↓
Switch Node (route by action type)
  ├─→ Branch 1: ask_user
  │     ↓
  │   Format Response
  │     ↓
  │   Telegram Send Message
  │
  ├─→ Branch 2: create_event
  │     ↓
  │   AI Agent (Groq LLM) - Agent 2 (Check Availability)
  │     ↓
  │   Google Calendar: List Events
  │     ↓
  │   IF Node (conflict detection)
  │     ├─→ Conflict: Suggest alternatives
  │     └─→ Available: Continue to Agent 3
  │           ↓
  │         AI Agent (Groq LLM) - Agent 3 (Create Event)
  │           ↓
  │         Google Calendar: Create Event
  │           ↓
  │         Format Success Message
  │
  └─→ Branch 3: reply
        ↓
      Format General Response
        ↓
      Telegram Send Message
  ↓
Redis Set (save conversation turn)
  ↓
End
```

### Rate Limiting & Performance

| Component | Limit | Optimization |
|-----------|-------|--------------|
| Groq API | 30 req/min (free) | Cache common responses |
| Google Calendar API | 1M req/day | Batch availability checks |
| Telegram Bot API | 30 msg/sec | Queue messages if burst |
| Redis | No hard limit | Use pipelining for bulk ops |
| n8n Workflow | Depends on hosting | Add queue for concurrent users |

## 🔮 Future Enhancements

- [ ] **Meeting Cancellation**: "Cancel my meeting with john@example.com tomorrow"
- [ ] **Rescheduling**: "Move my 2pm meeting to 3pm"
- [ ] **Recurring Meetings**: "Schedule weekly standup every Monday 10am"
- [ ] **Multi-language Support**: Spanish, Hindi, Arabic, French
- [ ] **Automatic Meeting Notes**: Post-meeting summary generation
- [ ] **CRM Integration**: Sync with HubSpot, Salesforce
- [ ] **Voice Commands**: Telegram voice message support
- [ ] **Zoom/Teams Integration**: Alternative to Google Meet
- [ ] **Smart Suggestions**: ML-based optimal meeting time suggestions
- [ ] **Meeting Analytics**: Track scheduling patterns and productivity

## 📝 API Reference

### Telegram Webhook Payload

```json
{
  "update_id": 123456789,
  "message": {
    "message_id": 1,
    "from": {
      "id": 987654321,
      "is_bot": false,
      "first_name": "John",
      "username": "johndoe"
    },
    "chat": {
      "id": 987654321,
      "first_name": "John",
      "username": "johndoe",
      "type": "private"
    },
    "date": 1700230400,
    "text": "Schedule meeting tomorrow 2pm with john@example.com"
  }
}
```

### Agent Communication Protocol

```json
// Agent 1 → Agent 2
{
  "action": "check_availability",
  "start": "2025-11-18T14:00:00Z",
  "end": "2025-11-18T15:00:00Z",
  "duration_minutes": 60
}

// Agent 2 → Agent 3
{
  "available": true,
  "proceed": true,
  "event_data": {
    "summary": "standup",
    "start": "2025-11-18T14:00:00Z",
    "end": "2025-11-18T15:00:00Z",
    "attendees": ["john@example.com"]
  }
}

// Agent 3 → Response Formatter
{
  "success": true,
  "event_id": "abc123xyz",
  "event_link": "https://calendar.google.com/event?eid=...",
  "meet_link": "https://meet.google.com/xyz-abcd-123"
}
```

## 📄 License

MIT License

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

### Development Setup

```bash
# Clone the repository
git clone https://github.com/yourusername/telegram-ai-scheduler.git

# Set up environment
cp .env.example .env

# Edit .env with your credentials

# Import workflow to n8n

# Test with sample messages
```

