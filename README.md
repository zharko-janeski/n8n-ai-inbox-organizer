📬 AI Inbox Organizer & Priority Alert
demo

An n8n workflow that reads every incoming email with an LLM, classifies it(urgent / newsletter / job-related / invoice / other), and acts on it:labels it, archives newsletters, auto-drafts replies, creates paymentreminders in Google Calendar, and pushes urgent alerts to Telegram.

Runs fully autonomous — and the classifier is a local LLM on my ownhardware: zero API cost, zero data leaves the network.

Architecture
┌─────────────────┐ ┌──────────────────────┐
│ Server PC │ LAN │ Inference PC │
│ n8n (npm) │───────▶│ LM Studio │
│ the workflow │ │ Qwen 2.5 7B Instruct│
└────────┬────────┘ └──────────┬───────────┘
│ │ OpenAI-compatible API :1234
▼ ▼
Gmail / Calendar API Gemini (fallback model)


Flow: **Gmail Trigger → Normalize (Code) → LLM Classify → Validate JSON (Code)
→ Switch → 5 action branches**

| Category | Actions |
|---|---|
| 🚨 urgent | label `AI/Urgent` + Telegram alert with AI summary & priority (1–5) |
| 💼 job-related | label `AI/Jobs` + auto-drafted reply (saved as draft — never sent blindly) |
| 🧾 invoice | label `AI/Invoices` + Google Calendar payment reminder |
| 📰 newsletter | marked as read |
| 🤷 other | explicit No-Op |

## Local inference

![LM Studio serving Qwen over LAN](screenshots/lmstudio.png)

*Qwen 2.5 7B (GGUF Q4_K_S, 4.4 GB) via LM Studio on a separate machine,
served over the LAN through an OpenAI-compatible endpoint. Each log line is a
real email classified locally — ~25 output tokens per classification.*

![Production run](screenshots/canvas.png)

## Design decisions worth stealing

- **Local-first AI** — primary model is Qwen on my own hardware. No tokens,
  no bills, no emails leaving the network. Gemini sits on the **Fallback Model**
  input: if the inference PC is down, the cloud catches the email instead of
  the workflow dying.
- **Never trust the model** — a dedicated validation node strips markdown
  fences, regex-extracts JSON, whitelists categories, clamps priority to 1–5,
  and degrades gracefully (`other`) on parse failure instead of crashing.
- **Version-resilient parsing** — the normalizer handles 3 different Gmail
  trigger output shapes (legacy `payload` format, new pre-parsed headers, demo
  data). Survived an n8n update in production without dropping an email.
- **Reversibility ladder** — actions are ranked by reversibility: labels,
  mark-as-read, drafts. **The workflow can never delete an email**, because AI
  classification *will* eventually misfire.
- **Demo mode** — a manual trigger + sample-email node (disabled by default)
  runs the full pipeline on fake data, no real inbox needed for testing.

## Setup

<details>
<summary>Full setup guide</summary>

1. Import `workflow.json` into n8n
2. Google Cloud project → enable **Gmail API** + **Google Calendar API** →
   OAuth consent screen (add yourself as test user) → OAuth client (Web) →
   paste Client ID/Secret into n8n (use n8n's redirect URL)
3. Create Gmail labels `AI/Urgent`, `AI/Jobs`, `AI/Invoices`, select them in
   the label nodes
4. **Local LLM:** LM Studio → download `qwen2.5-7b-instruct` → Developer tab →
   Start Server → enable *Serve on Local Network* → n8n credential:
   Base URL `http://INFERENCE_PC_IP:1234/v1`, any text as API key,
   **Responses API: OFF**
5. **Fallback:** Google AI Studio API key → Gemini node (a *chat* model — not TTS!)
6. **Telegram:** BotFather → create bot → token into n8n credential → message
   the bot once → get chat ID from `api.telegram.org/bot<TOKEN>/getUpdates`
7. Optional: enable the demo nodes (Manual Trigger + Sample Email) to test
   without touching a real inbox

</details>

## Roadmap
- [ ] Accuracy eval: 50 labeled test emails → publish % here
- [ ] Parse invoice due dates → schedule the reminder on the real deadline
- [ ] Gmail push (watch) instead of 60s polling
- [ ] Weekly newsletter digest to Telegram

## Skills demonstrated
LLM structured-output prompting · output validation & graceful degradation ·
primary/fallback model architecture · OAuth 2.0 (Gmail, Calendar) · self-hosted
inference over LAN (LM Studio) · n8n workflow design · production debugging

---
*Built with [n8n](https://n8n.io) · Classifier: Qwen 2.5 7B (local) with Gemini fallback*