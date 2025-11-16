📅 Telegram AI Meeting Scheduler (Google Calendar + AI Agents)

This project is an AI-powered meeting scheduler that works through Telegram and automatically schedules events in Google Calendar. It is built internally in n8n using an intelligent Three-Agent Architecture to understand user requests, check availability, and create meetings effortlessly.

🚀 Features
Feature	Description
🤖 Natural language scheduling	"Schedule meeting tomorrow 2pm with John"
📧 Email attendee extraction	Detects one or multiple emails automatically
📆 Google Calendar integration	Creates real events with attendees
🧠 Memory-aware conversations	Remembers previous context using Redis
⏳ Conflict checking	Avoids double-booking and suggests alternatives
🎥 Optional Google Meet	Creates Meet links when requested
🌐 Works over Telegram	Users schedule meetings through simple chat
🧩 Three-Agent Architecture

The system is built using 3 specialized AI agents:

Agent	Purpose
Agent 1: Conversation Router	Understands user intent, asks for missing info
Agent 2: Availability Checker	Checks calendar conflicts
Agent 3: Event Creator	Creates the actual calendar event
⚙️ Architecture Diagram (Mermaid & Excalidraw/Erseer Compatible)
Mermaid Code
flowchart TD
A[Telegram Webhook] --> B[Parse Telegram]

B --> C[Agent 1 - Conversation Router]

C -->|Not Meeting| R[Send Response to Telegram]

C -->|Incomplete| R

C -->|Complete Meeting Info| D[Agent 2 - Availability Checker]

D -->|Conflict| R

D -->|Available| E[Agent 3 - Create Event]

E --> F[Google Calendar API]

F --> R[Send Response to Telegram]

Mermaid Sequence Diagram
sequenceDiagram
User->>Telegram Bot: e.g., "Meeting tomorrow 2pm with john@example.com"
Telegram Bot->>n8n: Webhook event
n8n->>Agent 1: Understand message
Agent 1-->>n8n: Missing info? or Full details
n8n->>Agent 2: Check calendar availability
Agent 2->>Google Calendar: Get events
Google Calendar-->>Agent 2: Availability result
Agent 2-->>n8n: Free or Conflict
n8n->>Agent 3: If available, create event
Agent 3->>Google Calendar: Create event
Google Calendar-->>Agent 3: Confirmation
Agent 3-->>n8n: Success message
n8n-->>Telegram Bot: "Meeting scheduled!"
Telegram Bot-->>User: Confirmation

🧠 Example User Conversations
1️⃣ Complete Scheduling

User:

Schedule meeting tomorrow at 2pm with john@example.com about standup


Bot:

✅ Meeting scheduled!
📌 standup
📅 Nov 17, 2025 - 14:00 UTC
👤 john@example.com

2️⃣ Missing Details

User:

Schedule a meeting with alice@example.com


Bot:

What date should the meeting be scheduled for?

3️⃣ Conflict Detection

User:

Meeting tomorrow 10am with bob@example.com


Bot:

⚠️ That time is already booked. Suggested times: 11am, 1pm, 3pm.

🏗 Requirements
Component	Requirement
n8n Version	v1.72+
Google Calendar API	OAuth2 Credentials
Telegram Bot	BotFather token
Redis (optional but recommended)	For chat memory
📌 Environment Variables
Variable	Description
TELEGRAM_BOT_TOKEN	Token from BotFather
GOOGLE_CLIENT_ID	Google OAuth
GOOGLE_CLIENT_SECRET	Google OAuth
REDIS_URL	Redis storage for memory
📦 Installation & Setup
1️⃣ Create Telegram Bot

Message @BotFather:

/newbot

2️⃣ Add Webhook
https://api.telegram.org/bot<token>/setWebhook?url=<your-n8n-webhook-url>

3️⃣ Import n8n Workflow

Import JSON file into n8n.

4️⃣ Connect credentials:

Google Calendar OAuth2

Redis (optional)

🧪 Test Messages
Try sending one of these messages in Telegram
Schedule meeting tomorrow 2pm with alex@example.com
Book call next Monday morning with team@company.com
Meeting Friday 10am with john@example.com and sam@example.com
🛠 Troubleshooting
Issue	Fix
Bot replies "skip:true"	Check webhook message format
Calendar event fails	Ensure Google Calendar OAuth is valid
JSON parsing error	Verify Structured Output Parser schema
Forgot context	Enable Redis memory
📘 Developer Documentation

This contains system logic & prompts for each agent.

Agent 1 Prompt (Router)

Purpose: Detect if message is meeting-related; ask for missing details.

🔽 Copy prompt from here:
(You already have the full prompt in previous messages)

Agent 2 Prompt (Availability)

Checks Google Calendar via tool Get many events.

Agent 3 Prompt (Create Event)

Uses tool Create an event in Google Calendar.

🎯 Future Enhancements

Multi-language (Hindi, Spanish, Arabic, French)

Automatic meeting notes + summarizer

CRM Integrations (HubSpot, Salesforce)

Voice assistant mode

📝 License

MIT License.