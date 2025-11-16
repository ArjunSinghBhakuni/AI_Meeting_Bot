📚 AI Meeting Scheduler — Full Technical Documentation

This document explains every step and component involved in building the AI-powered Telegram Meeting Scheduler that automatically creates Google Calendar events using Groq Llama 3, n8n, and Redis conversation memory.

🧩 System Overview

This system allows a Telegram user to schedule meetings in natural language:

"Schedule meeting tomorrow 2pm with john@example.com
 about standup"

The workflow uses three specialized AI agents:

Agent	Responsibility
Agent 1 – Meeting Interpreter	Understands message & extracts scheduling data
Agent 2 – Availability Checker	Checks Google Calendar for conflicts
Agent 3 – Event Creator	Actually creates calendar event

All logic runs inside n8n, with Groq Llama 3 as LLM and Google Calendar API for scheduling.

🏗 System Architecture
Telegram Webhook
      ↓
Parse Telegram message
      ↓
AI Agent 1: Interpret Request
      ↓
If incomplete → Ask user for missing info
If complete → Agent 2 checks availability
      ↓
Agent 2 Google Calendar Check
      ↓
If conflict → Suggest alternative times
If free → Agent 3 creates calendar event
      ↓
Event Confirmation sent on Telegram

🔧 Technology Stack
Component	Technology
Messaging	Telegram Bot API
Workflow Engine	n8n
AI LLM	Groq Llama 3 (free API)
Calendar	Google Calendar API
Auth	Google Cloud OAuth2
Memory	Redis (chat context)
Server	Self-hosted / Cloud / n8n Cloud
1️⃣ Telegram Bot Setup
Step 1: Create bot using BotFather

Open Telegram → Search @BotFather → Send:

/newbot


Provide:

Bot name

Bot username (must end in bot)

You will receive:

Your bot token: xxxxxxxxxxxxxxxxxx


Save this. It becomes:

TELEGRAM_BOT_TOKEN

Step 2: Connect Webhook to n8n

Set webhook URL:

https://api.telegram.org/bot<TELEGRAM_BOT_TOKEN>/setWebhook?url=https://YOUR_DOMAIN/webhook/telegram


Response should be:

"ok": true, "result": true


n8n Node used:

Webhook Node (POST)
Path: telegram

2️⃣ Groq Free API Setup (Llama 3)
Step 1: Create account

https://console.groq.com/

Step 2: Generate API Key
GROQ_API_KEY=xxxxx

Step 3: n8n AI Model Node

Choose model:

llama-3.3-70b-versatile


No billing required.

3️⃣ Google Calendar API + GCP Setup
Step 1: Go to Google Cloud Console

https://console.cloud.google.com/

Step 2: Create project
Step 3: Enable API

Go to APIs & Services > Enable API, search:

Google Calendar API

Step 4: Create OAuth Credentials

Go to Credentials > Create OAuth client ID

Application type → Web Application

Authorized redirect URI:

https://YOUR_N8N_URL/rest/oauth2-credential/callback


Copy:

GOOGLE_CLIENT_ID

GOOGLE_CLIENT_SECRET

Step 5: Add to n8n Google Calendar Credential Settings
4️⃣ Redis Memory Setup (Optional but recommended)
Why?

It allows the bot to remember conversation context:

Example:

User: schedule meeting with john@example.com
Bot: When should it be?
User: Tomorrow 3pm
→ Bot knows the attendee from earlier

Environment variable:
REDIS_URL=redis://localhost:6379

In n8n:

Use node:

Memory Buffer Window (Redis)
Session Key = Telegram chat_id

5️⃣ AI Agent Prompt (Used in n8n)

Core system prompt:

You are a Meeting Creator AI that automatically schedules Google Calendar events.
...
Respond ONLY with JSON.


Model Response Output MUST be one of:

Ask for missing info:
{
  "action": "ask_user",
  "question": "What time should the meeting start?"
}

Create event:
{
  "action": "create_event",
  "summary": "standup",
  "description": "Scheduled via Telegram Meeting Assistant",
  "start": "2025-11-17T14:00:00Z",
  "end": "2025-11-17T15:00:00Z",
  "attendees": ["john@example.com"],
  "googleMeet": false
}

General reply:
{
  "action": "reply",
  "message": "How can I help you with your meetings?"
}

6️⃣ n8n Node Flow (Visual Summary)
Step	Node
1	Telegram Webhook
2	Parse Telegram Message
3	Groq LLM + Prompt
4	Switch by action
5A	Reply back to Telegram
5B	Google Calendar Availability
5C	Create Calendar Event
6	Send Response via Telegram API
7️⃣ Example Supported User Commands
User Says	Bot Action
Schedule meeting tomorrow 2pm with john@example.com about standup	Creates event
Meeting with alex@gmail.com	Asks for date/time
Book call Friday morning with team@company.com	Interprets morning = 09:00 UTC
Cancel meeting tomorrow	(Future enhancement)
8️⃣ Installation Summary
Required Environment Variables
TELEGRAM_BOT_TOKEN=
GROQ_API_KEY=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
REDIS_URL=redis://localhost:6379

Recommended Deployment
Option	Works?
n8n Cloud	✅ Best option
Docker Compose	✅ Recommended
Railway / Render / Fly.io	✅ Scalable
9️⃣ Future Enhancements (Roadmap)
Feature	Status
Cancel meetings via Telegram	⏳ Planned
Reschedule meetings	⏳ Planned
Multi-language support	⏳ Planned
Google Meet auto-link	✔ Supported
Integrate Zoom / Teams	⏳ Possible
🔚 Conclusion

This system enables hands-free, natural language meeting scheduling using:

🧠 AI (Groq Llama 3)

📆 Google Calendar

📲 Telegram

⚙ n8n automation

🔁 Redis memory

It reduces friction and saves time by eliminating manual calendar management.

🎉 End of Documentation
