📬 AI Inbox Organizer & Priority Alert

demo.gif

An n8n automation that classifies incoming Gmail messages with an LLM and routes each email to an appropriate action.

The workflow classifies emails into five categories:

urgent · newsletter · job-related · invoice · other

Based on the classification, it can apply Gmail labels, archive newsletters, create reply drafts, schedule payment reminders in Google Calendar, or send urgent notifications to Telegram.

The primary classifier runs locally using Qwen 2.5 7B through LM Studio, with Gemini configured as a fallback model.

Architecture
                         ┌──────────────────────┐
                         │      Inference PC     │
                         │                      │
                         │      LM Studio       │
                         │   Qwen 2.5 7B        │
                         └──────────▲───────────┘
                                    │
                         OpenAI-compatible API
                                    │
┌─────────────────┐                │
│    Server PC    │                │
│                 │                │
│      n8n        │────────────────┘
│    Workflow     │
└────────┬────────┘
         │
         ├────────── Gmail
         │
         ├────────── Google Calendar
         │
         └────────── Telegram

                  Gemini
               (Fallback)
Workflow
Gmail Trigger
      ↓
Prepare Email
      ↓
Classify Email
      ↓
Parse AI Output
      ↓
Route by Category
      ↓
┌──────────┬─────────────┬────────────┬─────────────┬─────────┐
│  Urgent  │  Newsletter │ Job-related│   Invoice   │  Other  │
└──────────┴─────────────┴────────────┴─────────────┴─────────┘
Email Routing
Category	Action
Urgent	Adds AI/Urgent label and sends a Telegram alert containing the AI summary and priority (1–5).
Job-related	Adds AI/Jobs label and creates a reply draft. The reply is never sent automatically.
Invoice	Adds AI/Invoices label and creates a Google Calendar payment reminder.
Newsletter	Marks the email as read.
Other	No action is performed.
Local LLM
lmstudio.png
The primary classifier runs on a separate machine using Qwen 2.5 7B through LM Studio.

Model:       Qwen 2.5 7B Instruct
Format:      GGUF Q4_K_S
Size:        4.4 GB
Interface:   OpenAI-compatible API
Network:     Local LAN
Endpoint:    :1234

The n8n server communicates with the inference machine over the local network.

Gemini is configured as the fallback model so the workflow can continue processing emails if the local inference server is unavailable.

(canvas.png)

Output Validation

The LLM output is passed through a dedicated validation step before the workflow performs any actions.

The validation handles:

Markdown code fences
JSON extraction
Allowed category validation
Priority validation and clamping to 1–5
Invalid output fallback to other

This keeps the routing logic independent from the model's raw response.

Gmail Input Normalization

The workflow normalizes Gmail data before classification.

The normalization logic supports three Gmail trigger output formats:

Legacy payload format
Pre-parsed headers
Demo/sample data

This allows the same classification pipeline to work with different Gmail trigger output structures.

Action Safety

The workflow intentionally uses reversible actions.

Labels
   ↓
Mark as read
   ↓
Create draft

The workflow never deletes emails.

For job-related emails, the AI only creates a Gmail draft. The message remains under manual control and is never sent automatically.

Demo Mode

The workflow includes a disabled-by-default demo path using:

Manual Trigger
      ↓
Sample Email
      ↓
Full workflow

This allows the complete classification and routing pipeline to be tested without using a real inbox.

Setup
<details> <summary><strong>Setup Guide</strong></summary>
1. Import the workflow

Import workflow.json into n8n.

2. Google APIs

Create a Google Cloud project and enable:

Gmail API
Google Calendar API

Configure the OAuth consent screen, add yourself as a test user, create a Web OAuth client, and configure the credentials in n8n using n8n's redirect URL.

3. Gmail Labels

Create the following Gmail labels:

AI/Urgent
AI/Jobs
AI/Invoices

Select the corresponding labels in the workflow.

4. Local LLM

In LM Studio:

Download qwen2.5-7b-instruct
Open the Developer tab
Start the server
Enable Serve on Local Network

Configure the n8n credential with:

Base URL: http://INFERENCE_PC_IP:1234/v1
API Key: any text
Responses API: OFF
5. Gemini Fallback

Add a Google AI Studio API key to the Gemini chat model used as the fallback.

6. Telegram

Create a Telegram bot through BotFather, add the bot token to n8n, and send the bot a message.

The chat ID can then be retrieved through the Telegram Bot API.

7. Demo Mode

Enable the Manual Trigger and Sample Email nodes to test the workflow without connecting it to a real inbox.

</details>
Built With

n8n · Gmail API · Google Calendar · Telegram · LM Studio · Qwen 2.5 7B · Gemini

Primary classifier: Qwen 2.5 7B running locally via LM Studio · Fallback: Gemini
